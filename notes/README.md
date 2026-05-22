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
- **[7f - Subset Pattern](7f%20-%20MongoDB%20Data%20Modeling%20Subset%20Pattern.md)**

### [08 - Introduction to DynamoDB](08%20-%20Introduction%20to%20DynamoDB.md)
**Key-Value and Document Databases with Amazon DynamoDB**

Introduction to Amazon DynamoDB as a serverless, fully managed NoSQL database. Covers core concepts including tables, items, primary keys, secondary indexes, and billing modes. Includes hands-on setup with Python and boto3.

**Key Topics:**
- Serverless and fully managed database concepts
- Key-value and document store data models
- Partition keys, sort keys, and composite primary keys
- Local Secondary Indexes (LSI)
- Environment setup with boto3 (AWS SDK for Python)
- Basic CRUD operations and query patterns
- DynamoDB vs. MongoDB comparison

### [09 - DynamoDB Data Modeling](09%20-%20DynamoDB%20Data%20Modeling.md)
**DynamoDB Data Modeling -- Single-Table Design and Multi-Attribute Keys**

Advanced DynamoDB data modeling techniques for storing multiple entity types in a single table. Covers Global Secondary Indexes, the inverted index pattern, and the multi-attribute keys feature that eliminates synthetic key concatenation.

**Key Topics:**
- Single-table design pattern and motivation
- Comparison with MongoDB Single Collection Pattern
- Global Secondary Indexes (GSI) and the inverted index pattern
- Multi-attribute keys for GSI partition and sort keys
- Walkthrough: users, orders, and line items in one table
- Access-pattern-driven data modeling principles

### [10 - DynamoDB Design Patterns](10%20-%20DynamoDB%20Design%20Patterns.md)
**DynamoDB Design Patterns -- From Single-Table to Aggregate-Oriented Design**

Revisits the Module 09 single-table design and shows how to improve it using aggregate-oriented design principles. Covers natural keys, item collections, identifying relationships, aggregate boundary decisions, and when to use multiple tables.

**Key Topics:**
- Problems with generic "everything-table" designs
- Natural keys over generic identifiers
- Item collections and identifying relationships
- Aggregate-oriented design with access correlation analysis
- When to use multiple tables vs. item collections
- Short-circuit denormalization, sparse GSIs, hierarchical sort keys
- Side-by-side comparison of Module 09 vs. improved design
