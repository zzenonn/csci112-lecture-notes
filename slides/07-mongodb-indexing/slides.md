---
theme: discs
title: "MongoDB Indexing"
highlighter: shiki
layout: cover
codeCopy: true
---

MongoDB Indexing

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Explain how MongoDB executes queries and the cost of a collection scan
2. Read and interpret `explain()` output to diagnose query performance
3. Choose the right index type for a given access pattern
4. Apply the **ESR rule** to design effective compound indexes
5. Understand write amplification and manage indexes safely in production

---
layout: section
---

# Part I: How MongoDB Executes Queries

---

# COLLSCAN vs IXSCAN

When you run a query, MongoDB chooses a **query plan**:

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:1rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#991b1b;">COLLSCAN</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      Scans every document in the collection.<br>
      Cost grows linearly with collection size.<br>
      Unavoidable when no suitable index exists.
    </div>
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#166534;">IXSCAN</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      Walks the index B-tree to find matching keys.<br>
      Cost proportional to matching documents, not collection size.<br>
      Requires a suitable index on the queried fields.
    </div>
  </div>
</div>

<div style="margin-top:1rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.6rem 1rem; font-size:0.88rem;">
  The <code>labs.movies</code> collection has <strong>44,488 documents</strong>. With no index, every query reads all 44,488 — even if only one document matches.
</div>

---

# Reading `explain()` Output

```python
result = movies.find({"year": 1994}).explain("executionStats")
```

Key fields to check:

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.8rem; margin-top:1rem; font-size:0.84rem;">
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.4rem;"><code>winningPlan.stage</code></div>
    <code>COLLSCAN</code> = no index used<br>
    <code>IXSCAN</code> = index used<br>
    <code>FETCH</code> = fetched full doc after index<br>
    <code>SORT</code> = in-memory sort (expensive)
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.4rem;"><code>executionStats</code></div>
    <code>nReturned</code> — docs returned<br>
    <code>totalDocsExamined</code> — docs scanned<br>
    <code>totalKeysExamined</code> — index entries scanned<br>
    <code>executionTimeMillis</code> — elapsed ms
  </div>
</div>

<div style="margin-top:1rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.6rem 1rem; font-size:0.87rem;">
  A healthy query has <code>nReturned ≈ totalDocsExamined</code>. If <code>totalDocsExamined</code> is much larger, the index is missing or poorly selective.
</div>

---

# COLLSCAN → IXSCAN Demo

```python
movies.find({"year": 1994})   # run explain() before creating the index
movies.create_index("year")   # create the index
movies.find({"year": 1994})   # run explain() again — observe the difference
```

Before and after creating `{"year": 1}`:

| Metric | No index (COLLSCAN) | With index (IXSCAN) |
|--------|--------------------:|--------------------:|
| `nReturned` | 550 | 550 |
| `totalDocsExamined` | **44,488** | **550** |
| `executionTimeMillis` | 17 ms | 1 ms |

<div style="margin-top:0.5rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.86rem;">
  Try this in your script — call <code>explain("executionStats")</code> before and after, and check <code>winningPlan.stage</code> and <code>executionStats</code> yourself.
</div>

---

# Index Cardinality and Selectivity

**Cardinality** — how many distinct values a field has.

**Selectivity** — what fraction of documents a query eliminates.

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.8rem; margin-top:1rem; font-size:0.85rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">High cardinality — good index</div>
    <code>year</code> (~130 distinct values in movies)<br><br>
    <code>year == 1994</code> matches 550 of 44,488 — only 1.2%. Index eliminates 98.8% instantly.
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">Low cardinality — poor index</div>
    <code>type</code> (3 values: movie / series / N/A)<br><br>
    <code>type == "movie"</code> matches 44,075 of 44,488 — 99%. Index saves almost nothing.
  </div>
</div>

<div style="margin-top:0.8rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.86rem;">
  Rule of thumb: an index is most beneficial when the query matches fewer than ~5% of the collection.
</div>

