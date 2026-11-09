---
type: log
project: dnc-cron-scrub
last_synced_dev_commit: efe26472877bc5ab14b7d942551106937f05056c
tags: [project/dnc-cron-scrub, doc-sync]
---
# Doc-sync log

Written by the weekday 07:00 CST Hermes cron job "dnc-cron-scrub doc sync". Each run aligns the docs to `Dev` `main`: it auto-fixes factual drift and lists judgment calls under **Needs Michelle**. Newest entry on top. The job stops once the project is marked finished (project `README.md` status `✅ Done`/`🔴 Closed`, or the folder moved to `Projects/_Closed/`).

> [!info]- Job definition (schedule `0 7 * * 1-5`, skill `obsidian`, deliver `local`)
> Daily documentation-sync job for the dnc-cron-scrub project. Run autonomously (no questions). Keep output terse and factual.
>
> Locations (Docker sandbox):
> - Project folder: /workspace/vault/Projects/dnc-cron-scrub
> - Code repo (git, branch main): /workspace/vault/Projects/dnc-cron-scrub/Dev
> - Docs repo (git): /workspace/vault/Projects/dnc-cron-scrub/Docs
> - Log note: /workspace/vault/Projects/dnc-cron-scrub/Docs/Doc-sync log.md (frontmatter key last_synced_dev_commit)
> - Project manual: /workspace/vault/Projects/dnc-cron-scrub/CLAUDE.md (read it first and follow it)
>
> Hard rules:
> - Never read, print or edit Dev/.env. Never touch the project data folder. Never write secrets, lead data or PII anywhere.
> - Never git commit, push, checkout, stash, reset or branch in either repo. Edit working-tree files only.
> - Never change story or epic status, backlog checkboxes, scope, gap register, decisions or MoSCoW; those go to the report.
>
> Step 0, stop condition: if the project README status line says Done or Closed, or the project folder no longer exists (moved under Projects/_Closed), write nothing and reply only: dnc-cron-scrub is finished, remove the cron job named dnc-cron-scrub doc sync.
>
> Step 1, change gate: H = output of git rev-parse main in the code repo. If H equals last_synced_dev_commit in the log note, insert one line under the marker comment 'entries below, newest first': '- YYYY-MM-DD: no change on Dev main (short H).' Reply with that line and stop. Otherwise collect git log --oneline from the last synced commit to main (last 20 commits if the value is none) and git diff --stat between them.
>
> Step 2, audit the docs against Dev main. Read code from the main commit with git show main:PATH and git ls-tree -r main --name-only, not from the working tree (it may be on a feature branch). Check:
> a) Dev/README.md and Dev/.env.example vs code on main: every environment variable the code reads (name, required, default), stages, CLI flags, entrypoint commands, Dockerfile defaults, volumes, output layout, module list, test instructions.
> b) The project CLAUDE.md: layout table, key files, commands and repo facts still true.
> c) The project README.md (SPEAR format): Current Status table matches each story's frontmatter status in Docs/scrum; the Point line is not contradicted.
> d) Docs/README.md and the intro of Docs/scrum/Backlog - dnc-cron-scrub.md: counts of Epic and Story files and the GAP-xx / D-xx ranges in Docs/scrum/DNC Update - Scope.md.
> e) Wikilinks in those files resolve to an existing note somewhere under /workspace/vault.
> f) Judgment signals, report only: commits on main that look like they implement a story still marked to-do or in-review; backlog checkbox ticks that do not match status (ticked only when status is done); new TODO/FIXME or failure modes in the diff with no GAP in the Scope gap register; unresolved questions lacking the #open-question tag.
>
> Step 3, auto-fix factual drift only (items a to e) with minimal targeted edits in the existing style. Edit Dev/README.md and Dev/.env.example only when the code repo's current branch is main; otherwise list the proposed edit in the report. .env.example holds placeholders only.
>
> Step 4, log: insert a new entry directly under the marker comment in the log note (newest on top), then set last_synced_dev_commit to H. Entry format: a heading '## YYYY-MM-DD: Dev main <short H>', then **Commits since last sync** (one-line list or 'first baseline'), **Auto-fixed** (bullets: file, change, evidence path:line; or none), **Needs Michelle** (bullets of judgment items and anything not auto-fixed, each with evidence and a suggested action; or none). Keep it under about 25 lines. Final reply: the entry text.

<!-- entries below, newest first -->

