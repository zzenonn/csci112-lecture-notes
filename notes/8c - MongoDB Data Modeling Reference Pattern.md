# MongoDB Data Modeling: Reference Pattern

**Course:** CSCI 112 / 212 - Contemporary Databases
**Topic:** Data Modeling with the Extended Reference Pattern in MongoDB

---

## Overview

In MongoDB, data modeling is a critical aspect of designing performant and scalable applications. While MongoDB is a NoSQL database and does not support traditional JOIN operations like relational databases, there are still ways to model relationships between data.

One such method is the **Reference Pattern**, where documents reference other documents in separate collections. This pattern is useful when entities are distinct and should be stored independently, such as customers, orders, and inventory in an e-commerce application.

However, when frequent JOIN-like operations are required to retrieve commonly accessed data, the **Extended Reference Pattern** becomes particularly useful.

All examples use **PyMongo** running on your laptop, connecting to `mongod` on a VM.

---

## The Problem with Frequent JOINs

Consider an e-commerce application with the following collections:

- **Customer**
- **Order**
- **Inventory**

Each of these collections represents a separate logical entity. For example:

### Customer Collection

```json
{
    "_id": 123,
    "name": "Katrina Pope",
    "street": "123 Main St",
    "city": "Somewhere",
    "country": "Someplace"
}
```

### Order Collection

```json
{
    "_id": "507f1f77bcf86cd799439011",
    "date": "2019-02-18",
    "customer_id": 123,
    "order": [
        {
            "product": "widget",
            "qty": 5,
            "cost": {
                "value": "11.99",
                "currency": "USD"
            }
        }
    ]
}
```

### Inventory Collection

```json
{
    "_id": "507f1f77bcf86cd111111111",
    "name": "widget",
    "cost": {
        "value": "11.99",
        "currency": "USD"
    },
    "on_hand": 98325
}
```

In this model, retrieving a complete order with customer and product details requires multiple queries or application-side JOINs. This can become inefficient, especially when these operations are frequent.

---

## The Extended Reference Pattern

The **Extended Reference Pattern** addresses this inefficiency by embedding only the most frequently accessed fields from a referenced document directly into the parent document. This avoids full duplication of data and reduces the need for repeated JOIN-like operations.

### When to Use

- When you frequently access a subset of fields from a referenced document.
- When full embedding would result in excessive duplication.
- When performance is a higher priority than strict normalization.

---

## Example: Applying the Extended Reference Pattern

### Original Customer Collection

```json
{
    "_id": 123,
    "name": "Katrina Pope",
    "street": "123 Main St",
    "city": "Somewhere",
    "country": "Someplace",
    "date_of_birth": "1992-11-03",
    "social_handles": [
        {
            "twitter": "@somethingamazing123"
        }
    ]
}
```

### Modified Order Collection (Using Extended Reference)

```json
{
    "_id": "507f1f77bcf86cd799439011",
    "date": "2019-02-18",
    "customer_id": 123,
    "shipping_address": {
        "name": "Katrina Pope",
        "street": "123 Main St",
        "city": "Somewhere",
        "country": "Someplace"
    },
    "order": [
        {
            "product": "widget",
            "qty": 5,
            "cost": {
                "value": "11.99",
                "currency": "USD"
            }
        }
    ]
}
```

In this example, only the customer's name and address are embedded in the order document. These are the fields most commonly accessed when viewing an order, such as for shipping or display purposes. Other customer details (e.g., date of birth, social handles) remain in the customer collection and can be queried separately if needed.

---

## Benefits

- **Improved Read Performance**: Reduces the need for multiple queries or application-side JOINs.
- **Reduced Duplication**: Only frequently accessed fields are duplicated.
- **Simplified Queries**: Common queries become faster and easier to write.

---

## Trade-offs

- **Data Redundancy**: Some duplication still exists, which may require additional logic to keep data in sync if it changes.
- **Write Complexity**: Updates to embedded fields may require updating multiple documents.

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
db        = client["ecommerce"]
customers  = db["customers"]
orders     = db["orders"]
inventory  = db["inventory"]
```

The `customers`, `orders`, and `inventory` variables are the **collection handles** used in every example below.

---

## Step-by-Step Example

### Step 1: Insert Sample Documents

Insert a customer, an inventory item, and a basic order that references the customer by ID:

```python
customers.insert_one({
    "_id": 123,
    "name": "Katrina Pope",
    "street": "123 Main St",
    "city": "Somewhere",
    "country": "Someplace",
    "date_of_birth": "1992-11-03",
    "social_handles": [{"twitter": "@somethingamazing123"}]
})

