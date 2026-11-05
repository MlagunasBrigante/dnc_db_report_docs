---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: must
s1_order: 6
status: merged — awaiting first real run with business recipients
gaps: [GAP-02, GAP-03]
jira: ""
size: S
---
# Story: Monthly report email as a summary table

> [!todo] Michelle — action needed
> - [ ] Ask j.fernandez to confirm the tenant allows external sending from `it.admin@` to `jordanbelfort.com`.

## User Story
**As** a report recipient (management and the team), **I want** the monthly report email to show the run's summary table (counts and actions) in the email body, with no attachments, **so that** I can read the month's DNC outcome at a glance, and the email can never fail because of its size.

## Acceptance Criteria
- [ ] **Given** a successful run, **when** the report email is sent, **then** its body contains the Summary table (`Metric | Count | Actions Taken`, the same rows as the xlsx "Summary" sheet, using the corrected labels from [[Story - Report labels and unmatched warning fixed]]) as an HTML table, plus the report date and run duration. **No attachments.**
- [ ] **Given** the report email is built, **when** its content is inspected, **then** it holds **aggregate counts only, with no lead-level data** (no names, phones or lead IDs), because one recipient is external (see Notes).
- [ ] **Given** the run produces the xlsx and csv, **when** the email is sent, **then** those files are still written to the run's output folder (`data/{date}/processed/`) for internal use; they're just not emailed. `_zip_output` is no longer called for this email.
- [ ] **Given** the recipient config, **when** the email is sent, **then** it goes **to** `brad@briganteimmigration.com`, `r.jimenez@briganteimmigration.com` and `matt.hirschberg@jordanbelfort.com`, with **CC** to `m.lagunas@`, `j.fernandez@` and `e.cervera@` (briganteimmigration.com). Recipients are read from `Dev/.env` (e.g. `REPORT_RECIPIENTS_TO` / `REPORT_RECIPIENTS_CC`), not hardcoded.
- [ ] **Given** a real run, **when** delivery is checked, **then** an internal recipient (Michelle, CC) confirms it arrived, and the external recipient's copy isn't bounced (no NDR in the `it.admin@` sender mailbox).

## Notes
- **Mailbox confirmed by Michelle 2026-11-05**: the report email arrived as an HTML table with no attachments. Still open: set `REPORT_RECIPIENTS_TO`/`_CC` once j.fernandez confirms external sending, then check delivery on the next real run (December 1) with no NDR.
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T232423Z/`, `DRYRUN_EMAIL_RECIPIENTS` set to Michelle): the report email was sent (TO Michelle, no CC) and Graph accepted it. **Waiting on Michelle's mailbox check**: HTML table renders and there are no attachments. The real-run checks (business recipients, external delivery) stay open until `REPORT_RECIPIENTS_TO` is set.
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T222929Z/`, `main` @ `ea3d03c`, `DRYRUN_INJECT_FAILURES=batch:0`): both emails were built, but sending was skipped because `DRYRUN_EMAIL_RECIPIENTS` is unset. Delivery, attachment size and HTML rendering still need one dry-run with it set.
- **In review 2026-11-04** (`94ca49d` + `c433d66` removing the now-unused zip helper): the body is an HTML table built from `build_summary_rows`, plus the date and run duration. No attachments; xlsx and csv are still written. Recipients come from `REPORT_RECIPIENTS_TO`/`_CC`, falling back to today's recipients. No business address is in the code or `.env.example`.
- Branch: `s1/fix/report-email-summary-table`, stacked on `s1/fix/alert-emails-compact-excerpt`. **Safety default:** if `REPORT_RECIPIENTS_TO` isn't set, the report keeps going to today's recipients, so business recipients only start receiving it once Michelle adds them to `.env`.
- Carries over from [[Story - Report labels and unmatched warning fixed]] (2026-11-04): the email table must use the same rows as the xlsx Summary, built by `dnc/report_summary.build_summary_rows`, including "Raw snapshot complete". Its counts must match the run log's Summary block.
- **Decided 2026-10-29 (stories interview Q2):** the report email carries **no files**. It focuses on the actual report: a succinct table of counts and actions. The report format may change after S3 statistical modeling; until then it stays as it is (Summary sheet content).
- **This email and the alert emails are separate audiences.** Warning, error, unresolved-item and watchdog alerts go to `it.admin@` + Michelle, CC j.fernandez and e.cervera, plus the Teams channel (interview Q1). The report email goes to the business recipients above.
- **External recipient:** `matt.hirschberg@jordanbelfort.com` is outside the tenant. Implications:
  - nothing lead-level may appear in the body
  - the tenant must allow external sending from the `it.admin@` mailbox (verify with j.fernandez / microsoft-learn)
- **Closes GAP-02 and GAP-03 by removing attachments entirely.** The report zip (`run.py:1152-1171`) and the inline attachment (`run.py:1208-1209`) go away for this email. It also removes the first memory spike in Stage 6 (run report: +500 MiB while building the success email).
- Summary table source: `generate_report`, `run.py:1064-1097` (`summary_df`: Metric / Count / Actions Taken).
- Renamed from "Success email carries only the report" (2026-10-29).

## Open questions
- [ ] #open-question Can `it.admin@briganteimmigration.com` send to the external domain **`jordanbelfort.com`** (`matt.hirschberg@`)? Ask j.fernandez to check Exchange transport rules, outbound policy and any app access policy on the Graph app. This is the only external recipient in the project, now that the soft-deletion email is internal-only (D-10).

## Dependencies
- [[Story - Report labels and unmatched warning fixed]] (the corrected labels appear in this email).
