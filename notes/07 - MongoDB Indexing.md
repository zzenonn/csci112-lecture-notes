# MongoDB Indexing

CSCI 112 / 212 - Contemporary Databases
Department of Computer Science

---

## Overview

This note covers MongoDB indexing from first principles: how queries are executed, how to diagnose performance with `explain()`, what index types exist and when to use each, how to design indexes that complement the data modeling patterns from Module 08, and how to manage indexes safely in production.

All examples use **PyMongo** running on your laptop, connecting to `mongod` on a VM, against the **`labs.movies`** collection — the same 44,488-document dataset used in Modules 04 and 05. Using a real, large collection makes the performance difference between COLLSCAN and IXSCAN concrete and measurable.

### The `labs.movies` collection

Each document looks like this:

```json
{
  "_id": ObjectId("573a1390f29313caabcd421c"),
  "title": "A Turn of the Century Illusionist",
  "year": 1899,
  "runtime": 1,
  "cast": ["Georges Méliès"],
  "plot": "A short silent film ...",
  "type": "movie",
  "directors": ["Georges Méliès"],
  "imdb": { "rating": 6.6, "votes": 580, "id": 246 },
  "countries": ["France"],
  "genres": ["Short"],
  "tomatoes": {
    "viewer": { "rating": 3.8, "numReviews": 32 },
    "lastUpdated": ISODate("2015-08-20T18:46:44.000Z")
  }
}
```

Key fields for indexing demos:

| Field | Type | Cardinality | Notes |
|-------|------|-------------|-------|
| `year` | integer | high (~130 distinct years) | equality and range queries |
| `title` | string | very high (nearly unique) | single-field, covered queries |
| `genres` | array | moderate (~28 genres) | multikey index |
| `cast` | array | very high | multikey index |
| `imdb.rating` | float | moderate (0–10) | compound ESR, partial index |
| `type` | string | **very low** (3 values) | cardinality demo |
| `tomatoes.lastUpdated` | Date | high | TTL index |

---

## PyMongo Setup

```python
from pymongo import MongoClient, ASCENDING, DESCENDING, TEXT

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db     = client["labs"]
movies = db["movies"]
```

---

## Part I: How MongoDB Executes Queries

### COLLSCAN vs IXSCAN

When MongoDB processes a query it selects a **query plan**. The two most important plan stages are:

- **COLLSCAN** — collection scan: reads every document in the collection and checks each one against the filter. Cost is O(N) where N is the number of documents.
- **IXSCAN** — index scan: walks the B-tree index to find only the matching keys, then fetches those documents. Cost is O(log N + K) where K is the number of matching documents.

For 44,488 movies this is the difference between reading 44,488 documents and reading 550.

### `explain()` in PyMongo

```python
import pprint

result = movies.find({"year": 1994}).explain("executionStats")
pprint.pprint(result)
```

The two sections you care about:

**`queryPlanner.winningPlan`** — the plan MongoDB chose:

```
"winningPlan": {
  "stage": "COLLSCAN",
  "filter": { "year": { "$eq": 1994 } }
}
```

**`executionStats`** — what actually happened:

```
"executionStats": {
  "nReturned": 550,
  "totalDocsExamined": 44488,
  "totalKeysExamined": 0,
  "executionTimeMillis": 17
}
```

Key ratio: `nReturned / totalDocsExamined`. Here it is `550 / 44488 = 0.012` — only 1.2% of scanned documents matched. The other 98.8% were wasted work.

After creating an index the plan changes:

```
"winningPlan": {
  "stage": "FETCH",
  "inputStage": {
    "stage": "IXSCAN",
    "indexName": "idx_year",
    "keyPattern": { "year": 1 }
  }
}
```

```
"executionStats": {
  "nReturned": 550,
  "totalDocsExamined": 550,
  "totalKeysExamined": 550,
  "executionTimeMillis": 1
}
```

The ratio is now `550 / 550 = 1.0` — every document examined was returned. Time dropped from **17 ms to 1 ms**.

### Before / After Summary

| Metric | No index (COLLSCAN) | With index (IXSCAN) |
|--------|--------------------:|--------------------:|
| `nReturned` | 550 | 550 |
| `totalDocsExamined` | 44,488 | 550 |
| `totalKeysExamined` | 0 | 550 |
| `executionTimeMillis` | 17 ms | 1 ms |
| Efficiency ratio | 1.2% | 100% |

