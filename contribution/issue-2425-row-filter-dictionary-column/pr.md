Branch: `fix/row-filter-dictionary-column`

Title:

```
fix(table): evaluate row filters on dictionary columns
```

Body:

```markdown
## Description

Fixes #2425

`filterRecords` evaluated the substrait predicate directly on the file batch,
before projection. When the file's embedded Arrow schema stores a column as a
dictionary (PyIceberg does this for a pandas `Categorical` column), arrow-go's
expression executor rejected the field reference, because the predicate
expects the value type (for example `utf8`) and only `large_utf8` and
`large_binary` are accepted as substitutes.

Cast top-level dictionary columns to their value type for evaluating the
predicate only, then apply the mask to the original batch. The output keeps
the input schema, so projection decodes the column as before. Batches without
dictionary columns take the same path as before, without extra allocation.
Both callers, the scan record filter and `prepareBatchFilter`, go through
`filterRecords`.

Dictionary columns nested inside structs, lists or maps are not decoded.

## Testing

`TestPrepareBatchFilterOnDictionaryColumn` fails on main with
`referenced field ... was dictionary<values=utf8, indices=int8, ordered=false>,
but should have been utf8` and passes with this change. I also checked a scan
over a Parquet file written with the stored Arrow schema and a dictionary
column: it errored on main and returned the two matching rows with this change.
`go test ./table/...` passes.
```

Checklist: `make lint`, `go test ./table/...`, `git commit -s`.
