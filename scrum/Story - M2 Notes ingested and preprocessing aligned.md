---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Data readiness for modeling]]"
sprint: S3
moscow: tbd
status: to-do
milestone: M2
jira: ""
size: S
---
# Story: M2 Notes ingested and preprocessing aligned

> [!todo] Michelle — action needed
> - [ ] Feed your notes on preprocessing into the project.

## User Story
**As** Michelle, **I want** the preprocessing logic in the pipeline aligned with the statistical assumptions I'm documenting in my notes, **so that** the data transformations don't invalidate the model before I even start building it.

## Acceptance Criteria
- [ ] **Given** Michelle feeds notes into the project (format TBD: markdown files in `Docs/`, comments in issues, or a shared doc), **when** those notes describe preprocessing requirements (e.g., "exclude leads with no phone," "treat NULL opt-in as FALSE," "age bucket definitions: <90 days, 90–180 days, >180 days"), **then** each requirement is recorded in a tracking document (suggested: `Docs/S3-M2-Preprocessing-Alignment.md`) with a status: already implemented / needs code change / deferred.
- [ ] **Given** a preprocessing requirement needs a code change (e.g., filtering logic in a query, a new calculated column), **when** implemented, **then** it is tested with a sample query against the PostgreSQL stage tables and the result matches Michelle's expectation.
- [ ] **Given** Michelle's notes reference a metric or transformation (e.g., "DNC → Litigator → not a case"), **when** that path is traced through the stage tables, **then** the SQL or Python logic producing it is documented and verified to match her definition.
- [ ] **M2 go/no-go criterion:** Michelle confirms the preprocessing logic aligns with her statistical assumptions, and any mismatches are either fixed or explicitly deferred with her agreement. No model-building (M4) starts until this passes.

## Notes
- This story is **lightweight on engineering effort** — Michelle leads, engineering implements small changes and validates queries.
- If a preprocessing change affects how the pipeline writes data (e.g., a new column in a stage table), that is **out of scope for S3** — flag it as a follow-on S2 story or defer to a future sprint. S3 works with the data as backfilled.
- The tracking document cross-references Michelle's notes (by date or section) so the reasoning is traceable.
- If Michelle's notes are informal (Slack messages, verbal discussions), this story includes **capturing them in writing** so the alignment is documented, not just verbal.

## Dependencies
- **Depends on:** [[Story - M1 Data and pipeline audit]] (can't align preprocessing without knowing the data structure)
- **Blocks:** [[Story - M4 Model tested with sample data]] (model assumes preprocessing is correct)
