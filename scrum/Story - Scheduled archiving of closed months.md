---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Archiving closed months]]"
sprint: S5
moscow: tbd
status: to-do
jira: ""
size: M
---
# Story: Scheduled archiving of closed months

## User Story
**As** IT, **I want** closed months automatically archived on a schedule, **so that** archiving happens reliably without manual intervention and the live PostgreSQL database doesn't grow indefinitely.

## Acceptance Criteria
- [ ] **Given** the archive procedure (Story 2) is documented, **when** a script automates it (suggested: `Dev/archive/archive_month.sh` or `Dev/archive/archive_month.py`), **then** the script: (1) identifies closed months (per the retention policy from Story 1), (2) dumps each month's partitions with `pg_dump`, (3) checksums the dump (SHA-256), (4) copies to the archive target (disk/cloud), (5) verifies the checksum on the destination, (6) records the archive in a manifest (JSON/CSV/database), and (7) optionally drops the original partition from production (if the retention policy says "archive = remove from live").
- [ ] **Given** the script is written, **when** tested manually on one closed month (not the same month as Story 2's drill; use a different month to prove repeatability), **then** it completes without errors, the month is archived with a verified checksum, the manifest is updated, and the partition is dropped (if policy allows).
- [ ] **Given** the script works, **when** a systemd timer (unit files `dnc-archive.service` and `dnc-archive.timer`) is created, **then** the timer runs the script monthly (e.g., on the 15th, after the DNC scrub's monthly run and patch pass are done) with logging to `/var/log/dnc-archive.log` or similar, and the unit files are versioned in `Dev/deploy/` so the setup is reproducible.
- [ ] **Given** the timer is installed, **when** tested (trigger the timer manually with `systemctl start dnc-archive.service` or wait for the next scheduled run), **then** the archiving completes successfully, the log shows no errors, and the manifest is updated.
- [ ] **Given** archiving fails (e.g., archive target is unreachable, checksum mismatch, `pg_dump` errors), **when** the error is logged, **then** an alert is raised (email or Teams, reusing the S1 alert channels) so IT knows to investigate, and the original partition is **not dropped** (failure = safe, don't delete data).

## Notes
- This story automates the **real archival** — unlike Story 2's drill, this one drops partitions from production (if the retention policy allows).
- The script must be **idempotent** — if run twice on the same month, it should detect the month is already archived (check the manifest) and skip it, not fail or create duplicate archives.
- The schedule (monthly on the 15th) is a suggestion; adjust based on when the DNC scrub's monthly run finishes. The timer should not run while the scrub is still processing the current month.
- If the retention policy (Story 1) has multiple tiers (e.g., "archive to disk, then migrate to cloud after 2 years"), this story implements the **first tier only**. The second-tier migration can be a follow-up story.
- The manifest should track: month archived, archive path, checksum, archive date, and whether the partition was dropped from production. This lets IT audit what's been archived and restore a specific month if needed.
- **Restorability is tested in Story 2**; this story assumes the procedure works and focuses on automation. If a real restore is needed post-automation, the procedure from Story 2 is followed (retrieve from manifest, verify checksum, `pg_restore`).
- Output: a working systemd timer, the archive script versioned in the repo, and the setup documented in `Dev/deploy/README.md` or a dedicated `Dev/archive/README.md`.

## Dependencies
- **Depends on:** [[Story - Archive and restore one closed month]] (automation requires a proven procedure)
- **Closes:** GAP-25 (4.1 GB of lead PII now has a retention policy and archiving mechanism)
