# UpdateSpec: RenameField onto a void field's name renames the wrong field

- Issue: 없음 (2026-10-24 검색: "rename partition field void", "RenameField", "UpdateSpec rename" → #467(원래 UpdateSpec PR), #1834, #1651, #1641 은 무관)
- Verified against main `7768540`

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

In v1 tables a removed partition field stays in the spec as a `void` transform and keeps its name. Renaming another field to that name should move the void field out of the way and then apply the rename. That is what Java does:

```java
// BaseUpdatePartitionSpec.renameField
PartitionField existingField = nameToField.get(newName);
if (existingField != null && isVoidTransform(existingField)) {
  // rename the old deleted field that is being replaced by the new field
  renameField(existingField.name(), existingField.name() + "_" + existingField.fieldId());
}
... renames.put(name, newName);
```

Go (`table/update_spec.go:435-442`) instead renames the **source** field to `<name>_<voidFieldId>` and returns, dropping the requested rename:

```go
if exists && isVoidTransform {
	return us.renameField(name, fmt.Sprintf("%s_%d", name, existingField.FieldID))()
}
```

No error is returned, so the resulting spec is silently wrong. `addField` handles the same "name taken by a void field" case correctly; only `renameField` differs.

### Repro

Spec `[1000: id_identity identity(1), 1001: street_void void(5)]`, `RenameField("id_identity", "street_void")`:

```text
new spec: [ 1000: id_identity_1001: identity(1)   1001: street_void: void(5) ]
Error: Not equal: expected: "street_void"  actual: "id_identity_1001"
--- FAIL: TestRenameFieldOntoVoidField
```

Expected (Java): `[1000: street_void identity(1), 1001: street_void_1001 void(5)]`.
