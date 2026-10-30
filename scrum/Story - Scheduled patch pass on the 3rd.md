---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reproducible scheduling on the ITServer]]"
sprint: S2
moscow: must
status: to-do
gaps: []
jira: ""
size: S
---
# Story: Scheduled patch pass on the 3rd
## User Story
**As** IT operations, **I want** the patch pass to run automatically on the 3rd of every month, **so that** transient failures from the 1st are retried without manual intervention.

## Acceptance Criteria
- [ ] **Given** the repo is updated with the patch pass units, **when** `Dev/deploy/install.sh` is executed, **then** it installs `dnc-scrub-patch.service` and `dnc-scrub-patch.timer` to `/etc/systemd/system/`, and enables the timer (verified by `systemctl list-timers` showing `dnc-scrub-patch.timer` with next run on the 3rd at 02:00 UTC)
- [ ] **Given** the patch timer fires on the 3rd, **when** it executes, **then** the service runs `docker run --rm -v ... dnc-scrub:latest once --patch <last_run_id>` where `<last_run_id>` is the most recent run's run_id from `pipeline_run` (verified by checking the service's ExecStart command or a wrapper script that queries Postgres)
- [ ] **Given** the patch service runs, **when** it completes, **then** systemd logs the exit code and the patch pass's stdout/stderr (verified by `journalctl -u dnc-scrub-patch.service` showing "Patch pass completed: 3 resolved, 1 unresolved")
- [ ] **Given** Persistent=true on the patch timer, **when** the server is down on the 3rd and boots on the 4th, **then** the missed patch pass fires after boot (verified by systemd journal)
- [ ] **Given** the patch pass is scheduled, **when** no run occurred on the 1st (e.g. the run timer failed or was disabled), **then** the patch pass logs "No recent run found, skipping patch" and exits 0 (verified by testing with an empty `pipeline_run` table or a run older than 5 days)

## Notes
- **Unit files:** `dnc-scrub-patch.service` and `dnc-scrub-patch.timer` (OnCalendar=*-*-03 02:00:00 UTC, Persistent=true)
- **Getting last_run_id:** The service's ExecStart can call a wrapper script `Dev/deploy/run-patch.sh` that queries `SELECT run_id FROM pipeline_run WHERE start_time >= NOW() - INTERVAL '5 days' ORDER BY start_time DESC LIMIT 1` and passes it to `--patch`
- **No recent run:** If the query returns no rows, the wrapper logs a warning and exits 0 (don't fail the systemd unit)
- **Install script:** `install.sh` is updated to install both the run and patch units

## Dependencies
- [[Story - Patch pass reprocesses a run's failed items]] (the --patch CLI mode must exist)
- [[Story - Monthly run from versioned systemd units]] (install.sh and the unit-file pattern are established)
