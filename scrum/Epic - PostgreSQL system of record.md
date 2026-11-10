---
type: epic
project: dnc-cron-scrub
sprint: S2
status: to-do
gaps: [GAP-08, GAP-18]
---
# Epic: PostgreSQL system of record
## Description
Move the pipeline's data layer from SQLite to PostgreSQL on the ITServer. Each stage's results are written to partitioned tables (one partition per monthly run), with foreign keys tracing every row back through the stages to the original raw snapshot. Closed months become read-only to protect legal traceability. Michelle creates the database objects on the ITServer's existing PostgreSQL; this epic defines the DDL and integration.

## Gaps closed
- **GAP-18** (S1 closed this in run.py; S2 adjusts schema to avoid re-reading): Full lead history held twice in memory — the Postgres schema avoids the Stage 4 full re-read by querying only the necessary columns and joining through FKs instead of re-ingesting raw_data
- **GAP-08** (partial — recording is fixed; patch pass comes in Epic - Failures captured and patched): Failed tagging requests recorded as DONE — S1 fixed the status recording; S2 adds the patch pass

## Key decisions
- [[Decision - Dedicated PostgreSQL database]]: dedicated `dnc` database vs tables in the existing DB; role layout and socket connection (proposed, awaiting Michelle)

## Open questions (to answer before [[Story - Raw snapshot and run record land in Postgres]] starts)
- [x] #open-question **What does the existing database on the ITServer hold, and will S3/S4 need to join DNC data with it?** ✅ 2026-11-10: no joins needed → dedicated `dnc` database (D-18). If no, the dedicated `dnc` database wins outright. If yes, weigh the cross-database joins against the isolation benefits. See [[Decision - Dedicated PostgreSQL database]].
- [x] #open-question **Does Michelle have admin rights (`CREATEDB` / `CREATEROLE`) on that PostgreSQL server?** ✅ 2026-11-10: the database isn't created yet; Michelle creates it and adds the required roles herself, from the DDL guide. Needed to create the database or schema and the `dnc_*` roles under either option.

- [x] #open-question **D-04: rewrite the process in pure n8n?** ✅ 2026-11-10: no; Python + Docker + PostgreSQL. Decide before S2 starts. It changes how this whole sprint is built (Python + Postgres vs n8n workflows).

## Breakdown

### INVEST check
> [!question]+
> Why is the `Rating` column missing?
>
> ✅ *Applied 2026-10-29: table now uses the same `Criterion | Rating | Notes` format as the S1 epics; "Small" dropped because the method skips it at epic level.*

| Criterion | Rating | Notes |
|---|---|---|
| Independent | ⚠️ | Depends on S1 gaps closed (alerts, logging, secrets) but independent of other S2 epics |
| Negotiable | ✅ | Michelle deferred cutover acceptance criteria; Postgres could start with fewer stages |
| Valuable | ✅ | Legal traceability (immutable closed months) + observability (query across runs) |
| Estimable | ✅ | 4 stories, each is concrete DDL + migration + verification |
| Testable | ✅ | Row counts, checksums, FK integrity, read-only verification are all concrete |

### Splitting pattern applied
**Pattern 6: Major Effort** — the first stage (raw snapshot + run record) end-to-end, then extend to all stages, then add read-only enforcement, then verify report generation from the new store.

Earlier patterns rejected:
- Pattern 1 (Workflow Steps) would create 6 stage-table stories, each too small and none delivering observable value alone
- Pattern 2 (Operations) — CRUD on Postgres is not a user-facing operation split
- Pattern 7 (Simple/Complex) does not apply; every stage table has similar complexity

### Cynefin calibration
**Complicated** — known tech (Postgres, partitioning, FKs), clear requirements (one table per stage, per-run uniqueness, lineage FKs), but non-trivial: Michelle must create the DB objects, partition management is new to the project, and the schema must accommodate the flaky LawRuler API's failure modes (NULLable FKs where upstream stages fail).

### Split evaluation
- **Does it reveal low-value work?** No — legal traceability and queryability across runs are both requirements; no story is optional.
- **Are the sizes roughly equal?** Stories 1–2 are Medium, 3–4 are Small; acceptable imbalance driven by the first story's bootstrap cost (DDL + guide for Michelle).

> [!check] Before closing this epic, verify Michelle's actions were actually done
> - [x] Answer the two database questions in the epic and settle the dedicated-DB decision. (2026-11-10, D-18) — [[Story - Raw snapshot and run record land in Postgres]]
> - [ ] Create the database objects on the ITServer using the DDL guide. — [[Story - Raw snapshot and run record land in Postgres]]

## Child user stories
- [[Story - Raw snapshot and run record land in Postgres]]
- [[Story - All stages write to Postgres with lineage]]
- [[Story - Closed months become read-only]]
- [[Story - Report generated from Postgres]]

## Dependencies / sequencing
- **Upstream:** S1 must close GAP-01 through GAP-07 (alerts, secrets, logging) before S2 starts, so the code is stable.
- **Internal:** Stories run in order (raw → all stages → read-only → report).
- **Downstream:** Epic - Reproducible scheduling on the ITServer and Epic - Cutover to the ITServer both depend on this epic being done.
