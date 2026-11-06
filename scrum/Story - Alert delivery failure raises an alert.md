---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: must
s1_order: 6
status: in-review
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
- **Fix branch `s1/fix/alert-delivery-gaps` (`9afc77b`, `f006357`, 2026-11-06), reviewed, 246 tests pass:** (a) when every alert channel fails, the CRITICAL line now comes first and the end line reads `⚠️ Pipeline completed but alert delivery failed on all channels (exit 3)`, followed by `Report: <path>`; (b) a config-validation failure now posts the Teams error card too, with the same cross-reporting (new `dnc/config_failure_alerts.py`); exit code stays 1. Awaiting merge, rebuild and a Mac re-check.
- **Error-path dry-run 2026-11-06 17:24 UTC** (`auth_url` broken + `email:fail`, `data/_dryrun/2026-11-06T172427Z/`): auth 404 not retried, `CRITICAL Pipeline failed`, error email failed (injected), Teams error card posted, exit 1 ✅. Card content (exception + failed-email note) to be confirmed by Michelle. Warning-path run still to do (the 17:29 run kept the broken `auth_url`, so it repeated the error path).
- **`email:fail,teams:fail` dry-run 2026-11-06 01:05 UTC** (`data/_dryrun/2026-11-06T003829Z/`): both failed, `CRITICAL Alert delivery failed on all channels`, exit 3 ✅. Michelle confirmed the Teams card from the `email:fail` run carried the failed-email note ✅. Not yet seen on the Mac: warning/error cards and warning/error email cross-reports (unit-tested only). Found 2026-11-06: (a) the end line still says `✅ Pipeline completed successfully` right before the all-channels CRITICAL; (b) a config-validation failure (exit 1) sends only the error email, not Teams, so missing O365 keys alert nobody.
- **`email:fail` dry-run 2026-11-06 00:16 UTC** (`data/_dryrun/2026-11-05T234847Z/`): success email failed (injected), Teams posted, exit 0. The card should carry "Success email failed to send; run succeeded but report delivery failed" (`dnc/alert_delivery.py`, `run.py:2336-2350`); Michelle to confirm on the card. Still to run: `email:fail,teams:fail`, expecting exit 3.
- **Mac dry-runs 2026-11-05:** Teams fails, so an email reports it: verified (`teams:fail`, exit 0). Still to verify: `email:fail` (Teams reports the email failure) and `email:fail,teams:fail` (all channels failed, exit 3).
- **In review 2026-11-05** (5 commits `c69bffc`…`d4d0797` + review fix `05df099`, stacked on `s1/feat/teams-alert-channel`). Each alert's email and Teams sends return `sent` / `failed` / `skipped` (not configured or dry-run without a target; skipped isn't a failure). Order: email first, then Teams.
  - A failed email adds a note to the Teams card.
  - A failed Teams post sends a short follow-up email.
  - If every configured channel of a due alert fails: CRITICAL "Alert delivery failed on all channels" and **exit 3** after cleanup. A pipeline error stays exit 1, and its CRITICAL line is still logged.
  - `scheduler.py` logs exit 3 at ERROR and keeps running.
  - Dry-run tokens `email:fail` / `teams:fail` (`teams:fail` counts Teams as configured even without a test URL).
  - **Until `TEAMS_WEBHOOK_URL` is set, a failed email has no second channel, so the run exits 3.** That's intended: nobody was alerted.
  - Review fix: the alert senders re-logged the injection banner six times per dry-run.
- Started 2026-11-05 on `s1/fix/alert-delivery-failure`, stacked on the Teams branch. If both channels fail, the run exits with code 3 (distinct from 1 = pipeline error).
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
