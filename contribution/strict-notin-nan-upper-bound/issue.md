# Strict metrics evaluator: NotIn returns "rows must match" when the upper bound is NaN

- Issue: 없음 (2026-10-24 검색: PR "strict NaN", "NaN evaluator", issue "NaN bound", "NaN" → #1103/#1106(strict NotEq/NotIn 의 null·NaN count), #1091/#1098, #2131 은 다른 문제)
- Verified against main `7768540`

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

`strictMetricsEval.VisitNotIn` (`table/evaluators.go:1772`) guards the **lower** bound against NaN but not the **upper** bound.

```go
if m.isNan(lowerBound) { return rowsMightNotMatch }   // present
...
values = removeBoundCheck(upperBound, values, -1)       // no NaN check before this
if len(values) == 0 { return rowsMustMatch }
```

Go's float comparison (`cmp.Compare`) orders NaN **below** every number, so `removeBoundCheck(NaN, values, -1)` removes every literal and the evaluator answers `rowsMustMatch`.

- `VisitNotEqual` in the same file checks both bounds for NaN ("NaN indicates unreliable bounds"). So a one-value `NotIn` (bound as `NotEqual`) is correct and a two-value `NotIn` is not.
- Java's `StrictMetricsEvaluator` treats NaN bounds as unreliable as well. Java's `Double.compare` also orders NaN above everything, so in Java a NaN upper bound removes nothing.

Spec-compliant writers do not put NaN in bounds, but older or foreign writers did, which is why the NaN guards exist. Strict evaluation is used by `Delete` to decide whether a file can be dropped whole (`table/transaction.go:2845`). On a file with rows `{7.0, NaN}`, `lower=7.0` and `upper=NaN`, the evaluator says every row satisfies `x NOT IN (7.0, 8.0)`, which is wrong for the `7.0` row.

### Repro

```text
=== RUN   TestStrictNotInNaNUpperBound
    NotIn(term=Reference(name='x'), {[7 8]}): strict evaluator claimed every row matches
--- FAIL
```

(`NotIn(x, 7.0)`, which becomes `NotEqual`, is already correct.)
