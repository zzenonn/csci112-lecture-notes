# MongoDB Aggregation Pipeline
CSCI 112 / 212 - Contemporary Databases

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Data Pipelines and Data Lineage](#data-pipelines-and-data-lineage)
- [Aggregation Pipeline Basics](#aggregation-pipeline-basics)
- [Sample Data Setup](#sample-data-setup)
- [Common Aggregation Stages](#common-aggregation-stages)
  - [$match](#match)
  - [$project](#project)
  - [$group](#group)
  - [$sort and $limit](#sort-and-limit)
  - [$unwind](#unwind)
  - [$addFields](#addfields)
  - [$out](#out)
- [Accumulators](#accumulators)
- [End-to-End Examples](#end-to-end-examples)
  - [Count action movies](#count-action-movies)
  - [Highest rated expensive movie](#highest-rated-expensive-movie)
  - [Average IMDB rating per category](#average-imdb-rating-per-category)
  - [Data cleaning example](#data-cleaning-example)
- [Memory Limits and allowDiskUse](#memory-limits-and-allowdiskuse)
- [Practice Problems](#practice-problems)
- [Sample Outputs](#sample-outputs)
- [Additional Resources](#additional-resources)

---

## Overview

The aggregation pipeline is MongoDB's framework for data transformation and analysis. Documents flow through a sequence of stages — each stage receives the output of the previous one and passes transformed results forward.

All examples in this module use **PyMongo** running on your host machine. The query operator syntax (`$match`, `$group`, etc.) is identical to mongosh — you write them as Python dicts.

---

## Objectives

By the end of this module, you should be able to:

- Describe the purpose and syntax of common aggregation stage operators in MongoDB
- Understand how documents flow through the aggregation pipeline and how each stage transforms the data
- Apply the aggregation framework to perform data analysis and transformation tasks using PyMongo

---

## Data Pipelines and Data Lineage

Before diving into MongoDB's aggregation pipeline, it's important to understand the broader concept of data pipelines. A data pipeline is a series of processing steps where data flows from one stage to the next, with each stage transforming it in some way.

![Data Pipeline Example](../images/data-pipeline-example.svg)

A typical pipeline:
- **Raw Input Data** — unprocessed data (e.g., `customerId`, `region`, `items`)
- **Transformations** — filter, flatten arrays, compute values, aggregate totals, reshape output
- **Final Dataset** — the processed result ready for consumption

**Data lineage** refers to tracking data from its origin through all transformations — useful for debugging, compliance, and optimization.

MongoDB's aggregation pipeline is a specific implementation: documents enter, flow through stages, and exit as aggregated results.

---

## Aggregation Pipeline Basics

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db = client["movies"]
movies_collection = db["movies_small"]

results = movies_collection.aggregate([
    { "$stage1": { ... } },
    { "$stage2": { ... } },
    # ...
])

for doc in results:
    print(doc)
```

Each stage:
- Receives input documents from the previous stage (or the collection for the first stage)
- Performs an operation on those documents
- Passes the resulting documents to the next stage

---

## Sample Data Setup

Run this script from your host machine to create the `movies_small` collection used throughout this module:

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db = client["movies"]
movies_collection = db["movies_small"]

movies_collection.drop()
movies_collection.insert_many([
  { "title": "Batman",                                           "category": ["action", "adventure"],            "imdb_rating": 7.6, "budget": 35,  "year": 1989 },
  { "title": "Godzilla",                                         "category": ["action", "adventure", "sci-fi"],  "imdb_rating": 6.6, "budget": 42,  "year": 1998 },
  { "title": "Avengers: Age of Ultron",                          "category": ["action", "adventure", "sci-fi"],  "imdb_rating": 7.4, "budget": 112, "year": 2015 },
  { "title": "The Lord of the Rings: The Fellowship of the Ring","category": ["adventure", "drama", "fantasy"],  "imdb_rating": 8.8, "budget": 75,  "year": 2001 },
  { "title": "Titanic",                                          "category": ["drama", "romance"],               "imdb_rating": 7.8, "budget": 241, "year": 1997 },
  { "title": "Big Hero 6",                                       "category": ["animation", "action", "adventure"],"imdb_rating": 7.8, "budget": 123, "year": 2014 },
  { "title": "Maze Runner: The Scorch Trials",                   "category": ["action", "sci-fi", "thriller"],   "imdb_rating": 6.3, "budget": 23,  "year": 2015 },
  { "title": "Home Alone",                                       "category": ["family", "comedy"],               "imdb_rating": 7.4, "budget": 11,  "year": 1990 },
  { "title": "How the Grinch Stole Christmas",                   "category": ["family", "comedy", "fantasy"],    "imdb_rating": 7.4, "budget": 21,  "year": 2000 },
])
```

---

## Common Aggregation Stages

### $match

Filters documents based on specified criteria — equivalent to `.find()` but used inside a pipeline.

```python
movies_collection.aggregate([
    { "$match": { "imdb_rating": { "$gte": 7.0 } } }
])
```

> Place `$match` as early as possible in the pipeline to reduce the number of documents flowing into later (more expensive) stages.

---

### $project

Shapes documents by including, excluding, or computing new fields.

```python
# Include only title; rename budget field
movies_collection.aggregate([
    { "$project": {
        "_id": 0,
        "title": 1,
        "budget_in_millions": "$budget"
    }}
])
```

The `$` prefix before a field name (e.g. `"$budget"`) references the value of that field from the incoming document.

---

### $group

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

`_id` is the grouping key — set it to `None` to aggregate across all documents:

```python
movies_collection.aggregate([
    { "$group": {
        "_id":       None,
        "total":     { "$sum": 1 },
        "avg_rating":{ "$avg": "$imdb_rating" }
    }}
])
```

---

### $sort and $limit

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

`$sort`: `1` = ascending, `-1` = descending.

---

### $unwind

Deconstructs an array field — outputs one document per array element.

```python
# { "title": "Batman", "category": ["action", "adventure"] }
# becomes two documents after $unwind: one with "action", one with "adventure"

movies_collection.aggregate([
    { "$unwind": "$category" },
    { "$group": {
        "_id":   "$category",
        "count": { "$sum": 1 }
    }}
])
```

Without `$unwind`, grouping by `$category` would treat the entire array `["action", "adventure"]` as a single key.

---

### $addFields

Adds new fields to documents, or overwrites existing ones, **without** removing any other fields. Unlike `$project`, the rest of the document is preserved.

```python
# Add a boolean field
movies_collection.aggregate([
    { "$addFields": {
        "is_blockbuster": { "$gte": ["$budget", 100] }
    }}
])
```

```python
# Combine $addFields and $project — budget_k is budget in thousands
movies_collection.aggregate([
    { "$addFields": { "budget_k": { "$multiply": ["$budget", 1000] } } },
    { "$project":   { "_id": 0, "title": 1, "budget_k": 1 } }
])
```

---

### $out

Writes the aggregation result to a collection (creates or replaces it).

```python
movies_collection.aggregate([
    { "$group": { "_id": "$category", "count": { "$sum": 1 } } },
    { "$out":   { "db": "movies", "coll": "category_counts" } }
])
```

> The 16 MB per-document limit still applies when using `$out`.

---

## Accumulators

Used inside `$group` (and some other stages):

| Accumulator | Description |
|-------------|-------------|
| `$sum`      | Total of values, or count with `1` |
| `$avg`      | Average |
| `$min`      | Minimum value |
| `$max`      | Maximum value |
| `$push`     | Collect all values into an array |
| `$first`    | First value in the group |
| `$last`     | Last value in the group |

```python
movies_collection.aggregate([
    { "$group": {
        "_id":    "$year",
        "titles": { "$push": "$title" },
        "best":   { "$max": "$imdb_rating" },
        "worst":  { "$min": "$imdb_rating" }
    }}
])
```

---

## End-to-End Examples

### Count action movies

```python
movies_collection.aggregate([
    { "$match": { "category": "action" } },
    { "$count": "num_action_movies" }
])
```

### Highest rated expensive movie

```python
movies_collection.aggregate([
    { "$match":   { "budget": { "$gt": 25 } } },
    { "$sort":    { "imdb_rating": -1 } },
    { "$limit":   1 },
    { "$project": { "_id": 0, "title": 1, "imdb_rating": 1, "budget": 1 } }
])
```

### Average IMDB rating per category

```python
movies_collection.aggregate([
    { "$unwind":  "$category" },
    { "$group": {
        "_id":        "$category",
        "avg_rating": { "$avg": "$imdb_rating" }
    }},
    { "$project": { "_id": 0, "category": "$_id", "avg_rating": 1 } },
    { "$sort":    { "avg_rating": -1 } }
])
```

### Data cleaning example

Group the larger `movies` collection (from the lab dataset) by number of directors, compute average Metacritic score. This collection has `directors` (array) and `metacritic` fields not present in `movies_small`.

```python
# Re-bind to the lab movies collection for this example
movies_full = db["movies"]

movies_full.aggregate([
    { "$group": {
        "_id": {
            "numDirectors": {
                "$cond": [{ "$isArray": "$directors" }, { "$size": "$directors" }, 0]
            }
        },
        "numFilms":          { "$sum": 1 },
        "averageMetacritic": { "$avg": "$metacritic" }
    }},
    { "$sort": { "_id.numDirectors": -1 } }
])
```

---

## Memory Limits and allowDiskUse

- Each pipeline stage is limited to **100 MB** of RAM by default.
- Each result document must not exceed **16 MB**.
- Starting in MongoDB 6.0, `allowDiskUseByDefault` can be set server-side to allow stages to write temporary files to disk.
- Pass `allowDiskUse=True` to `aggregate()` in PyMongo to enable per-query:

```python
movies_collection.aggregate(
    [
        { "$unwind":  "$category" },
        { "$group":   { "_id": "$category", "count": { "$sum": 1 } } },
        { "$sort":    { "count": -1 } }
    ],
    allowDiskUse=True
)
```

More info: [MongoDB Aggregation Pipeline Limits](https://www.mongodb.com/docs/v7.0/core/aggregation-pipeline-limits/#memory-restrictions)

---

## Practice Problems

Write Python scripts using PyMongo for each exercise.

1. Find the most massive planet in the `sample` database's planets collection.
2. Find the average IMDB rating per genre in the `labs` movies collection and sort with the highest rating first.
3. Find the top 5 classes with the highest average exam score and write the output to a collection named `top_in_exams`.

---

## Sample Outputs

### Average IMDB Rating per Genre

```
{ 'category': 'fantasy',   'avg_rating': 8.53 }
{ 'category': 'drama',     'avg_rating': 8.20 }
{ 'category': 'romance',   'avg_rating': 7.80 }
...
```

### Top 5 Classes by Average Exam Score

```
{ '_id': 219, 'average': 56.33 }
{ '_id': 204, 'average': 55.82 }
...
```

---

## Additional Resources

- [PyMongo aggregation docs](https://www.mongodb.com/docs/languages/python/pymongo-driver/current/aggregation/)
- [MongoDB Aggregation Documentation](https://www.mongodb.com/docs/manual/aggregation/)
- [Aggregation Pipeline Operators](https://www.mongodb.com/docs/manual/reference/operator/aggregation-pipeline/)
