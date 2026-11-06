Branch: `fix/glue-schema-columns`

Title:

```
fix(catalog/glue): add glue.schema-columns to skip StorageDescriptor columns
```

Body:

```markdown
## Description

Fixes #701

Every Glue create/update writes all Iceberg columns, current and historical,
into `StorageDescriptor.Columns`. With a few thousand top-level fields the
request goes over Glue's payload limit and every commit fails with
`InvalidInputException: Payload size of request exceeded limit`.

Add the catalog property `glue.schema-columns` (default `true`). When it is
`false`, `constructTableInput` sends only the location in the
StorageDescriptor. The default keeps the current behaviour, which matches Java
and PyIceberg. Iceberg readers load the schema from the metadata file, so the
table stays usable; tools that read Glue columns (Athena column listing, Lake
Formation column grants) will not see them.

This continues #769, which was closed while the property name was discussed.

## Testing

`TestGlueTableInputPayloadLargeSchema` serializes `UpdateTable` with the real
AWS SDK client and a recording HTTP client (no network). With 3600 columns the
request is 546,378 bytes by default and well under 400 KiB with
`glue.schema-columns=false`. `go test ./catalog/glue/...` and `go vet` pass.
```

Checklist: maintainer agreed on the property in #701 first, `make lint`, `go test ./catalog/glue/...`, `git commit -s`.
