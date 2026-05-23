---
theme: discs
title: "MongoDB Data Modeling: Patterns I"
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

1. Explain why **data modeling** matters differently in NoSQL vs relational databases
2. Apply the **Inheritance Pattern** to store polymorphic documents in one collection
3. Use the **Computed Pattern** to pre-compute values and reduce read-time overhead
4. Use the **Approximation Pattern** to reduce write load for high-volume counters
5. Choose the right pattern based on read/write trade-offs

---
layout: section
---

# Why Data Modeling Matters in NoSQL

---

# NoSQL vs Relational Design

<div style="overflow-x:auto; margin-top:1rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.88rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Aspect</th>
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Relational (SQL)</th>
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Document (MongoDB)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Schema</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Fixed — all rows same columns</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Flexible — each document can differ</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Primary informer</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Entity relationships</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Access patterns</td>
    </tr>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Data assembly</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">JOINs at read time</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Embedded at write time</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Redundancy</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Minimized (normalized)</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Accepted for read performance</td>
    </tr>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Polymorphism</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Separate tables or nullable columns</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Different shapes in one collection</td>
    </tr>
  </tbody>
</table>
</div>

---

# The Golden Rule

<div style="display:flex; justify-content:center; margin:2rem 0;">
  <div style="background:#f0fdf4; border:2px solid #16a34a; border-radius:8px; padding:1.5rem 2rem; font-size:1.1rem; font-weight:700; color:#166534; text-align:center; max-width:600px;">
    Data that is accessed together<br>should be stored together.
  </div>
</div>

Relational design normalizes data to eliminate redundancy and uses JOINs to reassemble it at query time.

MongoDB stores **pre-assembled** documents — the shape of your data should mirror how your application reads it.

---

# MongoDB Modeling Principles

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:1rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Model for access patterns</div>
    <div style="font-size:0.88rem; color:#444;">Design documents around how your app reads data, not around how data is normalized.</div>
  </div>
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:1rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Embed what is read together</div>
    <div style="font-size:0.88rem; color:#444;">If two pieces of data are always fetched together, embed them in one document.</div>
  </div>
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:1rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Pre-compute on write</div>
    <div style="font-size:0.88rem; color:#444;">Store derived values (averages, counts) so reads are O(1) instead of re-aggregating on demand.</div>
  </div>
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:1rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Accept controlled redundancy</div>
    <div style="font-size:0.88rem; color:#444;">Duplication that speeds up reads is often worth the extra storage and write complexity.</div>
  </div>
</div>

---
layout: section
---

# Part I: Inheritance Pattern

---

# What is the Inheritance Pattern?

Store multiple **related but distinct** document types in a **single collection**, using a discriminator field to tell them apart.

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.8rem; margin-top:1.5rem;">
  <div style="border:2px solid #f97316; border-radius:6px; padding:0.9rem; text-align:center;">
    <div style="font-size:1.5rem;">📖</div>
    <div style="font-weight:700; margin:0.4rem 0; font-size:0.9rem;">Printed Book</div>
    <div style="font-size:0.8rem; color:#555; line-height:1.6;">title, author, publisher<br><em>+ pages, stock_level</em></div>
  </div>
  <div style="border:2px solid #f97316; border-radius:6px; padding:0.9rem; text-align:center;">
    <div style="font-size:1.5rem;">💻</div>
    <div style="font-weight:700; margin:0.4rem 0; font-size:0.9rem;">Ebook</div>
    <div style="font-size:0.8rem; color:#555; line-height:1.6;">title, author, publisher<br><em>+ eformats, download_url</em></div>
  </div>
  <div style="border:2px solid #f97316; border-radius:6px; padding:0.9rem; text-align:center;">
    <div style="font-size:1.5rem;">🎧</div>
    <div style="font-weight:700; margin:0.4rem 0; font-size:0.9rem;">Audiobook</div>
    <div style="font-size:0.8rem; color:#555; line-height:1.6;">title, author, publisher<br><em>+ narrator, length_minutes</em></div>
  </div>
</div>

<div style="margin-top:1.2rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.7rem 1rem; font-size:0.9rem;">
  All three types share common fields → store together, distinguish with <code>product_type</code>.
</div>

---

# When to Use the Inheritance Pattern

Use it when documents:

- Have **more similarities than differences**
- Are **queried together** (e.g., "show all books")
- Belong to a natural **type hierarchy**

