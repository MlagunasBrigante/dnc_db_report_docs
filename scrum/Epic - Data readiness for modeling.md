---
type: epic
project: dnc-cron-scrub
sprint: S3
status: to-do
gaps: []
---
# Epic: Data readiness for modeling

> [!important] Sprint-start ritual
> At the **start of this sprint**, re-run **Prioritize & De-risk** (scrum-scoping Phase 4): MoSCoW the stories (their `moscow` property is `tbd` until then), re-run the pre-mortem and update [[Prioritization and Risk Register]].

> [!tip] Soft, flexible deadlines (risk R16)
> Michelle's biggest risk is getting stuck or overwhelmed in S3–S4. Each milestone/story gets a **soft target date** set at sprint start; deadlines can slide by agreement. **Nudges:** a weekly check-in nudge ("what's the next small step?") and a nudge 3 days before each soft date. If a milestone stalls for 2 weeks, split it into smaller steps rather than push harder.

## Description
Make the existing DNC scrub data accessible and fit for the statistical model Michelle will develop. This is a **"cold" epic for engineering** — Michelle leads the statistical work with simple tools; engineering provides audit visibility, ingests her notes, and validates the metrics that S4's dashboard will show.

Starts only after the S2 migration is **done and stable** (PostgreSQL on ITServer, legacy backfill complete, at least one scheduled run passed).

Data sources: PostgreSQL partitioned stage tables (container-era 2026-08→ and legacy 2025-05→2026-07 runs, backfilled in S2) plus `pipeline_run` metadata. Metrics include opt-in status, age buckets, DNC→Litigator→not-a-case paths, blocked/invalid numbers, new leads in 30 days, litigators to soft-delete, and tag completion. Decision D-01 (soft vs hard delete of known litigators) may affect metrics — reference but do not decide.

## Open questions
- [ ] #open-question **D-02:** how should non-US numbers be handled? Affects M2/M3 preprocessing.
- [ ] #open-question **D-03:** add a reassigned-number check (6-month refresh)? Affects data-quality criteria (M3).
- [ ] #open-question **M3 quality criteria** ("is the data clean enough?"), defined by Michelle during the sprint.

## Breakdown

### INVEST check
| Criterion | Rating | Notes |
|---|---|---|
| **Independent** | ✅ | Depends on S2 being done+stable; no S4 dependency |
| **Negotiable** | ✅ | M3 quality criteria and M4 test scope defined with Michelle during the sprint |
| **Valuable** | ✅ | Each milestone is a go/no-go checkpoint delivering observable evidence to Michelle and management |
| **Estimable** | ⚠️ | HIGH uncertainty — S3 is exploratory; milestones bound risk |
| **Testable** | ✅ | Go/no-go criteria explicit in each milestone story |
| (Small skipped) | | Epic-level check |

### Splitting pattern applied
**Pattern 1: Workflow Steps.**

Each of the five child stories is a **milestone** (M1–M5) and represents a sequential step in the data-readiness workflow. Normally this pattern is suspect — it often produces technical slices with no standalone value — but here each milestone is a **go/no-go checkpoint** that delivers observable evidence to a persona:
- **M1** (audit) → Michelle and engineering see the data lineage and folder structure
- **M2** (notes ingested) → Michelle's preprocessing logic is aligned with the pipeline
- **M3** (data clean enough?) → compliance and Michelle decide whether to proceed
- **M4** (model tested) → Michelle and management see whether the model works on sample data
- **M5** (metrics validated) → the dashboard team (S4) receives validated metrics to display

Each milestone is independently valuable as a learning checkpoint in a **high-uncertainty sprint**.

**Why earlier patterns didn't fit:**
- Pattern 2 (Operations): no Create/Read/Update/Delete operations here; this is audit + preprocessing alignment
- Pattern 3 (Business Rule Variations): no rule variations (opt-in vs not, age buckets, etc.) — those are metric *outputs*, not story splits
- Pattern 4 (Data Variations): no per-data-source splits; the backfilled PostgreSQL data is the single source
- Pattern 9 (Spike): M1 and M2 are investigative, but all five milestones deliver **decisions**, not just learning

### Cynefin calibration
**Complex** (HIGH uncertainty, sense-respond). S3 is explicitly exploratory: Michelle is defining the statistical model as she goes, M3's quality criteria are "to be defined," and we don't yet know if the legacy data is clean enough to model. The milestone structure provides frequent decision points rather than committing to a fixed plan. Stories are kept **few and learning-first**.

### Split evaluation
Five stories is appropriate for a Complex sprint with go/no-go gates. If M3 fails, S3 may halt or loop back to data cleanup (which would spawn new stories). If stories feel too large during execution, split by **data subset** (e.g., M4 tests one metric category first) or **decision scope** (e.g., M3 splits into separate quality gates per stage table).

> [!check] Before closing this epic, verify Michelle's actions were actually done
> - [ ] Feed your notes on preprocessing into the project. — [[Story - M2 Notes ingested and preprocessing aligned]]
> - [ ] Define the M3 quality criteria ("is the data clean enough?"). — [[Story - M3 Data quality gate]]

## Child user stories
- [[Story - M1 Data and pipeline audit]]
- [[Story - M2 Notes ingested and preprocessing aligned]]
- [[Story - M3 Data quality gate]]
- [[Story - M4 Model tested with sample data]]
- [[Story - M5 Metrics validated for the dashboard]]

## Dependencies / sequencing
- **Starts after:** S2 migration done and stable (backfill complete, first scheduled run passed on ITServer)
- **Feeds into:** S4 (dashboard consumes the M5-validated metrics from PostgreSQL)
- **External dependency:** Michelle's notes (fed during the sprint)
