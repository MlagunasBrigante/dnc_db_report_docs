---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: must
s1_order: 6
status: to-do
gaps: [GAP-05]
jira: ""
size: M
---
# Story: Alert delivery failure raises an alert

## User Story
**As** the IT & Data lead, **I want** a failure in one alert channel (email or Teams) to raise a loud alarm through the other channel, **so that** I never have a silent alert loss when something critical happens.

## Acceptance Criteria
- [ ] **Given** the success email fails to send, **when** the Teams channel is available, **then** the Teams message states "Success email failed to send; run succeeded but report delivery failed" and logs the error.
- [ ] **Given** the Teams message fails to post, **when** the email channel is available, **then** the email body states "Teams alert failed to post; see logs for details".
- [ ] **Given** both channels fail, **when** the run completes, **then** the failure is logged as CRITICAL (not swallowed as WARNING), the run exits with a non-zero code, and the watchdog (S2) will catch it.
- [ ] **Given** a warning or error email fails, **when** the other channel is available, **then** the working channel reports the failure (same pattern as AC 1-2).
- [ ] Verified by temporarily breaking one channel (e.g. invalid recipient, wrong Teams channel ID) and confirming the other channel reports it.

## Notes
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th.
- **Code reference** (verified against `Dev/run.py`):
  - Current behavior: `run.py:1423-1446` — email failures are caught and logged as WARNING, never re-raised; run still reports success
  - README note: "email failures never fail the run"
- **Root cause from run report 2026-10-28 (GAP-05)**: the warning email returned 504, was logged as WARNING, and the run exited 0 ("Pipeline completed successfully"). The alert about a transient failure was silently lost.
- **Scope**: this story changes the error-handling logic so that alert-delivery failures are never swallowed. If one channel fails, the other must report it. If both fail, the run must exit non-zero so the S2 watchdog (out of scope for S1) can catch it.
- **Project rule** (scope constraints): "LawRuler is flaky, so never 'fail the run if N items fail'" applies to *data* work (Stage 1-4). Alerting is different — if both alert channels fail, nobody knows the run happened, so exiting non-zero is correct.
- The S2 watchdog (out of scope) will alert if there's no successful run that month, or a run is still going, or unresolved items remain. That catches the "both channels failed" scenario.

## Dependencies
- [[Story - Teams alert channel]] — must exist for the redundant-channel logic to work. This story can be implemented in parallel and will no-op gracefully if Teams posting is not yet available.
