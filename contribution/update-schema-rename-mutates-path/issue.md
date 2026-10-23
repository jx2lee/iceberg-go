# UpdateSchema: RenameColumn overwrites the caller's path slice, so a reused path or a second Apply fails

- Issue: 없음 (2026-10-24 검색: "RenameColumn path", "UpdateSchema rename mutates" → 관련 issue·PR 없음)
- Verified against main `7768540`

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

`UpdateSchema.RenameColumn` (and `UpdateColumn` with a new name) writes the new name into the caller's `path` slice.

`table/update_schema.go:450`:

```go
newFullName := strings.Join(append(path[:len(path)-1], update.Name.Val), ".")
```

`path[:len(path)-1]` shares its backing array with `path` and has room for one more element, so `append` overwrites `path[len(path)-1]`. After the op runs, `path` is `{"address", "town"}` instead of `{"address", "city"}`.

Ops are stored as closures and re-run on every `Apply()`/`BuildUpdates()`. That leads to two failures:

1. A later op that reuses the same slice resolves the new name against the old schema:
   `RenameColumn(path, "town").UpdateColumn(path, doc)` fails with `field not found: address.town`.
   In Java, every op in an `UpdateSchema` refers to columns by their original name, so this works there.
2. Previewing with `Apply()` and then calling `BuildUpdates()`/`Commit()` re-runs the rename on the already modified path. This fails even when the path is an inline literal:
   `RenameColumn([]string{"address","city"}, "town")`, then `Apply()`, then `BuildUpdates()` → `field not found: address.town`.

Top-level columns are affected too: `path[:0]` still has capacity, so `RenameColumn([]string{"name"}, "name2")` turns the caller's slice into `[name2]`, and `Apply()` followed by `BuildUpdates()` fails with `field not found: name2`.

### Repro

```text
=== RUN   TestRenameColumnDoesNotMutatePath
    path after Apply: [address town]
    Error: Received unexpected error: field not found: address.town
--- FAIL
=== RUN   TestRenameColumnApplyThenBuildUpdates
    Error: Received unexpected error: field not found: address.town
--- FAIL
```

Fix: copy the prefix before appending (`slices.Clone(path[:len(path)-1])`).
