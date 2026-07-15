# Google — Senior Data Engineer (Round 1: Live Coding)

**Format:** ~45 min in Google's own browser editor (`interview.google.com`) — no execution, no autocomplete. One SQL question with follow-ups, then one Python question. Interviewer probes **time/space complexity out loud** the whole way.

Loop structure (this role): ① SQL + Python live coding → ② ETL design + ML → ③ Hiring manager / behavioral.

---

## SQL — Year-over-Year revenue change

Given a table `data.revenue`:

| customer | year | revenue |
|---|---|---|
| ABC | 2018 | 100 |
| ABC | 2019 | 1000 |
| ABC | 2020 | 200 |
| ABC | 2021 | 300 |
| BCD | 2018 | 11000 |
| BCD | 2020 | 72000 |
| BCD | 2021 | 300 |

**Q1. Calculate the year-over-year % change in revenue for each customer as a new column.**

```sql
SELECT
  customer,
  year,
  revenue,
  (revenue - LAG(revenue) OVER (PARTITION BY customer ORDER BY year)) * 100.0
    / LAG(revenue) OVER (PARTITION BY customer ORDER BY year) AS yoy_pct_change
FROM data.revenue;
```

**Q2 (follow-up). Notice BCD is missing 2019 — find the missing years per customer.**

Approach: generate the full year range per customer (min→max year), left join / anti-join actual rows.

```sql
WITH bounds AS (
  SELECT customer, MIN(year) AS min_y, MAX(year) AS max_y
  FROM data.revenue GROUP BY customer
),
expected AS (
  SELECT b.customer, y.year
  FROM bounds b
  JOIN years y ON y.year BETWEEN b.min_y AND b.max_y   -- years = generated series
)
SELECT e.customer, e.year
FROM expected e
LEFT JOIN data.revenue r
  ON r.customer = e.customer AND r.year = e.year
WHERE r.year IS NULL;
```

**Q3 (follow-up). Flag significant outlier changes in the new YoY column — e.g. 2× over or under the customer's average change.**

```sql
WITH yoy AS (
  SELECT customer,
         (revenue - LAG(revenue) OVER (PARTITION BY customer ORDER BY year)) * 100.0
           / LAG(revenue) OVER (PARTITION BY customer ORDER BY year) AS pct_change
  FROM data.revenue
)
SELECT customer, pct_change,
       CASE WHEN ABS(pct_change) > 2 * AVG(ABS(pct_change))
                 OVER (PARTITION BY customer)
            THEN 'outlier' END AS flag
FROM yoy;
```

The exact threshold matters less than saying *why*: define a per-customer baseline, then flag deviations beyond a multiple of it.

---

## Python — Anagram check

> Write a function that determines if two given strings are anagrams (a word or phrase made by transposing the letters of another). **Propose and develop different approaches, and explain their time and space complexities.**

Examples given: `tar / rat`, `listen / silent`.

**Approach 1 — sort and compare.** O(n log n) time, O(n) space.

```python
def is_anagram(a: str, b: str) -> bool:
    return sorted(a) == sorted(b)
```

**Approach 2 — counter.** O(n) time, O(k) space (k = alphabet size).

```python
from collections import Counter

def is_anagram(a: str, b: str) -> bool:
    return len(a) == len(b) and Counter(a) == Counter(b)
```

They want you to *volunteer* the trade-off (sorting is simpler; counting is asymptotically better and short-circuits on length) — not wait to be asked.

---

## Takeaways

- The SQL question is easy; the round is won or lost on the **follow-up chain** (missing years, outliers) and how calmly you extend your first query.
- Complexity talk is graded. If saying "this is O(n log n) because of the sort, and here's how I'd get to O(n)" doesn't come out naturally, drill it before the interview — it sinks people who can otherwise solve the problem.
