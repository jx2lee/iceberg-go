Branch: `fix/update-spec-rename-onto-void`

Title:

```
fix(table): move the void field aside when renaming onto its name
```

Body:

```markdown
## Description

Fixes #<issue>

When the target name of `RenameField` belonged to a void (removed) partition
field, `renameField` renamed the *source* field to `<name>_<voidFieldId>` and
returned, so the requested rename was lost without an error.

Follow Java's `BaseUpdatePartitionSpec.renameField`: rename the void field to
`<newName>_<voidFieldId>` first, then apply the requested rename.

## Testing

`TestRenameFieldOntoVoidField` fails on main and passes with this change;
`go test ./table/...` passes.
```

Checklist: `make lint`, `go test ./table/...`, `git commit -s`.
