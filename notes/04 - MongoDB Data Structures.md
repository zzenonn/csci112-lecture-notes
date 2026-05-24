# MongoDB Data Structures and CRUD Operations

CSCI 112 / 212 - Contemporary Databases

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Lab Data Setup](#lab-data-setup)
  - [Prerequisites](#prerequisites)
  - [Download Lab Datasets](#download-lab-datasets)
  - [Extract and Restore Databases](#extract-and-restore-databases)
  - [Validate Installation](#validate-installation)
  - [Important Notes](#important-notes)
- [MongoDB Basics](#mongodb-basics)
  - [MongoDB IDs](#mongodb-ids)
- [Inserting Documents — mongosh](#inserting-documents--mongosh)
  - [Insert](#insert)
  - [Ordered vs. Unordered Inserts](#ordered-vs-unordered-inserts)
- [Database Drivers](#database-drivers)
  - [What is a Driver?](#what-is-a-driver)
  - [PyMongo Setup](#pymongo-setup)
  - [Connecting to MongoDB from Your Host Machine](#connecting-to-mongodb-from-your-host-machine)
- [CRUD Operations — PyMongo](#crud-operations--pymongo)
  - [Insert (PyMongo)](#insert-pymongo)
  - [Sample Data Setup](#sample-data-setup)
  - [Finding Documents](#finding-documents)
  - [Dot Notation for Nested Fields](#dot-notation-for-nested-fields)
  - [Deleting Documents](#deleting-documents)
- [Projections](#projections)
- [Cursors](#cursors)
  - [Cursor Methods](#cursor-methods)
- [Query Operators](#query-operators)
  - [Comparison Operators](#comparison-operators)
  - [Logical Operators](#logical-operators)
- [Array Query Operators](#array-query-operators)
  - [$elemMatch](#elemmatch)
- [Updating Documents](#updating-documents)
  - [replace_one()](#replace_one)
  - [update_one()](#update_one)
  - [update_many()](#update_many)
- [Other Update Operators](#other-update-operators)
- [Practice Exercises](#practice-exercises)
- [Summary](#summary)

## Overview

This document introduces MongoDB data structures and CRUD operations. We start with the MongoDB shell (`mongosh`) so you can see how data goes in, then introduce **PyMongo** — the official Python driver — which is how application code talks to MongoDB in practice.

> **Setup mental model:** Your Python script runs on your **host machine** (laptop). It connects over the network to the MongoDB server running in your **VM**. The VM's IP address is all you need — you do not need to be inside the VM to run queries.

---

## Objectives

- Understand MongoDB document structure and data types
- Learn how MongoDB generates and uses `_id` fields
- Perform basic CRUD operations using both mongosh and PyMongo
- Use cursor methods: `.sort()`, `.skip()`, `.limit()`
- Understand query operators and projections
- Work with arrays and nested documents

---

## Lab Data Setup

Before starting the exercises, you'll need to restore the lab databases. Follow these steps to download and restore the required datasets.

### Prerequisites

1. **SSH into your MongoDB server:**
   ```bash
   ssh username@192.168.122.61
   ```

2. **Install wget (if not already installed):**
   ```bash
   sudo yum install wget -y
   ```

### Download Lab Datasets

3. **Download the sample database:**
   ```bash
   cd /tmp
   wget https://admu-contempo.s3.ap-southeast-1.amazonaws.com/data/mongodb_sample.zip
   ```

4. **Download the labs database:**
   ```bash
   wget https://admu-contempo.s3.ap-southeast-1.amazonaws.com/data/mongo_data.zip
   ```

### Extract and Restore Databases

5. **Extract the archives:**
   ```bash
   unzip mongodb_sample.zip
   unzip mongo_data.zip
   ```

6. **Restore the sample database:**
   ```bash
   mongorestore --db sample mongodb_sample/sample/
   ```

7. **Restore the labs database:**
   ```bash
   mongorestore --db labs sample/
   ```

### Validate Installation

8. **Connect to MongoDB and verify the databases:**
   ```bash
   mongosh
   ```

   In the MongoDB shell:
   ```js
   // Check available databases
   show dbs
   
   // Verify sample database collections
   use sample
   show collections
   db.inspections.countDocuments()
   
   // Verify labs database collections
   use labs
   show collections
   db.movies.countDocuments()
   ```

### Important Notes

- **Database References in Labs:** Throughout the lab exercises, when we refer to collections, they may be located in either the `sample` or `labs` database. Each lab will specify which database to use.
- **Collection Examples:** 
  - `sample` database contains: inspections, stories, grades, companies, zips, posts
  - `labs` database contains: movies and other lab-specific collections
- The lab instructions will clearly indicate which database and collection to use for each exercise.

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

## Inserting Documents — mongosh

> **Shell context:** All `db.*` commands below run inside **mongosh**, not the Linux shell.
> Connect first from your Linux shell, then select a database:
> ```bash
> # Linux shell
> mongosh --host <ip_address>
> ```
> ```js
> // mongosh
> use movies
> ```
> Your prompt will change to `movies>` once the database is selected.

### Insert

```js
// mongosh — movies>

// Insert a single document (_id auto-generated)
db.movies.insertOne({ "title": "Jaws" })

// Insert multiple documents
db.movies.insertMany([
  { "title": "Batman", "category": ["action", "adventure"] },
  { "title": "Godzilla", "category": ["action", "sci-fi"] },
  { "title": "Home Alone", "category": ["family", "comedy"] }
])
```

If `_id` is not specified, MongoDB will generate one automatically. Providing a duplicate `_id` raises a `DuplicateKeyError`.

### Ordered vs. Unordered Inserts

- **Ordered** (default): Stops on first error
- **Unordered**: Continues inserting remaining documents despite errors

```js
// mongosh — movies>
db.movies.insertMany([...], { ordered: false })
```

---

## Database Drivers

### What is a Driver?

A **database driver** is a library that allows an application to communicate with a database server using the database's native protocol. Without a driver, your Python code would have no way to speak to MongoDB.

MongoDB maintains official drivers for every major language. The Python driver is called **PyMongo**.

```
┌─────────────────────────┐          ┌──────────────────────────┐
│   Host Machine (laptop) │          │   VM (MongoDB server)    │
│                         │          │                          │
│   Python script         │  ──────► │   mongod (port 27017)    │
│   + PyMongo driver      │  TCP/IP  │                          │
└─────────────────────────┘          └──────────────────────────┘
```

This is how most real applications work — the database server runs separately from the application code, and a driver bridges them. `mongosh` is a convenience shell for admin tasks; application code always uses a driver.

### PyMongo Setup

Install PyMongo on your **host machine** (not inside the VM). Always use a virtual environment so your project dependencies stay isolated.

```bash
# Create a hidden virtual environment
python -m venv .venv

# Activate (Mac/Linux)
source .venv/bin/activate

# Activate (Windows)
.venv\Scripts\activate

# Install PyMongo inside the active environment
pip install pymongo
```

Your shell prompt will show `(.venv)` when the environment is active. Always activate it before running your scripts.

### Connecting to MongoDB from Your Host Machine

```python
from pymongo import MongoClient

# Replace with your VM's IP address
VM_IP_ADDRESS = "192.168.1.100"

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")

# Select database and collection
db = client["movies"]
movies_collection = db["movies"]
```

> Make sure your VM has `bindIp: 0.0.0.0` in `/etc/mongod.conf` and port 27017 open in the firewall (covered in Module 03).

---

## CRUD Operations — PyMongo

From here on, all examples use PyMongo. The query operator syntax (`$gt`, `$or`, etc.) is identical to mongosh — you just write them as Python dicts instead of JavaScript objects.

### Insert (PyMongo)

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db = client["movies"]
movies_collection = db["movies"]

# Insert a single document
result = movies_collection.insert_one({ "title": "Jaws" })
print(result.inserted_id)  # auto-generated _id
```

```python
# Insert multiple documents
movies_collection.insert_many([
  { "title": "Batman", "category": ["action", "adventure"] },
  { "title": "Godzilla", "category": ["action", "sci-fi"] },
  { "title": "Home Alone", "category": ["family", "comedy"] }
])
```

```python
# Duplicate _id raises DuplicateKeyError
movies_collection.insert_one({ "_id": "Star Wars" })  # Success
movies_collection.insert_one({ "_id": "Star Wars" })  # DuplicateKeyError
```

For unordered inserts (continue past duplicates):

```python
from pymongo.errors import BulkWriteError

docs = [
  { "_id": "Batman", "year": 1989 },
  { "_id": "Home Alone", "year": 1990 },
  { "_id": "Ghostbusters", "year": 1984 },
  { "_id": "Ghostbusters", "year": 1984 },   # duplicate
  { "_id": "Lord of the Rings", "year": 2001 }
]

try:
    movies_collection.insert_many(docs, ordered=False)
except BulkWriteError as e:
    print(e.details)
```

### Sample Data Setup

Before continuing, insert the full sample dataset that the remaining examples depend on. Run this from your host machine:

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
movies_collection = client["movies"]["movies"]

movies_collection.drop()
movies_collection.insert_many([
  {
    "title": "Batman",
    "category": ["action", "adventure"],
    "imdb_rating": 7.6,
    "box_office": { "budget": 35, "gross": 70 },
    "rotten_tomatoes": 7.1
  },
  {
    "title": "Godzilla",
    "category": ["action", "adventure", "sci-fi"],
    "imdb_rating": 6.6,
    "box_office": { "budget": 70, "gross": 20 },
    "rotten_tomatoes": 8.0
  },
  {
    "title": "Home Alone",
    "category": ["family", "comedy"],
    "imdb_rating": 7.4,
    "box_office": { "budget": 10, "gross": 15 },
    "rotten_tomatoes": 6.3
  }
])
```

### Finding Documents

```python
# Returns all documents
for doc in movies_collection.find():
    print(doc)

# Filter by field value
doc = movies_collection.find_one({ "title": "Batman" })

# Filter by array element (any match)
for doc in movies_collection.find({ "category": "family" }):
    print(doc)

# Filter by exact array
for doc in movies_collection.find({ "category": ["family", "comedy"] }):
    print(doc)
```

### Dot Notation for Nested Fields

```python
# Query nested field using dot notation string
for doc in movies_collection.find({ "box_office.gross": { "$gt": 50 } }):
    print(doc)
```

### Deleting Documents

```python
# Delete first matching document
movies_collection.delete_one({ "category": "action" })

# Delete all matching documents
result = movies_collection.delete_many({ "category": "action" })
print(result.deleted_count)

# Drop the entire collection
movies_collection.drop()

# Drop the entire database
client.drop_database("movies")
```

---

## Projections

Specify which fields to include (`1`) or exclude (`0`) in the result.

```python
# Include only title (plus _id by default)
for doc in movies_collection.find({}, { "title": 1 }):
    print(doc)

# Exclude title
for doc in movies_collection.find({}, { "title": 0 }):
    print(doc)

# Exclude _id explicitly
for doc in movies_collection.find({}, { "title": 1, "_id": 0 }):
    print(doc)
```

> You cannot mix include and exclude in the same projection, except for `_id`.

---

## Cursors

`.find()` returns a **cursor** — an iterator over the result set. In PyMongo, you iterate over it with a `for` loop or convert it to a list.

### Cursor Methods

```python
# Sort: 1 = ascending, -1 = descending
for doc in movies_collection.find().sort("imdb_rating", -1):
    print(doc)

# Limit and skip
for doc in movies_collection.find().skip(5).limit(5):
    print(doc)

# Chained
for doc in movies_collection.find().sort("title", 1).skip(5).limit(5):
    print(doc)
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

Query operator syntax is identical to mongosh — just Python dicts:

```python
for doc in movies_collection.find({ "imdb_rating": { "$gte": 7 } }):
    print(doc)

for doc in movies_collection.find({ "category": { "$ne": "family" } }):
    print(doc)

for doc in movies_collection.find({ "title": { "$in": ["Batman", "Godzilla"] } }):
    print(doc)

for doc in movies_collection.find({ "box_office.gross": { "$gt": 50 } }):
    print(doc)
```

### Logical Operators

| Operator | Description |
|----------|-------------|
| $or      | Match any |
| $and     | Match all |
| $not     | Negate condition |
| $nor     | Match none |

```python
for doc in movies_collection.find({ "$or": [
    { "category": "sci-fi" },
    { "imdb_rating": { "$gte": 7 } }
] }):
    print(doc)

for doc in movies_collection.find({ "$or": [
    { "category": "sci-fi", "imdb_rating": { "$gte": 8 } },
    { "category": "family", "imdb_rating": { "$gte": 7 } }
] }):
    print(doc)
```

---

## Array Query Operators

| Operator    | Description |
|-------------|-------------|
| $all        | All values must be present |
| $size       | Array must have exact length |
| $elemMatch  | Match at least one element with all conditions |

```python
for doc in movies_collection.find({ "category": { "$size": 3 } }):
    print(doc)

for doc in movies_collection.find({ "category": { "$all": ["sci-fi", "action"] } }):
    print(doc)
```

### $elemMatch

`$elemMatch` matches documents where at least one array element satisfies all conditions simultaneously. First set up the data:

```python
movies_collection.drop()
movies_collection.insert_many([
  {
    "title": "Raiders of the Lost Ark",
    "filming_locations": [
      { "city": "Los Angeles", "state": "CA", "country": "USA" },
      { "city": "Rome", "state": "Lazio", "country": "Italy" },
      { "city": "Florence", "state": "SC", "country": "USA" }
    ]
  },
  {
    "title": "Hannibal",
    "filming_locations": [
      { "city": "Washington, D.C.", "state": "WA", "country": "USA" },
      { "city": "Florence", "state": "Lazio", "country": "Italy" },
      { "city": "Richmond", "state": "VA", "country": "USA" }
    ]
  }
])

# Finds only Hannibal — Florence in Italy, not South Carolina
for doc in movies_collection.find({
    "filming_locations": {
        "$elemMatch": { "city": "Florence", "country": "Italy" }
    }
}):
    print(doc)
```

---

## Updating Documents

### replace_one()

Replaces the entire document (except `_id`). Use with caution — all fields not in the replacement are removed.

```python
movies_collection.replace_one(
    { "title": "Batman" },
    { "imdb_rating": 7.7 }   # Batman now only has _id and imdb_rating
)
```

### update_one()

Modifies specific fields without replacing the whole document.

```python
movies_collection.update_one(
    { "title": "Batman" },
    { "$set": { "imdb_rating": 7.7 } }
)

movies_collection.update_one(
    { "title": "Godzilla" },
    { "$set": { "box_office.budget": 1 } }
)
```

### update_many()

Updates all matching documents.

```python
movies_collection.update_many({}, { "$set": { "sequels": 0 } })
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

```python
# Increment Home Alone's budget by 5
movies_collection.update_one(
    { "title": "Home Alone" },
    { "$inc": { "box_office.budget": 5 } }
)

# Set current date on all documents
movies_collection.update_many({}, {
    "$currentDate": { "release_date": { "$type": "date" } }
})
```

---

## Practice Exercises

Write Python scripts using PyMongo for each exercise. Connect to your VM's MongoDB instance.

1. Find all sci-fi movies.
2. Find either sci-fi or comedy movies.
3. Find all movies that made a profit (gross > budget):
```python
for doc in movies_collection.find({ "$expr": { "$gt": ["$box_office.gross", "$box_office.budget"] } }):
    print(doc)
```
4. Increment Batman's IMDB rating by 1.
5. Increment Home Alone's box office budget by 5.
6. Remove Home Alone's box office budget field.
7. Multiply Batman's IMDB rating by 4.
8. Rename the field `box_office.budget` to `box_office.budget_in_millions`.

---

## Troubleshooting FAQ

### Q: `ModuleNotFoundError: No module named 'pymongo'` — I already ran `pip install pymongo`

You installed pymongo in a different Python environment than the one running your script. Always activate your virtual environment first:

```bash
source .venv/bin/activate   # Mac/Linux
.venv\Scripts\activate      # Windows
pip install pymongo        # install inside the active .venv
python your_script.py      # now runs with the correct environment
```

If you're using VS Code, select the correct interpreter: `Ctrl+Shift+P` → **Python: Select Interpreter** → choose the one inside your `.venv/` folder.

---

### Q: My script runs but VS Code / Jupyter can't find pymongo

Same root cause as above — VS Code is using a different Python interpreter than your .venv. Set the interpreter to the one inside your `.venv/` folder (see above). For Jupyter notebooks, select the .venv kernel from the kernel picker in the top right.

---

### Q: `ServerSelectionTimeoutError` / connection refused — can't connect to MongoDB

Work through this checklist:

1. **Is `mongod` running on the VM?**
   ```bash
   # Inside the VM
   sudo systemctl status mongod
   # If not running:
   sudo systemctl start mongod
   ```

2. **Is `bindIp` set to `0.0.0.0`?**
   ```bash
   sudo nano /etc/mongod.conf
   # Look for: bindIp: 127.0.0.1
   # Change to: bindIp: 0.0.0.0
   sudo systemctl restart mongod
   ```

3. **Is port 27017 open in the firewall?**
   ```bash
   sudo firewall-cmd --permanent --add-port=27017/tcp
   sudo firewall-cmd --reload
   ```

4. **Are you using the correct IP address?** Run `ip a` inside the VM and use the IP of your network interface (usually `eth0` or `enp0s3`).

---

### Q: `DuplicateKeyError` when inserting

You're inserting a document with an `_id` that already exists in the collection. Either omit `_id` (MongoDB will generate one), use a different value, or drop the collection first with `movies_collection.drop()`.

---

### Q: I inserted data in Python but can't see it in mongosh (or vice versa)

Both are connected to the same server, so the data should be the same. Check:
- Are you looking at the same **database** and **collection** name?
- In mongosh, run `use <database_name>` before querying.
- In Python, confirm `client["db_name"]["collection_name"]` matches what you expect.

---

### Q: `find()` returns an empty cursor even though I just inserted data

Most likely a wrong database or collection name. Python string comparisons are case-sensitive — `"Movies"` and `"movies"` are different collections. Double-check both names.

---

## Summary

This guide covers MongoDB data structures and CRUD operations. `mongosh` gives you a direct view into the database — useful for learning the syntax and quick admin tasks. PyMongo is the recommended approach for all application code: it runs on your host machine and connects to the VM over the network.

For further reading:
- PyMongo documentation: https://www.mongodb.com/docs/languages/python/pymongo-driver/current/
- MongoDB query operators: https://www.mongodb.com/docs/manual/reference/operator/query/
