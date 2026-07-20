---
theme: discs
title: "DynamoDB Data Modeling"
highlighter: shiki
layout: cover
codeCopy: true
---

DynamoDB Data Modeling

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Explain why DynamoDB data modeling requires knowing access patterns upfront
2. Understand **Global Secondary Indexes** and how they differ from LSIs
3. Apply **single-table design** to store multiple entity types in one table
4. Use **multi-attribute keys** to compose natural GSI keys without string concatenation
5. Use **sparse indexing** to automatically filter entities across GSIs

---
layout: section
---

# Why Access Patterns Come First

---

# DynamoDB Query Constraints

In relational databases and MongoDB, you can design the schema first and write queries later. The query engine is flexible enough to handle many access patterns on the same data.

**DynamoDB is different:**

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:0.9rem; font-size:0.86rem;">
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">What DynamoDB cannot do</div>
    Full-table scans in production<br>
    JOIN across tables<br>
    Filter by arbitrary attribute without an index
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">What DynamoDB guarantees</div>
    Single-digit millisecond latency<br>
    Consistent performance at any scale<br>
    Any query that uses a key or index
  </div>
</div>

<div style="margin-top:0.9rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 1rem; font-size:0.86rem;">
  The constraint is the guarantee. DynamoDB achieves predictable performance by restricting what queries are allowed. You must design your key schema around your access patterns before you write any code.
</div>

---

# Global Secondary Indexes (GSI)

A GSI creates an **automatically maintained alternate view** of your table with a different key structure.

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:0.9rem; font-size:0.85rem;">
  <div style="border:1.5px solid #94a3b8; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.3rem;">LSI (revisited)</div>
    Same partition key as base table<br>
    Different sort key<br>
    Must be defined at table creation<br>
    Max 5 per table
  </div>
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.8rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#0369a1;">GSI</div>
    Completely different partition key<br>
    Optional different sort key<br>
    Can be added any time<br>
    Eventually consistent; has own capacity
  </div>
</div>

<div style="margin-top:0.9rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.5rem 1rem; font-size:0.86rem;">
  GSIs support <strong>multi-attribute keys</strong> — up to 4 attributes can compose each key, eliminating the need for synthetic string prefixes.
</div>

---
layout: section
---

# Single-Table Design

---

# What is Single-Table Design?

Store **multiple entity types** in one DynamoDB table. Use attributes and indexes to distinguish and relate them.

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:0.9rem; font-size:0.85rem;">
  <div style="border:1.5px solid #94a3b8; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.3rem;">Multi-table (relational mindset)</div>
    Separate tables for Users, Orders, Items<br>
    Requires multiple queries or JOIN<br>
    Each table has its own capacity settings
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">Single-table</div>
    All entities in one table<br>
    Related data retrieved in one GSI query<br>
    One set of capacity settings and backups
  </div>
</div>

<div style="margin-top:0.9rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.85rem;">
  Same idea as MongoDB's <strong>Single Collection Pattern</strong> — store related entities together to avoid costly joins or multiple queries.
</div>

---

# Single-Table vs Single Collection (MongoDB)

| Aspect | MongoDB Single Collection | DynamoDB Single-Table |
|--------|--------------------------|----------------------|
| Entity identification | `docType` field | `entity_type` attribute |
| Relationships | `relatedTo` array | Shared attribute values |
| Reverse lookups | Query `relatedTo` | GSI with multi-attribute keys |
| Query model | MQL — any field | Must use primary key or index |

---
layout: section
---

# Multi-Attribute Keys

---

# The Problem with Synthetic Keys

The old approach to single-table design used **string concatenation** to encode multiple values into one key attribute:

```python
# Synthetic key prefix — old pattern
{ "PK": "USER#aarchamb",  "SK": "ORDER#abc123" }
{ "PK": "USER#aarchamb",  "SK": "ITEM#itm001" }

# Querying required string matching:
Key('PK').eq('USER#aarchamb') & Key('SK').begins_with('ORDER#')
# Parsing on read, backfill on schema change, type coercion problems
```

<div style="margin-top:0.7rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.85rem;">
  Multi-attribute keys eliminate this entirely — compose GSI keys from natural, typed attributes without concatenation.
</div>

---

# Multi-Attribute Keys in Practice

**1 partition key + 2 sort keys** on one GSI — so the same index answers queries at three levels of specificity.

<div style="font-size:0.72rem; margin-top:0.3rem;">

```python
global_secondary_indexes = [{ 'IndexName': 'tournament-index', 'KeySchema': [
    {'AttributeName': 'tournament_id', 'KeyType': 'HASH'},   # PK
    {'AttributeName': 'round',         'KeyType': 'RANGE'},  # SK1
    {'AttributeName': 'match_id',      'KeyType': 'RANGE'}], # SK2
  'Projection': {'ProjectionType': 'ALL'} }]
```

