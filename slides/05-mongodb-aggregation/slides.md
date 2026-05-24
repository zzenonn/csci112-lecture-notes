---
theme: discs
title: "MongoDB Aggregation"
highlighter: shiki
layout: cover
codeCopy: true
---

MongoDB Aggregation

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Understand the **aggregation pipeline** architecture
2. Apply common pipeline stages: `$match`, `$project`, `$group`, `$sort`, `$limit`, `$unwind`, `$addFields`, `$out`
3. Use **accumulator expressions**: `$sum`, `$avg`, `$max`, `$min`, `$push`
4. Chain multiple stages into an **end-to-end analytic query**
5. Understand **memory limits** and `allowDiskUse`

---
layout: section
---

# Data Pipelines

---

# Data Pipelines

Data flows through a series of transformation stages — each stage receives documents, transforms them, and passes results forward.

<div style="display:flex; align-items:center; justify-content:center; gap:0.6rem; margin-top:2rem; flex-wrap:wrap; font-size:0.82rem;">
  <div style="background:#f0f9ff; border:1.5px solid #00b0f0; border-radius:6px; padding:0.5rem 0.9rem; text-align:center;">
    <div style="font-weight:700;">Raw Data</div>
    <div style="color:#666; font-size:0.75rem;">customerId, region, items</div>
  </div>
  <div style="color:#94a3b8; font-size:1.1rem;">→</div>
  <div style="background:#fef9c3; border:1.5px solid #ca8a04; border-radius:6px; padding:0.5rem 0.9rem; text-align:center;">
    <div style="font-weight:700;">Filter</div>
    <div style="color:#666; font-size:0.75rem;">remove irrelevant rows</div>
  </div>
  <div style="color:#94a3b8; font-size:1.1rem;">→</div>
  <div style="background:#fef9c3; border:1.5px solid #ca8a04; border-radius:6px; padding:0.5rem 0.9rem; text-align:center;">
    <div style="font-weight:700;">Flatten</div>
    <div style="color:#666; font-size:0.75rem;">arrays → rows</div>
  </div>
  <div style="color:#94a3b8; font-size:1.1rem;">→</div>
  <div style="background:#fef9c3; border:1.5px solid #ca8a04; border-radius:6px; padding:0.5rem 0.9rem; text-align:center;">
    <div style="font-weight:700;">Compute</div>
    <div style="color:#666; font-size:0.75rem;">qty × price</div>
  </div>
  <div style="color:#94a3b8; font-size:1.1rem;">→</div>
  <div style="background:#fef9c3; border:1.5px solid #ca8a04; border-radius:6px; padding:0.5rem 0.9rem; text-align:center;">
    <div style="font-weight:700;">Aggregate</div>
    <div style="color:#666; font-size:0.75rem;">sum by customer</div>
  </div>
  <div style="color:#94a3b8; font-size:1.1rem;">→</div>
  <div style="background:#bbf7d0; border:1.5px solid #16a34a; border-radius:6px; padding:0.5rem 0.9rem; text-align:center;">
    <div style="font-weight:700;">Output</div>
    <div style="color:#666; font-size:0.75rem;">customerId, orderTotal</div>
  </div>
</div>

<div style="margin-top:1.5rem; font-size:0.9rem; color:#555;">
MongoDB's aggregation pipeline is an implementation of this concept — documents flow through stages, each transforming the data.
</div>

---
layout: section
---

# The Aggregation Pipeline

---

# Pipeline Syntax

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db = client["movies"]
movies_collection = db["movies_small"]

results = movies_collection.aggregate([
    { "$match":   { ... } },
    { "$group":   { ... } },
    { "$sort":    { ... } },
    { "$limit":   5 }
])

for doc in results:
    print(doc)
```

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  Each stage receives the output of the previous stage. Order matters.
</div>

---

# Sample Data Setup

Before running the examples, insert the sample dataset from the notes.

<div style="margin-top:1.5rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:1rem 1.25rem; border-radius:0 6px 6px 0; font-size:0.95rem;">
  Run the <strong>Sample Data Setup</strong> script in <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/05%20-%20MongoDB%20Aggregation.md#sample-data-setup">notes/05 — MongoDB Aggregation</a> before proceeding.
</div>

---
layout: section
---

# Pipeline Stages

---

# `$match` and `$count`

Filter documents — `$match` works like `.find()` but inside a pipeline.

```python
# Movies with IMDB rating ≥ 7.0
movies_collection.aggregate([
    { "$match": { "imdb_rating": { "$gte": 7.0 } } }
])

