---
type: story
project: dnc-cron-scrub
epic: "[[Epic - Memory optimization]]"
sprint: S1
status: archived
gaps: [GAP-18]
jira: ""
size: M
---

> [!note] Archived 2026-11-05
> Dropped as low value: Stage 1 has the lowest mean memory (see [[Epic - Memory optimization]]). Kept for history only; not part of the backlog.
# Story: Stage 1 streams pages instead of holding them

## User Story
**As** the IT & Data lead, **I want** Stage 1 to stream pages into the DataFrame incrementally instead of accumulating all pages in a list before conversion, **so that** Stage 1 memory usage is flat instead of climbing linearly, reducing peak memory by ~300 MiB.

## Acceptance Criteria
- [ ] **Given** the `_get_custom_report` method, **when** each page is fetched, **then** it is converted to a DataFrame immediately and appended to a running DataFrame (or written to the ODS database incrementally), instead of being accumulated in the `leads_created` list (`run.py:280, 309-324`).
- [ ] **Given** one real run completes with per-stage memory instrumentation, **when** I compare before/after, **then** Stage 1 memory usage is flat (or grows slowly) instead of climbing linearly from 108 → 721 MiB (baseline from run report).
- [ ] **Given** the run completes, **when** I verify the output, **then** the `raw_data` table row count matches the baseline (no data regression), and the full raw snapshot is still stored in the ODS database.
- [ ] Verified by running with instrumentation, comparing Stage 1 peak memory before/after, and confirming the Stage 1 result DataFrame has the same schema and row count.

## Notes
- **Code references** (verified against `Dev/run.py`):
  - Accumulate pages: `run.py:280` — `leads_created = []` (empty list)
  - Append pages: `run.py:309-317` — `list(csv_reader)` converts each page to a list of dicts, then `leads_created.extend(leads)` appends to the list
  - Convert to DataFrame: `run.py:324` — `pd.DataFrame(leads_created)` converts the full list to a DataFrame → both the list and the DataFrame coexist briefly, doubling memory
- **Gap reference**: GAP-18 — "Full lead history held twice in memory (Stage 1 + Stage 4 re-read)" — this story addresses the Stage 1 half (the list→DataFrame conversion); [[Story - Stage 4 stops reloading the full lead history]] addresses the Stage 4 half.
- **Root cause from run report 2026-10-28**: "Stage 1 linear climb 108 → 721 MiB" — memory grows as pages are fetched, tracking the size of `leads_created`. The climb is linear (each page adds ~1.5 MiB) and peaks at 721 MiB before the DataFrame conversion adds more. At 21:15:56 (Stage 1 end), memory.peak = 788 MiB.
- **Streaming approaches**:
  1. **Incremental DataFrame append**: convert each page to a DataFrame immediately and `pd.concat()` with the running DataFrame. Pro: simple. Con: `pd.concat()` on every page is slow (O(n²) if not careful).
  2. **Chunked write to SQLite**: write each page to the ODS `raw_data` table immediately (idempotent INSERT OR IGNORE), then read the full table at the end. Pro: memory stays flat. Con: more I/O; the DataFrame is still needed for `clean()`.
  3. **Generator + chunked DataFrame**: yield pages from `_get_custom_report` as a generator, consume in `extract()` with `pd.concat(…, ignore_index=True)` in chunks of N pages. Pro: balances memory and speed. Con: more complex.
- **Recommendation**: use approach 2 (chunked write to SQLite) because the ODS database already exists for idempotent storage, and the full DataFrame is only needed after all pages are fetched. Stage 1 already writes to ODS at the end (`run.py:254-259`); move that write inside the page loop.
- **Data contract check**: the full `raw_data` table must still exist in the ODS database after Stage 1 (for S2 backfill and for Stage 4 if [[Story - Stage 4 stops reloading the full lead history]] is not done). The DataFrame returned by `extract()` can be empty or a summary; `clean()` can read from ODS if needed, or the DataFrame can be re-read after the write.
- **Scope**: the raw snapshot must still be stored in full (legal traceability). This story only changes *how* Stage 1 builds the snapshot, not what it stores.
- This is M-sized because it changes the Stage 1 data flow (list→chunked write) and may affect `clean()` if it depends on the DataFrame.

## Dependencies
- **Blocked until** the other S1 epics are done and the code is stable (scope: "starts ONLY after the essential gaps above are closed").
- **Depends on** [[Story - Per-stage duration and memory in every run]] — must be complete so before/after memory is measurable.
- **Sequence within the epic**: this story should complete AFTER [[Story - Stage 4 stops reloading the full lead history]], because the Stage 4 fix is lower-risk and confirms the instrumentation works.
