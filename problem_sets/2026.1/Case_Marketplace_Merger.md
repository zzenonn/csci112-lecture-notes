# Case Study: Marketplace Merger

**CSCI 112 - Contemporary Databases**
**Midterm, First Semester SY 2026-2027**

## Background

**Bentahan** is a Philippine online marketplace where people buy and sell
second-hand items: furniture, appliances, books, baby gear, and more. It has about
3 million monthly active users and 2.8 million listings.

Last quarter Bentahan acquired two smaller apps:

- **GadgetSwap**, a resale app for phones, tablets, and laptops (about 700,000
  listings).
- **UkayCloset**, a pre-loved fashion app for clothes, shoes, and bags (about
  500,000 listings).

The three apps were built by different teams, and each stores its listings
differently. Bentahan must merge all three into one platform so that a buyer
searching for "iPhone" or "linen polo" sees everything in one place.

## How the Business Works

A seller posts a listing with a title, price, category, photos, location, and
category-specific details. For phones, buyers care about storage, battery health,
and whether the unit is openline. For clothes, buyers care about size, brand, and
condition. Buyers browse, open a listing, check the seller's rating and response
rate, and message the seller. Most deals close by meet-up or courier. When a deal
closes, the seller marks the listing as sold.

A sold item that still shows as available is the most common complaint Bentahan
receives. Buyers message the seller, travel to a meet-up, or pay a courier deposit
for an item that is gone. Support tickets and one-star reviews follow.

The merged platform cannot go down for a one-time migration of about 4 million
legacy listings. The GadgetSwap and UkayCloset apps stay live for a three-month
transition window, and their users keep posting, editing, and marking items sold
in the old apps during that window. Their listings must appear on the merged
platform throughout.

Treat each legacy seller as one platform seller. Merging duplicate accounts across
apps is out of scope.

## Access Patterns

| # | Access pattern | Who | Volume / frequency | How current / exact |
|---|---|---|---|---|
| AP1 | Search and browse all listings by category, price range, and location, newest first, 40 per page. Within a category, buyers can also filter by its specific details (phones by storage and battery health; clothes by size and brand). | Buyers | ~2,600 per second at peak (8–11 PM); about 600 of these use category-specific filters | Only available items may appear; must reflect "sold" immediately. Result totals ("about 12,400 results") may be approximate. New listings may take up to 1 minute to appear. |
| AP2 | Open a listing's detail page, including the seller's name, rating, and response rate | Buyers | ~3,000 per second at peak | Sold status must be exact. Seller rating and response rate may be up to 24 hours old. |
| AP3 | Post a new listing | Sellers | ~60,000 per day; peaks of 50 per second on weekends | Visible in search within 1 minute |
| AP4 | Mark a listing as sold | Sellers | ~25,000 per day | Exact. Once confirmed, the item must never show as available in search or on its detail page. |
| AP5 | Update a seller profile (display name, city) | Sellers | ~5,000 per day | The new name must show on all of the seller's listings within 24 hours. Top sellers have more than 2,000 listings. |
| AP6 | Show a seller's shop page: all their available listings, newest first | Buyers | ~300 per second | Sold status exact |

## Existing Data

Each app still runs on its own database. The samples below are typical, including
their inconsistencies: field names, price and date formats, photo lists, and
locations differ between apps and sometimes within one app. Some listings are
missing fields entirely.

**Bentahan** stores listings and sellers separately.

```json
{ "_id": "65f1a2c47e1d3b0a9c8e4401", "title": "IKEA Study Table, white",
  "price": 1500, "category": "Home & Living", "location": "Quezon City",
  "photos": ["https://cdn.bentahan.ph/p/8841.jpg"], "seller_id": 88213,
  "status": "available", "created_at": "2025-11-03T09:14:00Z" }

{ "_id": "65f1a3d97e1d3b0a9c8e4402", "title": "Graco Stroller, barely used",
  "price": 4200, "category": "Baby & Kids",
  "photos": [], "seller_id": 90117,
  "status": "sold", "created_at": "2026-01-20T02:40:00Z" }

{ "_id": 88213, "display_name": "Tita Baby Furniture", "rating": 4.8,
  "review_count": 312, "response_rate": "97%" }
```

