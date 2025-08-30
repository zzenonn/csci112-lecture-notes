# Lab 2: MongoDB Query Operations

## Objective
Practice MongoDB query operations including find, update, and aggregation using various collections in the `labs` database. This lab focuses on applying query operators, logical operators, and update operations to real-world datasets.

## Prerequisites
- Access to MongoDB server with `labs` database
- Collections: `grades`, `posts`, `stories`, `customers`, `inspections`
- Basic understanding of MongoDB query syntax and operators

## Database Setup
Connect to your MongoDB instance and switch to the labs database:
```js
use labs
show collections
```

## Query Operators Review

### Comparison Operators
- `$gt`, `$gte`: Greater than, greater than or equal
- `$lt`, `$lte`: Less than, less than or equal
- `$eq`, `$ne`: Equal, not equal
- `$in`, `$nin`: In array, not in array

### Logical Operators
- `$and`: Match all conditions
- `$or`: Match any condition
- `$not`: Negate condition
- `$nor`: Match none of the conditions

### Update Operators
- `$set`: Set field value
- `$inc`: Increment numeric value
- `$unset`: Remove field
- `$push`: Add to array
- `$pull`: Remove from array

## Lab Tasks

### Task 1: Student Grades Query (8 points)
**Requirement**: Find all students who have an exam grade higher than 75 in the `grades` collection.

```js
// Your query here
```

### Task 2: Posts with Logical Conditions (8 points)
**Requirement**: Find all posts that are either authored by a machine OR tagged with either "lamb" or "fold" in the `posts` collection.

```js
// Your query here
```

### Task 3: Digg Stories with Sorting (8 points)
**Requirement**: Find all digg stories where the container is "Gaming" OR the topic is "Microsoft" and sort by the number of comments from greatest to least in the `stories` collection.

```js
// Your query here
```

### Task 4: Customer Demographics Query (8 points)
**Requirement**: Find all gold customers younger than 40 in the `customers` collection.

```js
// Your query here
```

### Task 5: Bulk Update Operation (10 points)
**Requirement**: For all inspections with a "Fail" result, set a fine value of 100 in the `inspections` collection.

```js
// Your query here
```

### Task 6: Conditional Update Operation (10 points)
**Requirement**: Update all inspections done in the city of "ROSEDALE". For failed inspections, raise the "fine" value by 150 in the `inspections` collection.

```js
// Your query here
```

## Deliverables

Submit **only**:
- **CSCI112-[StudentID1]-[LastName1]-[StudentID2]-[LastName2]-MongoQueries.js**

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

// Lab 2 Solution - MongoDB Query Operations
// Students: [Student1 Name], [Student2 Name]

// Task 1: Students with exam grade > 75 (8 points)
// Your query here

// Task 2: Posts by machine or tagged with lamb/fold (8 points)
// Your query here

// Task 3: Gaming/Microsoft stories sorted by comments (8 points)
// Your query here

// Task 4: Gold customers under 40 (8 points)
// Your query here

// Task 5: Set fine for failed inspections (10 points)
// Your query here

// Task 6: Update ROSEDALE failed inspection fines (10 points)
// Your query here
```

## Grading

**Total: 50 points**

- Task 1: Student grades query (8 points)
- Task 2: Posts logical conditions (8 points)
- Task 3: Stories with sorting (8 points)
- Task 4: Customer demographics (8 points)
- Task 5: Bulk update operation (10 points)
- Task 6: Conditional update (10 points)
