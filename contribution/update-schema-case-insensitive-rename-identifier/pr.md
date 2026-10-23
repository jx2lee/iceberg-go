Branch: `fix/update-schema-case-insensitive-identifier-rename`

Title:

```
fix(table): keep identifier fields on a case-insensitive rename
```

Body:

```markdown
## Description

Fixes #<issue>

`identifierFieldNames` is keyed by the schema's own column names, but
`updateColumn` looked it up with the caller's path. With `caseSensitive=false`
and a differently cased path, the lookup missed, the identifier kept the old
name, and `Apply` failed with `identifier field not found`.

Look the entry up by the field's canonical name (`FindColumnName(field.ID)`).
Apply the same change to `deleteColumn`.

## Testing

`TestCaseInsensitiveRenameIdentifierColumn` fails on main and passes with this
change.
```

Checklist: `make lint`, `go test ./table/...`, `git commit -s`.
