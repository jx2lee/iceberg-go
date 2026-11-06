# literals: string cast to timestamp_ns / timestamptz_ns silently wraps outside the nanosecond range

- Issue: 없음 (2026-10-24 검색, 2026-11-06 재확인: PR "timestamp nanosecond overflow" → #1533(merged, `TimestampLiteral` 경로만). issue "timestamp nanosecond overflow", "string timestamp_ns" 결과 없음)
- Verified against main `2e69b8a` (2026-11-06)

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

`StringLiteral.To(TimestampNsType / TimestampTzNsType)` (`literals.go:1019-1036`) converts with `tm.UTC().UnixNano()`. Go documents `UnixNano` as undefined outside roughly 1677-09-21 … 2262-04-11, and in practice it wraps around. The cast returns a wrong value instead of an error.

#1533 added a range check to `TimestampLiteral.To(TimestampNs)` (`literals.go:842-844`), but the string path was missed.

Java's `StringLiteral.to(TIMESTAMP_NANO)` uses `DateTimeUtil.isoTimestampToNanos` → `ChronoUnit.NANOS.between`, which uses checked arithmetic and throws `ArithmeticException`.

Filters coming from JSON (`ParseExpr`, e.g. REST scan planning) pass values as strings, so they hit this path:

```text
lit("1600-01-01T00:00:00").To(timestamp_ns)          = 2184-07-20 23:34:33.709551616
lit("1600-01-01T00:00:00+00:00").To(timestamptz_ns)  = 2184-07-20 23:34:33.709551616
lit("2500-01-01T00:00:00").To(timestamp_ns)          → wraps to 1915-06-14
```

So `ts < '1600-01-01T00:00:00'` becomes `ts < 2184-07-20…` and matches every row from 1970.

### Repro

```text
=== RUN   TestStringToTimestampNsOutOfRange
    timestamp_ns: expected out-of-range error, got 2184-07-20 23:34:33.709551616
    timestamptz_ns: expected out-of-range error, got 2184-07-20 23:34:33.709551616
    parsed filter: LessThan(term=Reference(name='ts'), literal=2184-07-20 23:34:33.709551616)
    1970-01-01 < 1600-01-01 evaluated to true
--- FAIL
```
