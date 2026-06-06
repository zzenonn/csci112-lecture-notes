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

# Multi-Attribute Keys: The New Way

A GSI's sort key can be composed of **up to 4 natural attributes**:

```python
global_secondary_indexes = [{
    'IndexName': 'user-index',
    'KeySchema': [
        {'AttributeName': 'username',    'KeyType': 'HASH'},  # PK
        {'AttributeName': 'entity_type', 'KeyType': 'RANGE'}, # SK part 1
        {'AttributeName': 'entity_id',   'KeyType': 'RANGE'}, # SK part 2
    ],
    'Projection': {'ProjectionType': 'ALL'}
}]
```

Query rules for multi-attribute sort keys:
- All **PK** attributes → equality (`=`) required
- Sort key attributes → queried **left to right**; last one can use range operators

```python
# Valid: PK + first SK attribute (entity_type)
Key('username').eq('aarchamb') & Key('entity_type').eq('ORDER')

# INVALID: skipping entity_type to query entity_id directly
Key('username').eq('aarchamb') & Key('entity_id').eq('abc123')
```

---

# Sparse Indexing

If an item **lacks any attribute** that is part of a GSI key, that item **does not appear in the GSI** — automatically.

<div style="overflow-x:auto; margin-top:0.9rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.84rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Entity</th>
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Has <code>username</code>?</th>
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Has <code>order_id</code>?</th>
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">user-index GSI</th>
      <th style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">order-index GSI</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">User profile</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">✓</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#dc2626;">✗</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">Included</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#dc2626;">Excluded</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Order</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">✓</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">✓</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">Included</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">Included</td>
    </tr>
    <tr>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Line item</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#dc2626;">✗</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">✓</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#dc2626;">Excluded</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1; color:#16a34a;">Included</td>
    </tr>
  </tbody>
</table>
</div>

<div style="margin-top:0.7rem; font-size:0.85rem; color:#475569;">Sparse indexing acts as a natural entity filter — no application logic needed.</div>

---
layout: section
---

# Walkthrough: Users, Orders, and Line Items

---

# Table Design

Three entity types, one table. Base table: `entity_type` (PK) + `entity_id` (SK).

```
Base table
┌──────────────┬──────────────┬──────────┬──────────┬─────────────────┐
│ entity_type  │ entity_id    │ username │ order_id │ Other attrs     │
├──────────────┼──────────────┼──────────┼──────────┼─────────────────┤
│ USER         │ aarchamb     │ aarchamb │  —       │ email, address  │
│ ORDER        │ abc123       │ aarchamb │ abc123   │ status, address │
│ ITEM         │ itm001       │  —       │ abc123   │ product, qty    │
│ ITEM         │ itm002       │  —       │ abc123   │ product, qty    │
└──────────────┴──────────────┴──────────┴──────────┴─────────────────┘
```

Two GSIs with multi-attribute sort keys:
- **user-index**: `username` (PK) + `entity_type` + `entity_id` (SK)
- **order-index**: `order_id` (PK) + `entity_type` + `entity_id` (SK)

---

# Access Pattern Queries

**Get everything for a user** (profile + orders — items excluded by sparse indexing):

```python
table.query(
    IndexName='user-index',
    KeyConditionExpression=Key('username').eq('aarchamb')
)
```

**Get only orders for a user** (filter by entity_type in multi-attribute SK):

```python
table.query(
    IndexName='user-index',
    KeyConditionExpression=Key('username').eq('aarchamb') &
                           Key('entity_type').eq('ORDER')
)
```

**Get all line items for an order:**

```python
table.query(
    IndexName='order-index',
    KeyConditionExpression=Key('order_id').eq('abc123') &
                           Key('entity_type').eq('ITEM')
)
```

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