</div>

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.8rem; align-items:start; margin-top:0.4rem; font-size:0.68rem;">
<div>

**PK** → whole tournament

```python
Key('tournament_id')
  .eq('WINTER2024')
```

</div>
<div>

**PK + SK1** → one round

```python
Key('tournament_id')
  .eq('WINTER2024') &
Key('round')
  .eq('SEMIFINALS')
```

</div>
<div>

**PK + SK1 + SK2** → a match

```python
Key('tournament_id')
  .eq('WINTER2024') &
Key('round')
  .eq('SEMIFINALS') &
Key('match_id')
  .eq('match-001')
```

</div>
</div>

<div style="font-size:0.74rem; margin-top:0.35rem; color:#475569;">Left to right — stop at any level, but you can't skip <code>round</code> (SK1) to query <code>match_id</code> (SK2) directly.</div>

---

# Sparse Indexing

If an item **lacks any attribute** that is part of a GSI key, that item **does not appear in the GSI** — automatically. Example: only **completed** matches carry a `winner` attribute, so a GSI on `winner` holds only finished matches.

<div style="overflow-x:auto; margin-top:0.9rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.84rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Match</th>
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Status</th>
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Has <code>winner</code>?</th>
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">winner-index GSI</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">match-001</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Completed</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">✓</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">Included</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">match-002</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Completed</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">✓</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">Included</td>
    </tr>
    <tr>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">match-003</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Scheduled</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#dc2626;">✗</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#dc2626;">Excluded</td>
    </tr>
  </tbody>
</table>
</div>

<div style="margin-top:0.7rem; font-size:0.85rem; color:#475569;">Querying <code>winner-index</code> returns only completed matches — sparse indexing acts as a natural filter, no application logic needed.</div>

---
layout: section
---

# Case Study: E-Commerce

---

# Step 1: List the Access Patterns

Design the table **around the queries** the application must serve. List them first:

<div style="margin-top:0.8rem; font-size:1.02rem;">

1. **Get user profile**
2. **Get orders for user**
3. **Get single order and order items**
4. **Get orders for user by status and date**
5. **Get all pending orders** <span style="color:#ca8a04;">✳</span>

</div>

<div style="margin-top:1rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 1rem; font-size:0.86rem;">
  <span style="color:#ca8a04;">✳</span> Pattern 5 is left as an <strong>exercise</strong> — it needs a <strong>sparse status index</strong> (<code>status</code> as PK, only orders carry it). Apply the same multi-attribute technique shown for patterns 1–4.
</div>

---

# Table Design

Three entity types, one table. Base table: `entity_type` (PK) + `entity_id` (SK).

<div style="margin-top:0.5rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.8rem; font-family:'Roboto Condensed',sans-serif;">
  <thead>
    <tr style="background:#e2e8f0;">
      <th colspan="2" style="border:1px solid #9e9e9e; padding:0.3rem; text-align:center;">Primary Key</th>
      <th colspan="3" style="border:1px solid #9e9e9e; padding:0.3rem; text-align:center;">Attributes</th>
    </tr>
    <tr style="background:#f1f5f9;">
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">entity_type (PK)</th>
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">entity_id (SK)</th>
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">username</th>
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">order_id</th>
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">Other attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">USER</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">aarchamb</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">aarchamb</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; color:#94a3b8;">—</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">email, address</td>
    </tr>
    <tr>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">ORDER</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">abc123</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">aarchamb</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">abc123</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">address, status</td>
    </tr>
    <tr>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">ORDER</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">def456</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">tgrimes1</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">def456</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">address, status</td>
    </tr>
    <tr>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">ITEM</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">itm001</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; color:#94a3b8;">—</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">abc123</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">product_name, qty, price</td>
    </tr>
    <tr>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">ITEM</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">itm003</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; color:#94a3b8;">—</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">def456</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">product_name, qty, price</td>
    </tr>
  </tbody>
</table>
</div>

<div style="font-size:0.8rem; margin-top:0.5rem; color:#475569;">
Two GSIs, multi-attribute sort keys — <strong>user-index</strong>: <code>username</code> (PK) + <code>entity_type</code> + <code>entity_id</code> · <strong>order-index</strong>: <code>order_id</code> (PK) + <code>entity_type</code> + <code>entity_id</code>
</div>

---

# Access Pattern Queries

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.9rem; align-items:start; margin-top:0.5rem; font-size:0.78rem;">
<div>

**Everything for a user** — profile + orders (items excluded by sparse indexing):

```python
table.query(
    IndexName='user-index',
    KeyConditionExpression=
        Key('username')
        .eq('aarchamb')
)
```

</div>
<div>

**Only orders for a user** — filter by `entity_type` in the multi-attribute SK:

