---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Reliable alerts]]"
sprint: S1
moscow: must
s1_order: 6
status: done
gaps: [GAP-02]
jira: ""
size: M
---
# Story: Soft-deletion request to the LawRuler team

> [!todo] Michelle — action needed
> - [ ] Identify our LawRuler account manager (name, address, domain).
> - [ ] Until then, forward each monthly request yourself.

## User Story
**As** our LawRuler account manager, the person who carries out soft-deletes in LawRuler on our request, **I want** one internal email each month listing the leads to remove (DNC leads without opt-in, plus litigators to soft-delete) and the deadline, **so that** the removals promised in the report ("Will be removed by the 7th of <month>") actually happen on time without anyone compiling the list by hand.

## Acceptance Criteria
- [ ] **Given** a run where "DNC leads without opt-in" or "Litigators to soft-delete" is greater than 0, **when** the lists are final, **then** one email is sent to our LawRuler account manager with:
  - both counts and the deadline ("please soft-delete by the 7th of <month>")
  - each list as a CSV attachment (lead ID plus the columns the team needs to find the lead), with the litigators as a separate list
- [ ] **Given** both counts are 0, **when** the run finishes, **then** no request email is sent.
- [ ] **Given** the recipient config (`SOFT_DELETE_RECIPIENTS` / `SOFT_DELETE_CC` in `Dev/.env`), **when** the email is built, **then** every address must be on an allowlist of domains: `briganteimmigration.com` plus the LawRuler vendor domain, configured in `.env` once the account manager is known. Any other address makes the job log an error and skip the send. Lead-level data may only go to LawRuler, which already holds it, and never to other external recipients such as the report email's external recipient.
- [ ] **Given** an unusually large month (e.g. a full re-scrub: 2026-08-12 had about 130k DNC-tagged leads), **when** the attachments would exceed a size guard (e.g. 3 MB), **then** the email gives the counts and says where the full lists are stored (run output folder now, Postgres after S2), and an alert is raised so Michelle can hand the lists over another way.
- [ ] **Given** timing, **when** the email is sent, **then** in **S1** (Mac, no patch pass yet) it goes at the end of the run; **after S2** it goes once the scheduled patch pass on the 3rd has finished, so the lists already include patched items and the 7th deadline still leaves time.
- [ ] Verified with a `once --dry-run`: the email is built, attachments are under the guard, recipients pass the domain check, and nothing is sent in dry-run unless an explicit test recipient is set.

## Notes
- **Account manager identified 2026-11-05 (Michelle):** Lindsey Humphreys, `lindsey.humphreys@lawruler.com`. Michelle forwards the `[For review]` email to her (manual, D-10). The address stays out of the job's recipients and allowlist.
- **Verified on the Mac 2026-11-05** (`data/_dryrun/2026-11-05T173150Z/`): 200 DNC without opt-in and 10 litigators, the same counts as the report Summary. Two CSVs (11 KB, under the guard) were attached; the recipient passed the internal allowlist and the email was delivered. `s1/fix/soft-delete-internal-review` is merged; the `[For review]` subject and banner show up on the next run. Forwarding to LawRuler stays manual (D-10).
- **Decided 2026-11-05 (Michelle, D-10): internal review first, manual forward.** The job emails the lists only to internal reviewers (`[For review]` subject, review banner). Michelle checks them and forwards to the LawRuler team herself. The allowlist stays internal-only; the "add the LawRuler vendor domain" step is dropped. Branch `s1/fix/soft-delete-internal-review`, which also fixes the deadline: it was the 7th of the *following* month, and is now the 7th of the run's month, matching the report labels.
- **Before the S2 patch pass:** the forward is manual, so the account-manager question only decides who Michelle forwards to, not code.
- **In review 2026-11-05** (`4ee178d`, plus review fixes `d6b7635` and `f4ce996`, on top of `s1/fix/degraded-run-polish` `378e54a`). Review fixes:
  - `generate_report` now writes both lists from the **same masks** as the Summary counts, and the email reads those CSVs. The subagent had copied the mask logic, so the two could drift apart.
  - The request runs **before** the warning-email snapshot. Previously, "recipients not set", allowlist blocks and send failures were logged too late to reach the alert email.
  - Corrected the interim recipient in `.env.example`.

  CSV columns: `leadId`, `First Name`, `Last Name`, `Phone`, `Status`. To verify on the Mac: a dry-run with `DRYRUN_EMAIL_RECIPIENTS` set. Synthetic results give about 35% DNC, so the email should arrive with both attachments.
- **Started 2026-11-05** on branch `s1/feat/soft-deletion-request`, stacked on `s1/fix/degraded-run-polish`. Settings: `SOFT_DELETE_RECIPIENTS`, `SOFT_DELETE_CC`, `SOFT_DELETE_ALLOWED_DOMAINS` (default internal domain only). If recipients aren't set, a WARNING is logged and nothing is sent. Over 2.5 MB of attachments, the email carries counts plus the CSV location instead.
- **Decided 2026-10-29 (stories interview Q6):** the mass soft-deletion is done by the **LawRuler team on email request**. Automating it through the LawRuler API is out of scope until **S6** ([[Epic - Automated soft-deletion]]).
- **Recipient: our LawRuler account manager. Identity unknown; open item (stories interview Q7, 2026-10-29).** Until it's known, the job sends the request to Michelle (CC e.cervera) so she can forward it, and the domain allowlist holds only the internal domain.
- **Precedent:** `Knowledge/Reporting/Known Litigator Soft-delete Report.md`. On 2025-08-22 the litigator list was emailed to **Alfredo**, who soft-deleted 187 leads, following Matt's instructions about which categories stay.
- Replaces "Litigator action list for soft-deletes" (Q3). The Q3 internal-only rule becomes an allowlist: internal domain plus the LawRuler vendor domain. The scope widens to the DNC-without-opt-in list.
- Litigator definition: not a case, DNC, not Single Opt-In, Litigator, and Status not in `ACTIVE_PROCESS_STATUSES` (`run.py:50-71`). DNC without opt-in: not a case, DNC, not Single Opt-In. Decision D-01 (soft vs hard delete) may change the requested action.
- Reference volumes: 2,243 + 4 on 2026-10-28.

## Open questions
- [x] Who is our LawRuler account manager? **Lindsey Humphreys (lindsey.humphreys@lawruler.com)**, Michelle, 2026-11-05. Michelle forwards the review email to her.
- [ ] #open-question D-01: litigators soft-delete or hard-delete? May change the requested action.

## Dependencies
- [[Story - Report labels and unmatched warning fixed]] (same definitions and deadline wording)
- [[Story - Monthly report email as a summary table]] (shares the email-building code)
