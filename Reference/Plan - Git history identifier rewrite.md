---
type: plan
project: dnc-cron-scrub
status: deferred
created: 2026-11-19
gaps: [GAP-35]
tags: [project/dnc-cron-scrub, plan, git, pii, deferred]
---
# Plan — Rewrite git history to remove owner identifiers (deferred)

> **Status:** deferred. The working tree of both repos was masked on 2026-11-19 (placeholders `<owner>`, `<owner-email>`, `<owner-mac>`, `<vault-root>`, `<docs-repo-owner>`). History still carries the original strings. This plan removes them from history when `<owner>` schedules it. Tracked as GAP-35 in [[DNC Update - Scope]].
>
> **2026-11-20 (D-30):** two `Docs/` commits made after the masking pass (`922b745` "Masking names", `5a7a868` "S2 handoff 2026-11-20") carry a personal author identity and are already pushed. They stay as they are; this plan covers them if it is ever run. Enforcement from now on: GAP-36.

## What history still holds

| Where | Dev (`dnc_db_report`) | Docs (`dnc_db_report_docs`) |
|---|---|---|
| Commit author/committer identities with the owner's name, personal and work email, and Mac hostnames in the email domain | most commits | most commits |
| Old blobs with the name, Mac vault paths and `<owner-mac>` references (pre-masking versions of README, docs, comments) | yes | yes |
| Deleted notebook-era files (`notebooks/*.ipynb` and root `*.ipynb`) with outputs and metadata | yes | no |

Exact counts: re-run the audit in step 1; don't trust numbers copied here.

## Steps

1. **Inventory (read-only).** In a fresh mirror clone outside the vault (`git clone --mirror <remote> /var/tmp/hist-<repo>.git`):
   - `git log --all --format='%an <%ae>%n%cn <%ce>' | sort | uniq -c` → every identity to rewrite.
   - `git log --all -S'<string>' --oneline` per original string → blobs to replace.
   - Write the real strings only to `/var/tmp/hist/mailmap` and `/var/tmp/hist/replacements.txt` (never commit them; they are the PII).
2. **Mailmap.** Map every owner identity to the neutral author `DNC Dev <dev@dnc-scrub.local>`.
3. **Replace text.** `replacements.txt` uses the same placeholders as the tip masking (`literal:<original>==><placeholder>`), most specific first (full Mac path → home path → emails → hostnames → GitHub handle → full name → first name → username).
4. **Rewrite** in the mirror: `git filter-repo --mailmap /var/tmp/hist/mailmap --replace-text /var/tmp/hist/replacements.txt`. For Dev, decide whether to also drop the notebook-era paths (`--path notebooks/ --invert-paths`); that overlaps the credential purge (GAP-24) and must be done after rotation.
5. **Verify:** repeat step 1 → zero hits; `git fsck`; diff the rewritten tip against the masked tip (should be empty).
6. **Force-push.** Dev `main` is protected (PR-only): an org admin temporarily allows force-push on `main`, pushes all branches and tags, then restores protection. Docs: force-push as `<docs-repo-owner>`.
7. **Re-clone everywhere:** the shared checkouts in the vault (`Dev/`, `Docs/`), the ITServer clone (see [[ITServer PostgreSQL setup]] step 4), and any worktrees. Old SHAs no longer exist; `CODE_VERSION` stamps in past memory-log rows will point to dead SHAs (record the old→new map from `filter-repo`'s `commit-map`).
8. **GitHub caches:** forks, PR refs and cached views can keep old objects; ask GitHub Support to purge if needed.

## Risks

- Every clone diverges; anyone who pushes from an old clone re-introduces the history. Coordinate before step 6.
- Open PRs break; merge or close them first.
- Docs wikilinks or notes that cite commit SHAs become stale.

## Prerequisites

- No open PRs or unpushed branches in either repo.
- Credential rotation (GAP-24) done first if notebooks are dropped in the same pass.
- `git-filter-repo` available on the machine doing the rewrite.

#open-question When to schedule it, and whether Dev's notebook-era files are dropped in the same pass.
