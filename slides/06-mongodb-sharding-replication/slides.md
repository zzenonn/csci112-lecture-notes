---
theme: discs
title: "MongoDB Sharding and Replication"
highlighter: shiki
layout: cover
codeCopy: true
---

MongoDB Sharding and Replication

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Understand the **sharded cluster** architecture: shards, config servers, mongos
2. Explain **shard keys**, chunks, and data distribution
3. Compare **range-based** and **hashed** sharding strategies
4. Understand **replica sets** and the combined sharded + replicated topology
5. Weigh **advantages and disadvantages** of sharding
6. Know when to choose vertical scaling or replication-only instead

---
layout: section
---

# Scaling MongoDB

---

# Why Scale?

<div style="display:flex; gap:2rem; margin-top:0.75rem;">
  <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:6px; padding:1rem;">
    <div style="font-weight:700; margin-bottom:0.5rem;">Vertical Scaling</div>
    <div style="font-size:0.9rem; line-height:1.7; color:#444;">
      Upgrade the single server — more CPU, RAM, storage.<br><br>
      <strong>Limit:</strong> hardware ceilings; cloud providers have max instance sizes.
    </div>
  </div>
  <div style="flex:1; border:1.5px solid #00b0f0; border-radius:6px; padding:1rem; background:#f0f9ff;">
    <div style="font-weight:700; margin-bottom:0.5rem;">Horizontal Scaling</div>
    <div style="font-size:0.9rem; line-height:1.7; color:#444;">
      Add more machines, each handling a subset of the workload.<br><br>
      <strong>MongoDB's approach:</strong> <em>sharding</em> — distribute data across a cluster.
    </div>
  </div>
</div>

<div style="display:flex; justify-content:center; margin-top:1.5rem;">
  <img src="/diagram-horizontal-scaling.svg" style="height:110px;" />
</div>

---
layout: section
---

# Sharded Cluster Architecture

---

# Sharded Cluster Components

<div style="display:flex; gap:1.5rem; margin-top:1rem; font-size:0.9rem;">
  <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:8px; padding:1rem;">
    <div style="font-weight:700; font-size:1rem; margin-bottom:0.4rem;">Shards</div>
    <div style="color:#444; line-height:1.6;">Each shard holds a <strong>subset</strong> of the data. Each shard is typically deployed as a <strong>replica set</strong> for high availability.</div>
  </div>
  <div style="flex:1; border:1.5px solid #f97316; border-radius:8px; padding:1rem; background:#fff7ed;">
    <div style="font-weight:700; font-size:1rem; margin-bottom:0.4rem;">Config Servers</div>
    <div style="color:#444; line-height:1.6;">Store <strong>metadata</strong> and configuration for the cluster — which chunks live on which shard. Must be deployed as a replica set (CSRS).</div>
  </div>
  <div style="flex:1; border:1.5px solid #16a34a; border-radius:8px; padding:1rem; background:#f0fdf4;">
    <div style="font-weight:700; font-size:1rem; margin-bottom:0.4rem;">mongos</div>
    <div style="color:#444; line-height:1.6;">The <strong>query router</strong>. Applications connect only to mongos — never directly to shards. Routes operations based on the shard key.</div>
  </div>
</div>

<div style="margin-top:1.25rem; display:flex; align-items:center; justify-content:center; gap:1rem; font-size:0.85rem; color:#555;">
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.4rem 0.8rem;">Client App</div>
  <div>→</div>
  <div style="border:1.5px solid #16a34a; border-radius:6px; padding:0.4rem 0.8rem; background:#f0fdf4; font-weight:600;">mongos</div>
  <div>→</div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.4rem 0.8rem;">Shard 1</div>
  <div style="color:#aaa;">/</div>
  <div style="border:1.5px solid #cbd5e1; border-radius:6px; padding:0.4rem 0.8rem;">Shard 2</div>
</div>

---

# Shard Keys and Chunks

<div style="display:flex; gap:2rem; margin-top:0.5rem; align-items:flex-start;">
<div style="flex:1; font-size:0.88rem; line-height:1.6;">

**Shard Key** — field used to partition data across shards. Must be indexed; cannot be changed after sharding.

**Chunk** — a contiguous range of shard key values assigned to one shard. MongoDB splits and migrates chunks automatically to balance load.

</div>
<div style="flex:1.2; display:flex; justify-content:center; align-items:center;">
  <img src="/diagram-chunk-distribution.svg" style="width:100%; max-width:440px;" />
</div>
</div>

---

# Choosing a Shard Key

A poor shard key is one of the most common causes of performance problems in a sharded cluster.

