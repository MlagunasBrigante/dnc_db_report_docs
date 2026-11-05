---
type: story
project: dnc-cron-scrub
epic: "[[Epic - No silent data loss]]"
sprint: S1
moscow: must
s1_order: 5
status: done
gaps: [GAP-16, GAP-14]
jira: ""
size: S
---
# Story: Report labels and unmatched warning fixed

## User Story
**As** the IT & Data lead, **I want** report labels to accurately describe the data (e.g. "Total Leads Scrubbed" should be leads scrubbed, not all raw_data) and the dead unmatched-records warning removed, **so that** the report is honest and the logs don't emit false warnings.

## Acceptance Criteria
- [ ] **Given** `generate_report` (`run.py:1065-1091`), **when** the Summary table is built, **then** it has exactly these rows and columns (`Metric | Count | Action`), with the counts computed as today:

  | Metric | Count source (unchanged) | Action |
  |---|---|---|
  | Total leads in LawRuler | all `raw_data` (snapshot) | — (snapshot) |
  | Leads scrubbed this run | `full_report` (not scrubbed in last 90 days) | Done by the job: scrubbed and tagged in LawRuler |
  | …of which not a case | `~case_mask` | Done by the job: scrubbed and tagged |
  | DNC leads without opt-in | not case & DNC & not opt-in | **Will be removed by the 7th of <month>** |
  | Litigators to soft-delete | …& Litigator & not in an active process | **Will be removed by the 7th of <month>** |

  `<month>` is the run's month (e.g. "Will be removed by the 7th of November"). The xlsx Summary sheet and the report email use the same table.
- [ ] **Given** the old labels ("Total Leads Scrubbed", "Older than 90 Days & Untagged", "Non-Cases", "Not Opt-In DNC Leads"), and the action "Soft Deleted", which the job never did, **when** the change ships, **then** none of them appear in the report or the email.
- [ ] **Given** the "unmatched records" warning in `_translate_tags` (`run.py:738-745`), **when** the change ships, **then** it is removed: the inner join drops unmatched rows, so it can never fire (GAP-14).
- [ ] **Given** one real run, **when** the report email arrives, **then** the table matches the table above, its counts match the run log's `Summary:` block, and no "unmatched records" warning appears in the log.

## Notes
- **Mac dry-run 2026-11-04** (`data/_dryrun/2026-11-04T190951Z/`, `DRYRUN_INJECT_FAILURES=page:5,batch:1`; all five S1 branches merged and pushed, `main` @ `33558a0`): the report Summary shows the new labels ("Total leads in LawRuler 423,266 — (snapshot)" and the rest, ending with "Raw snapshot complete"). The old labels are gone, and no "unmatched records" warning appeared (1 WARNING in the run, the injected page). The email-table part of the AC moves to [[Story - Monthly report email as a summary table]]: the email has no table until that story ships.
- **Code references** (verified against `Dev/run.py`):
  - Report summary: `run.py:1072-1094` — "Total Leads Scrubbed" label; count from `run.py:1061` (`total_leads` from `raw_data`)
  - Misleading labels: GAP-16 — "Misleading report labels (\"Total Leads Scrubbed\" = all `raw_data`)" at `generate_report`
  - Unmatched warning: `run.py:738-745` — merge with `how='inner'` at `run.py:738`, then check `leads_to_tag['ResultCode'].isna()` at `run.py:743-745`, which can never be true because the inner join drops unmatched rows
  - GAP-14: "\"Unmatched records\" warning can never fire (inner join)"
- **Root cause**: the report was written when the counts were not yet instrumented, so labels were guessed. The unmatched warning was copied from a left-join version and never updated when the join became inner.
- **Scope**: fix the labels in `generate_report` to match the actual data sources, and delete the dead unmatched warning in `_translate_tags`.
- **Decided 2026-10-29 (stories interview Q6):** labels and actions as in the AC table. The job **never soft-deletes**; it only tags. The two removal rows are carried out by the **LawRuler team on email request** ([[Story - Soft-deletion request to the LawRuler team]]). "By the 7th" fits the failure-protection cadence: run on the 1st, patch pass on the 3rd, watchdog on the 5th, removal by the 7th. Automating the soft-deletion is deferred to S6 ([[Epic - Automated soft-deletion]]).
- Reference counts (2026-10-28 run): 421,776 / 7,713 / 7,631 / 2,243 / 4.
- This is a quick fix (S-sized) — label changes and deleting dead code.

## Dependencies
- Feeds [[Story - Monthly report email as a summary table]] and [[Story - Soft-deletion request to the LawRuler team]].
