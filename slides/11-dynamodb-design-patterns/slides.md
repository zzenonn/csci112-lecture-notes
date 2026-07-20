---
theme: discs
title: "DynamoDB Design Patterns"
highlighter: shiki
layout: cover
codeCopy: true
---

DynamoDB Design Patterns

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Identify the limitations of a generic "everything-table" design
2. Apply **natural keys** over generic identifiers for self-documenting schemas
3. Use **item collections** to group related entities without GSIs
4. Make aggregate boundary decisions using access correlation analysis
5. Apply additional patterns: denormalization, sparse GSIs, hierarchical sort keys

---
layout: section
---

# Problems with the Everything-Table

---

# What We Built in Module 10

```
Base table: entity_type (PK) + entity_id (SK)
GSI1: user-index  — username (PK) + entity_type + entity_id (SK)
GSI2: order-index — order_id (PK) + entity_type + entity_id (SK)
```

This eliminates synthetic key prefixes and works. But as the application grows:

<div style="margin-top:0.8rem; display:grid; grid-template-columns:1fr 1fr; gap:0.8rem; font-size:0.85rem;">
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">Generic keys are unreadable</div>
    <code>entity_type = 'ORDER' AND entity_id = 'abc123'</code><br>
    vs.<br>
    <code>order_id = 'abc123'</code><br><br>
    Every developer must understand the entire design before writing a query.
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">Operational coupling</div>
    One Stream with mixed user/order/item events.<br>
    Cannot backup orders without users.<br>
    Cannot restrict a microservice to only order data.<br>
    Write spikes on items affect user reads.
  </div>
</div>

---

# Principle 1: Natural Keys

Your keys should describe what they identify:

<div style="overflow-x:auto; margin-top:0.9rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.86rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;">Instead of...</th>
      <th style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;">Use...</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>entity_type</code></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>user_id</code>, <code>order_id</code>, <code>product_sku</code></td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>sort_key</code></td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>review_id</code>, <code>lesson_id</code>, <code>ORDER#abc123</code></td>
    </tr>
    <tr>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>user-index</code> (GSI name)</td>
      <td style="padding:0.5rem 0.8rem; border:1px solid #cbd5e1;"><code>OrdersByCustomer</code></td>
    </tr>
  </tbody>
</table>
</div>

<div style="margin-top:0.9rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.5rem 1rem; font-size:0.86rem;">
  A new developer looking at <code>PK: user_id = 'aarchamb', SK: 'ORDER#abc123'</code> immediately understands the relationship — without reading documentation first.
</div>

---

# Principle 2: Item Collections

An **item collection** is a group of items that share the same partition key.

```
UserOrders table  (PK: user_id, SK: sk)
┌──────────┬──────────────┬─────────────────────────────┐
│ user_id  │ sk           │ Attributes                  │
├──────────┼──────────────┼─────────────────────────────┤
│ aarchamb │ PROFILE      │ email, address, fullname    │
│ aarchamb │ ORDER#abc123 │ address, status, created_at │
│ tgrimes1 │ PROFILE      │ email, address, fullname    │
│ tgrimes1 │ ORDER#xyz789 │ address, status, created_at │
└──────────┴──────────────┴─────────────────────────────┘
```

All three common patterns hit the **base table** — no GSI needed: `get_item` on `PROFILE` (profile), `query` on `user_id` alone (everything), or add `& Key('sk').begins_with('ORDER#')` (orders only).

---

# Identifying Relationships

An **identifying relationship** exists when a child entity cannot exist without its parent. Use the parent's ID as the partition key.

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:0.9rem; font-size:0.85rem;">
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#991b1b;">Standard (more expensive)</div>
    <code>Orders</code> table: PK = <code>order_id</code><br>
    Need a GSI to query "orders by user"<br>
    Extra GSI storage + write amplification
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.3rem; color:#166534;">Item collection (cost-optimized)</div>
    <code>UserOrders</code> table: PK = <code>user_id</code><br>
    SK = <code>ORDER#&lt;order_id&gt;</code><br>
    Query by user hits base table directly — no GSI
  </div>
</div>

<div style="margin-top:0.8rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 1rem; font-size:0.85rem;">
  If the parent ID is always available when querying children, use an identifying relationship — it saves ~50% in write costs (no GSI overhead).
</div>

---
layout: section
---

# Aggregate-Oriented Design

---

# Access Correlation Analysis

Before deciding how to group entities, measure how often they are accessed together:

<div style="display:grid; grid-template-columns:1fr 1fr 1fr; gap:0.7rem; margin-top:0.9rem; font-size:0.84rem;">
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.8rem; background:#f0fdf4; text-align:center;">
    <div style="font-size:1.2rem; font-weight:700; color:#16a34a;">&gt;90%</div>
    <div style="font-weight:700; margin-bottom:0.3rem;">together</div>
    Single item aggregate<br>(one DynamoDB item)
  </div>
  <div style="border:1.5px solid #ca8a04; border-radius:6px; padding:0.8rem; background:#fef9c3; text-align:center;">
    <div style="font-size:1.2rem; font-weight:700; color:#ca8a04;">50–90%</div>
    <div style="font-weight:700; margin-bottom:0.3rem;">together</div>
    Item collection aggregate<br>(same partition, different SK)
  </div>
  <div style="border:1.5px solid #dc2626; border-radius:6px; padding:0.8rem; background:#fef2f2; text-align:center;">
    <div style="font-size:1.2rem; font-weight:700; color:#dc2626;">&lt;50%</div>
    <div style="font-weight:700; margin-bottom:0.3rem;">together</div>
    Separate tables
  </div>