<div style="margin-top:1.5rem; display:grid; grid-template-columns:1fr 1fr; gap:1rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#166534;">Use it</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      All book types display on the same product listing page.<br>
      A single <code>books.find()</code> returns everything.
    </div>
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:1rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#991b1b;">Avoid it</div>
    <div style="font-size:0.87rem; line-height:1.7; color:#444;">
      Types are never queried together and have completely different fields — separate collections are cleaner.
    </div>
  </div>
</div>

---

# Example: Bookstore Data Structure

<div style="display:grid; grid-template-columns:1fr 1.2fr; gap:1.2rem; margin-top:0.8rem;">
  <div>
    <div style="font-weight:700; color:#0c4a6e; margin-bottom:0.4rem;">Shared fields (all types)</div>
    <div style="background:#f0f9ff; border:1.5px solid #00b0f0; border-radius:6px; padding:0.7rem 0.9rem; font-size:0.85rem; line-height:1.8;">
      <code>_id</code>, <code>product_id</code>, <code>title</code><br>
      <code>authors</code>, <code>publisher</code>, <code>language</code>, <code>description</code>
    </div>
    <div style="margin-top:0.8rem; font-weight:700; color:#0c4a6e; margin-bottom:0.4rem;">Discriminator</div>
    <div style="background:#f0fdf4; border:1.5px solid #16a34a; border-radius:6px; padding:0.7rem 0.9rem; font-size:0.85rem;">
      <code>product_type</code>: <code>"book"</code> | <code>"ebook"</code> | <code>"audiobook"</code>
    </div>
  </div>
  <div>
    <div style="font-weight:700; color:#9a3412; margin-bottom:0.4rem;">Type-specific fields</div>
    <div style="display:grid; grid-template-columns:1fr; gap:0.5rem;">
      <div style="background:#fff7ed; border:1.5px solid #f97316; border-radius:6px; padding:0.55rem 0.8rem; font-size:0.82rem;">
        <strong>book</strong> → <code>pages</code>, <code>stock_level</code>, <code>catalogues</code>
      </div>
      <div style="background:#fff7ed; border:1.5px solid #f97316; border-radius:6px; padding:0.55rem 0.8rem; font-size:0.82rem;">
        <strong>ebook</strong> → <code>eformats</code>, <code>download_url</code>
      </div>
      <div style="background:#fff7ed; border:1.5px solid #f97316; border-radius:6px; padding:0.55rem 0.8rem; font-size:0.82rem;">
        <strong>audiobook</strong> → <code>narrator</code>, <code>length_minutes</code>, <code>time_by_chapter</code>
      </div>
    </div>
  </div>
</div>

<div style="margin-top:1rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.6rem 1rem; font-size:0.85rem;">
  Full document examples: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md">notes/7a — Inheritance Pattern</a>
</div>

---

# PyMongo Setup

All examples in this deck use PyMongo from your laptop, connecting to `mongod` on a VM:

```python
from pymongo import MongoClient, ReturnDocument

client = MongoClient("mongodb://<vm_ip_address>:27017/")
db     = client["bookstore"]
books  = db["books"]
```

The `books` variable is the **collection handle** — every example below calls methods on it
(`books.insert_many`, `books.aggregate`, `books.update_one`, …).

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.6rem 1rem; font-size:0.85rem;">
  Need to set up <code>.venv</code> and install <code>pymongo</code>? See <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/04%20-%20MongoDB%20Data%20Structures.md">notes/04 — MongoDB Data Structures</a>.
</div>

---

# Migrating Existing Data

In practice, you rarely start with clean data. You inherit a collection where documents drifted apart over time:

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.6rem; margin-top:1rem; font-size:0.78rem;">
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.6rem;">
    <div style="font-weight:700; margin-bottom:0.3rem;">Doc 1</div>
    <code>details: "..."</code><br>
    <code>authors: [...]</code><br>
    <code>pages: 514</code>
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.6rem;">
    <div style="font-weight:700; margin-bottom:0.3rem;">Doc 2</div>
    <code>desc: "..."</code><br>
    <code>authors: "a, b, c"</code><br>
    <code>eformats: {...}</code>
  </div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.6rem;">
    <div style="font-weight:700; margin-bottom:0.3rem;">Doc 3</div>
    <code>desc: "..."</code><br>
    <code>author: "x"</code><br>
    <code>length_minutes: 1200</code>
  </div>
</div>

<div style="margin-top:1rem; font-size:0.9rem;">
  <strong>Goal:</strong> reshape every document so it conforms to the Inheritance Pattern — consistent field names, all carrying a <code>product_type</code> discriminator.
</div>

