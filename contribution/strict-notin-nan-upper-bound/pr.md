Branch: `fix/strict-notin-nan-upper-bound`

Title:

```
fix(table): treat a NaN upper bound as unreliable in strict NotIn
```

Body:

```markdown
## Description

Fixes #<issue>

`strictMetricsEval.VisitNotIn` checked the lower bound for NaN but not the
upper bound. NaN compares below every value in Go, so a NaN upper bound removed
every literal and the evaluator returned rows-must-match. `VisitNotEqual`
already guards both bounds, so `NotIn` with one value (bound as `NotEqual`) and
with several values disagreed.

Return rows-might-not-match for a NaN upper bound, as for the lower bound.

## Testing

`TestStrictNotInNaNUpperBound` fails on main for the two-value `NotIn` and
passes with this change.
```

Checklist: `make lint`, `go test ./table/...`, `git commit -s`.
