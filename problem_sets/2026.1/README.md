# CSCI 112 Midterm: Data Modeling Case Studies

**Course:** CSCI 112 - Contemporary Databases
**Term:** First Semester, SY 2026-2027
**Format:** Individual, written

Five case studies are listed below. **Answer one case for a maximum grade of B,
or two cases for a maximum grade of A.** For each case you pick, design a
MongoDB data model for the application and explain your design decisions in a
short paper.

The cases do not have a single correct answer. Different models can earn full marks
if the case's access patterns justify them.

If anything in a case is unclear or missing, make an assumption and state it in
your design paper. Each assumption must be reasonable for the business and must
not contradict the case.

---

## Table of Contents

- [Case Studies](#case-studies)
- [What to Submit](#what-to-submit)
- [The Design Paper](#the-design-paper)
- [The Appendix](#the-appendix)
- [Format and Rules](#format-and-rules)
- [Grading](#grading)
- [Academic Honesty and GenAI](#academic-honesty-and-genai)

---

## Case Studies

1. [Airline Seat Sale](Case_Airline_Seat_Sale.md)
2. [Food Delivery](Case_Food_Delivery.md)
3. [Music and Podcast Streaming](Case_Music_Podcast_Streaming.md)
4. [Marketplace Merger](Case_Marketplace_Merger.md)
5. [Mobile Game](Case_Mobile_Game.md)

Each case gives you:

- **Background:** the business, its users, and its volumes.
- **How the business works:** what happens day to day, from the business's point
  of view.
- **Access patterns:** what users and staff need to see and do, how often, and
  how current or exact each result must be.
- **Existing data:** how the application stores its data today. It does not
  serve every access pattern well.
- **Change request:** a new requirement that arrives later.
- **Implementation task:** the code you must write and run.

The cases describe the business, not the data. Deciding what data the application
must collect and store is part of your design.

---

## What to Submit

Submit **one ZIP file per case** to the Midterm assignment on Canvas. Each ZIP
contains:

1. **A PDF** with two parts, in this order:
   - **Design paper** (graded for its reasoning; page limit applies)
   - **Appendix** (evidence; no page limit)
2. **Your code** as Python (`.py`) files, ready to run:
   - a script that inserts the mock/source data required by the case;
   - the case's migration and implementation task;
   - the queries for your top-ranked access patterns.

Name the ZIP and the PDF inside it the same way:

```
CSCI112-[StudentID]-[LastName]-Midterm-<CaseName>.zip
CSCI112-[StudentID]-[LastName]-Midterm-<CaseName>.pdf
```

For example: `CSCI112-181234-Cruz-Midterm-AirlineSeatSale.zip`.

---

## The Design Paper

Write in prose. Use headings for the four sections below.

### 1. Access pattern analysis

- Rank the case's access patterns by priority. Use the case's numbers (frequency,
  volume) to explain the ranking.
- Identify which results must be exact and which may be stale or approximate.
- State any assumption you had to make where the case is unclear or silent.
- Identify the access patterns that pull the model in different directions.

### 2. Data model and decisions

- State the data the application must collect and store to serve the access
  patterns.
- Describe your collections and the shape of their documents.
- Name the patterns you applied (for example, Computed, Subset, Extended Reference).
- For each significant decision, name the access pattern it serves and explain why,
  using the case's numbers.
- Explain how the model handles growth, and how duplicated data is kept up to date.

### 3. Trade-offs and alternatives

- For your most important decision, describe a viable alternative you rejected and
  why the access patterns favor your choice.
- State which access patterns your model makes slow or expensive.
- State what consistency, staleness, or approximation error each result can have.

### 4. Change request

- Explain what changes in your model to support the change request: documents,
  indexes, queries, and existing data.
- If existing data must be migrated, say how and when.

---

## The Appendix

The appendix supports the design paper. Arguments placed only in the appendix
are not graded. Include:

1. **ERD:** the entities and relationships before they are mapped to MongoDB.
2. **Sample documents:** one or more representative documents per collection.
3. **Indexes:** each index you create, with its field order and the access pattern
   it supports.
4. **Mock/source data:** what your data script inserts, with the document count per
   collection. About 10–20 documents per collection is enough, as long as the data
   exercises every access pattern, the migration, and the implementation task.
5. **Output:** the actual output from running the code in your ZIP against your
   MongoDB instance. For writes, show the relevant state before and after.

Submit the code itself as Python source files in the ZIP, not in the PDF. Use
PyMongo.

---

## Format and Rules

- **Page limit:** the design paper is at most **two pages per case**.
- **Formatting:** double-spaced, Times New Roman 12 pt, 1-inch margins.
- Content past the second page of the design paper is not graded.
- The appendix does not count toward the page limit.
- **MongoDB only.** Do not use another database in your model.
- Run your code on your own MongoDB instance (such as the VM from the labs).

---

## Grading

Each case is graded separately with the rubric below. Each criterion is scored
**0–4**, so a case is worth **16 points**.

- **One case:** your midterm score is that case's score, and your grade is capped
  at **B**.
- **Two cases:** your midterm score is the average of the two case scores, and
  you can earn up to an **A**.

| Criterion | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|
| **Access pattern analysis** | Ranks the case's reads and writes by priority using its numbers. Identifies which results must be exact and which may be stale or approximate, states needed assumptions, and names the access patterns that conflict. | Ranks the main access patterns using the case's numbers, but exactness requirements, assumptions, or conflicts between patterns are incomplete. | Lists the access patterns, but the ranking is missing or not tied to the case's numbers, or a main write or exactness requirement is missed. | Restates the scenario or lists entities without deriving access patterns. | No assessable analysis. |
| **Data model and justification** | The model supports every required access pattern. Each significant decision is justified by a named access pattern and the case's numbers. Growth and the maintenance of duplicated data are handled. | The model supports the main access patterns. Most decisions are justified, but some justifications are generic, or growth or duplicate maintenance is incomplete. | The model supports some access patterns, but a main access pattern is unsupported or most justifications are generic. | Shows collections or fields without connecting them to access patterns, or the model cannot serve the main access pattern. | No assessable data model. |
| **Trade-offs, alternatives, and change request** | Compares the key decision with a viable alternative and explains why the access patterns favor the choice. States the model's slow or expensive paths and the consistency, staleness, or approximation error of each result. Traces the change request to documents, indexes, queries, and existing data. | Compares an alternative and names the main costs, but consistency and staleness, or the change request's effects, are incomplete. | Lists advantages and disadvantages in general terms, or the change request response is not traced to the model. | Asserts the design is best without comparing alternatives or costs. | No assessable trade-off analysis. |
| **Implementation** | Mock data exercises every access pattern. Code performs the case's implementation task and main access patterns on the proposed model. The output verifies the results, including before and after state for writes. Indexes match those in the design paper. | Code runs and performs the implementation task, but coverage of the main access patterns or verification of the output is incomplete. | Code runs basic inserts and queries, but the mock data cannot exercise the main access patterns, or the implementation task is missing, fails, or its output cannot be checked. | Code is present but does not run or does not match the proposed model. | No code or output. |

### How the rubric is applied

- **Any justified model can earn a 4.** You are graded on whether the case's access
  patterns justify your model, not on whether it matches a reference answer.
- **Stated requirements must hold.** If your model breaks a requirement the case
  says must be exact or consistent, **Data model and justification** is capped at 2.
- **Justifications must be specific to the case.** A justification that would apply
  unchanged to any application counts as generic.
- **Output must be real.** Output that does not come from running the submitted
  code scores 0 for **Implementation**.
- **Assumptions must make sense.** A stated, reasonable assumption is accepted.
  An assumption that contradicts the case, or is unreasonable for the business, is
  treated as an error in the work that depends on it.
- **Each error is counted once**, under the criterion it belongs to.

---

## Academic Honesty and GenAI

The midterm is an **🟠 Amber** deliverable. You may use generative AI, and you must
disclose how you used it.

Place the GenAI Usage Declaration immediately after the department's Certificate of
Authorship (CoA), at the start of each PDF and at the top of each code file. Follow the format in the course Readme.
If you did not use GenAI, keep the CoA and use the "no GenAI" version of the
declaration.

**You own the output.** You are responsible for the accuracy of everything you
submit, including material produced with GenAI. Verify every claim, and run every
query and pipeline you submit.
