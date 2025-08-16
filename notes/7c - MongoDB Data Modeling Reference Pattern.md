# MongoDB Data Modeling: Reference Pattern

**Course:** CSCI 122 / 212 - Contemporary Databases  
**Topic:** Data Modeling with the Extended Reference Pattern in MongoDB

---

## Overview

In MongoDB, data modeling is a critical aspect of designing performant and scalable applications. While MongoDB is a NoSQL database and does not support traditional JOIN operations like relational databases, there are still ways to model relationships between data.

One such method is the **Reference Pattern**, where documents reference other documents in separate collections. This pattern is useful when entities are distinct and should be stored independently, such as customers, orders, and inventory in an e-commerce application.

However, when frequent JOIN-like operations are required to retrieve commonly accessed data, the **Extended Reference Pattern** becomes particularly useful.

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
    "country": "Someplace",
    ...
}
```

### Order Collection

```json
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "date": ISODate("2019-02-18"),
    "customer_id": 123,
    "order": [
        {
            "product": "widget",
            "qty": 5,
            "cost": {
                "value": NumberDecimal("11.99"),
                "currency": "USD"
            }
        }
    ]
}
```

### Inventory Collection

```json
{
    "_id": ObjectId("507f1f77bcf86cd111111111"),
    "name": "widget",
    "cost": {
        "value": NumberDecimal("11.99"),
        "currency": "USD"
    },
    "on_hand": 98325,
    ...
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
    "date_of_birth": ISODate("1992-11-03"),
    "social_handles": [
        {
            "twitter": "@somethingamazing123"
        }
    ]
    ...
}
```

### Modified Order Collection (Using Extended Reference)

```json
{
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "date": ISODate("2019-02-18"),
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
                "value": NumberDecimal("11.99"),
                "currency": "USD"
            }
        }
    ],
    ...
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

## Summary

The Extended Reference Pattern is a practical compromise between full embedding and full referencing. It allows developers to optimize for read performance while minimizing unnecessary data duplication. When designing your MongoDB schema, consider how your application accesses data and whether this pattern can help reduce complexity and improve efficiency.

---

For more information, refer to the [MongoDB Data Modeling Documentation](https://www.mongodb.com/docs/manual/core/data-modeling-introduction/).
