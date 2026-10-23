Branch: `fix/update-schema-rename-path-alias`

Title:

```
fix(table): do not overwrite the caller's path in UpdateSchema rename
```

Body:

```markdown
## Description

Fixes #<issue>

`updateColumn` built the renamed field's full name with
`append(path[:len(path)-1], newName)`. The prefix shares its backing array with
`path`, so the append overwrote the caller's last path element. Because
`UpdateSchema` re-runs its ops on every `Apply`/`BuildUpdates`, a later op on
the same slice, or a second `Apply`, looked up the new name in the old schema
and failed with `field not found`.

Clone the prefix before appending.

## Testing

- `TestRenameColumnDoesNotMutatePath`: rename then update doc through the same
  slice, and the slice is unchanged afterwards.
- `TestRenameColumnApplyThenBuildUpdates`: `Apply()` then `BuildUpdates()` on a
  nested rename.
- Both fail on main with `field not found: address.town` and pass with the fix.
```

Checklist: `make lint`, `go test ./table/...`, `git commit -s`.
