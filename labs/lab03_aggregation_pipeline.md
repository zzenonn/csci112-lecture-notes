# Lab 3: MongoDB Aggregation Pipeline

## Objective
Practice MongoDB aggregation pipeline operations to perform complex data analysis and transformations using various collections in the `labs` database. This lab focuses on applying aggregation stages like `$match`, `$group`, `$sort`, `$unwind`, and `$bucket` to extract meaningful insights from real-world datasets.

## Prerequisites
- Access to MongoDB server with `labs` database
- Collections: `posts`, `inspections`, `companies`, `customers`
- Basic understanding of MongoDB aggregation framework

## Database Setup
Connect to your MongoDB instance and switch to the labs database:
```js
use labs
show collections
```

## Aggregation Pipeline Review

### Core Pipeline Stages
- `$match`: Filter documents (like find())
- `$group`: Group documents and perform calculations
- `$sort`: Sort documents by specified fields
- `$project`: Include/exclude fields or create new ones
- `$unwind`: Deconstruct array fields into separate documents
- `$bucket`: Group documents into buckets based on ranges
- `$limit`: Limit number of documents
- `$count`: Count documents in pipeline

### Aggregation Operators
- `$sum`: Sum values or count documents
- `$avg`: Calculate average
- `$max`, `$min`: Find maximum/minimum values
- `$push`: Create arrays from grouped values
- `$addToSet`: Create arrays with unique values

## Lab Tasks

### Task 1: Top 10 Most Common Tags (20 points)
**Requirement**: In the `posts` collection, find the top 10 most common tags.

```js
// Your aggregation pipeline here
```

### Task 2: Failed Inspections by Zip Code in Jamaica (15 points)
**Requirement**: Find how many inspections in Jamaica failed per zip code, sort from the most failures to the least failures.

```js
// Your aggregation pipeline here
```

### Task 3: Top 10 US Companies with Most Competitors (10 points)
**Requirement**: In the `companies` collection, find the top 10 companies with an office in the US with the most competitors.

```js
// Your aggregation pipeline here
```

### Task 4: Customer Demographics by Age Groups (5 points)
**Requirement**: In the `customers` collection, find the number of silver and gold customers based on the following age groups: under 18, 18-64, 65 and above.

**Hint**: Use the `$bucket` operator to group customers by age ranges.

```js
// Your aggregation pipeline here
```

## Deliverables

Submit **only**:
- **CSCI112-[StudentID1]-[LastName1]-[StudentID2]-[LastName2]-AggregationPipeline.js**

**File format example**:
```javascript
/*
Certificate of Authorship:
I have not discussed the JavaScript language code in my program with anyone 
other than my instructor or the teaching assistants assigned to this course.
I have not used JavaScript language code obtained from another student, 
or any other unauthorized source, either modified or unmodified.
If any JavaScript language code or documentation used in my program 
was obtained from another source, such as a textbook or course notes, 
that has been clearly noted with a proper citation in the comments of my program.
*/

// Lab 3 Solution - MongoDB Aggregation Pipeline
// Students: [Student1 Name], [Student2 Name]

// Task 1: Top 10 most common tags in posts (20 points)
// Your aggregation pipeline here

// Task 2: Failed inspections by zip code in Jamaica (15 points)
// Your aggregation pipeline here

// Task 3: Top 10 US companies with most competitors (10 points)
// Your aggregation pipeline here

// Task 4: Silver and gold customers by age groups (5 points)
// Your aggregation pipeline here
```

## Grading

**Total: 50 points**

- Task 1: Top 10 most common tags (20 points)
- Task 2: Failed inspections by zip code (15 points)
- Task 3: Top 10 US companies with most competitors (10 points)
- Task 4: Customer demographics by age groups (5 points)
