---
theme: discs
title: "MongoDB Data Structures"
highlighter: shiki
layout: cover
codeCopy: true
---

MongoDB Data Structures

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Understand MongoDB **document structure** and data types
2. Connect to MongoDB using a **Python driver** (PyMongo)
3. Perform **CRUD operations** from a Python script
4. Use **query operators**, **projections**, and **cursor methods**
5. Work with **arrays** and **nested documents**

---
layout: section
---

# MongoDB Basics

---

# MongoDB IDs

Every document has a unique `_id` field — generated automatically if not provided

<div style="display:flex; justify-content:center; margin-top:1.5rem;">
  <div style="display:flex; gap:0; font-size:0.85rem; font-family:monospace; text-align:center;">
    <div style="background:#bfdbfe; border:1.5px solid #3b82f6; padding:0.5rem 0.9rem; border-radius:6px 0 0 6px; letter-spacing:0.05em;">66eaf030</div>
    <div style="background:#bbf7d0; border:1.5px solid #16a34a; border-left:none; padding:0.5rem 0.9rem; letter-spacing:0.05em;">ff3099</div>
    <div style="background:#fef08a; border:1.5px solid #ca8a04; border-left:none; padding:0.5rem 0.9rem; letter-spacing:0.05em;">f18b</div>
    <div style="background:#fecaca; border:1.5px solid #dc2626; border-left:none; padding:0.5rem 0.9rem; border-radius:0 6px 6px 0; letter-spacing:0.05em;">c2d30c</div>
  </div>
</div>

<div style="display:flex; justify-content:center; margin-top:0.5rem;">
  <div style="display:flex; gap:0; font-size:0.78rem; text-align:center;">
    <div style="background:#bfdbfe; border:1.5px solid #3b82f6; padding:0.4rem 0.9rem; border-radius:6px 0 0 6px;">
      <div style="font-weight:700;">Timestamp</div>
      <div style="color:#555;">4 bytes · seconds since epoch</div>
    </div>
    <div style="background:#bbf7d0; border:1.5px solid #16a34a; border-left:none; padding:0.4rem 0.9rem;">
      <div style="font-weight:700;">Machine ID</div>
      <div style="color:#555;">3 bytes</div>
    </div>
    <div style="background:#fef08a; border:1.5px solid #ca8a04; border-left:none; padding:0.4rem 0.9rem;">
      <div style="font-weight:700;">Process ID</div>
      <div style="color:#555;">2 bytes</div>
    </div>
    <div style="background:#fecaca; border:1.5px solid #dc2626; border-left:none; padding:0.4rem 0.9rem; border-radius:0 6px 6px 0;">
      <div style="font-weight:700;">Counter</div>
      <div style="color:#555;">3 bytes</div>
    </div>
  </div>
</div>

<div style="margin-top:1rem; font-size:0.85rem; color:#555; text-align:center;">
  <code>0x66eaf030</code> → <strong>1726718000</strong> seconds since Unix epoch → <strong>Sep 19, 2024</strong>
</div>

---
layout: section
---

# CRUD Operations

---

# Insert

```javascript
// Connect to the MongoDB shell
mongosh "mongodb://<vm_ip_address>/movies"

// Insert a single document (_id auto-generated)
db.movies.insertOne({ "title": "Jaws" })

// Insert multiple documents
db.movies.insertMany([
  { "title": "Batman", "category": ["action", "adventure"] },
  { "title": "Godzilla", "category": ["action", "sci-fi"] },
  { "title": "Home Alone", "category": ["family", "comedy"] }
])
```

<div style="margin-top:0.5rem; font-size:0.82rem; color:#666;">
  Ordered vs. unordered inserts, duplicate key handling: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/04%20-%20MongoDB%20Data%20Structures.md">notes/04 — MongoDB Data Structures</a>
</div>

---
layout: section
---

# Database Drivers

How your code talks to MongoDB

---

# What is a Driver?

In production, applications never use the shell — they use a **driver**: a library that speaks MongoDB's native protocol.

