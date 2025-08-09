# MongoDB Data Structures and CRUD Operations

CSCI 112 / 212 - Contemporary Databases

## Overview

This document provides an introduction to MongoDB data structures and basic CRUD operations. It also covers how MongoDB handles document identifiers, querying, projections, cursors, and update operations.

---

## Objectives

- Understand MongoDB document structure and data types
- Learn how MongoDB generates and uses `_id` fields
- Perform basic CRUD operations
- Use cursor methods: `.count()`, `.sort()`, `.skip()`, `.limit()`
- Understand query operators and projections
- Work with arrays and nested documents

---

## MongoDB Basics

### MongoDB IDs

- MongoDB automatically generates a unique `_id` field if one is not provided.
- The default `_id` is an ObjectId, which is 12 bytes (24-character hexadecimal string).
- ObjectId structure:
  1. Timestamp (4 bytes)
  2. Machine Identifier (3 bytes)
  3. Process ID (2 bytes)
  4. Counter (3 bytes)

Example:
```js
ObjectId("66eaf030ff3099f18bc2d30c")
```

You can extract the timestamp from an ObjectId to determine when a document was created.

---

## CRUD Operations

### Insert

```js
movies.insert_one({ "title": "Jaws" })
```

If `_id` is not specified, MongoDB will generate one.

Examples:

```js
movies.insert_one({ "_id": "Star Wars" }) // Success
movies.insert_one({ "_id": "Star Wars" }) // DuplicateKeyError
movies.insert_one({ "_id": ["Star Wars", "Empire"] }) // Invalid _id
```

### Ordered vs. Unordered Inserts

- Ordered (default): Stops on first error
- Unordered: Continues inserting remaining documents

```js
movies.insert_many([
  { "_id": "Batman", "year": 1989 },
  { "_id": "Home Alone", "year": 1990 },
  { "_id": "Ghostbusters", "year": 1984 },
  { "_id": "Ghostbusters", "year": 1984 },
  { "_id": "Lord of the Rings", "year": 2001 }
], { ordered: false })
```

### Inserting Arrays

```js
movies.insert_many([
  { "title": "Batman", "category": ["action", "adventure"] },
  { "title": "Godzilla", "category": ["action", "adventure", "sci-fi"] },
  { "title": "Home Alone", "category": ["family", "comedy"] },
  { "title": "The Muppets", "category": ["comedy", "family"] }
])
```

---

## Finding Documents

### Basic Find

```js
movies.find({ "title": "Batman" })
movies.find({ "category": ["family", "comedy"] })
movies.find({ "category": "family" })
```

### Dot Notation for Nested Fields

```js
movies.find({ "box_office.gross": 760 })
```

---

## Deleting Documents

```js
movies.delete_one({ "category": "action" }) // Deletes first match
movies.delete_many({ "category": "action" }) // Deletes all matches
movies.drop() // Deletes the collection
```

### Deleting a Database

```js
client.drop_database('database_name')
```

---

## Projections

Specify which fields to include or exclude in the result.

```js
movies.find({}, { "title": 1 }) // Include only title
movies.find({ "_id": ObjectId("...") }, { "title": 0 }) // Exclude title
```

---

## Cursors

When using `.find()`, MongoDB returns a cursor.

```js
cursor = movies.find()
cursor.hasNext()
cursor.next()
```

### Cursor Methods

```js
movies.find().sort("a", 1) // Ascending
movies.find().sort("a", -1) // Descending
movies.find().limit(5)
movies.find().skip(5)
movies.find().sort("a", 1).limit(5).skip(5)
```

---

## Query Operators

### Comparison Operators

| Operator | Description |
|----------|-------------|
| $lt      | Less than |
| $lte     | Less than or equal |
| $gt      | Greater than |
| $gte     | Greater than or equal |
| $eq      | Equal |
| $ne      | Not equal or does not exist |
| $in      | In array |
| $nin     | Not in array |

Examples:

```js
movies.find({ "imdb_rating": { "$gte": 7 } })
movies.find({ "category": { "$ne": "family" } })
movies.find({ "title": { "$in": ["Batman", "Godzilla"] } })
movies.find({ "title": { "$nin": ["Batman", "Godzilla"] } })
```

### Logical Operators

| Operator | Description |
|----------|-------------|
| $or      | Match any |
| $and     | Match all |
| $not     | Negate condition |
| $nor     | Match none |

Examples:

```js
movies.find({ "$or": [
  { "category": "sci-fi" },
  { "imdb_rating": { "$gte": 7 } }
] })

movies.find({ "$or": [
  { "category": "sci-fi", "imdb_rating": { "$gte": 8 } },
  { "category": "family", "imdb_rating": { "$gte": 7 } }
] })
```

---

## Array Query Operators

| Operator    | Description |
|-------------|-------------|
| $all        | All values must be present |
| $size       | Array must have exact length |
| $elemMatch  | Match at least one element with all conditions |

Examples:

```js
movies.find({ "category": { "$size": 3 } })
movies.find({ "category": { "$all": ["sci-fi", "action"] } })
movies.find({ "category": { "$in": ["sci-fi", "action"] } })
```

### $elemMatch

```js
movies.find({
  "filming_locations": {
    "$elemMatch": {
      "city": "Florence",
      "country": "Italy"
    }
  }
})
```

---

## Updating Documents

### replace_one()

Replaces the entire document (except `_id`).

```js
movies.replace_one(
  { "title": "Batman" },
  { "imdb_rating": 7.7 }
)
```

### update_one()

Modifies specific fields.

```js
movies.update_one(
  { "title": "Batman" },
  { "$set": { "imdb_rating": 7.7 } }
)

movies.update_one(
  { "title": "Godzilla" },
  { "$set": { "budget": 1 } }
)
```

### update_many()

Updates multiple documents.

```js
movies.update_many({}, { "$set": { "sequels": 0 } })
```

---

## Other Update Operators

| Operator        | Description |
|-----------------|-------------|
| $set            | Set value |
| $unset          | Remove field |
| $inc            | Increment |
| $mul            | Multiply |
| $rename         | Rename field |
| $min            | Set if less than current |
| $max            | Set if greater than current |
| $currentDate    | Set to current date or timestamp |

Example:

```js
movies.update_many({}, {
  "$currentDate": {
    "release_date": { "$type": "date" }
  }
})
```

---

## Practice Exercises

1. Find all sci-fi movies.
2. Find either sci-fi or comedy movies.
3. Find all movies that made a profit:
```js
{
  "$expr": {
    "$gt": ["$revenue", "$budget"]
  }
}
```
4. Increment Batman’s IMDB rating by 1:
```js
movies.update_one({ "title": "Batman" }, { "$inc": { "imdb_rating": 1 } })
```
5. Increment Home Alone’s budget by 5.
6. Remove Home Alone’s budget.
7. Multiply Batman’s IMDB rating by 4.
8. Rename the field `budget` to `budget_in_millions`.

---

## Summary

This guide provides a foundational understanding of MongoDB's document model, CRUD operations, query operators, and update mechanisms. Practice using these commands in the `mongosh` shell to reinforce your understanding.

For further reading, consult the official MongoDB documentation at: https://www.mongodb.com/docs/manual/
