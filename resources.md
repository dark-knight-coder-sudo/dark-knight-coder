# Resources worth reading

Curated — these are the ones that actually helped prep for the loops in this repo.

## Company-specific guides

- [Dataford — Providence Data Engineer interview guide](https://www.dataford.io) — loop structure and question themes for Providence DE.
- [InterviewQuery — Providence Data Engineer questions](https://www.interviewquery.com) — practical SQL/pipeline question bank.
- [Glassdoor — Providence interview reviews](https://www.glassdoor.com) — ⚠️ **filter carefully**: "Providence" results mix in Providence, RI employers and Providence's India campus hiring, which have a completely different bar.

## Azure Data Factory (kept coming up in healthcare loops)

- [DataCamp — ADF interview questions](https://www.datacamp.com/blog/azure-data-factory-interview-questions) — covers the pipeline/activity/dataset/linked-service model and IR types.
- [InterviewBit — ADF interview questions](https://www.interviewbit.com/azure-data-factory-interview-questions/) — good on triggers (schedule vs tumbling window) and incremental load patterns.

## The drill list that pays off across every DE loop

- Window functions: `LAG`/`LEAD`, `ROW_NUMBER`, rolling averages, dedupe-keep-latest.
- The healthcare classic: 30-day readmission.
- Python: flatten nested JSON, anagrams, valid parentheses, stride iteration over flat lists — **with big-O narration out loud**.
- Concepts: SCD1 vs SCD2, star vs snowflake, CTE vs temp table, ETL vs ELT, schema evolution, Spark partitioning, "your overnight job failed — recover."
