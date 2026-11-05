---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Lean and secret-free logs]]"
sprint: S1
status: archived
gaps: [GAP-24]
jira: ""
size: L
---

> [!note] Archived 2026-11-05
> Superseded by [[Story - Rotate exposed credentials and retire the old repo]] (decision: `git bundle` backup plus repo deletion, no history rewrite). Kept for history only; not part of the backlog.
# Story: Rotate exposed credentials and purge git history

## User Story
**As** the compliance/legal owner, **I want** every credential that was committed to GitHub or stored in plaintext in legacy files to be rotated and the git history purged, **so that** exposed secrets are invalidated and cannot be used even if the git history or OneDrive files are compromised.

## Acceptance Criteria
- [ ] **Given** the DNC_LOCALSERVER GitHub repo contains `.env` and a token in `notebooks/manual_lead_creation.ipynb` (now archived at `Dev/Archive/dnctag_local_py`), **when** the rotation completes, **then** the LawRuler OAuth `client_secret` is regenerated in the LawRuler admin console, the new value is updated in `Dev/.env` (not committed), and the old value is invalidated.
- [ ] **Given** the DNC_LOCALSERVER GitHub repo history contains secrets, **when** the purge completes, **then** `git filter-repo` or BFG has removed all commits containing `.env`, `notebooks/manual_lead_creation.ipynb`, and any other files with secrets; the purged repo is force-pushed to GitHub as MlagunasBrigante (vault CLAUDE.md §15); and the old history is backed up locally before the push.
- [ ] **Given** passwords are inline in `Dev/Archive/notebooks/*.ipynb`, **when** the rotation completes, **then** those notebooks are either deleted (if no longer needed) or redacted (replace passwords with `***REDACTED***`), per Michelle's call.
- [ ] **Given** all rotations are complete, **when** I test the new credentials, **then** one `--dry-run` completes successfully using the rotated LawRuler client_secret and any other rotated keys.
- [ ] A summary report is committed to `Docs/Reference/credential-rotation-log.md` stating which credentials were rotated, which git commits/files were purged, verification steps, and Michelle's decisions.

## Notes
- **Gap reference**: GAP-24 — "Secrets committed to `github.com/MlagunasBrigante/DNC_LOCALSERVER` (`.env`, token in `notebooks/manual_lead_creation.ipynb`); inline passwords in `Dev/Archive/notebooks/*.ipynb`"
- **Exposed secrets inventory** (from scope GAP-24):
  1. `.env` in git history → contains LawRuler OAuth `client_secret` and other keys
  2. Token in `notebooks/manual_lead_creation.ipynb` (now archived) → rotate or delete
  3. Inline passwords in archived notebooks → redact or delete per Michelle's call
- **Git history purge**:
  - Use `git filter-repo --path .env --path notebooks/ --invert-paths` to remove all commits touching those files, or BFG Repo-Cleaner
  - Back up the old `.git` directory locally before the purge (e.g. `cp -r .git .git.backup`)
  - Force-push to GitHub as MlagunasBrigante (vault CLAUDE.md §15: "Any push to GitHub must authenticate as `MlagunasBrigante`")
  - Coordinate with Michelle: anyone who has cloned the repo must re-clone after the force-push
- **Credential rotation**:
  - LawRuler OAuth `client_secret`: regenerate in LawRuler admin console (or equivalent provider); update `Dev/.env` locally (never commit)
  - Notebooks: if Michelle says delete, remove the files; if redact, replace passwords with `***REDACTED***`
- **Verification**: run `--dry-run` with the new credentials to confirm they work; grep the new git history for secret patterns to confirm the purge worked
- **Scope constraint**: "Any push to GitHub must authenticate as `MlagunasBrigante`" (vault CLAUDE.md §15). "Scope first, then build" — confirm with Michelle before any force-push.
- This is L-sized because it spans multiple systems (GitHub, LawRuler, Azure, local files) and requires careful coordination (git history rewrite, force-push, credential testing).

## Dependencies
None — can start immediately, but should coordinate with Michelle on timing (force-push affects anyone with a clone).
