---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Project documentation and housekeeping]]"
sprint: S1
moscow: should
status: done
gaps: [GAP-22]
jira: "DNCUP-5"
size: M
---
# Story: Per-stage inputs and outputs reference

> [!todo] Michelle — action needed
> - [x] Read the updated `[[pipeline-stages-reference]]` (Stage 4 SQL join, Stage 6 Alerts, memory log, new S2 mapping); then this story closes.

## User Story
**As** the S2 schema designer and future maintainer, **I want** a reference document stating each stage's inputs, outputs, keys, business rules, and failure modes, **so that** I can design the Postgres schema and understand the pipeline without reading all of `run.py`.

## Acceptance Criteria
- [x] **Given** the six pipeline stages (Ingest, Scrub, Interpret, Update, Report, Email), **when** the reference doc is written, **then** each stage section states:
  - **Inputs**: data sources (LawRuler API, dncscrub.com, SQLite tables, env vars)
  - **Outputs**: SQLite tables, files, API side-effects (LawRuler tags, emails)
  - **Keys / uniqueness**: primary keys, composite keys, uniqueness constraints (e.g. `(Lead ID, Phone)`)
  - **Business rules**: DNC code translation, tag logic, filtering (e.g. scrubbed within last 90 days), active process statuses
  - **Failure modes**: what can go wrong (page error, batch error, API timeout, email 504) and current handling (retry, skip, alert)
- [x] **Given** the reference doc is complete, **when** the S2 schema designer reads it, **then** they can map each stage to a Postgres table without reading `run.py`.
- [x] The doc is committed to `Docs/Reference/pipeline-stages-reference.md` and linked from the project README and this story.
- [x] ~~Jira DNCUP-5 (originally "Document each stage's data contract") is closed when this story is done.~~ Dropped: Jira deprecated (D-13).

## Notes
- **Closed 2026-11-10 (Michelle).** Doc approved. In S2 it must drop the SQLite references once Postgres is the system of record (criterion added to [[Story - All stages write to Postgres with lineage]]).
- **Updated 2026-11-10 against `main` `a03e293`:** Stage 4 SQL working set (quirk 4 resolved, measured 731.5/799 MiB vs 1,103/1,207), Stage 6 renamed *Alerts* (email + Teams, exit 3, config-failure card), per-stage metrics and `memory_log.csv` columns, exit codes 0/1/3, and a new **S2 Mapping** table (stage → SQLite table/file → suggested Postgres table). Line refs re-verified; spot-checked by the main session (`run.py:835`, `1266`, `2486`, `2506`, `dnc/memory_log.py` columns, `CODE_VERSION` from `GIT_SHA`). AC1–AC3 met; awaiting Michelle's read.
- **Scope trimmed 2026-11-05 (Michelle):** sprints aren't mixed, so the doc describes the **current S1 pipeline only**. The S2 table-mapping proposal was removed; S2 schema design starts from this doc when S2 begins.
- **Gap reference**: GAP-22 — "No per-stage inputs/outputs document (DNCUP-5)"
- **Jira**: DNCUP-5 maps to this story; close DNCUP-5 when this story is done.
- **Scope**: this is the input contract for the S2 Postgres schema. It documents the current SQLite-based design; S2 will translate it to partitioned Postgres tables with `run_id` and surrogate keys.
- **Sources**:
  - Code: `Dev/run.py` (each stage class and function)
  - Run report: `Docs/Run Reports/DNC Run Report 2026-10-28.md` (per-stage timeline, volumes, keys)
  - Scope: `DNC Update - Scope.md` (business rules, ACTIVE_PROCESS_STATUSES, DNC_CODES)
- **Written 2026-11-05**: [[pipeline-stages-reference]]
- **Stage outline** (not prescriptive; adjust during writing):
  1. **Stage 1: Data Ingestion** — inputs: LawRuler Custom Report API (LayoutId 139, 2022-01-01 → report_date); outputs: `raw_data` (ODS), `cleaned_phone_numbers` (staging); keys: `(Lead ID, Phone)`; rules: filter recently scrubbed (< 90 days), remove bad numbers; failures: page error → truncate (GAP-09, fixed in S1)
  2. **Stage 2: DNC Scrubbing** — inputs: `cleaned_phone_numbers`; outputs: `scrub_results` (batch JSON); keys: `batch_id`; rules: 1,000-phone batches; failures: batch error → skip (GAP-10, fixed in S1)
  3. **Stage 3: Interpret** — inputs: `scrub_results`; outputs: `scrub_results_translated`; keys: `Phone`; rules: DNC_CODES map, litigator flag; failures: none expected (pure data transform)
  4. **Stage 4: Update LawRuler** — inputs: `raw_data`, `scrub_results_translated`; outputs: `leads_with_tags`, `update_status`, `update_results`; keys: `leadId`; rules: inner join, tag translation (DNC/Litigator/Blocked); failures: per-lead API error → buffered retry, mark DONE anyway (GAP-08, S2)
  5. **Stage 5: Report** — inputs: `leads_with_tags`, `raw_data`; outputs: XLSX, CSV, zip; keys: none (report); rules: summary counts, litigator soft-delete list; failures: none expected (file I/O)
  6. **Stage 6: Email** — inputs: zip, log; outputs: Graph email, Teams message (S1); keys: none; rules: attachment size, recipient lists; failures: 504 on oversized attachment (GAP-01/02/03, fixed in S1)
- This is M-sized because it requires reading the code, run report, and scope to extract the contract, then writing a structured reference doc.

## Dependencies
None — can start immediately. Should complete before S2 schema design starts.
