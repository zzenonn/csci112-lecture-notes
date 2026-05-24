# MongoDB Data Modeling: Inheritance Pattern

**Course:** CSCI 112 / 212 - Contemporary Databases
**Topic:** MongoDB Inheritance Pattern

---

## Overview

In this module, we explore the **Inheritance Pattern** in MongoDB data modeling. This pattern allows us to store related but distinct document types within a single collection by leveraging polymorphism — a key feature of MongoDB's flexible schema design.

All examples use **PyMongo** running on your laptop, connecting to `mongod` on a VM.

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

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db     = client["bookstore"]
books  = db["books"]
```

The `books` variable is the **collection handle** — every example below calls methods on it.

---

## Applying the Inheritance Pattern to Existing Data

In real-world applications, you may need to transform an existing collection to follow the Inheritance Pattern. MongoDB's Aggregation Framework is a powerful tool for this task.

---

## Step-by-Step Example

We will apply the Inheritance Pattern to a collection named `books`. This collection contains three documents representing different book types, each with slightly different structures.

### Step 1: Insert Sample Documents

First, insert the sample documents into our collection:

```python
books.insert_many([
    {
        "_id": 1,
        "product_id": 34538756,
        "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
        "details": "MongoDB explained by MongoDB champions",
        "authors": ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
        "publisher": "O'Reilly",
        "language": "English",
        "pages": 514,
        "catalogues": {"isbn10": "1491954469", "isbn13": "978-1491954461"}
    },
    {
        "_id": 2,
        "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
        "desc": "MongoDB explained by MongoDB champions",
        "authors": ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
        "publisher": "O'Reilly",
        "language": "English",
        "eformats": {"epub": {"pages": 774}, "pdf": {"pages": 502}},
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

```python
for doc in books.find():
    print(doc)
```

Notice the inconsistencies: `desc` vs `details` vs `description`, `author` (string) vs `authors` (array or comma-separated string), missing `product_id` on document 2. The pattern migration will normalize these.

---

## Step 3: Update Documents to Follow the Inheritance Pattern

We will update each document to:

- Add a `product_type` field.
- Standardize field names (`desc` → `description`, `details` → `description`).
- Convert `author` or `authors` to an array of strings.
- Add missing `product_id` fields where necessary.

### Option A: Manual Updates

You can manually update each document using `replace_one`:

#### Document 1: Printed Book

```python
books.replace_one(
    {"_id": 1},
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
    }
)
```

#### Document 2: Ebook

```python
books.replace_one(
    {"_id": 2},
    {
        "_id": 2,
        "product_id": 44538756,
        "product_type": "ebook",
        "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
        "description": "MongoDB explained by MongoDB champions",
        "authors": ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
        "publisher": "O'Reilly",
        "language": "English",
        "eformats": {"epub": {"pages": 774}, "pdf": {"pages": 502}},
        "isbn10": "1491954469"
    }
)
```

#### Document 3: Audiobook

```python
books.replace_one(
    {"_id": 3},
    {
        "_id": 3,
        "product_id": 54538756,
        "product_type": "audiobook",
        "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
        "description": "The complete book of MongoDB by its employees",
        "authors": ["Eoin Brazil"],
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

```python
apply_inheritance_pattern_pipeline = [
    {
        "$project": {
            "_id": "$_id",
            "product_id":   {"$ifNull": ["$product_id", 0]},
            "product_type": {"$ifNull": ["$product_type", "Unspecified"]},
            "title": "$title",
            "description": {
                "$ifNull": ["$desc", "$description", "$details", "Unspecified"]
            },
            "authors": {
                "$cond": {
                    "if":   {"$isArray": "$authors"},
                    "then": "$authors",
                    "else": {
                        "$cond": {
                            "if":   {"$isArray": "$author"},
                            "then": "$author",
                            "else": [{"$ifNull": ["$author", "Unspecified"]}]
                        }
                    }
                }
            },
            "publisher": "$publisher",
            "language":  "$language",
            "pages":      "$pages",
            "catalogues": "$catalogues",
            "eformats":   "$eformats",
            "isbn10":     "$isbn10",
            "isbn13":     "$isbn13",
            "narrator":       "$narrator",
            "length_minutes": "$length_minutes"
        }
    },
    {
        "$merge": {
            "into": "books",
            "on": "_id",
            "whenMatched": "replace",
            "whenNotMatched": "discard"
        }
    }
]

books.aggregate(apply_inheritance_pattern_pipeline)
```

#### Update Audiobook Product Types

```python
cleanup_audiobook_pipeline = [
    {"$match": {
        "$and": [
            {"product_type": "Unspecified"},
            {"length_minutes": {"$gte": 0}}
        ]
    }},
    {"$set": {"product_type": "audiobook"}},
    {"$merge": {
        "into": "books",
        "on": "_id",
        "whenMatched": "replace",
        "whenNotMatched": "discard"
    }}
]

books.aggregate(cleanup_audiobook_pipeline)
```

#### Update Book Product Types

```python
cleanup_book_pipeline = [
    {"$match": {
        "$and": [
            {"product_type": "Unspecified"},
            {"pages": {"$gte": 0}},
            {"catalogues": {"$exists": True}}
        ]
    }},
    {"$set": {"product_type": "book"}},
    {"$merge": {
        "into": "books",
        "on": "_id",
        "whenMatched": "replace",
        "whenNotMatched": "discard"
    }}
]

books.aggregate(cleanup_book_pipeline)
```

#### Update Ebook Product Types

```python
cleanup_ebook_pipeline = [
    {"$match": {
        "$and": [
            {"product_type": "Unspecified"},
            {"eformats": {"$exists": True}}
        ]
    }},
    {"$set": {"product_type": "ebook"}},
    {"$merge": {
        "into": "books",
        "on": "_id",
        "whenMatched": "replace",
        "whenNotMatched": "discard"
    }}
]

books.aggregate(cleanup_ebook_pipeline)
```

---

## Step 4: Verify the Updates

```python
for doc in books.find():
    print(doc)
```

Expected output (after Option B — aggregation pipeline only):

```json
[
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
    "_id": 2,
    "product_id": 0,
    "product_type": "ebook",
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "description": "MongoDB explained by MongoDB champions",
    "authors": ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
    "publisher": "O'Reilly",
    "language": "English",
    "eformats": {"epub": {"pages": 774}, "pdf": {"pages": 502}},
    "isbn10": "1491954469"
  },
  {
    "_id": 3,
    "product_id": 54538756,
    "product_type": "audiobook",
    "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
    "description": "The complete book of MongoDB by its employees",
    "authors": ["Eoin Brazil"],
    "narrator": "Eoin Brazil",
    "publisher": "O'Reilly",
    "language": "English",
    "length_minutes": 1200
  }
]
```

> **Note:** Doc 2's `product_id` is `0` because it was missing from the original document — the pipeline's `$ifNull` defaults it to `0`. The manual Option A approach assigns it `44538756` explicitly. If you need real product IDs for all documents, use Option A or add an `$addFields` stage to set the correct value.

Query a specific type:

```python
for doc in books.find({"product_type": "audiobook"}):
    print(doc)
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
- [PyMongo Documentation](https://pymongo.readthedocs.io/)

For further questions or assistance, please reach out during office hours or post in the course discussion forum.

---

## Full Example

The script below is the entire walkthrough in one file — copy-paste it into a Python REPL or a `.py` file after editing the `VM_IP_ADDRESS` constant. It drops `books`, inserts the three messy sample documents, runs the aggregation-based migration (Pass 1 normalizes fields; Pass 2 sets `product_type` per subtype), and prints the final, normalized documents.

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db     = client["bookstore"]
books  = db["books"]

# Reset for a clean run
books.drop()

# 1. Insert the three messy sample documents
books.insert_many([
    {
        "_id": 1,
        "product_id": 34538756,
        "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
        "details": "MongoDB explained by MongoDB champions",
        "authors": ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
        "publisher": "O'Reilly",
        "language": "English",
        "pages": 514,
        "catalogues": {"isbn10": "1491954469", "isbn13": "978-1491954461"}
    },
    {
        "_id": 2,
        "title": "MongoDB: The Definitive Guide: Powerful and Scalable Data Storage",
        "desc": "MongoDB explained by MongoDB champions",
        "authors": ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
        "publisher": "O'Reilly",
        "language": "English",
        "eformats": {"epub": {"pages": 774}, "pdf": {"pages": 502}},
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

# 2. Pass 1 — normalize field names with $project + $ifNull, write back via $merge
apply_inheritance_pattern_pipeline = [
    {
        "$project": {
            "_id": "$_id",
            "product_id":   {"$ifNull": ["$product_id", 0]},
            "product_type": {"$ifNull": ["$product_type", "Unspecified"]},
            "title": "$title",
            "description": {
                "$ifNull": ["$desc", "$description", "$details", "Unspecified"]
            },
            "authors": {
                "$cond": {
                    "if":   {"$isArray": "$authors"},
                    "then": "$authors",
                    "else": {
                        "$cond": {
                            "if":   {"$isArray": "$author"},
                            "then": "$author",
                            "else": [{"$ifNull": ["$author", "Unspecified"]}]
                        }
                    }
                }
            },
            "publisher": "$publisher",
            "language":  "$language",
            "pages":      "$pages",
            "catalogues": "$catalogues",
            "eformats":   "$eformats",
            "isbn10":     "$isbn10",
            "isbn13":     "$isbn13",
            "narrator":       "$narrator",
            "length_minutes": "$length_minutes"
        }
    },
    {
        "$merge": {
            "into": "books",
            "on": "_id",
            "whenMatched": "replace",
            "whenNotMatched": "discard"
        }
    }
]
books.aggregate(apply_inheritance_pattern_pipeline)

# 3. Pass 2 — classify each document by a field unique to its subtype
cleanup_pipelines = {
    "audiobook": [
        {"$match": {"$and": [{"product_type": "Unspecified"},
                              {"length_minutes": {"$gte": 0}}]}},
        {"$set": {"product_type": "audiobook"}},
        {"$merge": {"into": "books", "on": "_id",
                     "whenMatched": "replace", "whenNotMatched": "discard"}}
    ],
    "book": [
        {"$match": {"$and": [{"product_type": "Unspecified"},
                              {"pages": {"$gte": 0}},
                              {"catalogues": {"$exists": True}}]}},
        {"$set": {"product_type": "book"}},
        {"$merge": {"into": "books", "on": "_id",
                     "whenMatched": "replace", "whenNotMatched": "discard"}}
    ],
    "ebook": [
        {"$match": {"$and": [{"product_type": "Unspecified"},
                              {"eformats": {"$exists": True}}]}},
        {"$set": {"product_type": "ebook"}},
        {"$merge": {"into": "books", "on": "_id",
                     "whenMatched": "replace", "whenNotMatched": "discard"}}
    ],
}
for pipeline in cleanup_pipelines.values():
    books.aggregate(pipeline)

# 4. Verify
for doc in books.find():
    print(doc)
```

After running, every document carries a normalized `description`, an `authors` array, and a `product_type` discriminator (`book`, `ebook`, or `audiobook`) — ready to use with the Computed and Approximation patterns in [notes/8b](8b%20-%20MongoDB%20Data%20Modeling%20Computed%20and%20Approximation%20Pattern.md).
