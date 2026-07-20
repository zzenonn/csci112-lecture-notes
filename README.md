# CSCI 112 - Contemporary Databases
**Ateneo de Manila University**

This repository contains materials for CSCI 112/212 - Contemporary Databases, a comprehensive course exploring NoSQL database systems and their applications in modern data management. The course covers various NoSQL database types including document-oriented, key-value, column-family, and graph databases, with hands-on experience in multiple database technologies.

## Repository Structure

- **[📊 Slides](slides/)** - Slidev presentation decks, one per lecture module
- **[📚 Notes](notes/)** - Lecture notes and theoretical materials covering NoSQL concepts and database technologies
- **[🧪 Labs](labs/)** - Laboratory exercises and practical assignments for hands-on experience
- **[📝 Problem Sets](problem_sets/)** - Graded problem sets organized by academic year

## Slides and Their Relationship to the Notes

The slides and notes are **companion materials** designed to be used together. Each
slide deck under `slides/` is numbered to match the corresponding note in `notes/`,
so the two trees line up module-for-module (e.g. `slides/10-dynamodb-data-modeling/`
pairs with `notes/10 - DynamoDB Data Modeling.md`).

The split between them is intentional:

- **Slides** carry the *pattern* — the talking points, small self-contained examples,
  and generalized command/structure templates a student should recall from lecture.
- **Notes** carry the *full instance* — complete sample documents, runnable scripts,
  end-to-end command output, and the detailed walkthroughs referenced from the slides.

Where a slide intentionally trims a listing, it links out to the matching note for the
full version. Each deck ends with a **Further reading** slide pointing at its note.

### Slide Modules

| # | Deck | Companion Note(s) |
|---|------|-------------------|
| 01 | [Databases in the Real World](slides/01-databases-real-world/slides.md) | *(slides only)* |
| 02 | [NoSQL: Not Only SQL](slides/02-nosql-intro/slides.md) | *(slides only)* |
| 03 | [The Document-oriented Database](slides/03-document-oriented-db/slides.md) | [03 - The Document-oriented Database](notes/03%20-%20The%20Document-oriented%20Database.md) |
| 04 | [MongoDB Data Structures](slides/04-mongodb-data-structures/slides.md) | [04 - MongoDB Data Structures](notes/04%20-%20MongoDB%20Data%20Structures.md) |
| 05 | [MongoDB Aggregation](slides/05-mongodb-aggregation/slides.md) | [05 - MongoDB Aggregation](notes/05%20-%20MongoDB%20Aggregation.md) |
| 06 | [MongoDB Sharding & Replication](slides/06-mongodb-sharding-replication/slides.md) | [06 - MongoDB Sharding and Replication](notes/06%20-%20MongoDB%20Sharding%20and%20Replication.md) |
| 07 | [MongoDB Indexing](slides/07-mongodb-indexing/slides.md) | [07 - MongoDB Indexing](notes/07%20-%20MongoDB%20Indexing.md) |
| 08a | [Modeling: Inheritance, Computed, Approximation](slides/08a-modeling-inheritance-computed-approximation/slides.md) | [8a - Inheritance](notes/8a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md), [8b - Computed & Approximation](notes/8b%20-%20MongoDB%20Data%20Modeling%20Computed%20and%20Approximation%20Pattern.md) |
| 08b | [Modeling: Reference, Schema Versioning, Single Collection](slides/08b-modeling-reference-versioning-single-collection/slides.md) | [8c - Reference](notes/8c%20-%20MongoDB%20Data%20Modeling%20Reference%20Pattern.md), [8d - Schema Versioning](notes/8d%20-%20MongoDB%20Data%20Modeling%20Schema%20Versioning.md), [8e - Single Collection](notes/8e%20-%20MongoDB%20Data%20Modeling%20Single%20Collection.md) |
| 08c | [Modeling: Subset Pattern](slides/08c-modeling-subset/slides.md) | [8f - Subset Pattern](notes/8f%20-%20MongoDB%20Data%20Modeling%20Subset%20Pattern.md) |
| 09 | [Introduction to DynamoDB](slides/09-dynamodb-intro/slides.md) | [09 - Introduction to DynamoDB](notes/09%20-%20Introduction%20to%20DynamoDB.md) |
| 10 | [DynamoDB Data Modeling](slides/10-dynamodb-data-modeling/slides.md) | [10 - DynamoDB Data Modeling](notes/10%20-%20DynamoDB%20Data%20Modeling.md) |
| 11 | [DynamoDB Design Patterns](slides/11-dynamodb-design-patterns/slides.md) | [11 - DynamoDB Design Patterns](notes/11%20-%20DynamoDB%20Design%20Patterns.md) |

### Running the Slides

The slides are a [Slidev](https://sli.dev/) monorepo managed with **pnpm workspaces**
(one self-contained deck per module). From the `slides/` directory, target a single
deck by its folder path:

```bash
cd slides
pnpm install                                       # once, sets up shared node_modules

pnpm --filter ./10-dynamodb-data-modeling dev      # live dev server (hot reload)
pnpm --filter ./10-dynamodb-data-modeling build    # static build → deck's dist/
pnpm --filter ./10-dynamodb-data-modeling export   # export to PDF
```

See [`slides/README.md`](slides/README.md) for full details — prerequisites, the deck ↔
package-name table, and a script to generate all deck PDFs at once.

## Getting Started

1. Follow along with the [Slides](slides/) during lecture for the core concepts
2. Read the [Notes](notes/) for full code listings, sample data, and command output
3. Practice with the [Labs](labs/) and [Problem Sets](problem_sets/) to reinforce your learning
4. Refer to official documentation for each database technology covered
