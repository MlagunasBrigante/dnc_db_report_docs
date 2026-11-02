# dnc-cron-scrub — Documentation

Planning, reference and run-history docs for the **DNC Scrub Pipeline**. Every month the pipeline scrubs LawRuler lead phone numbers against dncscrub.com, tags the matching leads in LawRuler, builds an Excel/CSV report and sends results through Microsoft 365.

The code lives in a separate repo, [`dnc_db_report`](https://github.com/MlagunasBrigante/dnc_db_report) (the `Dev/` folder). Its `README.md` covers stages, env vars, volumes and Docker usage. This repo ([`dnc_db_report_docs`](https://github.com/MlagunasBrigante/dnc_db_report_docs)) holds documentation only, no code.

## Current state

- **Production:** the `dnc-scrub` Docker container on Michelle's Mac fires at 02:00 UTC on the 1st of every month. Each run is stored in SQLite under `data/{date}/`.
- **Plan:** re-scoped on 2026-10-29. The goal is a monthly scrub that is reliable, observable and auditable on the AlmaLinux ITServer with PostgreSQL, followed by statistical modeling, a metrics dashboard and archiving. The physical cutover is deliberately not rushed.

## Layout

| Folder | Contents |
|---|---|
| [`scrum/`](scrum/) | **Plan of record.** Holds the scope, backlog, prioritization/risk register, 14 epics and 52 stories |
| [`Reference/`](Reference/) | Current reference material: dncscrub.com (CCC) API PDFs and design decisions |
| [`Run Reports/`](Run%20Reports/) | Per-run analyses (evidence, failures, memory profile) |
| [`Archive/`](Archive/) | Notebook-era and Azure-Function-era notes from 2025. Kept for history and not maintained |

## Start here

1. [DNC Update - Scope](scrum/DNC%20Update%20-%20Scope.md) covers scope by sprint, out of scope, Definition of Done, the **gap register (GAP-01…28)** and **decision items (D-01…07)**.
2. [Backlog - dnc-cron-scrub](scrum/Backlog%20-%20dnc-cron-scrub.md) lists sprints S1–S6 with their epics and stories as a checkbox list (ticked only when the item's `status` is `done`).
3. [Prioritization and Risk Register](scrum/Prioritization%20and%20Risk%20Register.md) has the MoSCoW ranking and pre-mortem risks.
4. [DNC Run Report 2026-10-28](Run%20Reports/DNC%20Run%20Report%202026-10-28.md) is the baseline run used for gap-closure and migration scoping.

## Sprints

| Sprint | Focus |
|---|---|
| S1 | Gap-closing: reliable alerts, lean and secret-free logs, no silent data loss, documentation, memory |
| S2 | Technical migration: PostgreSQL system of record, failure capture and patch pass, systemd scheduling, historical backfill, ITServer cutover |
| S3 | Statistics: data readiness and modeling (milestones M1–M5) |
| S4 | Metrics dashboard |
| S5 | Archiving closed months |
| S6 | Automated soft-deletion (optional; only if D-07 = automate) |

## Conventions

- **Files are named by type:** `Epic - <outcome>.md`, `Story - <outcome>.md`, `Decision - <topic>.md`, `DNC Run Report YYYY-MM-DD.md`.
- **Frontmatter properties:** stories carry `type`, `epic`, `sprint`, `moscow`, `status`, `gaps`, `jira` (legacy, left empty) and `size`. Priority comes from the `moscow` property. Stories in S3–S6 stay at `moscow: tbd` until that sprint's prioritization pass.
- **Open questions** are tagged `#open-question` inside the story or epic that needs them, and are answered when that story starts.
- **Owner actions:** Michelle's to-dos are `[!todo]` callouts in stories. Each epic has a `[!check]` list that confirms those to-dos were done before the epic closes.
- **Reviews:** annotate directly in the files (`> [!question]`, `> [!warning]`, `**MICHELLE:** …`) rather than in chat. The next pass handles only what was annotated.
- **Links:** notes are written for Obsidian and use `[[wikilinks]]`, which GitHub shows as plain text. Browse in Obsidian for working links.
- **Jira:** deprecated on 2026-10-29. Scrum lives only in this repo; the old `DNCUP` keys survive as historical references.

## Data and secrets

This repo must never contain lead data, PII, credentials or run logs. Run data stays in `data/` next to this folder (bind-mounted into the container, not committed), and secrets stay in `Dev/.env`. Run reports may quote aggregate counts and log excerpts only after secrets are masked.
