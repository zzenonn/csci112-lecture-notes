# MongoDB Indexing

CSCI 112 / 212 - Contemporary Databases
Department of Computer Science

---

## Overview

This note covers MongoDB indexing from first principles: how queries are executed, how to diagnose performance with `explain()`, what index types exist and when to use each, how to design indexes that complement the data modeling patterns from Module 08, and how to manage indexes safely in production.

All examples use **PyMongo** running on your laptop, connecting to `mongod` on a VM. We reuse the `books` and `customers` collections built up across Modules 04–08a:

- **`books`** — the bookstore collection first created in [notes/8a](8a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md). By the time you reach this module it contains documents with fields like `title`, `price`, `product_type` (`"book"`, `"ebook"`, `"audiobook"`), `authors` (array), and a `rating` subdocument.
- **`customers`** — an e-commerce customer collection introduced in [notes/8c](8c%20-%20MongoDB%20Data%20Modeling%20Reference%20Pattern.md) (Reference / Extended Reference Pattern). Documents have `name`, `street`, `city`, `country`, and optionally `schema_version`.

If you are starting fresh, run the setup scripts in notes/8a and notes/8b first, or insert a few sample documents manually before running the index examples below.

---

## PyMongo Setup

```python
from pymongo import MongoClient, ASCENDING, DESCENDING, TEXT

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client    = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db        = client["bookstore"]
books     = db["books"]
customers = db["customers"]
```

---

## Part I: How MongoDB Executes Queries

### COLLSCAN vs IXSCAN

When MongoDB processes a query it picks a **query plan**. The two most important plan stages are:

- **COLLSCAN** — collection scan: reads every document in the collection and checks each against the filter. Cost is O(N) where N is collection size.
- **IXSCAN** — index scan: walks the B-tree index to find only the matching keys. Cost is O(log N + K) where K is the number of matching documents.

For large collections, the difference is dramatic. A collection with 10 million documents and no index on the filter field means 10 million document reads — even if only one document matches.

### `explain()` in PyMongo

```python
result = books.find({"product_type": "ebook"}).explain("executionStats")

import pprint
pprint.pprint(result)
```

The output has two main sections:

**`queryPlanner.winningPlan`** — the plan MongoDB chose:

```json
{
  "stage": "COLLSCAN",
  "filter": { "product_type": { "$eq": "ebook" } }
}
```

After adding an index it becomes:

```json
{
  "stage": "FETCH",
  "inputStage": {
    "stage": "IXSCAN",
    "keyPattern": { "product_type": 1 },
    "indexName": "product_type_1"
  }
}
```

**`executionStats`** — what actually happened at runtime:

```json
{
  "nReturned": 2,
  "totalDocsExamined": 10000,
  "totalKeysExamined": 2,
  "executionTimeMillis": 45
}
```

Key ratio to watch: `nReturned / totalDocsExamined`. A value close to 1.0 means the index is highly selective. A value close to 0.0 means most scanned documents were discarded — a sign of a missing or poorly designed index.

### Index Cardinality and Selectivity

**Cardinality** is the number of distinct values a field has across the collection.

- **High cardinality** (e.g., `email`, `_id`, `order_number`): each index entry points to very few documents. Queries are fast and selective.
- **Low cardinality** (e.g., `status: "active"/"inactive"`, `is_deleted: true/false`): many documents share each index entry. The optimizer may choose COLLSCAN over IXSCAN because the index saves little work.

Rule of thumb: an index is most useful when the matching documents are less than ~5% of the collection.

---

## Part II: Index Types

### Single-Field Index

```python
# Ascending index on title
books.create_index("title")

# Descending index on price (useful for "sort by most expensive")
books.create_index([("price", DESCENDING)])

# Unique index — rejects duplicate values
db["users"].create_index("email", unique=True)

# Name the index explicitly for easier management
books.create_index("title", name="idx_title_asc")
```

List all indexes on a collection:

```python
for idx in books.list_indexes():
    print(idx)
```

### Compound Index and the ESR Rule

A compound index covers multiple fields. MongoDB can use it for queries that filter or sort on a **prefix** of the index fields.

The **ESR rule** gives the optimal field order:

1. **E**quality fields first — fields tested with `==`
2. **S**ort fields second — fields used in `sort()`
3. **R**ange fields last — fields tested with `$gt`, `$lt`, `$in`, `$regex`

#### Why this order — the mechanical explanation

