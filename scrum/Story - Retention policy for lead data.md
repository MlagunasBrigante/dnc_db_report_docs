---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Archiving closed months]]"
sprint: S5
moscow: tbd
status: to-do
jira: ""
size: S
---
# Story: Retention policy for lead data

## User Story
**As** compliance and legal, **I want** a documented retention policy defining how long lead PII is kept live, archived, and when it is permanently deleted, **so that** the organization has a defensible position if audited and we're not keeping data longer than legally required or shorter than legally safe.

## Acceptance Criteria
- [ ] **Given** the project holds ~4.1 GB of lead PII in PostgreSQL (post-S2) with no prior retention policy (GAP-25), **when** compliance and legal are consulted, **then** they define retention tiers: e.g., "12 months live in PostgreSQL," "24 months archived on external disk or cloud," "7 years total retention, then hard delete," or "indefinite retention for closed-case leads, 3 years for non-case leads." The specific durations are decided by compliance, not engineering.
- [ ] **Given** retention tiers are defined, **when** documented (suggested: `Docs/S5-Lead-Data-Retention-Policy.md`), **then** the document states: (1) retention duration for each tier, (2) what "archived" means (offsite, checksummed, restorable but not live-queryable), (3) what "deleted" means (partitions dropped, backups purged, irreversible), (4) who authorized the policy (names/roles), and (5) the date the policy takes effect.
- [ ] **Given** the policy references "closed months" (read-only partitions, established in S2), **when** a month is considered "closed," **then** the criteria are explicit: e.g., "month is >90 days old and no tags were updated in that partition in the last 30 days," or simply "any month older than the current and prior month." If the dashboard (S4) affects what "closed" means, that dependency is stated.
- [ ] **Given** the policy is documented, **when** reviewed with Michelle, **then** she confirms it aligns with the business need (e.g., "we need 12 months live for statistical modeling" or "compliance requires 7 years total") and any conflicts are resolved before the policy is finalized.

## Notes
- This is a **decision-making story**, not a technical build story. Engineering facilitates the conversation and documents the outcome, but compliance/legal own the decision.
- If compliance/legal are unavailable or the decision stalls, escalate to Michelle and agree on an interim policy (e.g., "archive after 12 months, defer deletion decision") so S5 can proceed. Mark the interim policy as "pending legal review."
- The policy does not have to match other systems' retention (e.g., LawRuler's retention, email retention) — this is specific to the DNC scrub pipeline's data. But if conflicts arise (e.g., "we're deleting lead data while LawRuler keeps it indefinitely"), flag them.
- **Decision D-01** (soft vs hard delete of known litigators) may inform the policy — if litigators are soft-deleted, are they archived separately or treated as normal leads? Document the interaction.
- Output: a retention policy document, reviewed and approved by compliance/legal, versioned in the repo. This document is referenced by the archiving automation (Story 3).

## Dependencies
- **Depends on:** S2 migration complete (closed months are read-only, so the concept of "archive a closed month" is meaningful)
- **Blocks:** [[Story - Archive and restore one closed month]] (can't archive without knowing the retention tiers) and [[Story - Scheduled archiving of closed months]] (automation requires the policy rules)