### Index Cardinality and Selectivity

**Cardinality** is the number of distinct values a field has. **Selectivity** is what fraction of documents a query eliminates.

The `type` field in `labs.movies` has only three values: `"movie"`, `"series"`, `"N/A"`. An index on `type` is nearly useless for `{type: "movie"}` — 99% of documents match, so the index saves almost no work:

```
# Low cardinality: type has 3 values, "movie" matches 44,075 of 44,488 docs
"stage": "FETCH",
"inputStage": "IXSCAN",
"nReturned": 44075,
"totalDocsExamined": 44075,
"executionTimeMillis": 27
```

MongoDB used the index but still examined 44,075 documents — nearly a full scan. The optimizer may skip such an index entirely on a different query.

Contrast with `year` (high cardinality, ~130 distinct values): a query for one year returns ~550 of 44,488 documents — a selectivity of 1.2%, making the index highly effective.

**Rule of thumb:** an index is most beneficial when the query matches fewer than ~5% of the collection.

---

## Part II: Index Types

### Single-Field Index

```python
# Ascending on year
movies.create_index("year", name="idx_year")

# Descending on imdb.rating (dot notation for nested field)
movies.create_index([("imdb.rating", DESCENDING)], name="idx_rating_desc")

# Unique on title (will error if duplicates exist — movies does have some)
# db["users"].create_index("email", unique=True)

# List all indexes
for idx in movies.list_indexes():
    print(idx["name"], idx["key"])
```

**explain() output after creating `idx_year`:**

```
winningPlan:
  stage: FETCH
  inputStage:
    stage: IXSCAN
    indexName: idx_year

executionStats:
  nReturned:          550
  totalDocsExamined:  550
  totalKeysExamined:  550
  executionTimeMillis: 1
```

### Compound Index and the ESR Rule

A compound index covers multiple fields. MongoDB can use it for queries that filter or sort on a **left prefix** of those fields.

The **ESR rule** gives the optimal field order:

1. **E**quality fields first — `==` comparisons
2. **S**ort fields second — fields used in `sort()`
3. **R**ange fields last — `$gt`, `$lt`, `$in`, `$regex`

#### Why this order — the mechanical explanation

A B-tree stores keys in sorted order. To understand ESR, picture walking that sorted list:

**Equality first:** An equality predicate (`genres == "Action"`) jumps the B-tree walk to a contiguous block of matching keys. All qualifying entries are adjacent, so every step after is confined to that small block — eliminating the rest of the collection instantly.

**Sort second:** Within the equality block the keys are still in B-tree order. If the sort field is next in the index, MongoDB reads results in the correct order directly — no in-memory sort step. This is called an *index-provided sort*.

**Range last:** A range predicate (`imdb.rating > 7`) matches a contiguous stretch of keys. If it came before the sort field, the matches would be spread across many sub-ranges — MongoDB could not guarantee sort order and would have to collect all matches then sort them in memory (a `SORT` stage in the plan). Putting range last avoids this: equality has already narrowed the set and sort order has been established before the range filter is applied.

**What happens when you violate ESR:**

| Violation | Consequence |
|-----------|-------------|
| Range before Sort | In-memory `SORT` stage required |
| Range before Equality | Scans much more of the index |
| Sort before Equality | Index walk covers too many keys |

#### Demo: genres (E) + imdb.rating (S + R)

Query: all Action movies rated above 7, sorted by rating descending.

**Before index (COLLSCAN + in-memory SORT):**

```
winningPlan:
  stage: SORT              ← in-memory sort required
  inputStage: COLLSCAN

executionStats:
  nReturned:          1071
  totalDocsExamined:  44488
  executionTimeMillis: 24
```

```python
movies.create_index(
    [("genres", ASCENDING), ("imdb.rating", DESCENDING)],
    name="idx_genres_rating_esr"
)
```

**After ESR compound index (FETCH ← IXSCAN, no SORT):**

```
winningPlan:
  stage: FETCH
  inputStage:
    stage: IXSCAN
    indexName: idx_genres_rating_esr
    isMultiKey: true        ← genres is an array field

executionStats:
  nReturned:          1071
  totalDocsExamined:  1071
  totalKeysExamined:  1071
  executionTimeMillis: 2
```

