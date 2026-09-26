# Case Study: Food Delivery

**CSCI 112 - Contemporary Databases**
**Midterm, First Semester SY 2026-2027**

## Background

Kain Na! is a food delivery app operating in Metro Manila, Cebu, and Davao. It
connects three groups of users:

- **Customers** order food from the mobile app. There are 1.2 million registered
  customers, of whom about 450,000 order at least once a month.
- **Restaurants** (about 9,000, from single carinderias to chain branches) receive
  orders on a tablet, prepare them, and manage their menus.
- **Riders** (about 6,000 active on a typical day) pick up food and deliver it.

The platform handles about 80,000 orders a day. Traffic is concentrated at lunch
(11:30 AM–1:30 PM) and dinner (6:00–8:30 PM), when orders peak at about 25 per
second. Payday weekends (the 15th and 30th) run about 40% above normal.

The app is now slow for its most loyal customers, and operations is worried
about the next payday rush.

## How the Business Works

A customer opens the app and lands on the home screen, which shows their name,
their default delivery address, and their last few orders with a **Reorder**
button. Regulars reorder the same meal several times a week, so this screen is the
most important in the app.

Each restaurant page shows its menu (30–80 items; up to 200 for chains) with
current prices, sold-out flags, and the restaurant's star rating and review
count. Restaurants change prices a few times a month and mark items sold out
during the day, about 5,000 menu edits a day in total. A new price must appear on
the menu within 1 minute.

When a customer checks out, the order records what they bought, the price of
each item at that moment, the delivery fee, the total, and the address the food
goes to. Payment is by card, GCash, or cash on delivery. The order then moves through
placed, accepted, preparing, picked up, and delivered (or cancelled). Live order
tracking for customers, restaurants, and riders runs on a separate system and is
out of scope for this case.

After delivery, the customer may rate the restaurant from 1 to 5 stars and leave
a short comment. About 30% of delivered orders get a review.

