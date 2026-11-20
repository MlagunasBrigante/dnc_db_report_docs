---
type: story
project: dnc-cron-scrub
epic: "[[Epic - PostgreSQL system of record]]"
sprint: S2
moscow: must
s2_order: 1
status: in-review
gaps: [GAP-18, GAP-34]
jira: ""
size: M
---
# Story: Raw snapshot and run record land in Postgres
> [!todo] `<owner>` — action needed
> - [x] Answer the two database questions in the epic and settle the dedicated-DB decision. (2026-11-10, D-18)
> - [x] Create the database objects on the ITServer using the DDL guide. (2026-11-18, guide Steps 1–9)

## User Story
**As** `<owner>` (pipeline owner), **I want** each monthly run's raw LawRuler snapshot and its metadata stored in PostgreSQL with a SHA-256 checksum, **so that** I have an immutable legal record of what was ingested and can trace every downstream result back to the original data.

## Acceptance Criteria
- [x] **Given** a fresh PostgreSQL database on the ITServer, **when** `<owner>` runs the provided DDL script, **then** the `pipeline_run` table and the partitioned `raw_snapshot` table are created in both `dnc` and `dnc_test` schemas without error (verified by `\d+ pipeline_run` and `\d+ raw_snapshot` in each schema showing the expected columns, the composite primary key `(snapshot_month, run_id, lead_id, phone)`, and partition structure, and by querying the partition list)
- [x] **Given** the schema is deployed, **when** run.py executes Stage 1 on the ITServer as a dry-run (forced into dnc_test per D-19), **then** it writes one `pipeline_run` row (run_id UUID, trigger, start_time, code_version from git SHA, raw_snapshot_checksum per D-21(b)) and all raw rows to the `raw_snapshot_YYYYMM` partition for that month with lead_id, phone, and the full row as JSONB payload (verified by row count matching the SQLite-era `raw_data` count for the same run)
- [x] **Given** a completed run, **when** `Dev/tools/verify_raw_snapshot.py` recomputes the canonical checksum from the stored rows, **then** it matches the checksum in `pipeline_run` (verified by the script's exit code and output)
- [ ] **Given** the pipeline writes to Postgres, **when** the same run folder is resumed and that run already has a stored snapshot (checksum recorded in pipeline_run), **then** the raw write is skipped and pipeline_run's started_at, row count and checksum are unchanged (verified by log message and stable values); **and given** a previous attempt failed mid-write, **then** the snapshot is written in full and verifies (D-21(d))
- [ ] **Given** `<owner>` needs to create the schema, **when** she reads the step-by-step guide [[ITServer PostgreSQL setup]], **then** it tells her the dedicated database name, the roles to create, the DDL to run, how to create the test schema, and how to grant the pipeline's service account access (verified by `<owner>`'s review or by a second reader executing the steps on a scratch Postgres)
- [x] **Given** run.py connects to Postgres, **when** it starts, **then** it reads the connection string from `Dev/.env` (POSTGRES_DSN), never hardcoded (verified by grep)
- [x] **Given** POSTGRES_DSN is unset (the Mac, D-19), **when** a run executes, **then** it behaves exactly as before and imports no Postgres driver (verified by unit test)

