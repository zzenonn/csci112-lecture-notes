---
theme: discs
title: "MongoDB Data Modeling: Patterns II"
highlighter: shiki
layout: cover
codeCopy: true
---

MongoDB Data Modeling

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Explain the trade-off between **embedding** and **referencing** in MongoDB
2. Apply the **Extended Reference Pattern** to embed only the fields you need
3. Use the **Schema Versioning Pattern** to evolve a schema without downtime
4. Apply the **Single Collection Pattern** to co-locate related documents for efficient access

---
layout: section
---

# Part I: Reference / Extended Reference Pattern

---

# Embedding vs Referencing

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:1rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.5rem;">Embedding</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      All data in one document.<br>
      Fast single-document reads.<br>
      Risk: large documents, duplication.
    </div>
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.5rem;">Referencing</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      Data lives in separate collections.<br>
      Clean normalization, no duplication.<br>
      Cost: multiple queries to assemble a view.
    </div>
  </div>
</div>

<div style="margin-top:1.2rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.7rem 1rem; font-size:0.9rem;">
  The <strong>Extended Reference Pattern</strong> is the middle ground: reference the source of truth, but embed the fields you read most often.
</div>

---

# The Problem — Frequent Joins

An e-commerce app has three separate collections:

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.7rem; margin-top:1rem; font-size:0.82rem;">
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.7rem;">
    <div style="font-weight:700; margin-bottom:0.3rem;">customers</div>
    <code>_id, name, street, city,<br>country, date_of_birth, ...</code>
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.7rem;">
    <div style="font-weight:700; margin-bottom:0.3rem;">orders</div>
    <code>_id, date, customer_id,<br>order: [...]</code>
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.7rem;">
    <div style="font-weight:700; margin-bottom:0.3rem;">inventory</div>
    <code>_id, name, cost,<br>on_hand, ...</code>
  </div>
</div>

<div style="margin-top:1rem; font-size:0.88rem;">
  Displaying an order requires the customer's <strong>name + shipping address</strong> from <code>customers</code> and the <strong>product name + price</strong> from <code>inventory</code>. That's three round trips or a costly <code>$lookup</code> chain on every page load.
</div>

---

# Extended Reference Pattern

Embed **only the fields accessed together** with the order:

```json
{
  "_id": "order_001",
  "date": "2024-02-18",
  "customer_id": 123,
  "shipping_address": {
    "name":    "Katrina Pope",
    "street":  "123 Main St",
    "city":    "Somewhere",
    "country": "Someplace"
  },
  "order": [
    { "product": "widget", "qty": 5,
      "cost": { "value": 11.99, "currency": "USD" } }
  ]
}
```

`customer_id` still references `customers` for the full record. The shipping address is duplicated — but it is the only subset accessed every time an order is read.

---

# PyMongo Setup

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client     = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db         = client["ecommerce"]
customers  = db["customers"]
orders     = db["orders"]
```

---

# Extended Reference — In Practice

When creating an order, fetch only the fields needed and embed them:

```python
c = customers.find_one(
    {"_id": 123}, {"name": 1, "street": 1, "city": 1, "country": 1}
)

