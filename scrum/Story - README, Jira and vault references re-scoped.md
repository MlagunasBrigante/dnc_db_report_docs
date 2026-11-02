---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Project documentation and housekeeping]]"
sprint: S1
moscow: should
status: to-do
gaps: [GAP-23]
jira: ""
size: M
---
# Story: README, Jira and vault references re-scoped

> [!todo] Michelle — action needed
> - [ ] Review the README rewrite.

## User Story
**As** a new contributor or backlog consumer, **I want** the project README, Dev/README, Jira tickets, and vault references to reflect the Container→Dev and DNC-Update→dnc-cron-scrub renames, **so that** docs are current and the Jira↔vault sync works.

## Acceptance Criteria
- [ ] **Given** the project root README.md and `Dev/README.md`, **when** they are updated, **then** all references to `Container/` are changed to `Dev/` and all references to `DNC-Update` are changed to `dnc-cron-scrub`.
- [ ] **Given** Jira tickets DNCUP-2, DNCUP-3, DNCUP-6, **when** they are reviewed, **then** they are closed as "Superseded by the S1-S5 backlog" (if they are fully covered by new stories) or re-scoped to point to the specific S1 story that addresses them (if still relevant).
- [ ] **Given** Jira tickets DNCUP-4 and DNCUP-5, **when** they are reviewed, **then** DNCUP-4 is mapped to [[Story - Runbook for failed or degraded runs]] and DNCUP-5 is mapped to [[Story - Per-stage inputs and outputs reference]] (update Jira description or comments to link to the vault story).
- [ ] **Given** Jira ticket DNCUP-7, **when** it is reviewed, **then** it is kept open (if still relevant) or closed (if superseded); document the decision in the ticket.
- [ ] **Given** the Jira↔vault sync SOP (`Knowledge/Project Management/Jira-Vault-Sync-SOP.md`), **when** the `vault_path` for the DNC project is updated, **then** it points to the renamed folder (`Projects/dnc-cron-scrub/`), a sync baseline reset is performed (per the SOP), and Michelle approves the change before the sync runs.
- [ ] **Given** the vault CLAUDE.md §16, **when** stale references to removed `.cursor` folders are found, **then** they are deleted or updated to reflect the current project structure.

## Notes
- **Progress 2026-11-02 (doc maintenance pass):** project `README.md` rewritten in SPEAR format for the re-scope; `Dev/README.md` paths moved to `dnc-cron-scrub/`, env-var table completed (dry-run, logging, retry, outage), dry-run section unified, `/output`/`/logs` mounts dropped; `Dev/.env.example` `OUTPUT_DIR`/`LOG_DIR` commented out and retry vars added. **Remaining:** Michelle's review (todo above), and vault-wide references outside this project that still say `DNC-Update` (vault `CLAUDE.md` §8.7 example, `Knowledge/Project Management/Jira-Vault-Sync-SOP.md`, `Knowledge/Reference/Jira-Mirror.md`) plus the §16 `.cursor` cleanup.
- **Jira is fully deprecated (Michelle, 2026-10-29).** Scrum lives in the vault (`Docs/scrum/`). Ignore any acceptance criteria or notes below about re-scoping DNCUP tickets or fixing the Jira↔vault sync mapping; there's no risk of losing tracking. What remains: rewrite the project `README.md`, update `Dev/README.md` for the renames, and clean up stale vault references (e.g. vault `CLAUDE.md` §16 `.cursor` paths). The vault-wide Jira sync (vault `CLAUDE.md` §9/§12, scheduled `jira-sync-*` tasks) is outside this project; flag it to Michelle separately.
- **Gap reference**: GAP-23 — "Stale README and Jira (DNCUP-2/3/6 superseded); docs still say `Container/`; Jira↔vault sync broken for the renamed folder; stale vault CLAUDE.md §16 references to removed .cursor folders"
- **Rename history** (from scope open item 2):
  - `Container/` → `Dev/` (directory rename)
  - `DNC-Update` → `dnc-cron-scrub` (project name and vault folder rename)
  - `Scrub_API/` triaged and removed (2026-10-29)
- **Jira ticket inventory** (from scope):
  - DNCUP-2, DNCUP-3, DNCUP-6: likely superseded by the S1-S5 backlog (review and close or re-scope)
  - DNCUP-4: "Runbook for DNC scrub failures" → [[Story - Runbook for failed or degraded runs]]
  - DNCUP-5: "Document each stage's data contract" → [[Story - Per-stage inputs and outputs reference]]
  - DNCUP-7: kept (scope says "kept"); confirm it's still relevant or close
- **Jira↔vault sync** (from scope):
  - SOP location: `Knowledge/Project Management/Jira-Vault-Sync-SOP.md`
  - Action: update `vault_path` for the DNC project to `Projects/dnc-cron-scrub/`, perform a sync baseline reset (per the SOP), and get Michelle's approval before running the sync
- **Vault CLAUDE.md §16**: scan for references to `.cursor` or removed folders (e.g. `Scrub_API/`, `Container/`) and update or delete them
- **Scope**: this is a housekeeping story — text edits, Jira updates, and sync config. No code changes.
- This is M-sized because it touches multiple artifacts (README, Jira, SOP, CLAUDE.md) and requires careful review of Jira tickets.

## Dependencies
None — can start immediately.
