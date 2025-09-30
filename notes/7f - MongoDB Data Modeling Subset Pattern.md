# MongoDB Data Modeling: Subset Pattern

**Course:** CSCI 122 / 212 - Contemporary Databases  
**Topic:** MongoDB Subset Pattern

---

## Overview

In this module, we explore the **Subset Pattern** in MongoDB data modeling. This pattern is used to improve performance by reducing document size when only a portion of embedded data is frequently accessed. It is especially useful when documents contain large arrays of subdocuments, such as reviews or comments, but only a small subset of those subdocuments are regularly needed by the application.

---

## What is the Subset Pattern?

The Subset Pattern is a schema design strategy that helps reduce the size of large documents by moving infrequently accessed data into a separate collection. This is particularly beneficial when:

- The document contains a large array of embedded subdocuments.
- Only a small portion of that array is frequently accessed.
- The size of the working set (frequently accessed data) exceeds available memory (RAM).

MongoDB’s default storage engine, **WiredTiger**, maintains a cache of frequently accessed data in memory. When the working set exceeds this internal cache, performance degrades due to increased disk access. By applying the Subset Pattern, we reduce the size of each document, allowing more documents to fit into memory and improving overall performance.

---

## Use Case: Bookstore Application with Reviews

In our bookstore application, each book document contains an array of embedded `reviews`. Some of our most popular books have **dozens of reviews**, but our application only displays the **first three** reviews on the product page. The rest are rarely accessed.

To optimize performance:

- We will **extract all reviews** from each book document and store them in a separate `reviews` collection.
- We will **retain only the first three reviews** in the original `books` collection.

This approach introduces **some duplication**, but since review content changes infrequently, the trade-off is acceptable for the performance gain.

---

## Why Use the Subset Pattern?

- **Improves Cache Efficiency**: Smaller documents fit better in memory.
- **Increases Query Performance**: Less data is read from disk.
- **Reduces Working Set Size**: Only frequently accessed data is kept in the main document.
- **Simplifies Access Patterns**: Frequently accessed data is colocated.

---

## Step-by-Step Example

We will apply the Subset Pattern to a `books` collection that contains embedded `reviews`.

### Step 1: Insert Sample Documents

Let us begin by inserting sample documents into the `books` collection:

```javascript
db.books.insertMany([
  {
    _id: 1,
    product_id: 34538756,
    product_type: 'book',
    title: 'MongoDB: The Definitive Guide: Powerful and Scalable Data Storage',
    description: 'MongoDB explained by MongoDB champions',
    authors: [ 'Shannon Bradshaw', 'Eoin Brazil', 'Christina Chodorow' ],
    publisher: "O'Reilly",
    language: 'English',
    pages: 514,
    catalogues: { isbn10: '1491954469', isbn13: '978-1491954461' },
    reviews: [
      { review_id: 11, text: 'This book was pretty good.', user_id: 111, upvotes: 10, rating: 10 },
      { review_id: 12, text: 'I loved this book. It was the perfect way to get started with MongoDB and understand the basics of MQL and the MongoDB architecture.', user_id: 222, upvotes: 193, rating: 7 },
      { review_id: 13, text: 'My favorite book about MongoDB, hands down. It helped me understand the basics of MongoDB and MQL.', user_id: 333, upvotes: 102, rating: 9 },
      { review_id: 14, text: 'Great resource for learning MongoDB. I highly recommend it.', user_id: 444, upvotes: 87, rating: 8 },
      { review_id: 15, text: 'I loved this book. It was the perfect way to get started with MongoDB and understand the basics of MQL and the MongoDB architecture.', user_id: 555, upvotes: 193, rating: 7 },
      { review_id: 16, text: 'My favorite book about MongoDB, hands down. It helped me understand the basics of MongoDB and MQL.', user_id: 666, upvotes: 102, rating: 9 },
      { review_id: 17, text: 'Great resource for learning MongoDB. I highly recommend it.', user_id: 777, upvotes: 87, rating: 8 },
      { review_id: 18, text: 'I loved this book. It was the perfect way to get started with MongoDB and understand the basics of MQL and the MongoDB architecture.', user_id: 888, upvotes: 28, rating: 7 },
      { review_id: 19, text: 'My favorite book about MongoDB, hands down. It helped me understand the basics of MongoDB and MQL.', user_id: 999, upvotes: 23, rating: 9 },
      { review_id: 20, text: 'Great resource for learning MongoDB. I highly recommend it.', user_id: 822, upvotes: 7, rating: 8 },
      { review_id: 21, text: 'I really enjoyed this books. I am not one for reading physical books, but the information is definitely worth it.', user_id: 724, upvotes: 81, rating: 9 },
      { review_id: 22, text: 'Great resource for anyone looking to get into MongoDB and starting off with best practices', user_id: 135, upvotes: 3, rating: 8 }
    ]
  },
  {
    _id: 2,
    product_id: 34538757,
    product_type: 'book',
    title: 'Practical MongoDB Aggregations',
    description: 'The official guide to developing optimal aggregation pipelines with MongoDB 7.0',
    authors: [ 'Paul Done' ],
    publisher: 'MongoDB Press',
    language: 'English',
    pages: 312,
    catalogues: { isbn10: '1835080642', isbn13: '978-1835080641' },
    reviews: [
      { review_id: 23, text: 'I loved this book. It was the perfect way to get started with the powerful aggregation framework.', user_id: 888, upvotes: 12, rating: 7 },
      { review_id: 24, text: 'My favorite book about MongoDB, hands down. It helped me understand the basics of aggregation.', user_id: 999, upvotes: 19, rating: 9 },
      { review_id: 25, text: 'Great resource for learning MongoDB and aggregation. I highly recommend it.', user_id: 822, upvotes: 7, rating: 8 },
      { review_id: 26, text: 'I really enjoyed this books. I am not one for reading physical books, but the information is definitely worth it.', user_id: 724, upvotes: 18, rating: 9 },
      { review_id: 27, text: 'Great resource for anyone looking to get into really taking advantage of the aggregation framework. It is practically its own language.', user_id: 135, upvotes: 312, rating: 8 }
    ]
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
    catalogues: { isbn10: '183546047X', isbn13: '978-1835460474' },
    reviews: [
      { review_id: 28, text: 'I loved this book. It was the perfect way to get started with MongoDB and understand the basics of MQL and the MongoDB architecture.', user_id: 881, upvotes: 32, rating: 7 },
      { review_id: 29, text: 'My favorite book about MongoDB, hands down. It helped me understand the basics of MongoDB and MQL.', user_id: 991, upvotes: 27, rating: 9 },
      { review_id: 30, text: 'Great resource for learning MongoDB. I highly recommend it.', user_id: 831, upvotes: 15, rating: 8 },
      { review_id: 31, text: 'I really enjoyed this books. I am not one for reading physical books, but the information is definitely worth it.', user_id: 731, upvotes: 71, rating: 9 },
      { review_id: 32, text: 'Great resource for anyone looking to get into MongoDB and starting off with best practices', user_id: 131, upvotes: 32, rating: 8 }
    ]
  }
])
```

