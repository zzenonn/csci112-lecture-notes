# Final Project: NoSQL Data Modeling and Use Cases

**CSCI 112 / 212 — Contemporary Databases**

In a group, pick a NoSQL database and a real use case, then design the data model
for it. You will:

1. Submit a short **proposal** (see [Proposal (Submit This First!)](#proposal--submit-this-first)).
2. Explain **how the database works**: its data organization, query mechanisms,
   capabilities, limitations, and production behavior.
3. Run a **data modeling session**: state your access patterns, draw an ERD, design
   the model, and justify every decision against those access patterns.
4. **Implement** the model and demonstrate create, read, update, and delete
   operations, along with the queries your use case needs.
5. Present all of the above to the class.

The project uses four criteria, each scored **0–4**, for a total of **16 points**.
See [Grading](#grading).

---

## Table of Contents

- [Learning Goals](#learning-goals)
- [Choosing a Database](#choosing-a-database)
- [Proposal — Submit This First](#proposal--submit-this-first)
- [The Data Modeling Session (Main Deliverable)](#the-data-modeling-session-main-deliverable)
- [The Database Technology](#the-database-technology)
- [The Demo](#the-demo)
- [Presentation Format](#presentation-format)
- [Grading](#grading)
- [Tips](#tips)
- [Academic Honesty and GenAI](#academic-honesty-and-genai)

---

## Learning Goals

By the end of this project you should be able to:

- Explain the database's data organization, read/write mechanisms, capabilities,
  and limitations, including its behavior in production.
- Take a real-world use case and derive its **access patterns** before writing any
  schema.
- Justify a data model in terms of those access patterns.
- Explain the trade-offs your model makes: what it makes fast, what it makes slow, etc.
- Demonstrate CRUD operations and application queries on the implemented model.

---

## Choosing a Database

Pick any NoSQL database **except the ones we cover in class** (**no MongoDB and
no DynamoDB**). Identify its family and the modeling decisions your use case
requires:

| Family | Examples | Modeling centers on |
|---|---|---|
| Document | Firestore, RavenDB | Embedding vs. referencing, document shape per access pattern |
| Key-value | Redis, Riak, Memcached | Key design, value structure, what you denormalize into the value |
| Wide-column | Cassandra, ScyllaDB, HBase | Partition and clustering/sort keys, query-driven table design |
| Graph | Neo4j, Amazon Neptune | Nodes, relationships, and traversals as first-class citizens |
| Search / analytics | OpenSearch, Elasticsearch | Index mappings, field types and analyzers, denormalizing for query-time speed |

---

## Proposal (Submit This First!)

Post this to the **Project Proposal** discussion on Canvas. One or
two paragraphs plus the bullets below. This is a checkpoint so I can steer you
before you spend too much time on the option.

1. **Group number and members.**
2. **Database** and its family from the table above.
3. **Use case** in one or two sentences. What is the application? Who uses it?
4. **Three to five access patterns** you already expect, in plain language
   (for example, "show a user their orders from the last 30 days, newest first").
5. **Why this database** for this use case, in one sentence.

I will reply on the thread to approve or ask you to narrow the scope. Do not start
building the model until the proposal is approved.

---

## The Data Modeling Session (Main Deliverable)

Use the same application example throughout the following steps. Show the
requirements, the model, and the implementation so they can be compared.

### 1. Use case and requirements

State the application concretely. Describe the entities, the volumes you expect
(rough orders of magnitude are fine), and the read/write mix. A model for a
write-heavy event log looks is every different from a model for a read-heavy catalog.

### 2. Access patterns

List every access pattern the application needs, as specific queries. This is the
single most important artifact in the project. For each one, note:

- the operation and its inputs, including identifiers and filters,
- the expected result shape and ordering, where relevant,
- its frequency and priority relative to the other operations.

Cover the main reads and writes, including updates and deletes. State plausible
data volumes and read/write assumptions, and use them to explain which access
patterns the design must prioritize.


### 3. Conceptual model (ERD)

Start with an entity-relationship diagram. This is the vendor-neutral picture of
your data before you contort it to fit the database. I won't be strict on diagramming
convention. I just want to get a conceptual idea of how it works.

### 4. Physical model with justification

Translate the ERD into your database's actual structures — collections and
document shapes, tables and key schemas, node and relationship types, or search
index mappings and analyzers. For **every significant decision**, say which access
pattern justifies it. 

Show how the model handles growing relationships, updates, and deletes. If data
is duplicated, explain how copies are maintained and what consistency the
application can expect.

Explain the **trade-offs**: which access patterns are fast, which are expensive,
and which need an index or an application-side step. Compare a key modeling
decision with a viable alternative and explain why your access patterns favor
the chosen design. Identify a changed requirement and trace its effect on the
model's structures and queries.

### 5. Working implementation

**You must implement the model and the example you discuss**, in the programming
language of your choice. The sample data must exercise the relationships, keys, or
mappings you discuss.

---

## The Database Technology

Explain how the chosen database works. This is assessed under
**Understanding of the database technology** in the rubric.

### Core concepts and capabilities

- Explain how the database organizes and represents data.
- Explain how reads and writes work, including the relevant query, indexing,
  traversal, or search mechanisms.
- Describe its distinguishing capabilities and limitations. Connect them to
  the kinds of applications and workloads the database can support.
- Use diagrams or concrete examples to show how the mechanisms produce the
  behavior you describe.

### Production behavior

Explain the relevant architecture, scaling, replication, and consistency
behavior, including the consequences for an application. State any limits of the
chosen database or deployment. Cover the mechanisms relevant to your database:

- How the database **distributes data across nodes** (sharding, partitioning,
  consistent hashing; whatever your database calls it).
- How it **replicates** for availability, and what happens when a node fails.
- How a client finds the right node for a given piece of data.

A diagram and a technically correct explanation are sufficient for the production
component. Building a full production cluster is optional.

---

## The Demo

Demonstrate both the **implemented data model for your use case** and **CRUD**
(create, read, update, delete) using that model:

1. Show how the ERD maps to the actual stored structures. Display representative
   records, keys, relationships, or index mappings.
2. Run create, read, update, and delete operations on that data. Show the relevant
   state before and after mutations so the results can be checked.
3. Run the application's main access patterns. Connect each query to the model
   structure that supports it and verify the returned results.

A script, notebook, or database console is sufficient. Keep the model and query
results visible during the demonstration.

---

## Presentation Format

A **30-minute** group presentation, followed by questions.

Every group member must present part of the material and be able to answer
questions about the model.

---

## Grading

| Criterion | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|
| **Quality of data model** | Derives coherent conceptual models from explicit access patterns and workloads. Justifies structural choices and trade-offs, including handling of growth, updates/deletes, and consistency where applicable. | Models correctly support the main access patterns. Choices are linked to queries, but workload, growth, or trade-off analysis is incomplete. | Provides a use-case model, but its derivation from access patterns is unclear, the conceptual-to-physical mapping is incomplete, or a structural flaw leaves a main query unsupported. | Lists entities or fields without connecting them to the application's access patterns and usable database structures. | No assessable data model. |
| **Quality of demo** | Demonstrates the working use-case model, all four CRUD operations, and the main application queries; verifies results and state changes against the model. | Demonstrates the working use-case model and its main queries, but CRUD coverage or verification of state changes is incomplete. | Shows CRUD in the proposed use-case model, but application queries are unimplemented, fail, or have unverifiable results. | Shows CRUD on isolated records; no use-case data model is demonstrated. | No working database operation or observable query result is demonstrated. |
| **Understanding of the database technology** | Accurately explains data organization, read/write mechanisms, and required production behavior; connects distinguishing capabilities, limitations, and architectural trade-offs to their consequences for applications. | Correctly explains data organization, read/write mechanisms, main capabilities, and required production behavior, but leaves their application consequences or limits unexplained. | Describes capabilities and operations, but leaves a core mechanism unexplained or makes an error about storage, query processing, or distributed behavior. | Identifies the database and names features or terminology without explaining what they do or how the database works. | No assessable explanation of the database technology. |
| **Quality of presentation** | Organizes the technology, model, and demo into a sequence classmates can follow. Defines unfamiliar terms, makes visuals and results legible, and explains what to notice before moving on. | The main explanation and demonstration can be followed and central visuals are legible, but supporting terms, transitions, or results are left unexplained. | The main sections are identifiable, but missing connections, unreadable evidence, or rushed steps obscure a central part of the explanation or demo. | Presents slides, code, or outputs as fragments without enough context or guidance for the audience to follow the main idea or demonstration. | No assessable presentation. |

---

## Tips

- **Write the access patterns first.** If you find yourself designing the schema
  before you have listed them, stop and go back.
- **Use the notes.** The design patterns and single-table walkthroughs are
  directly reusable. The databases may differ, but the access patterns may be similar.
- **Pick a use case you understand.** You will model it better if you already know
  how the application behaves.
- **Keep the data small and real.** A handful of representative documents beats a
  million rows of noise for showing that your model works.
- **Don't overthink it** The model does not need to be complex. Things like session state or shopping
  cart data are simple use cases which are perfectly viable as long as you explain the usecase clearly.

---

## Academic Honesty and GenAI

This is a **Green** deliverable. You may use generative AI to help with the project.

**You own the output.** You are responsible for the accuracy and quality of
everything you submit and present, including material produced with GenAI.

Every group member must:

- Review and verify GenAI output before using it, including checking factual
  claims and testing generated code and queries.
- Understand the submitted model, code, results, and explanations.
- Be ready to explain how the work functions, justify the design choices, and
  defend your answers during the presentation and Q&A.
