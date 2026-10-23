# Metadata parsing: legacy v1 partition specs without field IDs get table-wide IDs, so partition values are read under the wrong fields

- Issue: 없음 (2026-10-24 검색: "partition field id missing v1", "partition field IDs v1", "assignMissingPartitionFieldIDs", "partition field-id 1000 legacy" → #1514(이 로직을 추가한 PR), #1988, #1641, #1662, #1665 는 per-spec 번호를 다루지 않음. open PR 없음)
- Verified against main `7768540`
- **먼저 방향 질문**: #1514 는 "spec 사이에서 생성된 ID 가 충돌하지 않도록" 테이블 전체 기준 할당을 **의도적으로** 선택했습니다. 이 issue 는 그 결정을 v1 레거시 spec 에 한해 뒤집자는 제안이라 메인테이너 동의가 필요합니다.

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

#1514 assigns missing partition `field-id`s table-wide: one counter across all specs, starting after `last-partition-id` and every explicit ID (`table/metadata.go:2264-2330`).

For legacy **v1** metadata whose specs predate partition field IDs, that does not match the IDs the reference implementation used, and which are already written in the table's manifests:

- Spec: v1 partition field IDs "were assigned sequentially starting at 1000 in the reference implementation". That numbering is per spec, by position.
- Java: `PartitionSpecParser.buildFromJsonFields` → `PartitionSpec.Builder.nextFieldId()` starts at 1000 for **every** spec it parses.
- Manifests written for spec N carry partition struct field IDs 1000, 1001, ….
- Go reads `DataFile.Partition()` by the Avro field-id in the manifest, but through `spec.PartitionType()` IDs from the parsed metadata.

So for a v1 table whose spec evolved from `[x]` to `[x, y]`, Go assigns spec 1 the IDs `[1001, 1002]` instead of `[1000, 1001]`. Reading a spec-1 manifest then shifts the partition tuple: `x` gets `y`'s value and `y` is nil. That silently breaks pruning, partition summaries and paths.

### Repro

v1 metadata with `partition-specs` `[{spec-id 0: x identity}, {spec-id 1: x identity, y identity}]`, no field IDs, and a manifest for spec 1 written with IDs 1000 (x=10) and 1001 (y=20):

```text
spec 1 field IDs: [1001 1002] (Java: [1000 1001])
partition tuple read through parsed spec 1: x=20 y=<nil> (want x=10 y=20)
--- FAIL: TestLegacyV1PartitionFieldIDsPerSpec
```

### Proposal

For `format-version: 1` only, when a spec has **no** explicit field IDs at all, number its fields `1000 + position`, as Java does. Specs that mix explicit and missing IDs, and v2/v3 metadata, keep the current #1514 behaviour. The existing #1514 tests (`TestAssignMissingPartitionFieldIDsAcrossSpecs`, `TestParseMetadataBytesAssignsMissingPartitionFieldIDs`) still pass with this change.

Does this direction work for you, or was the cross-spec allocation meant to cover v1 legacy specs as well?
