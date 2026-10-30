---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Historical backfill]]"
sprint: S2
moscow: must
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Profile legacy runs against the stage contract
## User Story
**As** Michelle (data owner), **I want** a compatibility report for each of the 20 legacy runs (May 2025–July 2026) showing which are loadable as-is, which need mapping, and which are incompatible, **so that** I know what history can be preserved without forcing lossy conversions.

## Acceptance Criteria
- [ ] **Given** the legacy runs in `data/legacy/` (1st_run, 2nd_run, 3rd_run, 4th_run, plus 16 dated runs 2025-12-09 → 2026-07-01), **when** `Dev/profile_legacy.py` is executed, **then** it produces a JSON or CSV report `legacy-compatibility-report.json` with one row per run (verified by checking the file exists and has 20 rows)
- [ ] **Given** the profile script inspects a run, **when** it checks `data/legacy/2026-06-10/`, **then** it reports: run_date, SQLite files present (scrub_ods.db? scrub_staging.db?), table schemas (columns and types), row counts per table, and compatibility status ('loadable' / 'needs_mapping' / 'not_loadable') — verified by inspecting the report for 2026-06-10
- [ ] **Given** a legacy run has SQLite with the expected schema (same columns as container-era), **when** profiled, **then** it is marked 'loadable' (verified by the report showing at least one run with status='loadable')
- [ ] **Given** a legacy run has SQLite but different column names or missing tables (e.g. the notebook era's interim/ and results/ folders with CSV/JSON instead of scrub_staging.db), **when** profiled, **then** it is marked 'needs_mapping' with a note of what differs (verified by the report for 1st_run showing 'needs_mapping' and noting "no scrub_staging.db, has results/*.json")
- [ ] **Given** a legacy run has no SQLite or is missing critical data (e.g. bug/ folder or a run with only logs), **when** profiled, **then** it is marked 'not_loadable' (verified by the report showing at least one run with status='not_loadable')
- [ ] **Given** the profile script completes, **when** the report is reviewed, **then** it includes a summary: "20 runs profiled: X loadable, Y need mapping, Z not loadable" (verified by the report's summary section)

## Notes
- **Profile script:** `Dev/profile_legacy.py` is a one-time analysis tool, not part of the pipeline; it reads folder structures and SQLite schemas but does not write to Postgres
- **Compatibility logic:**
  - 'loadable': SQLite schema matches container-era (same tables, same columns)
  - 'needs_mapping': Data exists but in a different format (CSV/JSON, renamed columns, missing tables that can be inferred)
  - 'not_loadable': Missing critical data or fundamentally incompatible (e.g. only logs, no lead data)
- **Output format:** JSON or CSV with columns: run_date, status, notes, file_structure, table_schemas, row_counts
- **Time-boxed:** If profiling reveals that most runs need custom mapping, the story's deliverable is the report — the decision to invest in mapping happens after reviewing the report

## Dependencies
- None (reads from data/legacy/ only; no Postgres dependency)
