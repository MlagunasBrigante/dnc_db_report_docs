---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: should
status: done
gaps: [GAP-21]
jira: "DNCUP-4"
size: S
---
# Story: Runbook for failed or degraded runs

> [!todo] Michelle — action needed
> - [ ] Brief e.cervera that they're the backup, will receive error alerts, and should act when you're out.

## User Story
**As** the IT on-call, **I want** a written runbook stating who acts on each alert type, through which channel, and the steps to take, **so that** I know immediately what to do when a DNC alert arrives.

## Acceptance Criteria
- [x] **Given** a success alert arrives, **when** I consult the runbook, **then** it states "no action required; archive the report".
- [x] **Given** a warning alert arrives (run succeeded but emitted warnings), **when** I consult the runbook, **then** it states who reviews the warning log, which warnings are urgent (e.g. config missing, partial data loss) vs informational (e.g. transient retry), and where to find the full log.
- [x] **Given** an error alert arrives (run failed), **when** I consult the runbook, **then** it states who acts, the escalation path, how to check the last successful run, and how to manually trigger a patch pass (S2 feature; note the limitation in S1).
- [x] **Given** an alert-delivery failure is reported in the working channel, **when** I consult the runbook, **then** it states how to verify alert delivery, check the output directory for the report, and restore the failed channel.
- [x] The runbook is committed to `Docs/Reference/DNC-runbook.md` and linked from the project README.

## Notes
- **Approved by Michelle 2026-11-05.** Log messages in the runbook were checked against the code and corrected (successful retries log at DEBUG only; no `Config validation failed` line exists). The e.cervera briefing above stays open as Michelle's action.
- **Watchdog heartbeat (Q9):** the runbook includes "No watchdog heartbeat in Teams by the end of the 5th → check `systemctl status dnc-scrub-watchdog.timer` / `journalctl` on the ITServer" and "DNC watchdog FAILED alert → …".
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th. The runbook names the owner and the backup and gives each alert type its response time.
- **Gap reference**: GAP-21 — "No runbook for failed or degraded runs (DNCUP-4)".
- **Jira**: DNCUP-4 (originally "Runbook for DNC scrub failures") maps to this story; close DNCUP-4 when this story is done.
- **Scope**: S1 runbook covers alert types (success, warning, error, alert-delivery failure), who acts, and immediate actions. The S2 watchdog, patch pass, and scheduler recovery are out of scope for S1 but should be noted as "coming in S2" where relevant.
- **Personas** (from scope): Michelle (IT & Data lead, pipeline owner, compliance/legal owner), IT on-call.
- **Channels**: email (today), Teams (by end of S1).
- **Alert types**:
  - Success — no action
  - Warning — run succeeded, but emitted WARNING logs (e.g. transient retry that succeeded, unmatched records, invalid phones)
  - Error — run failed (exception in any stage)
  - Alert-delivery failure — one or both channels failed to deliver the alert itself
- Reference the "Reliable alerts" epic stories for what each alert type now contains (compact excerpt, size guard, redundant channel).

## Dependencies
None — can start immediately. The runbook describes the alert behavior that other stories in this epic will implement, so it can be written in parallel and updated as those stories complete.

## Work Log
- **2026-11-05:** Runbook written at [[DNC-runbook]] (`Docs/Reference/DNC-runbook.md`). All behaviors grounded in code (`run.py`, `dnc/*.py`). Covers success/warning/error/alert-delivery-failure/soft-deletion-review alerts, manual re-run, log/output locations, channel restoration, S2 limitations. Linked from project README. Story set to in-review.