---
layout: section
---

# Part II: Index Types

---

# Single-Field Index

The simplest index — one field, ascending or descending:

```python
# Ascending on year
movies.create_index("year", name="idx_year")

# Descending on imdb.rating (dot notation for nested fields)
movies.create_index([("imdb.rating", -1)], name="idx_rating_desc")

# Unique constraint — typically on users, not movies
db["users"].create_index("email", unique=True)
```

<div style="margin-top:0.8rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.6rem 1rem; font-size:0.87rem;">
  MongoDB automatically creates a unique index on <code>_id</code>. You never need to create one manually for the primary key.
</div>

---

# Compound Index and the ESR Rule

A compound index covers multiple fields. **Field order matters.**

The **ESR rule** gives the optimal order:

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.7rem; margin-top:0.8rem; font-size:0.84rem;">
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.8rem; background:#f0f9ff; text-align:center;">
    <div style="font-size:1.4rem; font-weight:700; color:#00b0f0;">E</div>
    <div style="font-weight:700;">Equality</div>
    <div style="color:#444; margin-top:0.3rem;">Fields tested with <code>==</code> first</div>
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4; text-align:center;">
    <div style="font-size:1.4rem; font-weight:700; color:#16a34a;">S</div>
    <div style="font-weight:700;">Sort</div>
    <div style="color:#444; margin-top:0.3rem;">Fields used in <code>sort()</code> second</div>
  </div>
  <div style="border:1.5px solid #ca8a04; border-radius:6px; padding:0.8rem; background:#fef9c3; text-align:center;">
    <div style="font-size:1.4rem; font-weight:700; color:#ca8a04;">R</div>
    <div style="font-weight:700;">Range</div>
    <div style="color:#444; margin-top:0.3rem;">Fields tested with <code>$gt</code>/<code>$lt</code> last</div>
  </div>
</div>

```python
# genres → E (equality on array), imdb.rating → S + R (sort + range)
movies.create_index([("genres", 1), ("imdb.rating", -1)])
```

---

# ESR Demo — SORT Stage Disappears

Query: all Action movies rated above 7, sorted by rating descending.

```python
movies.find({"genres": "Action", "imdb.rating": {"$gt": 7}}).sort("imdb.rating", -1)
```

| Metric | No index | ESR compound index |
|--------|----------:|-------------------:|
| `winningPlan.stage` | SORT + COLLSCAN | FETCH ← IXSCAN |
| `nReturned` | 1,071 | 1,071 |
| `totalDocsExamined` | **44,488** | **1,071** |
| `executionTimeMillis` | 24 ms | 2 ms |

<div style="margin-top:0.5rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.5rem 0.9rem; font-size:0.86rem;">
  The <code>SORT</code> stage disappears — results come out of the index already in order. ESR field order lets MongoDB read sorted results in one B-tree walk.
</div>

---

# Multikey Index

When a field is an **array**, MongoDB creates one index entry per array element:

```python
# cast is an array: ["Tom Hanks", "Robin Wright", ...]
movies.create_index("cast", name="idx_cast")

# Now this uses IXSCAN (explain shows isMultiKey: true):
movies.find({"cast": "Tom Hanks"})
# nReturned: 50, totalDocsExamined: 50, executionTimeMillis: 1
```

<div style="margin-top:0.8rem; display:grid; grid-template-columns:1fr 1fr; gap:0.8rem; font-size:0.85rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">Works</div>
    One array field per compound index.
    <code>["cast", "year"]</code> is fine — <code>cast</code> is the array.
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">Rejected</div>
    Two array fields in one compound index.
    <code>["cast", "genres"]</code> → <code>OperationFailure: cannot index parallel arrays</code>
  </div>
</div>

---

# Partial and Sparse Indexes

**Partial index** — only index documents matching a filter expression:

```python
# Only index highly-rated movies — 2,284 of 44,488 qualify (~5%)
movies.create_index(
    [("imdb.rating", -1)],
    partialFilterExpression={"imdb.rating": {"$gte": 8}},
    name="idx_rating_partial"
)
```

