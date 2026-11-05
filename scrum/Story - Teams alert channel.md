---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: should
status: in-review
gaps: []
jira: ""
size: M
---
# Story: Teams alert channel

> [!todo] Michelle — action needed
> - [ ] Create the dedicated Teams channel and share its team and channel IDs (needed before this story starts).

## User Story
**As** the IT & Data lead and IT on-call, **I want** success, warning, error and unresolved-failure alerts posted to a Teams channel, **so that** I have a second alert channel and a visible timeline of run outcomes in the team workspace.

## Acceptance Criteria
- [ ] **Given** the existing Graph connector (`_get_o365_account`, `run.py:1120`) and the spike's recommendation, **when** a run completes successfully, **then** a success message is posted to the configured Teams channel or chat with the report date and a summary (leads tagged, DNC count).
- [ ] **Given** a run emits warnings but succeeds, **when** the run completes, **then** a warning message is posted with the count of warnings and a link or excerpt.
- [ ] **Given** a run fails, **when** the error is caught, **then** an error message is posted with the exception type and a compact traceback excerpt.
- [ ] **Given** one alert channel fails (email or Teams), **when** the other channel succeeds, **then** the working channel reports the failure of the other (e.g. "Warning email failed to send; see logs").
- [ ] Verified with one `--dry-run` that injects a failure and confirms both email and Teams alerts arrive.

## Notes
- **Mac dry-runs 2026-11-05:** with `DRYRUN_TEAMS_WEBHOOK_URL` set, the log shows "Teams alert posted successfully" (`data/_dryrun/2026-11-05T230036Z/`). With `teams:fail`, the success email still went and a follow-up email reported the Teams failure (exit 0). Still to confirm: the card renders in the channel (Michelle). Warning and error cards haven't been seen yet; they'll show in the `email:fail` / `email:fail,teams:fail` runs.
- **In review 2026-11-05** (`405d125` + review fix `df36e69`). Adaptive Card to `TEAMS_WEBHOOK_URL` (dry-run: `DRYRUN_TEAMS_WEBHOOK_URL` only) for success, warning and error; aggregate counts only. Shared retry policy (`TEAMS_POLICY`: 3 attempts); never fails the run. Review fixes:
  - Webhook signatures were **not masked** in request-error messages, because `requests` prints the path without `https://`. `sig=` is now masked on its own.
  - The three webhook masking tests never ran: they sat after `if __name__`.
  - Unrelated edits to the existing masking patterns were reverted.

  AC 4 (cross-channel reporting) is delivered with [[Story - Alert delivery failure raises an alert]].
- Started 2026-11-05 on `s1/feat/teams-alert-channel`, using the webhook design.
- **Spike outcome 2026-11-05 ([[teams-graph-spike]]):** use a Teams **Workflows webhook** (`TEAMS_WEBHOOK_URL`, Adaptive Card POST), not Graph and not `_get_o365_account`. Team and channel IDs are no longer needed, only the webhook URL. Dry-run posts only to `DRYRUN_TEAMS_WEBHOOK_URL`.
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th. Story is blocked until Michelle has created the channel and shared its team and channel identifiers for config.
- **Code reference** (verified against `Dev/run.py`):
  - Existing Graph connector: `_get_o365_account` at `run.py:1120` — already used for email
- **Depends on**: [[Story - Spike Teams posting through Graph]] — must complete first to know which Graph API and permission to use.
- **Scope note**: this reuses the existing Graph connector and credentials in `Dev/.env`. No new secrets; the connector already exists for email.
- **Alert-delivery failure** is handled by this story's AC 4 and by [[Story - Alert delivery failure raises an alert]]. If both channels fail, the run logs the failure and exits — that scenario is caught by the S2 watchdog (out of scope for S1).
- Message format: plain text or Adaptive Card (decide during implementation based on the spike's recommendation and Graph API constraints).
- **Never fail the run** because Teams posting failed — log the failure and continue. The project rule is "flaky APIs → never 'fail the run if N items fail'".

## Open questions
- [ ] #open-question Webhook URL of the dedicated channel's "Send webhook alerts to a channel" workflow (Michelle creates the channel and the workflow, and stores the URL in `Dev/.env` as `TEAMS_WEBHOOK_URL`).

## Dependencies
- [[Story - Spike Teams posting through Graph]] — must complete first.
