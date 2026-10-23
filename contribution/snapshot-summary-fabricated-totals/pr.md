Branch: `fix/snapshot-summary-missing-totals`

Title:

```
fix(table): do not invent snapshot totals missing from the parent summary
```

Body:

```markdown
## Description

Fixes #<issue>

`updateSnapshotSummaries` treated a total that was missing from the parent
summary as 0, so the new snapshot reported a total that counted only this
commit. That happens when the parent was written by a writer that does not
track totals, and after Go itself drops a total that would go negative.

Match Java's `SnapshotProducer.updateTotal` and PyIceberg's `_update_totals`:
carry a total forward only when the parent has it and it parses. A table with
no parent snapshot still starts every total at 0.

## Testing

- `TestSummaryTotalsMissingInParent`
- `TestSummaryTotalNotResurrectedAfterInvalidation`

Both fail on main and pass with this change; `go test ./table/...` passes.
```

Checklist: `make lint`, `go test ./table/...`, `git commit -s`.
