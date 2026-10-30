---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: must
s1_order: 4
status: to-do
gaps: [GAP-09]
jira: ""
size: M
---
# Story: Lead download never truncates silently

## User Story
**As** the compliance/legal owner, **I want** a Stage 1 page error to be retried and, if still failing, to alert loudly and mark the raw snapshot as incomplete, **so that** a page error never silently truncates the lead download.

## Acceptance Criteria
- [ ] **Given** a page request returns non-200 (e.g. 500, 504, connection timeout), **when** the shared retry policy ([[Story - One retry policy for every API call]]) has exhausted its retries. Today non-200 pages are not retried at all and the loop just `break`s, **then** the run alerts CRITICAL "Stage 1 page X failed after N retries; raw snapshot incomplete" and the raw snapshot is marked incomplete (e.g. a flag in the ODS database or a marker file).
- [ ] **Given** the raw snapshot is marked incomplete, **when** the run completes (or fails), **then** the alert email and Teams message state "INCOMPLETE DATA: Stage 1 page error; lead count may be understated".
- [ ] **Given** a page error occurs but retries succeed, **when** the run completes, **then** no warning is emitted (transient errors that self-heal do not count as warnings — see [[Story - Warning and error emails always arrive]] for the WarningCollector fix).
- [ ] **Given** a page error is injected (e.g. by mocking the API to return 500 for page 5), **when** the run completes, **then** the alert states the incomplete-data condition and the raw snapshot is marked as such.
- [ ] Verified with one `--dry-run` that injects a page error and confirms the loud alert.

## Notes
- **Code reference** (verified against `Dev/run.py`):
  - Page fetch: `run.py:306-308` — `if response.status_code != 200: … break` — exits the loop without retrying the page or alerting
  - Retry logic: `run.py:288-300` — already retries each page up to 3 times for Timeout/ConnectionError
- **Root cause from run report 2026-10-28 (GAP-09)**: "A Stage-1 page error silently truncates the download" — if a page returns non-200 after the per-page retries, the loop breaks and the run proceeds with incomplete data, never alerting that the snapshot is partial.
- **Scope**: this story fixes the page-error handling so it (1) logs CRITICAL, (2) marks the raw snapshot incomplete, and (3) alerts in the email/Teams message. The general failed-items store and scheduled patch pass for every stage are S2 features (out of scope for S1).
- **Incomplete-snapshot marker**: options include a flag column in the ODS `raw_data` table (`incomplete BOOLEAN`), a marker file in the output directory (`data/{date}/.incomplete`), or a row in a new `pipeline_run` table (S2 schema). Choose the simplest for S1; S2 will have a proper `pipeline_run` table.
- **Project rule**: "LawRuler is flaky, so retry logic and the failure model must assume it." This story is part of that assumption — a page error is expected, so the pipeline must handle it gracefully and loudly.

## Open questions
- [ ] #open-question D-05: keep the 2 s/page LawRuler throttle? The full snapshot is legally required; only the throttle is in question.

## Dependencies
- [[Story - One retry policy for every API call]] (provides the retry and the failed-item event; this story adds the incomplete-snapshot marker and the alert wording).
