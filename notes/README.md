# Lecture Notes - CSCI 112 Contemporary Databases

## Table of Contents

### [03 - The Document-oriented Database](03%20-%20The%20Document-oriented%20Database.md)
**MongoDB Development Environment Setup**

Introduction to document-oriented databases and MongoDB installation guide. Covers the fundamental concepts of NoSQL databases, their advantages and disadvantages, and step-by-step instructions for setting up a MongoDB development environment on Rocky Linux virtual machines.

**Key Topics:**
- Document-oriented database concepts and use cases
- MongoDB installation on Rocky Linux
- Basic server configuration and firewall setup
- MongoDB shell (mongosh) connection and usage
- Administrative tasks and log monitoring

### [04 - MongoDB Data Structures](04%20-%20MongoDB%20Data%20Structures.md)
**MongoDB Data Structures and CRUD Operations**

Comprehensive guide to MongoDB's document structure, data types, and fundamental CRUD operations. Explores ObjectId generation, query operators, projections, and cursor methods for effective data manipulation.

**Key Topics:**
- MongoDB document structure and ObjectId system
- CRUD operations (Create, Read, Update, Delete)
- Query operators (comparison, logical, array)
- Projections and cursor methods (sort, limit, skip)
- Working with arrays and nested documents
- Update operators and document modification

### [05 - MongoDB Aggregation](05%20-%20MongoDB%20Aggregation.md)
**MongoDB Aggregation Pipeline**

In-depth exploration of MongoDB's aggregation framework for data processing and analysis. Covers pipeline stages, data transformation, and complex query operations for advanced data manipulation.

**Key Topics:**
- Aggregation pipeline architecture and syntax
- Common pipeline stages ($match, $project, $group, $sort)
- Data transformation with $unwind and $addFields
- Grouping and accumulation operations
- Performance considerations and memory limits
- Practical examples and data analysis techniques

### [06 - MongoDB Sharding and Replication](06%20-%20MongoDB%20Sharding%20and%20Replication.md)
**MongoDB Sharding and Replication**

Advanced MongoDB concepts for building scalable and highly available database systems. Covers horizontal scaling through sharding and data redundancy through replication.

**Key Topics:**
- Sharded cluster architecture (shards, config servers, mongos)
- Shard key selection and chunk management
- Sharding strategies (range-based vs. hashed)
- Replica sets and replication benefits
- Combined sharded and replicated clusters
- Alternatives to sharding and operational considerations

### [06b - MongoDB Sharding and Replication on AWS](06b%20-%20MongoDB%20Sharding%20and%20Replication%20on%20AWS.md)
**MongoDB Deployment on AWS**

Practical guide for deploying MongoDB clusters on Amazon Web Services, covering cloud-specific considerations and best practices for production environments.

**Key Topics:**
- AWS deployment strategies for MongoDB
- Cloud infrastructure considerations
- Scaling and monitoring in AWS environments
- Security and backup strategies

### MongoDB Data Modeling Patterns

- **[7a - Inheritance Pattern](7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md)**
- **[7b - Computed and Approximation Pattern](7b%20-%20MongoDB%20Data%20Modeling%20Computed%20and%20Approximation%20Pattern.md)**
- **[7c - Reference Pattern](7c%20-%20MongoDB%20Data%20Modeling%20Reference%20Pattern.md)**
- **[7d - Schema Versioning](7d%20-%20MongoDB%20Data%20Modeling%20Schema%20Versioning%20copy.md)**
- **[7e - Single Collection](7e%20-%20MongoDB%20Data%20Modeling%20Single%20Collection.md)**
