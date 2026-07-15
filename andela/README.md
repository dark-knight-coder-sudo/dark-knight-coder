# Andela — Data Engineer (Client Screening Take-Home)

**Format:** take-home exercise with provided datasets. Two parts: clean a messy user-records file, then analyze product usage data. Deliverable is your cleaned output + insights, with the code that produced them.

---

## Part 1 — Messy user records (data cleaning)

A spreadsheet of synthetic user records with columns like:

```
first_name, last_name, email, home_phone, mobile_phone,
street_address, city, state, postal_code, date_of_birth
```

Deliberately dirty, in the classic ways:

- Missing values scattered across every column (some rows have no email, some no phone).
- Inconsistent phone formats.
- Duplicate / near-duplicate people.

What they look for: a repeatable cleaning pipeline (not hand-edits) — standardize formats, define a dedupe key, decide explicitly what to do with unfixable rows, and **document every decision**.

---

## Part 2 — Usage analytics

Two CSVs to join:

**`usage_data.csv`** — one row per user per feature per day:

```
date, username, feature, sessions, time_spent, average_time_spent
2017-06-01, kwalter, Reporting, 14, 23, 1.64
```

**`user_data.csv`** — user attributes:

```
first_name, last_name, username, role, region, subscription_plan
Addison, Tucker, atucker, Sales, US-Central, free_trial
```

Note the landmines: `subscription_plan` has blanks, and the join key is `username` — which only exists cleanly on one side if you did Part 1 properly.

Typical asks: usage by feature / role / region, free-trial vs paid behavior, most and least engaged segments — presented as findings, not just tables.

---

## Takeaways

- Take-homes like this are graded on **judgment**, not code volume: how you handle nulls, dupes, and join mismatches, and whether you wrote the decisions down.
- Keep it boring and reproducible: one script or notebook, top-to-bottom runnable, short README of assumptions.
