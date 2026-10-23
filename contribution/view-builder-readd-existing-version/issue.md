# view: MetadataBuilder appends a duplicate when re-adding a version that already exists in the base

- Issue: 없음 (2026-10-24 검색: issue "view duplicate version", "duplicate version-id", PR "view version reuse", "addVersion view builder" → #2108/#2109(version-log), #1525(parse 시 중복 거부) 는 다른 문제)
- Verified against main `7768540`
- 영향 범위: repo 안에서 base metadata 에서 시작해 view update 를 적용하는 곳은 없음(non-REST catalog 는 UpdateView 미구현). 공개 API `view.MetadataBuilderFromBase` + `SetCurrentVersion` / `AddViewVersionUpdate.Apply` 를 쓰는 사용자(예: Go 로 만든 REST catalog 서버)가 영향을 받음

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

`view.MetadataBuilder.addVersion` (`view/metadata_builder.go:147-163`) finds an existing identical version through `reuseOrCreateNewVersionID`, but returns early only if that version was added **in this builder**:

```go
if _, ok := b.versionsById[newVersionID]; ok {
	for _, upd := range b.updates {
		if vvu, ok := upd.(*addViewVersionUpdate); ok && vvu.Version.VersionID == newVersionID {
			b.lastAddedVersionID = &newVersionID
			return newVersionID, nil
		}
	}
	// falls through when the version came from the base metadata
}
```

When the version came from the base metadata, it falls through, appends a second copy to `versionList` and emits an `AddViewVersion` update. `Build()` then fails with `invalid view metadata: duplicate version-id 1`.

Java's `ViewMetadata.Builder.addVersionInternal` always returns early when `versionsById.containsKey(newVersionId)`. It sets `lastAddedVersionId` to the ID only if the version was added in this builder, and to `null` otherwise. As a result, in Java:

- replacing a view with an unchanged definition is a no-op, and
- reverting to an earlier definition reuses that version's ID and makes it current.

Both fail in Go.

### Repro

```text
=== RUN   TestSetCurrentVersionReusesBaseVersion
    err=invalid view metadata: duplicate version-id 1
--- FAIL
=== RUN   TestSetCurrentVersionRevertsToEarlierVersion
    err=invalid view metadata: duplicate version-id 1
--- FAIL
```