<div style="margin-top:0.8rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.6rem 1rem; font-size:0.85rem;">
  Full sample documents and <code>insert_many</code> snippet: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md#step-1-insert-sample-documents">notes/7a — Step 1</a>
</div>

---

# Two Migration Strategies

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:1rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Option A — <code>replace_one</code></div>
    <div style="font-size:0.85rem; line-height:1.6; color:#444;">
      Manually rewrite each document with the corrected shape.<br><br>
      <strong>Best for:</strong> small collections, one-off fixes, exploratory work.<br><br>
      <strong>Drawback:</strong> doesn't scale — N documents = N statements.
    </div>
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Option B — Aggregation pipeline</div>
    <div style="font-size:0.85rem; line-height:1.6; color:#444;">
      One pipeline rewrites the whole collection, classifies each document by its unique fields, and writes results back via <code>$merge</code>.<br><br>
      <strong>Best for:</strong> bulk migrations on real data.
    </div>
  </div>
</div>

<div style="margin-top:1rem; font-size:0.85rem; color:#555;">
We'll walk through both — the choice depends on collection size and how much logic you need to express.
</div>

---

# Option A — Manual `replace_one`

Rewrite each document with normalized fields and a `product_type` discriminator:

```python
books.replace_one(
    { "_id": 1 },
    {
        "_id": 1, "product_id": 34538756,
        "product_type": "book",                  # discriminator added
        "description": "...",                    # was "details"
        "authors": ["Shannon Bradshaw", "..."],  # always an array
        # ... rest of normalized fields
    }
)
```

<div style="margin-top:0.8rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.82rem;">
  Repeat for each <code>_id</code>. Full version: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md#option-a-manual-updates">notes/7a — Option A</a>
</div>

---

# Option B — Aggregation Pipeline

Two passes — Pass 1 normalizes field names, Pass 2 classifies each doc by its unique fields.

**Pass 1 — the `$ifNull` trick** to resolve inconsistent field names:

```python
books.aggregate([
    { "$project": {
        "description": { "$ifNull": ["$desc", "$description", "$details", "Unspecified"] },
        "authors":     { "$cond": {
                           "if":   { "$isArray": "$authors" },
                           "then": "$authors",
                           "else": [{ "$ifNull": ["$author", "Unspecified"] }]
                        }},
        # ... pass through the rest
    }},
    { "$merge": { "into": "books", "on": "_id", "whenMatched": "replace" } }
])
```

`$merge` writes the projection back into `books` (`whenMatched: "replace"`).

---

# Pass 2 — Classify by Unique Fields

After Pass 1, every document has `product_type: "Unspecified"`. Pass 2 sets the correct value by matching on a field unique to each subtype:

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.6rem; margin-top:0.8rem; font-size:0.78rem;">
  <div style="border:1.5px solid #f97316; border-radius:6px; padding:0.55rem; background:#fff7ed;">
    <div style="font-weight:700; margin-bottom:0.2rem;">audiobook</div>
    <code>length_minutes &gt;= 0</code>
  </div>
  <div style="border:1.5px solid #f97316; border-radius:6px; padding:0.55rem; background:#fff7ed;">
    <div style="font-weight:700; margin-bottom:0.2rem;">book</div>
    <code>pages</code> &amp; <code>catalogues</code> exist
  </div>
  <div style="border:1.5px solid #f97316; border-radius:6px; padding:0.55rem; background:#fff7ed;">
    <div style="font-weight:700; margin-bottom:0.2rem;">ebook</div>
    <code>eformats</code> exists
  </div>
</div>

```python
# One pipeline per type — audiobook example:
books.aggregate([
    { "$match": { "product_type": "Unspecified", "length_minutes": { "$gte": 0 } } },
    { "$set":   { "product_type": "audiobook" } },
    { "$merge": { "into": "books", "on": "_id", "whenMatched": "replace" } }
])
```

<div style="margin-top:0.6rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.82rem;">
  Full pipelines for all three types: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md#option-b-aggregation-pipeline-automation">notes/7a — Option B</a>
</div>

---

# Verify Results

```python
for doc in books.find():
    print(doc)
```

Each document now carries a `product_type` and consistent fields — ready to query polymorphically:

```python
books.find({ "product_type": "audiobook" })   # type-specific query
books.find()                                   # all types together
```

<div style="margin-top:1rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.6rem 1rem; font-size:0.85rem;">
  Expected output for all three documents: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md#step-4-verify-the-updates">notes/7a — Verify</a>
</div>

---

