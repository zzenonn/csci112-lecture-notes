# Introduction to Amazon DynamoDB

**Course:** CSCI 112 / 212 - Contemporary Databases  
**Topic:** Key-Value and Document Databases with Amazon DynamoDB

---

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [What is Amazon DynamoDB?](#what-is-amazon-dynamodb)
  - [Serverless and Fully Managed](#serverless-and-fully-managed)
  - [Key-Value and Document Store](#key-value-and-document-store)
  - [When to Use DynamoDB](#when-to-use-dynamodb)
- [Core Concepts](#core-concepts)
  - [Tables, Items, and Attributes](#tables-items-and-attributes)
  - [Primary Keys](#primary-keys)
  - [Secondary Indexes](#secondary-indexes)
  - [Billing Modes](#billing-modes)
- [Environment Setup](#environment-setup)
  - [Prerequisites](#prerequisites)
  - [Clone the Sample Repository](#clone-the-sample-repository)
  - [Create and Activate Virtual Environment](#create-and-activate-virtual-environment)
  - [Install Dependencies](#install-dependencies)
  - [Configure AWS Credentials](#configure-aws-credentials)
- [Working with DynamoDB using Python (boto3)](#working-with-dynamodb-using-python-boto3)
  - [Creating a Table](#creating-a-table)
  - [Inserting Items](#inserting-items)
  - [Querying Data](#querying-data)
- [DynamoDB vs. MongoDB](#dynamodb-vs-mongodb)
- [Practice Exercises](#practice-exercises)
- [Summary](#summary)
- [Additional Resources](#additional-resources)

---

## Overview

This module introduces Amazon DynamoDB, a fully managed, serverless NoSQL database provided by AWS. Unlike MongoDB, which requires you to install and manage a database server, DynamoDB is a cloud-native service where AWS handles all infrastructure, scaling, and maintenance. We will learn how DynamoDB organizes data using primary keys and secondary indexes, and interact with it using Python and the boto3 SDK.

---

## Objectives

- Understand what a serverless, fully managed database is and how it differs from self-hosted databases
- Learn DynamoDB's data model: tables, items, attributes, and primary keys
- Understand partition keys, sort keys, and Local Secondary Indexes (LSI)
- Set up a Python development environment with boto3 to interact with DynamoDB
- Perform basic CRUD operations against a DynamoDB table

---

## What is Amazon DynamoDB?

Amazon DynamoDB is a fully managed NoSQL database service provided by Amazon Web Services (AWS). It is designed for applications that need consistent, single-digit millisecond performance at any scale.

### Serverless and Fully Managed

Unlike MongoDB, where you installed and configured the database server on a virtual machine, DynamoDB is **serverless**. This means:

- **No servers to manage** -- AWS handles all hardware provisioning, software patching, and cluster scaling
- **No installation required** -- you interact with DynamoDB through the AWS API, console, or SDKs
- **Automatic scaling** -- DynamoDB can scale to handle millions of requests per second without manual intervention
- **Built-in high availability** -- data is automatically replicated across multiple Availability Zones within an AWS Region
- **Pay for what you use** -- you are charged based on the amount of data stored and the number of read/write operations performed

This is a fundamentally different operational model from self-hosted databases. With DynamoDB, you focus on your data model and application logic rather than database administration.

### Key-Value and Document Store

DynamoDB supports two data models:

1. **Key-Value Store** -- each item is identified by a primary key and can store any set of attributes. You retrieve items by their exact key.
2. **Document Store** -- items can contain nested attributes (maps and lists), similar to JSON documents in MongoDB.

Unlike MongoDB's flexible querying with MQL, DynamoDB is optimized for **known access patterns**. You design your table schema around the queries your application will make, rather than exploring data ad hoc.

### When to Use DynamoDB

DynamoDB is a good fit when:

- You need **predictable, low-latency performance** at any scale
- Your application has **well-defined access patterns** that can be modeled with primary keys and indexes
- You want a **fully managed service** with no operational overhead
- You are building **serverless applications** on AWS (e.g., with AWS Lambda)
- You need **high availability** across multiple data centers

DynamoDB is less suitable when:

- You need complex ad hoc queries or joins across multiple entities
- Your access patterns are unpredictable or change frequently
- You need full-text search capabilities (consider Amazon OpenSearch instead)

---

## Core Concepts

### Tables, Items, and Attributes

DynamoDB organizes data into **tables**. Each table contains multiple **items** (equivalent to rows in a relational database or documents in MongoDB). Each item is a collection of **attributes** (equivalent to columns or fields).

| DynamoDB Term | Relational Equivalent | MongoDB Equivalent |
|---------------|----------------------|-------------------|
| Table         | Table                | Collection        |
| Item          | Row                  | Document          |
| Attribute     | Column               | Field             |

Unlike relational databases, DynamoDB is **schemaless** (aside from the primary key). Different items in the same table can have different attributes. This is similar to how MongoDB documents in the same collection can have different fields.

### Primary Keys

Every DynamoDB table must have a **primary key** defined at table creation time. The primary key uniquely identifies each item in the table. There are two types:

1. **Simple Primary Key (Partition Key only)**
   - A single attribute serves as the partition key (also called a **hash key**)
   - DynamoDB uses the partition key value to determine which internal partition stores the item
   - Example: a `Users` table with `user_id` as the partition key

2. **Composite Primary Key (Partition Key + Sort Key)**
   - Two attributes together form the primary key
   - The **partition key** (HASH) determines the storage partition
   - The **sort key** (RANGE) sorts items within the same partition and enables range queries
   - Items with the same partition key are stored together and sorted by the sort key
   - Example: a `Music` table with `artist` as partition key and `song` as sort key

```
Music Table (Composite Primary Key)
┌──────────────┬──────────────┬────────────┬──────┐
│ artist (PK)  │ song (SK)    │ album      │ year │
├──────────────┼──────────────┼────────────┼──────┤
│ Taylor Swift │ Cardigan     │ Folklore   │ 2020 │
│ Taylor Swift │ Exile        │ Folklore   │ 2020 │
│ Taylor Swift │ The 1        │ Folklore   │ 2020 │
│ Lady Gaga    │ Rain on Me   │ Chromatica │ 2020 │
│ Lady Gaga    │ Sour Candy   │ Chromatica │ 2020 │
│ Lady Gaga    │ Stupid Love  │ Chromatica │ 2020 │
└──────────────┴──────────────┴────────────┴──────┘
```

Items with the same partition key (`Taylor Swift`) are grouped together and sorted by the sort key (`song`).

### Secondary Indexes

Sometimes you need to query data by attributes other than the primary key. DynamoDB supports two types of **secondary indexes**:

**Local Secondary Index (LSI)**
- Must be created at table creation time (cannot be added later)
- Shares the **same partition key** as the base table but uses a **different sort key**
- Useful when you want an alternative sort order within the same partition
- Example: for the Music table, an LSI with `artist` (partition key) and `album` (sort key) lets you query "all songs by Taylor Swift from the Folklore album"

**Global Secondary Index (GSI)**
- Can be created at any time
- Uses a **completely different partition key** (and optionally a different sort key) from the base table
- Essentially creates a separate, automatically maintained copy of the data with a different key structure
- GSIs now support **multi-attribute keys**, which we will explore in the next module

### Billing Modes

DynamoDB offers two billing modes:

| Mode | Description | Best For |
|------|-------------|----------|
| **On-Demand (PAY_PER_REQUEST)** | Pay per read/write request with no capacity planning | Unpredictable workloads, development/testing |
| **Provisioned** | Pre-allocate read/write capacity units | Predictable workloads, cost optimization |

For this course, we use **on-demand** billing to avoid capacity planning complexity.

---

## Environment Setup

### Prerequisites

- Python 3.10 or higher
- Git
- An AWS account with DynamoDB access (credentials will be provided)

### Clone the Sample Repository

```bash
git clone https://github.com/zzenonn/dynamodb_sample.git
cd dynamodb_sample
```

### Create and Activate Virtual Environment

**On Linux/macOS:**

```bash
python3 -m venv .env
source .env/bin/activate
```

**On Windows:**

```bash
python -m venv .env
.env\Scripts\activate
```

### Install Dependencies

```bash
pip install -r requirements.txt
```

If you experience connection issues on campus networks:

```bash
pip install -r requirements.txt -i https://mirrors.cloud.tencent.com/pypi/simple
```

### Configure AWS Credentials

Create the AWS credentials file:

```bash
mkdir -p ~/.aws
```

Edit `~/.aws/credentials`:

```ini
[default]
aws_access_key_id = YOUR_ACCESS_KEY_HERE
aws_secret_access_key = YOUR_SECRET_ACCESS_KEY_HERE
```

Edit `~/.aws/config`:

```ini
[default]
region = us-east-1
output = json
```

On Linux/macOS, secure your credentials:

```bash
chmod 600 ~/.aws/credentials
chmod 600 ~/.aws/config
```

---

## Working with DynamoDB using Python (boto3)

The sample repository contains three Python scripts that demonstrate the core DynamoDB operations. We use **boto3**, the AWS SDK for Python, to interact with DynamoDB programmatically.

### Creating a Table

> **Source code:** [`music.py`](https://github.com/zzenonn/dynamodb_sample/blob/main/music.py)

The `music.py` script creates a DynamoDB table with a composite primary key and a Local Secondary Index.

**Key elements of table creation:**

1. **Attribute Definitions** -- declare the attributes used in keys and indexes, along with their data types (`S` for String, `N` for Number, `B` for Binary)
2. **Key Schema** -- define which attributes form the primary key (`HASH` for partition key, `RANGE` for sort key)
3. **Local Secondary Index** -- define an alternative sort key that shares the same partition key

```python
import boto3

dynamodb = boto3.resource('dynamodb', region_name='us-east-1')

# Only attributes used in keys/indexes need to be defined here
attribute_definitions = [
    {'AttributeName': 'artist', 'AttributeType': 'S'},
    {'AttributeName': 'song', 'AttributeType': 'S'},
    {'AttributeName': 'album', 'AttributeType': 'S'}
]

# Primary key: artist (partition) + song (sort)
key_schema = [
    {'AttributeName': 'artist', 'KeyType': 'HASH'},
    {'AttributeName': 'song', 'KeyType': 'RANGE'}
]

# LSI: same partition key (artist), different sort key (album)
local_secondary_indexes = [{
    'IndexName': 'album_index',
    'KeySchema': [
        {'AttributeName': 'artist', 'KeyType': 'HASH'},
        {'AttributeName': 'album', 'KeyType': 'RANGE'}
    ],
    'Projection': {'ProjectionType': 'ALL'}
}]

table = dynamodb.create_table(
    TableName='music',
    KeySchema=key_schema,
    AttributeDefinitions=attribute_definitions,
    BillingMode='PAY_PER_REQUEST',
    LocalSecondaryIndexes=local_secondary_indexes
)
```

Run the script to create the table:

```bash
python music.py
```

### Inserting Items

> **Source code:** [`insert_music.py`](https://github.com/zzenonn/dynamodb_sample/blob/main/insert_music.py)

The `insert_music.py` script demonstrates inserting items using `put_item()`. DynamoDB items are Python dictionaries.

```python
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
music_table = dynamodb.Table('music')

song = {
    'artist': 'Taylor Swift',    # Partition key (required)
    'song': 'Cardigan',          # Sort key (required)
    'album': 'Folklore',         # Additional attribute
    'year': 2020                 # Additional attribute
}

music_table.put_item(Item=song)
```

Key points about `put_item()`:

- The partition key and sort key attributes are **required** in every item
- Additional attributes are optional and can vary between items (schemaless)
- If an item with the same primary key already exists, `put_item()` **replaces** the entire item
- Items can contain nested structures (maps) and lists

Run the script to insert sample data:

```bash
python insert_music.py
```

### Querying Data

> **Source code:** [`read_music.py`](https://github.com/zzenonn/dynamodb_sample/blob/main/read_music.py)

The `read_music.py` script demonstrates three query patterns using `KeyConditionExpression`:

**1. Query by partition key only** -- get all songs by an artist:

```python
from boto3.dynamodb.conditions import Key

response = table.query(
    KeyConditionExpression=Key('artist').eq('Lady Gaga')
)
songs = response['Items']
```

**2. Query by partition key + sort key** -- get a specific song:

```python
response = table.query(
    KeyConditionExpression=Key('artist').eq('Taylor Swift') & 
                           Key('song').eq('Cardigan')
)
```

**3. Query using Local Secondary Index** -- get songs by artist and album:

```python
response = table.query(
    IndexName='album_index',
    KeyConditionExpression=Key('artist').eq('Taylor Swift') & 
                           Key('album').eq('Folklore')
)
```

Run the script to test queries:

```bash
python read_music.py
```

**Important:** In DynamoDB, `query()` is fundamentally different from MongoDB's `find()`. A DynamoDB query **always requires the partition key** and can optionally filter by the sort key. You cannot query arbitrary attributes without an index. This is by design -- it guarantees predictable performance regardless of table size.

---

## DynamoDB vs. MongoDB

| Feature | MongoDB | DynamoDB |
|---------|---------|----------|
| **Deployment** | Self-hosted or Atlas (managed) | Fully managed (AWS only) |
| **Scaling** | Manual sharding configuration | Automatic, transparent |
| **Query Language** | MQL (flexible, ad hoc) | Key-based queries (designed access patterns) |
| **Schema** | Flexible | Flexible (but keys are fixed at creation) |
| **Indexes** | Created anytime | LSI at creation, GSI anytime |
| **Joins** | `$lookup` aggregation | Not supported (use single-table design) |
| **Max Item Size** | 16 MB | 400 KB |
| **Billing** | Infrastructure cost | Per-request or provisioned capacity |
| **Best For** | Flexible querying, complex aggregations | High-scale, predictable access patterns |

---

## Practice Exercises

1. Create the music table and insert the sample data using the provided scripts.
2. Query all songs by "Lady Gaga" and observe how items are sorted by the sort key.
3. Query a specific song using both the partition key and sort key.
4. Use the LSI to find all songs by "Lady Gaga" from the album "Chromatica".
5. Insert a new artist with at least 3 songs across 2 albums. Verify your insertions using queries.
6. Try inserting two items with the same `artist` and `song` values. What happens to the first item?
7. Compare: how would you model this same music data in MongoDB? What queries would be easier or harder in each system?

---

## Summary

- DynamoDB is a **serverless, fully managed** NoSQL database -- no servers to install or manage
- Data is organized into **tables** containing **items** with flexible **attributes**
- Every table requires a **primary key** (partition key, or partition key + sort key)
- **Local Secondary Indexes** provide alternative sort orders within the same partition
- We use **boto3** (AWS SDK for Python) to interact with DynamoDB programmatically
- DynamoDB is optimized for **known access patterns** -- design your keys around your queries

In the next module, we will explore **DynamoDB data modeling** techniques, including single-table design, Global Secondary Indexes, and the new multi-attribute keys feature.

---

## Additional Resources

- [Amazon DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
- [boto3 DynamoDB Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/dynamodb.html)
- [DynamoDB Core Components](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html)
- [Sample Code Repository](https://github.com/zzenonn/dynamodb_sample)
