---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reproducible scheduling on the ITServer]]"
sprint: S2
moscow: must
s2_order: 7
status: to-do
gaps: [GAP-12, GAP-37]
jira: ""
size: M
---
# Story: Monthly run from versioned systemd units
## User Story
**As** IT operations, **I want** the monthly DNC run triggered by a systemd timer (not the in-container scheduler), **so that** the schedule survives container rebuilds and server reboots, and the setup is reproducible from the repo.

## Acceptance Criteria
- [ ] **Given** the repo is cloned on the ITServer, **when** `Dev/deploy/install.sh` is executed, **then** it copies `dnc-scrub-run.service` and `dnc-scrub-run.timer` to `/etc/systemd/system/`, runs `systemctl daemon-reload`, and enables the timer (verified by `systemctl list-timers` showing `dnc-scrub-run.timer` with next run on the 1st at 02:00 UTC)
- [ ] **Given** the timer is installed, **when** the ITServer is rebooted before the 1st (simulated in a test VM), **then** the timer remains enabled and the next run still fires on the 1st (verified by `systemctl status dnc-scrub-run.timer` after reboot)
- [ ] **Given** the timer is configured with `Persistent=true`, **when** the server is down on the 1st and boots on the 3rd, **then** the missed run fires shortly after boot (verified by systemd journal showing "Missed timer, running now")
- [ ] **Given** the service unit runs, **when** it executes, **then** it runs `docker run --rm -v /srv/dnc-scrub/data:/app/data -v /srv/dnc-scrub/Dev/.env:/app/.env dnc-scrub:latest once` (or equivalent), and the exit code is propagated to systemd (verified by `systemctl status dnc-scrub-run.service` showing success/failure)
- [ ] **Given** the install script is idempotent, **when** it is run twice, **then** the second run does not fail or duplicate the units (verified by running `install.sh` twice and checking `systemctl list-units` shows one timer, not two)
- [ ] **Given** GAP-12 (in-memory scheduler can skip a month on restart), **when** the in-container `scheduler.py` is reviewed, **then** it is either removed or documented as deprecated (verified by checking `Dev/scheduler.py` is deleted or has a "DO NOT USE" comment at the top)

- [ ] **Given** the monthly run's service exits non-zero or is killed, **when** systemd records the failure, **then** `OnFailure=dnc-scrub-notify-failure@%n.service` sends an email and a Teams alert; the same notifier is used by the patch-pass and watchdog units (aligned 2026-10-29, interview Q9). Verified by forcing a failing run in a test.

## Notes
- **Unit files:** `dnc-scrub-run.service` (runs the container) and `dnc-scrub-run.timer` (OnCalendar=*-*-01 02:00:00 UTC, Persistent=true)
- **Install script:** `Dev/deploy/install.sh` is idempotent (uses `cp -f` or checks if units exist), requires root, and prints "DNC scrub monthly run installed" on success
- **Volume mounts:** The service unit must mount `data/` and `Dev/.env` so the container has access to the run history and secrets
- **Logging:** systemd journal captures stdout/stderr; no need for separate log forwarding in S2
- **Deprecating scheduler.py:** The in-container scheduler (`Dev/scheduler.py`) is replaced by systemd; either delete it or add a warning that it's no longer the production path

## Open questions
- [x] ✅ 2026-11-20 (`<owner>`, D-33): data stays at `/home/it.admin/dnc/data` mounted at `/data`, clone at `/home/it.admin/dnc/git_container/dnc_db_report`; align this story's ACs (`/srv/dnc-scrub/...`, `/app/data`) when it starts. ~~#open-question~~ **Production paths on the ITServer** (found 2026-11-20): the ACs here use `/srv/dnc-scrub/...` mounted at `/app/data`, while the setup run used `/home/it.admin/dnc/data` mounted at `/data` (see [[ITServer PostgreSQL setup]] Step 11). Settle the production data and clone paths when this story starts, then align this story and [[Story - Cut over and complete a scheduled run on the ITServer]].

- [x] ✅ 2026-11-23 (`<owner>`, D-35): **dependency kept (path B)**; this story stays at order 7, after order 2. ~~#open-question~~ **Drop the dependency on order 2?** (2026-11-23, D-35) This story is pulled ahead of order 2 so backups can ship sooner, and the timer should replace the in-container scheduler before 2026-12-01. That only works if the dependency below is dropped. Reasoning: this story moves the *clock* (who starts the run), order 2 changes the *engine* (what the run does); the unit runs whatever image is current, so it works with today's pipeline and picks up order 2 after a rebuild. The original dependency was sequencing, not a technical need. Risk if dropped: a wrongly configured unit fails on Dec 1 → guarded by keeping `dnc-scrub` running until the timer passes one dry-run through the service. If kept: systemd and backups wait for order 2, the switch moves to Jan 1, R6 stays open longer. Awaiting `<owner>`.

## Notes (2026-11-23, D-35)
- The service must also mount the `dnc-pgsocket` volume (order 1 writes the raw snapshot to Postgres); align the `docker run` AC with the production paths of D-33.
- Switch rule (kept from the path-A analysis): retire the in-container scheduler (stop `dnc-scrub`) only after one dry-run through `dnc-scrub-run.service` passes, so a month is never left without a scheduler.

## Dependencies
- [[Story - All stages write to Postgres with lineage]] (the pipeline must be fully functional before scheduling it) — kept 2026-11-23 (D-35, path B)
