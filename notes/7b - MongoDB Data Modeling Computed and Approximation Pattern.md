# MongoDB Data Modeling: Computed and Approximation Patterns

CSCI 122 / 212 - Contemporary Databases  
Department of Computer Science

---

## Overview

This README introduces two important schema design patterns in MongoDB: the **Computed Pattern** and the **Approximation Pattern**. Both are used to optimize performance in read-heavy applications by reducing the cost of computation. These patterns are demonstrated using a bookstore application and are essential tools for designing scalable, efficient MongoDB schemas.

---

## Computed Pattern

### Purpose

The Computed Pattern is used to improve read performance by pre-computing values when data is written or updated, rather than recalculating them on every read. This is particularly useful when the same computation is needed frequently and the cost of computing it repeatedly is high.

### Use Case: Book Rating

In our bookstore application, each book has multiple reviews. Each review includes a `star_rating`. To avoid recalculating the average rating every time a user views the book, we store a computed average rating and review count directly in the book document.

#### Example Schema

```json
{
  "title": "Example Book",
  "rating": {
    "review_count": 3,
    "average_rating": 4.3
  }
}
```

#### Update Logic

When a new review is added:
1. Multiply the current average by the current review count to get the total stars.
2. Add the new review's star rating.
3. Increment the review count.
4. Divide the new total stars by the new review count to get the updated average.

This ensures fast reads with accurate data, at the cost of additional writes.

---

### Roll-Up Operations

Roll-up operations are another form of computed values, where data is aggregated periodically rather than on demand.

#### Use Case: Product Type Summary

Internal stakeholders want to see daily summaries of:
- Total number of books by product type
- Average number of authors per product type

Instead of computing this on every request, we use an aggregation pipeline to generate and store summary documents at regular intervals.

#### Aggregation Pipeline Example

```javascript
var roll_up_product_type_and_number_of_authors_pipeline = [
  {
    $group: {
      _id: "$product_type",
      count: { $sum: 1 },
      averageNumberOfAuthors: { $avg: { $size: "$authors" } }
    }
  }
];
```

#### Sample Output

```json
[
  { "_id": "audiobook", "count": 1, "averageNumberOfAuthors": 1 },
  { "_id": "ebook", "count": 1, "averageNumberOfAuthors": 3 },
  { "_id": "book", "count": 1, "averageNumberOfAuthors": 3 }
]
```

This output can be stored in a summary collection and updated on a schedule (e.g., daily), reducing the need for real-time computation.

---

## Approximation Pattern

### Purpose

The Approximation Pattern is used when exact values are not critical and the cost of maintaining perfect accuracy is too high. This pattern is especially useful for high-volume data and big data applications.

### Key Idea

Instead of updating computed values on every write, we update them occasionally using a statistically valid approximation. This reduces the number of writes and can help avoid contention on frequently updated documents.

### Use Case: High-Volume Book Ratings

When a book has thousands or millions of reviews, each new review has minimal impact on the average rating. In such cases, recalculating the average rating for every new review is inefficient.

Instead, we:
- Use a random number generator to decide whether to update the rating.
- Only update the rating when the random number hits a threshold (e.g., 10 out of 10).
- When updating, extrapolate the new review as if 10 reviews were added.

#### Example Logic

1. Generate a random number between 1 and 10.
2. If the number is 10:
   - Multiply the new review's rating by 10.
   - Add this to the total stars.
   - Increment the review count by 10.
   - Recalculate the average rating.

This reduces the number of writes to the book document by approximately 90%, while maintaining a statistically valid average.

### Important Notes

- The Approximation Pattern is implemented in the application logic, not in the database schema.
- The schema still includes fields for `review_count` and `average_rating`, just like in the Computed Pattern.
- This pattern trades a small loss in accuracy for significant performance gains.

---

## Summary

| Pattern              | Accuracy      | Performance Impact     | Implementation Location |
|----------------------|---------------|-------------------------|--------------------------|
| Computed Pattern     | High (Exact)  | More writes, fast reads | Database + Application   |
| Approximation Pattern| Moderate      | Fewer writes, fast reads| Application Only         |

Both the Computed and Approximation Patterns are valuable tools for optimizing MongoDB schema design. Choosing between them depends on your application's performance needs and tolerance for approximation.

---

## Lab Assignment

In the upcoming lab, you will implement both the Computed and Approximation Patterns in the bookstore application. You will:

- Use the Computed Pattern to maintain accurate average ratings for books with a small number of reviews.
- Apply the Approximation Pattern for books with a large number of reviews to reduce write overhead.
- Create a roll-up summary of product types using an aggregation pipeline.

This lab will help you understand how to balance accuracy and performance in real-world MongoDB applications.

Be sure to review this README and the lecture notes before starting the lab.

---

## Additional Resources

- [MongoDB Schema Design Patterns](https://www.mongodb.com/blog/post/building-with-patterns-a-summary)
- [MongoDB Aggregation Framework](https://docs.mongodb.com/manual/aggregation/)
- [MongoDB University Courses](https://university.mongodb.com/)

---

