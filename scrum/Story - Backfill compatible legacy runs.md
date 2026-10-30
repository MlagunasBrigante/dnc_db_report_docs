---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Historical backfill]]"
sprint: S2
moscow: must
status: to-do
gaps: [GAP-25]
jira: ""
size: M
---
# Story: Backfill compatible legacy runs
## User Story
**As** Michelle (data owner), **I want** the legacy runs marked 'loadable' or 'needs_mapping' in the compatibility report to be loaded into Postgres, **so that** S3 has as much historical data as possible for statistical modeling.

## Acceptance Criteria
- [ ] **Given** the compatibility report from Story - Profile legacy runs, **when** the report shows N runs as 'loadable', **then** the backfill script `Dev/backfill.py --legacy` loads those N runs into Postgres with row counts and checksums matching the source (verified by querying `pipeline_run WHERE trigger='legacy_backfill'` and counting rows)
- [ ] **Given** a legacy run is marked 'needs_mapping' (e.g. 1st_run with CSV/JSON instead of SQLite), **when** the backfill script processes it, **then** it applies the documented mapping (e.g. reads `results/*.json` and transforms to scrub_results_YYYYMM schema) and loads it (verified by querying the partition and spot-checking a few rows against the source JSON)
- [ ] **Given** a legacy run is marked 'not_loadable', **when** the backfill script runs, **then** it skips that run and logs "Skipping <run_date>: not loadable (reason: ...)" without failing (verified by log inspection and checking that `pipeline_run` has no row for that run)
- [ ] **Given** all compatible legacy runs are backfilled, **when** the Postgres database is inspected, **then** `pipeline_run` has rows for each loaded legacy run with `trigger='legacy_backfill'`, and partitions exist back to 2025-05 (or the earliest loadable run) — verified by `SELECT DISTINCT date_trunc('month', start_time) FROM pipeline_run ORDER BY 1`
- [ ] **Given** the backfill script is idempotent, **when** it is run twice, **then** already-loaded legacy runs are skipped (verified by stable row counts and logs showing "Run 2026-06-10 already loaded")
- [ ] **Given** the backfill completes, **when** the backfill summary is reviewed, **then** it reports "Loaded X legacy runs, skipped Y (not loadable), Z already present" (verified by the script's final output)

## Notes
- **Mapping logic:** If the profile report identifies common patterns (e.g. all 1st_run–4th_run have results/*.json), the backfill script implements those mappings; edge cases are documented but may be skipped if they're one-offs
- **Partial history is acceptable:** If only 10 of 20 legacy runs are loadable, that's still valuable for S3; the story is done when all loadable runs are loaded, not when all 20 are loaded
- **GAP-25 closure:** Once legacy runs are backfilled, the data/legacy/ folder can be moved off OneDrive to the ITServer at cutover (S5 handles retention policy)
- **Checksum:** For legacy runs without a raw CSV, compute the checksum from the SQLite raw_data table's content
- **Foreign keys:** Same as container-era backfill — ensure FKs are populated correctly

## Dependencies
- [[Story - Profile legacy runs against the stage contract]] (the compatibility report must exist)
- [[Story - Backfill the container-era runs]] (backfill script and schema are proven)
