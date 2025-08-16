# MongoDB: Sharding and Replication  
CSCI 122 / 212 - Contemporary Databases  

## Overview  
This document provides an overview of MongoDB's sharding and replication features. These are essential concepts for building scalable and highly available database systems. By the end of this module, students should be able to:  

- Understand the concept of database sharding  
- Understand the need for database replication  
- Implement a sharded and replicated cluster in MongoDB  

---

## Sharded Cluster Architecture  

A **sharded cluster** in MongoDB consists of the following components:

### Shard  
A shard is a subset of your data. Each shard is typically deployed as a replica set to ensure high availability and fault tolerance.

### Config Servers  
These servers store metadata and configuration settings for the cluster. They are essential for managing the cluster's state and routing queries.

### mongos  
The mongos acts as a query router. It interfaces with client applications and directs operations to the appropriate shard(s) based on the shard key.

---

## Sharding a Collection  

Sharding is the process of distributing a dataset across multiple servers. This allows for horizontal scaling, enabling the system to handle more data and traffic by adding more machines.

### Key Concepts  

- **Shard Key**: The field used to partition data across shards. The choice of shard key is critical and directly impacts performance and scalability.  
- **Chunk**: A chunk is a range of data defined by shard key values. MongoDB automatically splits and migrates chunks to balance the load across shards.  
- **Horizontal Scaling**: Also known as "scaling out," this involves adding more machines to distribute the load, as opposed to vertical scaling which increases the power of a single machine.

### Chunk Management  

- MongoDB splits chunks when they exceed a configured size.  
- Inserts and updates can trigger chunk splits.  
- The smallest chunk range is a single unique shard key value.  
- Chunks can be moved between shards to balance load without changing the shard key.

Example:  
If the shard key is the `name` field and all names starting with "M" are in one chunk, that chunk can be split into "M-A" and "M-B" chunks. These can then be distributed to different shards without resharding.

---

## Choosing a Shard Key  

Two important attributes of an effective shard key:

- **High Cardinality**: The key should have many possible values to allow for even distribution.  
- **Well-Distributed Frequency**: The values should be evenly distributed to avoid hotspots.

A poor shard key can lead to uneven chunk distribution, performance bottlenecks, and scalability issues.

---

## Sharding Strategies  

### Range-Based Sharding  
- Documents are partitioned based on a range of shard key values.  
- Requires a lookup table to determine which range belongs to which shard.  
- Reflects the natural structure of the data.  
- Efficient for range queries but can lead to uneven data distribution.

### Hashed Sharding  
- The shard key is hashed to ensure even distribution across shards.  
- No lookup table required.  
- Reduces the risk of hotspots.  
- Drawbacks:
  - Queries may be broadcast to multiple shards.
  - Resharding is expensive and may require significant downtime.

---

## Replication in MongoDB  

Replication is the process of synchronizing data across multiple MongoDB servers. A replica set consists of:

- **Primary Node**: Handles all write operations.  
- **Secondary Nodes**: Maintain copies of the primary's data and can serve read operations.

### Benefits of Replication  

- **High Availability**: Failover to a secondary node if the primary fails.  
- **Read Scaling**: Read operations can be distributed across secondaries.  
- **Data Redundancy**: Protects against data loss.

---

## Sharded and Replicated Cluster  

Each shard in a sharded cluster is typically a replica set. This architecture combines the benefits of both sharding and replication:

### Advantages  

- Increased read/write throughput  
- Increased storage capacity  
- High availability  

### Disadvantages  

- Additional query overhead (e.g., merging results from multiple shards)  
- Increased latency  
- More complex administration  
- Higher infrastructure costs  

---

## Alternatives to Sharding  

### Vertical Scaling  
- Involves upgrading the existing server (e.g., more CPU, RAM, or storage).  
- Simpler to implement but has physical and cost limitations.

### Replication Without Sharding  
- Suitable for applications with moderate data and traffic requirements.  
- Easier to set up and manage.  
- No need for data partitioning.  
- Ideal when the dataset fits within a single server's capacity.

---

## Summary  

Sharding and replication are powerful tools for building scalable and resilient MongoDB deployments. However, they introduce complexity and cost. Choosing the right architecture depends on your application's specific needs, including data volume, traffic patterns, and availability requirements.  

When designing your MongoDB deployment:

- Choose your shard key carefully  
- Understand the trade-offs between range-based and hashed sharding  
- Consider whether replication alone is sufficient for your use case  
- Be aware of the operational complexity and plan accordingly  

For further reading and best practices, consult the official MongoDB documentation.
