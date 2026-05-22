---
theme: discs
title: "The Document-Oriented Database"
highlighter: shiki
layout: cover
codeCopy: true
---

The Document-Oriented Database

::author-info::
Miguel Zenon Nicanor Lerias Saavedra, PhD

::subject-info::
CSCI 112 / 212
Contemporary Databases

---

# Objectives

1. Understand the structure of **document-oriented databases**
2. Install and configure **MongoDB** on a virtual machine
3. Perform basic **administrative tasks** using the MongoDB shell

---
layout: section
---

# What is a Document-Oriented Database?

---

# Document-Oriented Databases

Stores data as **documents** — key-value pairs in a JSON-like format

<div style="display:flex; gap:2rem; margin-top:1rem;">
  <div style="flex:1;">
    <div style="font-weight:700; margin-bottom:0.5rem;">Advantages</div>
    <ul style="font-size:0.95rem; line-height:1.8;">
      <li><strong>Better Performance</strong> depending on use case</li>
      <li><strong>Horizontal Scalability</strong></li>
      <li><strong>Dynamic and Flexible Schema</strong></li>
    </ul>
  </div>
  <div style="flex:1;">
    <div style="font-weight:700; margin-bottom:0.5rem;">Disadvantages</div>
    <ul style="font-size:0.95rem; line-height:1.8;">
      <li><strong>Limited Built-in Functionality</strong></li>
      <li><strong>More Application Logic Required</strong></li>
      <li><strong>Lack of Standardization</strong></li>
    </ul>
  </div>
</div>

---

# Databases

<div style="display:flex; gap:3rem; margin-top:1rem; align-items:center;">
  <div style="flex:1;">
    <ul style="font-size:1rem; line-height:2;">
      <li>Top-level containers for data</li>
      <li>Commands that are run on a database:
        <ul style="margin-top:0.25rem; line-height:1.8;">
          <li>User creation</li>
          <li>Configuration</li>
          <li>Creating collections</li>
        </ul>
      </li>
    </ul>
  </div>
  <div style="flex:1; display:flex; flex-direction:column; align-items:center;">
    <svg width="180" height="200" viewBox="0 0 180 200" xmlns="http://www.w3.org/2000/svg">
      <!-- cylinder body -->
      <rect x="20" y="40" width="140" height="130" fill="#e5e7eb" stroke="#404040" stroke-width="2.5"/>
      <!-- top ellipse -->
      <ellipse cx="90" cy="40" rx="70" ry="22" fill="#d1d5db" stroke="#404040" stroke-width="2.5"/>
      <!-- bottom ellipse -->
      <ellipse cx="90" cy="170" rx="70" ry="22" fill="#d1d5db" stroke="#404040" stroke-width="2.5"/>
      <!-- horizontal dividers -->
      <line x1="20" y1="90" x2="160" y2="90" stroke="#404040" stroke-width="2"/>
      <line x1="20" y1="130" x2="160" y2="130" stroke="#404040" stroke-width="2"/>
      <!-- partial ellipses at dividers -->
      <path d="M20,90 Q90,104 160,90" fill="none" stroke="#404040" stroke-width="2"/>
      <path d="M20,130 Q90,144 160,130" fill="none" stroke="#404040" stroke-width="2"/>
    </svg>
  </div>
</div>

---

# Collections

<div style="display:flex; gap:2rem; margin-top:0.75rem; align-items:flex-start;">
  <div style="flex:0 0 auto;">
    <ul style="font-size:0.95rem; line-height:1.9; margin:0;">
      <li>Equivalent to <strong>tables</strong> in SQL</li>
      <li>Stores multiple documents</li>
      <li>Implicitly created on insert</li>
      <li>Example:
        <ul style="line-height:1.7;">
          <li>Database: <code>products</code></li>
          <li>Collections: <code>books</code>, <code>movies</code>, <code>music</code></li>
        </ul>
      </li>
    </ul>
  </div>
  <div style="flex:1; display:flex; flex-direction:row; gap:0.5rem; font-size:0.75rem; font-family:monospace; line-height:1.5; min-width:0;">
    <div style="display:flex; flex-direction:column; gap:0.3rem; width:100%;">
      <div style="font-size:0.7rem; color:#888; font-family:sans-serif; font-weight:600;">books</div>
      <div style="background:#f8fafc; border:1.5px solid #cbd5e1; border-radius:6px; padding:0.4rem 0.6rem;">
        { "_id": ObjectId("…a1"),<br>&nbsp;&nbsp;"title": "Dune",<br>&nbsp;&nbsp;"author": "Frank Herbert", "year": 1965 }
      </div>
      <div style="background:#f8fafc; border:1.5px solid #cbd5e1; border-radius:6px; padding:0.4rem 0.6rem;">
        { "_id": ObjectId("…a2"),<br>&nbsp;&nbsp;"title": "Foundation",<br>&nbsp;&nbsp;"author": "Isaac Asimov", "year": 1951 }
      </div>
      <div style="background:#f8fafc; border:1.5px solid #cbd5e1; border-radius:6px; padding:0.4rem 0.6rem;">
        { "_id": ObjectId("…a3"),<br>&nbsp;&nbsp;"title": "Neuromancer",<br>&nbsp;&nbsp;"author": "William Gibson", "year": 1984 }
      </div>
    </div>
  </div>
