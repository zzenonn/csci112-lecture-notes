---
theme: discs
title: Databases in the Real World
highlighter: shiki
layout: cover
---

Design considerations and workload requirements

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Understand the **design considerations** that need to be made when choosing a database technology
2. Understand the **workload requirements** for running a database

<div style="margin-top:1.5rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.7rem 1.1rem; font-size:0.95rem;">
  Later in this course you will meet the specific design criteria for each database service. For now, the goal is to know <strong>which questions to ask</strong> before picking one.
</div>

---
layout: section
---

# Design Considerations

When localhost is no longer enough

---

# Where Does Your Database Live?

<div style="display:grid; grid-template-columns:1fr 1.15fr; gap:1.5rem; align-items:start; margin-top:0.5rem;">
<div>

How have you been running your apps?

- `http://localhost:8080`
- Physical server
- Virtual Machines
- Serverless Infrastructure

</div>
<div style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:1rem;">
  <div style="font-size:0.8rem; color:#666; text-align:center; margin-bottom:0.6rem;">Co-located: one machine</div>
  <div style="background:#9ca3af; color:#fff; text-align:center; padding:0.7rem; border-radius:4px; margin-bottom:0.5rem; font-size:0.85rem;">
    PHP / Java / NodeJS / React
  </div>
  <div style="background:#6b7280; color:#fff; text-align:center; padding:0.7rem; border-radius:4px; font-size:0.85rem;">
    MySQL / PostgreSQL / MSSQL
  </div>
  <div style="text-align:center; margin-top:0.6rem; color:#888; font-size:0.8rem;">Computer</div>
</div>
</div>

<div style="margin-top:0.9rem; font-size:0.95rem;">
  Everything on one box is the simplest thing that works — until one of the considerations below forces it apart.
</div>

---

# Performance

<div style="display:grid; grid-template-columns:1fr 1.2fr; gap:1.5rem; align-items:start; margin-top:0.3rem;">
<div>

What **performance** features does the workload require?

- Required latency
- IOPS
- Read/write throughput
- Concurrency

</div>
<div style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:1rem; font-size:0.85rem;">
  <div style="font-size:0.8rem; color:#666; text-align:center; margin-bottom:0.7rem;">Separate database server</div>
  <div style="display:flex; align-items:center; justify-content:center; gap:0.6rem; margin-bottom:0.6rem;">
    <div style="border:1.5px solid #93c5fd; background:#dbeafe; border-radius:4px; padding:0.4rem 0.7rem;">User</div>
    <span style="color:#9ca3af;">&rarr;</span>
    <div style="border:1.5px solid #93c5fd; background:#dbeafe; border-radius:4px; padding:0.4rem 0.7rem; text-align:center;">
      Application Server<br><span style="font-size:0.7rem; color:#888;">http://example.com/</span>
    </div>
  </div>
  <div style="text-align:center; color:#9ca3af; margin-bottom:0.4rem;">&darr;</div>
  <div style="border:1.5px solid #f97316; border-radius:6px; padding:0.6rem;">
    <div style="background:#ffedd5; border:1.5px solid #fdba74; border-radius:4px; padding:0.4rem; text-align:center;">Database Server</div>
    <div style="color:#f97316; font-size:0.72rem; text-align:center; margin-top:0.4rem;">Private Network</div>
  </div>
</div>
</div>

<div style="margin-top:0.8rem; font-size:0.95rem;">
  Moving the database onto its own host stops the application from competing with it for CPU, memory, and disk.
</div>

---

# High Availability

<div style="display:grid; grid-template-columns:1fr 1.3fr; gap:1.4rem; align-items:start; margin-top:0.3rem;">
<div>

What **high availability** features does the workload require?

- Read replicas
- Clustering
- Geo-distributed deployments

