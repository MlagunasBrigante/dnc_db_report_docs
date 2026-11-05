---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
status: archived
gaps: [GAP-02]
jira: ""
size: S
---

> [!note] Archived 2026-11-05
> Superseded by [[Story - Soft-deletion request to the LawRuler team]] (stories interview Q3/Q6, 2026-10-29). Kept for history only; not part of the backlog.
# Story: Litigator action list for soft-deletes

## User Story
**As** the person who performs the manual soft-deletes in LawRuler, **I want** an internal email listing the litigators to soft-delete each month, **so that** I can act on the "Litigators to Soft Delete → Manually Soft Deleted" line of the report now that the report email carries no attachments.

## Acceptance Criteria
- [ ] **Given** a run where "Litigators to Soft Delete" is greater than 0, **when** the run finishes, **then** a separate **internal-only** email is sent with the list (the same columns as today's xlsx "Litigators" sheet, built by `litigators_to_delete` in `generate_report`) as a table in the body, or as a small csv attachment if it doesn't fit in a readable table.
- [ ] **Given** the count is 0, **when** the run finishes, **then** no action-list email is sent, and the report email shows 0.
- [ ] **Given** the recipient config (`ACTION_LIST_RECIPIENTS` in `Dev/.env`), **when** the email is built, **then** every recipient must be in the internal domain (`briganteimmigration.com`). An external address makes the run log an error and skips the send, so lead-level data never leaves the tenant.
- [ ] **Given** a size guard (e.g. 3 MB), **when** an unusually large list would exceed it, **then** the email says how many rows there are and where the full list is stored (run output folder now, Postgres after S2) instead of attaching it, and an alert is raised.
- [ ] Verified with a `once --dry-run` that has litigators (email built and recipient check passes) and one with 0 litigators (no email).

## Notes
- **Decided 2026-10-29 (stories interview Q3):** a separate internal "action list" email. It's sent only when there's something to act on and never goes to the external report recipient. From S4, the dashboard may replace it.
- Volume reference: 184 known litigators as of 2025-08-04 (`Docs/Archive/DNC Short Report 2025-08-04.md`). Monthly incremental lists are expected to be much smaller.
- **Recipient(s): to be named by Michelle** (who performs the soft-deletes today). Until named, it defaults to the alert owner (Michelle).
- Litigator definition: not a case, DNC, not Single Opt-In, Litigator, and Status not in `ACTIVE_PROCESS_STATUSES` (`run.py:50-71`). Decision D-01 (soft vs hard delete) may change the action wording later.

## Dependencies
- [[Story - Monthly report email as a summary table]] (shares the email-building code; this story restores the lead-level list that the report email dropped).
