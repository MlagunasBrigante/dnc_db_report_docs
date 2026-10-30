---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Historical backfill]]"
sprint: S2
moscow: must
status: to-do
gaps: []
jira: ""
size: M
---
# Story: Backfill the container-era runs
## User Story
**As** Michelle (data owner), **I want** the 4 container-era SQLite runs (2026-08-12, 09-01, 10-01, 10-28) loaded into Postgres with matching row counts and checksums, **so that** the historical data is preserved in the new system of record before cutover.

## Acceptance Criteria
- [ ] **Given** the Postgres schema exists, **when** the backfill script `Dev/backfill.py --source data/2026-10-28` is executed, **then** it loads `scrub_ods.db` (raw_data → raw_snapshot_202610) and `scrub_staging.db` (all stage tables → corresponding Postgres partitions), creates a `pipeline_run` row with `trigger='backfill'`, and computes the raw_snapshot_checksum (verified by querying `pipeline_run` and checking `trigger='backfill'`)
- [ ] **Given** the 2026-10-28 run is backfilled, **when** row counts are compared, **then** Postgres `raw_snapshot_202610` has 421,776 rows (matching SQLite `raw_data`), `scrub_results_202610` has 7,768 rows, `leads_with_tags_202610` has 7,713 rows — all matching the SQLite tables (verified by `SELECT COUNT(*) FROM ...` for each table and comparing to SQLite)
- [ ] **Given** the backfill script is run twice on the same run (idempotency test), **when** the second execution detects an existing `pipeline_run` row for that date, **then** it logs "Run 2026-10-28 already backfilled, skipping" and exits 0 without inserting duplicate rows (verified by stable row counts and `pipeline_run` having one row per run)
- [ ] **Given** all 4 container-era runs are backfilled (08-12, 09-01, 10-01, 10-28), **when** the Postgres database is inspected, **then** `pipeline_run` has 4 rows with `trigger='backfill'`, and partitions exist for 202608, 202609, 202610 (verified by querying `pipeline_run` and `\d+ raw_snapshot*`)
- [ ] **Given** the backfill script computes checksums, **when** the SHA-256 of the backfilled raw_snapshot rows is recomputed, **then** it matches the `raw_snapshot_checksum` in `pipeline_run` (verified by a SQL query or Python script that hashes the partition's data and compares)
- [ ] **Given** the backfill script handles missing or corrupted SQLite files, **when** a .db file is not found or unreadable, **then** the script logs the error and exits non-zero (verified by renaming a .db file and running the backfill)

## Notes
- **Backfill script:** `Dev/backfill.py` takes `--source data/YYYY-MM-DD`, reads the SQLite DBs, and writes to Postgres using the same schema as run.py
- **run_id generation:** Backfilled runs get a deterministic UUID (e.g. UUID v5 from the run date) so re-running the backfill produces the same run_id
- **Checksum:** For backfilled runs, the checksum is computed from the SQLite `raw_data` table (CSV format if available, else a hash of the table's rows)
- **Foreign keys:** The backfill script must populate FKs correctly (e.g. scrub_results.cleaned_phone_id references cleaned_phones.id); test by checking FK constraint violations
- **Idempotency:** The script checks `SELECT COUNT(*) FROM pipeline_run WHERE trigger='backfill' AND start_time::date = '<YYYY-MM-DD>'` before inserting

## Dependencies
- [[Story - All stages write to Postgres with lineage]] (Postgres schema must exist)