orders.insert_one({
    "_id": "order_001", "customer_id": 123,
    "shipping_address": {
        "name": c["name"], "street": c["street"],
        "city": c["city"], "country": c["country"],
    },
})
```

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.8rem 1rem; font-size:0.87rem;">
  Reading an order is now one query — <code>shipping_address</code> is already embedded.
</div>

---

# Extended Reference — When to Use

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#166534;">Good fit</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      You always read a <em>subset</em> of the related document together with the parent.<br>
      The referenced entity changes infrequently (shipping address rarely changes after order is placed).
    </div>
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:1rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#991b1b;">Poor fit</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      The embedded fields change frequently — you'd need to update every order when a customer moves.<br>
      You need the full referenced document on every read — just embed it.
    </div>
  </div>
</div>

<div style="margin-top:1rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.83rem;">
  Full example: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7c%20-%20MongoDB%20Data%20Modeling%20Reference%20Pattern.md">notes/7c — Reference Pattern</a>
</div>

---
layout: section
---

# Part II: Schema Versioning Pattern

---

# The Schema Evolution Problem

Updating a schema in a relational database means:

1. Stop the application
2. Run a migration (`ALTER TABLE ...`)
3. Restart the application

Any failure means rollback complexity and downtime.

<div style="margin-top:1.2rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.7rem 1rem; font-size:0.9rem;">
  MongoDB documents are polymorphic — different shapes can coexist in the same collection. The <strong>Schema Versioning Pattern</strong> turns that into a deliberate strategy.
</div>

---

# How Schema Versioning Works

Add a `schema_version` field to new documents. Application code branches on it:

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem; font-size:0.82rem;">
  <div style="border:1.5px solid #94a3b8; border-radius:6px; padding:0.8rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Version 1 (legacy — no field)</div>
    <pre style="margin:0; font-size:0.8rem; background:none;">{ "name": "Anakin Skywalker",
  "home": "503-555-0000",
  "work": "503-555-0010" }</pre>
  </div>
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.8rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Version 2 (new shape)</div>
    <pre style="margin:0; font-size:0.8rem; background:none;">{ "schema_version": "2",
  "name": "Anakin Skywalker",
  "contact_method": [
    { "work":    "503-555-0210" },
    { "mobile":  "503-555-0220" },
    { "twitter": "@anakinskywalker" }
  ] }</pre>
  </div>
</div>

<div style="margin-top:1rem; font-size:0.87rem;">
  Documents without <code>schema_version</code> are implicitly version 1. Both shapes live in the same collection.
</div>

---

# Application Handles Both Versions

```python
def get_contacts(customer):
    version = customer.get("schema_version", "1")
    if version == "1":
        return {
            "home":  customer.get("home"),
            "work":  customer.get("work"),
        }
    elif version == "2":
        return {m: v for contact in customer["contact_method"]
                     for m, v in contact.items()}
```

<div style="margin-top:0.8rem; display:grid; grid-template-columns:1fr 1fr; gap:0.8rem; font-size:0.85rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">No downtime</div>
    Old documents keep working. New documents use the new shape. Deploy the new app first, migrate data later — or never.
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">Incremental migration</div>
    Update documents lazily (on next write), in bulk on a schedule, or leave old documents as-is permanently.
  </div>
</div>

---

# Schema Versioning — When to Use

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#166534;">Good fit</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      Long-lived applications where schema evolves over time.<br>
      Large collections where full up-front migration is too slow or risky.<br>
      Zero-downtime deployments.
    </div>
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:1rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#991b1b;">Watch out for</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      App code must handle every supported version — increases complexity.<br>
      Indexes may need to span multiple field shapes.<br>
      Don't let the number of live versions grow unbounded.
    </div>
  </div>
</div>

<div style="margin-top:1rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.83rem;">
  Full example: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7d%20-%20MongoDB%20Data%20Modeling%20Schema%20Versioning.md">notes/7d — Schema Versioning</a>
</div>

---
layout: section
---

# Part III: Single Collection Pattern

---

# Why a Single Collection?

Our bookstore now has three related entities queried together:

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.7rem; margin-top:1rem; font-size:0.82rem;">
  <div style="border:1.5px solid #f97316; border-radius:6px; padding:0.7rem; background:#fff7ed; text-align:center;">
    <div style="font-size:1.3rem;">📖</div>
    <div style="font-weight:700;">books</div>
  </div>
  <div style="border:1.5px solid #f97316; border-radius:6px; padding:0.7rem; background:#fff7ed; text-align:center;">
    <div style="font-size:1.3rem;">👤</div>
    <div style="font-weight:700;">users</div>
  </div>
  <div style="border:1.5px solid #f97316; border-radius:6px; padding:0.7rem; background:#fff7ed; text-align:center;">
    <div style="font-size:1.3rem;">⭐</div>
    <div style="font-weight:700;">reviews</div>
  </div>
</div>

<div style="margin-top:1rem; font-size:0.88rem; line-height:1.7;">
  Embedding is not feasible — a book can have thousands of reviews. Separate collections require expensive <code>$lookup</code> chains. The Single Collection Pattern co-locates all three document types in one collection, with a <code>docType</code> field to tell them apart.
</div>

---

# Single Collection Document Shape

Each document carries `docType` and `relatedTo` to express its type and relationships:

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.6rem; margin-top:1rem; font-size:0.78rem;">
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.7rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.3rem;">book</div>
    <code>docType: "book"</code><br>
    <code>relatedTo: [product_id]</code><br>
    <code>book_id, title, authors, ...</code>
  </div>
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.7rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.3rem;">user</div>
    <code>docType: "user"</code><br>
    <code>relatedTo: [purchased book_ids]</code><br>
    <code>name, email, ...</code>
  </div>
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.7rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.3rem;">review</div>
    <code>docType: "review"</code><br>
    <code>relatedTo: [book_id]</code><br>
    <code>text, rating, user_id, ...</code>
  </div>
</div>

<div style="margin-top:1rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.83rem;">
  One query on <code>relatedTo: &lt;product_id&gt;</code> returns the book <em>and</em> all its reviews.
</div>

---

# Migration via Aggregation Pipeline

Use `$set` + `$merge` to add `docType` / `relatedTo` and write into `books_catalog`:

```python
# Books — full pipeline; reviews and users follow the same pattern
db["books"].aggregate([
    { "$set":   { "docType": "book",
                  "relatedTo": ["$product_id"], "book_id": "$product_id" } },
    { "$unset": ["product_id"] },
    { "$merge": { "into": "books_catalog", "on": "_id" } }
])

