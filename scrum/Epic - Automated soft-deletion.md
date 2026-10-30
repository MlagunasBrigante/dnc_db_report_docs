---
type: epic
project: dnc-cron-scrub
sprint: S6
status: to-do
gaps: []
---
# Epic: Automated soft-deletion

> [!important] Sprint-start ritual
> At the **start of this sprint**, re-run **Prioritize & De-risk** (scrum-scoping Phase 4): MoSCoW the stories (their `moscow` property is `tbd` until then), re-run the pre-mortem and update [[Prioritization and Risk Register]].

## Description
Today, and through S5, leads to remove (DNC without opt-in, plus litigators) are soft-deleted **by the LawRuler team on email request** ([[Story - Soft-deletion request to the LawRuler team]]). This epic would let the job perform those soft-deletes itself through the LawRuler API, keeping the "by the 7th of <month>" commitment without manual work. **It starts only after S5**, and only if Michelle decides to automate (D-07).

## Open questions
- [ ] #open-question **D-07:** automate the soft-deletion, or keep the email request? Decided after the spike.
- [ ] #open-question **D-01:** litigators soft-delete or hard-delete?

## Breakdown

### INVEST check
| Criterion | Rating | Notes |
|---|---|---|
| Independent | ⚠️ | Needs S2 (Postgres lineage, failure model) and a LawRuler API capability that hasn't been confirmed |
| Negotiable | ✅ | Could stay manual indefinitely; scope depends on D-01 (soft vs hard delete for litigators) |
| Valuable | ✅ | Removes a monthly manual hand-off and makes the 7th deadline enforceable by the job |
| Estimable | ⚠️ | Unknown until the spike confirms whether the LawRuler API supports soft-delete |
| Testable | ✅ | A dry-run list vs actual deletions, reconciled against the next snapshot |

### Splitting pattern applied
**Pattern 9: Spike.** Whether LawRuler exposes a soft-delete operation is unknown, so a time-boxed spike comes first and the stories are split only after it. Patterns 1–8 don't apply until that uncertainty is resolved.

### Cynefin calibration
**Complex / high uncertainty** — depends on the vendor API and on legal sign-off for automated deletion. Learning first; no story enumeration yet.

### Split evaluation
- **Reveals low-value work?** Yes. If the spike finds no API support, the epic closes with "stay manual".
- **Equal sizes?** N/A until after the spike.

## Child user stories
- [[Story - Spike LawRuler soft-delete via API]]

## Dependencies / sequencing
- After S5. Requires decision D-07 (automate or not) and, ideally, D-01.
