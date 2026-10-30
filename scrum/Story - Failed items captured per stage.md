---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Failures captured and patched]]"
sprint: S2
moscow: must
status: to-do
gaps: [GAP-08]
jira: ""
size: S
---
# Story: Failed items captured per stage
## User Story
**As** Michelle (pipeline owner), **I want** the failed-item events already emitted by the shared retry policy ([[Story - One retry policy for every API call]], S1) to be persisted in Postgres for every stage (pages, batches, leads), **so that** I can see which specific items failed and nothing is silently skipped.

## Acceptance Criteria
- [ ] **Given** Stage 1 encounters a LawRuler page fetch error (simulated by mocking a 500 response for page 42), **when** the run completes, **then** the `failed_items` table contains a row with `(run_id, stage='stage1', item_key='page_42', error='HTTP 500', attempts=3)` and the run does not abort (verified by querying `failed_items` and checking that the run's status is still 'completed')
- [ ] **Given** Stage 2 batch scrub fails (simulated by mocking a dncscrub.com timeout for batch 5), **when** the run completes, **then** `failed_items` records `stage='stage2', item_key='batch_5', error='Timeout'` and the run continues to tag the leads that did scrub successfully (verified by query + log inspection)
- [ ] **Given** Stage 4 tagging fails for lead 475667 (as happened in the 2026-10-28 run with a transient connect timeout), **when** the run completes, **then** `failed_items` records `stage='stage4', item_key='lead_475667', error='ConnectTimeout', attempts=1` initially, and if the retry succeeds, the row is updated to `attempts=2, resolved_at=<timestamp>` (verified by query showing the resolved row)
- [ ] **Given** GAP-08 (failed tagging recorded as DONE) is addressed, **when** a tagging request fails and is recorded in `failed_items`, **then** the `update_status` table marks it as 'FAILED', not 'DONE' (verified by querying `update_status WHERE leadId=475667 AND status='FAILED'`)
- [ ] **Given** the failed_items table schema, **when** it is inspected, **then** it has columns `(id SERIAL, run_id UUID, stage TEXT, item_key TEXT, error TEXT, attempts INT DEFAULT 1, created_at TIMESTAMPTZ, resolved_at TIMESTAMPTZ)` and a partial unique index on `(run_id, stage, item_key) WHERE resolved_at IS NULL` to prevent duplicate unresolved failures (verified by `\d failed_items`)
- [ ] **Given** a run with some failed items, **when** the run's final log is reviewed, **then** it logs "Stage 1: 1 failed item (page_42); Stage 2: 1 failed item (batch_5); Stage 4: 0 failed items (all resolved)" at INFO level (verified by grepping the log)

## Notes
- **Schema:** `failed_items` table with run_id FK to pipeline_run, stage (stage1/stage2/stage4), item_key (page_N, batch_N, lead_N), error text, attempts count, resolved_at timestamp
- **Item key format:** `page_<N>`, `batch_<N>`, `lead_<leadId>` — consistent naming for easy filtering
- **Resolved vs unresolved:** `resolved_at IS NULL` means still failing; `resolved_at IS NOT NULL` means a later retry succeeded
- **GAP-08 fix:** The S1 fix to `run.py:819-838` (recording status as DONE even on failure) is replaced by checking the failed_items table: if the lead is in failed_items with resolved_at=NULL, it's not DONE

## Dependencies
- [[Story - One retry policy for every API call]] (S1). This story persists its events; no per-stage capture logic is written here.
- [[Story - All stages write to Postgres with lineage]] (run_id and stage tables must exist)
