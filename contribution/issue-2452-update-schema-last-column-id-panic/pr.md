> 참고: 리포터가 이미 PR #2453 을 같은 방식으로 열어 둠(2026-11-06). 아래는 기록용이며 올리지 않는다.

Branch: `fix/update-schema-last-column-id-panic`

Title:

```
fix(table): return an error from UpdateSchema when last-column-id is too low
```

Body:

```markdown
## Description

Fixes #2452

`NewUpdateSchema` seeds its id counter from `last-column-id`. When the metadata
has a `last-column-id` below a field id already in use, a new column gets a
duplicate id and `Apply()` / `Commit()` panic in `iceberg.NewSchema`
(`invalid schema: multiple fields for id 3: z and w`).

Run the existing `MetadataBuilder.validateLastColumnID()` in `NewUpdateSchema`
and store the error in `u.err`, so `Apply()`, `BuildUpdates()` and `Commit()`
return `ErrInvalidMetadata` instead of panicking.

## Testing

`TestUpdateSchemaLastColumnIDBelowHighestFieldID` panics on main and passes with
this change. `go test ./table/...` and `go vet ./table/...` pass.
```

Checklist: `make lint`, `go test ./table/...`, `git commit -s`.
