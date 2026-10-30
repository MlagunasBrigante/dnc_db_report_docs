---
type: story
project: dnc-cron-scrub
epic: "[[Epic - PostgreSQL system of record]]"
sprint: S2
moscow: must
status: to-do
gaps: [GAP-18]
jira: ""
size: M
---
# Story: Raw snapshot and run record land in Postgres
## User Story
**As** Michelle (pipeline owner), **I want** each monthly run's raw LawRuler snapshot and its metadata stored in PostgreSQL with a SHA-256 checksum, **so that** I have an immutable legal record of what was ingested and can trace every downstream result back to the original data.

## Acceptance Criteria
- [ ] **Given** a fresh PostgreSQL database on the ITServer, **when** Michelle runs the provided DDL script, **then** the `pipeline_run` table and the partitioned `raw_snapshot` table are created without error (verified by `\d+ pipeline_run` and `\d+ raw_snapshot` showing the expected columns and partition structure)
- [ ] **Given** the schema is deployed, **when** run.py executes Stage 1, **then** it writes one `pipeline_run` row (run_id UUID, trigger, start_time, code_version from git SHA, raw_snapshot_checksum as SHA-256 of the fetched CSV) and all raw rows to the `raw_snapshot_YYYYMM` partition for that month (verified by row count matching the SQLite-era `raw_data` count for the same run)
- [ ] **Given** a completed run, **when** the raw_snapshot_checksum is recomputed from the stored rows, **then** it matches the checksum in `pipeline_run` (verified by a SQL query that hashes the partition's data and compares)
- [ ] **Given** the pipeline writes to Postgres, **when** the same run is re-executed (idempotent test), **then** it upserts the run row and partition rows without duplication, using `(run_id, lead_id, phone)` uniqueness (verified by stable row counts)
- [ ] **Given** Michelle needs to create the schema, **when** she reads the step-by-step guide (committed to Docs/), **then** it tells her the dedicated database/schema name, the roles to create, the DDL to run, and how to grant the pipeline's service account access (verified by Michelle's review or by a second reader executing the steps on a scratch Postgres)
- [ ] **Given** run.py connects to Postgres, **when** it starts, **then** it reads the connection string from `Dev/.env` (POSTGRES_DSN), never hardcoded (verified by grep)

## Notes
- **Database layout:** see [[Decision - Dedicated PostgreSQL database]] (dedicated `dnc` DB, roles `dnc_owner` / `dnc_writer` / `dnc_reader` / `dnc_archive`, Unix-socket connection). Pending Michelle's decision.
- **Input:** S1's per-stage inputs/outputs document defines the raw_snapshot schema (15 columns, all TEXT per the brief's "current SQLite tables" reference)
- **Partition naming:** `raw_snapshot_YYYYMM` (e.g. `raw_snapshot_202610`); the run_id foreign key ties rows to `pipeline_run`
- **Uniqueness:** `(run_id, lead_id, phone)` unique constraint per partition, not a PK (allows surrogate key `id SERIAL`)
- **Checksum:** SHA-256 of the raw CSV bytes fetched from LawRuler, stored before any cleaning/transformation
- **DDL + guide:** Committed to `Dev/schema/` (DDL) and `Docs/ITServer-setup.md` or similar (guide for Michelle)
- **Roles:** Suggest `dnc_app` (read/write), `dnc_readonly` (analyst queries), `dnc_archive` (the only role allowed to UPDATE/DELETE closed partitions in S5)

## Open questions
- [ ] #open-question Answer the two questions in the "Open questions" section of [[Epic - PostgreSQL system of record]] (existing DB contents/joins; admin rights) and settle [[Decision - Dedicated PostgreSQL database]].

## Dependencies
- S1 DNCUP-5 (per-stage inputs/outputs document) defines the raw_snapshot columns
