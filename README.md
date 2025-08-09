# CSCI 112 - Contemporary Databases
**Ateneo de Manila University**

This repository contains lecture notes and materials for CSCI 112/212 - Contemporary Databases, focusing on MongoDB and document-oriented database systems.

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

### [06 - MongoDB Sharding and Replication](06%20-%20MongoDB%20Sharding%20and%20Replication)
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

## Course Learning Objectives

By completing this course, students will be able to:

1. **Understand NoSQL Database Concepts**
   - Differentiate between relational and document-oriented databases
   - Identify appropriate use cases for MongoDB

2. **Master MongoDB Operations**
   - Perform CRUD operations efficiently
   - Design effective query strategies using various operators
   - Implement complex data transformations using aggregation pipelines

3. **Design Scalable Database Systems**
   - Implement sharding for horizontal scaling
   - Configure replication for high availability
   - Choose appropriate shard keys and replication strategies

4. **Deploy Production-Ready Systems**
   - Set up MongoDB in cloud environments
   - Implement security best practices
   - Monitor and maintain database performance

## Prerequisites

- Basic understanding of database concepts
- Familiarity with command-line interfaces
- Knowledge of JSON data format
- Basic understanding of virtualization (for VM setup)

## Getting Started

1. Start with [03 - The Document-oriented Database](03%20-%20The%20Document-oriented%20Database.md) to set up your development environment
2. Progress through the materials in numerical order
3. Practice the examples and exercises provided in each module
4. Refer to the official [MongoDB Documentation](https://www.mongodb.com/docs/manual/) for additional details

## Additional Resources

- [MongoDB University](https://university.mongodb.com/) - Free online courses
- [MongoDB Community Forums](https://www.mongodb.com/community/forums/) - Community support
- [MongoDB Compass](https://www.mongodb.com/products/compass) - GUI for MongoDB

## Course Information

- **Course Code:** CSCI 112 / 212
- **Institution:** Ateneo de Manila University
- **Focus:** Contemporary Databases with MongoDB
- **Level:** Undergraduate/Graduate