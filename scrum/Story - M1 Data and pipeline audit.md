---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Data readiness for modeling]]"
sprint: S3
moscow: tbd
status: to-do
milestone: M1
jira: ""
size: M
---
# Story: M1 Data and pipeline audit

## User Story
**As** Michelle (IT & Data lead), **I want** a complete audit of the backfilled data structure, PostgreSQL schema, and pipeline lineage, **so that** I understand what data is available, how it flows through the stages, and whether the legacy runs are usable for statistical modeling.

## Acceptance Criteria
- [ ] **Given** the S2 migration is complete, **when** the audit runs, **then** a report documents: (1) row counts per stage table and per partition (container-era 2026-08→ and legacy 2025-05→2026-07), (2) schema of each stage table (columns, types, constraints), (3) foreign-key lineage from raw → cleaned → scrubbed → tagged, (4) any schema mismatches or missing data in legacy runs, (5) the folder structure under `data/legacy/` mapped to PostgreSQL partitions.
- [ ] **Given** the audit report exists, **when** Michelle reviews it, **then** she can identify which runs and which stage tables are complete enough to use for modeling, and which (if any) have gaps that block statistical analysis.
- [ ] **Given** the pipeline code (`Dev/run.py`, stages 1–5), **when** documented, **then** a lineage diagram or table shows what each stage reads, transforms, and writes (e.g., Stage 1: LawRuler API → `raw_data` table; Stage 2: `raw_data` → `cleaned_phones` → dncscrub API → `scrub_results`), cross-referenced with the per-stage I/O doc from S1 (DNCUP-5).
- [ ] **Given** the audit identifies any unusable runs (schema mismatch, incomplete stages, missing files), **when** those runs are flagged, **then** they are documented in the report with a recommendation: exclude from modeling, or backfill/repair in a follow-up story.
- [ ] **M1 go/no-go criterion:** Michelle and engineering agree the data structure is understood and the audit report is complete. No modeling work starts without this checkpoint passing.

## Notes
- The audit is **read-only** — it does not fix data, only documents what exists.
- Legacy runs may have schema differences (notebook-era vs container-era); S2 backfill already checked them against the stage-table contract, so this story surfaces what S2 excluded or flagged.
- The folder structure under `data/legacy/` includes `1st_run`–`4th_run` (May–Aug 2025), 16 dated runs (2025-12-09 → 2026-07-01), and a `bug/` folder; the audit maps each to its PostgreSQL partition or explains why it's absent.
- Output: a markdown document (suggested: `Docs/S3-M1-Data-and-Pipeline-Audit.md`) in the repo, reviewed with Michelle before closing the story.

## Dependencies
- **Blocks:** all other S3 stories (M2–M5 require understanding the data first)
- **Depends on:** S2 migration complete (backfill done, schema created, at least one scheduled run passed on ITServer)
