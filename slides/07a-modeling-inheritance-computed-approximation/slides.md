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

# The Golden Rule

<div style="display:flex; justify-content:center; margin:2rem 0;">
  <div style="background:#f0fdf4; border:2px solid #16a34a; border-radius:8px; padding:1.5rem 2rem; font-size:1.1rem; font-weight:700; color:#166534; text-align:center; max-width:600px;">
    Data that is accessed together<br>should be stored together.
  </div>
</div>

Relational design normalizes data to eliminate redundancy and uses JOINs to reassemble it at query time.

MongoDB stores **pre-assembled** documents — the shape of your data should mirror how your application reads it.

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
      A single <code>db.books.find()</code> returns everything.
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

Three book types with shared and unique fields — all in one `books` collection:

```javascript
// product_type discriminator distinguishes each shape
{ _id: 1, product_type: "book",      title: "...", pages: 514, catalogues: { isbn10: "..." } }
{ _id: 2, product_type: "ebook",     title: "...", eformats: { epub: {...}, pdf: {...} } }
{ _id: 3, product_type: "audiobook", title: "...", narrator: "...", length_minutes: 1200 }
```

<div style="margin-top:1rem; font-size:0.88rem; color:#555;">
Shared fields: <code>title</code>, <code>authors</code>, <code>publisher</code>, <code>language</code>, <code>description</code><br>
Unique fields differentiate the three subtypes.
</div>

---

# Step 1 — Insert Sample Documents

```javascript
db.books.insertMany([
  {
    _id: 1, product_id: 34538756,
    title: "MongoDB: The Definitive Guide",
    details: "MongoDB explained by MongoDB champions",
    authors: ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
    publisher: "O'Reilly", language: "English",
    pages: 514, catalogues: { isbn10: "1491954469", isbn13: "978-1491954461" }
  },
  {
    _id: 2,
    title: "MongoDB: The Definitive Guide",
    desc: "MongoDB explained by MongoDB champions",
    authors: "Shannon Bradshaw, Eoin Brazil, and Christina Chodorow",
    publisher: "O'Reilly", language: "English",
    eformats: { epub: { pages: 774 }, pdf: { pages: 502 } },
    isbn10: "1491954469"
  },
  {
    _id: 3, product_id: 54538756,
    title: "MongoDB: The Definitive Guide",
    desc: "The complete book of MongoDB by its employees",
    author: "Eoin Brazil", narrator: "Eoin Brazil",
    publisher: "O'Reilly", language: "English",
    length_minutes: 1200
  }
])
```

Notice: inconsistent field names (`desc` vs `details`), `author` vs `authors` — the pattern migration will normalize these.

---

# Option A — Manual Updates (`replaceOne`)

Replace each document with a fully normalized shape including `product_type`:

```javascript
db.books.replaceOne({ _id: 1 }, {
  _id: 1, product_id: 34538756, product_type: "book",
  title: "MongoDB: The Definitive Guide",
  description: "MongoDB explained by MongoDB champions",
  authors: ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
  publisher: "O'Reilly", language: "English",
  pages: 514, catalogues: { isbn10: "1491954469", isbn13: "978-1491954461" }
})
```

```javascript
db.books.replaceOne({ _id: 2 }, {
  _id: 2, product_id: 44538756, product_type: "ebook",
  title: "MongoDB: The Definitive Guide",
  description: "MongoDB explained by MongoDB champions",
  authors: ["Shannon Bradshaw", "Eoin Brazil", "Christina Chodorow"],
  publisher: "O'Reilly", language: "English",
  eformats: { epub: { pages: 774 }, pdf: { pages: 502 } }, isbn10: "1491954469"
})
```

Works for small collections. For bulk migrations, use an aggregation pipeline.

---

# Option B — Aggregation Pipeline (`$project` + `$merge`)

Normalize all documents in one pass:

```javascript
var applyInheritancePattern = [
  { $project: {
      _id: "$_id",
      product_id: { $ifNull: ["$product_id", NumberInt(0)] },
      product_type: { $ifNull: ["$product_type", "Unspecified"] },
      title: "$title",
      description: { $ifNull: ["$desc", "$description", "$details", "Unspecified"] },
      authors: { $cond: {
        if: { $isArray: "$authors" }, then: "$authors",
        else: [{ $ifNull: ["$author", "Unspecified"] }]
      }},
      publisher: "$publisher", language: "$language",
      pages: "$pages", catalogues: "$catalogues",
      eformats: "$eformats", isbn10: "$isbn10",
      narrator: "$narrator", length_minutes: "$length_minutes"
  }},
  { $merge: { into: "books", on: "_id", whenMatched: "replace", whenNotMatched: "discard" }}
]

db.books.aggregate(applyInheritancePattern)
```

`$ifNull` resolves inconsistent field names; `$merge` writes results back to the same collection.

---

# Cleanup — Classify by Type

After the initial pass, documents with `product_type: "Unspecified"` still need classification.

