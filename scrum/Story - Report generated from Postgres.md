---
type: story
project: dnc-cron-scrub
epic: "[[Epic - PostgreSQL system of record]]"
sprint: S2
moscow: must
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Report generated from Postgres
## User Story
**As** Michelle (pipeline owner), **I want** the Stage 5 Excel/CSV report built from Postgres instead of SQLite, **so that** the entire pipeline reads and writes one system of record and I can verify the Postgres schema is correct by comparing reports.

## Acceptance Criteria
- [ ] **Given** a run completes and writes to Postgres, **when** Stage 5 report generation executes, **then** it queries `leads_with_tags_YYYYMM` (and joins to other tables as needed) instead of reading from SQLite, and produces `leads_with_tags_YYYY-MM-DD.xlsx` and `.csv` (verified by inspecting run.py and confirming no `sqlite3` imports remain in the report stage)
- [ ] **Given** the 2026-10-28 run's data is backfilled into Postgres (via Story - Backfill container-era runs), **when** the report is regenerated from Postgres for that run_id, **then** the row count, column names, and a sample of 10 lead IDs match the original SQLite-era report (verified by diffing the two CSVs)
- [ ] **Given** the report query joins across partitions, **when** the query plan is examined (`EXPLAIN SELECT ...`), **then** Postgres prunes irrelevant partitions (only queries the YYYYMM partition for the requested run_id) — performance check (verified by inspecting EXPLAIN output showing "Partitions scanned: 1")
- [ ] **Given** the report includes metrics like "Total Leads Scrubbed" (which GAP-16 in S1 fixed to not be misleading), **when** the Postgres-generated report is reviewed, **then** the labels and counts match the corrected S1 definitions (verified by comparing to the S1-fixed SQLite report)
- [ ] **Given** Stage 5 runs, **when** Postgres is unreachable (simulated by stopping the Postgres container or breaking the DSN), **then** the error is caught and logged, and the run is marked failed (not silent) — verified by injecting a connection failure

## Notes
- **Query source:** The report queries `leads_with_tags_YYYYMM` as the primary table, joining to `scrub_results_translated` for interpreted codes and `raw_snapshot` for original lead data if needed
- **Backward compatibility:** The report file format (columns, labels, XLSX structure) should match the SQLite era, so downstream consumers (if any) don't break
- **Performance:** Partition pruning ensures the query only scans one month's data; verify with EXPLAIN
- **Verification dataset:** Use 2026-10-28 (already run under SQLite) as the gold standard for comparison

## Dependencies
- [[Story - All stages write to Postgres with lineage]] (leads_with_tags_YYYYMM must exist)
- S1 GAP-16 fix (misleading report labels corrected)