The `SORT` stage is gone — results come out of the index already in order. Time dropped from **24 ms to 2 ms**.

#### Index prefix rule

Compound index `(a, b, c)` supports queries on left prefixes:
- filter on `a` ✓
- filter on `a` and `b` ✓
- sort on `a` (with equality on `a`) ✓
- filter starting at `b` or `c` alone ✗

### Multikey Index

When you index an array field, MongoDB creates one index entry per array element — this is a **multikey index**. It activates automatically; no special option needed.

```python
movies.create_index("cast", name="idx_cast")

# Now this uses IXSCAN:
movies.find({"cast": "Tom Hanks"})
```

**explain() output:**

```
winningPlan:
  stage: FETCH
  inputStage:
    stage: IXSCAN
    indexName: idx_cast
    isMultiKey: true     ← confirms array field

executionStats:
  nReturned:          50
  totalDocsExamined:  50
  totalKeysExamined:  50
  executionTimeMillis: 1
```

**Compound multikey constraint:** only one array field per compound index. MongoDB cannot index the cartesian product of two arrays:

```python
# OK — cast is array, year is scalar
movies.create_index([("cast", ASCENDING), ("year", ASCENDING)])

# ERROR — both genres and cast are arrays
movies.create_index([("genres", ASCENDING), ("cast", ASCENDING)])
# → OperationFailure: cannot index parallel arrays
```

### Partial Index

Indexes only documents matching a filter expression — smaller than a full index, lower write overhead.

```python
# Only index highly-rated movies (imdb.rating >= 8)
# 2,284 of 44,488 docs qualify — index is ~5% the size of a full index
movies.create_index(
    [("imdb.rating", DESCENDING)],
    partialFilterExpression={"imdb.rating": {"$gte": 8}},
    name="idx_rating_partial"
)
```

**explain() output — query on highly-rated movies:**

```
winningPlan:
  stage: FETCH
  inputStage:
    stage: IXSCAN
    indexName: idx_rating_partial

executionStats:
  nReturned:          2284
  totalDocsExamined:  2284
  totalKeysExamined:  2284
  executionTimeMillis: 3
```

For the query `{"imdb.rating": {"$gte": 8}}` the partial index is used. For `{"imdb.rating": {"$gte": 6}}` MongoDB falls back to COLLSCAN — the query filter does not guarantee the partial expression is true, so the index cannot be trusted to contain all matches.

### Sparse Index

Only indexes documents where the field exists. Useful for optional fields:

```python
# Only index movies that have a runtime field
movies.create_index("runtime", sparse=True, name="idx_runtime_sparse")
```

For most schemas, a **partial index is more expressive** than sparse — you can specify exactly which documents to include rather than relying on field existence alone.

### TTL Index

Automatically deletes documents after a time-to-live period. The indexed field must be a BSON `Date`:

```python
from datetime import datetime, timezone, timedelta

# Create a sessions collection with automatic 1-hour expiry
db["sessions"].create_index(
    "created_at",
    expireAfterSeconds=3600,
    name="idx_sessions_ttl"
)

db["sessions"].insert_one({
    "user": "mjlsaavedra",
    "token": "abc123",
    "created_at": datetime.now(timezone.utc)
})
# This document is automatically deleted ~1 hour after created_at
```

The `tomatoes.lastUpdated` field in `labs.movies` is a Date — it could serve as a TTL field in a real streaming-updates scenario.

**Constraints:** single-field only; cleanup thread runs every ~60 seconds so expiration is approximate.

### Text Index

Full-text search across string fields:

```python
movies.create_index(
    [("title", TEXT), ("plot", TEXT)],
    name="idx_text_title_plot"
)

# Search for movies about space adventure
movies.find({"$text": {"$search": "space adventure"}})
```

**explain() output:**

```
winningPlan:
  stage: TEXT_MATCH
  inputStage: FETCH

executionStats:
  nReturned:          921
  totalDocsExamined:  921
  executionTimeMillis: 2
```

```python
# Sort by relevance score
movies.find(
    {"$text": {"$search": "space adventure"}},
    {"score": {"$meta": "textScore"}, "title": 1}
).sort([("score", {"$meta": "textScore"})])
```

