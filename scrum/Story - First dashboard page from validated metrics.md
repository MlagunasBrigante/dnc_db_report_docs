---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Metrics dashboard]]"
sprint: S4
moscow: tbd
status: to-do
jira: ""
size: M
---
# Story: First dashboard page from validated metrics

## User Story
**As** management and compliance, **I want** one complete metric displayed on a working dashboard, **so that** I can see real DNC data in a browser and verify the end-to-end flow (PostgreSQL → backend → frontend → deployment) works before scaling out to all metrics.

## Acceptance Criteria
- [ ] **Given** the spike (Story 1) chose a stack and hosting location, **when** the dashboard is built, **then** it uses that stack (Streamlit/Dash/Bokeh) and runs in Docker on the chosen host (ITServer or `autobot`).
- [ ] **Given** one validated metric from S3 M5 (e.g., "opt-in vs not opt-in count"), **when** the dashboard queries PostgreSQL, **then** it retrieves the data using the SQL/Python logic validated in M5, and displays it as a chart or table (implementation: bar chart, pie chart, or simple two-column table — whatever fits the metric).
- [ ] **Given** the dashboard is running, **when** accessed in a browser (localhost or via reverse proxy if on `autobot`), **then** the metric displays correctly, the page loads in <5 seconds, and the data matches the M5 validation output (spot-check one sample month).
- [ ] **Given** the dashboard is deployed, **when** Docker is restarted or the host reboots, **then** the dashboard comes back up automatically (systemd service or Docker restart policy), and the metric still displays.
- [ ] **Given** the deployment is reproducible, **when** the setup is documented (suggested: `Docs/S4-Dashboard-Deployment.md` or a `README` in `Dev/dashboard/`), **then** another developer can rebuild the dashboard from the repo without tribal knowledge (Dockerfile, docker-compose.yml or equivalent, and any reverse-proxy config are versioned).

## Notes
- This is **one complete vertical slice** — one metric, end-to-end, proving the stack works. Do not build multiple metrics yet; that's Story 3.
- If the chosen host is `autobot`, this story includes configuring the NPM reverse proxy to route a subdomain or path (e.g., `dnc.example.com` or `autobot.local/dnc`) to the dashboard container. If on ITServer, decide whether to expose a port or set up nginx/Traefik.
- The metric chosen should be **simple** (a count or percentage, not a multi-dimensional breakdown) to keep the story focused on proving the stack, not on complex UI.
- If PostgreSQL connection issues arise (e.g., `autobot` cannot reach ITServer's PostgreSQL due to firewall), this story includes resolving them (open firewall rule, use SSH tunnel, or escalate if it's a broader infrastructure issue).
- Output: a working dashboard, Dockerized and deployed, serving one metric from PostgreSQL, with deployment docs in the repo.

## Dependencies
- **Depends on:** [[Story - Spike dashboard stack and hosting]] (can't build without knowing the stack and host)
- **Blocks:** [[Story - Remaining metrics on the dashboard]] (can't scale out until the first slice works)
