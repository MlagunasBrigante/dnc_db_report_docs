---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Failures captured and patched]]"
sprint: S2
moscow: must
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Unresolved items reach the alerts
## User Story
**As** Michelle (pipeline owner), **I want** the email and Teams alerts to include unresolved failed-item counts and item keys after the main run and patch pass, **so that** I know immediately if any leads were not tagged and can investigate.

## Acceptance Criteria
- [ ] **Given** a run completes with 5 unresolved failed items (3 Stage 4 tagging failures, 2 Stage 2 batch failures), **when** the success email is sent, **then** it includes a section "Unresolved failures: 5 items (Stage 2: 2, Stage 4: 3)" with the item keys listed (e.g. "batch_7, batch_8, lead_12345, lead_67890, lead_11111") — verified by inspecting the email body
- [ ] **Given** the patch pass runs on the 3rd and resolves 4 of the 5 failed items, **when** the patch pass completes, **then** its alert email includes "Patch pass: 4 resolved, 1 still failing (lead_11111: ConnectTimeout, 3 attempts)" (verified by email inspection)
- [ ] **Given** unresolved items exist after the patch pass, **when** the Teams alert is sent, **then** it includes the same unresolved-item summary as the email (verified by checking the Teams channel)
- [ ] **Given** all failed items are resolved (no unresolved rows in `failed_items` for this run), **when** the alert is sent, **then** it includes "All items processed successfully (0 unresolved)" and does not trigger a warning-level alert (verified by log level and email subject not containing "WARNING")
- [ ] **Given** unresolved items exist, **when** the alert is sent, **then** the run's `pipeline_run.status` is still 'completed', not 'failed' — a run is never marked failed only because some items failed (verified by querying `pipeline_run` and confirming status='completed')
- [ ] **Given** the alert logic queries unresolved items, **when** the query is reviewed, **then** it is `SELECT stage, item_key, error, attempts FROM failed_items WHERE run_id=? AND resolved_at IS NULL ORDER BY stage, attempts DESC` (verified by reading run.py alert code)

## Notes
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th.
- **Alert timing:** The main run's alert (after Stage 5) reports unresolved items as of that moment; the patch pass alert (on the 3rd) reports what's still unresolved after the patch
- **Email format:** Add an "Unresolved failures" section to the existing success/warning email template; if count > 0, the subject should include "with unresolved items" to make it visible in the inbox
- **Teams format:** Same data, formatted as a card or adaptive card if already in use
- **No false positives:** Resolved items (resolved_at IS NOT NULL) are excluded from the alert
- **Run status:** The run is marked 'completed' as long as the pipeline itself didn't crash; unresolved items are reported but don't change the run's outcome to 'failed'

## Dependencies
- [[Story - Failed items captured per stage]] (failed_items table must exist)
- [[Story - Patch pass reprocesses a run's failed items]] (patch pass must mark items resolved)
- S1 email + Teams alerting (the alert infrastructure must exist)