A B-tree index stores keys in sorted order. To understand ESR, picture walking through that sorted list:

**Equality first:** An equality predicate (`product_type == "ebook"`) jumps the B-tree walk directly to the matching key range. All qualifying documents are grouped together as a contiguous block. Putting equality fields first means the entire subsequent walk is confined to that small block — every step after it is already pre-filtered.

**Sort second:** Once the walk is confined to the equality block, the remaining keys inside that block are still in sorted B-tree order. If the sort field is next in the index, MongoDB can read the results directly in the correct order — no in-memory sort step needed. This is called a *"covered sort"* or *"index-provided sort"*.

**Range last:** A range predicate (`price > 5`) matches a contiguous stretch of keys, but that stretch can span the entire remaining index if it comes before the sort field. If range came before sort, the matching keys would be scattered across many sub-ranges — MongoDB could not guarantee they are in sort order, so it would have to collect all matches first and then sort them in memory. Putting range last avoids this: by the time MongoDB hits the range condition, equality has already narrowed the result set, and the sort order has already been established.

**Summary of what goes wrong when you violate ESR:**

| Violation | Consequence |
|-----------|-------------|
| Range before Sort | In-memory sort required — loses the index-provided sort benefit |
| Range before Equality | Scans a large portion of the index before narrowing by equality |
| Sort before Equality | Index walk covers too many keys before reaching the equality block |

```python
# Query pattern: product_type == "ebook", sorted by price, price > 5
# ESR order: product_type (E), price (S+R) — price serves as both sort and range
books.create_index([("product_type", ASCENDING), ("price", ASCENDING)])

# Confirm the plan uses IXSCAN with an index-provided sort (no SORT stage)
result = books.find(
    {"product_type": "ebook", "price": {"$gt": 5}},
    sort=[("price", ASCENDING)]
).explain("executionStats")

print(result["queryPlanner"]["winningPlan"]["inputStage"]["stage"])
# → IXSCAN  (no separate SORT stage above it)
```

#### Index prefix rule

A compound index `(a, b, c)` is usable for queries that filter or sort on a **left-prefix** of the fields:
- filter on `a` ✓
- filter on `a` and `b` ✓
- filter on `a`, `b`, and `c` ✓
- sort on `a` (with equality filter on `a`) ✓

It does **not** support queries that start with `b` or `c` alone — MongoDB cannot use the index efficiently if it must skip the leading field.

### Multikey Index

When you index an array field, MongoDB creates one index entry per array element — this is called a **multikey index**. It works automatically; you don't need a special option.

```python
# books.authors is an array: ["Alice Steinberg", "Bob Lim"]
books.create_index("authors")

# Both of these now use IXSCAN:
books.find({"authors": "Alice Steinberg"})
books.find({"authors": {"$in": ["Alice Steinberg", "Bob Lim"]}})
```

**Compound multikey constraint:** only one field in a compound index may be an array. Indexing two array fields would require indexing the cartesian product, which MongoDB rejects.

```python
# OK — authors is array, price is scalar
books.create_index([("authors", ASCENDING), ("price", ASCENDING)])

# ERROR — both are arrays: MongoDB raises OperationFailure
books.create_index([("authors", ASCENDING), ("tags", ASCENDING)])
```

### Unique Index

```python
db["users"].create_index("email", unique=True)

# Compound unique — the *combination* must be unique, not each field individually
books.create_index([("isbn", ASCENDING), ("edition", ASCENDING)], unique=True)
```

Inserting a document with a duplicate value raises `DuplicateKeyError`.

### Partial Index

Indexes only documents that match a filter expression. Smaller than a full index, lower write overhead:

```python
# Only index books that are currently in stock
books.create_index(
    "price",
    partialFilterExpression={"in_stock": True},
    name="idx_price_in_stock"
)

# This query uses the partial index:
books.find({"in_stock": True, "price": {"$lt": 20}})

# This query does NOT (in_stock: False docs are not indexed):
books.find({"in_stock": False, "price": {"$lt": 20}})
```

For a query to use a partial index, the query filter must **guarantee** that the partial filter expression is true — otherwise MongoDB falls back to COLLSCAN.

### Sparse Index

A sparse index only includes documents that have the indexed field. Documents without the field are skipped entirely.

```python
# Old customer documents have no schema_version field
# A sparse index skips them, keeping the index small
customers.create_index("schema_version", sparse=True)
```