```python
table.query(
    IndexName='user-index',
    KeyConditionExpression=
        Key('username')
        .eq('aarchamb') &
        Key('entity_type')
        .eq('ORDER')
)
```

</div>
<div>

**All line items for an order** — query the `order-index`:

```python
table.query(
    IndexName='order-index',
    KeyConditionExpression=
        Key('order_id')
        .eq('abc123') &
        Key('entity_type')
        .eq('ITEM')
)
```

</div>
</div>

---

# Access Pattern 2: Get Orders for User

**user-index** groups a user's profile **and** their orders into one item collection — the shared `username` partition key. One query returns both (items excluded by sparse indexing).

<div style="margin-top:0.5rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.8rem; font-family:'Roboto Condensed',sans-serif;">
  <thead>
    <tr style="background:#e2e8f0;">
      <th colspan="3" style="border:1px solid #9e9e9e; padding:0.3rem; text-align:center;">Primary Key (user-index GSI)</th>
      <th colspan="2" style="border:1px solid #9e9e9e; padding:0.3rem; text-align:center;">Attributes</th>
    </tr>
    <tr style="background:#f1f5f9;">
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">username (PK)</th>
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">entity_type (SK1)</th>
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">entity_id (SK2)</th>
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">order_id</th>
      <th style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; text-align:left;">Other attributes</th>
    </tr>
  </thead>
  <tbody>
    <tr style="background:#f0f9ff;">
      <td rowspan="2" style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; vertical-align:middle; font-weight:600;">aarchamb</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">ORDER</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">abc123</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">abc123</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">address, status</td>
    </tr>
    <tr style="background:#f0f9ff;">
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">USER</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">aarchamb</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; color:#94a3b8;">—</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">email, fullname, address</td>
    </tr>
    <tr style="background:#f0fdf4;">
      <td rowspan="2" style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; vertical-align:middle; font-weight:600;">tgrimes1</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">ORDER</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">def456</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">def456</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">address, status</td>
    </tr>
    <tr style="background:#f0fdf4;">
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">USER</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">tgrimes1</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem; color:#94a3b8;">—</td>
      <td style="border:1px solid #9e9e9e; padding:0.3rem 0.6rem;">email, fullname, address</td>
    </tr>
  </tbody>
</table>
</div>

<div style="font-size:0.78rem; margin-top:0.4rem; color:#475569;">Each merged <code>username</code> block = one item collection returned by a single query.</div>

---

# Access Pattern 3: Get Single Order and Items

**order-index** inverts the relationship — `order_id` becomes the partition key, grouping the **order and all its line items** into one collection. This is the classic **inverted index** pattern.

<div style="margin-top:0.5rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.76rem; font-family:'Roboto Condensed',sans-serif;">
  <thead>
    <tr style="background:#e2e8f0;">
      <th colspan="3" style="border:1px solid #9e9e9e; padding:0.25rem; text-align:center;">Primary Key (order-index GSI)</th>
      <th colspan="2" style="border:1px solid #9e9e9e; padding:0.25rem; text-align:center;">Attributes</th>
    </tr>
    <tr style="background:#f1f5f9;">
      <th style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; text-align:left;">order_id (PK)</th>
      <th style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; text-align:left;">entity_type (SK1)</th>
      <th style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; text-align:left;">entity_id (SK2)</th>
      <th style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; text-align:left;">product_name</th>
      <th style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; text-align:left;">qty, price, status</th>
    </tr>
  </thead>
  <tbody>
    <tr style="background:#f0f9ff;">
      <td rowspan="3" style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; vertical-align:middle; font-weight:600;">abc123</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">ITEM</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">itm001</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">Nintendo Switch</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">1, 368.95, Filled</td>
    </tr>
    <tr style="background:#f0f9ff;">
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">ITEM</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">itm002</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">Seagate 2TB External</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">1, 59.95, Filled</td>
    </tr>
    <tr style="background:#f0f9ff;">
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">ORDER</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">abc123</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; color:#94a3b8;">—</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">address, status</td>
    </tr>
    <tr style="background:#f0fdf4;">
      <td rowspan="2" style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; vertical-align:middle; font-weight:600;">def456</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">ITEM</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">itm003</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">HyperX Fury 16GB</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">2, 39.99, Filled</td>
    </tr>
    <tr style="background:#f0fdf4;">
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">ORDER</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">def456</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem; color:#94a3b8;">—</td>
      <td style="border:1px solid #9e9e9e; padding:0.25rem 0.6rem;">address, status</td>
    </tr>
  </tbody>
</table>
</div>

<div style="font-size:0.76rem; margin-top:0.4rem; color:#475569;">Query <code>order_id = 'abc123'</code> on order-index → the order plus every line item, in one call.</div>

---

# Access Pattern 4: Status & Date

