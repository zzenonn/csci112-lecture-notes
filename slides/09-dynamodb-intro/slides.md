---
theme: discs
title: "Introduction to Amazon DynamoDB"
highlighter: shiki
layout: cover
codeCopy: true
---

Introduction to Amazon DynamoDB

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Explain what a **serverless, fully managed** database is and how it differs from self-hosted
2. Describe DynamoDB's data model: tables, items, attributes, and primary keys
3. Distinguish **partition keys**, **sort keys**, and **Local Secondary Indexes**
4. Set up a Python environment with **boto3** to interact with DynamoDB
5. Perform basic CRUD operations and queries using Python

---
layout: section
---

# What is Amazon DynamoDB?

---

# Serverless and Fully Managed

Unlike MongoDB — or a traditional SQL server — where you installed and configured the server on a VM, DynamoDB is **serverless**:

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:1rem; font-size:0.86rem;">
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">Self-hosted (MongoDB or SQL)</div>
    You install and configure the server (<code>mongod</code>, PostgreSQL, MySQL…).<br>
    You manage updates, storage, replication.<br>
    You pay for the VM whether it is idle or busy.
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">DynamoDB (managed)</div>
    AWS handles hardware, patching, scaling.<br>
    No installation — interact via API or SDK.<br>
    Pay only for reads, writes, and storage used.
  </div>
</div>

<div style="margin-top:0.9rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.5rem 1rem; font-size:0.86rem;">
  Data is automatically replicated across multiple AWS Availability Zones. High availability is built in, not configured.
</div>

---

# Key-Value and Document Store

DynamoDB supports two data models in one service:

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:0.9rem; font-size:0.86rem;">
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.8rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.3rem;">Key-Value</div>
    Identify an item by its exact primary key.<br>
    Retrieve the full item in a single operation.<br>
    <em>Fastest possible lookup.</em>
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem;">Document</div>
    Items can contain nested maps and lists.<br>
    Similar to JSON documents in MongoDB.<br>
    <em>Rich structure, still key-accessed.</em>
  </div>
</div>

<div style="margin-top:0.9rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 1rem; font-size:0.86rem;">
  Unlike MongoDB's flexible MQL, DynamoDB is optimized for <strong>known access patterns</strong>. You design your key schema around the queries your application will make — not the other way around.
</div>

---

# Tables, Items, and Attributes

<div style="font-size:0.9rem;">

| DynamoDB | Relational | MongoDB |
|----------|------------|---------|
| Table | Table | Collection |
| Item | Row | Document |
| Attribute | Column | Field |

</div>

DynamoDB is **schemaless** aside from the primary key — different items in the same table can have different attributes.

```python
{ "artist": "Taylor Swift", "song": "Cardigan", "album": "Folklore", "year": 2020 }
{ "artist": "Lady Gaga",    "song": "Rain on Me", "feat": "Ariana Grande" }
```

<div style="margin-top:0.6rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.4rem 1rem; font-size:0.82rem;">
  Max item size in DynamoDB: <strong>400 KB</strong>. MongoDB documents can be up to 16 MB.
</div>

---
layout: section
---

# Primary Keys and Indexes

---

# Simple vs Composite Primary Key

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; align-items:start; margin-top:0.5rem;">
<div>

**Simple key** — partition key only:

```
Users table
┌──────────────┬──────────────┐
│ user_id (PK) │ name         │
├──────────────┼──────────────┤
│ u001         │ Ana Reyes    │
│ u002         │ Marco Santos │
└──────────────┴──────────────┘
```

</div>
<div>

**Composite key** — partition + sort key:

```
Music table
┌──────────────┬────────────┬────────────┐
│ artist (PK)  │ song (SK)  │ album      │
├──────────────┼────────────┼────────────┤
│ Taylor Swift │ Cardigan   │ Folklore   │
│ Taylor Swift │ The 1      │ Folklore   │
│ Lady Gaga    │ Rain on Me │ Chromatica │
└──────────────┴────────────┴────────────┘
```

</div>
</div>

Items with the same partition key are **stored together** and **sorted by the sort key**.

---

# Local Secondary Index (LSI)

An LSI keeps the **same partition key** but uses a **different sort key** — an alternative sort order within the same partition.

```python
# Base table: artist (PK) + song (SK)
# LSI:        artist (PK) + album (SK)   ← lets you query by artist + album

response = table.query(
    IndexName='album_index',
    KeyConditionExpression=Key('artist').eq('Taylor Swift') &
                           Key('album').eq('Folklore')
)
```

<div style="margin-top:0.8rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 1rem; font-size:0.86rem;">
  <strong>LSIs must be defined at table creation time.</strong> They cannot be added later. Plan your LSIs when you design the table.
</div>

---
layout: section
---

# Billing and Environment Setup

---

# Billing Modes

| Mode | How you pay | Best for |
|------|-------------|----------|
| **On-Demand** (`PAY_PER_REQUEST`) | Per read/write request | Unpredictable workloads, dev/test |
| **Provisioned** | Pre-allocated capacity units | Predictable workloads, cost optimization |

For this course we use **on-demand** billing.

---