**Constraint:** only one text index per collection; it can span multiple fields.

---

## Part III: Index Design for Data Modeling

### Indexes for the Patterns from Module 08

**Inheritance Pattern** (`product_type` discriminator on `books`):

```python
# Queries almost always filter by product_type first
books.create_index("product_type")

# Combined with sort — books by type sorted by price (ESR)
books.create_index([("product_type", ASCENDING), ("price", ASCENDING)])
```

**Schema Versioning Pattern** (`schema_version` — optional field):

```python
# Sparse: old documents without schema_version are excluded from the index
customers.create_index("schema_version", sparse=True)
```

**Single Collection Pattern** (`docType` + `relatedTo`):

```python
# Compound ESR: both are equality filters, relatedTo first
# (the common query is "all docs related to product X")
db["books_catalog"].create_index(
    [("relatedTo", ASCENDING), ("docType", ASCENDING)]
)
```

**Computed Pattern** (sort by cached `average_rating`):

```python
# Descending — "top rated books" query
books.create_index([("rating.average_rating", DESCENDING)])
```

### Covered Queries

A covered query is served entirely from the index — MongoDB never fetches the document. `totalDocsExamined` will be **0**.

```python
movies.create_index(
    [("genres", ASCENDING), ("title", ASCENDING)],
    name="idx_genres_title_covered"
)

# Projection includes only indexed fields, _id excluded
movies.find(
    {"genres": "Action"},
    {"_id": 0, "title": 1}
).explain("executionStats")
```

**explain() output:**

```
winningPlan:
  stage: PROJECTION_COVERED
  inputStage:
    stage: IXSCAN
    indexName: idx_genres_title_covered

executionStats:
  nReturned:          5655
  totalDocsExamined:  0       ← zero document reads
  totalKeysExamined:  5655
  executionTimeMillis: 3
```

`totalDocsExamined: 0` — no documents were read at all. The entire result was served from the index.

For a query to be covered:
1. All filter fields must be in the index.
2. All projected fields must be in the index.
3. `_id` must be excluded (or included in the index).

### Index Intersection vs Compound Index

MongoDB can sometimes merge two single-field indexes in memory, but a compound index is almost always faster:

```python
# Option A: two separate indexes (index intersection — implicit, unpredictable)
movies.create_index("genres")
movies.create_index("year")

# Option B: one compound index (explicit, predictable, supports ESR)
movies.create_index([("genres", ASCENDING), ("year", ASCENDING)])
```

Prefer compound indexes for any query pattern that regularly filters on multiple fields together.

---

## Part IV: Index Maintenance

### Checking Index Usage with `$indexStats`

```python
for stat in movies.aggregate([{"$indexStats": {}}]):
    print(f"{stat['name']:40s}  ops={stat['accesses']['ops']}")
```

An index with `ops: 0` — or very low ops relative to others — is a candidate for removal.

### Hiding an Index Before Dropping

```python
# Step 1 — hide: invisible to query planner but still maintained on writes
db.command("collMod", "movies", index={"name": "idx_year", "hidden": True})

# Monitor for several days — check that query performance does not degrade
# Step 2 — drop safely
movies.drop_index("idx_year")

# To unhide (rollback):
db.command("collMod", "movies", index={"name": "idx_year", "hidden": False})
```

`hideIndex` is available from MongoDB 4.4. It gives you a free rollback window — the index can be re-exposed instantly if performance degrades after hiding.

### Write Amplification

Every write (insert, update, delete) must update all indexes on the collection:

```
1 document insert  →  1 write to collection
                    +  1 write per index
```

A collection with 5 indexes means 6 write operations per insert. Measure this with `$indexStats` over time, and drop indexes that are never used.

This is why the **Approximation Pattern** (reducing writes by ~90%) pairs well with heavily-indexed collections.

---

## Summary

| Index Type | Use When | Watch Out For |
|------------|----------|---------------|
| Single-field | Filter or sort on one field | Low cardinality may not help |
| Compound (ESR) | Multi-field filter + sort | Field order must follow ESR |
| Multikey | Querying inside array fields | One array field per compound index |
| Partial | Only a subset of documents qualify | Query must imply the filter expression |
| Sparse | Optional field in many documents | Prefer partial for more control |
| TTL | Auto-expiring documents | ~60-second cleanup granularity |
| Text | Full-text search on strings | One text index per collection |