</div>

---

# BSON

MongoDB stores data in **BSON** (Binary JSON) — optimized for speed and space

<div style="display:flex; align-items:center; justify-content:center; gap:1.5rem; margin-top:2rem;">
  <div style="text-align:center;">
    <div style="padding:0.6rem 1.2rem; background:#fef9c3; border:2px solid #ca8a04; border-radius:6px; font-weight:700; font-size:0.95rem;">BSON</div>
    <div style="font-size:0.75rem; color:#888; margin-top:0.3rem;">/var/lib/mongo/</div>
    <div style="font-size:0.7rem; color:#aaa;">Do not edit manually</div>
  </div>
  <div style="display:flex; flex-direction:column; align-items:center; gap:0.3rem;">
    <div style="font-size:0.8rem; color:#666;">Driver converts</div>
    <div style="font-size:1.25rem; color:#aaa;">⇄</div>
    <div style="font-size:0.8rem; color:#666;">automatically</div>
  </div>
  <div style="text-align:center;">
    <div style="padding:0.6rem 1.2rem; background:#dcfce7; border:2px solid #16a34a; border-radius:6px; font-weight:700; font-size:0.95rem;">JSON</div>
    <div style="font-size:0.75rem; color:#888; margin-top:0.3rem;">Client Application</div>
  </div>
</div>

<div style="text-align:center; margin-top:1.5rem; font-size:0.85rem; color:#666;">
  BSON supports additional types not in JSON: <code>Date</code>, <code>ObjectId</code>, <code>BinData</code>, <code>Decimal128</code>
</div>

---
layout: section
---

# Installation

Setting up MongoDB on Rocky Linux

---

# Set Up a Virtual Machine

<div style="display:flex; gap:2rem; margin-top:1rem;">
  <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:6px; padding:1rem;">
    <div style="font-weight:700; margin-bottom:0.5rem;">Windows / Linux</div>
    <p style="font-size:0.9rem; margin:0 0 0.5rem;">Install <strong>VMware Workstation Player</strong></p>
    <p style="font-size:0.9rem; margin:0;">Download <strong>Rocky Linux</strong> ISO and install as a VM</p>
  </div>
  <div style="flex:1; border:1.5px solid #cbd5e1; border-radius:6px; padding:1rem;">
    <div style="font-weight:700; margin-bottom:0.5rem;">Mac</div>
    <p style="font-size:0.9rem; margin:0 0 0.5rem;">Install <strong>UTM</strong> (self-supported)</p>
    <p style="font-size:0.9rem; margin:0;">Download <strong>Rocky Linux</strong> ISO and install as a VM</p>
  </div>
</div>

<div style="margin-top:1.25rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  After the VM is running, enable SSH so you can connect from your host machine:
  <pre style="margin:0.5rem 0 0; font-size:0.85rem; background:transparent; padding:0;">sudo systemctl start sshd
sudo systemctl enable sshd</pre>
</div>

<div style="margin-top:0.75rem; font-size:0.85rem; color:#666;">
  Full setup instructions: <a href="https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-red-hat/">MongoDB Docs — Install on Red Hat</a> · <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/03%20-%20The%20Document-oriented%20Database.md">notes/03 — The Document-oriented Database</a>
</div>

---

# Install MongoDB on Rocky Linux

Follow the official guide for Red Hat-based systems, then configure for remote access:

```bash
# Allow connections from all interfaces
sudo nano /etc/mongod.conf
# Set: bindIp: 0.0.0.0

# Open firewall port
sudo firewall-cmd --permanent --add-port=27017/tcp
sudo firewall-cmd --reload

# Start and enable on boot
sudo systemctl restart mongod
sudo systemctl enable mongod
```

<div style="margin-top:0.75rem; font-size:0.85rem; color:#666;">
  Full step-by-step instructions: <a href="https://github.com/zzenonn/csci112-lecture-notes/blob/main/notes/03%20-%20The%20Document-oriented%20Database.md">notes/03 — The Document-oriented Database</a>
</div>

---
layout: section
---

# Basic Administrative Tasks

---

# Service Management

```bash
# Check status
sudo systemctl status mongod.service

# Start and enable on boot
sudo systemctl start mongod
sudo systemctl enable mongod
```

<div style="margin-top:1rem; background:#f0f9ff; border-left:4px solid #00b0f0; padding:0.75rem 1rem; border-radius:0 6px 6px 0; font-size:0.9rem;">
  Logs are stored in <code>/var/log/mongodb/</code>
  <pre style="margin:0.5rem 0 0; font-size:0.85rem; background:transparent; padding:0;">sudo tail -f /var/log/mongodb/mongod.log</pre>
</div>

---

# Connecting with mongosh

```bash
# Connect by host IP
mongosh --host <ip_address>

# Connect to a specific database
mongosh --host <ip_address> <database_name>

# Using connection string format
mongosh "mongodb://<ip_address>/<database_name>"
```

Once connected, switch databases inside the shell:

```javascript
use <database_name>
```

---

# Common Shell Commands

```javascript
// List all databases
show dbs

// List collections in current database
show collections
```

---
layout: end
---

# Thank You
