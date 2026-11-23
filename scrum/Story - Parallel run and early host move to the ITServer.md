---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Cutover to the ITServer]]"
sprint: S2
moscow: must
s2_order: 0
status: done
gaps: [GAP-13, GAP-37, GAP-38, GAP-39]
jira: ""
size: M
---
# Story: Parallel run and early host move to the ITServer
> [!success] Done 2026-11-23: parallel run passed; the ITServer is the production host (next run 2026-12-01 02:00 UTC). The Mac's `dnc-scrub` is stopped, not removed.

## User Story
**As** `<owner>` (pipeline owner), **I want** the Mac and the ITServer to run the same real month once, side by side, and the ITServer to take over the monthly job when the results match, **so that** the move off the Mac is proven on real data before 2026-12-01 and S2 continues on the target host.

## Acceptance Criteria
- [x] **Given** both hosts, **when** the parallel run starts, **then** both images are built from the same `Dev/` commit on `main` (verified by `CODE_VERSION` in both run logs / memory-log rows). The Mac's live `dnc-scrub` container isn't recreated for this; only the throwaway `--rm ... once` containers use the rebuilt image.
- [x] **Given** the ITServer `.env` (typed by hand, D-26), **when** it is checked before the run, **then** every variable name in `.env.example` that a real run needs is present (names only, values never printed), and the report, alert and removal-request recipients equal the Mac's.
- [x] **Given** both hosts are ready, **when** `<owner>` starts a real `once` run on each within a few minutes of each other, on the same UTC date, **then** both Stage 1 downloads finish before either run reaches Stage 4 (verified by the Stage 1 end and Stage 4 start timestamps in both logs). Reason: Stage 4 writes `Summary: Scrubbed Date: <report_date>` to every tagged lead and Stage 1 drops leads scrubbed in the last 90 days (D-31).
- [x] **Given** both runs finished, **when** the outputs are compared, **then** the pass/fail checks in Notes (Comparison) all pass or every difference is explained in this story.
- [x] **Given** the ITServer run is real, **when** Stage 1 completes, **then** its raw snapshot is written to the production `dnc` schema and `tools/verify_raw_snapshot.py --schema dnc` returns MATCH (D-32).
- [x] **Given** the comparison passes, **when** the host moves, **then** the Mac's `dnc-scrub` container is stopped (not removed), and the ITServer runs a persistent self-scheduling container (`--restart unless-stopped`, image default `SCHEDULE_CRON`, Postgres socket volume mounted) whose log shows the next run at 2026-12-01 02:00 UTC — all before 2026-11-30 20:00 CST.
- [x] **Given** the host moved, **when** a rollback is needed, **then** written steps exist and were checked: stop the ITServer container, `docker start dnc-scrub` on the Mac, `once` on the Mac if a month was missed.
- [x] **Given** ITServer runs have no off-machine copy yet (R6), **when** each ITServer run finishes, **then** the interim copy chosen in Open questions exists until the backups story ships. *Amended by D-34 (2026-11-23): no copy leaves the server; run folders and `dnc` persist on the ITServer only, and R6 stays open until the backups story.*
- [x] **Given** the host moved, **when** the docs are reviewed, **then** [[ITServer PostgreSQL setup]] § "No real runs on the ITServer before cutover", [[DNC-runbook]] and `Dev/README.md` name the ITServer as the production host. *Docs done 2026-11-23; `Dev/README.md` is carried to the next `Dev/` branch (order 2), since `Dev/` is committed per story.*

## Notes
- **Created 2026-11-20** from D-31/D-32: the one parallel run happens as soon as the blocking items are closed, before 2026-12-01, and the ITServer takes over right after (option (i)). The physical move is split in two:
  1. **this story:** host move with today's pipeline (SQLite system of record, raw snapshot also written to Postgres; the image's own scheduler, like the Mac);
  2. [[Story - Cut over and complete a scheduled run on the ITServer]], at the end of S2: switch to systemd timers and the Postgres pipeline on the same host (criteria 3–5 of D-31).
- **If the comparison isn't satisfactory by 2026-11-30 20:00 CST:** the Mac's scheduled run on 2026-12-01 02:00 UTC runs as usual; the ITServer must not schedule that run. Retry the move for 2027-01-01.
- **Heads-up, 90-day filter:** leads scrubbed in the parallel run are skipped by the 2026-12-01 run (and so on for 90 days), so the Dec 1 run will be small. Same effect the 2026-11-10 ad-hoc run had.
- **Report date is the UTC date** (`TZ=UTC` in the image, `run.py` `report_date`): starting one run before and one after 18:00 CST gives two different Scrubbed Dates.
- **Duration:** Stage 1 is ~27 minutes on the ITServer (429 pages, verified 2026-11-19); the Mac must stay awake for the whole run.
- **Race window is small (2026-11-10 real run, memory log):** Stage 1 28.3 min, Stage 2 6 s, Stage 3 < 1 s, then Stage 4 ~26 min of LawRuler writes. A host's Stage 4 starts ~28 min after its start, so start both within ~1 minute. Any lead that the slower host drops in Stage 1 *because its Scrubbed Date already equals the run date* was written by the faster host mid-download: an explained difference, counted in the comparison.
- **Mac `.env` has no Postgres:** the Mac run logs `Postgres writing disabled (POSTGRES_DSN not set)`; only the ITServer writes `dnc`.
- **Comparison (accepted D-33):** per run folder, compare and record side by side:
  - Stage 1: leads downloaded and leads selected after the 90-day filter (equal, or difference explained by leads changed between the two downloads);
  - Stage 2: phones scrubbed, result-code distribution, failed batches;
  - Stage 3/4: `leads_with_tags` rows (lead, phone, tag) — set difference = 0 or explained; LawRuler update counts and failures;
  - Stage 5: report summary table identical; both removal requests list the same leads;
  - run outcome (success/degraded), failed-item counts, duration per stage;
  - spot-check a few tagged leads in LawRuler: Scrubbed Date = the run date (the fingerprint of the test, `<owner>` 2026-11-20).