</div>
<div style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:0.9rem; font-size:0.82rem;">
  <div style="font-size:0.78rem; color:#666; text-align:center; margin-bottom:0.6rem;">Primary&ndash;replica replication</div>
  <div style="display:flex; align-items:center; justify-content:center; gap:0.5rem; margin-bottom:0.6rem;">
    <div style="border:1.5px solid #93c5fd; background:#dbeafe; border-radius:4px; padding:0.35rem 0.6rem;">User</div>
    <span style="color:#9ca3af;">&rarr;</span>
    <div style="border:1.5px solid #93c5fd; background:#dbeafe; border-radius:4px; padding:0.35rem 0.6rem;">Load Balancer</div>
  </div>
  <div style="text-align:center; color:#9ca3af; margin-bottom:0.4rem;">&darr;</div>
  <div style="border:1.5px solid #f97316; border-radius:6px; padding:0.6rem;">
    <div style="display:flex; gap:0.8rem; align-items:center; justify-content:center;">
      <div style="display:flex; flex-direction:column; gap:0.35rem;">
        <div style="background:#e5e7eb; border-radius:4px; padding:0.3rem 0.7rem; text-align:center;">app-1</div>
        <div style="background:#e5e7eb; border-radius:4px; padding:0.3rem 0.7rem; text-align:center;">app-2</div>
      </div>
      <span style="color:#9ca3af;">&rarr;</span>
      <div style="display:flex; flex-direction:column; gap:0.35rem;">
        <div style="background:#bfdbfe; border:1.5px solid #3b82f6; border-radius:4px; padding:0.3rem 0.6rem;">Primary <span style="color:#666; font-size:0.72rem;">r/w</span></div>
        <div style="text-align:center; color:#9ca3af; font-size:0.72rem;">&updownarrow; replication</div>
        <div style="background:#e5e7eb; border:1.5px solid #9ca3af; border-radius:4px; padding:0.3rem 0.6rem;">Replica <span style="color:#666; font-size:0.72rem;">read</span></div>
      </div>
    </div>
    <div style="color:#f97316; font-size:0.72rem; text-align:center; margin-top:0.5rem;">Private Network</div>
  </div>
</div>
</div>

---

# Backup and Recovery

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1.5rem; align-items:start; margin-top:0.3rem;">
<div>

What **backup and recovery** features does the workload require?

- Automated and scheduled backups
- Point-in-Time Recovery (PITR)
- Disaster recovery and redundancy
- Fast recovery and minimal downtime

</div>
<div style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:1rem; font-size:0.88rem;">
  <div style="margin-bottom:0.8rem;">
    <div style="font-weight:700; color:#0369a1;">RPO &mdash; Recovery Point Objective</div>
    How much data can you afford to lose? Sets your <strong>backup frequency</strong>.
  </div>
  <div>
    <div style="font-weight:700; color:#0369a1;">RTO &mdash; Recovery Time Objective</div>
    How long can you afford to be down? Sets your <strong>restore strategy</strong>.
  </div>
</div>
</div>

<div style="margin-top:0.9rem; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.6rem 1.1rem; font-size:0.95rem;">
  A backup you have never restored is not a backup. Both numbers are business decisions, not technical ones.
</div>

---

<div style="display:flex; align-items:center; justify-content:center; height:78%; text-align:center;">
  <p style="font-size:2.4rem; font-weight:700; color:#404040; line-height:1.4;">
    Based on your workload, identify which design criteria are <strong style="color:#00b0f0;">most important</strong>.
  </p>
</div>

---
layout: section
---

# Workload Requirements

When SQL is no longer enough

---

# What Database Are You Running?

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1.5rem; align-items:center; margin-top:0.5rem;">
<div style="display:flex; justify-content:center;">
  <img src="/db-sql.png" style="max-height:280px; object-fit:contain;" />
</div>
<div>

For most of your degree, the answer has been a **relational database** — and for most workloads, that is still the right answer.

The rest of this section is about the workloads where it stops being the right answer.

</div>
</div>

---

