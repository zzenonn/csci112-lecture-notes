# Lab 2: MongoDB Query Operations

## Objective
Practice MongoDB query operations including find and update using various collections in the `labs` database. This lab focuses on applying query operators, logical operators, and update operations to real-world datasets with **PyMongo**.

## Prerequisites
- Python 3.x with PyMongo installed — see [PyMongo Setup](../notes/04%20-%20MongoDB%20Data%20Structures.md#pymongo-setup) in Notes 04
- Access to a MongoDB server, reachable from your host machine on port 27017
- **Database: `labs`** — collections `grades`, `posts`, `stories`, `customers`, `inspections`
- The `labs` datasets restored with `mongorestore` — see [Lab Data Setup](../notes/04%20-%20MongoDB%20Data%20Structures.md#lab-data-setup) in Notes 04
- Basic understanding of MongoDB query syntax and operators

## Database Setup

Every collection in this lab lives in the **`labs`** database. Start your script with this
preamble — it names the database once, and binds one variable per collection so each query
below is self-describing:

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "<IP_ADDRESS>"   # replace with your VM's IP address

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")

# Database: labs
labs = client["labs"]

grades      = labs["grades"]
posts       = labs["posts"]
stories     = labs["stories"]
customers   = labs["customers"]
inspections = labs["inspections"]
```

### Confirm your data is there

Run this before you start. If any count is 0, your `mongorestore` did not complete and no
query below will return anything:

```python
for name in ["grades", "posts", "stories", "customers", "inspections"]:
    count = labs[name].count_documents({})
    print(f"{name:12} {count}")
```

Expected output:

```
grades       100000
posts        1000
stories      10000
customers    1000
inspections  81047
```

## Query Operators Review

In PyMongo, a query is a **Python dict**. The operator names are the same as in `mongosh`, but
they are quoted strings and the whole filter is a dict:

```python
# mongosh:  db.customers.find({ age: { $lt: 40 } })
# PyMongo:
for doc in customers.find({ "age": { "$lt": 40 } }):
    print(doc)
```

`find()` returns a **cursor** — a lazy iterator. Nothing is fetched until you loop over it, so
remember to iterate (or wrap it in `list()`) or you will see only `<pymongo.cursor.Cursor ...>`.

### Comparison Operators
- `$gt`, `$gte`: Greater than, greater than or equal
- `$lt`, `$lte`: Less than, less than or equal
- `$eq`, `$ne`: Equal, not equal
- `$in`, `$nin`: In array, not in array

### Logical Operators
- `$and`: Match all conditions
- `$or`: Match any condition
- `$not`: Negate condition
- `$nor`: Match none of the conditions

### Array and Nested Field Operators
- `$elemMatch`: One array element must satisfy **all** the given conditions at once
- Dot notation: `"address.city"` reaches into a nested subdocument

### Update Operators
- `$set`: Set field value
- `$inc`: Increment numeric value
- `$unset`: Remove field
- `$push`: Add to array
- `$pull`: Remove from array

Updates take two arguments — a filter and an update document. Try it on a scratch collection you
own, so you are not experimenting on the restored data:

```python
# Database: labs    Collection: lab2_toy (scratch — safe to create and drop)
lab2_toy = labs["lab2_toy"]

lab2_toy.drop()
lab2_toy.insert_many([{ "n": 1 }, { "n": 2 }, { "n": 9 }])

result = lab2_toy.update_many(
    { "n": { "$lt": 3 } },                    # filter: which documents
    { "$inc": { "n": 10 } }                   # update: what to change
)
print(result.matched_count, result.modified_count)   # 2 2

print(sorted(doc["n"] for doc in lab2_toy.find()))   # [9, 11, 12]
lab2_toy.drop()
```

## Lab Tasks

All six tasks run against the **`labs`** database using the variables bound in the preamble above.

### Task 1: Student Grades Query (8 points)
**Requirement**: Find all students who have an exam grade higher than 75 in the `grades` collection.

**Note**: each document holds a `scores` array whose elements have a `type` and a `score`. Both
conditions must hold for the *same* array element.

```python
# Collection: labs.grades
# Your query here
```

### Task 2: Posts with Logical Conditions (8 points)
**Requirement**: Find all posts that are either authored by a machine OR tagged with either "lamb" or "fold" in the `posts` collection.

```python
# Collection: labs.posts
# Your query here
```

### Task 3: Digg Stories with Sorting (8 points)
**Requirement**: Find all digg stories where the container is "Gaming" OR the topic is "Microsoft" and sort by the number of comments from greatest to least in the `stories` collection.

**Note**: `container` and `topic` are subdocuments, not strings.

```python
# Collection: labs.stories
# Your query here
```

### Task 4: Customer Demographics Query (8 points)
**Requirement**: Find all gold customers younger than 40 in the `customers` collection.

```python
# Collection: labs.customers
# Your query here
```

### Task 5: Bulk Update Operation (10 points)
**Requirement**: For all inspections with a "Fail" result, set a fine value of 100 in the `inspections` collection.

```python
# Collection: labs.inspections
# Your query here
```

### Task 6: Conditional Update Operation (10 points)
**Requirement**: Update all inspections done in the city of "ROSEDALE". For failed inspections, raise the "fine" value by 150 in the `inspections` collection.

```python
# Collection: labs.inspections
# Your query here
```

> **Tasks 5 and 6 modify the restored data.** `$inc` compounds, so running Task 6 twice raises
> the same fines twice and your numbers will not match your first run. Print
> `result.modified_count` after each update so you can see what changed.

If you need to start over, remove the field you added and re-run Tasks 5 and 6 from a clean state:

```python
# Removes only the fine field — the rest of labs.inspections is untouched
result = inspections.update_many({}, { "$unset": { "fine": "" } })
print("fines cleared:", result.modified_count)
```

## Deliverables

Submit **only** one `.zip` file, named exactly:

**CSCI112-[StudentID]-[LastName]-MongoQueries.zip** — e.g. `CSCI112-181234-Cruz-MongoQueries.zip`

The zip must contain your Python source and a `requirements.txt`:

```
CSCI112-181234-Cruz-MongoQueries.zip
├── mongo_queries.py
└── requirements.txt
```

**Do not include a virtual environment or installed dependencies** — no `.venv/`, no
`site-packages/`, no wheels. Graders build a fresh environment from your `requirements.txt`.

Generate it from your activated virtual environment:

```bash
pip freeze > requirements.txt
```

For this lab that is two lines (your versions may differ):

```
dnspython==2.8.0
pymongo==4.17.0
```

**Python file format example**:
```python
"""
Certificate of Authorship:
I have not discussed the Python language code in my program with anyone
other than my instructor or the teaching assistants assigned to this course.
I have not used Python language code obtained from another student,
or any other unauthorized source, either modified or unmodified.
If any Python language code or documentation used in my program
was obtained from another source, such as a textbook or course notes,
that has been clearly noted with a proper citation in the comments of my program.
"""

# Lab 2 Solution - MongoDB Query Operations
# Student: [Your Name]

from pymongo import MongoClient

VM_IP_ADDRESS = "<IP_ADDRESS>"   # replace with your VM's IP address

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")

# Database: labs
labs = client["labs"]

grades      = labs["grades"]
posts       = labs["posts"]
stories     = labs["stories"]
customers   = labs["customers"]
inspections = labs["inspections"]

# Task 1: Students with exam grade > 75 (8 points)
# Collection: labs.grades
# Your query here

# Task 2: Posts by machine or tagged with lamb/fold (8 points)
# Collection: labs.posts
# Your query here

# Task 3: Gaming/Microsoft stories sorted by comments (8 points)
# Collection: labs.stories
# Your query here

# Task 4: Gold customers under 40 (8 points)
# Collection: labs.customers
# Your query here

# Task 5: Set fine for failed inspections (10 points)
# Collection: labs.inspections
# Your query here

# Task 6: Update ROSEDALE failed inspection fines (10 points)
# Collection: labs.inspections
# Your query here
```

## Grading

**Total: 50 points**

- Task 1: Student grades query (8 points)
- Task 2: Posts logical conditions (8 points)
- Task 3: Stories with sorting (8 points)
- Task 4: Customer demographics (8 points)
- Task 5: Bulk update operation (10 points)
- Task 6: Conditional update (10 points)

## Notes
- Use PyMongo from your host machine, not `mongosh`
- Print your results — a bare `find()` call produces a cursor and shows nothing
- Do not name your script `pymongo.py` or `bson.py`; Python will import your file instead of the driver