inventory.insert_one({
    "_id": 1001,
    "name": "widget",
    "cost": {"value": "11.99", "currency": "USD"},
    "on_hand": 98325
})

orders.insert_one({
    "_id": 1,
    "date": "2019-02-18",
    "customer_id": 123,
    "order": [
        {
            "product": "widget",
            "qty": 5,
            "cost": {"value": "11.99", "currency": "USD"}
        }
    ]
})
```

### Step 2: Demonstrate the JOIN Problem

Reading the order does not include the customer's shipping address — a second query is required:

```python
order = orders.find_one({"_id": 1})
print("Order (no address):", order)

# Need a second round-trip to get the shipping address
customer = customers.find_one({"_id": order["customer_id"]})
print("Customer address:", customer["street"], customer["city"], customer["country"])
```

### Step 3: Apply the Extended Reference Pattern

Look up the customer's shipping fields and embed them directly into the order:

```python
customer = customers.find_one({"_id": 123})

shipping_address = {
    "name":    customer["name"],
    "street":  customer["street"],
    "city":    customer["city"],
    "country": customer["country"]
}

orders.update_one(
    {"_id": 1},
    {"$set": {"shipping_address": shipping_address}}
)
```

### Step 4: Read the Order — O(1), No Join Required

```python
order = orders.find_one({"_id": 1})
print("Order with embedded shipping address:")
print(order)
```

The full shipping address is now available in a single document read, with no additional query to the `customers` collection.

---

## Summary

The Extended Reference Pattern is a practical compromise between full embedding and full referencing. It allows developers to optimize for read performance while minimizing unnecessary data duplication. When designing your MongoDB schema, consider how your application accesses data and whether this pattern can help reduce complexity and improve efficiency.

---

## Additional Resources

- [MongoDB Data Modeling Documentation](https://www.mongodb.com/docs/manual/core/data-modeling-introduction/)
- [PyMongo Documentation](https://pymongo.readthedocs.io/)
- [MongoDB Schema Design Best Practices](https://www.mongodb.com/docs/manual/core/data-model-design/)

---

## Full Example

The script below is the entire walkthrough in one file — copy-paste it into a Python REPL or a `.py` file after editing the `VM_IP_ADDRESS` constant. It drops the three collections, inserts sample data, demonstrates the two-query problem, applies the Extended Reference Pattern, and reads the order in a single O(1) lookup.

```python
from pymongo import MongoClient

VM_IP_ADDRESS = "192.168.1.100"   # replace with your VM's IP

client    = MongoClient(f"mongodb://{VM_IP_ADDRESS}:27017/")
db        = client["ecommerce"]
customers  = db["customers"]
orders     = db["orders"]
inventory  = db["inventory"]

# Reset for a clean run
customers.drop()
orders.drop()
inventory.drop()

# 1. Insert sample documents
customers.insert_one({
    "_id": 123,
    "name": "Katrina Pope",
    "street": "123 Main St",
    "city": "Somewhere",
    "country": "Someplace",
    "date_of_birth": "1992-11-03",
    "social_handles": [{"twitter": "@somethingamazing123"}]
})

inventory.insert_one({
    "_id": 1001,
    "name": "widget",
    "cost": {"value": "11.99", "currency": "USD"},
    "on_hand": 98325
})

orders.insert_one({
    "_id": 1,
    "date": "2019-02-18",
    "customer_id": 123,
    "order": [
        {
            "product": "widget",
            "qty": 5,
            "cost": {"value": "11.99", "currency": "USD"}
        }
    ]
})

# 2. Demonstrate the problem: reading the order requires a second query for the address
order = orders.find_one({"_id": 1})
print("Order without embedded address:", order)

customer = customers.find_one({"_id": order["customer_id"]})
print("Had to fetch customer separately:", customer["name"], customer["city"])

# 3. Apply the Extended Reference Pattern:
#    embed just the shipping fields into the order document
shipping_address = {
    "name":    customer["name"],
    "street":  customer["street"],
    "city":    customer["city"],
    "country": customer["country"]
}

orders.update_one(
    {"_id": 1},
    {"$set": {"shipping_address": shipping_address}}
)

# 4. Read the order — O(1), shipping address is embedded, no second query needed
order = orders.find_one({"_id": 1})
print("\nOrder with embedded shipping address:")
print(order)
# → {"_id": 1, "date": "2019-02-18", "customer_id": 123,
#    "shipping_address": {"name": "Katrina Pope", "street": "123 Main St",
#                         "city": "Somewhere", "country": "Someplace"},
#    "order": [{"product": "widget", "qty": 5, ...}]}
```
