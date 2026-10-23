Branch: `fix/v1-legacy-partition-field-ids`

Title:

```
fix(table): number legacy v1 partition field IDs per spec
```

Body:

```markdown
## Description

Fixes #<issue>

v1 partition specs that predate partition field IDs were numbered per spec,
from 1000 by position, by the reference implementation, and the table's
manifests carry those IDs. #1514 assigns missing IDs table-wide, so the second
spec of an evolved legacy table got `[1001, 1002]` instead of `[1000, 1001]`,
and partition tuples were read under the wrong fields.

For format v1, when a spec has no explicit field IDs at all, assign
`1000 + position` within that spec, as Java's `PartitionSpec.Builder` does.
Mixed specs and v2/v3 keep the table-wide allocation from #1514.

## Testing

- `TestLegacyV1PartitionFieldIDsPerSpec`: parses a two-spec legacy v1 table
  and reads a spec-1 manifest written with IDs 1000/1001. Fails on main,
  passes with this change.
- The #1514 tests still pass; `go test ./table/...` passes.
```

Checklist: `make lint`, `go test . ./table/...`, `git commit -s`.