---

### Step 2: Extract All Reviews to a New Collection

We will use an aggregation pipeline to extract all review subdocuments and move them to a new `reviews` collection.

Create a file named `copyReviews.js` with the following pipeline:

```javascript
const copyReviews = [
  {
    $unwind: { path: "$reviews" }
  },
  {
    $set: {
      "reviews.book_id": "$_id"
    }
  },
  {
    $replaceRoot: {
      newRoot: "$reviews"
    }
  },
  {
    $out: "reviews"
  }
];

db.books.aggregate(copyReviews);
```

Then run the pipeline in `mongosh`:

```javascript
load("/lab/copyReviews.js")
```

Verify the new collection:

```javascript
db.reviews.find()
```

You should see output like:

```json
[
  {
    "_id": ObjectId("..."),
    "review_id": 23,
    "text": "I loved this book. It was the perfect way to get started with the powerful aggregation framework.",
    "user_id": 888,
    "upvotes": 12,
    "rating": 7,
    "book_id": 2
  },
  ...
]
```

---

### Step 3: Retain Only the First Three Reviews in Each Book Document

Now that all reviews are backed up, we can reduce the size of each book document by keeping only the first three reviews.

Run the following aggregation pipeline:

```javascript
db.books.updateMany(
  {},
  [
    {
      $set: {
        reviews: { $slice: ["$reviews", 3] }
      }
    }
  ]
)
```

This operation uses the `$slice` operator to retain only the first three review subdocuments in each book document.

---

### Step 4: Verify the Changes

Run the following command to confirm that each book document now contains only three reviews:

```javascript
db.books.find({}, { title: 1, reviews: 1 }).pretty()
```

---

## Summary

- The **Subset Pattern** is a powerful schema design technique for improving performance when working with large documents.
- It works by **splitting frequently accessed data** from infrequently accessed data.
- In our bookstore example, we moved all reviews to a separate `reviews` collection and kept only the first three reviews in the `books` collection.
- This reduces the size of each book document, allowing more documents to fit in memory and improving query performance.

---

## Additional Resources

- [MongoDB Aggregation Framework Documentation](https://www.mongodb.com/docs/manual/aggregation/)
- [MongoDB Schema Design Best Practices](https://www.mongodb.com/docs/manual/core/data-model-design/)
- [MongoDB Subset Pattern](https://www.mongodb.com/blog/post/building-with-patterns-the-subset-pattern)

--- 

## Lab Validation

To validate your work, run the following in the terminal:

```bash
python3 /lab/validate_subset_pattern_changes.py
```

If successful, you have completed the lab and applied the Subset Pattern correctly.
