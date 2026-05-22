---
theme: discs
title: Databases in the Real World
highlighter: shiki
layout: cover
---

Databases in the Real World

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Understand the **design considerations** that need to be made when choosing a database technology
2. Understand the **workload requirements** for running a database

---
layout: section
---

# Design Considerations

When localhost is no longer enough

---

# How Have You Been Running Your Apps?

- `http://localhost:8080`
- Physical server
- Virtual Machines
- Serverless Infrastructure

---

# Where Do You Put Your Database?

<div style="display:flex; justify-content:center; margin-top:2rem;">
  <div style="border:2px solid #ccc; border-radius:8px; padding:2rem; width:65%;">
    <div style="background:#888; color:#fff; text-align:center; padding:1rem; border-radius:4px; margin-bottom:0.75rem; font-family:monospace;">
      PHP / Java / NodeJS / React / Etc
    </div>
    <div style="background:#666; color:#fff; text-align:center; padding:1rem; border-radius:4px; font-family:monospace;">
      MySQL / PostgreSQL / MSSQL / Etc
    </div>
    <div style="text-align:center; margin-top:0.75rem; color:#888;">Computer</div>
  </div>
</div>

---

# Design Considerations

### Performance

What are the **performance** features required for the workload?

- Required latency
- IOPS
- Read/write throughput
- Concurrency

---

# Where Do You Put Your Database?

### Separate Database Server

<div style="display:flex; align-items:center; justify-content:center; gap:2.5rem; margin-top:2.5rem;">
  <div style="display:flex; flex-direction:column; align-items:center; gap:0.5rem;">
    <div style="width:3.5rem; height:3.5rem; background:#dbeafe; border-radius:50%; display:flex; align-items:center; justify-content:center; font-size:1.75rem; border:2px solid #93c5fd;">👤</div>
    <span style="font-size:0.85rem; color:#888;">User</span>
  </div>
  <div style="color:#aaa; font-size:1.5rem; font-weight:bold;">→</div>
  <div style="display:flex; flex-direction:column; align-items:center; gap:0.5rem;">
    <div style="width:5rem; height:3.5rem; background:#dbeafe; border:2px solid #93c5fd; border-radius:6px; display:flex; align-items:center; justify-content:center; font-size:1.5rem;">🖥️</div>
    <span style="font-size:0.85rem; color:#888;">Application Server</span>
    <span style="font-size:0.7rem; color:#aaa;">http://example.com/</span>
  </div>
  <div style="color:#aaa; font-size:1.5rem; font-weight:bold;">→</div>
  <div style="display:flex; flex-direction:column; align-items:center; gap:0.5rem;">
    <div style="width:5rem; height:3.5rem; background:#ffedd5; border:2px solid #f97316; border-radius:6px; display:flex; align-items:center; justify-content:center; font-size:1.5rem;">🗄️</div>
    <span style="font-size:0.85rem; color:#888;">Database Server</span>
    <span style="font-size:0.7rem; color:#f97316;">Private Network</span>
  </div>
</div>

---

# Design Considerations

### High Availability

What are the **high availability** features required for the workload?

- Read replicas
- Clustering
- Geo-distributed deployments

---

# Where Do You Put Your Database?

### Primary-Replica Database Replication

<div style="display:flex; align-items:center; justify-content:center; gap:1.5rem; margin-top:1.5rem; font-size:0.9rem;">
  <div style="display:flex; flex-direction:column; align-items:center; gap:0.4rem;">
    <div style="font-size:2rem;">👤</div>
    <span style="color:#888;">User</span>
  </div>
  <div style="color:#aaa; font-size:1.25rem;">→</div>
  <div style="display:flex; flex-direction:column; align-items:center; gap:0.4rem;">
    <div style="width:4rem; height:2.5rem; background:#dbeafe; border:1.5px solid #93c5fd; border-radius:4px; display:flex; align-items:center; justify-content:center;">⚖️</div>
    <span style="color:#888; font-size:0.8rem;">Load Balancer</span>
    <span style="color:#aaa; font-size:0.7rem;">http://example.com/</span>
  </div>
  <div style="color:#aaa; font-size:1.25rem;">→</div>
  <div style="border:2px solid #f97316; border-radius:8px; padding:1rem;">
    <div style="color:#f97316; font-size:0.75rem; text-align:center; font-weight:bold; margin-bottom:0.5rem;">app-backend</div>
    <div style="display:flex; gap:1.5rem;">
      <div style="display:flex; flex-direction:column; gap:0.5rem;">
        <div style="width:3.5rem; height:2rem; background:#e5e7eb; border-radius:4px; display:flex; align-items:center; justify-content:center; font-size:0.75rem; color:#404040;">app-1</div>
        <div style="width:3.5rem; height:2rem; background:#e5e7eb; border-radius:4px; display:flex; align-items:center; justify-content:center; font-size:0.75rem; color:#404040;">app-2</div>
      </div>
      <div style="display:flex; flex-direction:column; gap:0.5rem;">
        <div style="display:flex; align-items:center; gap:0.5rem;">
          <div style="width:4rem; height:2rem; background:#bfdbfe; border:1.5px solid #3b82f6; border-radius:4px; display:flex; align-items:center; justify-content:center; font-size:0.75rem; color:#404040;">Primary</div>
          <span style="color:#888; font-size:0.7rem;">← read/write</span>
        </div>
        <div style="display:flex; align-items:center; gap:0.5rem;">
          <div style="width:4rem; height:2rem; background:#e5e7eb; border:1.5px solid #9ca3af; border-radius:4px; display:flex; align-items:center; justify-content:center; font-size:0.75rem; color:#404040;">Replica</div>
          <span style="color:#888; font-size:0.7rem;">← read</span>
        </div>
        <div style="font-size:0.65rem; color:#aaa; text-align:center;">Replication ↕</div>
      </div>
    </div>
    <div style="color:#f97316; font-size:0.7rem; text-align:center; margin-top:0.5rem;">Private Network</div>
  </div>
</div>

---

# Design Considerations

### Backup and Recovery

What are the **backup and recovery** features required for the workload?

- Automated and Scheduled Backups
- Point-in-Time Recovery (PITR)
- Disaster Recovery and Redundancy
- Fast Recovery and Minimal Downtime

---

# Based on Your Workload...

<div style="display:flex; align-items:center; justify-content:center; height:70%; text-align:center;">
  <p style="font-size:2.5rem; font-weight:700; color:#404040; line-height:1.4;">
    Identify which design criteria seem to be <strong style="color:#00b0f0;">most important</strong>.
  </p>
</div>

---
layout: section
---

# Workload Requirements

When SQL is no longer enough

---

# What Database Are You Running?

<div style="display:flex; justify-content:center; margin-top:1.5rem;">
  <img src="/db-sql.png" style="max-height:380px; object-fit:contain;" />
</div>

---

# Workload Requirements

### Data Storage

What type of **data storage** do you need for your workload?

- File system
- Object store
- Relational database
- Nonrelational database

---

# Workload Requirements

### Data Volume, Velocity, and Variety

What is the **volume, velocity, and variety** of data in your workload?

- Data volume
- Data velocity
- Data variety

---

# Workload Requirements

### Data Usage

How will the **data** in your workload be used?

- SQL data organization
  - OLTP or OLAP
  - DSS
  - Data warehouse
- NoSQL access patterns
  - IoT
  - Session state

---

# What Database Are You Running?

<div style="display:flex; justify-content:center; margin-top:1.5rem;">
  <img src="/db-logos.png" style="max-height:380px; object-fit:contain;" />
</div>

---
layout: end
---

# Thank You