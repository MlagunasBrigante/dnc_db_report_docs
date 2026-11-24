---
type: story
project: dnc-cron-scrub
epic: "[[Epic - PostgreSQL system of record]]"
sprint: S2
moscow: must
s2_order: 2
status: merged — awaiting remaining ACs (3, 4, 6, 7, 8)
gaps: [GAP-40]
jira: ""
size: M
---
# Story: All stages write to Postgres with lineage
> [!todo] `<owner>` — action needed
> - [x] Push `main` ✅ 2026-11-24: merged via PR #3 (`4979639`; direct push is refused by the branch rule, PR merge works). Delete the merged branch `s2/feat/all-stages-postgres` when convenient.
> - [ ] Decide before 2026-12-01: recreate `dnc-scrub` on the ITServer with the order-2 image (the 2026-12-01 run then writes all stage tables to `dnc`), or keep the current image until the remaining ACs close. #open-question
> - [ ] ITServer clone back on `main` (step 7 of the order-2 test commands).

## User Story
**As** `<owner>` (pipeline owner), **I want** every pipeline stage (cleaned phones, scrub batches/results, interpreted results, tag updates) to write to Postgres with foreign keys back to the raw snapshot, **so that** I can trace any lead's final tag back through every transformation to the original LawRuler row.

## Acceptance Criteria
- [x] **Given** the raw_snapshot schema from Story - Raw snapshot exists, **when** run.py executes Stages 2–4, **then** it writes to partitioned tables `cleaned_phones_YYYYMM`, `scrub_batches_YYYYMM`, `scrub_results_YYYYMM`, `scrub_results_translated_YYYYMM`, `leads_with_tags_YYYYMM`, all partitioned by run month (verified by `\d+` showing partitions and FKs) ✅ 2026-11-24: ITServer `verify_all_stages.sql` on `dnc_test` shows every table partitioned by `snapshot_month` with `_202611` partitions and FKs (plus `tag_updates`).
- [x] **Given** the tables are created, **when** a row is inserted into `scrub_results`, **then** its `cleaned_phone_id` FK references `cleaned_phones(id)`, and `cleaned_phones` has a foreign key to `raw_snapshot` by `(snapshot_month, run_id, lead_id, phone)`, forming a complete lineage chain (verified by `\d scrub_results` showing FK constraints and a test query joining through the chain) ✅ 2026-11-24, as built: `cleaned_phones → raw_snapshot` (via `raw_phone`), `scrub_results_translated → scrub_results` (`scrub_result_id`), `leads_with_tags → cleaned_phones` (via `cleaned_phone`) and `→ scrub_results_translated`, `tag_updates → leads_with_tags`. Orphan checks 0; the end-to-end join returns 1,001 chains for the latest dry-run. Unlinked rows are expected, see GAP-40.
- [ ] **Given** Stage 2 scrub fails for some batches (simulated by injecting an API error), **when** the run completes, **then** `scrub_batches` records both successful and failed batches, and `scrub_results` rows have NULLable FKs (no FK to scrub_results if the phone wasn't scrubbed) — this handles GAP-10 silent skips (verified by querying `scrub_batches WHERE status='failed'` and `scrub_results WHERE scrub_result_id IS NULL`)
- [ ] **Given** all stages write to Postgres, **when** the Stage 5 report generation reads from `leads_with_tags_YYYYMM`, **then** the row counts and sample lead IDs match the SQLite-era `leads_with_tags` table for the same run (verified by side-by-side diff of the 2026-11-21 run: its SQLite tables vs the same run loaded into `dnc_test` with this story's copy code, D-36) — 2026-11-24: **row counts match on all 8 tables** (see Notes); still open: the lead-ID set comparison, and Stage 5 still reads SQLite (copy pattern, D-36; reading from Postgres is [[Story - Report generated from Postgres]])
- [x] **Given** a run writes to multiple partitioned tables, **when** each table's DDL is reviewed, **then** every table has `(run_id, lead_id, phone)` uniqueness where applicable and a `created_at TIMESTAMPTZ DEFAULT NOW()` audit column (verified by inspection of Dev/schema/*.sql) ✅ 2026-11-24: `Dev/schema/002_stage_tables.sql`; uniqueness checks on the ITServer return 0 duplicates.
- [ ] **Given** the scrub_results table stores the raw JSON from dncscrub.com, **when** a scrub_results row is inserted, **then** the `response_json` column (JSONB) holds the full API response unchanged, preserving legal traceability (verified by querying `response_json->>'resultCode'` and comparing to the SQLite-era `scrub_results.response_json`)
- [ ] **Given** S2 (Postgres), **when** the migration lands, **then** the per-stage metrics rows (same columns as `data/metrics/memory_log.csv`, see `dnc/memory_log.py`) are written to a `run_stage_metrics` table, the existing CSV rows are loaded into it, and the CSV can be retired. — 2026-11-24: rows written (6 per run, dry-run `2ef4e626` on the ITServer); still open: loading the CSV history and retiring the CSV (Moved from [[Story - Memory log with quarterly review]], 2026-11-10.)
- [ ] **Given** all stages write to Postgres, **when** the migration lands, **then** `Docs/Reference/pipeline-stages-reference.md` is updated: SQLite tables, ATTACH and file paths are replaced by the Postgres tables (its *S2 Mapping* section is the starting point), and the runbook is checked for the same. (`<owner>`, 2026-11-10.)

## Notes
- **Input:** `Docs/Reference/pipeline-stages-reference.md` (per-stage inputs/outputs, formerly S1 DNCUP-5) defines the schema for all stage tables
- **Per-run uniqueness:** `(run_id, lead_id, phone)` prevents duplicates within a run but allows the same lead to appear in different monthly runs
- **FK strategy:** Surrogate keys (`id SERIAL`) as PKs for downstream tables; `cleaned_phones` references `raw_snapshot` by the composite key `(snapshot_month, run_id, lead_id, phone)` (FK to the partitioned parent, allowed since PG12); other stage tables reference parent stage's `id`, plus `run_id` for partition pruning. Per D-24 (2026-11-13): any unique key/PK on a partitioned table must include its partition column.
- **NULLable FKs:** Where upstream stages can fail (e.g. a lead with no scrub result because the batch failed), the FK is `NULL` — this is legal and expected
- **Partition creation:** settled by order 1: a SECURITY DEFINER `ensure_<table>_partition(month)` function, called by the run (pattern of `ensure_raw_snapshot_partition`, `Dev/schema/001_raw_snapshot.sql`). Originally: either auto-created by Postgres 10+ declarative partitioning (CREATE TABLE ... PARTITION OF) or pre-created by a migration script for each month; document the approach in Docs/
- **`pipeline_run.status` / `finished_at` are never written yet** (2026-11-13 branch review of `s2/feat/raw-snapshot-postgres`, finding #13): the raw-snapshot story creates both columns (`Dev/schema/001_raw_snapshot.sql`) but only Stage 1 writes to `pipeline_run`, so they stay NULL and the `status` column in `schema/verify_raw_snapshot.sql` is always empty. Set them when the run's final stage completes or fails, as part of this story.

## Decisions (2026-11-23, D-36)
- **Copy pattern:** SQLite still drives stages 2–4; after each stage its results are copied into Postgres (as Stage 1 does). A copy failure is logged and alerted; the run continues. SQLite is retired at cutover (order 13).
- **Grants:** `dnc_writer` gets INSERT/SELECT only on every new table (no UPDATE/DELETE; nothing is modified mid-run). `pipeline_run` keeps UPDATE for `status`/`finished_at`.
- **Fixture loader:** the copy code must also run against an existing run's SQLite files (needed to load 2026-11-21 into `dnc_test` for verification).

## Delivered (2026-11-24, merged 2026-11-24 via PR #3 as `4979639`, code `ca85ebd`)
- **Schema:** `Dev/schema/002_stage_tables.sql` adds `cleaned_phones`, `scrub_batches`, `scrub_results`, `scrub_results_translated`, `leads_with_tags`, `tag_updates`, `run_stage_metrics`, each with an `ensure_<table>_partition()` function and INSERT/SELECT-only grants for `dnc_writer` (D-36). Phone-format links: `raw_snapshot`/`leads_with_tags` store `+1` and 10 digits, `cleaned_phones`/`scrub_results_translated` 10 digits; the nullable link columns `raw_phone`/`cleaned_phone` are resolved at insert.
- **Copy code:** `dnc/pg_store.py` copies each stage's SQLite output by COPY into a temp table, then INSERT…SELECT; temp tables are analyzed before joining. A copy failure is logged and alerted as a warning and the run continues (D-36). Failures in the writes that only happen after Stage 6 (run status, stage metrics) send a short follow-up warning email + Teams alert. `pipeline_run.status`/`finished_at` are now set at the end of the run (closes the 2026-11-13 finding #13 note).
- **Fixture loader:** `Dev/tools/load_sqlite_fixture.py --run-date <date> --data-dir /data --target-schema dnc_test` loads an existing run's SQLite files; it keeps its run id in a temp dir (the data mount is read-only) and exits non-zero if any stage fails.
- **Verification:** `Dev/schema/verify_all_stages.sql` (structure, row counts per run, lineage, orphans, uniqueness, stage metrics; no lead rows printed).

## ITServer verification (2026-11-24)
| Table | `dnc_test` (fixture `a0afc7ba`) | SQLite 2026-11-21 |
|---|---|---|
| raw_snapshot | 428,864 | 428,864 |
| cleaned_phones | 362,261 | 362,261 |
| scrub_batches / scrub_results | 363 / 363 | 363 / 363 |
| scrub_results_translated | 358,698 | 358,698 |
| leads_with_tags | 362,181 | 362,181 |
| tag_updates | 362,181 | 362,181 |

Load took 51 s. Dry-run `2ef4e626` finished `success`: stages 1–4 copied, run status and 6 stage-metric rows written, no warnings. Earlier partial runs stay in `dnc_test` (`2dc59a49`, stages 1–2 only; `2a2b3bbb` from order 1): test data, harmless.

**Found on the ITServer during the test and fixed on the branch** (none reached production): `main()` crash from a shadowed `datetime` import; loader writing into the read-only run folder and still reporting success; loader import order; stage metrics sending `HH:MM:SS` instead of full timestamps; late Postgres-write failures not alerted; a per-row Stage 3 lookup that never finished on a full run (replaced by one grouped join); the verify script printing lead rows.

**Remaining before `done`:** AC3 (fault-injected failed batch), AC4 lead-ID set comparison, AC6 (`response_json` compared with SQLite), AC7 (load the CSV history, retire the CSV), AC8 (`Docs/Reference/pipeline-stages-reference.md`, runbook and `Dev/README.md` describe the new tables, loader and verify script). GAP-40 can be fixed in the same pass.

## Dependencies
- [[Story - Raw snapshot and run record land in Postgres]] (raw_snapshot and pipeline_run must exist)
- `Docs/Reference/pipeline-stages-reference.md` (per-stage inputs/outputs; formerly S1 DNCUP-5)
