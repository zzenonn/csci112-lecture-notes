# MongoDB Development Environment Setup

**Course:** CSCI 112 / 212 - Contemporary Databases  
**Topic:** Document-Oriented Databases with MongoDB (Setup)

---

## Overview

This repository contains instructions for setting up a MongoDB development environment for coursework in CSCI 112 / 212. MongoDB is a document-oriented NoSQL database that stores data in flexible, JSON-like documents. This guide will walk you through installing MongoDB on a virtual machine, configuring the server, and performing basic administrative tasks.

---

## Objectives

- Understand the structure and use cases of document-oriented databases
- Install and configure MongoDB on a virtual machine
- Perform basic administrative tasks using the MongoDB shell
- (Optional) Install and use MongoDB Compass for GUI-based interactions

---

## What is a Document-Oriented Database?

Document-oriented databases store data as documents, typically in a format similar to JSON. In MongoDB, these documents are stored in collections, which are grouped into databases.

### Advantages

- **Better Performance** (depending on use case)
- **Horizontal Scalability**
- **Dynamic and Flexible Schema**

### Disadvantages

- **Limited Built-in Functionality** (compared to relational databases)
- **More Application Logic Required**
- **Lack of Standardization** (compared to SQL)

### When to Use

- When dealing with highly variable or unstructured data
- When requiring high availability and scalability across distributed systems

---

## Installation Instructions

### 1. Set Up a Virtual Machine

#### For Windows/Linux Users:
- Install **VMware Workstation Player**:  
  https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion

- Download and install **Rocky Linux**:  
  https://rockylinux.org/download

#### For Mac Users:
- Use **UTM** (self-supported):  
  https://mac.getutm.app/

---

### 2. Initial VM Configuration

#### Start and Enable SSH Service

SSH allows you to connect to your VM remotely from your host machine:

```bash
sudo systemctl start sshd
sudo systemctl enable sshd
```

#### Get IP Address Information

To find your VM's IP address for remote connections:

```bash
ip a
```

Look for the IP address under your network interface (usually `eth0` or `enp0s3`). You'll need this IP address to connect via SSH.

#### Connect to VM via SSH

From your host machine (Windows/Mac/Linux), you can now connect to your VM:

```bash
ssh username@<vm_ip_address>
```

Replace `username` with your VM username and `<vm_ip_address>` with the IP address you found above.

#### Install and Configure Nano Text Editor

Install nano for easier file editing:

```bash
sudo dnf install nano -y
```

**Basic Nano Usage:**
- Open a file: `nano filename`
- Save file: `Ctrl + O`, then press Enter
- Exit nano: `Ctrl + X`
- Cut line: `Ctrl + K`
- Paste line: `Ctrl + U`
- Search: `Ctrl + W`
- Show line numbers: `Ctrl + C` (while in nano)

### 3. Install MongoDB on Rocky Linux

Follow the official MongoDB installation guide for Red Hat-based systems:  
https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-red-hat/

After installation, update the MongoDB configuration file to allow connections from all interfaces:

```bash
sudo nano /etc/mongod.conf
```

Find the `bindIp` setting and change it to:

```yaml
bindIp: 0.0.0.0
```

Configure the firewall to allow MongoDB traffic:

```bash
sudo firewall-cmd --permanent --add-port=27017/tcp
sudo firewall-cmd --reload
```

Then restart and enable the MongoDB service:

```bash
sudo systemctl restart mongod
sudo systemctl enable mongod
```

**Note:** The `enable` command ensures MongoDB starts automatically when the system boots.

---

## MongoDB Concepts

### Databases

- Top-level containers for data
- Each MongoDB instance can host multiple databases
- Each database can contain multiple collections

### Collections

- Equivalent to tables in relational databases
- Implicitly created when documents are inserted
- Example:
  - Database: `products`
  - Collections: `books`, `movies`, `music`

### BSON

- MongoDB stores data in BSON (Binary JSON)
- BSON is optimized for speed and space
- Stored in `/var/lib/mongo/` (Do not edit these files manually)

---

## Basic Administrative Tasks

### Check MongoDB Service Status

```bash
sudo systemctl status mongod.service
```

### Start and Enable MongoDB Service

If MongoDB is not running, start it and enable it to start automatically on boot:

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
```

### View Service Logs

Logs are stored in `/var/log/mongodb/`. Use tools like `tail`, `less`, or `grep` to view them:

```bash
sudo tail -f /var/log/mongodb/mongod.log
```

Logs include:

- Service start/stop events
- Authentication attempts
- Command executions
- Errors and warnings
- Performance metrics

### Connect to MongoDB Server Using `mongosh`

```bash
mongosh --host <ip_address>
```

Or using connection string format:

```bash
mongosh "mongodb://<ip_address>/"
```

To connect to a specific database:

```bash
mongosh --host <ip_address> <database_name>
```

Or:

```bash
mongosh "mongodb://<ip_address>/<database_name>"
```

### Switch Databases in Mongo Shell

```bash
use <database_name>
```

---

## Common Administrative Functions

### Create a New User

### List Databases

```javascript
show dbs
```

### List Collections in a Database

```javascript
show collections
```

### Insert a Document

```javascript
db.users.insertOne({ name: "Alice", age: 30 })
```

### Query Documents

```javascript
db.users.find({ age: { $gt: 25 } })
```

### Update a Document

```javascript
db.users.updateOne({ name: "Alice" }, { $set: { age: 31 } })
```

### Delete a Document

```javascript
db.users.deleteOne({ name: "Alice" })
```

---

## Optional: Install MongoDB Compass

MongoDB Compass is a GUI for interacting with MongoDB databases. It allows you to visually explore your data, run queries, and manage indexes.

Installation Guide:  
https://www.mongodb.com/docs/compass/install/

---

## Additional Resources

- MongoDB Manual: https://www.mongodb.com/docs/manual/
- MongoDB Shell (mongosh): https://www.mongodb.com/docs/mongodb-shell/
- BSON Specification: https://bsonspec.org/

---

## Notes

- Do not manually edit files in `/var/lib/mongo/`
- MongoDB is schema-less, but you should still apply consistent data modeling practices
- Always secure your MongoDB instance before deploying to production (authentication, IP whitelisting, etc.)


