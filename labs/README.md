# Laboratory Exercises - CSCI 112 Contemporary Databases

This directory contains hands-on laboratory exercises designed to provide practical experience with NoSQL database systems and contemporary data management techniques.

## Tooling

Labs 1–3 are **PyMongo** labs: you write a Python script on your host machine and it connects to
`mongod` on your VM over the network. Every one of them opens with the same preamble, so set up a
virtual environment once — see [PyMongo Setup](../notes/04%20-%20MongoDB%20Data%20Structures.md#pymongo-setup)
in Notes 04 — and reuse it:

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "<IP_ADDRESS>"   # replace with your VM's IP address

client = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")

# Database: labs
labs = client["labs"]
```

Lab 4 uses **boto3** against DynamoDB. Lab 5 is the exception: it stays in **`mongosh`**, because
cluster administration (`rs.initiate()`, `sh.addShard()`, `sh.status()`) is shell work, not
application work.

## Submission Format

Labs 1–4 are submitted as **one `.zip` file per lab**, named exactly
`CSCI112-[StudentID]-[LastName]-[LabName].zip` — the lab name is given in each lab's
**Deliverables** section. The zip holds your Python source plus a `requirements.txt` generated with
`pip freeze`:

```
CSCI112-181234-Cruz-MongoQueries.zip
├── mongo_queries.py
└── requirements.txt
```

**Do not zip your virtual environment or installed dependencies** — no `.venv/`, no
`site-packages/`, no wheels. Graders build a fresh environment from your `requirements.txt`.

Lab 5 has no file deliverable: it is an infrastructure diagram plus a live demonstration.

## Database Convention

All MongoDB labs read and write the **`labs`** database. The collections each lab needs are listed
in its **Prerequisites** section, and every lab names its database and collection in a comment
above the lookup. The `labs` datasets are restored by the `mongorestore` steps in
[notes/04 — MongoDB Data Structures](../notes/04%20-%20MongoDB%20Data%20Structures.md#lab-data-setup).

The `sample` database is used only for scratch collections in the lecture notes — never for lab deliverables.

## Available Labs

### [Lab 1: Intro to NoSQL - From ERD to JSON](lab01_erd_to_json.md)
**Focus**: Document Database Fundamentals (MongoDB / PyMongo)
- Convert relational database designs (ERD) to NoSQL document structures
- Learn JSON syntax, how it maps onto Python dicts, and document embedding strategies
- Practice `insert_one()` and verification with PyMongo
- Understand the differences between normalized relational data and embedded documents

**Database / collections**: `labs` → `lab1` (you create it)
**Prerequisites**: Python 3.x with PyMongo; basic understanding of relational databases, ERDs, and primary/foreign key concepts
**Duration**: 2-3 hours
**Points**: 30 points

### [Lab 2: MongoDB Query Operations](lab02_mongodb_queries.md)
**Focus**: Query and Update Operators (MongoDB / PyMongo)
- Apply comparison operators (`$gt`, `$lt`, `$in`) and logical operators (`$and`, `$or`, `$nor`)
- Query nested fields and arrays across several real-world datasets
- Perform bulk and conditional updates with `$set` and `$inc`

**Database / collections**: `labs` → `grades`, `posts`, `stories`, `customers`, `inspections`
**Prerequisites**: Python 3.x with PyMongo; [Notes 04 — MongoDB Data Structures](../notes/04%20-%20MongoDB%20Data%20Structures.md); `labs` datasets restored
**Duration**: 2-3 hours
**Points**: 50 points

### [Lab 3: MongoDB Aggregation Pipeline](lab03_aggregation_pipeline.md)
**Focus**: Aggregation Framework (MongoDB / PyMongo)
- Build multi-stage pipelines with `$match`, `$group`, `$sort`, `$unwind`, and `$bucket`
- Use accumulators (`$sum`, `$avg`, `$push`) to summarize grouped documents
- Bucket continuous values into ranges for demographic analysis

**Database / collections**: `labs` → `posts`, `inspections`, `companies`, `customers`
**Prerequisites**: Lab 2; Python 3.x with PyMongo; [Notes 05 — MongoDB Aggregation](../notes/05%20-%20MongoDB%20Aggregation.md)
**Duration**: 3-4 hours
**Points**: 50 points

### [Lab 4: DynamoDB Table Design with Local Secondary Indexes](lab04_dynamodb_lsi.md)
**Focus**: Key-Value / Wide-Column Modeling (DynamoDB)
- Design a partition key and sort key for an e-commerce order-tracking access pattern
- Add Local Secondary Indexes to support alternative sort orders
- Create the table, insert items, and query it with Python and boto3

**Service**: Amazon DynamoDB (AWS Learner Lab, `us-east-1`) — not MongoDB
**Prerequisites**: Python 3.x, `boto3`, AWS Learner Lab credentials; [Notes 09 — Introduction to DynamoDB](../notes/09%20-%20Introduction%20to%20DynamoDB.md)
**Duration**: 3-4 hours
**Points**: 60 points

### [Lab 5: Deploying a Sharded MongoDB Cluster on Cloud VMs](lab05_mongodb_sharding.md)
**Focus**: Horizontal Scaling and High Availability (MongoDB / mongosh)
- Deploy a 7-instance sharded and replicated cluster: config servers, shards, and mongos
- Initiate replica sets and register shards with the query router
- Shard a collection on a hashed key and verify chunk distribution

**Database / collections**: `labs` → `grades`, sharded on `class_id`
**Prerequisites**: 7 cloud VMs with MongoDB installed; [Notes 06 — MongoDB Sharding and Replication](../notes/06%20-%20MongoDB%20Sharding%20and%20Replication.md)
**Duration**: 4-6 hours
**Deliverables**: Infrastructure diagram plus a live demonstration of `sh.status()` and `db.grades.getShardDistribution()`

## Lab Guidelines

- Each lab includes detailed instructions, sample data, and clear deliverables
- Follow the grading criteria specified in each lab
- Test your solutions thoroughly before submission
- Use only the tools and technologies specified in each lab — PyMongo for labs 1–3, boto3 for lab 4, `mongosh` for lab 5
- Run your script and check its output before submitting; a query that prints nothing usually means the wrong database, not an empty result

## Getting Help

- Review the theoretical materials in the [Notes](../notes/) directory
- Refer to official database documentation
- Ask questions during lab sessions or office hours