**Sparse index** — only index documents where the field exists:

```python
# Documents without schema_version are not indexed
db["customers"].create_index("schema_version", sparse=True)
```

<div style="margin-top:0.8rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.6rem 1rem; font-size:0.87rem;">
  Both reduce index size and write overhead. Partial is more expressive — prefer it over sparse when you know the condition.
</div>

---

# TTL Index

Automatically deletes documents after a time-to-live period:

```python
# Delete session documents 1 hour after created_at
db["sessions"].create_index(
    "created_at",
    expireAfterSeconds=3600
)
```

The `tomatoes.lastUpdated` field in `labs.movies` is a BSON Date — it could serve as a TTL field in a rolling-update scenario.

**Requirements:** field must be a BSON date; single-field only (not compound); background thread runs every ~60 seconds.

<div style="margin-top:0.6rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.5rem 0.9rem; font-size:0.87rem;">
  Common use cases: sessions, cache documents, event logs, OTP tokens.
</div>

---

# Text and Wildcard Indexes

**Text index** — full-text search across string fields:

```python
movies.create_index([("title", "text"), ("plot", "text")])

# Search for movies about space adventure
movies.find({"$text": {"$search": "space adventure"}})
# Returns 921 movies — plan stage: TEXT_MATCH
```

**Wildcard index** — index all fields (or a subtree) for flexible/dynamic schemas:

```python
# Index every field in the document
db["events"].create_index({"$**": 1})

# Index only fields under metadata
db["events"].create_index({"metadata.$**": 1})
```

<div style="margin-top:0.8rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.83rem;">
  Wildcard indexes are large and write-expensive. Use them only when field names are genuinely unpredictable at schema design time.
</div>

---
layout: section
---

# Part III: Index Design for Data Modeling

---

# Indexes and the Patterns We Learned

Every data modeling pattern we covered depends on the right index:

<div style="overflow-x:auto; margin-top:1rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.85rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.5rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Pattern</th>
      <th style="padding:0.5rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Key field(s)</th>
      <th style="padding:0.5rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Index needed</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><strong>Inheritance</strong></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>product_type</code></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;">Single-field (low-cardinality, but filters most reads)</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><strong>Schema Versioning</strong></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>schema_version</code></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;">Sparse — old docs without the field are skipped</td>
    </tr>
    <tr>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><strong>Single Collection</strong></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>relatedTo</code>, <code>docType</code></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;">Compound ESR: <code>relatedTo</code> (E) + <code>docType</code> (E)</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><strong>Computed</strong></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>rating.average_rating</code></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;">Single-field descending (sort by rating)</td>
    </tr>
  </tbody>
</table>
</div>

---

# Covered Queries

A **covered query** is satisfied entirely from the index — MongoDB never fetches the document:

```python
movies.create_index([("genres", 1), ("title", 1)], name="idx_genres_title")

# All filter + projection fields in index; _id excluded → covered
movies.find({"genres": "Action"}, {"_id": 0, "title": 1}).explain("executionStats")
```

| Metric | Result |
|--------|-------:|
| `winningPlan.stage` | `PROJECTION_COVERED` |
| `nReturned` | 5,655 |
| `totalDocsExamined` | **0** |
| `executionTimeMillis` | 3 ms |

<div style="margin-top:0.5rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.5rem 0.9rem; font-size:0.86rem;">
  <code>totalDocsExamined: 0</code> — no documents read at all. The fastest possible read.
</div>

---

# Index Intersection vs Compound Index

MongoDB can sometimes combine two single-field indexes automatically (**index intersection**), but a compound index is almost always faster:

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem; font-size:0.85rem;">
  <div style="border:1.5px solid #94a3b8; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.3rem;">Index intersection</div>
    Two separate indexes on <code>genres</code> and <code>year</code>.<br>
    MongoDB intersects the result sets in memory.<br>
    Extra memory + CPU overhead.
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">Compound index</div>
    One index on <code>(genres, year)</code>.<br>
    Single B-tree walk, no merging.<br>
    Faster, smaller, supports sort.
  </div>
