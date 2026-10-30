---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: must
s1_order: 6
status: to-do
gaps: [GAP-02, GAP-03]
jira: ""
size: S
---
# Story: Monthly report email as a summary table

## User Story
**As** a report recipient (management and the team), **I want** the monthly report email to show the run's summary table (counts and actions) in the email body, with no attachments, **so that** I can read the month's DNC outcome at a glance, and the email can never fail because of its size.

## Acceptance Criteria
- [ ] **Given** a successful run, **when** the report email is sent, **then** its body contains the Summary table (`Metric | Count | Actions Taken`, the same rows as the xlsx "Summary" sheet, using the corrected labels from [[Story - Report labels and unmatched warning fixed]]) as an HTML table, plus the report date and run duration. **No attachments.**
- [ ] **Given** the report email is built, **when** its content is inspected, **then** it holds **aggregate counts only, with no lead-level data** (no names, phones or lead IDs), because one recipient is external (see Notes).
- [ ] **Given** the run produces the xlsx and csv, **when** the email is sent, **then** those files are still written to the run's output folder (`data/{date}/processed/`) for internal use; they're just not emailed. `_zip_output` is no longer called for this email.
- [ ] **Given** the recipient config, **when** the email is sent, **then** it goes **to** `brad@briganteimmigration.com`, `r.jimenez@briganteimmigration.com` and `matt.hirschberg@jordanbelfort.com`, with **CC** to `m.lagunas@`, `j.fernandez@` and `e.cervera@` (briganteimmigration.com). Recipients are read from `Dev/.env` (e.g. `REPORT_RECIPIENTS_TO` / `REPORT_RECIPIENTS_CC`), not hardcoded.
- [ ] **Given** a real run, **when** delivery is checked, **then** an internal recipient (Michelle, CC) confirms it arrived, and the external recipient's copy isn't bounced (no NDR in the `it.admin@` sender mailbox).

## Notes
- **Decided 2026-10-29 (stories interview Q2):** the report email carries **no files**. It focuses on the actual report: a succinct table of counts and actions. The report format may change after S3 statistical modeling; until then it stays as it is (Summary sheet content).
- **This email and the alert emails are separate audiences.** Warning, error, unresolved-item and watchdog alerts go to `it.admin@` + Michelle, CC j.fernandez and e.cervera, plus the Teams channel (interview Q1). The report email goes to the business recipients above.
- **External recipient:** `matt.hirschberg@jordanbelfort.com` is outside the tenant. Implications:
  - nothing lead-level may appear in the body
  - the tenant must allow external sending from the `it.admin@` mailbox (verify with j.fernandez / microsoft-learn)
- **Closes GAP-02 and GAP-03 by removing attachments entirely.** The report zip (`run.py:1152-1171`) and the inline attachment (`run.py:1208-1209`) go away for this email. It also removes the first memory spike in Stage 6 (run report: +500 MiB while building the success email).
- Summary table source: `generate_report`, `run.py:1064-1097` (`summary_df`: Metric / Count / Actions Taken).
- Renamed from "Success email carries only the report" (2026-10-29).

## Open questions
- [ ] #open-question Does the tenant allow external sending from `it.admin@` to `jordanbelfort.com`? Check with j.fernandez / microsoft-learn before the first real send.

## Dependencies
- [[Story - Report labels and unmatched warning fixed]] (the corrected labels appear in this email).
