# MongoDB Data Modeling: Computed and Approximation Patterns

CSCI 112 / 212 - Contemporary Databases
Department of Computer Science

---

## Overview

This note introduces two important schema design patterns in MongoDB: the **Computed Pattern** and the **Approximation Pattern**. Both are used to optimize performance in read-heavy applications by reducing the cost of computation. These patterns are demonstrated using a bookstore application and are essential tools for designing scalable, efficient MongoDB schemas.

All examples use **PyMongo** running on your laptop, connecting to `mongod` on a VM. We assume the `books` collection from `7a - MongoDB Data Modeling Inheritance Pattern.md` already exists.

---

## PyMongo Setup

```python
from pymongo import MongoClient, ReturnDocument

client = MongoClient("mongodb://<vm_ip_address>:27017/")
db     = client["bookstore"]
books  = db["books"]
```

---

## Computed Pattern

### Purpose

The Computed Pattern is used to improve read performance by **pre-computing values when data is written or updated**, rather than recalculating them on every read. This is particularly useful when the same computation is needed frequently and the cost of computing it repeatedly is high.

### Use Case: Book Rating

In our bookstore application, each book has multiple reviews. Each review includes a `star_rating`. To avoid recalculating the average rating every time a user views the book, we store a computed average rating, the running total of stars, and the review count directly in the book document.

#### Example Schema

```json
{
  "title": "MongoDB: The Definitive Guide",
  "rating": {
    "review_count":   3,
    "total_stars":    12.9,
    "average_rating": 4.3
  }
}
```

Initialize the rating subdocument when the book is first created — the three counters start at zero so the cached average is well-defined from the very first read:

```python
books.insert_one({
    "_id": 1,
    "title": "MongoDB: The Definitive Guide",
    "rating": {"review_count": 0, "total_stars": 0.0, "average_rating": 0.0}
})
```

If the book already exists without a `rating` subdocument, you can backfill it instead:

```python
books.update_one(
    {"_id": 1},
    {"$set": {"rating": {"review_count": 0, "total_stars": 0.0, "average_rating": 0.0}}}
)
```

#### Update Logic

When a new review is added:

1. Atomically increment `total_stars` by the new rating and `review_count` by 1.
2. Recompute the average from the updated totals.
3. Persist the new average back to the document.

```python
def add_review(book_id, star_rating):
    """Add a single review and update the cached average rating."""
    updated = books.find_one_and_update(
        {"_id": book_id},
        {"$inc": {
            "rating.review_count": 1,
            "rating.total_stars":  star_rating
        }},
        return_document=ReturnDocument.AFTER,
    )

    new_avg = updated["rating"]["total_stars"] / updated["rating"]["review_count"]

    books.update_one(
        {"_id": book_id},
        {"$set": {"rating.average_rating": new_avg}}
    )
```

This ensures fast reads with accurate data, at the cost of additional writes.

#### End-to-end Demo

Putting the three steps together — insert the book, send a few reviews through `add_review`, then read the cached average:

```python
# 1. Insert a new book — initialize the rating subdoc to zeros
books.insert_one({
    "_id": 1,
    "title": "MongoDB: The Definitive Guide",
    "rating": {"review_count": 0, "total_stars": 0.0, "average_rating": 0.0}
})

# 2. Each incoming review calls add_review — totals and average update on every write
add_review(1, 5.0)
add_review(1, 4.0)
add_review(1, 4.0)

# 3. Reads are O(1) — no aggregation, just a field lookup
print(books.find_one({"_id": 1}, {"rating": 1}))
# → {'_id': 1, 'rating': {'review_count': 3, 'total_stars': 13.0, 'average_rating': 4.333...}}
```

The average is computed at **write** time and cached. Every reader sees the latest value with one field read:

```python
book = books.find_one({"_id": 1}, {"title": 1, "rating.average_rating": 1})
```

---

### Roll-Up Operations

Roll-up operations are another form of computed values, where data is **aggregated periodically** rather than on demand.

#### Use Case: Product Type Summary

Internal stakeholders want to see daily summaries of:

- Total number of books by product type
- Average number of authors per product type

Instead of computing this on every request, we use an aggregation pipeline to generate and store summary documents at regular intervals.

#### Aggregation Pipeline Example

```python
roll_up_pipeline = [
    {"$group": {
        "_id":   "$product_type",
        "count": {"$sum": 1},
        "average_number_of_authors": {"$avg": {"$size": "$authors"}}
    }},
    {"$merge": {
        "into": "book_summary",
        "on": "_id",
        "whenMatched": "replace",
        "whenNotMatched": "insert"
    }}
]

books.aggregate(roll_up_pipeline)
```

Run this on a schedule (e.g., a nightly cron job or a serverless scheduled function).

#### Sample Output (in `book_summary`)

```python
for doc in db["book_summary"].find():
    print(doc)
```

