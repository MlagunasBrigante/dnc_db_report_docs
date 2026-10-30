---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Failures captured and patched]]"
sprint: S2
moscow: must
status: to-do
gaps: [GAP-11]
jira: ""
size: M
---
# Story: Patch pass reprocesses a run's failed items
## User Story
**As** Michelle (pipeline owner), **I want** a scheduled patch pass (or manual `once --patch <run_id>`) to retry only the failed items from a specific run, **so that** transient API errors are healed automatically without re-running the entire pipeline.

## Acceptance Criteria
- [ ] **Given** a completed run with unresolved failed items (e.g. run_id `abc-123` has 3 failed Stage 4 tagging requests), **when** `python run.py --patch abc-123` is executed, **then** it queries `failed_items WHERE run_id='abc-123' AND resolved_at IS NULL`, retries only those 3 leads, and updates `failed_items.resolved_at` for successes or increments `attempts` for continued failures (verified by querying `failed_items` before and after)
- [ ] **Given** the patch pass retries a failed Stage 1 page fetch, **when** the retry succeeds, **then** the newly fetched rows are upserted into `raw_snapshot_YYYYMM` (keyed by `run_id, lead_id, phone`) without duplicating rows that already exist from the main run (verified by stable row counts and a test query showing the patched leads now exist)
- [ ] **Given** the patch pass retries a failed Stage 2 batch, **when** the batch succeeds, **then** the scrub results are written to the same `scrub_results_YYYYMM` partition, FK'd back to the existing `cleaned_phones` rows (verified by querying the scrub_results for that batch and checking FK integrity)
- [ ] **Given** the patch pass retries a failed Stage 4 lead tag, **when** the retry succeeds, **then** the `update_status` row is updated to `status='DONE', updated_at=<now>` and the lead is marked resolved in `failed_items` (verified by query)
- [ ] **Given** a patch pass is run twice on the same run_id (idempotency test), **when** the second execution finds no unresolved items, **then** it logs "No failed items to retry for run abc-123" and exits successfully without making any API calls (verified by log inspection and no new rows in failed_items)
- [ ] **Given** GAP-11 (resume only works on the same day), **when** a patch pass is run for a September run in October, **then** it succeeds because the patch pass takes run_id, not report_date (verified by running `--patch <sept_run_id>` on a different calendar day)

## Notes
- **CLI:** `python run.py --patch <run_id>` runs the patch pass; the systemd timer on the 3rd calls this with the most recent run_id
- **Keyed by run_id:** The patch pass writes to the SAME monthly partitions as the original run (identified by run_id), not to a new run
- **Idempotency:** Re-running the patch pass on the same run_id only retries items still marked unresolved; resolved items are skipped
- **Stage isolation:** The patch pass can retry any stage independently (e.g. only Stage 4 failures without re-running Stages 1–3)
- **Failure model:** If the patch pass itself fails (e.g. dncscrub.com is down for the whole pass), unresolved items remain unresolved and are alerted by Story - Unresolved items reach the alerts

## Dependencies
- [[Story - Failed items captured per stage]] (failed_items table must exist)
- [[Story - All stages write to Postgres with lineage]] (stage tables must support upserts)