<div style="display:flex; align-items:center; justify-content:center; gap:1.5rem; margin-top:1.5rem; font-size:0.9rem;">
  <div style="text-align:center; border:1.5px solid #cbd5e1; border-radius:8px; padding:0.75rem 1.25rem; background:#f8fafc;">
    <div style="font-weight:700;">Host Machine</div>
    <div style="font-size:0.8rem; color:#666; margin-top:0.2rem;">Python script + PyMongo</div>
    <div style="font-size:0.75rem; color:#aaa; margin-top:0.2rem;">your laptop</div>
  </div>
  <div style="display:flex; flex-direction:column; align-items:center; gap:0.2rem; color:#64748b; font-size:0.8rem;">
    <div>TCP/IP</div>
    <div style="font-size:1.2rem;">→</div>
    <div>port 27017</div>
  </div>
  <div style="text-align:center; border:1.5px solid #f97316; border-radius:8px; padding:0.75rem 1.25rem; background:#fff7ed;">
    <div style="font-weight:700;">VM</div>
    <div style="font-size:0.8rem; color:#666; margin-top:0.2rem;">mongod running</div>
    <div style="font-size:0.75rem; color:#aaa; margin-top:0.2rem;">192.168.x.x</div>
  </div>
</div>

<div style="margin-top:1.5rem; font-size:0.9rem; color:#444;">
  You do <strong>not</strong> need to be inside the VM to run queries — your Python script connects over the network from your laptop.
</div>

---

# PyMongo Setup

Run these on your **host machine** (not inside the VM):

```bash
# Create a hidden virtual environment
python -m venv .venv

# Activate (Mac / Linux)
source .venv/bin/activate

# Activate (Windows)
.venv\Scripts\activate

# Install PyMongo inside the active environment
pip install pymongo
```

Your prompt will show `(.venv)` when the environment is active. **Always activate before running your scripts.**

---

# Connecting to MongoDB

```python
from pymongo import MongoClient

# Connect to your VM's MongoDB instance
client = MongoClient("mongodb://<vm_ip_address>:27017/")

# Select database and collection
db = client["movies"]
collection = db["movies"]
```

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  Make sure your VM has <code>bindIp: 0.0.0.0</code> in <code>/etc/mongod.conf</code> and port 27017 open — covered in Module 03.
</div>

---

# Insert (PyMongo)

```python
from pymongo import MongoClient

client = MongoClient("mongodb://<vm_ip_address>:27017/")
collection = client["movies"]["movies"]

# Insert a single document (_id auto-generated)
result = collection.insert_one({ "title": "Jaws" })
print(result.inserted_id)

# Insert multiple documents
collection.insert_many([
  { "title": "Batman", "category": ["action", "adventure"] },
  { "title": "Godzilla", "category": ["action", "sci-fi"] },
  { "title": "Home Alone", "category": ["family", "comedy"] }
])
```

---

# Sample Data Setup

Before running the following examples, insert the sample `movies` collection from the notes.

<div style="margin-top:1.5rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:1rem 1.25rem; border-radius:0 6px 6px 0; font-size:0.95rem;">
  Run the <strong>Sample Data Setup</strong> script in <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/04%20-%20MongoDB%20Data%20Structures.md#sample-data-setup">notes/04 — MongoDB Data Structures</a> before proceeding.
</div>

---

# Find

```python
# All documents
for doc in collection.find():
    print(doc)

# Filter by field value
doc = collection.find_one({ "title": "Batman" })

# Filter by array element (any match)
for doc in collection.find({ "category": "family" }):
    print(doc)

# Nested field using dot notation
for doc in collection.find({ "box_office.gross": { "$gt": 50 } }):
    print(doc)
```

---

# Delete

```python
# Delete first matching document
collection.delete_one({ "category": "action" })

# Delete all matching documents
result = collection.delete_many({ "category": "action" })
print(result.deleted_count)

# Drop the entire collection
collection.drop()

# Drop the entire database
client.drop_database("movies")
```

---

# Update

```python
# replace_one — replaces entire document (keeps _id)
collection.replace_one(
    { "title": "Batman" },
    { "imdb_rating": 7.7 }
)

# update_one — modifies specific fields only
collection.update_one(
    { "title": "Batman" },
    { "$set": { "imdb_rating": 7.7 } }
)

# update_many — update all matching documents
collection.update_many({}, { "$set": { "sequels": 0 } })
```

---

# Update Operators

