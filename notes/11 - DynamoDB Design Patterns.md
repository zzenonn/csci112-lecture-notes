# DynamoDB Design Patterns

**Course:** CSCI 112 / 212 - Contemporary Databases  
**Topic:** DynamoDB Design Patterns -- From Single-Table to Aggregate-Oriented Design

---

## Table of Contents

- [Overview](#overview)
- [Objectives](#objectives)
- [Revisiting the Users-Orders-Items Design](#revisiting-the-users-orders-items-design)
  - [What We Built in Module 10](#what-we-built-in-module-09)
  - [Problems with the Everything-Table Approach](#problems-with-the-everything-table-approach)
- [Principle 1: Natural Keys Over Generic Identifiers](#principle-1-natural-keys-over-generic-identifiers)
- [Principle 2: Item Collections](#principle-2-item-collections)
  - [What is an Item Collection?](#what-is-an-item-collection)
  - [Identifying Relationships](#identifying-relationships)
- [Principle 3: Aggregate-Oriented Design](#principle-3-aggregate-oriented-design)
  - [Types of Aggregates](#types-of-aggregates)
  - [Deciding Aggregate Boundaries](#deciding-aggregate-boundaries)
  - [When to Use Multiple Tables](#when-to-use-multiple-tables)
- [Redesigning Users-Orders-Items](#redesigning-users-orders-items)
  - [Step 1: Identify Access Patterns](#step-1-identify-access-patterns)
  - [Step 2: Analyze Access Correlation](#step-2-analyze-access-correlation)
  - [Step 3: Define Aggregate Boundaries](#step-3-define-aggregate-boundaries)
  - [Step 4: Design Keys with Natural Names](#step-4-design-keys-with-natural-names)
  - [Step 5: Add GSIs with Multi-Attribute Keys](#step-5-add-gsis-with-multi-attribute-keys)
- [Additional Patterns](#additional-patterns)
  - [Short-Circuit Denormalization](#short-circuit-denormalization)
  - [Sparse GSIs](#sparse-gsis)
  - [Hierarchical Sort Keys](#hierarchical-sort-keys)
  - [Temporal Access Patterns](#temporal-access-patterns)
- [Comparison: Module 10 vs Improved Design](#comparison-module-09-vs-improved-design)
- [Practice Exercises](#practice-exercises)
- [Summary](#summary)
- [Additional Resources](#additional-resources)

---

## Overview

In Module 10, we built a single-table design that stores users, orders, and line items together using multi-attribute GSI keys. That design works and eliminates synthetic key concatenation, but it has limitations. In this module, we examine those limitations and learn **aggregate-oriented design patterns** that lead to cleaner, more maintainable, and more operationally sound DynamoDB models.

The central idea: **let your access patterns reveal natural aggregates, then design your tables around those aggregates** rather than forcing everything into one table.

---

## Objectives

- Identify the problems with generic "everything-table" designs
- Learn to use natural, self-documenting key names
- Understand item collections and identifying relationships
- Apply aggregate-oriented design with clear boundary decisions
- Know when to use multiple tables versus item collections
- Apply additional optimization patterns: denormalization, sparse GSIs, hierarchical sort keys

---

## Revisiting the Users-Orders-Items Design

### What We Built in Module 10

In Module 10, we stored three entity types in a single table:

```
Base table: entity_type (PK) + entity_id (SK)

| entity_type | entity_id | username | order_id | Other Attributes     |
|-------------|-----------|----------|----------|----------------------|
| USER        | aarchamb  | aarchamb | --       | email, address       |
| ORDER       | abc123    | aarchamb | abc123   | address, status      |
| ITEM        | itm001    | --       | abc123   | product_name, price  |
```

With two multi-attribute GSIs (user-index and order-index) for cross-entity queries. This design eliminates synthetic key prefixes and uses sparse indexing to filter entity types.

### Problems with the Everything-Table Approach

While the Module 10 design works, it has significant drawbacks as the application grows:

**1. Generic, non-descriptive keys.** `entity_type` and `entity_id` don't communicate what the data represents. A new developer looking at the table has to understand the entire design before they can write a query. Compare `entity_type = 'ORDER' AND entity_id = 'abc123'` with `order_id = 'abc123'` -- the second is immediately clear.

**2. Operational coupling.** All entity types share the same table, which means:
- **One DynamoDB Stream** with mixed entity types -- a consumer processing order events must filter out user and item events
- **One backup/restore unit** -- you cannot restore only orders without also restoring users and items
- **Shared scaling** -- a spike in item writes affects the capacity available for user reads
- **One set of IAM permissions** -- you cannot restrict a microservice to only access orders

**3. Attribute redundancy without clear benefit.** Orders store both `entity_id = 'abc123'` and `order_id = 'abc123'` (the same value), because the generic base table key and the GSI key serve different roles. With natural keys, this redundancy disappears.

**4. Every cross-entity query requires a GSI.** Because the base table is organized by generic entity type, even the most common query ("get a user and their orders") requires a GSI lookup. With item collections, this query hits the base table directly.

---

## Principle 1: Natural Keys Over Generic Identifiers

Your keys should describe what they identify:

| Instead of... | Use... |
|----------------|--------|
| `entity_type` | `user_id`, `order_id`, `product_sku` |
| `entity_id` | The actual identifier for the entity |
| `sort_key` | A meaningful name like `review_id`, `lesson_id` |
| `user-index` (GSI) | `OrdersByCustomer` (GSI) |
| `order-index` (GSI) | `ItemsByOrder` (GSI) |

Self-documenting names make the design immediately understandable. When a new developer sees `PK: user_id, SK: "ORDER#abc123"`, they understand the data relationship instantly.

---

## Principle 2: Item Collections

### What is an Item Collection?

An **item collection** is a group of items that share the same partition key value. Within an item collection, items are distinguished by their sort key. This is DynamoDB's native mechanism for grouping related data.

```
UserOrders table
┌──────────────┬─────────────────┬──────────────────────────────┐
│ user_id (PK) │ sk              │ Attributes                   │
├──────────────┼─────────────────┼──────────────────────────────┤
│ aarchamb     │ PROFILE         │ email, address, fullname     │
│ aarchamb     │ ORDER#abc123    │ address, status, created_at  │
│ aarchamb     │ ORDER#def456    │ address, status, created_at  │
│ tgrimes1     │ PROFILE         │ email, address, fullname     │
│ tgrimes1     │ ORDER#xyz789    │ address, status, created_at  │
└──────────────┴─────────────────┴──────────────────────────────┘
```

Query patterns enabled:
- **Get user profile only:** `GetItem(user_id = 'aarchamb', sk = 'PROFILE')`
- **Get user + all orders:** `Query(user_id = 'aarchamb')` -- single query, no GSI
- **Get specific order:** `GetItem(user_id = 'aarchamb', sk = 'ORDER#abc123')`
- **Get only orders:** `Query(user_id = 'aarchamb', sk begins_with 'ORDER#')`

The critical advantage: **related data is retrieved in a single base table query** with no GSI needed. This is faster (strongly consistent reads available), cheaper (no GSI storage or write amplification), and simpler.

### Identifying Relationships

An **identifying relationship** exists when a child entity cannot exist without its parent. Orders cannot exist without a user. Reviews cannot exist without a product. In these cases, use the parent's ID as the partition key and the child's ID as the sort key.

**Standard approach (more expensive):**
```
Orders table: PK = order_id
GSI needed:   PK = user_id  (to query "orders by user")
Cost: base table writes + GSI writes + GSI storage
```

**Identifying relationship approach (cost optimized):**
```
UserOrders table: PK = user_id, SK = "ORDER#" + order_id
No GSI needed: Query directly by user_id
Cost savings: ~50% reduction (no GSI overhead)
```

Use identifying relationships when:
1. The parent ID is always available when querying children
2. You need to query all children for a given parent
3. Children are meaningless without their parent context

---

## Principle 3: Aggregate-Oriented Design

### Types of Aggregates

DynamoDB supports two levels of aggregation:

**1. Single Item Aggregate** -- multiple related entities combined into one DynamoDB item.
- Atomic updates across all data
- Single `GetItem` retrieval
- Subject to 400 KB item size limit
- Best when: >90% of queries need all the data together, combined size is bounded

**2. Item Collection Aggregate** -- related entities share a partition key but are stored as separate items with different sort keys.
- Flexible: access individual entities or the whole collection
- No size constraint on the collection (each item still limited to 400 KB)
- Query the collection with a single `Query` operation
- Best when: 50-90% of queries need related data together, entities updated independently

### Deciding Aggregate Boundaries

Use this decision framework to determine how entities should be grouped:

**Step 1: Analyze Access Correlation**
- **>90% accessed together** -> Single item aggregate candidate
- **50-90% accessed together** -> Item collection aggregate candidate
- **<50% accessed together** -> Separate tables

**Step 2: Check Constraints**
- **Size:** Will combined size exceed 100 KB? -> Item collection or separate tables
- **Updates:** Different update frequencies? -> Item collection (not single item)
- **Atomicity:** Need atomic updates across entities? -> Single item aggregate

**Step 3: Check Operational Coupling**
- Same backup/restore needs? -> Item collection is fine
- Different scaling patterns? -> Separate tables
- Need separate event streams? -> Separate tables
- Different access control? -> Separate tables

### When to Use Multiple Tables

Use separate tables when entities have:

- **Independent scaling needs** -- user profile reads vs. order write spikes
- **Different operational requirements** -- separate backup schedules, different TTL policies
- **Distinct event processing** -- order events processed by one service, user events by another
- **Low access correlation** -- less than 50% of queries need both entities together

Benefits of multiple tables:
- **Lower blast radius** -- table-level issues affect only one entity type
- **Granular backup/restore** -- restore orders without touching users
- **Clear cost attribution** -- understand costs per business domain
- **Clean event streams** -- each table's DynamoDB Stream contains logically related events
- **Natural service boundaries** -- microservices own domain-specific tables

---

## Redesigning Users-Orders-Items

Let's apply these principles to redesign the Module 10 example.

### Step 1: Identify Access Patterns

| # | Access Pattern | Type | Frequency |
|---|---------------|------|-----------|
| 1 | Get user profile by username | Read | High |
| 2 | Get all orders for a user | Read | High |
| 3 | Get user profile + recent orders | Read | High |
| 4 | Get all line items for an order | Read | High |
| 5 | Create a new user | Write | Low |
| 6 | Place an order (create order + items) | Write | Medium |
| 7 | Update order status | Write | Medium |
| 8 | Update item status | Write | Medium |

### Step 2: Analyze Access Correlation

**User + Orders:**
- Patterns #1, #2, #3 involve users and orders together
- ~60% of reads need both user profile and orders
- Orders cannot exist without users (identifying relationship)
- **Decision: Item Collection Aggregate**

**Order + Items:**
- Pattern #4 always fetches items with order context
- Items are always created with an order (Pattern #6)
- But items have different update patterns (Pattern #8)
- Items could grow large (many items per order)
- **Decision: Separate table with identifying relationship** -- items always belong to an order, but keeping them separate gives cleaner streams and independent scaling

### Step 3: Define Aggregate Boundaries

Two tables:

1. **UserOrders** -- users and their orders in an item collection
2. **OrderItems** -- line items, using order as the identifying relationship

### Step 4: Design Keys with Natural Names

**UserOrders table:**

| user_id (PK) | sk | email | address | status | created_at |
|---------------|----|-------|---------|--------|------------|
| aarchamb | PROFILE | aarchambault0@... | {home: ...} | -- | -- |
| aarchamb | ORDER#abc123 | -- | home | Placed | 2024-01-15 |
| aarchamb | ORDER#def456 | -- | office | Placed | 2024-01-20 |
| tgrimes1 | PROFILE | tgrimes1@... | {home: ...} | -- | -- |
| tgrimes1 | ORDER#xyz789 | -- | home | Placed | 2024-02-01 |

- `user_id` is the natural partition key (self-documenting)
- Sort key uses prefixes: `PROFILE` for user data, `ORDER#<id>` for orders
- Queries #1, #2, #3 all hit the base table directly -- no GSI needed

```python
# Get user profile (Pattern #1)
table.get_item(Key={'user_id': 'aarchamb', 'sk': 'PROFILE'})

# Get all orders for a user (Pattern #2)
table.query(
    KeyConditionExpression=Key('user_id').eq('aarchamb') &
                           Key('sk').begins_with('ORDER#')
)

# Get user profile + orders (Pattern #3)
table.query(
    KeyConditionExpression=Key('user_id').eq('aarchamb')
)
```

**OrderItems table:**

| order_id (PK) | item_id (SK) | product_name | quantity | price | status |
|----------------|--------------|--------------|----------|-------|--------|
| abc123 | itm001 | New Apple MacBook Pro | 1 | 2018.42 | Pending |
| abc123 | itm002 | Nintendo Switch | 1 | 368.95 | Pending |
| abc123 | itm003 | Seagate Portable 2TB | 1 | 59.95 | Pending |
| def456 | itm004 | HyperX Fury 16GB | 2 | 39.99 | Shipped |

- `order_id` is the partition key (identifying relationship -- items belong to an order)
- `item_id` is the sort key
- Pattern #4 hits the base table directly -- no GSI needed

```python
# Get all items for an order (Pattern #4)
table.query(
    KeyConditionExpression=Key('order_id').eq('abc123')
)
```

### Step 5: Add GSIs with Multi-Attribute Keys

What if we need access patterns not served by the base table keys? For example:

- "Find all orders with status 'Placed' for a user, sorted by date"
- "Find all items with status 'Pending' for an order"

These require GSIs. Use multi-attribute keys for clean, typed indexes:

**UserOrders GSI: OrdersByStatus**

```python
{
    'IndexName': 'OrdersByStatus',
    'KeySchema': [
        {'AttributeName': 'user_id', 'KeyType': 'HASH'},
        {'AttributeName': 'status', 'KeyType': 'RANGE'},
        {'AttributeName': 'created_at', 'KeyType': 'RANGE'}
    ],
    'Projection': {'ProjectionType': 'ALL'}
}
```

```python
# Orders by user and status, sorted by date
table.query(
    IndexName='OrdersByStatus',
    KeyConditionExpression=Key('user_id').eq('aarchamb') &
                           Key('status').eq('Placed')
)

# Orders in a date range (inequality must be last)
table.query(
    IndexName='OrdersByStatus',
    KeyConditionExpression=Key('user_id').eq('aarchamb') &
                           Key('status').eq('Placed') &
                           Key('created_at').between('2024-01-01', '2024-01-31')
)
```

This GSI uses a multi-attribute sort key: `status` + `created_at`. Place the equality-filtered attribute (`status`) before the range-filtered attribute (`created_at`), because **inequality conditions must be the last condition**.

---

## Additional Patterns

### Short-Circuit Denormalization

Duplicate a frequently needed attribute from a related entity to avoid an extra lookup:

```python
# OrderItems table: include product_name from Products table
item = {
    'order_id': 'abc123',
    'item_id': 'itm001',
    'product_id': 'prod-macbook',
    'product_name': 'New Apple MacBook Pro',  # Denormalized from Products
    'quantity': 1,
    'price': Decimal('2018.42')
}
```

Use this when:
1. The access pattern would otherwise require a join from another table
2. The duplicated attribute is mostly immutable (product names rarely change)
3. The attribute is small and doesn't significantly impact write cost

The trade-off: if the source value changes, you need to update all copies. Only denormalize attributes that change infrequently.

### Sparse GSIs

DynamoDB only writes a GSI entry when **all** key attributes exist on the item. If an attribute is missing, the item is excluded from the GSI. This is useful for indexing a minority of items.

Example: only index orders that are currently active:

```python
# Add 'active_status' attribute only to active orders
active_order = {
    'user_id': 'aarchamb',
    'sk': 'ORDER#abc123',
    'status': 'Processing',
    'active_status': 'Processing'  # Present -> appears in GSI
}

completed_order = {
    'user_id': 'aarchamb',
    'sk': 'ORDER#def456',
    'status': 'Delivered'
    # No 'active_status' -> excluded from GSI
}
```

A GSI with `active_status` as the sort key automatically contains only active orders. When an order is delivered, remove the `active_status` attribute and it disappears from the GSI. This saves storage and write costs -- if only 5% of orders are active, the GSI is 95% smaller.

### Hierarchical Sort Keys

For base table sort keys that encode hierarchy, use composite strings (since multi-attribute keys on base tables are a newer feature -- check current AWS documentation for support in your region):

```
LocationData table
PK: device_id
SK: "2024#01#15#14:30:00"  (year#month#day#time)
```

Query patterns enabled:
```python
# All readings for a device
Query(device_id = 'sensor-01')

# Readings for January 2024
Query(device_id = 'sensor-01', sk begins_with '2024#01')

# Readings for a specific day
Query(device_id = 'sensor-01', sk begins_with '2024#01#15')

# Readings in a time range
Query(device_id = 'sensor-01', sk BETWEEN '2024#01#15#00:00' AND '2024#01#15#23:59')
```

For GSIs on the same data, use multi-attribute keys instead of composite strings:

```python
# GSI with multi-attribute sort key for the same hierarchy
{
    'IndexName': 'LocationTimeIndex',
    'KeySchema': [
        {'AttributeName': 'location_id', 'KeyType': 'HASH'},
        {'AttributeName': 'year', 'KeyType': 'RANGE'},
        {'AttributeName': 'month', 'KeyType': 'RANGE'},
        {'AttributeName': 'day', 'KeyType': 'RANGE'},
        {'AttributeName': 'device_id', 'KeyType': 'RANGE'}
    ],
    'Projection': {'ProjectionType': 'ALL'}
}
```

### Temporal Access Patterns

DynamoDB has no dedicated datetime type. Choose your format based on query needs:

| Format | Example | Best For |
|--------|---------|----------|
| ISO 8601 string | `"2024-01-15T14:30:00Z"` | Human-readable, natural sort order, business applications |
| Unix epoch (number) | `1705326600` | Compact storage, math operations, TTL attribute, high-precision timestamps |

Use ISO 8601 strings as sort keys for chronological ordering. Use numeric timestamps for TTL (DynamoDB TTL requires Unix epoch seconds).

---

## Comparison: Module 10 vs Improved Design

| Aspect | Module 10 (Everything-Table) | Improved (Aggregate-Oriented) |
|--------|------------------------------|-------------------------------|
| **Tables** | 1 table, 3 entity types | 2 tables (UserOrders, OrderItems) |
| **Base table keys** | Generic: `entity_type`, `entity_id` | Natural: `user_id` + `sk`, `order_id` + `item_id` |
| **Get user + orders** | Requires GSI (user-index) | Base table query (no GSI) |
| **Get items for order** | Requires GSI (order-index) | Base table query (no GSI) |
| **GSIs needed** | 2 (minimum, for basic queries) | 0 (for basic queries), add only for advanced patterns |
| **Attribute redundancy** | `entity_id` = `order_id` on orders | Minimal (sort key prefix only) |
| **DynamoDB Streams** | Mixed events (users + orders + items) | Separate streams per table |
| **Backup/Restore** | All or nothing | Per-table granularity |
| **Readability** | Requires understanding the whole design | Self-documenting key names |
| **Cost** | GSI storage + write amplification for basic queries | Lower (basic queries on base table) |

The improved design is:
- **Cheaper** -- fewer GSIs for common queries
- **Faster** -- base table queries support strongly consistent reads; GSIs are eventually consistent
- **Cleaner** -- natural keys, separate concerns, self-documenting
- **More operationally sound** -- independent scaling, backup, and streaming per table

---

## Practice Exercises

1. Take the Module 10 `dynamodb_modeling` code and rewrite it to use the improved two-table design (UserOrders + OrderItems). Compare the query code for readability.

2. A social media application has users, posts, comments, and likes. Analyze the access correlation between these entities and decide which should be item collections and which should be separate tables. Justify your decisions using the aggregate boundary framework.

3. Design a GSI with multi-attribute keys for the UserOrders table that supports this query: "Find all orders for a user with a specific status, sorted by creation date." Write the `CreateTable` GSI definition and the `Query` call.

4. An e-commerce product catalog has products, reviews, and pricing history. Products are updated daily, reviews are added hourly, and pricing changes weekly. Using the operational coupling criteria, decide on the table structure. Would you use one, two, or three tables?

5. A ride-sharing application needs these access patterns:
   - Get a driver's profile
   - Get all rides for a driver, sorted by date
   - Get all rides for a rider, sorted by date
   - Get ride details with all waypoints
   - Find active rides in a city

   Design the table structure, keys, and GSIs. Which entities form item collections? Where do you use multi-attribute keys? Where do you use sparse GSIs?

6. Take the IoT sensor design from the hierarchical sort keys section. Add a sparse GSI that indexes only readings that exceed a threshold (anomaly detection). How would you implement this using the sparse GSI pattern?

---

## Summary

- **Avoid the everything-table anti-pattern** -- mixing unrelated entities creates operational coupling, mixed streams, and generic keys that are hard to understand
- **Use natural, self-documenting key names** -- `user_id` and `order_id` over `entity_type` and `entity_id`
- **Item collections** group related entities under a shared partition key, enabling single-query retrieval without GSIs
- **Identifying relationships** (PK = parent_id, SK = child_id) eliminate the need for GSIs on parent-child queries, saving ~50% on writes and storage
- **Aggregate-oriented design** uses access pattern correlation to decide boundaries: >90% -> single item, 50-90% -> item collection, <50% -> separate tables
- **Multiple tables** are preferred when entities have different operational needs (scaling, streams, backups, permissions)
- **Multi-attribute keys on GSIs** remain the right choice for indexed queries that need hierarchical filtering
- **Additional patterns** (denormalization, sparse GSIs, hierarchical sort keys) optimize for specific access pattern needs

The Module 10 design taught you the mechanics of multi-attribute keys and single-table design. This module taught you **when and how to apply them** within a principled design framework. The best DynamoDB models emerge from understanding your access patterns first, then choosing the simplest key structure that serves them.

---

## Additional Resources

- [DynamoDB Best Practices for Design](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-general-nosql-design.html)
- [DynamoDB Multi-Attribute Keys](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.DesignPattern.MultiAttributeKeys.html)
- [DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)
- [Rick Houlihan - Advanced DynamoDB Design Patterns (re:Invent)](https://www.youtube.com/results?search_query=rick+houlihan+dynamodb+design+patterns)
