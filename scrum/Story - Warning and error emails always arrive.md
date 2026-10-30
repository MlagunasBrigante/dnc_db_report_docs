---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: must
s1_order: 6
status: to-do
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
