---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: must
s1_order: 6
status: done
gaps: [GAP-01, GAP-04]
jira: ""
size: M
---
# Story: Warning and error emails always arrive

## User Story
**As** the IT & Data lead, **I want** warning and error emails to always arrive with a compact excerpt of relevant log lines instead of the full log, **so that** I never miss a critical failure notification because the email itself failed.

## Acceptance Criteria
- [ ] **Given** a run emits WARNING or ERROR logs, **when** the warning email is sent, **then** it attaches only a compact excerpt of WARNING+ lines (not the full log), capped at 2 MB, with a count of each level grouped by type (e.g. "3 ERROR, 5 WARNING").
- [ ] **Given** a run fails, **when** the error email is sent, **then** it gets the same fix as the warning email (compact excerpt, not full log; `run.py:1259-1260`).
- [ ] **Given** a batch or lead operation is retried successfully, **when** the run completes, **then** that transient error does not count as a warning (only FINAL failures count; `run.py:77-101,1420-1421`).
- [ ] **Given** the warning email was sent, **when** I verify in my mailbox, **then** it arrived and is under 3 MB total size.
- [ ] Verified with one `--dry-run` that injects a failure and confirms the warning email arrives.

## Notes
- **Mailbox confirmed by Michelle 2026-11-05**: the warning and error emails arrived with good formatting and a small excerpt attachment.
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T232423Z/`, `DRYRUN_EMAIL_RECIPIENTS` set to Michelle): the warning email was sent with the excerpt attached (2 ERROR, 1 WARNING, 1 CRITICAL). The 2026-11-05 config check also sent the error email with its excerpt. Graph accepted both. **Waiting on Michelle's mailbox check**: arrived, small attachment, readable.
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T222929Z/`, `main` @ `ea3d03c`, `DRYRUN_INJECT_FAILURES=batch:0`): both emails were built, but sending was skipped because `DRYRUN_EMAIL_RECIPIENTS` is unset. Delivery, attachment size and HTML rendering still need one dry-run with it set.
- **In review 2026-11-04** (`fc8d520` + review fix `49bf33d`): warning and error emails attach `dnc/log_excerpt.py`'s excerpt. It holds warning, error and critical lines, plus any traceback following an error, with level counts, capped at 1 MB, and **only the last run in `run.log`**. Same-day real runs append to one log, so an earlier aborted run's errors would otherwise land in the next run's alert (review fix). Self-healed retries log at DEBUG, so they never reach the warning email; only "All N retries exhausted" is an ERROR. Optional `ALERT_EMAIL_CC`. To verify on the Mac: a dry-run with `DRYRUN_EMAIL_RECIPIENTS` set to Michelle and an injected failure, then check the email arrives with a small attachment.
- **Graph limits checked 2026-11-04 against Microsoft Learn** ([attachment resource](https://learn.microsoft.com/graph/api/resources/attachment?view=graph-rest-1.0), [large attachments](https://learn.microsoft.com/graph/outlook-large-attachments)): a single-request attachment must be **under 3 MB**; 3–150 MB needs an upload session; the default Exchange Online message limit is 35 MB. The excerpt is capped at **1 MB of text** (about 1.33 MB once base64-encoded), well inside the inline limit. Branch: `s1/fix/alert-emails-compact-excerpt`.
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th.
- **Code references** (verified against `Dev/run.py`):
  - Warning email attachment: `run.py:1307-1308` — currently attaches full `run.log`
  - Error email attachment: `run.py:1259-1260` — same pattern, same fix
  - WarningCollector: `run.py:77-101` — captures every WARNING+ record
  - Warning snapshot: `run.py:1420-1421` — taken before Stage 6 to exclude email-stage noise
- **Root cause from run report 2026-10-28**: full `run.log` (473 MB, inflated by DEBUG logging) → 172 MB JSON request → 504 Gateway Timeout. A single transient, self-healed timeout triggered the oversized email.
- **Microsoft Graph limits** (per run report; to be re-verified with microsoft-learn MCP during implementation):
  - Inline attachment: < 3 MB
  - 3–150 MB: must use upload session
  - Default max message size: 35 MB
- Scope: this story fixes the attachment only. The log inflation (DEBUG logging writing full request bodies) is **GAP-06** in the "Lean and secret-free logs" epic.

## Dependencies
None — can start immediately.
