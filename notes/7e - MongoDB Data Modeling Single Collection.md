# MongoDB Data Modeling: Single Collection Pattern

**Course:** CSCI 112 / 212 - Contemporary Databases
**Topic:** MongoDB Data Modeling - Single Collection Pattern

---

## Overview: The Single Collection Pattern

In applications like our bookstore, we often deal with multiple related entities such as books, users, and reviews. These entities are frequently queried together. While embedding related documents might seem like a solution, it can lead to unbounded arrays or exceed MongoDB's 16MB document size limit. On the other hand, separating them into different collections requires expensive $lookup operations or multiple queries.

To address this, we use the Single Collection Pattern.

All examples use **PyMongo** running on your laptop, connecting to `mongod` on a VM.

### What is the Single Collection Pattern?

The Single Collection Pattern stores related documents of different types in a single collection. Each document includes a `docType` field to identify its type (e.g., `book`, `user`, `review`), and a `relatedTo` field to model relationships between documents.

This pattern is especially useful for:

- Many-to-many relationships where embedding is not feasible
- One-to-many relationships where joins are costly

### Variants of the Single Collection Pattern

1. Array of References (Many-to-Many or One-to-Many)
   - Use a `docType` field to identify the document type
   - Use a `relatedTo` field (array) to reference related documents

2. Overloaded Field (One-to-Many Only)
   - Use a single field (e.g., `_id`) to encode both the parent and child document identifiers
   - Enables efficient queries using prefix-based filtering

---

## PyMongo Setup

Activate your virtual environment and install PyMongo if you haven't already:

```bash
python -m venv .venv
source .venv/bin/activate          # Linux / macOS
pip install pymongo
```

Connect to your MongoDB instance:

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client        = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db            = client["bookstore"]
books         = db["books"]
users         = db["users"]
reviews       = db["reviews"]
books_catalog = db["books_catalog"]
```

The four variables are **collection handles** — every example below calls methods on them.

---

## Example: Applying the Single Collection Pattern

Let's walk through an example where we migrate three collections — `books`, `users`, and `reviews` — into a single collection called `books_catalog`.

---

## Step 1: Insert Initial Data into Separate Collections

Before we apply the Single Collection Pattern, we first insert the initial data into three separate collections.

### Insert Books

```python
books.insert_many([
    {
        "_id": 2,
        "product_id": 34538757,
        "product_type": "book",
        "title": "Practical MongoDB Aggregations",
        "description": "The official guide to developing optimal aggregation pipelines with MongoDB 7.0",
        "authors": ["Paul Done"],
        "publisher": "MongoDB Press",
        "language": "English",
        "pages": 312,
        "catalogues": {"isbn10": "1835080642", "isbn13": "978-1835080641"}
    },
    {
        "_id": 1,
        "product_id": 34538756,
        "product_type": "book",
        "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
        "description": "MongoDB explained by MongoDB champions",
        "authors": ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
        "publisher": "O'Reilly",
        "language": "English",
        "pages": 514,
        "catalogues": {"isbn10": "1491954469", "isbn13": "978-1491954461"}
    },
    {
        "_id": 3,
        "product_id": 34538758,
        "product_type": "book",
        "title": "Mastering MongoDB 7.0",
        "description": "Explore the full potential of MongoDB 7.0 with this comprehensive guide.",
        "authors": [
            "Marko Aleksendric",
            "Arek Borucki",
            "Leandro Domingues",
            "Malak Abu Hammad",
            "Elie Hannouch",
            "Rajesh Nair",
            "Rachelle Palmer"
        ],
        "publisher": "MongoDB Press",
        "language": "English",
        "pages": 185,
        "catalogues": {"isbn10": "183546047X", "isbn13": "978-1835460474"}
    }
])
```

### Insert Users

```python
users.insert_many([
    {
        "_id": 111,
        "user_id": 111,
        "name": "Alice",
        "email": "alice@example.com",
        "recent_reviews": [11],
        "purchased_books": [34538756]
    },
    {
        "_id": 222,
        "user_id": 222,
        "name": "Bob",
        "email": "bob@example.com",
        "recent_reviews": [22],
        "purchased_books": [34538757]
    },
    {
        "_id": 333,
        "user_id": 333,
        "name": "Charlie",
        "email": "charlie@example.com",
        "recent_reviews": [33],
        "purchased_books": [34538758]
    }
])
```

### Insert Reviews

```python
reviews.insert_many([
    {
        "_id": 22,
        "review_id": 22,
        "text": "This book was good.",
        "book_id": 34538757,
        "user_id": 222,
        "rating": 1
    },
    {
        "_id": 11,
        "review_id": 11,
        "text": "This book was pretty good.",
        "book_id": 34538756,
        "user_id": 111,
        "rating": 3
    },
    {
        "_id": 33,
        "review_id": 33,
        "text": "This book was insanely great.",
        "book_id": 34538758,
        "user_id": 333,
        "rating": 5
    }
])
```

---

## Step 2: Transform and Insert into Single Collection

We will use aggregation pipelines to transform and merge the documents into the new `books_catalog` collection.

### Books Pipeline

```python
books_pipeline = [
    {
        "$set": {
            "docType": "book",
            "relatedTo": ["$product_id"],
            "book_id": "$product_id"
        }
    },
    {
        "$unset": ["product_id"]
    },
    {
        "$merge": {
            "into": "books_catalog",
            "on": "_id"
        }
    }
]