# Count matching documents
movies_collection.aggregate([
    { "$match": { "imdb_rating": { "$gte": 7.0 } } },
    { "$count": "high_rated_count" }
])
```

<div style="margin-top:1rem; font-size:0.9rem; color:#555;">
  Place <code>$match</code> early in the pipeline to reduce the number of documents flowing into later stages.
</div>

---

# `$project`

Shapes documents — include, exclude, or compute fields.

```python
# Show only title; rename budget field
movies_collection.aggregate([
    { "$project": {
        "_id": 0,
        "title": 1,
        "budget_in_millions": "$budget"
    }}
])
```

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  <code>"budget_in_millions": "$budget"</code> — the <code>$</code> prefix references the field value from the incoming document.
</div>

---

# `$group`

<div style="display:flex; gap:2rem; align-items:flex-start; margin-top:0.5rem;">
<div style="flex:1;">

Groups documents by a key and applies accumulator expressions.

```python
movies_collection.aggregate([
    { "$group": {
        "_id":        "$year",
        "num_films":  { "$sum": 1 },
        "avg_rating": { "$avg": "$imdb_rating" },
        "max_budget": { "$max": "$budget" }
    }}
])
```

</div>
<div style="flex:1;">

<table style="font-size:0.85rem; border-collapse:collapse; width:100%;">
  <thead><tr style="border-bottom:2px solid #e2e8f0;">
    <th style="text-align:left; padding:0.3rem 0.5rem;">Accumulator</th>
    <th style="text-align:left; padding:0.3rem 0.5rem;">Description</th>
  </tr></thead>
  <tbody style="line-height:1.6;">
    <tr><td style="padding:0.2rem 0.5rem;"><code>$sum</code></td><td style="padding:0.2rem 0.5rem;">Total / count</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$avg</code></td><td style="padding:0.2rem 0.5rem;">Average</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$min</code> / <code>$max</code></td><td style="padding:0.2rem 0.5rem;">Min / max value</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$push</code></td><td style="padding:0.2rem 0.5rem;">Collect into array</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$first</code> / <code>$last</code></td><td style="padding:0.2rem 0.5rem;">First / last in group</td></tr>
  </tbody>
</table>

</div>
</div>

---

# `$sort` and `$limit`

```python
# Sort descending by average rating, take top 3
movies_collection.aggregate([
    { "$group": {
        "_id": "$year",
        "avg_rating": { "$avg": "$imdb_rating" }
    }},
    { "$sort":  { "avg_rating": -1 } },
    { "$limit": 3 }
])
```

`$sort`: `1` = ascending, `-1` = descending

---

# `$unwind`

Deconstructs an array field — one output document per array element.

```python
# Before: { "title": "Batman", "category": ["action", "adventure"] }
# After $unwind: two documents, one per category value

movies_collection.aggregate([
    { "$unwind": "$category" },
    { "$group": {
        "_id": "$category",
        "count": { "$sum": 1 }
    }}
])
```

<div style="margin-top:0.75rem; font-size:0.85rem; color:#666;">
  Without <code>$unwind</code>, grouping by <code>$category</code> would treat the whole array as one key.
</div>

---

# `$addFields`

Adds new fields (or overwrites existing ones) without removing the rest of the document — unlike `$project`.

```python
movies_collection.aggregate([
    { "$addFields": {
        "is_blockbuster": { "$gte": ["$budget", 100] }
    }}
])
```

```python
# Combine $addFields + $project
movies_collection.aggregate([
    { "$addFields": { "budget_k": { "$multiply": ["$budget", 1000] } } },
    { "$project":   { "_id": 0, "title": 1, "budget_k": 1 } }
])
```

---

# `$out`

Writes the aggregation result to a collection — creates it if it doesn't exist, replaces it if it does.

```python
movies_collection.aggregate([
    { "$unwind": "$category" },
    { "$group":  { "_id": "$category", "count": { "$sum": 1 } } },
    { "$sort":   { "count": -1 } },
    { "$out":    { "db": "movies", "coll": "category_counts" } }
])
```

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  <code>$out</code> must be the <strong>last</strong> stage in the pipeline. The 16 MB per-document limit still applies.
</div>

---
layout: section
---

# End-to-End Examples

---

# Count and Filter

```python
# Count action movies
movies_collection.aggregate([
    { "$match": { "category": "action" } },
    { "$count": "num_action_movies" }
])

# High-budget action movies only
movies_collection.aggregate([
    { "$match": { "category": "action", "budget": { "$gt": 50 } } },
    { "$sort":  { "imdb_rating": -1 } }
])
```

---

# Average Rating per Category

```python
movies_collection.aggregate([
    { "$unwind": "$category" },
    { "$group": {
        "_id": "$category",
        "avg_rating": { "$avg": "$imdb_rating" }
    }},
    { "$project": {
        "_id": 0,
        "category":   "$_id",
        "avg_rating": 1
    }},
    { "$sort": { "avg_rating": -1 } }
])
```

<div style="margin-top:0.5rem; font-size:0.85rem; color:#666;">
  Full sample output in <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/05%20-%20MongoDB%20Aggregation.md">notes/05 — MongoDB Aggregation</a>
</div>

---

# Highest Rated Expensive Movie

```python
movies_collection.aggregate([
    { "$match": { "budget": { "$gt": 25 } } },
    { "$sort":  { "imdb_rating": -1 } },
    { "$limit": 1 },
    { "$project": { "_id": 0, "title": 1, "imdb_rating": 1, "budget": 1 } }
])
```

---

# Memory Limits

<div style="display:flex; gap:2rem; margin-top:1rem;">
<div style="flex:1;">

Each pipeline stage is limited to **100 MB** of RAM by default.

```python
# Allow stages to spill to disk
movies_collection.aggregate(
    [ ... ],
    allowDiskUse=True
)
```

</div>
<div style="flex:1; font-size:0.88rem; color:#444; line-height:1.8;">

- Each result document ≤ **16 MB**
- MongoDB 6.0+: `allowDiskUseByDefault` can be set server-side
- Use `$match` early to keep working set small

</div>
</div>

---
layout: end
---

# Thank You