**Audiobooks** — have `length_minutes`:

```javascript
var cleanupAudiobooks = [
  { $match: { $and: [{ product_type: "Unspecified" }, { length_minutes: { $gte: 0 } }] }},
  { $set: { product_type: "audiobook" }},
  { $merge: { into: "books", on: "_id", whenMatched: "replace", whenNotMatched: "discard" }}
]
db.books.aggregate(cleanupAudiobooks)
```

**Printed books** — have `pages` and `catalogues`:

```javascript
var cleanupBooks = [
  { $match: { $and: [{ product_type: "Unspecified" }, { pages: { $gte: 0 } }, { catalogues: { $exists: true } }] }},
  { $set: { product_type: "book" }},
  { $merge: { into: "books", on: "_id", whenMatched: "replace", whenNotMatched: "discard" }}
]
db.books.aggregate(cleanupBooks)
```

**Ebooks** — have `eformats`:

```javascript
var cleanupEbooks = [
  { $match: { $and: [{ product_type: "Unspecified" }, { eformats: { $exists: true } }] }},
  { $set: { product_type: "ebook" }},
  { $merge: { into: "books", on: "_id", whenMatched: "replace", whenNotMatched: "discard" }}
]
db.books.aggregate(cleanupEbooks)
```

---

# Verify Results

```javascript
db.books.find()
```

Expected result — all three documents normalized:

```json
[
  { "_id": 1, "product_type": "book",      "title": "...", "pages": 514, "catalogues": { ... } },
  { "_id": 2, "product_type": "ebook",     "title": "...", "eformats": { ... } },
  { "_id": 3, "product_type": "audiobook", "title": "...", "narrator": "...", "length_minutes": 1200 }
]
```

Query a specific type:

```javascript
db.books.find({ product_type: "audiobook" })
```

---

# Inheritance Pattern — Benefits

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin-top:1rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#166534;">Single query</div>
    <div style="font-size:0.88rem; color:#444;">One <code>db.books.find()</code> retrieves all product types — no UNIONs, no application-level merging.</div>
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
- Use the **Aggregation Framework** (`$project` + `$merge`) for bulk migrations — `replaceOne` in a loop is slow at scale
- If a subtype's fields diverge so much that common queries rarely apply, consider **separate collections** instead

<div style="margin-top:1.2rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.7rem 1rem; font-size:0.88rem;">
  See <code>notes/7a - MongoDB Data Modeling Inheritance Pattern.md</code> for the full mongosh walkthrough.
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

Store pre-computed `review_count`, `total_stars`, and `average_rating` directly in the book document:

```json
{
  "title": "MongoDB: The Definitive Guide",
  "rating": {
    "review_count": 3,
    "total_stars": 12.9,
    "average_rating": 4.3
  }
}
```

**Update logic on each new review:**

1. `total_stars += new_star_rating`
2. `review_count += 1`
3. `average_rating = total_stars / review_count`

The book document always has the current average — no aggregation needed at read time.

---

# Roll-Up Operations

Roll-ups are a variant: **aggregate periodically** into a summary collection rather than updating on every write.

**Use case:** internal dashboard showing daily totals by product type.

```javascript
var rollUpProductTypeSummary = [
  { $group: {
      _id: "$product_type",
      count: { $sum: 1 },
      averageNumberOfAuthors: { $avg: { $size: "$authors" } }
  }}
]

db.books.aggregate(rollUpProductTypeSummary)
```

Sample output stored to a `book_summary` collection:

```json
[
  { "_id": "audiobook", "count": 1, "averageNumberOfAuthors": 1 },
  { "_id": "ebook",     "count": 1, "averageNumberOfAuthors": 3 },
  { "_id": "book",      "count": 1, "averageNumberOfAuthors": 3 }
]
```

Run this pipeline on a **schedule** (e.g., nightly cron) — readers always get a fast lookup, never a live aggregation.

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
  See <code>notes/7b - MongoDB Data Modeling Computed and Approximation Pattern.md</code> for the full example.
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

# How It Works — 1-in-10 Probabilistic Updates

The logic lives in **application code**, not the database schema:

```python
import random

def add_review(book_id, star_rating):
    if random.randint(1, 10) == 10:          # ~10% chance
        multiplied_rating = star_rating * 10  # extrapolate as if 10 reviews
        db.books.update_one(
            { "_id": book_id },
            { "$inc": {
                "rating.review_count": 10,
                "rating.total_stars": multiplied_rating
            }}
        )
        # recalculate average_rating from total_stars / review_count
```

- ~90% of reviews → **no write to the book document**
- ~10% of reviews → write with ×10 multiplier to maintain statistical validity
- Result: ~90% fewer writes; accuracy loss is proportional to 1/N (negligible at high volume)

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

- `notes/7a - MongoDB Data Modeling Inheritance Pattern.md`
- `notes/7b - MongoDB Data Modeling Computed and Approximation Pattern.md`
