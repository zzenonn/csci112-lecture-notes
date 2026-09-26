# Case Study: Airline Seat Sale

**CSCI 112 - Contemporary Databases**
**Midterm, First Semester SY 2026-2027**

## Background

Pulo Air is a Philippine low-cost carrier flying 51 domestic and regional routes
out of Manila, Cebu, and Clark. It operates about 380 flights a day on 180-seat
aircraft and sells up to 11 months ahead, so about 120,000 future flights are on
sale at any time.

Four times a year Pulo Air runs a piso-fare seat sale that opens at exactly
midnight. At the last sale, 1.4 million people visited in the first hour, searches
peaked at 30,000 per second, and booking attempts peaked at 4,000 per second.
Demand clusters on a few flights: one Manila–Caticlan flight before Holy Week
received 2,500 booking attempts in 20 seconds for its 20 piso-fare seats.

The old system sold 31 seats on that flight's 20-seat piso-fare bucket. Pulo Air
had to refund the extra passengers and give them vouchers, and the story trended
for two days. Management's first requirement is that this never happens again.

## How the Business Works

Each flight's seats are split into **fare buckets**: typically Piso Fare (₱1 base
fare plus taxes, 10 to 30 seats), Promo (₱499 to ₱1,499), Regular, and Flex. The
revenue team sets each bucket's size and price and changes them at any time,
including mid-sale, often moving unsold seats from a cheap bucket to a dearer one.

A customer searches a route over a range of dates, sees the lowest available fare
per day, picks a flight, and books one to nine passengers on one fare bucket. A
booking gets seats for every passenger or fails outright: if 3 seats are left and
a customer asks for 4, nothing is taken. When a bucket sells out, the site offers
the next cheapest bucket.

The price is locked in when the booking is confirmed. Later price changes never
change what a customer paid or sees as paid.

Operations publishes about 150 schedule changes a day (new departure time, terminal,
or flight number), and up to 2,000 a day in typhoon season. A changed flight can
have over 150 booked passengers, and each must see the new details the next time
they open their booking.

Flight pages show "N seats left at this fare" and "X people are looking at this
flight." Marketing added the second line because it lifts conversions.

## Access Patterns

| # | Access pattern | Who | Volume / frequency | How current / exact |
|---|---|---|---|---|
| AP1 | Search a route and date range (up to 31 days); show the lowest available fare per day | Customers | 30,000/s at sale peak; 800/s normal | Up to 10 s behind is fine; the flight page shows the real price |
| AP2 | Show a flight page with each bucket's price, "N seats left at this fare", and "X people are looking at this flight" | Customers | 12,000/s at peak; 400/s normal | Seats left may lag sales by a few seconds, but must not show seats in a bucket sold out for over 10 s. People looking: within 10% and up to 30 s behind |
| AP3 | Book 1–9 passengers on one bucket of one flight | Customers | 4,000/s at peak; up to 2,500 in 20 s on one flight | **Exact.** Never sell more seats than a bucket holds; all passengers get seats or the booking fails |
| AP4 | Record that a customer opened a flight page | System | 12,000/s at peak; 1,500/s on one hot flight | Roughly right |
| AP5 | View a booking: passengers, flights, times, fare paid | Customers, call center | 1,500/s on sale night; 200/s normal | Fare paid **exactly** as charged; schedule within 5 min of a change |
| AP6 | Publish a schedule change for a flight | Operations | 150/day; 2,000/day in typhoon season; a changed flight can have over 150 booked passengers | **Exact.** Every booked passenger on the flight must see the change |

## Existing Data

The old system, the one that oversold, stores one document per flight and one per
booking. A flight lists its fare buckets with their size and current price but no
count of seats sold: seats left are found by adding up the passengers on that
flight's bookings each time a page is shown. The site has no "people looking"
counter. During the last sale, search and flight pages slowed sharply, and
passengers on retimed flights kept seeing their old departure time until they
called the call center.

