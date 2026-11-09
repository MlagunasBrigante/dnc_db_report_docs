---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
status: in-review
moscow: could
gaps: [GAP-12, GAP-13]
jira: ""
size: S
---
# Story: Mac run-night safeguards until cutover

> [!todo] Michelle — action needed
> - [x] Run the container recreate command on the Mac and set the wake/no-sleep schedule.
> - [ ] Keep the Mac awake and on power on Nov 30 from 19:45 (Mexico City) until the run finishes.
> - [ ] Morning of Dec 2: confirm `data/2026-12-01/logs/run.log` ends with the final status line; otherwise run `once` by hand.

## User Story
**As** the pipeline owner, **I want** the scheduled container on the Mac to point at the renamed project folder and the Mac to stay awake on run nights, **so that** the monthly runs keep landing in the right place until the job moves to the ITServer.

## Acceptance Criteria
- [x] **Given** the project folder was renamed `DNC-Update` → `dnc-cron-scrub`, **when** the `dnc-scrub` container is recreated from the new path, **then** `docker inspect dnc-scrub` shows the bind mount at `.../Projects/dnc-cron-scrub/data` and `docker logs dnc-scrub` shows the next run at 02:00 UTC on the 1st.
- [x] **Given** a run night (the 1st, 02:00 UTC, which is 20:00 the previous evening Mexico City time), **when** the Mac would normally sleep, **then** a wake/no-sleep schedule keeps it awake (e.g. `pmset repeat wakeorpoweron`, or `caffeinate` while the run is going).
- [ ] **Given** the morning of the 2nd, **when** Michelle checks, **then** `data/<YYYY-MM-01>/` exists with a completed `run.log`. Otherwise she runs `once` by hand.

## Notes
- **2026-11-09:** `dnc-scrub` recreated from `main` `95dc32c` (image stamped `95dc32c`); scheduler log: next run 2026-12-01T02:00:00+00:00.
- **2026-11-06 (Michelle):** `docker inspect` shows the mount at `…/Projects/dnc-cron-scrub/data`; container recreated from `efe2647`, next run 2026-12-01 02:00 UTC. `sudo pmset repeat wakeorpoweron MTWRFSU 19:45:00` set (`pmset -g sched`: wakepoweron 7:45PM every day). **Decision:** no further sleep changes (no `pmset sleep 0`, no caffeinate job); Michelle keeps the Mac awake on run night by hand because the ITServer cutover is next. Remaining: the Dec 2 check (AC 3).
- **Could have** (Michelle, 2026-10-29). The last manual run was 2026-10-28, so there's almost a month of buffer, and a missed Nov 1 run is recoverable with a manual `once`.
- Recreate command (from the project root):
  ```bash
  docker rm -f dnc-scrub
  docker run -d --name dnc-scrub --restart unless-stopped \
    --env-file Dev/.env -e TZ=UTC -v "$PWD/data:/data" dnc-scrub:latest
  ```
- Retired at cutover ([[Story - Cut over and complete a scheduled run on the ITServer]]).

## Dependencies
- None.
