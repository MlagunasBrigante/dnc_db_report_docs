---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reproducible scheduling on the ITServer]]"
sprint: S2
moscow: must
status: to-do
gaps: [GAP-12]
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

## Dependencies
- [[Story - All stages write to Postgres with lineage]] (the pipeline must be fully functional before scheduling it)