Sparse indexes are useful for optional fields. However, a sparse index on a field that exists in most documents provides little benefit — use a partial index instead for more control.

### TTL Index

Automatically expires and deletes documents after a specified duration:

```python
from datetime import datetime, timezone

# Create a TTL index — expire after 3600 seconds (1 hour)
db["sessions"].create_index(
    "created_at",
    expireAfterSeconds=3600
)

# Insert a session — it will be deleted ~1 hour after created_at
db["sessions"].insert_one({
    "user_id": "u_001",
    "token": "abc123",
    "created_at": datetime.now(timezone.utc)
})
```

**Constraints:**
- The indexed field must be a BSON `Date` type (Python `datetime`). If it is missing or not a date, the document is never expired.
- TTL indexes are single-field only — no compound TTL indexes.
- A background thread checks for and removes expired documents every 60 seconds. Expiration is approximate, not exact.

**Common use cases:** session tokens, OTP codes, shopping cart items, event logs with a retention policy.

### Text Index

Full-text search across one or more string fields:

```python
# Create a text index on title and description
books.create_index([("title", TEXT), ("description", TEXT)])

# Search — returns documents containing both words (AND by default)
books.find({"$text": {"$search": "definitive guide"}})

# Phrase search
books.find({"$text": {"$search": "\"definitive guide\""}})

# Exclude a word
books.find({"$text": {"$search": "mongodb -javascript"}})

# Sort by relevance score
books.find(
    {"$text": {"$search": "mongodb guide"}},
    {"score": {"$meta": "textScore"}}
).sort([("score", {"$meta": "textScore"})])
```

**Constraints:** only one text index per collection. It can cover multiple fields.

### Wildcard Index

Indexes all fields in a document (or a subtree) — useful when field names are dynamic and unpredictable:

```python
# Index every field in the document
db["events"].create_index({"$**": 1})

# Index only fields nested under metadata
db["events"].create_index({"metadata.$**": 1})
```

Wildcard indexes are large and impose significant write overhead. Use them only when the schema is genuinely dynamic (e.g., user-defined attributes, polymorphic event payloads). For known field names, a compound index is always preferable.

---

## Part III: Index Design for Data Modeling

### Indexes for the Patterns from Module 08

**Inheritance Pattern** (`product_type` discriminator):

```python
# Single-field index — most reads filter by product type
books.create_index("product_type")

# Combined with sort — books by type sorted by price
books.create_index([("product_type", ASCENDING), ("price", ASCENDING)])
```

**Schema Versioning Pattern** (`schema_version` field):

```python
# Sparse — old documents without schema_version are skipped
customers.create_index("schema_version", sparse=True)
```

**Single Collection Pattern** (`docType` + `relatedTo`):

```python
# Both fields are equality — either order works, but relatedTo first
# because the common query is "all docs related to book X"
db["books_catalog"].create_index(
    [("relatedTo", ASCENDING), ("docType", ASCENDING)]
)

# Fetch the book and all its reviews in one query:
db["books_catalog"].find({"relatedTo": 34538756})

# Filter to just reviews:
db["books_catalog"].find({"relatedTo": 34538756, "docType": "review"})
```

**Computed Pattern** (sorting by cached average rating):

```python
# Descending — "top rated books" query
books.create_index([("rating.average_rating", DESCENDING)])
```

### Covered Queries

A covered query is served entirely from the index — MongoDB never reads the document itself. This is the fastest possible read path.

```python
# Index: (product_type, title)
books.create_index([("product_type", ASCENDING), ("title", ASCENDING)])

# Projection includes ONLY indexed fields (no _id = exclude it)
cursor = books.find(
    {"product_type": "ebook"},
    {"_id": 0, "title": 1}
)

# Check the plan — should show PROJECTION_COVERED with no FETCH stage
result = books.find(
    {"product_type": "ebook"},
    {"_id": 0, "title": 1}
).explain("executionStats")

stages = []
plan = result["queryPlanner"]["winningPlan"]
while plan:
    stages.append(plan["stage"])
    plan = plan.get("inputStage")
print(stages)
# → ['PROJECTION_COVERED', 'IXSCAN']  — no FETCH
```

For a query to be covered:
1. All filter fields must be in the index.
2. All projection fields must be in the index.
3. `_id` must be excluded from the projection (or included in the index).

### Index Intersection vs Compound Index

