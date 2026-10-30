---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Metrics dashboard]]"
sprint: S4
moscow: tbd
status: to-do
jira: ""
size: S
---
# Story: Spike dashboard stack and hosting

## User Story
**As** the development team, **I want** a time-boxed exploration of dashboard technology and hosting options, **so that** we choose a stack that fits the project's constraints (Docker runtime, PostgreSQL data source, may need to host on ITServer or `autobot`) without over-committing before we understand the trade-offs.

## Acceptance Criteria
- [ ] **Given** the spike is time-boxed (3–5 days maximum), **when** the spike explores Streamlit vs Dash vs Bokeh, **then** each option is evaluated on: ease of connecting to PostgreSQL, Docker packaging, deployment simplicity, ability to handle the expected metric count (~8–10 views), and whether it supports future auth/access control if needed.
- [ ] **Given** the hosting decision (ITServer vs `autobot`) is unresolved, **when** the spike evaluates both, **then** each option is assessed on: networking (can the dashboard reach PostgreSQL?), reverse proxy setup (NPM on `autobot` already exists; ITServer would need nginx or Traefik), SSL/domain setup, and resource usage (ITServer already runs the DNC process + PostgreSQL; adding a dashboard may be tight).
- [ ] **Given** the spike produces findings, **when** documented (suggested: `Docs/S4-Dashboard-Stack-Spike.md`), **then** the document includes: a **recommendation** (which stack, which host), trade-offs for each option, proof-of-concept code or screenshots (optional but helpful), and any blockers discovered (e.g., "Bokeh's PostgreSQL adapter is unmaintained," "autobot's firewall blocks PostgreSQL port").
- [ ] **Given** the spike is complete, **when** the recommendation is reviewed with Michelle, **then** a decision is made and recorded. If the decision differs from the recommendation, the rationale is documented.

## Notes
- This is **Pattern 9: Spike** — time-boxed exploration, delivers a recommendation, not production code.
- If the spike reveals a blocking issue (e.g., "autobot cannot reach PostgreSQL due to network segmentation, and VPN setup is a 2-week IT ticket"), escalate to Michelle immediately rather than burning the time-box trying to solve it.
- Proof-of-concept: a minimal "hello world" dashboard in each stack (Streamlit/Dash/Bokeh) that connects to a local PostgreSQL and displays one hardcoded metric is sufficient to assess ease-of-use. Do not build the real metrics yet.
- The spike may conclude "all three stacks work fine; Streamlit is simplest" or "Dash is the only one with a maintained PostgreSQL connector" — either outcome is valid as long as the reasoning is clear.
- **Do not extend the time-box.** If 5 days pass and the decision is still unclear, the spike failed; escalate the decision to Michelle with the partial findings rather than continuing exploration.

## Dependencies
- **Depends on:** S2 migration done+stable (PostgreSQL is the data source) and S3 M5 passed (validated metrics exist to display)
- **Blocks:** [[Story - First dashboard page from validated metrics]] (can't build without knowing the stack)
