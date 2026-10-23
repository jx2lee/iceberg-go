# UpdateSchema: case-insensitive rename of an identifier column fails with "identifier field not found"

- Issue: 없음 (2026-10-24 검색: "rename column identifier case insensitive", "identifier field case-insensitive", "rename identifier field" → #1689(SetIdentifierField 순서, closed), #1359, #2076 은 무관)
- Verified against main `7768540`

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

With `NewUpdateSchema(txn, caseSensitive=false, ...)`, renaming an identifier column using a different spelling fails in `Apply()`:

```go
// schema: id int (identifier), data string
NewUpdateSchema(txn, false, false).RenameColumn([]string{"ID"}, "key").Apply()
// error: identifier field not found: id
```

`identifierFieldNames` is keyed by the schema's own spelling (`initIdentifierFieldNames` uses `FindColumnName`, `table/update_schema.go:209`). `updateColumn` looks it up with the caller's spelling (`fullName`, `table/update_schema.go:469`):

```go
if _, ok := u.identifierFieldNames[fullName]; ok { // "ID" vs key "id"
	delete(u.identifierFieldNames, fullName)
	u.identifierFieldNames[newFullName] = struct{}{}
}
```

The lookup misses, so the identifier stays under the old name `id`. `Apply()` then cannot resolve `id` in the renamed schema.

The field lookup itself (`findField`) honours `caseSensitive=false`, so only the identifier bookkeeping is inconsistent. `deleteColumn` (`delete(u.identifierFieldNames, fullName)`, line 397) has the same mismatch.

### Repro

```text
=== RUN   TestCaseInsensitiveRenameIdentifierColumn
    Error: Received unexpected error: identifier field not found: id
--- FAIL
```