</div>

<div style="margin-top:0.8rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.83rem;">
  Prefer a compound index for any query that filters on multiple fields together regularly.
</div>

---
layout: section
---

# Part IV: Index Maintenance

---

# Write Amplification

Every write (insert, update, delete) must update **all indexes** on the collection:

```
1 document insert  →  1 write to collection
                    +  1 write per index
```

A collection with 5 indexes means **6 write operations** per insert.

<div style="margin-top:1rem; display:grid; grid-template-columns:1fr 1fr; gap:0.8rem; font-size:0.85rem;">
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">Cost</div>
    Higher write latency.<br>
    More disk I/O and storage.<br>
    Can become a bottleneck on write-heavy workloads (e.g., the Approximation Pattern).
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">Mitigation</div>
    Only create indexes that are actually used.<br>
    Use <code>hideIndex</code> to test before dropping.<br>
    Monitor with <code>$indexStats</code>.
  </div>
</div>

---

# Dropping Indexes Safely

Never drop an index without checking if it's in use:

```python
# 1. Check usage stats — ops: 0 means no queries used it
for stat in movies.aggregate([{"$indexStats": {}}]):
    print(stat["name"], "—", stat["accesses"]["ops"], "ops")

# 2. Hide first — hides from query planner but keeps the index
db.command("collMod", "movies", index={"name": "idx_year", "hidden": True})

# 3. Monitor for a few days — if no performance regression, drop
movies.drop_index("idx_year")
```

<div style="margin-top:0.8rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.6rem 1rem; font-size:0.87rem;">
  <code>hideIndex</code> was introduced in MongoDB 4.4. It lets you simulate dropping an index without actually removing it — safe rollback if needed.
</div>

---

# Building Indexes in Production

```python
# Non-blocking index build (default since MongoDB 4.2)
movies.create_index([("genres", 1), ("imdb.rating", -1)])
```

<div style="margin-top:0.8rem; font-size:0.87rem; line-height:1.7;">
  Since MongoDB 4.2, all index builds are <strong>non-blocking by default</strong> — they use an optimized hybrid build that holds locks only briefly at the start and end. You no longer need a separate <code>background: True</code> flag.
</div>

<div style="margin-top:0.8rem; display:grid; grid-template-columns:1fr 1fr; gap:0.8rem; font-size:0.85rem;">
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.7rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.2rem;">Watch for</div>
    Large collections can take minutes or hours to build. Monitor with <code>currentOp()</code>.
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.7rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.2rem;">On replica sets</div>
    Build runs on primary, then replicates. Secondaries build sequentially to avoid lag.
  </div>
</div>

---

# Summary

<div style="overflow-x:auto; margin-top:1rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.84rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.5rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Index Type</th>
      <th style="padding:0.5rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Use When</th>
      <th style="padding:0.5rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Watch Out For</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;"><strong>Single-field</strong></td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Filtering or sorting on one field</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Low cardinality may not help</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;"><strong>Compound (ESR)</strong></td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Multi-field filter + sort queries</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Field order must follow ESR</td>
    </tr>
    <tr>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;"><strong>Multikey</strong></td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Querying inside array fields</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Only one array field per compound</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;"><strong>Partial / Sparse</strong></td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Subset of documents qualify</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Query must match filter expression</td>
    </tr>
    <tr>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;"><strong>TTL</strong></td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Auto-expiring documents</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">60-second cleanup granularity</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;"><strong>Text</strong></td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Full-text search on string fields</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">One text index per collection</td>
    </tr>
  </tbody>
</table>
</div>

---

# Further Reading

See the matching notes for full PyMongo examples, `explain()` output, and index design walkthroughs:

- [notes/07 — MongoDB Indexing](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/07%20-%20MongoDB%20Indexing.md)
- [Course notes overview](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/README.md)
