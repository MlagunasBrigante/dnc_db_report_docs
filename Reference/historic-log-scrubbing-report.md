---
type: reference
project: dnc-cron-scrub
created: 2026-11-09
---
# Historic Log Scrubbing Report

## Summary
- **Logs processed**: 19
- **Checks passed**: 19
- **Checks failed**: 0
- **Original size**: 2,590,472,853 bytes (2.41 GB)
- **Trimmed size**: 4,443,509 bytes (0.00 GB)
- **Bytes to free**: 2,586,029,344 bytes (2.41 GB)

## Per-Run Results

| Original Path | Output File | Original (bytes) | Trimmed (bytes) | Lines | Secret Matches | Check |
|---------------|-------------|------------------|-----------------|-------|----------------|-------|
| 2026-08-12/logs/run.log | run.info.log | 1,288,736,914 | 827,798 | 8616/1090361 | 0 | ✓ |
| 2026-09-01/logs/run.log | run.info.log | 122,435,283 | 90,773 | 998/16786 | 0 | ✓ |
| 2026-10-01/logs/run.log | run.info.log | 513,233,984 | 151,418 | 1585/89824 | 0 | ✓ |
| 2026-10-28/logs/run.log | run.info.log | 472,825,874 | 101,028 | 1119/25591 | 0 | ✓ |
| 2026-11-01/logs/run.log | run.info.log | 127,369 | 37,348 | 410/1017 | 0 | ✓ |
| 2026-11-02/logs/run.log | run.info.log | 139,635 | 139,635 | 1535/1535 | 0 | ✓ |
| legacy/2026-01-16/logs/logs.log | logs.info.log | 1,251,114 | 31,195 | 371/10500 | 0 | ✓ |
| legacy/2026-01-16/logs/logs_2026-01-17.log | logs_2026-01-17.info.log | 11,434,205 | 84,187 | 943/100351 | 0 | ✓ |
| legacy/2026-02-11/logs/logs.log | logs.info.log | 1,310,936 | 7,980 | 88/11157 | 0 | ✓ |
| legacy/2026-02-12/logs/logs.log | logs.info.log | 0 | 0 | 0/0 | 0 | ✓ |
| legacy/2026-02-20/logs/logs.log | logs.info.log | 116 | 0 | 0/1 | 0 | ✓ |
| legacy/2026-02-21/logs/logs.log | logs.info.log | 0 | 0 | 0/0 | 0 | ✓ |
| legacy/2026-03-13/logs/logs.log | logs.info.log | 3,562,532 | 10,730 | 123/32864 | 0 | ✓ |
| legacy/2026-04-17/logs/logs.log | logs.info.log | 164,267,982 | 2,756,057 | 13446/1439573 | 0 | ✓ |
| legacy/2026-05-08/logs/logs.log | logs.info.log | 353 | 69 | 1/3 | 0 | ✓ |
| legacy/2026-05-12/logs/logs.log | logs.info.log | 1,098,836 | 276 | 4/10308 | 0 | ✓ |
| legacy/2026-06-10/logs/logs.log | logs.info.log | 8,581,539 | 48,169 | 459/76327 | 0 | ✓ |
| legacy/2026-06-30/logs/run.log | run.info.log | 1,204,028 | 78,650 | 868/11422 | 0 | ✓ |
| legacy/2026-07-01/logs/run.log | run.info.log | 262,153 | 78,196 | 861/2114 | 0 | ✓ |

## Totals
- Original: 2,590,472,853 bytes
- Trimmed: 4,443,509 bytes
- Freed: 2,586,029,344 bytes

## Secret Pattern Verification (Log Files Only)

_Every `*.log` under `data/` (dry-runs included), streamed line by line. Counts only; no values are recorded._

| Pattern | In copies (`*.info.log`) | In originals (to be deleted) |
|---------|--------------------------|------------------------------|
| client_secret= | 0 | 7 |
| Bearer tokens | 0 | 7 |
| access_token | 0 | 21 |
| password= | 0 | 0 |
| ASP.NET_SessionId | 0 | 0 |
| base64 blobs >1KB | 0 | 42 |
| Access Token: (legacy) | 0 | 22 |
| Bearer token: (legacy) | 0 | 0 |
| Token: (legacy) | 0 | 7 |

- **Total in copies: 0** (must be 0)
- Unreadable files: 0

## Originals to delete (after approval)

_Only originals whose copy passed every check are listed. Delete mode reads this list, re-checks each copy right before deleting, and skips anything that no longer passes._

- `data/2026-08-12/logs/run.log`
- `data/2026-09-01/logs/run.log`
- `data/2026-10-01/logs/run.log`
- `data/2026-10-28/logs/run.log`
- `data/2026-11-01/logs/run.log`
- `data/2026-11-02/logs/run.log`
- `data/legacy/2026-01-16/logs/logs.log`
- `data/legacy/2026-01-16/logs/logs_2026-01-17.log`
- `data/legacy/2026-02-11/logs/logs.log`
- `data/legacy/2026-02-12/logs/logs.log`
- `data/legacy/2026-02-20/logs/logs.log`
- `data/legacy/2026-02-21/logs/logs.log`
- `data/legacy/2026-03-13/logs/logs.log`
- `data/legacy/2026-04-17/logs/logs.log`
- `data/legacy/2026-05-08/logs/logs.log`
- `data/legacy/2026-05-12/logs/logs.log`
- `data/legacy/2026-06-10/logs/logs.log`
- `data/legacy/2026-06-30/logs/run.log`
- `data/legacy/2026-07-01/logs/run.log`

```bash
# From the dnc-cron-scrub/ project folder, after approving this report:
python3 Dev/tools/scrub_historic_logs.py --data-dir data --delete-originals \
  --approved-report Docs/Reference/historic-log-scrubbing-report.md
```

> [!todo] Michelle — action needed
> - [ ] Review this report and spot-check a few `*.info.log` copies.
> - [ ] Approve, then run the delete command above on the Mac.
> - [ ] After deletion: confirm the originals are gone and the copies remain.