books.aggregate(books_pipeline)
```

### Users Pipeline

```python
users_pipeline = [
    {
        "$set": {
            "docType": "user",
            "relatedTo": "$purchased_books"
        }
    },
    {
        "$unset": ["purchased_books"]
    },
    {
        "$merge": {
            "into": "books_catalog",
            "on": "_id"
        }
    }
]

users.aggregate(users_pipeline)
```

### Reviews Pipeline

```python
reviews_pipeline = [
    {
        "$set": {
            "docType": "review",
            "relatedTo": ["$book_id"]
        }
    },
    {
        "$unset": ["book_id"]
    },
    {
        "$merge": {
            "into": "books_catalog",
            "on": "_id"
        }
    }
]

reviews.aggregate(reviews_pipeline)
```

---

## Step 3: Querying the Single Collection

After running the pipelines, you can query the new collection:

```python
for doc in books_catalog.find():
    print(doc)
```

Filter by document type:

```python
for doc in books_catalog.find({"docType": "review"}):
    print(doc)
```

Retrieve all documents related to a specific book:

```python
for doc in books_catalog.find({"relatedTo": 34538756}):
    print(doc)
```

---

## Summary

We have successfully applied the Single Collection Pattern to consolidate books, users, and reviews into a single collection. This approach:

- Reduces the need for joins and multiple queries
- Improves query performance
- Simplifies data access for related entities

This pattern is especially useful when embedding is not feasible and relationships are complex.

---

## Additional Resources

- [PyMongo Documentation](https://pymongo.readthedocs.io/)
- [MongoDB Aggregation `$merge`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/merge/)
- [MongoDB Schema Design Best Practices](https://www.mongodb.com/docs/manual/core/data-model-design/)

---

## Full Example

The script below is the entire walkthrough in one file — copy-paste it into a Python REPL or a `.py` file after editing the `VM_IP_ADDRESS` constant. It drops all four collections, inserts the source data into `books`, `users`, and `reviews`, runs the three aggregation pipelines to populate `books_catalog`, then demonstrates all three query forms.

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client        = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db            = client["bookstore"]
books         = db["books"]
users         = db["users"]
reviews       = db["reviews"]
books_catalog = db["books_catalog"]

# Reset for a clean run
books.drop()
users.drop()
reviews.drop()
books_catalog.drop()

# 1. Insert source data into separate collections
books.insert_many([
    {
        "_id": 2,
        "product_id": 34538757,
        "product_type": "book",
        "title": "Practical MongoDB Aggregations",
        "description": "The official guide to developing optimal aggregation pipelines with MongoDB 7.0",
        "authors": ["Paul Done"],
        "publisher": "MongoDB Press",
        "language": "English",
        "pages": 312,
        "catalogues": {"isbn10": "1835080642", "isbn13": "978-1835080641"}
    },
    {
        "_id": 1,
        "product_id": 34538756,
        "product_type": "book",
        "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
        "description": "MongoDB explained by MongoDB champions",
        "authors": ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
        "publisher": "O'Reilly",
        "language": "English",
        "pages": 514,
        "catalogues": {"isbn10": "1491954469", "isbn13": "978-1491954461"}
    },
    {
        "_id": 3,
        "product_id": 34538758,
        "product_type": "book",
        "title": "Mastering MongoDB 7.0",
        "description": "Explore the full potential of MongoDB 7.0 with this comprehensive guide.",
        "authors": [
            "Marko Aleksendric",
            "Arek Borucki",
            "Leandro Domingues",
            "Malak Abu Hammad",
            "Elie Hannouch",
            "Rajesh Nair",
            "Rachelle Palmer"
        ],
        "publisher": "MongoDB Press",
        "language": "English",
        "pages": 185,
        "catalogues": {"isbn10": "183546047X", "isbn13": "978-1835460474"}
    }
])

users.insert_many([
    {
        "_id": 111,
        "user_id": 111,
        "name": "Alice",
        "email": "alice@example.com",
        "recent_reviews": [11],
        "purchased_books": [34538756]
    },
    {
        "_id": 222,
        "user_id": 222,
        "name": "Bob",
        "email": "bob@example.com",
        "recent_reviews": [22],
        "purchased_books": [34538757]
    },
    {
        "_id": 333,
        "user_id": 333,
        "name": "Charlie",
        "email": "charlie@example.com",
        "recent_reviews": [33],
        "purchased_books": [34538758]
    }
])

reviews.insert_many([
    {
        "_id": 22,
        "review_id": 22,
        "text": "This book was good.",
        "book_id": 34538757,
        "user_id": 222,
        "rating": 1
    },
    {
        "_id": 11,
        "review_id": 11,
        "text": "This book was pretty good.",
        "book_id": 34538756,
        "user_id": 111,
        "rating": 3
    },
    {
        "_id": 33,
        "review_id": 33,
        "text": "This book was insanely great.",
        "book_id": 34538758,
        "user_id": 333,
        "rating": 5
    }
])

# 2. Run aggregation pipelines to merge into books_catalog
books_pipeline = [
    {
        "$set": {
            "docType": "book",
            "relatedTo": ["$product_id"],
            "book_id": "$product_id"
        }
    },
    {"$unset": ["product_id"]},
    {"$merge": {"into": "books_catalog", "on": "_id"}}
]
books.aggregate(books_pipeline)

users_pipeline = [
    {
        "$set": {
            "docType": "user",
            "relatedTo": "$purchased_books"
        }
    },
    {"$unset": ["purchased_books"]},
    {"$merge": {"into": "books_catalog", "on": "_id"}}
]
users.aggregate(users_pipeline)

reviews_pipeline = [
    {
        "$set": {
            "docType": "review",
            "relatedTo": ["$book_id"]
        }
    },
    {"$unset": ["book_id"]},
    {"$merge": {"into": "books_catalog", "on": "_id"}}
]
reviews.aggregate(reviews_pipeline)

# 3. Query the single collection
print("=== All documents ===")
for doc in books_catalog.find():
    print(doc)

print("\n=== Reviews only ===")
for doc in books_catalog.find({"docType": "review"}):
    print(doc)

print("\n=== Everything related to book 34538756 ===")
for doc in books_catalog.find({"relatedTo": 34538756}):
    print(doc)
```
