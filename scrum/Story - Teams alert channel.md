---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: should
status: to-do
gaps: []
jira: ""
size: M
---
# Story: Teams alert channel

## User Story
**As** the IT & Data lead and IT on-call, **I want** success, warning, error and unresolved-failure alerts posted to a Teams channel, **so that** I have a second alert channel and a visible timeline of run outcomes in the team workspace.

## Acceptance Criteria
- [ ] **Given** the existing Graph connector (`_get_o365_account`, `run.py:1120`) and the spike's recommendation, **when** a run completes successfully, **then** a success message is posted to the configured Teams channel or chat with the report date and a summary (leads tagged, DNC count).
- [ ] **Given** a run emits warnings but succeeds, **when** the run completes, **then** a warning message is posted with the count of warnings and a link or excerpt.
- [ ] **Given** a run fails, **when** the error is caught, **then** an error message is posted with the exception type and a compact traceback excerpt.
- [ ] **Given** one alert channel fails (email or Teams), **when** the other channel succeeds, **then** the working channel reports the failure of the other (e.g. "Warning email failed to send; see logs").
- [ ] Verified with one `--dry-run` that injects a failure and confirms both email and Teams alerts arrive.

## Notes
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th. Story is blocked until Michelle has created the channel and shared its team and channel identifiers for config.
- **Code reference** (verified against `Dev/run.py`):
  - Existing Graph connector: `_get_o365_account` at `run.py:1120` — already used for email
- **Depends on**: [[Story - Spike Teams posting through Graph]] — must complete first to know which Graph API and permission to use.
- **Scope note**: this reuses the existing Graph connector and credentials in `Dev/.env`. No new secrets; the connector already exists for email.
- **Alert-delivery failure** is handled by this story's AC 4 and by [[Story - Alert delivery failure raises an alert]]. If both channels fail, the run logs the failure and exits — that scenario is caught by the S2 watchdog (out of scope for S1).
- Message format: plain text or Adaptive Card (decide during implementation based on the spike's recommendation and Graph API constraints).
- **Never fail the run** because Teams posting failed — log the failure and continue. The project rule is "flaky APIs → never 'fail the run if N items fail'".

## Open questions
- [ ] #open-question Team and channel identifiers of the dedicated channel (Michelle creates it).

## Dependencies
- [[Story - Spike Teams posting through Graph]] — must complete first.
