# Providence — Senior Data Engineer (Healthcare)

**Loop structure (typical):** recruiter screen → structured behavioral (1 hr video) → coding/technical round(s) → managerial → HR. Rounds are **practical** — SQL, warehousing, pipelines — not algorithm puzzles. 3–5 weeks end to end.

---

## Behavioral round (1 hr, structured)

Actually asked:

- Tell me about a **difficult situation** and how you handled it.
- Tell me about a time you **helped a team thrive**.
- **Why this role, why this organization?**
- How does your **education align** with this role?

Also expect questions pulled **straight from your resume** — even mid-behavioral, the interviewer may pick a listed tool (ADF, Synapse, Spark…) and ask you to walk through how you've used it.

---

## ADF — what to actually know

Enough to hold the conversation:

- **The four objects:** pipeline → activity → dataset → linked service. Be able to walk one flow through all four.
- **Integration Runtime:** Azure IR (cloud) vs self-hosted IR (on-prem/VNet sources). Healthcare loves self-hosted IR because clinical sources sit behind firewalls.
- **Triggers:** schedule vs **tumbling window** (the distinction interviewers reach for — tumbling windows are stateful, non-overlapping, support backfill and dependencies).
- **The senior-sounding answer:** metadata-driven pipelines — one parameterized pipeline + a control table, instead of 200 copy-pasted pipelines.
- **Incremental loads:** watermark column + lookup activity; idempotent re-runs.
- **Error handling:** activity retry policies, failure paths, alerts via Monitor/Log Analytics.
- **Security (healthcare will care):** managed identity over connection strings, Key Vault for secrets, private endpoints.
- Comparisons to have ready: **ADF vs Synapse pipelines**, **ADF vs Airflow**, **ADF vs Databricks workflows**.

---

## Reported technical questions (from interview-guide sites — indicative, not guaranteed)

- Rolling **7-day average of patient admissions** (window function).
- **Dedupe keeping the latest record** (`ROW_NUMBER() ... ORDER BY updated_at DESC`).
- **30-day readmission** query — the healthcare SQL classic (self-join or `LEAD` on admit dates per patient, filter gap ≤ 30 days).
- SCD Type 1 vs Type 2 — and when you'd choose each.
- Star vs snowflake schema.
- CTE vs temp table.
- "An Airflow job failed overnight — walk me through recovery."
- ETL vs ELT.
- Schema evolution handling.
- Spark partitioning strategies.

---

## Takeaways

- The technical bar is **practical DE fundamentals**, softer than big-tech loops — winnable with fundamentals + honest resume.
- Healthcare domain vocabulary helps: HL7/FHIR, EHR data, HIPAA handling, claims vs clinical data.
- **Warning about Glassdoor:** "Providence" interview data is polluted by Providence, RI employers and Providence's India campus hiring — different roles, different bar. Filter carefully.