MongoDB can merge two single-field index results in memory (index intersection), but a purpose-built compound index is almost always faster:

```python
# Scenario: frequently query {product_type: "ebook", price: {$lt: 20}}

# Option A: two separate indexes (index intersection)
books.create_index("product_type")
books.create_index("price")
# MongoDB may or may not intersect — and the merge has CPU/memory cost

# Option B: one compound index (preferred)
books.create_index([("product_type", ASCENDING), ("price", ASCENDING)])
# Single B-tree walk, supports ESR sort, supports covered queries
```

Use index intersection only when you cannot predict query combinations in advance. Otherwise design compound indexes explicitly.

---

## Part IV: Index Maintenance

### Checking Index Usage with `$indexStats`

```python
for stat in books.aggregate([{"$indexStats": {}}]):
    print(f"{stat['name']:30s}  ops={stat['accesses']['ops']}")
```

An index with `ops: 0` (or very low ops relative to others) is a candidate for removal.

### Hiding an Index Before Dropping

```python
# Step 1 — hide: the index is invisible to the query planner but still maintained
db.command("collMod", "books", index={"name": "old_index_name", "hidden": True})

# Monitor for several days — if no performance regression, proceed
# Step 2 — drop
books.drop_index("old_index_name")

# To unhide (rollback):
db.command("collMod", "books", index={"name": "old_index_name", "hidden": False})
```

`hideIndex` is available from MongoDB 4.4 onward. It is the safest way to decommission an index in production — you get a free rollback window.

### Write Amplification Demo

```python
import time

# Drop all non-default indexes
for idx in books.list_indexes():
    if idx["name"] != "_id_":
        books.drop_index(idx["name"])

# Measure inserts with NO extra indexes
start = time.perf_counter()
books.insert_many([{"title": f"book_{i}", "price": i} for i in range(10000)])
t_no_index = time.perf_counter() - start
books.delete_many({})

# Add 5 indexes
books.create_index("title")
books.create_index("price")
books.create_index([("title", ASCENDING), ("price", ASCENDING)])
books.create_index("product_type")
books.create_index([("product_type", ASCENDING), ("price", ASCENDING)])

# Measure inserts WITH 5 indexes
start = time.perf_counter()
books.insert_many([{"title": f"book_{i}", "price": i, "product_type": "book"} for i in range(10000)])
t_with_index = time.perf_counter() - start

print(f"No extra indexes:  {t_no_index:.2f}s")
print(f"With 5 indexes:    {t_with_index:.2f}s")
print(f"Write overhead:    {t_with_index / t_no_index:.1f}x")
```

The overhead grows with the number of indexes and the size of the indexed fields. This is why the Approximation Pattern (reducing writes by ~90%) pairs well with heavily-indexed collections.

### Building Indexes on Large Collections

Since MongoDB 4.2, all index builds are non-blocking by default. Monitor a build in progress:

```python
import pprint

# Check active index builds
ops = db.command("currentOp", {"$all": True})
for op in ops["inprog"]:
    if op.get("command", {}).get("createIndexes"):
        pprint.pprint(op)
```

On a replica set, the primary builds first. Secondaries replicate the build operation and build the index independently, so they may lag briefly during the build. The index is not available for queries until the build completes on each node.

---

## Summary

| Index Type | Use When | Watch Out For |
|------------|----------|---------------|
| Single-field | Filter or sort on one field | Low cardinality may not help |
| Compound (ESR) | Multi-field filter + sort | Field order must follow ESR |
| Multikey | Querying inside array fields | Only one array field per compound index |
| Unique | Enforce uniqueness constraint | DuplicateKeyError on insert |
| Partial | Only a subset of documents qualify | Query must match filter expression |
| Sparse | Field is optional in many docs | Use partial for more control |
| TTL | Auto-expiring documents | ~60-second cleanup granularity |
| Text | Full-text search on strings | One text index per collection |
| Wildcard | Truly dynamic field names | Large, expensive writes |

---

## Additional Resources