Customer support handles about 1,500 disputes and refund requests a day ("I was
charged ₱389 but the menu said ₱349"). **A receipt must always show exactly
what the customer was charged for each item and exactly where the order was
delivered, as it was at checkout.** It must not change when the restaurant later
changes prices or renames an item, or when the customer moves. Refunds are paid
from this receipt.

A reorder, by contrast, uses the restaurant's **current** menu: the customer pays
today's price, and items that are no longer offered or are sold out are flagged
before checkout.

## Access Patterns

| # | Access pattern | Who | Volume / frequency | How current / exact |
|---|---|---|---|---|
| AP1 | Open the home screen: name, default address, and the last 5 orders (restaurant, items, total, date) with a Reorder button | Customer | ~1.5M loads/day; ~150/sec at peak | Last orders must include an order placed a minute ago |
| AP2 | Browse a restaurant's menu with current prices, sold-out flags, star rating, and review count | Customer | ~6M views/day; ~600/sec at peak | Prices and sold-out flags must be current. Rating and review count may be up to 15 minutes behind; review count may be shown rounded ("1.2k reviews") |
| AP3 | Place an order | Customer | 80,000/day; 25/sec at peak | Must never be lost. Charged prices must match the menu at checkout |
| AP4 | Open a past receipt: items, per-item prices, fees, total, payment method, delivery address | Customer, support agent | ~20,000/day, including ~1,500 disputes | **Exact.** Must show prices and address as they were at checkout |
| AP5 | Submit a review for a delivered order; the restaurant's rating and review count update | Customer | ~24,000/day; popular restaurants get 300+ per day | Every review must be stored. The displayed rating may lag as stated in AP2 |
| AP6 | Page through full order history, 20 orders at a time, newest first | Customer | ~15,000/day; heavy users have 1,000–6,000 orders | Exact |

## Existing Data

Today, every order is stored inside the customer's document, in an array that
grows by one full order each time the customer checks out. Orders placed before
the app update in March 2025 store the delivery address as a single line of text;
newer orders store it in parts. Stars and review comments are kept on the order
inside the customer document. The heaviest 2% of customers have more than 1,000
embedded orders, and the home screen is slow for them. Restaurants and their
menus are stored separately. Three samples follow (the second is shortened).

```json
{
  "_id": "C-100482",
  "name": "Andrea Villanueva",
  "mobile": "+639171234567",
  "address": {"line1": "Unit 14B, The Grove Tower C", "barangay": "Ugong",
              "city": "Pasig", "lat": 14.5836, "lng": 121.0708},
  "orders": [
    {"order_no": "KN-2024-0000913", "placed": "2024-11-03T12:14:00+08:00",
     "restaurant": "Mang Inasal - Tiendesitas", "restaurant_id": "R-0412",
     "items": [{"name": "PM1 Paa Large", "qty": 2, "price": 189.00},
               {"name": "Halo-Halo", "qty": 1, "price": 99.00}],
     "delivery_fee": 49.00, "total": 526.00, "payment": "GCash",
     "address": "Unit 14B, The Grove Tower C, Ugong, Pasig",
     "status": "delivered", "stars": 5},
    {"order_no": "KN-2025-0417720", "placed": "2025-08-21T19:02:00+08:00",
     "restaurant": "Army Navy - Ortigas", "restaurant_id": "R-2207",
     "items": [{"name": "Steak Burrito", "qty": 1, "price": 285.00}],
     "delivery_fee": 39.00, "total": 324.00, "payment": "COD",
     "address": {"line1": "Unit 14B, The Grove Tower C", "barangay": "Ugong",
                 "city": "Pasig", "lat": 14.5836, "lng": 121.0708},
     "status": "delivered"}
  ]
}
```

```json
{
  "_id": "C-003117",
  "name": "Paolo Dizon",
  "mobile": "+639189876543",
  "address": {"line1": "22 Kamagong St", "barangay": "San Antonio",
              "city": "Makati", "lat": 14.5613, "lng": 121.0162},
  "orders": [
    {"order_no": "KN-2023-0001204", "placed": "2023-10-02T12:40:00+08:00",
     "restaurant": "Jollibee - Chino Roces", "restaurant_id": "R-0031",
     "items": [{"name": "1-pc Chickenjoy w/ Rice", "qty": 1, "price": 82.00}],
     "delivery_fee": 29.00, "total": 111.00, "payment": "Card",
     "address": "4F Dela Rosa Bldg, Legaspi Village, Makati",
     "status": "delivered", "stars": 4, "review": "Mainit pa pagdating"}
  ]
}
```

The second customer has 4,812 orders in the array; only one is shown. This
customer's profile address has since changed, but their old receipts must still
show the address each order was delivered to.

```json
{
  "_id": "R-0412",
  "name": "Mang Inasal - Tiendesitas",
  "city": "Pasig",
  "menu": [
    {"name": "PM1 Paa Large", "price": 199.00, "sold_out": false},
    {"name": "Halo-Halo", "price": 99.00, "sold_out": true}
  ]
}
```

Restaurant documents hold no rating or review count, and restaurant pages load
slowly at lunch.

## Change Request

Six months after the migration, two features are approved:

1. **Promos and bundles.** Restaurants can run time-limited promos, such as "20%
   off all milk tea, 2–5 PM weekdays until Oct 31" or a "₱399 Barkada Bundle" of
   several menu items sold together at one price. About 1,500 promos are active at
   any time. The menu (AP2) must show a promo only while it is running, and must
   stop showing it within 1 minute of it ending. Orders that used a promo must show
   the promo name and the discount on the receipt (AP4), exactly as applied, even
   after the promo ends or the restaurant edits it. No existing order has any promo
   or discount information.
2. **Saved addresses.** Customers can save up to 10 labeled addresses ("Home",
   "Office", "Lola's house") and choose one at checkout. Editing or deleting a saved
   address must not change any past receipt. Existing customers have only the
   single profile address shown above and no saved addresses; existing orders do
   not record which saved address was used.

## Implementation Task

1. **Insert mock data** in the existing shape shown above: about 15 customers
   (2 heavy customers with about 25 orders each, the rest with 1–5 orders),
   5 restaurants with 5–8 menu items each, a mix of text and structured delivery
   addresses, and reviews on some orders.
2. **Migrate** the data into your target model using an aggregation pipeline. No
   order may be lost or altered. Show the number of customers and orders, and the
   sum of order totals (₱), before and after; they must match. Show one heavy
   customer's document before and after, and that customer's AP1 home screen
   served from the new model.
3. **Show that receipts hold.** Change the price of an item at a restaurant and
   change a customer's address. Then open one of that customer's past receipts for
   that item (AP4) and show that it still displays the original price and delivery
   address.
4. **Run the queries** for your top-ranked access patterns (at least three,
   including at least one write, and AP6 for a heavy customer's oldest page). For
   writes, show the relevant state before and after. Show `explain()` output or
   the index used for each read.
