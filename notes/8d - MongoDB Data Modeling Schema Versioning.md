# MongoDB Data Modeling: Schema Versioning Pattern

**Course:** CSCI 112 / 212 - Contemporary Databases
**Topic:** MongoDB Data Modeling - Schema Versioning Pattern

---

## Overview

In software development, change is inevitable. This is especially true for database schemas, which often need to evolve over time to accommodate new requirements, services, or data types. Traditional relational databases can make schema changes difficult and risky, often requiring downtime and complex migrations. MongoDB, with its flexible document model, offers a more graceful solution through the use of the **Schema Versioning Pattern**.

This pattern allows multiple versions of a document schema to coexist within the same collection, enabling applications to evolve without service interruptions or costly migrations.

All examples use **PyMongo** running on your laptop, connecting to `mongod` on a VM.

---

## Why Schema Versioning?

Updating a schema in a traditional tabular database typically involves:

- Stopping the application
- Migrating the database
- Restarting the application

This process introduces downtime and risk. If the migration fails, rolling back can be even more complex.

MongoDB's document model supports polymorphism, allowing documents with different structures to exist side-by-side in the same collection. This capability is the foundation of the Schema Versioning Pattern.

---

## How the Schema Versioning Pattern Works

The implementation is straightforward:

1. **Start with an initial schema.**
2. **When changes are needed, create a new schema version.**
3. **Add a `schema_version` field to the new documents.**
4. **In the application, use the `schema_version` field to determine how to handle each document.**

If a document does not have a `schema_version` field, it can be assumed to be version 1. Each subsequent schema version increments this value.

Depending on the use case, you may choose to:

- Update all documents to the new schema immediately
- Update documents as they are accessed
- Leave older documents as-is

The application should include logic to handle each supported schema version.

---

## Sample Use Case: Customer Contact Information

### Version 1: Original Schema

```json
{
    "_id": "<ObjectId>",
    "name": "Anakin Skywalker",
    "home": "503-555-0000",
    "work": "503-555-0010"
}
```

### Version 1.1: Added Mobile Number

```json
{
    "_id": "<ObjectId>",
    "name": "Darth Vader",
    "home": "503-555-0100",
    "work": "503-555-0110",
    "mobile": "503-555-0120"
}
```

### Version 2: Flexible Contact Methods Using Attribute Pattern

```json
{
    "_id": "<ObjectId>",
    "schema_version": "2",
    "name": "Anakin Skywalker (Retired)",
    "contact_method": [
        { "work": "503-555-0210" },
        { "mobile": "503-555-0220" },
        { "twitter": "@anakinskywalker" },
        { "skype": "AlwaysWithYou" }
    ]
}
```

In this version, we've combined the **Schema Versioning Pattern** with the **Attribute Pattern** to allow for a more flexible and future-proof data model.

---

## Benefits

- **No Downtime:** Schema changes can be deployed without taking the application offline.
- **Incremental Migration:** Documents can be updated gradually or not at all.
- **Flexible Application Logic:** Applications can handle multiple schema versions simultaneously.
- **Reduced Technical Debt:** Easier to manage and evolve the data model over time.
- **Pattern Composability:** Can be combined with other patterns like the Attribute Pattern for even more flexibility.

---

## Considerations

- **Indexing:** If fields move or change structure between versions, multiple indexes may be required during migration.
- **Application Complexity:** The application must include logic to handle different schema versions.
- **Data Consistency:** Ensure that all versions are supported and tested to avoid inconsistencies.

---

## PyMongo Setup

Activate your virtual environment and install PyMongo if you haven't already:

```bash
python -m venv .venv
source .venv/bin/activate          # Linux / macOS
pip install pymongo
```

