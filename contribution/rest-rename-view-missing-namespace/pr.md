Branch: `fix/rest-rename-view-namespace-404`

Title:

```
fix(catalog/rest): report a missing namespace on RenameView 404
```

Body:

```markdown
## Description

Fixes #<issue>

#2173 split the rename 404 for tables by `error.type`, so a missing
destination namespace returns `ErrNoSuchNamespace` instead of
`ErrNoSuchTable`. `RenameView` has the same 404 semantics in the REST spec but
still mapped every 404 to `ErrNoSuchView`.

Apply the same `withErrorTypeOverride` to `RenameView`. An untyped 404 still
falls back to `ErrNoSuchView`.

## Testing

`TestRenameView404MissingNamespace` fails on main and passes with this change.
The existing `TestRenameView404` (typed `NoSuchViewException`) still passes.
```

Checklist: `make lint`, `go test ./catalog/rest/...`, `git commit -s`.
