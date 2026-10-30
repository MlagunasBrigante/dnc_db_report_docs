---
type: epic
project: dnc-cron-scrub
sprint: S2
status: to-do
gaps: [GAP-25]
---
# Epic: Historical backfill
## Description
Load the 4 container-era SQLite runs (2026-08-12, 09-01, 10-01, 10-28) into Postgres with matching row counts and checksums. Profile the 20 legacy runs in data/legacy/ (May 2025–July 2026, schemas may differ) against the stage-table contract; load what fits, report what doesn't. This history is S3's main input for statistical modeling. The backfill script is idempotent — re-running it skips already-loaded runs.

## Gaps closed
- **GAP-25** (partial — moves data off OneDrive at cutover; full retention policy in S5): About 4.1 GB of lead PII sits in OneDrive with no retention policy

## Breakdown

### INVEST check
> [!question]+
> Why is the `Rating` column missing?
>
> ✅ *Applied 2026-10-29: table now uses the same `Criterion | Rating | Notes` format as the S1 epics; "Small" dropped because the method skips it at epic level.*

| Criterion | Rating | Notes |
|---|---|---|
| Independent | ⚠️ | Depends on Epic - PostgreSQL system of record (schema must exist) but independent of scheduling/cutover |
| Negotiable | ✅ | Could load only the 4 container-era runs and defer legacy; but S3 needs the history |
| Valuable | ✅ | S3 statistical modeling depends on this history; also proves the Postgres schema handles real data |
| Estimable | ⚠️ | Story 1 (container-era) is concrete; Story 2 (profile legacy) is a learning story but time-boxed; Story 3 (load compatible) depends on Story 2's findings |
| Testable | ✅ | Row counts, checksums, FK integrity; Story 2 produces a compatibility report per run |

### Splitting pattern applied
**Pattern 9: Spike** for Story 2 (profile legacy runs) — a learning story that produces a report, not working software. Combined with **Pattern 4: Data Variations** (container-era vs legacy, which have different schemas and may need mapping).

Earlier patterns rejected:
- Pattern 1 (Workflow Steps) — backfill isn't a workflow; it's a one-time data migration
- Pattern 2 (Operations) — CRUD on Postgres isn't the split; the split is by data source (container vs legacy) and by uncertainty (known vs unknown schemas)

### Cynefin calibration
**Story 1 (container-era): Complicated** — known schema (scrub_ods.db, scrub_staging.db), known mappings, checksums verify correctness.

**Story 2 (profile legacy): Complex** — 20 runs with unknown/varying schemas, some from the notebook era (1st_run has CSV/JSON, not SQLite). The output is a report that guides Story 3, not a schema mapping we can write now.

**Story 3 (load compatible): Complicated** — once Story 2 classifies each run (loadable / needs mapping / not loadable), the load is straightforward.

### Split evaluation
- **Does it reveal low-value work?** Story 2 might reveal that most legacy runs are incompatible and not worth forcing in — that's a valuable finding (don't spend S2 time on a lossy conversion when S3 can work with partial history).
- **Are the sizes roughly equal?** Story 1 and 3 are Medium, Story 2 is Small (time-boxed); acceptable.

## Child user stories
- [[Story - Backfill the container-era runs]]
- [[Story - Profile legacy runs against the stage contract]]
- [[Story - Backfill compatible legacy runs]]

## Dependencies / sequencing
- **Upstream:** Epic - PostgreSQL system of record must land Story - All stages write to Postgres before backfill can run (schema must exist).
- **Internal:** Stories run in order (container-era → profile legacy → load compatible).
- **Downstream:** S3 (statistics) depends on this epic's history being loaded.