# Inheritance Pattern — Benefits

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#166534;">Single query</div>
    <div style="font-size:0.88rem; color:#444;">One <code>books.find()</code> retrieves all product types — no UNIONs, no application-level merging.</div>
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#166534;">Flexible schema</div>
    <div style="font-size:0.88rem; color:#444;">Each subtype carries only the fields it needs — no NULL columns for fields that don't apply.</div>
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#166534;">Filter by type</div>
    <div style="font-size:0.88rem; color:#444;">Index on <code>product_type</code> for efficient type-specific queries.</div>
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#166534;">Extend easily</div>
    <div style="font-size:0.88rem; color:#444;">Add new product types without schema migrations — just add new values for <code>product_type</code>.</div>
  </div>
</div>

---

# Inheritance Pattern — Best Practices

- Always include a **discriminator field** (e.g., `product_type`) on every document — never leave it null or unset
- **Index** the discriminator field if type-specific queries are common
- Keep common fields **consistently named** across all subtypes
- Use the **Aggregation Framework** (`$project` + `$merge`) for bulk migrations — `replace_one` in a loop is slow at scale
- If a subtype's fields diverge so much that common queries rarely apply, consider **separate collections** instead

<div style="margin-top:1.2rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.7rem 1rem; font-size:0.88rem;">
  Full mongosh walkthrough: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md">notes/7a — Inheritance Pattern</a>
</div>

---
layout: section
---

# Part II: Computed Pattern

---

# What is the Computed Pattern?

**Pre-compute derived values at write time** so reads are instant.

Instead of re-aggregating on every read:

```
read request → scan all reviews → calculate average → return
```

Compute once on write and cache the result in the document:

```
new review → update average_rating + review_count in the book document → fast read
```

<div style="margin-top:1.2rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.7rem 1rem; font-size:0.9rem;">
  Trade: <strong>slightly more work per write</strong> for <strong>O(1) reads</strong>. Best when reads vastly outnumber writes.
</div>

---

# Use Case — Book Ratings

Pre-compute `review_count`, `total_stars`, `average_rating` in the book document:

```json
{ "rating": { "review_count": 3, "total_stars": 12.9, "average_rating": 4.3 } }
```

**On each new review** — atomic `$inc`, then recompute the average:

```python
def add_review(book_id, star_rating):
    updated = books.find_one_and_update(
        { "_id": book_id },
        { "$inc": { "rating.review_count": 1,
                    "rating.total_stars":  star_rating } },
        return_document=ReturnDocument.AFTER,
    )
    avg = updated["rating"]["total_stars"] / updated["rating"]["review_count"]
    books.update_one({ "_id": book_id }, { "$set": { "rating.average_rating": avg } })
```

---

# Computed in Action

```python
# 1. Insert a new book — initialize the rating subdoc to zeros
books.insert_one({
    "_id": 1, "title": "MongoDB: The Definitive Guide",
    "rating": { "review_count": 0, "total_stars": 0.0, "average_rating": 0.0 }
})

# 2. Each incoming review calls add_review — totals + average update on every write
add_review(1, 5.0)
add_review(1, 4.0)
add_review(1, 4.0)

# 3. Reads are O(1) — no aggregation, just a field lookup
print(books.find_one({ "_id": 1 }, { "rating.average_rating": 1 }))
# → { "_id": 1, "rating": { "average_rating": 4.333... } }
```

<div style="margin-top:0.6rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.5rem 0.9rem; font-size:0.85rem;">
  The average is computed at <em>write</em> time and cached. Every reader sees the latest value with one field read.
</div>

---

# Roll-Up Operations

A variant: **aggregate periodically** into a summary collection instead of updating on every write. Use case: internal dashboard with daily totals by product type.

```python
roll_up_pipeline = [
    { "$group": {
        "_id": "$product_type",
        "count": { "$sum": 1 },
        "average_number_of_authors": { "$avg": { "$size": "$authors" } }
    }},
    { "$merge": { "into": "book_summary", "on": "_id", "whenMatched": "replace" } }
]

books.aggregate(roll_up_pipeline)   # run nightly via cron / scheduler
```

Readers query `book_summary` directly — fast lookup, never a live aggregation.

---

# Computed Pattern — When to Use

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#166534;">Good fit</div>
    <div style="font-size:0.88rem; line-height:1.7; color:#444;">
      Reads heavily outnumber writes<br>
      Expensive aggregate runs repeatedly<br>
      Data freshness can be eventual (roll-ups)<br>
      Dashboards, product pages, leaderboards
    </div>
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:1rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#991b1b;">Poor fit</div>
    <div style="font-size:0.88rem; line-height:1.7; color:#444;">
      Writes heavily outnumber reads<br>
      Computed value changes so often the cache is almost never hit<br>
      The computation is trivial and cheap to redo
    </div>
  </div>
