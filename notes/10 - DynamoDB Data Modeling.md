# DynamoDB Data Modeling

**Course:** CSCI 112 / 212 - Contemporary Databases  
**Topic:** DynamoDB Data Modeling -- Single-Table Design and Multi-Attribute Keys

---

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Why Data Modeling Matters in DynamoDB](#why-data-modeling-matters-in-dynamodb)
- [Global Secondary Indexes (GSI)](#global-secondary-indexes-gsi)
- [Single-Table Design](#single-table-design)
  - [What is Single-Table Design?](#what-is-single-table-design)
  - [Comparison with MongoDB Single Collection Pattern](#comparison-with-mongodb-single-collection-pattern)
- [Multi-Attribute Keys](#multi-attribute-keys)
  - [What are Multi-Attribute Keys?](#what-are-multi-attribute-keys)
  - [Query Rules](#query-rules)
  - [Sparse Indexing](#sparse-indexing)
- [Walkthrough: Users, Orders, and Items](#walkthrough-users-orders-and-items)
  - [Access Patterns](#access-patterns)
  - [Table Design](#table-design)
  - [Step 1: Create the Table](#step-1-create-the-table)
  - [Step 2: User Operations](#step-2-user-operations)
  - [Step 3: Order and Item Operations](#step-3-order-and-item-operations)
  - [Step 4: Querying with Multi-Attribute GSIs](#step-4-querying-with-multi-attribute-gsis)
  - [Running the Example](#running-the-example)
- [Multi-Attribute Keys for Hierarchical Data](#multi-attribute-keys-for-hierarchical-data)
- [Historical Context: Synthetic Key Prefixes](#historical-context-synthetic-key-prefixes)
- [Design Principles](#design-principles)
- [Practice Exercises](#practice-exercises)
- [Summary](#summary)
- [Additional Resources](#additional-resources)

---

## Overview

In the previous module, we introduced DynamoDB and learned how to perform basic operations with partition keys, sort keys, and Local Secondary Indexes. In this module, we explore how to model complex, multi-entity data in DynamoDB using **single-table design** and **multi-attribute keys** -- a GSI feature that lets you compose keys from multiple natural attributes, eliminating the need for synthetic string concatenation.

---

## Objectives

- Understand why DynamoDB data modeling differs from relational and MongoDB modeling
- Learn single-table design and how it stores multiple entity types in one table
- Understand Global Secondary Indexes and how they enable flexible query patterns
- Use multi-attribute keys to build GSIs with natural, typed attributes
- Understand sparse indexing and how it automatically filters entities across GSIs
- Model a real-world e-commerce scenario with users, orders, and line items

---

## Why Data Modeling Matters in DynamoDB

In relational databases and MongoDB, you can design your schema first and write queries later. The query engines are flexible enough to handle many different access patterns on the same data.

DynamoDB is different. Because it is optimized for **predictable performance at any scale**, it deliberately limits query flexibility:

- You **must always query by primary key** (no full-table scans for production workloads)
- **Joins do not exist** -- there is no equivalent of SQL `JOIN` or MongoDB `$lookup`
- You can only efficiently query attributes that are part of a **key or index**

This means you must **design your table schema around your access patterns**. You need to know the questions your application will ask before you design the table.

---

## Global Secondary Indexes (GSI)

A **Global Secondary Index** creates an automatically maintained alternate view of your table data with a different key structure. Unlike LSIs (which share the base table's partition key), a GSI can have a completely different partition key and sort key.

Key characteristics:
- Can be added to an existing table at any time
- Have their own throughput capacity (separate from the base table)
- Are **eventually consistent** (slight delay between base table write and GSI update)
- Can project all, some, or only key attributes from the base table
- Support **multi-attribute keys** (up to 4 attributes per partition key or sort key)

---

## Single-Table Design

### What is Single-Table Design?

Single-table design is a DynamoDB modeling strategy where **multiple entity types are stored in a single table**. Instead of having separate tables for users, orders, and items (as you would in a relational database), you put all of them in one DynamoDB table and use attributes and indexes to distinguish between and relate entity types.

Why use a single table?

1. **No joins in DynamoDB** -- with a single table and well-designed GSIs, you can retrieve related entities efficiently
2. **Fewer tables = simpler operations** -- one set of capacity settings, backups, and monitoring
3. **Efficient access patterns** -- retrieve a user and all their orders in a single GSI query

### Comparison with MongoDB Single Collection Pattern

You may recall the **Single Collection Pattern** from the MongoDB data modeling modules. DynamoDB's single-table design solves the same fundamental problem -- storing multiple related entity types together to avoid costly joins or multiple queries.

| Aspect | MongoDB Single Collection | DynamoDB Single-Table Design |
|--------|--------------------------|------------------------------|
| **Entity identification** | `docType` field | `entity_type` attribute |
| **Relationships** | `relatedTo` array field | Shared attribute values (e.g., `username`, `order_id`) |
| **Reverse lookups** | Query `relatedTo` array | GSI with multi-attribute keys |
| **Query flexibility** | Filter on any field | Must query by primary key or index |
| **Key structure** | `_id` (flexible) | Designed primary key + GSIs (fixed at creation) |

The key difference is that DynamoDB's approach is **more constrained** -- you must carefully design your key structure upfront. MongoDB gives you more flexibility to query ad hoc, but DynamoDB rewards careful design with better performance guarantees.

---

## Multi-Attribute Keys

### What are Multi-Attribute Keys?

Multi-attribute keys are a DynamoDB feature that allows **GSI partition keys and sort keys to be composed of up to four attributes each**. Instead of concatenating values into a single synthetic string, you define your GSI using multiple natural attributes from your domain model, and DynamoDB handles the composite key logic internally.

This is a **GSI-only feature** -- base table primary keys still support at most one partition key attribute and one sort key attribute.

In the `CreateTable` API, you specify multi-attribute keys by listing multiple attributes with the same key type (`HASH` for partition key, `RANGE` for sort key):

```python
global_secondary_indexes = [{
    'IndexName': 'my-index',
    'KeySchema': [
        {'AttributeName': 'username', 'KeyType': 'HASH'},         # PK
        {'AttributeName': 'entity_type', 'KeyType': 'RANGE'},     # SK attribute 1
        {'AttributeName': 'entity_id', 'KeyType': 'RANGE'}        # SK attribute 2
    ],
    'Projection': {'ProjectionType': 'ALL'}
}]
```

Two `RANGE` entries = multi-attribute sort key. DynamoDB composes them automatically.

Benefits:
- **No concatenation** when writing -- use natural domain attributes directly
- **No parsing** when reading -- attribute values come back as-is
- **No backfilling** -- adding a new multi-attribute GSI to an existing table works immediately because it indexes existing attributes
- **Type safety** -- attributes retain their original data types (string, number, etc.)

### Query Rules

**Partition key attributes** -- all must be specified with equality (`=`). You cannot omit any or use inequality operators.

**Sort key attributes** -- must be queried **left to right in order**. You cannot skip attributes. The first N can use equality, and the last one queried can use inequality (`>`, `>=`, `<`, `<=`, `BETWEEN`, `begins_with`).

```python
# Valid: PK only
Key('username').eq('aarchamb')

# Valid: PK + first sort key attribute
Key('username').eq('aarchamb') & Key('entity_type').eq('ORDER')

# Valid: PK + both sort key attributes
Key('username').eq('aarchamb') & Key('entity_type').eq('ORDER') & Key('entity_id').eq('abc123')

# Valid: PK + inequality on last sort key attribute queried
Key('username').eq('aarchamb') & Key('entity_type').eq('ORDER') & Key('entity_id').begins_with('abc')

# INVALID: skipping entity_type
Key('username').eq('aarchamb') & Key('entity_id').eq('abc123')
```

**Design tip:** Order sort key attributes from most general to most specific. In our design, `entity_type` comes first so we can filter by entity type before narrowing to a specific ID.

### Sparse Indexing

An important behavior of GSIs is **sparse indexing**: if an item in the base table is **missing any attribute** that is part of a GSI's key, that item will **not appear in the GSI**.

This is extremely useful for single-table design. Consider:
- A **user profile** has a `username` attribute but no `order_id` attribute
- A **line item** has an `order_id` attribute but no `username` attribute

With two GSIs:
- The **user-index** (PK = `username`) will contain user profiles and orders (which have `username`), but **not** line items (which lack `username`)
- The **order-index** (PK = `order_id`) will contain orders and line items (which have `order_id`), but **not** user profiles (which lack `order_id`)

Sparse indexing acts as a natural entity filter -- no application logic needed.

---

## Walkthrough: Users, Orders, and Items

> **Source code:** [dynamodb_modeling repository](https://github.com/zzenonn/dynamodb_modeling)

### Access Patterns

Before designing the table, we list the queries our application needs:

1. **Get a user profile** by username
2. **Get all orders** for a user
3. **Get everything** for a user (profile + orders)
4. **Get all line items** for an order
5. **Get a specific order** or item by type + ID

### Table Design

We store three entity types in one table. Each entity has natural, typed attributes. The base table uses `entity_type` + `entity_id` for direct lookups. Two multi-attribute GSIs handle the relational queries.

**Base table:**

| entity_type (PK) | entity_id (SK) | username | order_id | Other Attributes |
|-------------------|----------------|----------|----------|------------------|
| USER              | aarchamb       | aarchamb | --       | email, address   |
| ORDER             | abc123         | aarchamb | abc123   | address, status  |
| ORDER             | def456         | tgrimes1 | def456   | address, status  |
| ITEM              | itm001         | --       | abc123   | product_name, qty, price |
| ITEM              | itm002         | --       | abc123   | product_name, qty, price |
| ITEM              | itm003         | --       | def456   | product_name, qty, price |

Note the attribute patterns:
- **User profiles** have `username` but no `order_id`
- **Orders** have both `username` (links to user) and `order_id` (links to items)
- **Line items** have `order_id` but no `username`

**GSI1: user-index** (multi-attribute sort key)

| username (PK) | entity_type (SK1) | entity_id (SK2) | ... |
|----------------|-------------------|-----------------|-----|
| aarchamb       | ORDER             | abc123          | ... |
| aarchamb       | USER              | aarchamb        | ... |
| tgrimes1       | ORDER             | def456          | ... |
| tgrimes1       | USER              | tgrimes1        | ... |

Line items don't appear here (no `username` attribute -- sparse indexing).

**GSI2: order-index** (multi-attribute sort key)

| order_id (PK) | entity_type (SK1) | entity_id (SK2) | ... |
|----------------|-------------------|-----------------|-----|
| abc123         | ITEM              | itm001          | ... |
| abc123         | ITEM              | itm002          | ... |
| abc123         | ORDER             | abc123          | ... |
| def456         | ITEM              | itm003          | ... |
| def456         | ORDER             | def456          | ... |

User profiles don't appear here (no `order_id` attribute -- sparse indexing).

### Step 1: Create the Table

> **Source code:** [`create_users_orders_table.py`](https://github.com/zzenonn/dynamodb_modeling/blob/main/create_users_orders_table.py)

```python
import boto3

dynamodb = boto3.resource('dynamodb')

attribute_definitions = [
    {'AttributeName': 'entity_type', 'AttributeType': 'S'},
    {'AttributeName': 'entity_id', 'AttributeType': 'S'},
    {'AttributeName': 'username', 'AttributeType': 'S'},
    {'AttributeName': 'order_id', 'AttributeType': 'S'},
]

key_schema = [
    {'AttributeName': 'entity_type', 'KeyType': 'HASH'},
    {'AttributeName': 'entity_id', 'KeyType': 'RANGE'}
]

global_secondary_indexes = [
    {
        'IndexName': 'user-index',
        'KeySchema': [
            {'AttributeName': 'username', 'KeyType': 'HASH'},
            {'AttributeName': 'entity_type', 'KeyType': 'RANGE'},
            {'AttributeName': 'entity_id', 'KeyType': 'RANGE'}
        ],
        'Projection': {'ProjectionType': 'ALL'}
    },
    {
        'IndexName': 'order-index',
        'KeySchema': [
            {'AttributeName': 'order_id', 'KeyType': 'HASH'},
            {'AttributeName': 'entity_type', 'KeyType': 'RANGE'},
            {'AttributeName': 'entity_id', 'KeyType': 'RANGE'}
        ],
        'Projection': {
            'ProjectionType': 'INCLUDE',
            'NonKeyAttributes': [
                'quantity', 'price', 'status', 'product_name', 'username', 'address'
            ]
        }
    }
]

table = dynamodb.create_table(
    TableName='users-orders-items',
    KeySchema=key_schema,
    AttributeDefinitions=attribute_definitions,
    BillingMode='PAY_PER_REQUEST',
    GlobalSecondaryIndexes=global_secondary_indexes
)
```

Each GSI has two `RANGE` entries in its `KeySchema` -- that's what makes it a multi-attribute sort key. DynamoDB automatically composes `entity_type` + `entity_id` into the sort key and maintains the index as items are written.

### Step 2: User Operations

> **Source code:** [`user_ops.py`](https://github.com/zzenonn/dynamodb_modeling/blob/main/user_ops.py)

**Creating a user:**

```python
user = {
    'entity_type': 'USER',
    'entity_id': 'aarchamb',
    'username': 'aarchamb',       # Needed for user-index GSI
    'fullname': 'Adam Archambault',
    'email': 'aarchambault0@wikipedia.org',
    'address': {}
}
table.put_item(Item=user)
```

Every attribute is natural and self-documenting. The `username` attribute is present so this item appears in the user-index GSI. Note that `entity_id` and `username` are the same value for user profiles -- this redundancy is the cost of keeping attributes natural across entity types.

**Querying a user profile** (base table direct lookup):

```python
response = table.query(
    KeyConditionExpression=Key('entity_type').eq('USER') &
                           Key('entity_id').eq('aarchamb')
)
```

**Querying everything for a user** (user-index GSI):

```python
response = table.query(
    IndexName='user-index',
    KeyConditionExpression=Key('username').eq('aarchamb')
)
# Returns: user profile + all orders (line items excluded by sparse indexing)
```

### Step 3: Order and Item Operations

> **Source code:** [`order_ops.py`](https://github.com/zzenonn/dynamodb_modeling/blob/main/order_ops.py)

**Creating an order** (has both `username` and `order_id` so it appears in both GSIs):

```python
order = {
    'entity_type': 'ORDER',
    'entity_id': 'abc123',
    'username': 'aarchamb',       # Links to user (user-index GSI)
    'order_id': 'abc123',         # Links to items (order-index GSI)
    'address': 'home',
    'status': 'Placed'
}
table.put_item(Item=order)
```

**Adding a line item** (has `order_id` but no `username` -- only in order-index GSI):

```python
item = {
    'entity_type': 'ITEM',
    'entity_id': 'itm001',
    'order_id': 'abc123',         # Links to order (order-index GSI)
    'product_name': 'Nintendo Switch',
    'quantity': 1,
    'price': Decimal('368.95'),
    'status': 'Pending'
}
table.put_item(Item=item)
```

### Step 4: Querying with Multi-Attribute GSIs

**Get all orders for a user** (user-index GSI, filtering by entity_type):

```python
response = table.query(
    IndexName='user-index',
    KeyConditionExpression=Key('username').eq('aarchamb') &
                           Key('entity_type').eq('ORDER')
)
```

This uses the multi-attribute sort key: `entity_type = 'ORDER'` filters to only order entities. Without multi-attribute keys, the traditional approach would require synthetic prefixes and `begins_with('#ORDER#')`.

**Get all line items for an order** (order-index GSI, filtering by entity_type):

```python
response = table.query(
    IndexName='order-index',
    KeyConditionExpression=Key('order_id').eq('abc123') &
                           Key('entity_type').eq('ITEM')
)
```

**Get everything for an order** (order + all items):

```python
response = table.query(
    IndexName='order-index',
    KeyConditionExpression=Key('order_id').eq('abc123')
)
# Returns: the ORDER entity + all ITEM entities
```

### Running the Example

```bash
git clone https://github.com/zzenonn/dynamodb_modeling.git
cd dynamodb_modeling
python3 -m venv .env
source .env/bin/activate
pip install -r requirements.txt

python create_users_orders_table.py   # Create table with multi-attribute GSIs
python populate.py                     # Insert sample data
python read_queries.py                 # Run queries
python delete_users_orders_table.py    # Cleanup
deactivate
```

---

## Multi-Attribute Keys for Hierarchical Data

The walkthrough above shows multi-attribute keys used for entity-type filtering in a single-table design. But multi-attribute keys are especially powerful for **hierarchical data** where you want to query at different levels of specificity.

Consider a tournament match tracking system where matches have natural hierarchical attributes: tournament -> region -> round -> bracket.

**Table:** Simple `matchId` partition key on the base table.

**GSI with multi-attribute keys:**

```javascript
GlobalSecondaryIndexes: [{
    IndexName: 'TournamentRegionIndex',
    KeySchema: [
        { AttributeName: 'tournamentId', KeyType: 'HASH' },    // PK attribute 1
        { AttributeName: 'region', KeyType: 'HASH' },          // PK attribute 2
        { AttributeName: 'round', KeyType: 'RANGE' },          // SK attribute 1
        { AttributeName: 'bracket', KeyType: 'RANGE' },        // SK attribute 2
        { AttributeName: 'matchId', KeyType: 'RANGE' }         // SK attribute 3
    ],
    Projection: { ProjectionType: 'ALL' }
}]
```

This enables queries at every level of the hierarchy:

```javascript
// All matches for WINTER2024 in NA-EAST (both PK attributes required)
tournamentId = :t AND region = :r

// Only SEMIFINALS matches
tournamentId = :t AND region = :r AND round = :round

// SEMIFINALS in UPPER bracket
tournamentId = :t AND region = :r AND round = :round AND bracket = :bracket

// Range query: all rounds from QUARTERFINALS onward
tournamentId = :t AND region = :r AND round >= :round
```

Items use natural attributes -- no concatenation:

```javascript
const match = {
    matchId: 'match-001',
    tournamentId: 'WINTER2024',
    region: 'NA-EAST',
    round: 'SEMIFINALS',
    bracket: 'UPPER',
    player1Id: '101',
    player2Id: '103',
    matchDate: '2024-01-18',
    winner: '101',
    score: '3-2'
};
```

Other domains where hierarchical multi-attribute keys shine:
- **Time-series data:** `deviceId` + `locationId` (PK) with `year` + `month` + `day` + `timestamp` (SK) -- query at any time granularity
- **E-commerce analytics:** `sellerId` + `region` (PK) with `orderDate` + `category` + `orderId` (SK)
- **Organization structure:** `companyId` + `divisionId` (PK) with `departmentId` + `teamId` + `employeeId` (SK)

---

## Historical Context: Synthetic Key Prefixes

Before multi-attribute keys were available, the standard approach to single-table design used **generic key attribute names** (`pk`, `sk`) with **synthetic string prefixes** to encode entity type and relationships:

```python
# Traditional approach: synthetic concatenated keys
user  = {'pk': '#USER#aarchamb',  'sk': 'PROFILE',          'email': '...'}
order = {'pk': '#USER#aarchamb',  'sk': '#ORDER#abc123',    'status': 'Placed'}
item  = {'pk': '#ITEM#itm001',    'sk': '#ORDER#abc123',    'product_name': '...'}
```

An **inverted index** GSI swapped `pk` and `sk` to enable reverse lookups:

```python
# Inverted index GSI: sk becomes PK, pk becomes SK
response = table.query(
    IndexName='inverted-index',
    KeyConditionExpression=Key('sk').eq('#ORDER#abc123') &
                           Key('pk').begins_with('#ITEM#')
)
```

This works but has drawbacks:
- **String concatenation** when writing and **parsing** when reading
- **All keys are strings** -- you lose type safety on individual components
- **Backfilling required** -- adding a new GSI to an existing table requires updating all items with the new synthetic key
- **Generic attribute names** (`pk`, `sk`) are not self-documenting

Multi-attribute keys eliminate all of these issues. You will still encounter the synthetic key pattern in existing codebases and older tutorials, so it's important to recognize it. But for new designs, prefer multi-attribute keys.

---

## Design Principles

1. **Start with access patterns** -- list every query your application needs before designing your table. This is the most important step.

2. **Use single-table design** when entities are queried together. If you always fetch a user and their orders together, they belong in the same table with a GSI that groups them.

3. **Use multi-attribute GSI keys** with natural domain attributes. Let sparse indexing handle entity filtering rather than synthetic prefixes.

4. **Order sort key attributes from general to specific** to maximize query flexibility at each level.

5. **Accept attribute redundancy** as a trade-off. Orders storing both `entity_id` and `order_id` with the same value is the cost of keeping attributes natural and GSI-indexable across entity types.

6. **Keep items small** -- DynamoDB has a 400 KB item size limit (compared to MongoDB's 16 MB). Use references rather than embedding large nested structures.

7. **Denormalize strategically** -- since there are no joins, you may need to duplicate data across items. This is an intentional trade-off for read performance.

---

## Practice Exercises

1. Clone the [dynamodb_modeling repository](https://github.com/zzenonn/dynamodb_modeling) and run the complete example end-to-end. Trace each query and identify which key or GSI it uses.

2. Examine the output of `read_queries.py`. For each query result, explain why certain entity types appear and others don't (think about sparse indexing).

3. Add a new query to the example: "find all orders across all users that contain a specific product name." Can this be done with the existing GSIs? If not, what new GSI would you add?

4. Design a multi-attribute key GSI for an IoT application. Devices send temperature readings with attributes: `deviceId`, `locationId`, `year`, `month`, `day`, `timestamp`, and `temperature`. Define a GSI that supports these queries:
   - All readings for a device at a location
   - Readings for a specific year
   - Readings for a specific month within a year
   - Readings within a date range

5. Consider a university course registration system with students, courses, enrollments, and assignments. Design a single-table schema using multi-attribute GSIs. Define:
   - The base table key structure
   - At least two GSIs with multi-attribute keys
   - The access patterns each key supports
   - Which entities appear in which GSIs (sparse indexing)

6. Compare the DynamoDB single-table design with the MongoDB Single Collection Pattern from Module 8e. What are the trade-offs of each approach?

7. Take the tournament example from the hierarchical data section and implement it in Python using boto3. Create the table, insert sample matches, and write queries at each level of the hierarchy.

---

## Summary

- DynamoDB data modeling is **access-pattern-driven** -- design your keys around the queries your application needs
- **Single-table design** stores multiple entity types in one table, using GSIs to enable cross-entity queries
- **Multi-attribute keys** compose GSI partition and sort keys from multiple natural attributes (up to 4 each), eliminating synthetic string concatenation
- **Sparse indexing** automatically excludes items missing a GSI key attribute, acting as a natural entity-type filter
- Some attribute redundancy (e.g., `entity_id` and `order_id` having the same value on orders) is the trade-off for clean, typed, self-documenting keys
- The older synthetic prefix approach (`#USER#`, `#ORDER#`) still works but multi-attribute keys are preferred for new designs

---

## Additional Resources

- [DynamoDB Multi-Attribute Keys Documentation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.DesignPattern.MultiAttributeKeys.html)
- [Best Practices for DynamoDB Single-Table Design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-general-nosql-design.html)
- [Global Secondary Indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html)
- [Example Code: Single-Table with Multi-Attribute Keys](https://github.com/zzenonn/dynamodb_modeling)
- [Example Code: DynamoDB Basics (Music Table)](https://github.com/zzenonn/dynamodb_sample)
