---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Automated soft-deletion]]"
sprint: S6
moscow: tbd
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Spike LawRuler soft-delete via API

## User Story
**As** Michelle, **I want** a time-boxed investigation (max 2 days) into whether and how the LawRuler API can soft-delete leads, **so that** I can decide (D-07) whether to automate the monthly removals or keep the email request to the LawRuler team.

## Acceptance Criteria
- [ ] **Given** the LawRuler API (`api-legalcrmapp.aspx`, the Custom Report API) and the LawRuler team, **when** the spike ends, **then** a short recommendation in `Docs/Reference/` answers:
  - Is soft-delete supported, through which call, and is it reversible?
  - What are the rate limits?
  - What audit trail does LawRuler keep?
- [ ] **Given** a test lead agreed with the LawRuler team, **when** the call is exercised (if supported), **then** the lead is soft-deleted and restored successfully.
- [ ] **Given** the recommendation, **when** Michelle reviews it, **then** D-07 is decided and, if "automate", follow-up stories are split.

## Notes
- Starts only after S5. The removal deadline stays "by the 7th of <month>", in line with the run → patch → watchdog cadence.
- The flaky-API rule applies to any future automation: capture failures and re-process them in a patch pass; never fail the whole run.

## Dependencies
- S5 complete; decision D-01 (soft vs hard delete for litigators) ideally settled.