# Reviews: docType="review", relatedTo=["$book_id"], unset book_id
# Users:   docType="user",   relatedTo="$purchased_books", unset purchased_books
```

<div style="margin-top:0.6rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.83rem;">
  Full pipelines for all three collections: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7e%20-%20MongoDB%20Data%20Modeling%20Single%20Collection.md">notes/7e — Single Collection</a>
</div>

---

# Querying the Single Collection

```python
# All documents
list(books_catalog.find())

# Filter by type
list(books_catalog.find({ "docType": "review" }))

# Everything related to a specific book — book doc + all its reviews
list(books_catalog.find({ "relatedTo": 34538756 }))
```

<div style="margin-top:1rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.6rem 0.9rem; font-size:0.85rem;">
  One collection, one query, no <code>$lookup</code>. Access patterns drive the design — not entity relationships.
</div>

---

# Single Collection — When to Use

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#166534;">Good fit</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      Related entities are nearly always queried together.<br>
      Many-to-many or one-to-many relationships where embedding would create unbounded arrays.<br>
      You want to avoid <code>$lookup</code> entirely.
    </div>
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:1rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#991b1b;">Poor fit</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      Entity types are rarely queried together — separate collections are simpler.<br>
      Documents have nothing in common — a mixed collection makes queries harder to reason about.
    </div>
  </div>
</div>

<div style="margin-top:1rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.83rem;">
  Full example: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7e%20-%20MongoDB%20Data%20Modeling%20Single%20Collection.md">notes/7e — Single Collection</a>
</div>

---

# Pattern Selection Guide

<div style="overflow-x:auto; margin-top:1rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.88rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Pattern</th>
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Problem it solves</th>
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Key trade-off</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Extended Reference</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Expensive repeated joins for a subset of fields</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Embedded fields may drift from source of truth</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Schema Versioning</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Schema evolution without downtime or full migration</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">App must handle all live schema versions</td>
    </tr>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Single Collection</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Related entities queried together, embedding infeasible</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Collection shape is heterogeneous; requires <code>docType</code> discipline</td>
    </tr>
  </tbody>
</table>
</div>

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.6rem 0.9rem; font-size:0.87rem;">
  These patterns are composable — a single collection can also use schema versioning when its document shapes evolve over time.
</div>

---

# Further Reading

See the matching notes for full code listings, sample documents, and command output:

- [notes/7c — MongoDB Data Modeling: Reference Pattern](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7c%20-%20MongoDB%20Data%20Modeling%20Reference%20Pattern.md)
- [notes/7d — MongoDB Data Modeling: Schema Versioning Pattern](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7d%20-%20MongoDB%20Data%20Modeling%20Schema%20Versioning.md)
- [notes/7e — MongoDB Data Modeling: Single Collection Pattern](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7e%20-%20MongoDB%20Data%20Modeling%20Single%20Collection.md)
