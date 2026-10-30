---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Lean and secret-free logs]]"
sprint: S1
moscow: must
s1_order: 1
status: to-do
gaps: [GAP-24, GAP-07]
jira: ""
size: M
---
# Story: Rotate exposed credentials and retire the old repo

> [!todo] Michelle — action needed
> - [ ] Confirm who can rotate each credential (LawRuler admin, dncscrub, Entra app).
> - [ ] Delete the `DNC_LOCALSERVER` repo on GitHub as `MlagunasBrigante`, after the `git bundle` backup is verified.
> - [ ] Set the recurring reminders for any credential that can't be rotated yet.

## User Story
**As** the compliance/legal owner, **I want** every credential that appeared in git, logs or legacy files rotated, and the deprecated `DNC_LOCALSERVER` GitHub repo deleted after a local backup, **so that** the exposed secrets are useless and no copy of them stays online.

## Acceptance Criteria
- [ ] **Given** the exposure inventory (see Notes), **when** rotation is done, **then** each live credential is regenerated at its provider, the new value is only in `Dev/.env` (never committed, never passed with `docker run -e`), and the old value no longer authenticates (tested).
- [ ] **Given** the rotated credentials, **when** a `once --dry-run` runs, **then** it authenticates against LawRuler, dncscrub and Graph and completes. A real run follows before the story closes.
- [ ] **Given** the `DNC_LOCALSERVER` repo, **when** it is retired, **then**:
  1. a full backup is taken first: `git bundle create … --all` into `Dev/Archive/dnctag_local_py/` (gitignored), verified with `git bundle verify`
  2. the repo is **deleted on GitHub**, from settings, acting as **`MlagunasBrigante`** (vault `CLAUDE.md` §15)
  3. the repo URL returns 404
- [ ] **Given** inline passwords in `Dev/Archive/notebooks/*.ipynb` and the token notebook in the archived `dnctag_local_py`, **when** clean-up is done, **then** those values are redacted (`***REDACTED***`) or the files deleted, per Michelle's call at the time. A grep for the known secret patterns over `Dev/Archive/` returns zero matches (counts only).
- [ ] **Given** a credential can't be rotated now because other flows depend on it, **when** the story closes, **then** it's listed as *pending* in the rotation log, with its owner and a reminder that repeats until it's rotated. The story can close with pending items; the reminders carry them.
- [ ] A short log at `Docs/Reference/Credential Rotation Log.md` records what was rotated (names only, never values), when, by whom, and how it was verified.

## Notes
- **Decided 2026-10-29 (stories interview Q5): option A.** Delete the GitHub repo instead of rewriting its history. The project is deprecated and its full history survives in the local archive and the bundle. **Rotation happens first:** deleting the repo doesn't undo past exposure.
- **Exposure inventory** (names only):
  - **LawRuler:** OAuth client secret, API username/password and account key. These were in the committed `.env` of `DNC_LOCALSERVER`, and the client secret is in DEBUG run logs.
  - **dncscrub:** login ID. It was in the committed `.env`. Rotate if the provider allows.
  - **Graph/O365 app registration client secret:** the O365/oauthlib DEBUG logging can echo it in token requests. Confirm by counting matches in the historic logs before deleting them ([[Story - Scrub secrets from historic run logs]]). If present, rotate it in Entra. This is M365, so check with the microsoft-learn MCP (vault `CLAUDE.md` §13) and with j.fernandez as needed.
  - **`dnctag_local_py` endpoint API token and Postgres password:** the service is retired, so there's nothing to rotate; just make sure no live system reuses those values.
- **Other flows may depend on these credentials (Michelle, 2026-10-29).** Before rotating each one, list who and what else uses it. Rotate what we can safely. For any credential we can't rotate yet, because another flow or owner depends on it, create a **reminder** (a recurring Jira task or calendar entry, e.g. every 2 weeks) to push its owner until it's rotated. The rotation log records each one as rotated / pending (owner, next reminder).
- Who can rotate each provider (LawRuler admin, dncscrub account, Entra app) gets confirmed when the story starts.
- Renamed from "Rotate exposed credentials and purge git history" (2026-10-29). The git-history rewrite and force-push are no longer needed.

## Open questions
- [ ] #open-question Which other flows and systems use each credential? Rotating can break them (Michelle, 2026-10-29).
- [ ] #open-question Who can rotate each credential (LawRuler admin, dncscrub account, Entra app registration)?

## Dependencies
- Rotate before [[Story - Scrub secrets from historic run logs]] deletes the originals, and count the secret matches first so the inventory above is confirmed.
- [[Story - INFO logs with secret masking]] should be in place so new runs don't leak the new values.