<div style="display:flex; gap:2rem; align-items:flex-start; margin-top:0.5rem;">
<div style="flex:1;">
<table style="font-size:0.85rem; border-collapse:collapse; width:100%;">
  <thead><tr style="border-bottom:2px solid #e2e8f0;">
    <th style="text-align:left; padding:0.3rem 0.5rem;">Operator</th>
    <th style="text-align:left; padding:0.3rem 0.5rem;">Description</th>
  </tr></thead>
  <tbody style="line-height:1.6;">
    <tr><td style="padding:0.2rem 0.5rem;"><code>$set</code></td><td style="padding:0.2rem 0.5rem;">Set a field value</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$unset</code></td><td style="padding:0.2rem 0.5rem;">Remove a field</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$inc</code></td><td style="padding:0.2rem 0.5rem;">Increment a numeric field</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$mul</code></td><td style="padding:0.2rem 0.5rem;">Multiply a numeric field</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$rename</code></td><td style="padding:0.2rem 0.5rem;">Rename a field</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$min</code> / <code>$max</code></td><td style="padding:0.2rem 0.5rem;">Set if less / greater than current</td></tr>
    <tr><td style="padding:0.2rem 0.5rem;"><code>$currentDate</code></td><td style="padding:0.2rem 0.5rem;">Set to current date/timestamp</td></tr>
  </tbody>
</table>
</div>
<div style="flex:1;">

```python
# Increment Home Alone's budget by 5
collection.update_one(
    { "title": "Home Alone" },
    { "$inc": { "box_office.budget": 5 } }
)

# Set current date on all docs
collection.update_many({}, {
    "$currentDate": {
        "release_date": { "$type": "date" }
    }
})
```

</div>
</div>

---
layout: section
---

# Querying

---

# Projections

Control which fields are returned — `1` to include, `0` to exclude:

```python
# Include only title (plus _id by default)
for doc in collection.find({}, { "title": 1 }):
    print(doc)

# Exclude _id explicitly
for doc in collection.find({}, { "title": 1, "_id": 0 }):
    print(doc)

# Exclude a field
for doc in collection.find({}, { "rotten_tomatoes": 0 }):
    print(doc)
```

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  You cannot mix include and exclude in the same projection (except for <code>_id</code>).
</div>

---

# Cursors

`.find()` returns a **cursor** — iterate over it with a `for` loop or chain methods:

```python
# Sort: 1 = ascending, -1 = descending
for doc in collection.find().sort("imdb_rating", -1):
    print(doc)

# Limit and skip
for doc in collection.find().skip(0).limit(5):
    print(doc)

# Chained
for doc in collection.find().sort("title", 1).skip(5).limit(5):
    print(doc)
```

---

# Comparison Operators

| Operator | Meaning |
|---|---|
| `$lt` / `$lte` | Less than / less than or equal |
| `$gt` / `$gte` | Greater than / greater than or equal |
| `$eq` / `$ne` | Equal / not equal |
| `$in` / `$nin` | In array / not in array |

```python
collection.find({ "imdb_rating": { "$gte": 7 } })
collection.find({ "box_office.gross": { "$gt": 50 } })
collection.find({ "title": { "$in": ["Batman", "Godzilla"] } })
```

---

# Logical Operators

| Operator | Meaning |
|---|---|
| `$or` | Match any condition |
| `$and` | Match all conditions |
| `$not` | Negate a condition |
| `$nor` | Match none |

```python
for doc in collection.find({ "$or": [
    { "category": "sci-fi" },
    { "imdb_rating": { "$gte": 7 } }
] }):
    print(doc)
```

---

# Array Operators

| Operator | Description |
|---|---|
| `$all` | Array must contain all specified values |
| `$size` | Array must have exact length |
| `$elemMatch` | At least one element matches all conditions |

```python
collection.find({ "category": { "$size": 3 } })
collection.find({ "category": { "$all": ["sci-fi", "action"] } })

# $elemMatch on nested documents
collection.find({
    "filming_locations": {
        "$elemMatch": { "city": "Florence", "country": "Italy" }
    }
})
```

<div style="margin-top:0.75rem; font-size:0.85rem; color:#666;">
  Full sample data and $elemMatch setup: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/04%20-%20MongoDB%20Data%20Structures.md">notes/04 — MongoDB Data Structures</a>
</div>

---
layout: end
---

# Thank You