<div style="display:flex; gap:2rem; margin-top:1rem;">
  <div style="flex:1;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#16a34a;">Good shard key</div>
    <ul style="font-size:0.9rem; line-height:1.8;">
      <li><strong>High cardinality</strong> — many distinct values</li>
      <li><strong>Even frequency</strong> — values spread uniformly, no hotspots</li>
      <li>Supports common query patterns</li>
    </ul>
  </div>
  <div style="flex:1;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#dc2626;">Poor shard key</div>
    <ul style="font-size:0.9rem; line-height:1.8;">
      <li>Low cardinality (e.g. boolean, status with 3 values)</li>
      <li>Monotonically increasing (e.g. timestamps, ObjectId) → all writes to one shard</li>
      <li>Uneven distribution → hot shard</li>
    </ul>
  </div>
</div>

---
layout: section
---

# Sharding Strategies

---

# Range-Based Sharding

Documents are assigned to shards based on **ranges of shard key values**.

<div style="display:flex; gap:2rem; margin-top:1rem;">
<div style="flex:1;">

```
Shard 1: key 0 – 149
Shard 2: key 150 – 299
```

**Advantages**
- Efficient for range queries
- Reflects natural data structure

</div>
<div style="flex:1;">

**Disadvantages**
- Uneven distribution if data isn't uniformly spread
- Sequential inserts can create a write hotspot on one shard

</div>
</div>

---

# Hashed Sharding

The shard key value is **hashed** before assignment — distributes data evenly regardless of key distribution.

<div style="display:flex; gap:2rem; margin-top:0.75rem; align-items:flex-start;">
<div style="flex:1; display:flex; justify-content:center;">
  <img src="/diagram-hash-function.svg" style="width:100%; max-width:420px;" />
</div>
<div style="flex:1; font-size:0.9rem;">

**Advantages**
- Even distribution across shards
- No hotspots on sequential inserts

**Disadvantages**
- Range queries must broadcast to all shards
- Resharding is expensive — significant downtime

</div>
</div>

---
layout: section
---

# Replication

---

# Replica Sets

A **replica set** is a group of MongoDB instances that maintain the same dataset.

<div style="display:flex; gap:2rem; margin-top:1rem; font-size:0.9rem;">
  <div style="flex:1; border:1.5px solid #f97316; border-radius:8px; padding:0.9rem; background:#fff7ed;">
    <div style="font-weight:700; margin-bottom:0.3rem;">Primary</div>
    <div style="color:#555; line-height:1.6;">Receives all <strong>write</strong> operations. Logs changes to the oplog which secondaries replicate.</div>
  </div>
  <div style="flex:2; display:flex; gap:1rem;">
    <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:8px; padding:0.9rem;">
      <div style="font-weight:700; margin-bottom:0.3rem;">Secondary</div>
      <div style="color:#555; line-height:1.6;">Replicates the primary's oplog. Can serve <strong>read</strong> operations. Eligible to become primary on failover.</div>
    </div>
    <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:8px; padding:0.9rem;">
      <div style="font-weight:700; margin-bottom:0.3rem;">Secondary</div>
      <div style="color:#555; line-height:1.6;">Replicates the primary's oplog. Can serve <strong>read</strong> operations. Eligible to become primary on failover.</div>
    </div>
  </div>
</div>

<div style="margin-top:1rem; font-size:0.85rem; color:#555;">
  If the primary fails, the secondaries elect a new primary automatically — <strong>no manual intervention required</strong>.
</div>

---

# Benefits of Replication

<table style="font-size:0.9rem; border-collapse:collapse; width:100%; margin-top:1rem;">
  <thead><tr style="border-bottom:2px solid #e2e8f0;">
    <th style="text-align:left; padding:0.4rem 0.75rem; width:30%;">Benefit</th>
    <th style="text-align:left; padding:0.4rem 0.75rem;">Description</th>
  </tr></thead>
  <tbody style="line-height:1.7;">
    <tr><td style="padding:0.35rem 0.75rem; font-weight:600;">High Availability</td><td style="padding:0.35rem 0.75rem;">Automatic failover if primary goes down</td></tr>
    <tr style="background:#f8fafc;"><td style="padding:0.35rem 0.75rem; font-weight:600;">Read Scaling</td><td style="padding:0.35rem 0.75rem;">Read operations can be distributed to secondaries</td></tr>
    <tr><td style="padding:0.35rem 0.75rem; font-weight:600;">Data Redundancy</td><td style="padding:0.35rem 0.75rem;">Multiple copies protect against data loss</td></tr>
    <tr style="background:#f8fafc;"><td style="padding:0.35rem 0.75rem; font-weight:600;">Disaster Recovery</td><td style="padding:0.35rem 0.75rem;">Deploy secondaries across availability zones</td></tr>
  </tbody>
</table>

---

# Sharded + Replicated Cluster

Each shard in a production cluster is itself a **replica set** — combining the benefits of both.

<div style="display:flex; justify-content:center; margin-top:1rem;">
  <img src="/diagram-cluster-topology.svg" style="width:90%; max-width:500px;" />