**status-date-index**: `username` (PK) + `status_date` (SK) — composed **status first, then date** (`Placed#2024-01-15`), so orders group by status with dates ordered within.

<div style="margin-top:0.4rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.72rem; font-family:'Roboto Condensed',sans-serif;">
  <thead>
    <tr style="background:#e2e8f0;">
      <th colspan="2" style="border:1px solid #9e9e9e; padding:0.12rem; text-align:center;">Primary Key (status-date-index)</th>
      <th colspan="2" style="border:1px solid #9e9e9e; padding:0.12rem; text-align:center;">Attributes</th>
    </tr>
    <tr style="background:#f1f5f9;">
      <th style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem; text-align:left;">username (PK)</th>
      <th style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem; text-align:left;">status_date (SK)</th>
      <th style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem; text-align:left;">entity_id</th>
      <th style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem; text-align:left;">address</th>
    </tr>
  </thead>
  <tbody>
    <tr style="background:#f0f9ff;">
      <td rowspan="3" style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem; vertical-align:middle; font-weight:600;">aarchamb</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">Placed#2024-01-15</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">abc123</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">Home</td>
    </tr>
    <tr style="background:#f0f9ff;">
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">Placed#2024-02-28</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">ghi789</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">Office</td>
    </tr>
    <tr style="background:#f0f9ff;">
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">Shipped#2024-01-03</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">jkl012</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">Home</td>
    </tr>
    <tr style="background:#f0fdf4;">
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem; font-weight:600;">tgrimes1</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">Shipped#2024-02-11</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">def456</td>
      <td style="border:1px solid #9e9e9e; padding:0.12rem 0.6rem;">Home</td>
    </tr>
  </tbody>
</table>
</div>

<div style="display:grid; grid-template-columns:1.15fr 1fr; gap:1rem; align-items:center; margin-top:0.4rem;">
<div style="font-size:0.66rem;">

```python
table.query(
    IndexName='status-date-index',
    KeyConditionExpression=
        Key('username').eq('aarchamb') &
        Key('status_date').between(
            'Placed#2024-01-01', 'Placed#2024-01-31')
)  # → order abc123 only (ghi789 is Feb)
```

</div>
<div style="font-size:0.72rem; color:#475569;">
Equality on <code>username</code>, then a <strong>range</strong> on the composed <code>status_date</code>. Pinning the <code>Placed#</code> prefix and ranging the date works only because <strong>status comes first</strong> in the key.
</div>
</div>

---

# The Importance of Sort Order

Within a partition, DynamoDB **physically stores items sorted by the sort key** — and `status_date` puts **status first, then date**. A range query is just a **contiguous slice** of that order, so it stays fast.

<div style="display:grid; grid-template-columns:1.15fr 1fr; gap:1.2rem; align-items:start; margin-top:0.6rem;">
<div style="font-size:0.76rem;">

Stored order for `username = aarchamb`, sorted by `status_date`:

<div style="font-family:monospace; font-size:0.82rem; line-height:1.7; margin-top:0.4rem;">
<div style="background:#dcfce7; border-left:4px solid #16a34a; padding:0.15rem 0.6rem;">Placed#2024-01-15&nbsp;&nbsp;← in range</div>
<div style="background:#f1f5f9; border-left:4px solid #cbd5e1; padding:0.15rem 0.6rem;">Placed#2024-02-28</div>
<div style="background:#f1f5f9; border-left:4px solid #cbd5e1; padding:0.15rem 0.6rem;">Shipped#2024-01-03</div>
</div>

</div>
<div style="font-size:0.78rem; color:#475569;">

`status_date BETWEEN 'Placed#2024-01-01' AND 'Placed#2024-01-31'` walks straight to the `Placed#` block, then reads dates in order until it passes Jan 31 — **no full scan, no filter**.

Compose it **date first** (`2024-01-15#Placed`) and "all Placed orders this month" would scatter across statuses — no longer one contiguous slice. **Attribute order is a design decision.**

</div>
</div>

---

# Summary

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:0.8rem; font-size:0.85rem;">
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Single-table design</div>
    Store multiple entity types in one table<br>
    Use natural attributes to identify entity type<br>
    Design keys around access patterns first
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Multi-attribute keys</div>
    Compose GSI keys from up to 4 attributes<br>
    No synthetic string concatenation<br>
    Natural types; no backfill needed
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Sparse indexing</div>
    Missing attribute → item excluded from GSI<br>
    Natural entity filter across one table
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Query left-to-right</div>
    SK attributes queried in order<br>
    Cannot skip attributes in a multi-attribute SK
  </div>
</div>

---

# Further Reading

- [notes/10 — DynamoDB Data Modeling](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/10%20-%20DynamoDB%20Data%20Modeling.md)
- [Course notes overview](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/README.md)
