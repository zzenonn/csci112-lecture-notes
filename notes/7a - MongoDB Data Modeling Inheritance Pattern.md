# MongoDB Data Modeling: Inheritance Pattern

**Course:** CSCI 112 / 212 - Contemporary Databases  
**Topic:** MongoDB Inheritance Pattern

---

## Overview

In this module, we explore the **Inheritance Pattern** in MongoDB data modeling. This pattern allows us to store related but distinct document types within a single collection by leveraging polymorphism — a key feature of MongoDB's flexible schema design.

---

## What is the Inheritance Pattern?

The Inheritance Pattern is used when multiple document types share a common set of fields but also have unique attributes. Rather than splitting these documents into separate collections, we store them together in a single collection and use a field (commonly `product_type`) to distinguish between them.

This approach is ideal when:

- The documents have more similarities than differences.
- The documents are often queried together.
- You want to reduce the complexity of managing multiple collections.

---

## Example: Bookstore Application

Consider a bookstore application that manages different types of books:

- Printed books
- Ebooks
- Audiobooks

Each of these product types shares common fields such as `title`, `author`, `publisher`, and `language`, but also has unique attributes:

- **Printed books**: `pages`, `stock_level`, `delivery_time`
- **Ebooks**: `eformats`, `download_url`
- **Audiobooks**: `narrator`, `length_minutes`, `time_by_chapter`

Using the Inheritance Pattern, we store all these documents in a single `books` collection and use a `product_type` field to distinguish between them.

---

## Why Use the Inheritance Pattern?

- **Efficiency**: Documents that are accessed together are stored together.
- **Flexibility**: MongoDB allows documents with different shapes in the same collection.
- **Maintainability**: Easier to manage and query a single collection.

---

## Applying the Inheritance Pattern to Existing Data

In real-world applications, you may need to transform an existing collection to follow the Inheritance Pattern. MongoDB's Aggregation Framework is a powerful tool for this task.

---

## Step-by-Step Example

We will apply the Inheritance Pattern to a collection named:

```
books
```

This collection contains three documents representing different book types, each with slightly different structures.

### Step 1: Insert Sample Documents

First, let's insert the sample documents into our collection:

```javascript
db.books.insertMany([
  {
    "_id": 1,
    "product_id": 34538756,
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "details": "MongoDB explained by MongoDB champions",
    "authors": [ "Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow" ],
    "publisher": "O'Reilly",
    "language": "English",
    "pages": 514,
    "catalogues": { "isbn10": "1491954469", "isbn13": "978-1491954461" }
  },
  {
    "_id": 2,
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "desc": "MongoDB explained by MongoDB champions",
    "authors": "Shannon Bradshaw, Eoin Brazil, and Christina Chodorow",
    "publisher": "O'Reilly",
    "language": "English",
    "eformats": { "epub": { "pages": 774 }, "pdf": { "pages": 502 } },
    "isbn10": "1491954469"
  },
  {
    "_id": 3,
    "product_id": 54538756,
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "desc": "The complete book of MongoDB by its employees",
    "author": "Eoin Brazil",
    "narrator": "Eoin Brazil",
    "publisher": "O'Reilly",
    "language": "English",
    "length_minutes": 1200
  }
])
```

### Step 2: Review the Documents

Run the following command in `mongosh` to inspect the documents:

```javascript
db.books.find()
```

---

## Step 3: Update Documents to Follow the Inheritance Pattern

We will update each document to:

- Add a `product_type` field.
- Standardize field names (`desc` → `description`, `details` → `description`).
- Convert `author` or `authors` to an array of strings.
- Add missing `product_id` fields where necessary.

### Option A: Manual Updates

You can manually update each document using `replaceOne`:

#### Document 1: Printed Book

```javascript
db.books.replaceOne(
  { _id: 1 },
  {
    "_id": 1,
    "product_id": 34538756,
    "product_type": "book",
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "description": "MongoDB explained by MongoDB champions",
    "authors": [ "Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow" ],
    "publisher": "O'Reilly",
    "language": "English",
    "pages": 514,
    "catalogues": { "isbn10": "1491954469", "isbn13": "978-1491954461" }
  }
)
```

#### Document 2: Ebook

