---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Cutover to the ITServer]]"
sprint: S2
moscow: must
s2_order: 13
status: to-do
gaps: [GAP-13, GAP-25]
jira: ""
size: M
---
# Story: Cut over and complete a scheduled run on the ITServer
## User Story
**As** `<owner>` (pipeline owner), **I want** the DNC scrub job running on the AlmaLinux ITServer with the old Mac container retired, **so that** Mac sleep no longer pauses runs and PII is off OneDrive.

## Acceptance Criteria
- [ ] **Given** the cutover acceptance criteria from Story - Define cutover acceptance criteria, **when** each criterion is tested, **then** all pass (e.g. parallel run matches, alerts verified, rollback plan documented) — verified by checking off each criterion in that story or a cutover checklist
- [ ] **Given** the ITServer is prepared, **when** the `data/` directory is copied to the ITServer (e.g. via rsync; the ITServer `.env` already exists, typed by hand, never a copy of the Mac file, D-26), **then** the ITServer has access to all historical runs and secrets (verified by `ls /srv/dnc-scrub/data/` showing 2026-08-12 through 2026-10-28 plus legacy/)
- [ ] **Given** the systemd timers are installed on the ITServer, **when** `systemctl list-timers` is run, **then** it shows `dnc-scrub-run.timer`, `dnc-scrub-patch.timer`, and `dnc-scrub-watchdog.timer` all enabled with correct next-run dates (verified by systemctl output)
- [ ] **Given** the ITServer is ready, **when** the first scheduled run on the 1st fires (or is triggered manually via `systemctl start dnc-scrub-run.service`), **then** it completes successfully, writes to Postgres, generates the report, and sends the alert (verified by checking `pipeline_run` for the new run, the systemd journal, and the alert inbox)
- [ ] ~~**Given** the first ITServer run succeeds, **when** the Mac container is stopped (`docker stop dnc-scrub`), **then** it no longer runs scheduled jobs~~: moved to [[Story - Parallel run and early host move to the ITServer]] (D-32, 2026-11-20)
- [ ] **Given** GAP-13 (Mac sleep pauses runs), **when** the ITServer's first run is reviewed, **then** it has no sleep-induced pauses (verified by checking the run log for gaps or the run duration being consistent with the 2026-10-28 baseline)
- [ ] **Given** GAP-25 (PII on OneDrive), **when** the data/ directory is moved to the ITServer, **then** the OneDrive `data/` folder is either deleted or moved to a secure local backup (not synced to the cloud) — verified by `<owner>` confirming the OneDrive folder is gone or no longer syncing

## Notes
- **Acceptance criteria (2026-11-20):** judged by D-31 criteria 2–5 from [[Story - Define cutover acceptance criteria]] (alert test, full first cycle 1st/3rd/5th, rollback, container-era backfill).
- **Phase 2 since 2026-11-20 (D-32):** the parallel run and the move of the monthly job to the ITServer happen earlier, in [[Story - Parallel run and early host move to the ITServer]]. This story is now the switch, on the same host, from the image's own scheduler to the systemd timers and the Postgres pipeline, judged by D-31 criteria 2–5.
- **Phase 1 done 2026-11-23:** the ITServer runs the monthly job in a persistent `dnc-scrub` container (data `/home/it.admin/dnc/data`, D-33); rollback steps in [[DNC-runbook]] § 0. The Mac's run history stays on the Mac; no data moves from the ITServer to the Mac (D-34).
- **Cutover checklist (based on Story - Define cutover acceptance criteria):**
  1. Parallel run comparison (Mac vs ITServer) — optional if deferred by `<owner>`
  2. Alert verification (email + Teams) — required
  3. First scheduled run on the ITServer — required (this story's main criterion)
  4. Rollback plan documented — required before cutover
  5. Backfill verified (container-era + compatible legacy) — required
- **Cutover timing:** Schedule the cutover for a low-risk window (e.g. mid-month, after the 5th watchdog has run, so there's no conflict with the 1st run)
- **Rollback plan:** Document in [[ITServer PostgreSQL setup]] (§ Rollback) or [[DNC-runbook]]: stop ITServer timers, restart Mac container, verify last Mac run completed, resume Mac scheduling
- **Data migration:** `rsync -avz data/ itserver:/srv/dnc-scrub/data/` (or equivalent); verify checksums after transfer
- **Mac container retirement:** Stop the container, disable any host-level cron/launchd that might restart it, archive the Mac's `data/` locally as a cold backup

## Dependencies
- [[Story - Define cutover acceptance criteria]] (criteria must be agreed before cutover)
- All other S2 epics done (Postgres, failures, scheduling, backfill)