# Data Storage

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1.5rem; align-items:start; margin-top:0.3rem;">
<div>

What type of **data storage** does the workload need?

- File system
- Object store
- Relational database
- Nonrelational database

</div>
<div style="background:#f8fafc; border:1px solid #e2e8f0; border-radius:8px; padding:1rem; font-size:0.88rem;">
  Not every problem is a database problem. Large blobs, media, and logs usually belong in a <strong>file system</strong> or <strong>object store</strong>, with only their metadata in the database.
</div>
</div>

---

# Data Volume, Velocity, and Variety

<div style="display:grid; grid-template-columns:1fr 1.25fr; gap:1.4rem; align-items:start; margin-top:0.3rem;">
<div>

What is the **volume, velocity, and variety** of the workload's data?

- Data volume
- Data velocity
- Data variety

</div>
<div style="font-size:0.86rem;">
<table style="width:100%; border-collapse:collapse;">
  <tbody>
    <tr>
      <td style="border:1px solid #e2e8f0; padding:0.45rem 0.7rem; background:#f8fafc; font-weight:700; white-space:nowrap;">Volume</td>
      <td style="border:1px solid #e2e8f0; padding:0.45rem 0.7rem;">How much data, and how fast does it grow?</td>
    </tr>
    <tr>
      <td style="border:1px solid #e2e8f0; padding:0.45rem 0.7rem; background:#f8fafc; font-weight:700; white-space:nowrap;">Velocity</td>
      <td style="border:1px solid #e2e8f0; padding:0.45rem 0.7rem;">How quickly does it arrive, and must it be read back?</td>
    </tr>
    <tr>
      <td style="border:1px solid #e2e8f0; padding:0.45rem 0.7rem; background:#f8fafc; font-weight:700; white-space:nowrap;">Variety</td>
      <td style="border:1px solid #e2e8f0; padding:0.45rem 0.7rem;">Does every record have the same shape?</td>
    </tr>
  </tbody>
</table>
</div>
</div>

---

# Data Usage

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1.5rem; align-items:start; margin-top:0.3rem;">
<div style="border:1.5px solid #93c5fd; border-radius:8px; padding:0.9rem 1.1rem; background:#f0f9ff; font-size:0.92rem;">
  <div style="font-weight:700; color:#0369a1; margin-bottom:0.4rem;">SQL data organization</div>
  <ul style="margin:0; padding-left:1.2rem;">
    <li style="font-size:0.92rem; margin:0.25rem 0;">OLTP or OLAP</li>
    <li style="font-size:0.92rem; margin:0.25rem 0;">DSS</li>
    <li style="font-size:0.92rem; margin:0.25rem 0;">Data warehouse</li>
  </ul>
</div>
<div style="border:1.5px solid #86efac; border-radius:8px; padding:0.9rem 1.1rem; background:#f0fdf4; font-size:0.92rem;">
  <div style="font-weight:700; color:#166534; margin-bottom:0.4rem;">NoSQL access patterns</div>
  <ul style="margin:0; padding-left:1.2rem;">
    <li style="font-size:0.92rem; margin:0.25rem 0;">IoT</li>
    <li style="font-size:0.92rem; margin:0.25rem 0;">Session state</li>
  </ul>
</div>
</div>

<div style="margin-top:1rem; font-size:0.95rem;">
  How the data will be <strong>used</strong> matters more than how it is stored. SQL designs start from the <strong>schema</strong>; NoSQL designs start from the <strong>queries</strong> — a theme for the rest of the course.
</div>

---

# What Could You Be Running?

<div style="display:flex; justify-content:center; margin-top:0.3rem;">
  <img src="/db-logos.png" style="max-height:260px; object-fit:contain;" />
</div>

<div style="margin-top:0.9rem; font-size:0.95rem; text-align:center;">
  Each engine trades the criteria above against each other differently. Choosing well is the point of this course.
</div>

---
layout: end
---

# Thank You