**GadgetSwap** copies the seller into each listing at posting time. Sellers who
later changed their name still show the old name on older listings.

```json
{ "_id": "GS-2024-0091823", "item": "iPhone 13 128GB Midnight",
  "amount": "₱24,500", "cat": "phones",
  "specs": { "brand": "Apple", "storage": "128GB", "battery_health": 87,
             "openline": true },
  "pics": "https://gs.ph/i/a1.jpg,https://gs.ph/i/a2.jpg",
  "seller": { "name": "Jomar R.", "mobile": "09171234567", "stars": 4.6,
              "city": "Makati" },
  "sold": false, "posted": "03/15/2025 14:22" }

{ "_id": "GS-2025-0310440", "item": "Lenovo ThinkPad T480",
  "amount": "₱13,000.00", "cat": "laptops",
  "specs": { "brand": "Lenovo", "ram": "16GB", "storage": "512GB SSD" },
  "pics": "https://gs.ph/i/b7.jpg",
  "seller": { "name": "TechBodega PH", "stars": 4.9, "city": "Pasig" },
  "sold": true, "posted": "11/02/2025 09:05" }
```

**UkayCloset** uses Filipino field names and keeps sellers in a separate users
collection, keyed by handle.

```json
{ "_id": "66a0c1e20b4f5a7d1e2f3001", "pangalan": "Uniqlo Linen Polo",
  "presyo": 350.0, "uri": "tops", "size": "M", "brand": "Uniqlo",
  "kondisyon": "gently used", "tags": "linen, summer, polo",
  "larawan": ["https://uc.ph/x/91.jpg"], "user": "@ukay.ni.bea",
  "lugar": { "city": "Cebu City", "province": "Cebu" },
  "status": "SOLD", "date_listed": 1719900000 }

{ "_id": "66a0c2f80b4f5a7d1e2f3002", "pangalan": "Vintage Denim Jacket",
  "presyo": "800", "uri": "outerwear", "size": "Free Size",
  "tags": ["denim", "vintage"], "user": "@thriftwithjo",
  "lugar": { "city": "Davao City" },
  "status": "ACTIVE", "date_listed": 1741305600 }

{ "handle": "@ukay.ni.bea", "full_name": "Bea Villanueva",
  "avg_rating": 4.7, "replies_within_hour_pct": 0.92 }
```

## Change Request

Six months after the merger goes live, Bentahan adds a **Vehicles** category for
cars and motorcycles. Buyers filter vehicles by make, model, year, mileage,
transmission, fuel type, and whether the OR/CR (registration papers) is complete.
Vehicle listings will be fewer (about 2,000 new per day) but are viewed far more
per listing than a ₱350 polo.

At the same time, the listing format changes for all categories. A listing now
carries a list of meet-up spots and shipping options instead of one location, and
a price can be marked fixed or negotiable. New listings use the new format. The
more than 4 million existing listings do not have this information: none of them
has meet-up spots, shipping options, or a fixed/negotiable flag, and none is a
vehicle. Bentahan will not rewrite them, and they must keep working in search and
on detail pages.

## Implementation Task

Write Python/PyMongo code that does the following.

1. **Insert mock data** into collections that mirror the three old apps, in the
   legacy shapes above. Include the sample documents above plus about 10–15 more
   listings per source, with the same kinds of inconsistencies, both sold and
   available items, and 3–4 sellers per source. At least one seller should have 6
   or more listings.

2. **Migrate** the legacy data into your model using aggregation pipelines, not
   per-document Python loops. The migration cannot be one big-bang run: migrate
   only part of each source first. Show the listing count per source before and
   after migration, with any difference explained, and one listing from each
   source before and after migration.

3. **Run the transition window.** While some listings are still not migrated,
   insert new listings and mark some listings sold in the legacy shapes, as the old
   apps would. Show that these changes reach the merged platform without
   duplicates and without a sold item showing as available. Then add at least one
   new-format vehicle listing from the change request, and show that one search
   (AP1) and one detail page (AP2) return correct results across migrated
   listings, not-yet-migrated listings, and the vehicle listing.

4. **Run the queries for your top-ranked access patterns** (at least three,
   including one write), with before-and-after state for writes and the indexes
   each query uses (from `explain()`).
