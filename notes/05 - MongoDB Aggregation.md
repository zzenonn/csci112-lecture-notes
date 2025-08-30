# MongoDB Aggregation Pipeline  
CSCI 112 / 212 - Contemporary Databases  

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Aggregation Pipeline Basics](#aggregation-pipeline-basics)
  - [Syntax](#syntax)
- [Sample Data Setup](#sample-data-setup)
- [Common Aggregation Stages](#common-aggregation-stages)
  - [`$match`](#match)
  - [`$project`](#project)
  - [`$group`](#group)
  - [`$sort`](#sort)
  - [`$limit`](#limit)
  - [`$unwind`](#unwind)
  - [`$out`](#out)
- [The `$addFields` Stage](#the-addfields-stage)
  - [Purpose](#purpose)
  - [Syntax](#syntax-1)
  - [Example: Add a field for budget in millions](#example-add-a-field-for-budget-in-millions)
  - [Example: Add a field indicating if the movie is a blockbuster](#example-add-a-field-indicating-if-the-movie-is-a-blockbuster)
  - [Example: Combine with `$project`](#example-combine-with-project)
- [Sample Data](#sample-data)
- [Example Queries](#example-queries)
  - [Count the number of action movies](#count-the-number-of-action-movies)
  - [Highest rated expensive movie](#highest-rated-expensive-movie)
  - [Average IMDB rating per category](#average-imdb-rating-per-category)
- [Data Cleaning Example](#data-cleaning-example)
- [Practice Problems](#practice-problems)
- [Notes on `$out` and Pipeline Limits](#notes-on-out-and-pipeline-limits)
- [Sample Outputs](#sample-outputs)
  - [Most Massive Planet](#most-massive-planet)
  - [Average IMDB Rating per Genre](#average-imdb-rating-per-genre)
  - [Top 5 Classes by Average Exam Score](#top-5-classes-by-average-exam-score)
- [Additional Resources](#additional-resources)

## Overview  
This document provides a comprehensive overview of MongoDB’s Aggregation Framework, including key pipeline stages, syntax, and practical examples. 
---

## Objectives  
By the end of this module, you should be able to:

- Describe the purpose and syntax of common aggregation stage operators in MongoDB.
- Understand how documents flow through the aggregation pipeline and how each stage transforms the data.
- Apply the aggregation framework to perform data analysis and transformation tasks on MongoDB collections.

---

## Aggregation Pipeline Basics  

The aggregation pipeline is a framework for data aggregation modeled on the concept of data processing pipelines. Documents enter a multi-stage pipeline that transforms the documents into aggregated results.

### Syntax  
```js
db.<collection>.aggregate([
  { <stage1> },
  { <stage2> },
  ...
  { <stageN> }
], { options })
```

Each stage:
- Receives input documents.
- Performs an operation on those documents.
- Passes the resulting documents to the next stage.

---

## Sample Data Setup

Before exploring aggregation operations, let's set up a sample movies collection:

```js
db.movies_small.drop()
db.movies_small.insertMany([
  {
	"title": "Batman2",
	"category": ["action", "adventure"],
	"imdb_rating": 7.6,
	"budget": 35
  },
  {
	"title": "Godzilla",
	"category": ["action", "adventure", "sci-fi"],
	"imdb_rating": 6.6,
	"budget": 42
  },
  {
	"title": "Avengers: Age of Ultron",
	"category": ["action", "adventure", "sci-fi"],
	"imdb_rating": 7.4,
	"budget": 112
  },
  {
	"title": "The Lord of the Rings: The Fellowship of the Ring",
	"category": ["adventure", "drama", "fantasy"],
	"imdb_rating": 8.8,
	"budget": 75
  },
  {
	"title": "Titanic",
	"category": ["drama", "romance"],
	"imdb_rating": 7.8,
	"budget": 241
  },
  {
	"title": "Big Hero 6 ",
	"category": ["animation", "action", "adventure"],
	"imdb_rating": 7.8,
	"budget": 123
  },
  {
	"title": "Maze Runner: The Scorch Trials",
	"category": ["action", "sci-fi", "thriller"],
	"imdb_rating": 6.3,
	"budget": 23
  },
  {
	"title": "Home Alone",
	"category": ["family", "comedy"],
	"imdb_rating": 7.4,
	"budget": 11
  },
  {
	"title": "How the Grinch Stole Christmas",
	"category": ["family", "comedy", "fantasy"],
	"imdb_rating": 7.4,
	"budget": 21
  }
])
```

---

## Common Aggregation Stages  

### `$match`  
Filters documents based on specified criteria (similar to `.find()`).
```js
{ "$match": { "imdb_rating": { "$gte": 7.0 } } }
```

### `$project`  
Shapes documents by including, excluding, or computing new fields.
```js
{ "$project": { "_id": 0, "title": 1, "budget_in_million": "$budget" } }
```

### `$group`  
Groups documents by a specified identifier and performs accumulations like `$sum`, `$avg`, `$min`, `$max`.
```js
{ "$group": { "_id": "$year", "num_films": { "$sum": 1 } } }
```

### `$sort`  
Sorts documents in ascending (`1`) or descending (`-1`) order.
```js
{ "$sort": { "num_films": -1 } }
```

### `$limit`  
Limits the number of documents passed to the next stage.
```js
{ "$limit": 5 }
```

### `$unwind`  
Deconstructs an array field from the input documents to output a document for each element.
```js
{ "$unwind": "$category" }
```

### `$out`  
Writes the result of the aggregation pipeline to a specified collection.
```js
{ "$out": { "db": "sample", "coll": "celestial_body_names" } }
```

---

## The `$addFields` Stage  

### Purpose  
The `$addFields` stage is used to add new fields to documents or modify existing fields. It is similar to `$project`, but instead of reshaping the entire document, it adds or updates specific fields while preserving the rest of the document.

### Syntax  
```js
{ "$addFields": { <newField>: <expression>, ... } }
```

### Example: Add a field for budget in millions  
```js
db.movies_small.aggregate([
  {
    "$addFields": {
      "budget_millions": { "$divide": ["$budget", 1] }
    }
  }
])
```

### Example: Add a field indicating if the movie is a blockbuster  
```js
db.movies_small.aggregate([
  {
    "$addFields": {
      "is_blockbuster": { "$gte": ["$budget", 100] }
    }
  }
])
```

### Example: Combine with `$project`  
```js
db.movies_small.aggregate([
  {
    "$addFields": {
      "budget_millions": { "$divide": ["$budget", 1] }
    }
  },
  {
    "$project": {
      "_id": 0,
      "title": 1,
      "budget_millions": 1
    }
  }
])
```

---

## Sample Data  
The `movies_small` collection contains documents like:
```json
{
  "title": "Batman2",
  "category": ["action", "adventure"],
  "imdb_rating": 7.6,
  "budget": 35
}
```

---

## Example Queries  

### Count the number of action movies  
```js
db.movies_small.aggregate([
  { "$match": { "category": "action" } },
  { "$count": "movies" }
])
```

### Highest rated expensive movie  
```js
db.movies_small.aggregate([
  { "$match": { "budget": { "$gt": 25 } } },
  { "$group": { "_id": "movies", "highest": { "$max": "$imdb_rating" } } }
])
```

### Average IMDB rating per category  
```js
db.movies_small.aggregate([
  { "$unwind": "$category" },
  { "$group": { "_id": "$category", "avgIMDBrating": { "$avg": "$imdb_rating" } } },
  { "$project": { "_id": 0, "category": "$_id", "avgIMDBrating": 1 } },
  { "$sort": { "avgIMDBrating": -1 } }
])
```

---

## Data Cleaning Example  
Group movies by number of directors and calculate average Metacritic score:
```js
db.movies.aggregate([
  {
    "$group": {
      "_id": {
        "numDirectors": {
          "$cond": [{ "$isArray": "$directors" }, { "$size": "$directors" }, 0]
        }
      },
      "numFilms": { "$sum": 1 },
      "averageMetacritic": { "$avg": "$metacritic" }
    }
  },
  { "$sort": { "_id.numDirectors": -1 } }
])
```

---

## Practice Problems  

1. Find the most massive planet in the solar system.
2. Find average IMDB rating per genre in the movies collection and sort with highest rating first.
3. Find the top 5 classes with the highest average exam score and write the output to a collection named `top_in_exams`.

---

## Notes on `$out` and Pipeline Limits  

- Each document in the result must not exceed 16MB.
- Starting in MongoDB 6.0, the `allowDiskUseByDefault` parameter allows pipeline stages to write temporary files to disk if they exceed 100MB of memory.
- When using `$out` or `$merge`, the 16MB document size limit still applies.

More info: [MongoDB Aggregation Pipeline Limits](https://www.mongodb.com/docs/v7.0/core/aggregation-pipeline-limits/#memory-restrictions)

---

## Sample Outputs  

### Most Massive Planet  
```json
{ "name": "Jupiter", "mass": 1.89819e+27 }
```

### Average IMDB Rating per Genre  
```json
{ "_id": "Documentary", "averageRating": 7.24 }
{ "_id": "Drama", "averageRating": 6.62 }
...
```

### Top 5 Classes by Average Exam Score  
```json
{ "_id": 219, "average": 56.33 }
{ "_id": 204, "average": 55.82 }
...
```

---

## Additional Resources  
- [MongoDB Aggregation Documentation](https://www.mongodb.com/docs/manual/aggregation/)
- [Aggregation Pipeline Operators](https://www.mongodb.com/docs/manual/reference/operator/aggregation-pipeline/)

