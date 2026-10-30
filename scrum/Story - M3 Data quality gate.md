---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Data readiness for modeling]]"
sprint: S3
moscow: tbd
status: to-do
milestone: M3
jira: ""
size: M
---
# Story: M3 Data quality gate

## User Story
**As** compliance and Michelle, **I want** explicit quality criteria defining "clean enough to model," **so that** we can decide whether to proceed with statistical modeling or halt and clean the data first.

## Acceptance Criteria
- [ ] **Given** M1 and M2 are complete, **when** Michelle and compliance define quality criteria (e.g., "≤5% invalid phone numbers," "≤2% missing opt-in status," "no duplicate Lead IDs within a partition," "DNC result codes are all recognized values"), **then** those criteria are documented as testable rules (suggested: `Docs/S3-M3-Data-Quality-Criteria.md`).
- [ ] **Given** quality criteria are defined, **when** SQL queries or Python scripts test them against the PostgreSQL stage tables, **then** each criterion passes/fails with a measured value (e.g., "3.2% invalid phones" → PASS, "12% missing opt-in" → FAIL).
- [ ] **Given** any criterion fails, **when** the failure is reviewed with Michelle and compliance, **then** a decision is recorded: (a) acceptable, proceed anyway with a caveat in the model, (b) fixable in S3 with a small query or script, or (c) blocks S3, spawn a data-cleaning story.
- [ ] **M3 go/no-go criterion:** Michelle and compliance explicitly decide "the data is clean enough to model" or "halt and clean first." If the decision is "halt," S3 pauses and new stories are written for data cleaning; M4 and M5 do not start until M3 re-passes. The decision and its rationale are documented.

## Notes
- **M3 quality criteria are "to be defined"** (per the scope doc) — this story includes the definition process, not just the testing.
- This is a **decision-making story** as much as a technical story. The acceptance criteria force an explicit go/no-go, not a drift into "we'll model it and see."
- Quality checks may reveal issues introduced during S2 backfill (e.g., a batch of legacy runs has schema mismatches) — if so, flag them for S2 follow-up but decide whether they block modeling.
- Reference but do not resolve **Decision D-01** (soft vs hard delete of known litigators) — if it affects data quality (e.g., "litigator" tags are missing), document the dependency but don't block on the decision unless Michelle says it's a blocker.
- Output: quality criteria doc, test queries/scripts (versioned in `Dev/` or `Docs/scripts/`), and a decision record (markdown file or issue comment) stating the go/no-go outcome.

## Dependencies
- **Depends on:** [[Story - M1 Data and pipeline audit]] (can't test quality without knowing the data structure) and [[Story - M2 Notes ingested and preprocessing aligned]] (quality criteria depend on Michelle's statistical assumptions)
- **Blocks:** [[Story - M4 Model tested with sample data]] (no modeling until data is confirmed clean enough)
- **May spawn:** data-cleaning stories if M3 fails and the decision is "clean first"
