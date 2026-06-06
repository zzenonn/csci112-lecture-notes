---
theme: discs
title: "MongoDB Data Modeling: Subset Pattern"
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

1. Explain when large embedded arrays hurt performance and why
2. Apply the **Subset Pattern** to split hot and cold data into separate collections
3. Implement the migration using an aggregation pipeline and `$slice`
4. Recognize which other patterns this trade-off also appears in

---
layout: section
---

# The Problem: Large Embedded Arrays

---

# Working Set and WiredTiger Cache

MongoDB's **WiredTiger** storage engine keeps frequently accessed data in an in-memory cache.

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:1rem; font-size:0.86rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">When it works well</div>
    Documents are small → many fit in cache → reads are fast.<br><br>
    The <em>working set</em> (data accessed today) fits in RAM.
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">When it breaks down</div>
    Documents are large → few fit in cache → MongoDB reads from disk.<br><br>
    Disk I/O is 100-1000× slower than RAM.
  </div>
</div>

<div style="margin-top:0.9rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 1rem; font-size:0.86rem;">
  A book with 500 reviews embedded in the document is enormous. Every read of that book — even to show the title — loads all 500 reviews into cache.
</div>

---

# The Mismatch: All Reviews in One Document

A popular book has 500 reviews embedded. The product page shows only the **3 most recent**.

```json
{
  "_id": 1,
  "title": "MongoDB: The Definitive Guide",
  "reviews": [
    { "review_id": 1, "text": "...", "upvotes": 193, "rating": 9 },
    { "review_id": 2, "text": "...", "upvotes": 87,  "rating": 8 },
    ...
    // 497 more reviews nobody reads on page load
  ]
}
```

<div style="margin-top:0.8rem; display:grid; grid-template-columns:1fr 1fr; gap:0.8rem; font-size:0.85rem;">
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.7rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.2rem; color:#991b1b;">Hot data</div>
    Top 3 reviews — read on every page load
  </div>
  <div style="border:1.5px solid #94a3b8; border-radius:6px; padding:0.7rem; background:#f8fafc;">
    <div style="font-weight:700; margin-bottom:0.2rem; color:#475569;">Cold data</div>
    Reviews 4–500 — rarely accessed, only on "see all reviews" click
  </div>
</div>

---
layout: section
---

# The Subset Pattern

---

# Subset Pattern: Split Hot from Cold

Keep a **small subset** of frequently accessed data in the main document. Move the full dataset to a separate collection.

<div style="margin-top:1rem; display:grid; grid-template-columns:1fr 1fr; gap:1rem; font-size:0.85rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.9rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#166534;"><code>books</code> collection</div>
    Full book document.<br>
    <code>reviews</code> array contains <strong>only the top 3</strong> reviews.<br><br>
    Small, fast to load, fits in cache.
  </div>
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.9rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#0369a1;"><code>reviews</code> collection</div>
    All reviews for all books.<br>
    Each document has a <code>book_id</code> back-reference.<br><br>
    Queried separately only when needed.
  </div>
</div>

<div style="margin-top:0.9rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 1rem; font-size:0.86rem;">
  This introduces duplication — the top 3 reviews live in both collections. That is intentional. Review text changes rarely, so the cost of keeping two copies is low.
</div>

---

# Migration Step 1: Copy All Reviews Out

Use an aggregation pipeline to extract all embedded reviews into the `reviews` collection:

```javascript
const copyReviews = [
  { $unwind: { path: "$reviews" } },
  { $set: { "reviews.book_id": "$_id" } },
  { $replaceRoot: { newRoot: "$reviews" } },
  { $out: "reviews" }
]
db.books.aggregate(copyReviews)
```

Each review document in the new collection:

```json
{
  "review_id": 23,
  "text": "...",
  "upvotes": 12,
  "rating": 7,
  "book_id": 2         // ← back-reference to the book
}
```

---

# Migration Step 2: Trim Each Book to 3 Reviews

After the full copy is safe in `reviews`, shrink each book document:

```javascript
db.books.updateMany(
  {},
  [{ $set: { reviews: { $slice: ["$reviews", 3] } } }]
)
```

Verify:

```javascript
db.books.find({}, { title: 1, "reviews": 1 })
// Each book now has at most 3 review subdocuments
```

<div style="margin-top:0.8rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.5rem 1rem; font-size:0.86rem;">
  See <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/8f%20-%20MongoDB%20Data%20Modeling%20Subset%20Pattern.md" style="color:#16a34a;">notes/8f — Subset Pattern</a> for the full script including sample data and verification steps.
</div>

---

# When to Use the Subset Pattern

<div style="overflow-x:auto; margin-top:1rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.85rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.5rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Condition</th>
      <th style="padding:0.5rem 0.8rem; text-align:left; border:1px solid #cbd5e1;">Use Subset?</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Large array, but only the first N elements shown on page load</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1; color:#16a34a; font-weight:600;">Yes</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Array grows unboundedly over time (comments, events, log entries)</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1; color:#16a34a; font-weight:600;">Yes</td>
    </tr>
    <tr>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Array is small and always read in full</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1; color:#dc2626; font-weight:600;">No — embed it</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1;">Array items update frequently (price list, live inventory)</td>
      <td style="padding:0.45rem 0.8rem; border:1px solid #cbd5e1; color:#ca8a04; font-weight:600;">Consider — duplication cost rises</td>
    </tr>
  </tbody>
</table>
</div>

---

# Pattern Summary: All Patterns Together

<div style="overflow-x:auto; margin-top:0.8rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.81rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.4rem 0.7rem; text-align:left; border:1px solid #cbd5e1;">Pattern</th>
      <th style="padding:0.4rem 0.7rem; text-align:left; border:1px solid #cbd5e1;">Problem it solves</th>
      <th style="padding:0.4rem 0.7rem; text-align:left; border:1px solid #cbd5e1;">Trade-off</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;"><strong>Inheritance</strong></td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Polymorphic docs in one collection</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Sparse fields; needs <code>product_type</code> index</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;"><strong>Computed</strong></td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Expensive aggregation on hot read path</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Stale by one write cycle; extra write logic</td>
    </tr>
    <tr>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;"><strong>Approximation</strong></td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">High-frequency counter writes</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Slight inaccuracy; application-side logic</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;"><strong>Reference</strong></td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Shared child docs (address reuse)</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Requires a second query to fetch referenced doc</td>
    </tr>
    <tr>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;"><strong>Schema Versioning</strong></td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Gradual schema migration</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Application must handle multiple versions</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;"><strong>Single Collection</strong></td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Related entities queried together</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Mixed document shapes; needs <code>docType</code> index</td>
    </tr>
    <tr>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;"><strong>Subset</strong></td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Large array, only first N items used</td>
      <td style="padding:0.4rem 0.7rem; border:1px solid #cbd5e1;">Controlled duplication; two-collection reads for full list</td>
    </tr>
  </tbody>
</table>
</div>

---

# Further Reading

- [notes/8f — Subset Pattern](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/8f%20-%20MongoDB%20Data%20Modeling%20Subset%20Pattern.md)
- [Course notes overview](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/README.md)