## Result (2026-11-21 → 2026-11-23)
- **Start:** both `once` runs at 2026-11-21 00:37:34 UTC, 0.5 s apart, both `b71cf98`, report_date 2026-11-21. Routing settings checked by fingerprint (12/12 equal; the Mac `.env` has CRLF endings, harmless under `--env-file`). The ITServer was missing `REPORT_RECIPIENTS_TO/CC`; filled by hand before the run.
- **Comparison: identical on every stage** (counts and set fingerprints computed on both hosts from `interim/scrub_staging.db` and `processed/scrub_ods.db`): 428,864 leads downloaded; 362,261 selected after the 90-day filter; 363 scrub batches done; 358,698 scrub results; 362,181 tagged leads (lead + phone + code + tags fingerprint equal); 130,572 with a DNC tag; 362,181 LawRuler writes done, 0 failed. Outcome success on both.
- **Postgres:** one `dnc.pipeline_run` row (manual, not dry-run, 428,864 rows); `verify_raw_snapshot.py --schema dnc` = MATCH.
- **LawRuler spot check (`<owner>`):** Scrubbed Date 2026-11-21 present, two Summary entries per lead (one per host): the expected fingerprint.
- **Stage 4 lasted ~54 h** (Mac ends 2026-11-23 07:13 UTC, ITServer 02:41, ~4.5 h faster): the 2026-08-12 full scrub came due (358,767 of the selected leads), ~0.5 s per LawRuler write. Expected volume, not a fault; the full scrub was paid on both hosts and the 2026-12-01 run will be small (GAP-39).
- **Findings fixed during the run:**
  - The ITServer report CSV/XLSX and `run.log` were lost: its `.env` (from `.env.example`) set `OUTPUT_DIR=/output` and `LOG_DIR=/logs`, which real runs honour (dry-runs ignore them), so they were written inside the `--rm` container. Both lines removed 2026-11-23; only `DATA_DIR=/data` remains (GAP-37). The SQLite databases were unaffected.
  - The success/report email failed on both hosts (Graph 400, one recipient missing its `@`). Fixed in both `.env` 2026-11-23; business report for this run not resent (`<owner>`). Removal request and Teams went out (GAP-38).
- **Takeover 2026-11-23:** Mac `dnc-scrub` stopped and set to `--restart=no` (kept for rollback, still on its old image); ITServer `dnc-scrub` started `--restart unless-stopped` with the data and Postgres-socket volumes; log shows `Next run: 2026-12-01T02:00:00+00:00`. Docker is enabled at boot on the ITServer.
- **Rollback steps:** [[DNC-runbook]] § 0.

## Open questions
- [x] ✅ 2026-11-20 (`<owner>`, D-33): yes, run folder copied into the Mac's OneDrive `data/` + `pg_dump` of `dnc` after each ITServer run, until the backups story ships. ~~#open-question~~ **Interim off-machine copy of ITServer runs** until [[Story - Backups and restore drill for the dnc database]] ships: e.g. copy each ITServer run folder back into the Mac's OneDrive `data/` after the run (today's de facto off-machine copy), plus a `pg_dump` of `dnc`. The Mac's own `data/` stays where it is either way.
- [x] ✅ 2026-11-20 (`<owner>`, D-33): keep `/home/it.admin/dnc/data` mounted at `/data`. ~~#open-question~~ **Production data path on the ITServer:** keep `/home/it.admin/dnc/data` (used by the setup guide and the dry-runs) or move to the `/srv/dnc-scrub/...` layout of [[Story - Monthly run from versioned systemd units]] now. Moving later means moving the legal record twice.
- [x] ✅ 2026-11-20 (`<owner>`, D-33): today if nothing blocks it. It was already 2026-11-21 UTC when decided, so both runs carry report_date 2026-11-21. ~~#open-question~~ **Parallel-run date:** any day before 2026-11-30, both runs on the same UTC date before 18:00 CST, the Mac kept awake for its whole run.
- [x] ✅ 2026-11-20 (`<owner>`, D-33): yes, both images rebuilt from `main` `b71cf98` (b35a022..b71cf98 is masking only, no behaviour change). ~~#open-question~~ **Same code on both hosts:** rebuild both images from the same `main` commit (the Mac's live `dnc-scrub` container is not recreated; only the `--rm ... once` containers use it)? Claude recommends yes.
- [x] ✅ 2026-11-20 (`<owner>`, D-33): accepted as proposed in Notes. ~~#open-question~~ **Comparison checklist:** accept, or edit, the checks proposed in Notes (Comparison).

## Dependencies
- [[Story - Raw snapshot and run record land in Postgres]] (done 2026-11-20)
- Before [[Story - All stages write to Postgres with lineage]] and the rest of S2, which continue on the ITServer
