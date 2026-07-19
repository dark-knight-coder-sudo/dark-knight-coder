# Beacon Hill — Data Engineer (Disney client)

**Format:** two parts. A timed **coding assessment** (stdin/stdout, multiple test cases per input — CodeSignal-style), then a **verbal technical round** where you talk through scenarios on a clock (~3 min each).

---

## Coding — Exponents of 2

> Input: first line is `T`, the number of test cases. Each of the next `T` lines is a number. For each, print `1` if it is a power of two, else `0`.

The catch: inputs can be non-integers, and the intended definition accepts fractional powers too (`0.5`, `0.25`), since halving/doubling reaches exactly `1.0`. Anything `≤ 0` is `0`.

```python
import sys

def is_power_of_two(n: float) -> bool:
    if n <= 0:
        return False
    while n > 1:
        n /= 2
    while n < 1:
        n *= 2
    return n == 1.0

def main():
    data = sys.stdin.read().split('\n')
    t = int(data[0])
    out = ['1' if is_power_of_two(float(data[i])) else '0' for i in range(1, t + 1)]
    print('\n'.join(out))

main()
```

For integers only, the one-liner is `n > 0 and (n & (n - 1)) == 0` — worth mentioning out loud even if the float version is what the tests require.

---

## Coding — Sorting Bank Accounts

> For each test case: an integer `N`, then `N` account-number strings. Count how many times each account appears and print each **unique account** with its count, **sorted by account number**. Blank line separates test cases.

```python
import sys
from collections import Counter

def main():
    data = sys.stdin.read().split('\n')
    idx = 0
    t = int(data[idx].strip()); idx += 1
    blocks = []
    for _ in range(t):
        n = int(data[idx].strip()); idx += 1
        accounts = [data[idx + j].strip() for j in range(n)]
        idx += n
        if idx < len(data) and data[idx].strip() == '':   # skip separator
            idx += 1
        counts = Counter(accounts)
        blocks.append('\n'.join(f"{acct} {counts[acct]}" for acct in sorted(counts)))
    print('\n\n'.join(blocks))

main()
```

What it tests: careful **index-based stdin parsing** with a variable number of lines per case, `Counter` for the tally, and `sorted()` on the keys for the required ordering.

---

## Verbal technical round (reported)

Two scenarios, answered on a timer:

**1. In an ETL process, what is a common method for handling data quality issues during the transformation phase?**

Short answer to anchor on: validate on the way through and **quarantine, don't drop**. Concretely — schema/type checks and constraint rules (not-null, ranges, referential) applied as the data transforms; route failing rows to a **dead-letter / reject table** with the reason attached instead of silently discarding them; standardize and de-dupe; and emit **data-quality metrics** (row counts in vs out, % rejected, null rates) so a regression is visible. Tools people name: dbt tests, Great Expectations, or hand-rolled rule tables.

**2. A 2-TB Delta table partitioned by `event_date` has slow reads and high cost. In 3 minutes, describe the exact sequence of Databricks actions to diagnose and improve performance and cost.** (Hints given: file sizes, small files, Z-ORDER targets, VACUUM/retention safety, Photon, caching, autoscaling, query-side techniques.)

A tight sequence that hits every hint:

1. **Diagnose first.** Check the Spark UI / query profile for the slow query, and inspect the table: `DESCRIBE DETAIL` for `numFiles` and `sizeInBytes` (→ average file size), and look at partition skew. Slow reads on a partitioned table are almost always the **small-file problem** — millions of tiny files under each `event_date`.
2. **Compact.** `OPTIMIZE` to coalesce small files toward the ~128 MB–1 GB target; on Databricks, enable **auto-optimize / optimizeWrite** so it stays compacted going forward.
3. **Z-ORDER** on the high-cardinality columns that actually appear in `WHERE`/join predicates (not on the partition column) — that's what turns full scans into data-skipping.
4. **Reclaim storage safely.** `VACUUM` old files, but keep the default 7-day retention (or set it deliberately) so you don't break time-travel or concurrent readers — never `VACUUM ... RETAIN 0` on a live table.
5. **Engine + compute.** Turn on **Photon** for vectorized execution; right-size the cluster and enable **autoscaling**; use **Delta caching** for hot repeated reads.
6. **Query side.** Push predicates on `event_date` so partition pruning kicks in, select only needed columns, and broadcast small dimension joins.
7. **Reconsider the layout.** If `event_date` produces tiny partitions, a coarser partition (month) plus Z-ORDER, or **liquid clustering**, often beats over-partitioning.

The framing that scores: **diagnose → compact → cluster → reclaim → tune compute → fix the query → rethink layout.** Naming the ~128 MB–1 GB file target and the VACUUM retention safety note is what separates a senior answer from a keyword list.