# Environment Setup

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; align-items:start; margin-top:0.4rem; font-size:0.92rem;">
<div>

```bash
# 1. Clone sample repository
git clone https://github.com/zzenonn/dynamodb_sample.git
cd dynamodb_sample

# 2. Create + activate virtual env
python3 -m venv .venv
source .venv/bin/activate   # macOS/Linux
# .venv\Scripts\activate    # Windows

# 3. Install dependencies
pip install -r requirements.txt
```

</div>
<div>

Credentials in `~/.aws/credentials`, region in `~/.aws/config`:

```ini
[default]
aws_access_key_id     = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
region = us-east-1
```

Or via **environment variables** (override the files):

```bash
export AWS_ACCESS_KEY_ID=YOUR_ACCESS_KEY
export AWS_SECRET_ACCESS_KEY=YOUR_SECRET_KEY
export AWS_DEFAULT_REGION=us-east-1
```

</div>
</div>

---
layout: section
---

# Working with DynamoDB in Python

---

# Creating a Table — Keys and Attributes

```python
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')

table = dynamodb.create_table(
    TableName='music',
    # Primary key: artist (partition) + song (sort)
    KeySchema=[
        {'AttributeName': 'artist', 'KeyType': 'HASH'},
        {'AttributeName': 'song',   'KeyType': 'RANGE'},
    ],
    # Only attributes used in keys/indexes need to be declared here
    AttributeDefinitions=[
        {'AttributeName': 'artist', 'AttributeType': 'S'},
        {'AttributeName': 'song',   'AttributeType': 'S'},
        {'AttributeName': 'album',  'AttributeType': 'S'},
    ],
    BillingMode='PAY_PER_REQUEST',
    # ... continued on next slide
)
```

---

# Adding a Local Secondary Index

```python
table = dynamodb.create_table(
    TableName='music',
    KeySchema=[ ... ],
    AttributeDefinitions=[ ... ],
    # LSI: same partition key (artist), different sort key (album)
    LocalSecondaryIndexes=[{
        'IndexName': 'album_index',
        'KeySchema': [
            {'AttributeName': 'artist', 'KeyType': 'HASH'},
            {'AttributeName': 'album',  'KeyType': 'RANGE'},
        ],
        'Projection': {'ProjectionType': 'ALL'}
    }],
    BillingMode='PAY_PER_REQUEST',
)
```

<div style="margin-top:0.6rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.86rem;">
  LSIs must be declared at table creation. <strong>You cannot add an LSI to an existing table.</strong>
</div>

---

# Inserting and Reading Items

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; align-items:start; margin-top:0.4rem;">
<div>

**Insert with `put_item()`:**

```python
music_table = dynamodb.Table('music')

music_table.put_item(Item={
    'artist': 'Taylor Swift',  # PK — required
    'song':   'Cardigan',      # SK — required
    'album':  'Folklore',
    'year':   2020,
})
# Same PK+SK exists → REPLACES it entirely
```

</div>
<div>

**Query by partition key:**

```python
from boto3.dynamodb.conditions import Key

response = music_table.query(
    KeyConditionExpression=Key('artist').eq('Lady Gaga')
)
for item in response['Items']:
    print(item['song'])
```

</div>
</div>

<div style="margin-top:0.6rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.86rem;">
  <code>query()</code> always requires the partition key. You cannot query by arbitrary attributes without an index — by design.
</div>

---

# Querying with Sort Key and LSI

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; align-items:start; margin-top:0.4rem;">
<div>

**Filter by partition key + sort key:**

```python
response = music_table.query(
    KeyConditionExpression=
        Key('artist').eq('Taylor Swift') &
        Key('song').eq('Cardigan')
)
```

</div>
<div>

**Query via LSI (artist + album):**

```python
response = music_table.query(
    IndexName='album_index',
    KeyConditionExpression=
        Key('artist').eq('Taylor Swift') &
        Key('album').eq('Folklore')
)
```

</div>
</div>

<div style="margin-top:0.7rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.5rem 0.9rem; font-size:0.85rem;">
  See <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/09%20-%20Introduction%20to%20DynamoDB.md" style="color:#16a34a;">notes/09 — Introduction to DynamoDB</a> for the full scripts (<code>music.py</code>, <code>insert_music.py</code>, <code>read_music.py</code>).
</div>

---

# DynamoDB vs MongoDB

| Feature | MongoDB | DynamoDB |
|---------|---------|----------|
| **Deployment** | Self-hosted or Atlas | Fully managed (AWS) |
| **Query language** | MQL — flexible, ad hoc | Key-based — designed access patterns |
| **Indexes** | Created anytime | LSI at creation; GSI anytime |
| **Joins** | `$lookup` | Not supported — use single-table design |
| **Max item size** | 16 MB | 400 KB |
| **Billing** | VM infrastructure cost | Per-request or provisioned capacity |
| **Best for** | Flexible querying, complex aggregations | High-scale, predictable access patterns |

---

# Further Reading

- [notes/09 — Introduction to DynamoDB](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/09%20-%20Introduction%20to%20DynamoDB.md)
- [Course notes overview](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/README.md)
