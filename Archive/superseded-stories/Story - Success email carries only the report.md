---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
status: archived
gaps: [GAP-02, GAP-03]
jira: ""
size: M
---

> [!note] Archived 2026-11-05
> Renamed on 2026-10-29 to [[Story - Monthly report email as a summary table]]. Kept for history only; not part of the backlog.
# Story: Success email carries only the report

## User Story
**As** the IT & Data lead, **I want** the success email to attach only the Excel and CSV report (not the database), with a size guard for oversized attachments, **so that** I reliably receive the monthly report without it exceeding Microsoft Graph or Exchange limits.

## Acceptance Criteria
- [ ] **Given** the report is generated, **when** the zip is created, **then** it contains only the XLSX and CSV files, not `scrub_ods.db` (`run.py:1163-1169` currently walks the entire output dir).
- [ ] **Given** the zip is created, **when** it is attached to the success email, **then** a size guard checks the attachment: < 3 MB is sent inline; 3–150 MB uses a Graph upload session; ≥ 150 MB is not attached (body states "report too large, see output directory").
- [ ] **Given** the success email is sent with a `202 Accepted` response, **when** I verify in my mailbox, **then** the email arrived with the report zip and is under the tenant's message-size limit (default 35 MB; exact limit to be verified with microsoft-learn MCP).
- [ ] Verified with one real run that the success email arrives in the mailbox with the correctly sized zip.

## Notes
- **Alert routing (decided 2026-10-29, stories interview Q1):** owner **Michelle**, backup **e.cervera**, who acts when Michelle is out (Michelle informs them). Email goes **to** `it.admin@` and the owner, **CC** `j.fernandez@` and `e.cervera@` (same domain as `it.admin@`; recipients from `Dev/.env`, e.g. `EMAIL_RECIPIENTS_*` plus a new `EMAIL_CC`). Teams goes to a **dedicated channel that Michelle sets up**, not a 1:1 chat. Response times: **errors, alert-delivery failures and watchdog alerts** are acted on the next business day; **warnings and unresolved items** within the week, before the patch pass on the 3rd or the watchdog on the 5th.
- **Code references** (verified against `Dev/run.py`):
  - Zip creation: `run.py:1163-1169` (`os.walk(output_dir)`) — zips everything including `scrub_ods.db`
  - Success email attachment: `run.py:1208-1209` — attaches inline without size check
- **Root cause from run report 2026-10-28**: report zip was 47 MB (actual report ~1.6 MB XLSX+CSV; the rest was the 164 MB database compressed). Success email returned `202 Accepted` but delivery is unverified because the 63 MB request exceeds the 3 MB inline limit and the 35 MB default message size.
- **Microsoft limits** (per run report; re-verify with microsoft-learn MCP):
  - Inline attachment: < 3 MB
  - Upload session: 3–150 MB (chunked, ≤ 4 MB per PUT)
  - Default max message size: 35 MB (tenant-configurable)
- The 10-28 report was 0.66 MB XLSX + 0.93 MB CSV = ~1.6 MB zipped. A size guard is insurance for growth; the expected size fits inline.
- Do not fail the run if the attachment is too large — alert in the body and log; Michelle can retrieve from the output directory.

## Dependencies
None — can start immediately.
