# Laboratory Exercises - CSCI 112 Contemporary Databases

This directory contains hands-on laboratory exercises designed to provide practical experience with NoSQL database systems and contemporary data management techniques.

## Database Convention

All MongoDB labs read and write the **`labs`** database. Every lab that touches MongoDB
starts with `use labs`, and the collections it needs are listed in that lab's
**Prerequisites** section. The `labs` datasets are restored by the `mongorestore` steps in
[notes/04 — MongoDB Data Structures](../notes/04%20-%20MongoDB%20Data%20Structures.md#lab-data-setup).

The `sample` database is used only for scratch collections in the lecture notes — never for lab deliverables.

## Available Labs

### [Lab 1: Intro to NoSQL - From ERD to JSON](lab01_erd_to_json.md)
**Focus**: Document Database Fundamentals (MongoDB)
- Convert relational database designs (ERD) to NoSQL document structures
- Learn JSON syntax and document embedding strategies
- Practice MongoDB shell operations and data insertion
- Understand the differences between normalized relational data and embedded documents

**Database / collections**: `labs` → `lab1` (you create it)
**Prerequisites**: Basic understanding of relational databases, ERDs, and primary/foreign key concepts
**Duration**: 2-3 hours
**Points**: 30 points

### [Lab 2: MongoDB Query Operations](lab02_mongodb_queries.md)
**Focus**: Query and Update Operators (MongoDB)
- Apply comparison operators (`$gt`, `$lt`, `$in`) and logical operators (`$and`, `$or`, `$nor`)
- Query nested fields and arrays across several real-world datasets
- Perform bulk and conditional updates with `$set` and `$inc`

**Database / collections**: `labs` → `grades`, `posts`, `stories`, `customers`, `inspections`
**Prerequisites**: [Notes 04 — MongoDB Data Structures](../notes/04%20-%20MongoDB%20Data%20Structures.md); `labs` datasets restored
**Duration**: 2-3 hours
**Points**: 50 points

### [Lab 3: MongoDB Aggregation Pipeline](lab03_aggregation_pipeline.md)
**Focus**: Aggregation Framework (MongoDB)
- Build multi-stage pipelines with `$match`, `$group`, `$sort`, `$unwind`, and `$bucket`
- Use accumulators (`$sum`, `$avg`, `$push`) to summarize grouped documents
- Bucket continuous values into ranges for demographic analysis

**Database / collections**: `labs` → `posts`, `inspections`, `companies`, `customers`
**Prerequisites**: Lab 2; [Notes 05 — MongoDB Aggregation](../notes/05%20-%20MongoDB%20Aggregation.md)
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
**Focus**: Horizontal Scaling and High Availability (MongoDB)
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
- Use only the tools and technologies specified in each lab

## Getting Help

- Review the theoretical materials in the [Notes](../notes/) directory
- Refer to official database documentation
- Ask questions during lab sessions or office hours