```json
[
  {"_id": "audiobook", "count": 1, "average_number_of_authors": 1},
  {"_id": "ebook",     "count": 1, "average_number_of_authors": 3},
  {"_id": "book",      "count": 1, "average_number_of_authors": 3}
]
```

Readers query `book_summary` directly — they never trigger a live aggregation.

---

## Approximation Pattern

### Purpose

The Approximation Pattern is used when **exact values are not critical** and the cost of maintaining perfect accuracy is too high. This pattern is especially useful for high-volume data and big data applications.

### Key Idea

Instead of updating computed values on every write, we update them occasionally using a **statistically valid approximation**. This reduces the number of writes and can help avoid contention on frequently updated documents.

### Use Case: High-Volume Book Ratings

When a book has thousands or millions of reviews, each new review has minimal impact on the average rating. In such cases, recalculating the average rating for every new review is inefficient.

Instead, we:

- Use a random number generator to decide whether to update the rating.
- Only update the rating when the random number hits a threshold (e.g., 10 out of 10).
- When updating, extrapolate the new review as if 10 reviews were added.

#### Example Logic

```python
import random

def add_review_approx(book_id, star_rating):
    """Add a review using the Approximation Pattern.

    Only ~10% of reviews actually trigger a database update; when they do,
    the update is extrapolated as if 10 reviews were added.
    """
    if random.randint(1, 10) == 10:
        updated = books.find_one_and_update(
            {"_id": book_id},
            {"$inc": {
                "rating.review_count": 10,
                "rating.total_stars":  star_rating * 10
            }},
            return_document=ReturnDocument.AFTER,
        )

        new_avg = updated["rating"]["total_stars"] / updated["rating"]["review_count"]

        books.update_one(
            {"_id": book_id},
            {"$set": {"rating.average_rating": new_avg}}
        )
```

This reduces the number of writes to the book document by approximately 90%, while maintaining a statistically valid average. The accuracy loss is proportional to `1/N` and becomes negligible at high review volumes.

#### End-to-end Demo

To see the savings in action, insert a "hot" book, push 1000 reviews through `add_review_approx`, and inspect the resulting document. Even though 1000 reviews flow in, only ~100 of them actually write to MongoDB — but the totals still reflect the full population because each write is extrapolated ×10:

```python
# 1. Insert a hot book — initialize the rating subdoc the same way as the Computed example
books.insert_one({
    "_id": 2,
    "title": "Hot Book",
    "rating": {"review_count": 0, "total_stars": 0.0, "average_rating": 0.0}
})

# 2. Simulate 1000 incoming 5-star reviews
for _ in range(1000):
    add_review_approx(2, 5.0)

# 3. Inspect — review_count ≈ 1000, but the document was written only ~100 times
print(books.find_one({"_id": 2}, {"rating": 1}))
# → {'_id': 2, 'rating': {'review_count': ~1000,
#                          'total_stars':  ~5000,
#                          'average_rating': 5.0}}
```

**Key observation:** `review_count` reflects an estimate of the true number of incoming reviews (each successful write counts as 10), and `average_rating` converges to the correct value as N grows. The actual number of writes to MongoDB is ~10× smaller than the number of incoming reviews.

#### Comparing Computed vs Approximation on the Same Workload

Run the same 1000 reviews through both functions to feel the difference:

```python
import time

# Computed Pattern: 1000 incoming reviews → 1000 DB writes
start = time.perf_counter()
for _ in range(1000):
    add_review(1, 5.0)
print(f"Computed:      {time.perf_counter() - start:.2f}s, 1000 writes")

# Approximation Pattern: 1000 incoming reviews → ~100 DB writes
start = time.perf_counter()
for _ in range(1000):
    add_review_approx(2, 5.0)
print(f"Approximation: {time.perf_counter() - start:.2f}s, ~100 writes")
```

In a real high-volume system, that 10× write reduction directly relieves contention on the book document — a key benefit when the same book receives thousands of reviews per second.

### Important Notes

- The Approximation Pattern is implemented in the **application logic**, not in the database schema.
- The schema still includes the same `review_count`, `total_stars`, and `average_rating` fields as in the Computed Pattern.
- This pattern trades a small loss in accuracy for significant performance gains.

---

## Summary

| Pattern              | Accuracy      | Performance Impact      | Implementation Location  |
|----------------------|---------------|-------------------------|--------------------------|
| Computed Pattern     | High (Exact)  | More writes, fast reads | Database + Application   |
| Approximation Pattern| Moderate      | Fewer writes, fast reads| Application Only         |

Both the Computed and Approximation Patterns are valuable tools for optimizing MongoDB schema design. Choosing between them depends on your application's performance needs and tolerance for approximation.

---

## Additional Resources

