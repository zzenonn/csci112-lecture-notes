# Lab 3: MongoDB Aggregation Pipeline

## Objective
Practice MongoDB aggregation pipeline operations to perform complex data analysis and transformations using various collections in the `labs` database. This lab focuses on applying aggregation stages like `$match`, `$group`, `$sort`, `$unwind`, and `$bucket` to extract meaningful insights from real-world datasets using **PyMongo**.

## Prerequisites
- Python 3.x with PyMongo installed — see [PyMongo Setup](../notes/04%20-%20MongoDB%20Data%20Structures.md#pymongo-setup) in Notes 04
- Access to a MongoDB server, reachable from your host machine on port 27017
- **Database: `labs`** — collections `posts`, `inspections`, `companies`, `customers`
- The `labs` datasets restored with `mongorestore` — see [Lab Data Setup](../notes/04%20-%20MongoDB%20Data%20Structures.md#lab-data-setup) in Notes 04
- Basic understanding of the MongoDB aggregation framework — see [Notes 05 — MongoDB Aggregation](../notes/05%20-%20MongoDB%20Aggregation.md)

## Database Setup

Every collection in this lab lives in the **`labs`** database. Start your script with this
preamble:

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "<IP_ADDRESS>"   # replace with your VM's IP address

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")

# Database: labs
labs = client["labs"]

posts       = labs["posts"]
inspections = labs["inspections"]
companies   = labs["companies"]
customers   = labs["customers"]
```

### Confirm your data is there

Run this before you start. If any count is 0, your `mongorestore` did not complete and no
pipeline below will return anything:

```python
for name in ["posts", "inspections", "companies", "customers"]:
    count = labs[name].count_documents({})
    print(f"{name:12} {count}")
```

Expected output:

```
posts        1000
inspections  81047
companies    18801
customers    1000
```

## Aggregation Pipeline Review

In PyMongo, a pipeline is a **Python list of dicts** — one dict per stage. The stage names and
operators are the same as in `mongosh`; only the syntax around them is Python:

```python
# A pipeline is a list; each stage is a dict
pipeline = [
    { "$match": { "accountType": "gold" } },
    { "$group": { "_id": "$gender", "count": { "$sum": 1 } } },
    { "$sort":  { "count": -1 } }
]

for doc in customers.aggregate(pipeline):
    print(doc)
```

`aggregate()` returns a **cursor**, so iterate over it (or wrap it in `list()`) to see results.
Note the two different uses of `$`: `"$sum"` names an *operator*, while `"$gender"` means
*the value of the `gender` field*.

### Core Pipeline Stages
- `$match`: Filter documents (like `find()`) — put it first to shrink the pipeline early
- `$group`: Group documents and perform calculations
- `$sort`: Sort documents by specified fields (`1` ascending, `-1` descending)
- `$project`: Include/exclude fields or create new ones
- `$unwind`: Deconstruct array fields into separate documents
- `$bucket`: Group documents into buckets based on ranges
- `$limit`: Limit number of documents
- `$count`: Count documents in pipeline

### Aggregation Operators
- `$sum`: Sum values or count documents
- `$avg`: Calculate average
- `$max`, `$min`: Find maximum/minimum values
- `$push`: Create arrays from grouped values
- `$addToSet`: Create arrays with unique values
- `$size`: Length of an array — pair it with `$ifNull` if the field may be missing

## Lab Tasks

All four tasks run against the **`labs`** database using the variables bound in the preamble above.

### Task 1: Top 10 Most Common Tags (20 points)
**Requirement**: In the `posts` collection, find the top 10 most common tags.

**Note**: `tags` is an array, so counting it directly counts documents, not tags.

```python
# Collection: labs.posts
# Your aggregation pipeline here
```

### Task 2: Failed Inspections by Zip Code in Jamaica (15 points)
**Requirement**: Find how many inspections in Jamaica failed per zip code, sort from the most failures to the least failures.

**Note**: city and zip live inside the `address` subdocument, and city names are stored in
upper case.

```python
# Collection: labs.inspections
# Your aggregation pipeline here
```

### Task 3: Top 10 US Companies with Most Competitors (10 points)
**Requirement**: In the `companies` collection, find the top 10 companies with an office in the US with the most competitors.

**Note**: `offices` and `competitions` are both arrays of subdocuments. Check what value the
office country field actually holds before matching on it.

```python
# Collection: labs.companies
# Your aggregation pipeline here
```

### Task 4: Customer Demographics by Age Groups (5 points)
**Requirement**: In the `customers` collection, find the number of silver and gold customers based on the following age groups: under 18, 18-64, 65 and above.

**Hint**: Use the `$bucket` operator to group customers by age ranges. `$bucket` omits empty
buckets from its output, so do not be surprised if a range you defined does not appear.

```python
# Collection: labs.customers
# Your aggregation pipeline here
```

## Deliverables

Submit **only** one `.zip` file, named exactly:

**CSCI112-[StudentID]-[LastName]-AggregationPipeline.zip** — e.g. `CSCI112-181234-Cruz-AggregationPipeline.zip`

The zip must contain your Python source and a `requirements.txt`:

```
CSCI112-181234-Cruz-AggregationPipeline.zip
├── aggregation_pipeline.py
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

# Lab 3 Solution - MongoDB Aggregation Pipeline
# Student: [Your Name]

from pymongo import MongoClient

VM_IP_ADDRESS = "<IP_ADDRESS>"   # replace with your VM's IP address

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")

# Database: labs
labs = client["labs"]

posts       = labs["posts"]
inspections = labs["inspections"]
companies   = labs["companies"]
customers   = labs["customers"]

# Task 1: Top 10 most common tags in posts (20 points)
# Collection: labs.posts
# Your aggregation pipeline here

# Task 2: Failed inspections by zip code in Jamaica (15 points)
# Collection: labs.inspections
# Your aggregation pipeline here

# Task 3: Top 10 US companies with most competitors (10 points)
# Collection: labs.companies
# Your aggregation pipeline here

# Task 4: Silver and gold customers by age groups (5 points)
# Collection: labs.customers
# Your aggregation pipeline here
```

## Grading

**Total: 50 points**

- Task 1: Top 10 most common tags (20 points)
- Task 2: Failed inspections by zip code (15 points)
- Task 3: Top 10 US companies with most competitors (10 points)
- Task 4: Customer demographics by age groups (5 points)

## Notes
- Use PyMongo from your host machine, not `mongosh`
- Print your results — `aggregate()` returns a cursor and shows nothing on its own
- All four tasks are read-only; nothing in this lab modifies the restored data
- Do not name your script `pymongo.py` or `bson.py`; Python will import your file instead of the driver
