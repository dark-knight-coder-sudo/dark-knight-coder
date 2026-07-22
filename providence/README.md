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

## Live coding round — the actual questions (EMR / lab / genomics)

This is what was really asked, on real healthcare schemas. Two tables drive the SQL half:

- **`ORDER_RESULT`** — one row per lab test result: `patientid, encounter_id, order_id, result_time, external_name` (the test name), `ord_num_value` (the numeric result).
- **`PATIENT_ENCOUNTERS`** — one row per patient encounter: `patientid, encounter_id, encounter_type, contact_date`.

### 1.1 — Unique lab test names

```sql
SELECT DISTINCT external_name FROM ORDER_RESULT;
```

### 1.2 — Average WBC per patient

```sql
SELECT patientid, AVG(ord_num_value) AS avg_wbc
FROM ORDER_RESULT
WHERE external_name = 'WBC'
GROUP BY patientid
ORDER BY patientid;
```

### 1.3 — Identify patients with Pancytopenia

> Pancytopenia = abnormally low red cells, white cells, **and** platelets. Defined here by all three at once:
> Hgb/Hemoglobin < 8.0, Platelet Count < 50, WBC ≤ 2.0.

The pattern: **pivot lab rows into columns with conditional aggregation**, then filter in `HAVING` so all three co-occur for the same patient/encounter.

```sql
SELECT patientid, encounter_id
FROM ORDER_RESULT
GROUP BY patientid, encounter_id
HAVING MAX(CASE WHEN external_name IN ('Hgb','Hemoglobin') THEN ord_num_value END) < 8.0
   AND MAX(CASE WHEN external_name = 'Platelet Count'       THEN ord_num_value END) < 50
   AND MAX(CASE WHEN external_name = 'WBC'                  THEN ord_num_value END) <= 2.0
ORDER BY patientid;
```

### 1.4 — Same-day readmissions

> `PATIENT_ENCOUNTERS` has one row per encounter. Find patients with multiple admissions on the same day.

```sql
SELECT patientid, DATE(contact_date) AS admit_day, COUNT(*) AS admissions
FROM PATIENT_ENCOUNTERS
WHERE encounter_type = 'Inpatient'
GROUP BY patientid, DATE(contact_date)
HAVING COUNT(*) > 1;
```

### 1.5 — All labs for one patient's inpatient encounter on a date

> For `patientid = 'CBC1301104124'`, find all lab orders during the inpatient encounter on `2020-10-20`.

```sql
SELECT o.patientid, o.order_id, o.encounter_id, o.result_time,
       o.external_name, o.ord_num_value
FROM ORDER_RESULT o
JOIN PATIENT_ENCOUNTERS e ON o.encounter_id = e.encounter_id
WHERE o.patientid = 'CBC1301104124'
  AND e.encounter_type = 'Inpatient'
  AND DATE(e.contact_date) = '2020-10-20';
```

---

## Part 2 — Biomarker data mart (design / ops discussion)

The **`BIOMARKER`** table holds gene-sequencing order information. These are talk-through-it questions, not SQL:

**2.1 — A customer says they see no genomic data from a given lab (Caris) in the last two weeks. How do you confirm it?**
Query `BIOMARKER` filtered to that lab over the window and compare against the trailing baseline — e.g. count orders per day per lab for the last ~60 days, then show the last 14 are zero (or far below the lab's normal daily volume). Confirm it's lab-specific (other labs still flowing) and source-side vs pipeline-side by checking raw ingestion/landing counts against the mart. A single query with `GROUP BY lab, order_date` and a baseline comparison settles it.

**2.2 — Once you've identified the issue, how do you communicate it?**
Acknowledge to the customer with scope and impact (which lab, which date range, what's affected), open an incident, notify the data-source/vendor contact and downstream consumers, give an ETA and a workaround if any, then follow up with root cause and prevention. The point they're probing: structured, proactive comms — not silence until it's fixed.

**2.3 — Add six more labs to the existing Biomarker pipeline. Design a scalable architecture and produce a backlog with dependencies.**
Go **metadata-driven / config-per-lab**, not six copy-pasted pipelines: one parameterized ingestion framework, a config table describing each lab's format/endpoint/schema mapping, a normalization layer to a canonical biomarker schema, and per-lab data-quality checks. Backlog with dependencies:
1. Inventory the six labs' formats, delivery methods, and schemas *(blocks everything)*.
2. Design the canonical biomarker schema + mapping config *(depends on 1)*.
3. Build the parameterized ingestion + normalization framework *(depends on 2)*.
4. Onboard labs one at a time via config *(depends on 3)*.
5. Per-lab DQ rules + monitoring/alerting on volume drops *(depends on 4; this is what makes 2.1 detectable automatically)*.
6. Backfill, reconciliation, and cutover *(depends on 4)*.

---

## Python round — free-text clinical NLP

Three questions on `patients_nlp.json`, each patient a record with a free-text `note` field.

**Q1 — Parse the JSON and find NSCLC (Non-Small Cell Lung Cancer) patients from the `note` text.**

The subtlety: **"small cell lung cancer" is a substring of "non-small cell lung cancer"**, so check the NSCLC pattern first and require the `non-` prefix / the `NSCLC` acronym.

```python
import json, re

with open('patients_nlp.json') as f:
    patients = json.load(f)

NSCLC = re.compile(r'\bNSCLC\b|non[-\s]?small[-\s]cell\s+lung', re.I)
SCLC  = re.compile(r'\bSCLC\b|small[-\s]cell\s+lung', re.I)

def classify(note: str):
    if NSCLC.search(note):
        return 'NSCLC'
    if SCLC.search(note):     # reached only if not NSCLC — "non-small" already excluded
        return 'SCLC'
    return None

nsclc = [p for p in patients if classify(p['note']) == 'NSCLC']
print([p.get('patient_id', p) for p in nsclc])
```

**Q2 — Separate NSCLC and SCLC patients into two groups.**

```python
groups = {'NSCLC': [], 'SCLC': []}
for p in patients:
    label = classify(p['note'])
    if label:
        groups[label].append(p)
```

**Q3 — The `note` also holds vitals. Extract resting heart rate and find the mean resting heart rate of NSCLC patients.**

```python
HR = re.compile(r'resting\s+heart\s+rate[:\s]*?(\d+)', re.I)

rates = []
for p in patients:
    if classify(p['note']) == 'NSCLC':
        m = HR.search(p['note'])
        if m:
            rates.append(int(m.group(1)))

mean_hr = sum(rates) / len(rates) if rates else 0
print(f"Mean resting HR (NSCLC): {mean_hr:.1f}")
```

What the Python half tests: JSON parsing, careful **regex on messy clinical free text** (the NSCLC-vs-SCLC ordering trap is the whole point of Q1/Q2), and a guarded aggregation for the mean.

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
