# Snapshot summary: total-* counters are invented when the parent summary does not have them

- Issue: 없음 (2026-10-24 검색: "snapshot summary total", "updateSnapshotSummaries", "total-records summary", "summary totals missing" → #1582(int64 totals, merged) 는 무관. #2215 는 DV 카운트 문제로 다름)
- Verified against main `7768540`

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

`updateSnapshotSummaries` (`table/snapshots.go:738-746`) starts each total from `previous.GetInt64(totalProp, 0)`. A total that is **missing** from the parent summary, or does not parse, is treated as 0, and the new snapshot gets a total that only counts this commit.

Java's `SnapshotProducer.updateTotal` only carries a total forward when the previous summary has it:

```java
String totalStr = previousSummary.get(totalProperty);
if (totalStr != null) {
  try { ... summaryBuilder.put(totalProperty, String.valueOf(newTotal)); }
  catch (NumberFormatException e) { /* ignore and do not add total */ }
}
```

PyIceberg's `_update_totals` does the same (`if previous_total_str := previous_summary.get(...)`). Both start totals at `"0"` only when there is **no parent snapshot**; Go already does that part (`previous == nil`).

Two cases produce wrong totals:

1. **Parent never tracked totals** (written by a writer that does not maintain them): after one append the table reports `total-records=5`, `total-data-files=1` even though it may hold millions of rows.
2. **Total was dropped**: Go drops a total once it would go negative (`if newTotal >= 0`). The next snapshot then restarts it from 0, so the counter comes back with a wrong value.

Engines and tools read `total-records` / `total-data-files` for stats and planning, so this silently misreports table size.

### Repro

```text
=== RUN   TestSummaryTotalsMissingInParent
    total-records=5 fabricated; parent had no such total
    total-data-files=1 fabricated; parent had no such total
    total-files-size=100 fabricated; parent had no such total
    total-delete-files=0 fabricated; parent had no such total
--- FAIL
=== RUN   TestSummaryTotalNotResurrectedAfterInvalidation
    total-data-files=1 resurrected after being invalidated
--- FAIL
```