## Notes
- **In review 2026-11-11.** Branch `s2/feat/raw-snapshot-postgres` (11 commits, tip `c6a764f`): `Dev/schema/` (000 roles + database, 001 per-schema DDL, verify SQL), `dnc/pg_store.py`, Stage 1 hook (only when `POSTGRES_DSN` is set), `tools/verify_raw_snapshot.py`, 349 sandbox tests pass, trial merge into `main` clean. Review fixes: COPY via a staging table + `ON CONFLICT DO NOTHING` (resume), no identity column on the partitioned table (PG17+ only), schema derived with `current_schema()` inside function bodies, unique index on the parent, idempotent FK, streamed read-only SQLite read with incremental checksum (no full-history copy in memory), one ERROR per failure, schema allow-list (`dnc`/`dnc_test`), resume keeps `started_at`. SQL not executed yet (no Postgres in the sandbox): AC1–AC5 need `<owner>`'s ITServer run; AC6–AC7 are code/unit-test evidence. Guide: [[ITServer PostgreSQL setup]].
- **Branch review 2026-11-12/13** (interview with `<owner>`; tip now `8415677`, 19 commits; 355 sandbox tests pass, excluding `test_stage4_lean_working_set.py`, which can't import numpy in the sandbox and first runs on the Mac; trial merge into `main` @ `399841a` clean on 2026-11-13, README auto-merged, 355 pass on the merge result). Outcomes:
  - **#1 fixed** (`c2b2db1`, `cbdd5b9`, `0d78f3b`): an invalid `POSTGRES_SCHEMA` raised outside the non-fatal `try` and would have failed a real run (dry-runs force `dnc_test`, so they'd never catch it). The Stage 1 hook moved to `dnc/pg_store.py` `write_raw_snapshot_from_sqlite` (all of it inside the `try`, SQLite connection closed in `finally`); four tests call the real function, mutation-checked.
  - **#2 fixed** (`07ece06`): README table rows starting `||` and an absolute sandbox path to the setup guide.
  - **#3 and #9 → GAP-34** (open question below): checksum order dependency and verify-tool memory; robust fix to be discussed.
  - **#4 no change:** no NULL `Lead ID`/`Phone` in real data (2026-11-02, 2026-11-10); ≈570 phone-less leads per run stay in the snapshot as received.
  - **#5 fixed, D-22** (`48ce537`): FK no longer cascades on delete; `dnc_archive` SELECT only until S5.
  - **#6 fixed with #1** (SQLite connection closed on failure).
  - **#7 no change** (`<owner>`): the INFO "Postgres writing disabled" line and the "no new log noise" wording stay; the Mac only runs tests from now on.
  - **#8 test added** (`8415677`): importing `dnc.pg_store` in a fresh process doesn't load psycopg (AC7), mutation-checked with a stand-in psycopg.
  - **#10:** this note.
  - **#11–#13** (`e73565a`, `86ce5c1`): verify-script header prints the schema name; PII warning on its sample query; sandbox-path comment in `tools/backfill_memory_log.py`; `pipeline_run.status`/`finished_at` noted in [[Story - All stages write to Postgres with lineage]].
  - **Independent review** (fresh subagent, 2026-11-13): no critical/high findings; f-string schema hardening folded into GAP-34, the rest declined or cosmetic. Branch tip now `86ce5c1` (21 commits).
- **ITServer facts (2026-11-13):** 12 GiB RAM (~10 GiB available), 5.8 GiB swap (1.0 GiB used); `/` 70 GB (29 GB free), `/home` 224 GB (221 GB free); Docker Engine 29.3.0, Compose v5.1.0, daemon reachable as it.admin; existing PostgreSQL 13 socket at host /var/run/postgresql.
- **Database layout:** see [[Decision - Dedicated PostgreSQL database]] (dedicated `dnc` DB, roles `dnc_owner` / `dnc_writer` / `dnc_reader` / `dnc_archive`, Unix-socket connection). Accepted 2026-11-10 (D-18); `<owner>` creates the DB and roles from the guide.
- **Partition naming:** `raw_snapshot_YYYYMM` (e.g. `raw_snapshot_202610`); the run_id foreign key ties rows to `pipeline_run`
- **Uniqueness:** Composite primary key `(snapshot_month, run_id, lead_id, phone)` per D-24 (2026-11-13); lead_id and phone use COLLATE "C" so PK order = Python string order, a prerequisite for a streaming checksum verify under GAP-34.
- **DDL + guide:** Committed to `Dev/schema/` (DDL) and [[ITServer PostgreSQL setup]] (guide for `<owner>`)
- **D-19 (2026-11-11):** the Mac job never writes to Postgres; with `POSTGRES_DSN` unset the pipeline runs exactly as today. Postgres writing is turned on during the migration and tested on the ITServer against the test schema in the dedicated `dnc` database first. Target server: **PostgreSQL 13.23**, so the DDL and driver must work on 13 (no PG14+ syntax). Target amended by D-23 (2026-11-13): PostgreSQL 17 in its own container; the PG13-only restrictions no longer apply.
- **D-21 (2026-11-11):** Raw snapshot row format: `lead_id`, `phone` columns + full LawRuler row as JSONB `payload` (survives schema changes upstream). Checksum: SHA-256 of canonical serialization of stored rows (sorted by lead_id, phone), recomputable by `Dev/tools/verify_raw_snapshot.py`. Test target: schema `dnc_test` in the `dnc` database with identical DDL, selected by `POSTGRES_SCHEMA`; dry-runs forced into `dnc_test`. Run identity: every execution is its own run (new run_id, full snapshot); a resume of the same run folder reuses the stored run_id; if that run already has a stored snapshot (checksum recorded in pipeline_run), the raw write is skipped; otherwise the whole snapshot is written in one transaction (a failure rolls it back, so a partial snapshot can't exist). (d) amended 2026-11-13 (`<owner>`): skip-if-stored replaces ON CONFLICT DO NOTHING. Uniqueness key (snapshot_month, run_id, lead_id, phone) — the partition key must be part of any unique key on a partitioned table; key design under review (see Uniqueness note above). Implementation calls (Claude, 2026-11-11; not `<owner>`'s decisions): partitions created by a SECURITY DEFINER function owned by dnc_owner (so dnc_writer needs no CREATE); bulk load via psycopg 3 COPY (bounded memory); a Postgres write failure logs one ERROR and the run continues (SQLite stays the system of record during S2, D-19).
- **Implementation calls (Claude, 2026-11-13; not `<owner>`'s decisions), PostgreSQL 17 container:** data directory on `/home` (221 GB free), not `/` (29 GB free, also holds Docker images); server socket shared through the Docker named volume `dnc-pgsocket`, never the host's `/var/run/postgresql` (that is the PostgreSQL 13 server's socket); TCP disabled (`listen_addresses=''`), no port published, so no clash with the 13 server on 5432; image pinned to `postgres:17.12` (corrected 2026-11-17 from 17.11: 17.12 was already current, released 2026-11-12); socket (local) auth set to scram-sha-256 at initdb, because the official image trusts local connections by default and any container mounting the socket volume could otherwise connect as any role without a password; superuser password from `POSTGRES_PASSWORD_FILE`, never `-e POSTGRES_PASSWORD`; memory `shared_buffers=512MB`, `maintenance_work_mem=256MB`, `work_mem=16MB`, `max_connections=20`, `--shm-size=256m`; sizing estimate ~0.3–0.5 GB per stored run, ~4–6 GB/year (estimate, not measured). Details: [[ITServer PostgreSQL setup]].
- **D-22 (2026-11-12):** the legal raw snapshot can't be deleted as a side effect: the `raw_snapshot → pipeline_run` foreign key no longer cascades on delete (default NO ACTION), and `dnc_archive` is read-only (SELECT) until its detach/export capability is designed in S5 / [[Story - Closed months become read-only]] (e.g. a SECURITY DEFINER function owned by `dnc_owner`; `DETACH PARTITION` needs table ownership, not UPDATE/DELETE). `dnc_writer` keeps UPDATE on `pipeline_run` for the post-stream count and checksum.
- **D-23 (2026-11-13):** PostgreSQL 17 in its own Docker container on the ITServer (not the existing PG13 server); amends D-19 and D-18.
- **D-24 (2026-11-13):** raw_snapshot primary key = composite `(snapshot_month, run_id, lead_id, phone)`, no surrogate id (D-24); one index instead of two; downstream tables small.
- **D-26 (2026-11-19):** the ITServer `.env` uses the same LawRuler, O365 and dncscrub accounts as the Mac for now (typed by hand, `chmod 600`, never committed). D-20 and R2 stay open: the LawRuler user decision is still due at the end of S2. #open-question
- **ITServer setup progress (2026-11-18/19):** guide Steps 1–9 done and verified (Step 9 structure output matches D-24/D-25 design, both schemas); Step 10 `.env` in progress.
- **Roles:** use the decision note's layout (`dnc_owner`, `dnc_writer`, `dnc_reader`, `dnc_archive`); the earlier `dnc_app`/`dnc_readonly` names are superseded.
- **Branch:** Merged to main: s2/feat/raw-snapshot-postgres (1bdd6b9) and s2/feat/raw-snapshot-pg17 (d6b7454; main 04d42b0, 2026-11-17). GAP-34 follow-up on s2/fix/gap-34-streaming-checksum (in progress).

- **ITServer verification (2026-11-19):**
  - AC1: DDL applied to `dnc` and `dnc_test` (guide Steps 6–8); Step 9 `\d+` output shows the expected columns, composite PK, `C` collation and LIST partitioning in both schemas.
  - First dry-run found a real bug: psycopg 3 returns `uuid.UUID` for `RETURNING run_id`, so `write_run_id` failed (`'UUID' + 'str'`). Non-fatal; the whole transaction (partition + run row) rolled back. Fixed in d0f3457, merged via PR as b35a022 (`main` on GitHub is now PR-only).
  - AC2: re-run on b35a022 wrote 428,641 rows to `dnc_test.raw_snapshot_202611` in ~9 s; `pipeline_run` row: trigger `dry-run`, `is_dry_run` t, `code_version` b35a022, `snapshot_month` 2026-11-01, `raw_row_count` = actual rows = 428,641.
  - AC3: `tools/verify_raw_snapshot.py --schema dnc_test` recomputed the SHA-256 (`851bcb8b…`) and row count from the stored rows: MATCH, exit 0. Confirms D-25 (Postgres `C` order = Python order, JSONB round-trip byte-exact).
  - AC6: no DSN hardcoded in code (grep); read from `.env`. AC7: covered by unit tests (`tests/test_pg_store.py`, psycopg not imported when POSTGRES_DSN is unset).
  - Open: AC4 (skip-if-stored on the real server; dry-runs always get a new folder, so it is exercised directly against the stored run) and AC5 (guide pass for host git, sudo, SELinux note).

## Open questions
- [x] #open-question Answer the two questions in the "Open questions" section of [[Epic - PostgreSQL system of record]] (existing DB contents/joins; admin rights) and settle [[Decision - Dedicated PostgreSQL database]].
- [x] #open-question **Uniqueness/key design for raw_snapshot?** ✅ 2026-11-13: composite primary key `(snapshot_month, run_id, lead_id, phone)`, no surrogate id (D-24). One index instead of two; the key is the legal fact; downstream tables small so 4-column FKs are cheap.
- [x] #open-question **How should the raw-snapshot checksum become one streaming function shared by the writer and the verify tool (GAP-34)?** ✅ 2026-11-17: option A — one shared streaming checksum function used by the writer and the verify tool; it enforces strictly ascending (lead_id, phone) order and fails loudly on out-of-order or duplicate keys; the verify tool reads the run through a server-side cursor ordered by lead_id, phone (columns COLLATE "C", so database order = Python order), flat memory; schema names composed with psycopg.sql.Identifier (D-25). D-21(b) unchanged. Rejected: an order-independent combined hash (redefines D-21(b), weaker against deliberate tampering for a legal record). In progress on branch s2/fix/gap-34-streaming-checksum.

## Dependencies
- S1 DNCUP-5 (per-stage inputs/outputs document) defines the raw_snapshot columns