</div>

---

# Trade-offs

<div style="display:flex; gap:2rem; margin-top:1rem;">
  <div style="flex:1;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#16a34a;">Advantages</div>
    <ul style="font-size:0.9rem; line-height:1.9;">
      <li>Increased read/write throughput</li>
      <li>Increased storage capacity</li>
      <li>High availability via replication</li>
      <li>Scales horizontally as data grows</li>
    </ul>
  </div>
  <div style="flex:1;">
    <div style="font-weight:700; margin-bottom:0.5rem; color:#dc2626;">Disadvantages</div>
    <ul style="font-size:0.9rem; line-height:1.9;">
      <li>Additional query overhead (merging results)</li>
      <li>Increased latency for scatter-gather queries</li>
      <li>Complex administration</li>
      <li>Higher infrastructure cost</li>
    </ul>
  </div>
</div>

---

# When Not to Shard

Sharding introduces significant complexity — only adopt it when you need to.

<div style="display:flex; gap:2rem; margin-top:1rem;">
  <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:6px; padding:1rem;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Vertical Scaling</div>
    <div style="font-size:0.9rem; color:#444; line-height:1.6;">Simpler. Upgrade to a larger instance. Suitable when workload fits within the limits of available hardware.</div>
  </div>
  <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:6px; padding:1rem;">
    <div style="font-weight:700; margin-bottom:0.4rem;">Replication Only</div>
    <div style="font-size:0.9rem; color:#444; line-height:1.6;">Dataset fits on one server but you need HA and read scaling. No data partitioning required — just a replica set.</div>
  </div>
</div>

<div style="margin-top:1rem; font-size:0.85rem; color:#555; background:#fef9c3; border-left:4px solid #ca8a04; padding:0.75rem 1rem; border-radius:0 6px 6px 0;">
  Start with a replica set. Add sharding only when your data volume or throughput exceeds what a single replica set can handle.
</div>

---
layout: section
---

# Lab — Deploying on Cloud VMs

---

# Cluster on Cloud VMs

This lab deploys a minimal sharded + replicated cluster using **7 instances**:

<div style="display:flex; gap:1.5rem; margin-top:1rem; font-size:0.88rem;">
  <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:6px; padding:0.75rem;">
    <div style="font-weight:700;">2 × Config Servers</div>
    <div style="color:#666; margin-top:0.3rem;">Replica set (CSRS)<br>Port 27019</div>
  </div>
  <div style="flex:1; border:1.5px solid #f97316; border-radius:6px; padding:0.75rem; background:#fff7ed;">
    <div style="font-weight:700;">4 × Shard Servers</div>
    <div style="color:#666; margin-top:0.3rem;">2 shards × 2 nodes each<br>Port 27018</div>
  </div>
  <div style="flex:1; border:1.5px solid #16a34a; border-radius:6px; padding:0.75rem; background:#f0fdf4;">
    <div style="font-weight:700;">1 × mongos</div>
    <div style="color:#666; margin-top:0.3rem;">Query router<br>Port 27017</div>
  </div>
</div>

<div style="margin-top:1.25rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  Use the smallest available instance type (e.g. <code>e2-micro</code> on GCP, <code>t2.micro</code> on AWS). Stop instances when not in use. Full step-by-step instructions in <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/labs/lab05_mongodb_sharding.md">Lab 5</a>.
</div>

---

# Key Setup Steps

```bash
# 1. Start config server replica set (on each config server)
mongod --port 27019 --configsvr --replSet csrs \
       --bind_ip 0.0.0.0 --fork --logpath /var/log/mongodb/mongod.log

# 2. Start each shard as a replica set
mongod --port 27018 --shardsvr --replSet <replicasetname> \
       --bind_ip 0.0.0.0 --fork --logpath /var/log/mongodb/mongod.log

# 3. Start mongos, pointing at config servers
mongos --port 27017 --bind_ip 0.0.0.0 \
       --configdb csrs/<ip1>:27019,<ip2>:27019 \
       --fork --logpath /var/log/mongodb/mongod.log
```

<div style="margin-top:0.75rem; font-size:0.85rem; color:#666;">
  Full runbook including replica set initiation and shard registration: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/labs/lab05_mongodb_sharding.md">Lab 5</a>
</div>

---

# Enable Sharding on a Collection

Once mongos is running and shards are registered, enable sharding:

```javascript
// Connect to mongos, then:
use sample
db.grades.createIndex({ "class_id": "hashed" }, { background: true })
sh.enableSharding("sample")
sh.shardCollection("sample.grades", { "class_id": "hashed" }, false, { numInitialChunks: 2 })

// Verify distribution
db.grades.getShardDistribution()
```

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  These commands run in <strong>mongosh</strong> connected to the <strong>mongos</strong> router — not to any individual shard.
</div>

---
layout: end
---

# Thank You