- [PyMongo Documentation](https://pymongo.readthedocs.io/)
- [MongoDB `$inc` Operator](https://www.mongodb.com/docs/manual/reference/operator/update/inc/)
- [MongoDB Aggregation `$merge`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/merge/)

---

## Full Example

The script below is the entire walkthrough in one file — copy-paste it into a Python REPL or a `.py` file after editing the `<vm_ip_address>`. It assumes the `books` collection already exists from [notes/7a](7a%20-%20MongoDB%20Data%20Modeling%20Inheritance%20Pattern.md). It defines `add_review` (Computed) and `add_review_approx` (Approximation), runs both end-to-end demos, then runs a roll-up into `book_summary`.

```python
import random
import time
from pymongo import MongoClient, ReturnDocument

client = MongoClient("mongodb://<vm_ip_address>:27017/")
db     = client["bookstore"]
books  = db["books"]

# ---------------------------------------------------------------------------
# Computed Pattern: pre-compute on every write
# ---------------------------------------------------------------------------
def add_review(book_id, star_rating):
    """Add a single review and update the cached average rating exactly."""
    updated = books.find_one_and_update(
        {"_id": book_id},
        {"$inc": {
            "rating.review_count": 1,
            "rating.total_stars":  star_rating
        }},
        return_document=ReturnDocument.AFTER,
    )
    new_avg = updated["rating"]["total_stars"] / updated["rating"]["review_count"]
    books.update_one(
        {"_id": book_id},
        {"$set": {"rating.average_rating": new_avg}}
    )

# Computed demo: insert a fresh book, send three reviews, read the cached average
books.delete_one({"_id": 1})
books.insert_one({
    "_id": 1,
    "title": "MongoDB: The Definitive Guide",
    "rating": {"review_count": 0, "total_stars": 0.0, "average_rating": 0.0}
})

add_review(1, 5.0)
add_review(1, 4.0)
add_review(1, 4.0)

print("Computed result:", books.find_one({"_id": 1}, {"rating": 1}))
# → {'_id': 1, 'rating': {'review_count': 3, 'total_stars': 13.0, 'average_rating': 4.333...}}

# ---------------------------------------------------------------------------
# Approximation Pattern: only ~10% of reviews actually write to MongoDB
# ---------------------------------------------------------------------------
def add_review_approx(book_id, star_rating):
    """Add a review using the Approximation Pattern.

    Only ~10% of reviews actually trigger a database update; when they do,
    the update is extrapolated as if 10 reviews were added.
    """
    if random.randint(1, 10) == 10:
        updated = books.find_one_and_update(
            {"_id": book_id},
            {"$inc": {
                "rating.review_count": 10,
                "rating.total_stars":  star_rating * 10
            }},
            return_document=ReturnDocument.AFTER,
        )
        new_avg = updated["rating"]["total_stars"] / updated["rating"]["review_count"]
        books.update_one(
            {"_id": book_id},
            {"$set": {"rating.average_rating": new_avg}}
        )

# Approximation demo: insert a hot book, send 1000 reviews, inspect the document
books.delete_one({"_id": 2})
books.insert_one({
    "_id": 2,
    "title": "Hot Book",
    "rating": {"review_count": 0, "total_stars": 0.0, "average_rating": 0.0}
})

for _ in range(1000):
    add_review_approx(2, 5.0)

print("Approximation result:", books.find_one({"_id": 2}, {"rating": 1}))
# → review_count ≈ 1000, total_stars ≈ 5000, average_rating ≈ 5.0

# ---------------------------------------------------------------------------
# Side-by-side timing: 1000 incoming reviews each
# ---------------------------------------------------------------------------
start = time.perf_counter()
for _ in range(1000):
    add_review(1, 5.0)
print(f"Computed:      {time.perf_counter() - start:.2f}s, 1000 writes")

start = time.perf_counter()
for _ in range(1000):
    add_review_approx(2, 5.0)
print(f"Approximation: {time.perf_counter() - start:.2f}s, ~100 writes")

# ---------------------------------------------------------------------------
# Roll-up: aggregate the books collection into a daily summary
# ---------------------------------------------------------------------------
roll_up_pipeline = [
    {"$group": {
        "_id":   "$product_type",
        "count": {"$sum": 1},
        "average_number_of_authors": {"$avg": {"$size": "$authors"}}
    }},
    {"$merge": {
        "into": "book_summary",
        "on": "_id",
        "whenMatched": "replace",
        "whenNotMatched": "insert"
    }}
]
books.aggregate(roll_up_pipeline)

print("Roll-up summary:")
for doc in db["book_summary"].find():
    print(" ", doc)
```

Running this end-to-end exercises every pattern in the note: Computed produces an exact average from atomic `$inc` updates, Approximation skips ~90% of the writes for a high-volume book, the side-by-side timing makes the write-savings concrete, and the roll-up demonstrates the periodic-aggregation variant of the Computed Pattern.
