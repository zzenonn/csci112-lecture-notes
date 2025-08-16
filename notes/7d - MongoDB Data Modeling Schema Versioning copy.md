# MongoDB Data Modeling: Schema Versioning Pattern

**Course:** CSCI 122 / 212 - Contemporary Databases  
**Topic:** MongoDB Data Modeling - Schema Versioning Pattern

---

## Overview

In software development, change is inevitable. This is especially true for database schemas, which often need to evolve over time to accommodate new requirements, services, or data types. Traditional relational databases can make schema changes difficult and risky, often requiring downtime and complex migrations. MongoDB, with its flexible document model, offers a more graceful solution through the use of the **Schema Versioning Pattern**.

This pattern allows multiple versions of a document schema to coexist within the same collection, enabling applications to evolve without service interruptions or costly migrations.

---

## Why Schema Versioning?

Updating a schema in a traditional tabular database typically involves:

- Stopping the application
- Migrating the database
- Restarting the application

This process introduces downtime and risk. If the migration fails, rolling back can be even more complex.

MongoDB’s document model supports polymorphism, allowing documents with different structures to exist side-by-side in the same collection. This capability is the foundation of the Schema Versioning Pattern.

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

In this version, we’ve combined the **Schema Versioning Pattern** with the **Attribute Pattern** to allow for a more flexible and future-proof data model.

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

## Conclusion

The Schema Versioning Pattern is a powerful tool in MongoDB that allows developers to evolve their data models without downtime or complex migrations. By simply adding a `schema_version` field and updating application logic, teams can support multiple schema versions and reduce the risk and cost of change.

This pattern is especially useful in long-lived applications where data models are expected to evolve over time. When combined with other patterns like the Attribute Pattern, it provides a robust and scalable approach to data modeling in MongoDB.