Connect to your MongoDB instance:

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client    = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db        = client["crm"]
customers = db["customers"]
```

The `customers` variable is the **collection handle** used in every example below.

---

## Step-by-Step Example

### Step 1: Insert Version 1 Customers

These documents have no `schema_version` field — the application treats their absence as version 1:

```python
customers.insert_many([
    {
        "_id": 1,
        "name": "Anakin Skywalker",
        "home": "503-555-0000",
        "work": "503-555-0010"
    },
    {
        "_id": 2,
        "name": "Darth Vader",
        "home": "503-555-0100",
        "work": "503-555-0110",
        "mobile": "503-555-0120"
    }
])
```

### Step 2: Insert a Version 2 Customer

The new schema uses `schema_version: "2"` and a flexible `contact_method` array:

```python
customers.insert_one({
    "_id": 3,
    "schema_version": "2",
    "name": "Anakin Skywalker (Retired)",
    "contact_method": [
        {"work":    "503-555-0210"},
        {"mobile":  "503-555-0220"},
        {"twitter": "@anakinskywalker"},
        {"skype":   "AlwaysWithYou"}
    ]
})
```

### Step 3: Application-Level Schema Branching

The `handle_customer()` function branches on `schema_version` to print contact information regardless of which version the document is:

```python
def handle_customer(doc):
    version = doc.get("schema_version", "1")
    print(f"Customer: {doc['name']}  (schema v{version})")

    if version == "1":
        # V1 stores phone numbers as flat fields
        if "home" in doc:
            print(f"  home:   {doc['home']}")
        if "work" in doc:
            print(f"  work:   {doc['work']}")
        if "mobile" in doc:
            print(f"  mobile: {doc['mobile']}")
    elif version == "2":
        # V2 stores contact methods as an array of single-key dicts
        for entry in doc.get("contact_method", []):
            for method, value in entry.items():
                print(f"  {method}: {value}")
    else:
        print(f"  [unsupported schema version: {version}]")
```

### Step 4: Read All Customers Through the Handler

Both schema versions coexist in the same collection; `handle_customer()` normalizes the output:

```python
for doc in customers.find():
    handle_customer(doc)
    print()
```

Expected output:

```
Customer: Anakin Skywalker  (schema v1)
  home:   503-555-0000
  work:   503-555-0010

Customer: Darth Vader  (schema v1)
  home:   503-555-0100
  work:   503-555-0110
  mobile: 503-555-0120

Customer: Anakin Skywalker (Retired)  (schema v2)
  work:    503-555-0210
  mobile:  503-555-0220
  twitter: @anakinskywalker
  skype:   AlwaysWithYou
```

---

## Conclusion

The Schema Versioning Pattern is a powerful tool in MongoDB that allows developers to evolve their data models without downtime or complex migrations. By simply adding a `schema_version` field and updating application logic, teams can support multiple schema versions and reduce the risk and cost of change.

This pattern is especially useful in long-lived applications where data models are expected to evolve over time. When combined with other patterns like the Attribute Pattern, it provides a robust and scalable approach to data modeling in MongoDB.

---

## Additional Resources

- [PyMongo Documentation](https://pymongo.readthedocs.io/)
- [MongoDB Schema Design Best Practices](https://www.mongodb.com/docs/manual/core/data-model-design/)
- [MongoDB Attribute Pattern](https://www.mongodb.com/docs/manual/data-modeling/design-patterns/)

---

## Full Example

The script below is the entire walkthrough in one file — copy-paste it into a Python REPL or a `.py` file after editing the `VM_IP_ADDRESS` constant. It drops the collection, inserts two v1 customers and one v2 customer, then reads them all through `handle_customer()` to show both schema versions coexisting gracefully.

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client    = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db        = client["crm"]
customers = db["customers"]

# Reset for a clean run
customers.drop()

# 1. Insert v1 customers — no schema_version field; absence implies version 1
customers.insert_many([
    {
        "_id": 1,
        "name": "Anakin Skywalker",
        "home": "503-555-0000",
        "work": "503-555-0010"
    },
    {
        "_id": 2,
        "name": "Darth Vader",
        "home": "503-555-0100",
        "work": "503-555-0110",
        "mobile": "503-555-0120"
    }
])

# 2. Insert a v2 customer — uses schema_version: "2" and a flexible contact_method array
customers.insert_one({
    "_id": 3,
    "schema_version": "2",
    "name": "Anakin Skywalker (Retired)",
    "contact_method": [
        {"work":    "503-555-0210"},
        {"mobile":  "503-555-0220"},
        {"twitter": "@anakinskywalker"},
        {"skype":   "AlwaysWithYou"}
    ]
})

# 3. Application handler — branches on schema_version to normalize output
def handle_customer(doc):
    version = doc.get("schema_version", "1")
    print(f"Customer: {doc['name']}  (schema v{version})")

    if version == "1":
        if "home" in doc:
            print(f"  home:   {doc['home']}")
        if "work" in doc:
            print(f"  work:   {doc['work']}")
        if "mobile" in doc:
            print(f"  mobile: {doc['mobile']}")
    elif version == "2":
        for entry in doc.get("contact_method", []):
            for method, value in entry.items():
                print(f"  {method}: {value}")
    else:
        print(f"  [unsupported schema version: {version}]")

# 4. Read all customers — both versions coexist in the same collection
print("All customers:\n")
for doc in customers.find():
    handle_customer(doc)
    print()
```
