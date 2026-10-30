---
type: epic
project: dnc-cron-scrub
sprint: S5
status: to-do
gaps: [GAP-25]
---
# Epic: Archiving closed months

> [!important] Sprint-start ritual
> At the **start of this sprint**, re-run **Prioritize & De-risk** (scrum-scoping Phase 4): MoSCoW the stories (their `moscow` property is `tbd` until then), re-run the pre-mortem and update [[Prioritization and Risk Register]].

## Description
Move closed monthly partitions (PostgreSQL) and old logs to external disk and/or cloud storage. Closes **GAP-25**: ~4.1 GB of lead PII in `data/legacy/` currently has no retention policy.

Starts after the S4 dashboard launches (archived months are no longer displayed; live months remain accessible). **Closed months are read-only** (established in S2), so archiving them does not break the pipeline.

Legal traceability requirements: archived data must have **checksums** (SHA-256) recorded before archiving, and a **restore drill** must prove the data comes back intact. Storage target: external disk and/or cloud (undecided; Story 1 defines the policy, Story 2 proves the mechanism, Story 3 automates it).

## Breakdown

### INVEST check
| Criterion | Rating | Notes |
|---|---|---|
| **Independent** | ✅ | Depends on S2 (closed months read-only) and S4 (dashboard live); then self-contained |
| **Negotiable** | ✅ | Retention durations and storage target (disk/cloud) decided in Story 1 |
| **Valuable** | ✅ | Compliance gets a defensible retention policy; IT reduces live storage footprint |
| **Estimable** | ✅ | Story 2 (one month) bounds the technical risk; Story 3 is automation of a proven process |
| **Testable** | ✅ | Restore drill in Story 2 proves correctness; Story 3 verified by replaying the timer |
| (Small skipped) | | Epic-level check |

### Splitting pattern applied
**Pattern 1: Workflow Steps** (acceptable here because each step delivers value to a different persona).

- **Story 1 (retention policy):** compliance/legal decide how long lead PII is kept live vs archived vs deleted. This is a **decision-making story**, not a build story. Value: the organization has a defensible policy.
- **Story 2 (archive + restore one month):** IT proves the archive mechanism works — one closed month is checksummed, archived to external disk and/or cloud, and **restored in a test environment** with checksums verified. Value: IT and compliance see that archived data is recoverable.
- **Story 3 (scheduled archiving):** systemd timer (versioned in `Dev/deploy/`, reproducible) runs the archive process monthly. Value: archiving happens automatically without manual intervention.

**Why earlier patterns didn't fit:**
- Pattern 2 (Operations): archiving is *one* operation (move + verify); CRUD splits don't apply
- Pattern 4 (Data Variations): we're not splitting by month (Story 2 tests one; Story 3 does all future ones)
- Pattern 6 (Major Effort): considered, but Story 1 is a *decision*, not effort
- Pattern 9 (Spike): Story 1 delivers a *policy* (binding decision), not just learning

**Why this Workflow Steps split is acceptable:** each story delivers value to a distinct persona — compliance owns the policy, IT proves the mechanism, operations benefit from automation. This is not a "design then build then test" anti-pattern.

### Cynefin calibration
**Complicated** (known unknowns, sense-analyze-respond). Retention policies have legal implications (Story 1 may surface unknowns about compliance requirements), but the technical work (Story 2–3) is solved: `pg_dump` + `sha256sum` + `restic` or `rclone` + systemd timer. Story 2's restore drill is the forcing function that proves we didn't miss a step.

### Split evaluation
Three stories is appropriate. If Story 1 reveals **multiple retention tiers** (e.g., "keep 12 months live, 24 months archived on disk, 7 years archived in cold cloud, then delete"), split Story 2 by tier: "archive to disk" + "archive to cloud" + "purge after 7 years." If the cloud provider decision is contentious, extract a spike ("evaluate S3 vs Azure Blob vs Backblaze B2 for archived lead data").

> [!check] Before closing this epic, verify Michelle's actions were actually done
> - [ ] Schedule the retention conversation with compliance/Legal. — [[Story - Retention policy for lead data]]

## Child user stories
- [[Story - Retention policy for lead data]]
- [[Story - Archive and restore one closed month]]
- [[Story - Scheduled archiving of closed months]]

## Dependencies / sequencing
- **Starts after:** S4 dashboard launched (archived months no longer displayed live)
- **Builds on:** S2's closed-months-are-read-only design (partitions can be dropped after archiving)
- **GAP closed:** GAP-25 (4.1 GB lead PII with no retention policy)
