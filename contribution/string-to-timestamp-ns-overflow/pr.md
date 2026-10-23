Branch: `fix/string-timestamp-ns-overflow`

Title:

```
fix(literals): reject out-of-range strings cast to timestamp nanos
```

Body:

```markdown
## Description

Fixes #<issue>

`StringLiteral.To` converted timestamp_ns and timestamptz_ns with
`time.Time.UnixNano`, which wraps outside about 1677-2262. A filter such as
`ts < '1600-01-01T00:00:00'` was bound to a literal in 2184.

#1533 added this check for `TimestampLiteral`; do the same for strings and
return `ErrBadCast` when the instant does not fit in int64 nanoseconds, as
Java's checked `ChronoUnit.NANOS.between` does.

## Testing

`TestStringToTimestampNsOutOfRange` covers both types, both ends of the range
and the boundary values; it fails on main and passes with this change.
```

Checklist: `make lint`, `go test .`, `git commit -s`.