```javascript
db.books.replaceOne(
  { _id: 2 },
  {
    "_id": 2,
    "product_id": 44538756,
    "product_type": "ebook",
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "description": "MongoDB explained by MongoDB champions",
    "authors": [ "Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow" ],
    "publisher": "O'Reilly",
    "language": "English",
    "eformats": { "epub": { "pages": 774 }, "pdf": { "pages": 502 } },
    "isbn10": "1491954469"
  }
)
```

#### Document 3: Audiobook

```javascript
db.books.replaceOne(
  { _id: 3 },
  {
    "_id": 3,
    "product_id": 54538756,
    "product_type": "audiobook",
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "description": "The complete book of MongoDB by its employees",
    "authors": [ "Eoin Brazil" ],
    "narrator": "Eoin Brazil",
    "publisher": "O'Reilly",
    "language": "English",
    "length_minutes": 1200
  }
)
```

### Option B: Aggregation Pipeline Automation

You can also use the MongoDB Aggregation Framework to apply the Inheritance Pattern in bulk.

#### Apply Inheritance Pattern to All Documents

```javascript
var apply_inheritance_pattern_to_books_pipeline = [
  {
    $project: {
      _id: "$_id",
      product_id: { $ifNull: ["$product_id", NumberInt(0)] },
      product_type: {
        $ifNull: ["$product_type", "Unspecified"],
      },
      title: "$title",
      description: {
        $ifNull: [
          "$desc",
          "$description",
          "$details",
          "Unspecified",
        ],
      },
      authors: {
        $cond: {
          if: { $isArray: "$authors" },
          then: "$authors",
          else: {
            $cond: {
              if: { $isArray: ["$author"] },
              then: "$author",
              else: [{ $ifNull: ["$author", "Unspecified"] }]
            }
          }
        }
      },
      publisher: "$publisher",
      language: "$language",
      pages: "$pages",
      catalogues: "$catalogues",
      eformats: "$eformats",
      isbn10: "$isbn10",
      isbn13: "$isbn13",
      narrator: "$narrator",
      length_minutes: "$length_minutes",
    },
  },
  {
    $merge: {
      into: "books",
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "discard",
    },
  },
]

db.books.aggregate(apply_inheritance_pattern_to_books_pipeline)
```

#### Update Audiobook Product Types

```javascript
var cleanup_audiobook_entries_in_book_pipeline = [
  {
    $match: {
      $and: [{ product_type: "Unspecified" }, { length_minutes: { $gte: 0 } }],
    },
  },
  {
    $set: { product_type: "audiobook" },
  },
  {
    $merge: {
      into: "books",
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "discard",
    },
  },
];

db.books.aggregate(cleanup_audiobook_entries_in_book_pipeline);
```

---

## Step 4: Verify the Updates

Run the following command to verify that all documents now follow the Inheritance Pattern:

```javascript
db.books.find()
```

Expected output:

```json
[
  {
    "_id": 1,
    "product_id": 34538756,
    "product_type": "book",
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "description": "MongoDB explained by MongoDB champions",
    "authors": [ "Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow" ],
    "publisher": "O'Reilly",
    "language": "English",
    "pages": 514,
    "catalogues": { "isbn10": "1491954469", "isbn13": "978-1491954461" }
  },
  {
    "_id": 2,
    "product_id": 44538756,
    "product_type": "ebook",
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "description": "MongoDB explained by MongoDB champions",
    "authors": [ "Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow" ],
    "publisher": "O'Reilly",
    "language": "English",
    "eformats": { "epub": { "pages": 774 }, "pdf": { "pages": 502 } },
    "isbn10": "1491954469"
  },
  {
    "_id": 3,
    "product_id": 54538756,
    "product_type": "audiobook",
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "description": "The complete book of MongoDB by its employees",
    "authors": [ "Eoin Brazil" ],
    "narrator": "Eoin Brazil",
    "publisher": "O'Reilly",
    "language": "English",
    "length_minutes": 1200
  }
]
```

---

## Summary

- The **Inheritance Pattern** allows you to store polymorphic documents in a single collection.
- Use a `product_type` field to distinguish between different document shapes.
- MongoDB's **Aggregation Framework** and update operations can help you apply this pattern to existing data.
- This pattern is especially useful when documents are frequently queried together and share many common fields.

---

## Additional Resources

- [MongoDB Aggregation Framework Documentation](https://www.mongodb.com/docs/manual/aggregation/)
- [MongoDB Schema Design Best Practices](https://www.mongodb.com/docs/manual/core/data-model-design/)

For further questions or assistance, please reach out during office hours or post in the course discussion forum.