</div>

<div style="margin-top:1.2rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.7rem 1rem; font-size:0.88rem;">
  Full example: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7b%20-%20MongoDB%20Data%20Modeling%20Computed%20and%20Approximation%20Pattern.md">notes/7b — Computed &amp; Approximation</a>
</div>

---
layout: section
---

# Part III: Approximation Pattern

---

# What is the Approximation Pattern?

When **exact values are not critical** and the update cost is too high, update only occasionally using a **statistically valid sample**.

**Problem:** a book with 1 million reviews. Every new review triggers a write to update the average — contention on the book document becomes a bottleneck.

**Key insight:** adding one review to a million-review average changes it by less than 0.0001%. The precision loss from skipping most updates is negligible.

<div style="margin-top:1.2rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.7rem 1rem; font-size:0.9rem;">
  <strong>Rule of thumb:</strong> use the Approximation Pattern when the relative impact of one new data point is tiny and accuracy within a small margin is acceptable.
</div>

---

# 1-in-10 Probabilistic Updates

The logic lives in **application code**, not the database schema:

```python
import random

def add_review_approx(book_id, star_rating):
    if random.randint(1, 10) == 10:                # ~10% chance
        books.update_one(
            { "_id": book_id },
            { "$inc": { "rating.review_count": 10,        # extrapolate ×10
                        "rating.total_stars":  star_rating * 10 } },
        )
        # then recompute rating.average_rating as before
```

~90% of reviews skip the write; the 10% that hit extrapolate ×10 to keep totals statistically valid. Accuracy loss is proportional to 1/N — negligible at high volume.

---

# Approximation in Action

```python
# 1. Insert a hot book — initialize the rating subdoc
books.insert_one({
    "_id": 2, "title": "Hot Book",
    "rating": { "review_count": 0, "total_stars": 0.0, "average_rating": 0.0 }
})

# 2. Simulate 1000 incoming 5-star reviews
for _ in range(1000):
    add_review_approx(2, 5.0)

# 3. Inspect — review_count ≈ 1000, but the document was written only ~100 times
print(books.find_one({ "_id": 2 }, { "rating": 1 }))
# → { "_id": 2, "rating": { "review_count": ~1000,
#                            "total_stars":  ~5000,
#                            "average_rating": 5.0 } }
```

<div style="margin-top:0.6rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.5rem 0.9rem; font-size:0.85rem;">
  1000 reviews → ~100 writes. Totals still reflect the full population because each write counts as 10.
</div>

---

# Computed vs Approximation

<div style="overflow-x:auto; margin-top:1rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.9rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Aspect</th>
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Computed Pattern</th>
      <th style="padding:0.6rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Approximation Pattern</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Accuracy</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Exact</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Approximate (statistically valid)</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Write cost</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Every write updates the document</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">~10% of writes update the document</td>
    </tr>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Read cost</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">O(1) field lookup</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">O(1) field lookup</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Implementation</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">DB schema + application</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Application logic only</td>
    </tr>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Best for</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Moderate write volumes, exact counts</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Very high write volumes, display metrics</td>
    </tr>
  </tbody>
</table>
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
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Inheritance</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Mixed document types cluttering multiple collections</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Document shape varies; must handle in app code</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Computed</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Expensive re-aggregation on every read</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">More writes; stale risk if not kept in sync</td>
    </tr>
    <tr>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;"><strong>Approximation</strong></td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Write contention on hot documents</td>
      <td style="padding:0.55rem 0.8rem; border:1px solid #cbd5e1;">Small accuracy loss; logic in application</td>
    </tr>
  </tbody>
</table>
</div>

<div style="margin-top:1.2rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.7rem 1rem; font-size:0.88rem;">
  These patterns are not mutually exclusive — a document can use <strong>Inheritance + Computed</strong> simultaneously (e.g., polymorphic product with pre-computed average rating per type).
</div>

---

# Further Reading

See the matching notes for full code listings, sample documents, and command output:

- [notes/7a — MongoDB Data Modeling: Inheritance Pattern](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md)
- [notes/7b — MongoDB Data Modeling: Computed and Approximation Patterns](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/7b%20-%20MongoDB%20Data%20Modeling%20Computed%20and%20Approximation%20Pattern.md)
