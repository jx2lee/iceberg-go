Branch: `fix/empty-string-partition-bound`

Title:

```
fix(manifest): write empty string partition bounds as zero-length bytes
```

Body:

```markdown
## Description

Fixes #2407

`StringLiteral.MarshalBinary` returns a nil slice for `""`.
`partitionFieldStats.toSummary` stored a pointer to that slice, and the Avro
encoder writes a pointer to a nil slice as the null branch of the
`["null","bytes"]` union. A null `lower_bound` with `contains_null = false`
reads as "no non-null values", so the manifest evaluator (and Java/PyIceberg)
skipped the whole manifest for any predicate on that field.

Keep a present bound as a zero-length slice, as Java's `PartitionSummary` does.

## Testing

`TestManifestListKeepsEmptyPartitionBound` writes a manifest list with
partition values `{"", "us"}`, reads it back and rewrites it once more. It fails
on main for the string field (`lower_bound` is null) and passes with this
change. The end-to-end scan from the issue returns 1 row for each filter.
```

Checklist: `make lint`, `go test ./ ./table/...`, `git commit -s`.