- [PyMongo `create_index` documentation](https://pymongo.readthedocs.io/en/stable/api/pymongo/collection.html#pymongo.collection.Collection.create_index)
- [MongoDB Index Types](https://www.mongodb.com/docs/manual/indexes/)
- [MongoDB ESR Rule](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-rule/)
- [MongoDB `explain()`](https://www.mongodb.com/docs/manual/reference/method/cursor.explain/)
- [MongoDB `$indexStats`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/indexStats/)

---

## Full Example

The script below exercises every concept in this note end-to-end. Run it against a populated `books` collection (see [notes/8a](8a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md) for setup).

```python
import time
import pprint
from pymongo import MongoClient, ASCENDING, DESCENDING, TEXT

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db     = client["bookstore"]
books  = db["books"]

# ---------------------------------------------------------------------------
# 1. explain() — before index
# ---------------------------------------------------------------------------
result = books.find({"product_type": "ebook"}).explain("executionStats")
print("Before index — stage:", result["queryPlanner"]["winningPlan"]["stage"])
print("  docsExamined:", result["executionStats"]["totalDocsExamined"])
print("  nReturned:   ", result["executionStats"]["nReturned"])

# ---------------------------------------------------------------------------
# 2. Create a single-field index and re-explain
# ---------------------------------------------------------------------------
books.create_index("product_type", name="idx_product_type")

result = books.find({"product_type": "ebook"}).explain("executionStats")
plan = result["queryPlanner"]["winningPlan"]
print("\nAfter index — stage:", plan.get("stage"), "→", plan.get("inputStage", {}).get("stage"))
print("  docsExamined:", result["executionStats"]["totalDocsExamined"])
print("  nReturned:   ", result["executionStats"]["nReturned"])

# ---------------------------------------------------------------------------
# 3. Compound index (ESR) for filter + sort
# ---------------------------------------------------------------------------
books.create_index(
    [("product_type", ASCENDING), ("price", ASCENDING)],
    name="idx_type_price_esr"
)

result = books.find(
    {"product_type": "ebook", "price": {"$gt": 5}},
    sort=[("price", ASCENDING)]
).explain("executionStats")

plan = result["queryPlanner"]["winningPlan"]
inner = plan.get("inputStage", {})
print("\nESR compound — stages:", plan["stage"], "→", inner.get("stage"))

# ---------------------------------------------------------------------------
# 4. Covered query
# ---------------------------------------------------------------------------
books.create_index(
    [("product_type", ASCENDING), ("title", ASCENDING)],
    name="idx_type_title_covered"
)

result = books.find(
    {"product_type": "ebook"},
    {"_id": 0, "title": 1}
).explain("executionStats")

stages = []
node = result["queryPlanner"]["winningPlan"]
while node:
    stages.append(node["stage"])
    node = node.get("inputStage")
print("\nCovered query stages:", stages)
# Expected: ['PROJECTION_COVERED', 'IXSCAN']

# ---------------------------------------------------------------------------
# 5. TTL index (on sessions collection)
# ---------------------------------------------------------------------------
from datetime import datetime, timezone

db["sessions"].drop()
db["sessions"].create_index("created_at", expireAfterSeconds=5)
db["sessions"].insert_one({"token": "xyz", "created_at": datetime.now(timezone.utc)})
print("\nSession inserted. Will expire in ~5s (cleanup runs every 60s).")

# ---------------------------------------------------------------------------
# 6. $indexStats — check usage
# ---------------------------------------------------------------------------
print("\nIndex usage stats:")
for stat in books.aggregate([{"$indexStats": {}}]):
    print(f"  {stat['name']:40s}  ops={stat['accesses']['ops']}")

# ---------------------------------------------------------------------------
# 7. Write amplification demo
# ---------------------------------------------------------------------------
# Drop extra indexes
for idx in books.list_indexes():
    if idx["name"] not in ("_id_",):
        books.drop_index(idx["name"])

start = time.perf_counter()
books.insert_many([{"title": f"t_{i}", "price": i % 100} for i in range(5000)])
t_bare = time.perf_counter() - start
books.delete_many({"title": {"$regex": "^t_"}})

# Add 4 indexes
books.create_index("title")
books.create_index("price")
books.create_index([("title", ASCENDING), ("price", ASCENDING)])
books.create_index("product_type")

start = time.perf_counter()
books.insert_many([{"title": f"t_{i}", "price": i % 100, "product_type": "book"} for i in range(5000)])
t_indexed = time.perf_counter() - start
books.delete_many({"title": {"$regex": "^t_"}})

print(f"\nWrite amplification demo:")
print(f"  No extra indexes:  {t_bare:.2f}s")
print(f"  With 4 indexes:    {t_indexed:.2f}s")
print(f"  Overhead factor:   {t_indexed / t_bare:.1f}x")
```
