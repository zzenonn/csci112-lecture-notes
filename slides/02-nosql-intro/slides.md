---
theme: discs
title: "Introduction to NoSQL"
highlighter: shiki
layout: cover
---

Introduction to NoSQL

::author-info::
Miguel Zenon Nicanor Lerias Saavedra

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Understand the theory of **NoSQL databases**
2. Understand the **NoSQL BASE model**
3. Contrast **SQL and NoSQL** data models

---
layout: section
---

# What is NoSQL?

---

# CAP Theorem

<div style="display:flex; flex-direction:column; align-items:center; margin-top:0.5rem; gap:0;">
  <!-- Top label: Consistency -->
  <div style="text-align:center; margin-bottom:0.25rem;">
    <div style="font-weight:700; font-size:1.1rem; color:#404040;">Consistency</div>
    <div style="font-size:0.8rem; color:#666;">Ensuring that every request receives the most recent write</div>
  </div>
  <!-- Triangle SVG -->
  <svg width="460" height="180" viewBox="0 0 460 180" xmlns="http://www.w3.org/2000/svg">
    <polygon points="230,8 30,172 430,172" fill="#f0f9ff" stroke="#00b0f0" stroke-width="2.5"/>
    <circle cx="230" cy="8" r="5" fill="#00b0f0"/>
    <circle cx="30" cy="172" r="5" fill="#00b0f0"/>
    <circle cx="430" cy="172" r="5" fill="#00b0f0"/>
  </svg>
  <!-- Bottom labels: Availability and Partition Tolerance -->
  <div style="display:flex; justify-content:space-between; width:460px; margin-top:0.25rem;">
    <div style="text-align:center; width:45%;">
      <div style="font-weight:700; font-size:1.1rem; color:#404040;">Availability</div>
      <div style="font-size:0.8rem; color:#666;">Ensuring that all systems produce a non-error response</div>
    </div>
    <div style="text-align:center; width:45%;">
      <div style="font-weight:700; font-size:1.1rem; color:#404040;">Partition Tolerance</div>
      <div style="font-size:0.8rem; color:#666;">Ensuring that the system can survive the loss of a partition</div>
    </div>
  </div>
</div>

---

# ACID Compliance

<div style="line-height:1.4; font-size:1.1rem;">

**A**tomic — Complete success or failure

**C**onsistent — Data always follows schema and always returns the correct result

**I**solated — One transaction cannot interfere with another

**D**urable — Ensure that changes are permanent

</div>

---

# BASE

**B**asic **A**vailability
- Highly available

**S**oft-state
- System may change over time even without input

**E**ventual consistency
- System will be consistent over time as long as there is no more additional input


---

# NoSQL Schema

```json
{
  "_id": "ObjectId(\"5b8382b8c0eec0fe6479e901\")",
  "name": "Quezon City",
  "population": 2936116,
  "famous_for": ["circle", "mayor has an MA in Archeology"],
  "mayor": {
    "name": "Joy Belmonte",
    "party": "HNP"
  }
}
```

---

# Data Models: SQL vs NoSQL

| Characteristic | SQL | NoSQL |
|---|---|---|
| **Representation** | Multiple tables, columns and rows | Collection of documents; keys and values |
| **Primary Informer** | Entities | Access patterns |
| **Data Design** | Normalized relational or data warehouse | Denormalized document, wide column or key-value |
| **Query Style** | SQL | Multiple, depending on database service |
| **Scaling** | Vertical | Horizontal |


---
layout: end
---

# Thank You
