Branch: `fix/rest-update-view-commit-state-unknown`

Title:

```
fix(catalog/rest): return ErrCommitStateUnknown for UpdateView 5xx
```

Body:

```markdown
## Description

Fixes #<issue>

A 5xx from the view commit endpoint means the commit state is unknown, as for
table commits: the REST spec lists `CommitStateUnknown` for `replaceView`
500/502/504, and Java's `ViewCommitErrorHandler` throws
`CommitStateUnknownException` for 500/502/503/504.

`UpdateView` only mapped 404 and 409, so these surfaced as generic server
errors. Use the same status map as `CommitTable`.

## Testing

`TestUpdateViewErrCommitStateUnknown` covers 500/502/503/504; it fails on main
and passes with this change.
```

Checklist: `make lint`, `go test ./catalog/rest/...`, `git commit -s`.