---

## Additional Resources

- [PyMongo `create_index`](https://pymongo.readthedocs.io/en/stable/api/pymongo/collection.html#pymongo.collection.Collection.create_index)
- [MongoDB Index Types](https://www.mongodb.com/docs/manual/indexes/)
- [MongoDB ESR Rule](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-rule/)
- [MongoDB `explain()`](https://www.mongodb.com/docs/manual/reference/method/cursor.explain/)
- [MongoDB `$indexStats`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/indexStats/)

---

## Full Example

Copy-paste this script after setting `VM_IP_ADDRESS`. It runs every demo in this note in sequence and prints before/after `explain()` summaries. All indexes are dropped at the end — the collection is left in its original state.

```python
import pprint
from pymongo import MongoClient, ASCENDING, DESCENDING, TEXT

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db     = client["labs"]
movies = db["movies"]

def explain_summary(label, cursor):
    r = cursor.explain("executionStats")
    plan = r["queryPlanner"]["winningPlan"]
    stats = r["executionStats"]
    inner = plan.get("inputStage", {})
    print(f"\n{'─'*60}")
    print(f"  {label}")
    print(f"{'─'*60}")
    print(f"  top stage      : {plan['stage']}")
    if inner:
        print(f"  input stage    : {inner.get('stage','—')}  (index: {inner.get('indexName','—')})")
        print(f"  isMultiKey     : {inner.get('isMultiKey','—')}")
    print(f"  nReturned      : {stats['nReturned']}")
    print(f"  docsExamined   : {stats['totalDocsExamined']}")
    print(f"  keysExamined   : {stats['totalKeysExamined']}")
    print(f"  timeMillis     : {stats['executionTimeMillis']} ms")

# ── 1. Single-field ──────────────────────────────────────────────────────
explain_summary("BEFORE index — year == 1994",
    movies.find({"year": 1994}))

movies.create_index("year", name="idx_year")
explain_summary("AFTER  idx_year — year == 1994",
    movies.find({"year": 1994}))
movies.drop_index("idx_year")

# ── 2. ESR compound ──────────────────────────────────────────────────────
explain_summary("BEFORE compound — genres=Action, rating>7, sort rating desc",
    movies.find({"genres": "Action", "imdb.rating": {"$gt": 7}}).sort("imdb.rating", DESCENDING))

movies.create_index(
    [("genres", ASCENDING), ("imdb.rating", DESCENDING)],
    name="idx_genres_rating_esr"
)
explain_summary("AFTER  ESR compound",
    movies.find({"genres": "Action", "imdb.rating": {"$gt": 7}}).sort("imdb.rating", DESCENDING))
movies.drop_index("idx_genres_rating_esr")

# ── 3. Multikey ──────────────────────────────────────────────────────────
movies.create_index("cast", name="idx_cast")
explain_summary("Multikey — cast: Tom Hanks",
    movies.find({"cast": "Tom Hanks"}))
movies.drop_index("idx_cast")

# ── 4. Covered query ─────────────────────────────────────────────────────
movies.create_index([("genres", ASCENDING), ("title", ASCENDING)], name="idx_genres_title")
explain_summary("Covered query — genres=Action, project title only",
    movies.find({"genres": "Action"}, {"_id": 0, "title": 1}))
movies.drop_index("idx_genres_title")

# ── 5. Partial index ─────────────────────────────────────────────────────
movies.create_index(
    [("imdb.rating", DESCENDING)],
    partialFilterExpression={"imdb.rating": {"$gte": 8}},
    name="idx_rating_partial"
)
explain_summary("Partial index — imdb.rating >= 8",
    movies.find({"imdb.rating": {"$gte": 8}}))
movies.drop_index("idx_rating_partial")

# ── 6. Text index ────────────────────────────────────────────────────────
movies.create_index([("title", TEXT), ("plot", TEXT)], name="idx_text")
explain_summary("Text index — $search: space adventure",
    movies.find({"$text": {"$search": "space adventure"}}))
movies.drop_index("idx_text")

# ── 7. $indexStats — confirm only _id_ remains ───────────────────────────
print("\nIndexes after cleanup:")
for stat in movies.aggregate([{"$indexStats": {}}]):
    print(f"  {stat['name']}")
```