## 2026-11-09: Dev main efe2647
**Commits since last sync:** 6 (9afc77b…efe2647): alert-delivery gaps (success message, config-failure Teams alert, `dnc/config_failure_alerts.py`), per-stage duration/memory tracking (`dnc/stage_metrics.py`), review fixes. 10 files, +1456/−175. The 2026-11-06 README/.env.example fixes are now on main.
**Auto-fixed:**
- `Dev/README.md:235-240`: 6 broken `||` env rows repaired (again).
- `Dev/README.md:179`: dropped stale "`cron.log` from the host" (no host cron; `Dockerfile` CMD `schedule`).
- `Dev/README.md:242,260`: "email failures never change the exit code" → exit 3 when every channel fails (README Exit Codes, `run.py` alert outcomes).
- `Dev/README.md:345`: per-stage metrics are on the Teams **success** card only, not warning (`run.py:2355` passes `metrics`; `send_warning_teams_alert` `run.py:1908` has no metrics arg).
- `Dev/README.md` Local Development: added test command (`pytest.ini`, `requirements-dev.txt`).
- `README.md` (project): Current Status +1 row, [[Story - Per-stage duration and memory in every run]] 🔍 In Review; Point line updated.
**Needs Michelle:**
- Stories `in-review` but merged on main: Per-stage duration and memory (a3ca7c5, efe2647; container recreated, story line 24), Alert delivery failure (fa17869), Teams alert channel (405d125). Move to `merged — pending Mac verification` if review is done.
- `CLAUDE.md:34,36` (edit blocked again, protected file): `dnc/` helper list stale (add alert delivery, config check/failure alerts, Teams, soft-delete request, run status/outcome, stage metrics, log excerpt, report email/summary); `Docs/Reference/` row says runbook + I/O spec "not written yet" but `DNC-runbook.md`, `pipeline-stages-reference.md` exist. Edit by hand.
- `Dev/README.md:62,69,104,106,114,169`: paths still `Projects/DNC-Update/...`; confirm the live `dnc-scrub` mount, then update.
- `Dev/README.md:197` + project `README.md:84`: session cookie still "verify on a real run" though the cookie story is `done`; reword or close.
- `law_ruler_client_id/secret` marked required but absent from `dnc/config_check.py:10-17`; add to the check or mark optional.
- `Docs/README.md:37`, Backlog:98 "only if D-07 = automate" vs D-07 decided; check wording.
- Soft-deletion email (between Stages 5 and 6) is not timed by stage metrics (story line 24); no GAP. Accept or log a gap.
- Uncommitted: `Dev/README.md` (this run, on main); project `README.md` is outside both repos.

## 2026-11-06: Dev main 05df099
**Commits since last sync:** 16 (378e54a…05df099): degraded-run polish, soft-deletion review email (D-10), Teams Workflows webhook channel + masking, alert-delivery tracking/exit code 3, README exit codes. 21 files, +2182/−180.
**Auto-fixed:**
- `Dev/README.md`: re-fixed 12 broken `||` env rows; re-added env vars missing on main: `*_RETRY_*` ×9 (`dnc/retry.py:73-95`), `OUTAGE_*` ×3 (`retry.py:245-247`), `LOG_LEVEL`/`LOG_LIBRARY_DEBUG` (`dnc/log_setup.py:35,39`), `TEAMS_WEBHOOK_URL`, `DRYRUN_TEAMS_WEBHOOK_URL`, `DRYRUN_RETENTION_DAYS` (`run.py:2155`); `DATA_DIR` code default `./data`, container sets `/data` via `.env` (`run.py:2046`, `.env.example:57`).
- `Dev/.env.example`: commented retry-policy defaults block; commented `SCHEDULE_CRON` default (`Dockerfile:10`).
- `README.md` (project): Current Status table synced to story `status` (+6 rows: 3 done, 3 in-review; report email now "awaiting first real run with business recipients"); Point line updated.
- `Docs/README.md:23`, Backlog intro line 10: D-01…09 → D-01…10 (`Docs/scrum/DNC Update - Scope.md:214`).
**Needs Michelle:**
- Yesterday's `Dev/README.md`/`.env.example` fixes were never committed and are gone from the working tree (redone above). Commit them on main this time.
- `CLAUDE.md:34,36` (edit blocked, protected file): `dnc/` helper list stale (add alert delivery, Teams, soft-delete request, run status/outcome, config check, log excerpt, report email/summary); `Docs/Reference/` row says runbook and I/O spec "not written yet", but `DNC-runbook.md` and `pipeline-stages-reference.md` exist. Edit by hand.
- Teams alert channel + Alert delivery failure stories are `in-review` but merged on main (405d125, 0bc3dd5, 05df099). Move to `merged — pending Mac verification` if review is done.
- Project `README.md:83` Open Gaps + `Dev/README.md` `LAWRULER_SESSION_COOKIE` row still say "unconfirmed / verify on real run" though the cookie story is `done`; reword or close.
- `Dev/README.md`: 6 paths still `Projects/DNC-Update/...`; confirm the live `dnc-scrub` mount, then update.
- `law_ruler_client_id/secret` marked required but not in `dnc/config_check.py`; add to the check or mark optional.
- `Docs/README.md:37` S6 "only if D-07 = automate" vs D-07 decided; check wording.
- Dev working tree has uncommitted README/.env.example edits from this run.

