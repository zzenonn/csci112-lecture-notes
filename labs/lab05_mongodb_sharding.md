# Lab 5: Deploying a Sharded MongoDB Cluster on Cloud VMs

CSCI 112 / 212 - Contemporary Databases

**NOTE: Read the entire lab before starting. All values enclosed in `<>` must be replaced with your actual values.**

---

## Overview

This lab walks you through deploying a minimal sharded and replicated MongoDB cluster on cloud VMs (GCP, AWS, or any provider). The goal is to understand how the components fit together — not to build a production-grade system.

**Cluster topology for this lab:**

| Component | Count | Port |
|---|---|---|
| Config Servers (CSRS replica set) | 2 | 27019 |
| Shard 1 (replica set) | 2 nodes | 27018 |
| Shard 2 (replica set) | 2 nodes | 27018 |
| mongos (query router) | 1 | 27017 |
| **Total instances** | **7** | |

> **Cost tip:** Use the smallest available instance type (e.g. `e2-micro` on GCP, `t2.micro` on AWS). Stop instances when not in use — private IPs are preserved on restart; public IPs may change.

---

## Prerequisites

- 7 VMs provisioned with Rocky Linux (or equivalent)
- MongoDB installed on each (`mongod` and `mongos` available)
- Firewall rules open: port 27017, 27018, 27019 between all VMs
- SSH access to each instance

---

## Steps

### 1. Start Config Servers

Run on **both** config server instances:

```bash
sudo systemctl stop mongod

sudo mongod --port 27019 --configsvr --replSet csrs \
  --bind_ip 0.0.0.0 --fork \
  --logpath /var/log/mongodb/mongod.log
```

### 2. Initiate Config Server Replica Set

SSH to one config server, connect to mongosh, and initiate the replica set:

```bash
mongosh --port 27019 --host <config1_ip>
```

```javascript
rs.initiate({
  _id: "csrs",
  configsvr: true,
  members: [
    { "_id": 0, "host": "<config1_ip>:27019" },
    { "_id": 1, "host": "<config2_ip>:27019" }
  ]
})

rs.status()
```

`rs.status()` should show one primary and one secondary.

---

### 3. Start Shard Servers

Run on **both** nodes of Shard 1 (use `shard1` as the replica set name):

```bash
sudo systemctl stop mongod

sudo mongod --port 27018 --shardsvr --replSet shard1 \
  --bind_ip 0.0.0.0 --fork \
  --logpath /var/log/mongodb/mongod.log
```

Repeat on **both** nodes of Shard 2 (use `shard2` as the replica set name).

### 4. Initiate Shard Replica Sets

SSH to one node of Shard 1:

```bash
mongosh --port 27018 --host <shard1_node1_ip>
```

```javascript
rs.initiate({
  _id: "shard1",
  members: [
    { "_id": 0, "host": "<shard1_node1_ip>:27018" },
    { "_id": 1, "host": "<shard1_node2_ip>:27018" }
  ]
})

rs.status()
```

Repeat for Shard 2 (`shard2`, using its two node IPs).

---

### 5. Start mongos

Run on the mongos instance:

```bash
sudo mongos --port 27017 --bind_ip 0.0.0.0 \
  --configdb csrs/<config1_ip>:27019,<config2_ip>:27019 \
  --fork \
  --logpath /var/log/mongodb/mongod.log
```

### 6. Register Shards with mongos

Connect to the mongos instance:

```bash
mongosh --port 27017 --host <mongos_ip>
```

```javascript
sh.addShard("shard1/<shard1_node1_ip>:27018,<shard1_node2_ip>:27018")
sh.addShard("shard2/<shard2_node1_ip>:27018,<shard2_node2_ip>:27018")
sh.status()
db.adminCommand({ listShards: 1 })
```

---

### 7. Enable Sharding on a Collection

While connected to mongos, shard the `grades` collection by `class_id`:

```javascript
use sample
db.grades.createIndex({ "class_id": "hashed" }, { background: true })
sh.enableSharding("sample")
sh.shardCollection("sample.grades", { "class_id": "hashed" }, false, { numInitialChunks: 2 })
```

Import data and verify distribution:

```bash
mongoimport -d sample -c grades --host <mongos_ip> grades.json
```

```javascript
db.grades.getShardDistribution()
```

The data should be roughly evenly split across the two shards.

---

## Expected Output

1. Infrastructure diagram including VM IPs, ports, replica set names, and network configuration
2. Running cluster demonstrated live — `sh.status()` showing both shards registered and data distributed
3. `db.grades.getShardDistribution()` output showing balanced chunks across shards

---

## Additional Notes

- Stopping a VM may change its **public IP** but will preserve its **private IP**. Update `mongod` startup scripts with the correct IPs after restarting.
- Always organize your commands before running (replace all `<placeholders>` first).
- For security roles on a sharded cluster, refer to: https://www.mongodb.com/docs/manual/core/security-users/#sharded-cluster-users
