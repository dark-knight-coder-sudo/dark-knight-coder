# Airbnb — Data Engineer (CodeSignal Screen)

**Format:** timed CodeSignal test, ~60 minutes, **5 questions** — four SQL (MySQL 8, written inside a `CREATE PROCEDURE solution()` wrapper) and one Python. Proctored (screen sharing on). The SQL questions all use the same two booking tables, so read the schema once and reuse it.

**The shared schema:**

```
user_booking(reservation_id, guest_id, listing_id, destination, ts_booking, ds_booking)
users(guest_id, ts_account_created, ds_account_created, origin)
```

`origin` lives on **users** (where the guest is from); `destination` lives on **user_booking** (where the listing is). Every question below turns on joining or windowing these correctly.

---

## Q1 — Top origin→destination pair

> Return the origin-destination combination that had the most bookings, along with the booking count.

```sql
SELECT u.origin, ub.destination, COUNT(*) AS cnt
FROM user_booking ub
JOIN users u ON u.guest_id = ub.guest_id
GROUP BY u.origin, ub.destination
ORDER BY cnt DESC
LIMIT 1;
```

⚠️ **The trap that costs points:** ordering by `origin, destination` instead of by the **count** returns an alphabetical pair, not the top pair — the grader marks it wrong even though the query "looks" fine. `ORDER BY` must express the question.

## Q2 — Guests whose first booking was NYC

> Find the number of guests whose first booking was for NYC.

"First" = earliest `ts_booking` per guest → `ROW_NUMBER`:

```sql
WITH first_booking AS (
  SELECT guest_id, destination,
         ROW_NUMBER() OVER (PARTITION BY guest_id ORDER BY ts_booking) AS rn
  FROM user_booking
)
SELECT COUNT(*) AS guests
FROM first_booking
WHERE rn = 1 AND destination = 'NYC';
```

## Q3 — Listings whose first booking was by a first-time guest

> Find the number of listings whose first booking was by a first-time guest.

Two "firsts" at once — rank the **same rows** two ways and require both:

```sql
WITH ranked AS (
  SELECT listing_id, guest_id,
         ROW_NUMBER() OVER (PARTITION BY listing_id ORDER BY ts_booking) AS listing_rn,
         ROW_NUMBER() OVER (PARTITION BY guest_id  ORDER BY ts_booking) AS guest_rn
  FROM user_booking
)
SELECT COUNT(*) AS listings
FROM ranked
WHERE listing_rn = 1 AND guest_rn = 1;
```

The row that is simultaneously a listing's first booking **and** that guest's first booking is exactly the event being asked about.

## Q4 — Median nightly price per market (SQL)

> Table `listing_price(listing_id, market, nightly_price)`. Find the median nightly price per market. You may assume an odd number of listings per market.

Odd-count median = the middle row by rank:

```sql
WITH ranked AS (
  SELECT market, nightly_price,
         ROW_NUMBER() OVER (PARTITION BY market ORDER BY nightly_price) AS rn,
         COUNT(*)     OVER (PARTITION BY market)                        AS cnt
  FROM listing_price
)
SELECT market, nightly_price AS median_nightly_price
FROM ranked
WHERE rn = (cnt + 1) / 2
ORDER BY median_nightly_price DESC;
```

⚠️ **Graded literally:** the expected output named the column `median_nightly_price` and expected a specific row order. A working median that returns `median_price` still fails the diff. Copy the expected column names exactly before writing the query.

## Q5 — Median per market, in Python

> Input lines look like `San Francisco,100`, grouped by market. Compute the median nightly price per market and produce lines like `San Francisco,150` — in the order markets first appear. (Even-count groups average the two middle values.)

```python
def solution(arg1):
    data, order = {}, []
    for line in arg1:
        market, price = line.rsplit(",", 1)
        if market not in data:
            data[market] = []
            order.append(market)
        data[market].append(int(price))

    out = []
    for market in order:
        prices = sorted(data[market])
        n = len(prices)
        median = prices[n // 2] if n % 2 else (prices[n // 2 - 1] + prices[n // 2]) / 2
        if median == int(median):
            median = int(median)
        out.append(f"{market},{median}")
    return out
```

⚠️ **The gotcha that scored 0/2 with correct logic:** the function must **return** the list of strings — printing to stdout gives `Your return value: null` on every test. Read the harness contract before the algorithm. Also: preserve **first-appearance order** of markets (don't sort them), and format whole medians as integers (`150`, not `150.0`).

---

## Takeaways

- One schema, five questions — invest the first two minutes reading it properly and the rest of the test is window-function reps: `ROW_NUMBER` for "first X per Y", rank-vs-count for medians.
- **The grader is a string diff.** Column aliases, row order, return-vs-print, integer formatting — these cost more points here than SQL logic does.
- The double-`ROW_NUMBER` trick (Q3) is worth memorizing: any "first A that was also a first B" question is one CTE with two windows.