## 2026-11-05: Dev main ea3d03c
**Commits since last sync:** 20 (528f6b6…ea3d03c): lead-download truncation, failed batches reported, dry-run failure injection, config check before API, report labels, compact log excerpt on alerts, HTML summary report email, `_zip_output` removed, GAP-29 degraded run + review fixes. 22 files, +2804/−188.
**Auto-fixed:**
- `Dev/README.md`: repaired 6 broken `||` env rows; added 17 missing env vars with defaults (DRYRUN_*, OUTAGE_*, *_RETRY_*, LOG_LEVEL, LOG_LIBRARY_DEBUG) (`dnc/retry.py:73-95,235-237`, `dnc/log_setup.py:35,39`, `dnc/email_helpers.py:31`, `run.py:1713`); SCRUB_URL/LOGIN_ID real-run-only (`dnc/config_check.py:74`); DATA_DIR code default `./data` (`run.py:1613`); O365 partial-config fails startup (`run.py:1675-1681`); EMAIL_RECIPIENTS_* defaults (`run.py:1649-1651`); dropped stale host `cron.log`; added pytest command (`pytest.ini`, `requirements-dev.txt`).
- `Dev/.env.example`: commented retry-policy defaults block; O365 comment "email/Teams" → email only, enabled when any O365_* set (`run.py:1675`).
- `README.md` (project): Current Status table synced to story `status` (9 done, 2 merged-pending); Point line updated to match.
- `Docs/README.md`: 52→53 stories; GAP-01…29, D-01…09 (`Docs/scrum/DNC Update - Scope.md:199,213`). Backlog intro: same GAP/D ranges.
**Needs Michelle:**
- `CLAUDE.md` Layout row: `dnc/` helper list is stale (now also config check, fault injection, log excerpt, report email/summary, run outcome). Edit blocked (protected agent-instruction file); apply by hand. Branch examples (`s1/patch/one-retry-policy`) still valid.
- `Dev/README.md:62,69,104,106,114,143`: paths still `Projects/DNC-Update/...`; `Projects/DNC-Update/data/` still exists, so the live `dnc-scrub` mount likely still points there. Confirm the mount (`docker inspect dnc-scrub`), then update the README or recreate the container.
- 4 Story files are not linked from the Backlog (57 files vs 53 listed), all `to-do`: Success email carries only the report, Litigator action list for soft-deletes, Rotate exposed credentials and purge git history, Stage 1 streams pages instead of holding them. Delete if superseded or add to the Backlog.
- Commit c433d66 (zip attachment removed) overlaps to-do [[Story - Success email carries only the report]]; check whether it is now covered.
- Project README Open Gaps still says the session-cookie need is unconfirmed, but the cookie story is `done`; resolve or re-word. `Docs/README.md:37` S6 "only if D-07 = automate" vs D-07 decided (email request); check wording.
- `.env.example`/README mark `law_ruler_client_id/secret` required, but `dnc/config_check.py:64-71` does not validate them. Add to the check or mark optional.
- Docs repo has 13 uncommitted scrum edits (working tree); commit when ready.
- 2026-11-04: no change on Dev main (b3b5a85).
- 2026-11-03: no change on Dev main (b3b5a85).

## 2026-11-02: Dev main b3b5a85 (baseline)
**Commits since last sync:** first baseline (manual doc-maintenance pass in chat).
**Auto-fixed:** see the progress note in [[Story - README, Jira and vault references re-scoped]].
**Needs Michelle:** review both READMEs; decisions D-01…D-06 have no owner or date; PostgreSQL placement decision still proposed.
