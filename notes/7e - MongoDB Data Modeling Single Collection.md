# MongoDB Data Modeling: Single Collection Pattern

**Course:** CSCI 122 / 212 - Contemporary Databases  
**Topic:** MongoDB Data Modeling - Single Collection Pattern

---

## Overview: The Single Collection Pattern

In applications like our bookstore, we often deal with multiple related entities such as books, users, and reviews. These entities are frequently queried together. While embedding related documents might seem like a solution, it can lead to unbounded arrays or exceed MongoDB’s 16MB document size limit. On the other hand, separating them into different collections requires expensive $lookup operations or multiple queries.

To address this, we use the Single Collection Pattern.

### What is the Single Collection Pattern?

The Single Collection Pattern stores related documents of different types in a single collection. Each document includes a docType field to identify its type (e.g., book, user, review), and a relatedTo field to model relationships between documents.

This pattern is especially useful for:

- Many-to-many relationships where embedding is not feasible
- One-to-many relationships where joins are costly

### Variants of the Single Collection Pattern

1. Array of References (Many-to-Many or One-to-Many)
   - Use a docType field to identify the document type
   - Use a relatedTo field (array) to reference related documents

2. Overloaded Field (One-to-Many Only)
   - Use a single field (e.g., _id) to encode both the parent and child document identifiers
   - Enables efficient queries using prefix-based filtering

---

## Example: Applying the Single Collection Pattern

Let’s walk through an example where we migrate three collections — books, users, and reviews — into a single collection called books_catalog.

---

## Step 1: Insert Initial Data into Separate Collections

Before we apply the Single Collection Pattern, we first insert the initial data into three separate collections: books, users, and reviews.

### Insert Books

```javascript
db.books.insertMany([
  {
    _id: 2,
    product_id: 34538757,
    product_type: 'book',
    title: 'Practical MongoDB Aggregations',
    description: 'The official guide to developing optimal aggregation pipelines with MongoDB 7.0',
    authors: ['Paul Done'],
    publisher: 'MongoDB Press',
    language: 'English',
    pages: 312,
    catalogues: { isbn10: '1835080642', isbn13: '978-1835080641' }
  },
  {
    _id: 1,
    product_id: 34538756,
    product_type: 'book',
    title: 'MongoDB: The Definitive Guide: Powerful and Scalable Data Storage',
    description: 'MongoDB explained by MongoDB champions',
    authors: ['Shannon Bradshaw', 'Eoin Brazil', 'Christina Chodorow'],
    publisher: "O'Reilly",
    language: 'English',
    pages: 514,
    catalogues: { isbn10: '1491954469', isbn13: '978-1491954461' }
  },
  {
    _id: 3,
    product_id: 34538758,
    product_type: 'book',
    title: 'Mastering MongoDB 7.0',
    description: 'Explore the full potential of MongoDB 7.0 with this comprehensive guide.',
    authors: [
      'Marko Aleksendric',
      'Arek Borucki',
      'Leandro Domingues',
      'Malak Abu Hammad',
      'Elie Hannouch',
      'Rajesh Nair',
      'Rachelle Palmer'
    ],
    publisher: 'MongoDB Press',
    language: 'English',
    pages: 185,
    catalogues: { isbn10: '183546047X', isbn13: '978-1835460474' }
  }
]);
```

### Insert Users

```javascript
db.users.insertMany([
  {
    _id: 111,
    user_id: 111,
    name: 'Alice',
    email: 'alice@example.com',
    recent_reviews: [11],
    purchased_books: [34538756]
  },
  {
    _id: 222,
    user_id: 222,
    name: 'Bob',
    email: 'bob@example.com',
    recent_reviews: [22],
    purchased_books: [34538757]
  },
  {
    _id: 333,
    user_id: 333,
    name: 'Charlie',
    email: 'charlie@example.com',
    recent_reviews: [33],
    purchased_books: [34538758]
  }
]);
```

### Insert Reviews

```javascript
db.reviews.insertMany([
  {
    _id: 22,
    review_id: 22,
    text: 'This book was good.',
    book_id: 34538757,
    user_id: 222,
    rating: 1
  },
  {
    _id: 11,
    review_id: 11,
    text: 'This book was pretty good.',
    book_id: 34538756,
    user_id: 111,
    rating: 3
  },
  {
    _id: 33,
    review_id: 33,
    text: 'This book was insanely great.',
    book_id: 34538758,
    user_id: 333,
    rating: 5
  }
]);
```

---

## Step 2: Transform and Insert into Single Collection

We will use aggregation pipelines to transform and merge the documents into the new books_catalog collection.

### Books Pipeline

```javascript
const booksPipeline = [
  {
    $set: {
      docType: "book",
      relatedTo: ["$product_id"],
      book_id: "$product_id"
    }
  },
  {
    $unset: ["product_id"]
  },
  {
    $merge: {
      into: "books_catalog",
      on: "_id"
    }
  }
];

db.books.aggregate(booksPipeline);
```

### Users Pipeline

```javascript
const usersPipeline = [
  {
    $set: {
      docType: "user",
      relatedTo: "$purchased_books"
    }
  },
  {
    $unset: ["purchased_books"]
  },
  {
    $merge: {
      into: "books_catalog",
      on: "_id"
    }
  }
];

db.users.aggregate(usersPipeline);
```

### Reviews Pipeline

```javascript
const reviewsPipeline = [
  {
    $set: {
      docType: "review",
      relatedTo: ["$book_id"]
    }
  },
  {
    $unset: ["book_id"]
  },
  {
    $merge: {
      into: "books_catalog",
      on: "_id"
    }
  }
];

db.reviews.aggregate(reviewsPipeline);
```

---

## Step 3: Querying the Single Collection

After running the pipelines, you can query the new collection:

```javascript
db.books_catalog.find()
```

Filter by document type:

```javascript
db.books_catalog.find({ docType: "review" })
```

Retrieve all documents related to a specific book:

```javascript
db.books_catalog.find({ relatedTo: 34538756 })
```

---

## Summary

We have successfully applied the Single Collection Pattern to consolidate books, users, and reviews into a single collection. This approach:

- Reduces the need for joins and multiple queries
- Improves query performance
- Simplifies data access for related entities

This pattern is especially useful when embedding is not feasible and relationships are complex.

Continue exploring how MongoDB patterns can help you build scalable and performant applications.