Bookings made before a mid-2026 update store no fare, only the bucket name; their
booking page shows the bucket's current price, and the amount charged survives only
as formatted text. Newer bookings store the fare and a copy of the schedule taken at
booking time. Three samples follow.

```json
{
  "_id": "PU361-2027-03-25",
  "flight_no": "PU 361", "origin": "MNL", "destination": "MPH",
  "departs": "2027-03-25T06:10:00+08:00", "arrives": "2027-03-25T07:15:00+08:00",
  "terminal": "T3", "seats": 180,
  "buckets": [
    {"code": "PISO", "size": 20, "base_fare": 1.00},
    {"code": "PROMO", "size": 40, "base_fare": 999.00},
    {"code": "REG", "size": 100, "base_fare": 3499.00},
    {"code": "FLEX", "size": 20, "base_fare": 5299.00}
  ]
}
```

```json
{
  "_id": "BK-7Q2M4X",
  "flight_id": "PU361-2027-03-25",
  "route": "MNL-MPH",
  "departs": "25 Mar 2027 06:10",
  "bucket": "PISO",
  "passengers": ["Juan Dela Cruz", "Maria Dela Cruz"],
  "amount": "PHP 1,426.00",
  "status": "confirmed",
  "booked": "2026-05-02T00:00:07+08:00"
}
```

```json
{
  "_id": "BK-9ZK31P",
  "flight_id": "PU361-2027-03-25",
  "flight": {"flight_no": "PU 361", "departs": "2027-03-25T06:10:00+08:00",
             "terminal": "T3"},
  "bucket": "PROMO",
  "passengers": [{"first": "Ana", "last": "Reyes", "birthdate": "1998-04-17"}],
  "fare_paid": {"base": 999.00, "taxes": 712.50, "total": 1711.50},
  "status": "confirmed",
  "booked": "2026-08-14T00:00:41+08:00"
}
```

Since both bookings were made, the revenue team has raised the Promo price and
operations has moved PU 361 to 06:40. Neither booking reflects the new time.

## Change Request

Six months after the migration, Pulo Air starts selling **add-ons**: checked baggage,
meals, and travel insurance. The revenue team prices each add-on per route and
changes prices over time. Customers can buy add-ons at booking or later, up to 4
hours before departure, and may buy more than one kind or more than one of a kind
per passenger.

A booking must show exactly which add-ons the customer bought and the price they
were charged for each, even after add-on prices change. Existing bookings, whether
migrated from the old system or made before this change, have no add-on
information.

## Implementation Task

Using Python and PyMongo, demonstrate the following on your model.

1. **Insert mock data.** Write a script that inserts mock data in the **existing**
   shape shown above: 3 routes, 12–15 flights across a few dates, each with four
   fare buckets, and 15–20 bookings that mix the old and new booking formats. One
   flight's piso-fare bucket must have exactly 5 seats left.

2. **Migrate.** Move the data into your model using an aggregation pipeline
   (`$merge` or `$out`). Print document counts per collection before and after,
   and show one old-format and one new-format booking before and after migration.

3. **Last-seats rush and people-looking counter.** On the flight whose piso-fare
   bucket has 5 seats left, launch at least 50 concurrent booking attempts
   (threads or processes) at once, each for 1 to 4 passengers. Print the bucket
   before and after and each attempt's party size and outcome. Prove nothing was
   oversold: independently count the passengers booked on that bucket, show it
   matches what the flight shows as sold and does not exceed the bucket size, and
   show that failed attempts left no partial seats. Reset and run the rush at least
   three times. Then simulate about 10,000 flight-page views with a loop (these are
   not seed data) across a few flights, one hot flight getting most of them. Print
   the views simulated, the database writes your code made to record them, the
   value each flight would display, the true count, and the percentage error.

4. **Access pattern queries.** Run the queries for your top-ranked access patterns
   (at least AP1, AP2, and AP5). For AP5, show that a migrated booking keeps its
   fare paid after the bucket price changes, and shows the new departure time after
   a schedule change (AP6); print the state before and after each write. Include
   `explain()` output or the indexes each query uses.