</div>

<div style="margin-top:0.9rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.5rem 1rem; font-size:0.85rem;">
  Also check: different scaling patterns? Different backup needs? Different IAM scope? If yes → separate tables, regardless of access correlation.
</div>

---

# Redesigned: Two Tables

Applying the framework to users, orders, and items:

<div style="display:grid; grid-template-columns:1fr 1fr; gap:0.9rem; margin-top:0.9rem; font-size:0.85rem;">
  <div style="border:1.5px solid #00b0f0; border-radius:6px; padding:0.9rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#0369a1;">UserOrders table</div>
    <code>user_id</code> (PK) + <code>sk</code> (SK)<br><br>
    SK = <code>PROFILE</code> for user data<br>
    SK = <code>ORDER#abc123</code> for orders<br><br>
    Users + orders are accessed together ~60% → item collection
  </div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.9rem; background:#f0fdf4;">
    <div style="font-weight:700; margin-bottom:0.4rem; color:#166534;">OrderItems table</div>
    <code>order_id</code> (PK) + <code>item_id</code> (SK)<br><br>
    Identifying relationship: items belong to an order<br><br>
    Separate table: items have different update patterns and could have independent scaling needs
  </div>
</div>

---

# Module 10 vs Improved Design

<div style="margin-top:0.6rem;">
<table style="width:100%; border-collapse:collapse; font-size:0.82rem;">
  <thead>
    <tr style="background:#f1f5f9;">
      <th style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem; text-align:left;">Aspect</th>
      <th style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem; text-align:left;">Module 10</th>
      <th style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem; text-align:left;">Improved</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Tables</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">1 (3 entity types)</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">2 (UserOrders, OrderItems)</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Base table keys</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Generic: <code>entity_type</code>, <code>entity_id</code></td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Natural: <code>user_id</code>+<code>sk</code>, <code>order_id</code>+<code>item_id</code></td>
    </tr>
    <tr>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Get user + orders</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Requires GSI</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Base table query</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Get items for order</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Requires GSI</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Base table query</td>
    </tr>
    <tr>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">GSIs for basic queries</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">2 minimum</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">0</td>
    </tr>
    <tr style="background:#fafafa;">
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">DynamoDB Streams</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Mixed events</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Separate per table</td>
    </tr>
    <tr>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Cost</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">GSI write amplification</td>
      <td style="border:1px solid #cbd5e1; padding:0.35rem 0.7rem;">Lower — fewer GSIs</td>
    </tr>
  </tbody>
</table>
</div>

---
layout: section
---

# Additional Patterns

---

# Short-Circuit Denormalization

Duplicate a frequently needed attribute from a related entity to avoid an extra lookup:

<div style="display:grid; grid-template-columns:1.1fr 1fr; gap:1rem; align-items:start; margin-top:0.5rem;">
<div style="font-size:0.72rem;">

```python
# product_name copied onto the line item
item = {
    'order_id': 'abc123',
    'item_id': 'itm001',
    'product_id': 'prod-macbook',
    'product_name': 'New Apple MacBook Pro',
    #   ↑ duplicated — avoids a lookup
    'quantity': 1,
    'price': Decimal('2018.42')
}
```

</div>
<div style="font-size:0.8rem;">

**Use when:**
1. The access pattern would otherwise require a cross-table lookup
2. The duplicated attribute is mostly immutable (product names rarely change)
3. The attribute is small

</div>
</div>

<div style="margin-top:0.6rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.5rem 0.9rem; font-size:0.85rem;">
  If the source value changes, you must update all copies. Only denormalize stable attributes.
</div>

---

# Sparse GSIs

Only write a GSI entry when a specific attribute is present — automatically:

```python
# Active order — has 'active_status' → appears in GSI
active_order = {
    'user_id': 'aarchamb', 'sk': 'ORDER#abc123',
    'status': 'Processing',
    'active_status': 'Processing'  # Present → included in GSI
}

# Completed order — no 'active_status' → excluded from GSI
completed_order = {
    'user_id': 'aarchamb', 'sk': 'ORDER#def456',
    'status': 'Delivered'
    # Missing 'active_status' → excluded from GSI automatically
}
```

<div style="margin-top:0.6rem; background:#f0fdf4; border-left:4px solid #16a34a; padding:0.5rem 0.9rem; font-size:0.85rem;">
  If only 5% of orders are active, the GSI is 95% smaller. Remove <code>active_status</code> when an order completes and it disappears from the GSI instantly.
</div>

---

# Hierarchical Sort Keys & Time

**Hierarchical sort key** — encode hierarchy in the SK for prefix queries. For GSIs on the same data, use **multi-attribute keys** instead of composite strings.

```python
# LocationData: PK = device_id, SK = "year#month#day#time"
table.query(Key('device_id').eq('sensor-01') & Key('sk').begins_with('2024#01'))
# All readings for device in January 2024
```

**Timestamp format** — choose based on query needs:

| Format | Example | Best for |
|--------|---------|----------|
| ISO 8601 string | `"2024-01-15T14:30:00Z"` | Human-readable, natural sort order |
| Unix epoch (number) | `1705326600` | Math ops; **required** for TTL |

---

# Further Reading

- [notes/11 — DynamoDB Design Patterns](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/11%20-%20DynamoDB%20Design%20Patterns.md)
- [Course notes overview](https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/README.md)
