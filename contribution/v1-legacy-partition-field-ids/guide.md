# 수정 가이드: v1 레거시 partition spec 의 field ID 를 spec 별로 매기기

난이도: ★★★★☆ (코드 20줄 정도. 기존 PR 의 설계 결정과 부딪히므로 설득이 핵심)

## 1. 문제를 한 줄로

옛날 v1 테이블의 partition spec 에는 `field-id` 가 없습니다. Java 는 **spec 마다 1000 부터** 번호를 매겼고, manifest 파일에도 그 번호가 적혀 있습니다. Go 는 **테이블 전체에서 이어서** 번호를 매기기 때문에, 두 번째 spec 부터 manifest 와 번호가 어긋납니다.

```
spec 0: [x]       Java: x=1000           Go: x=1000
spec 1: [x, y]    Java: x=1000, y=1001   Go: x=1001, y=1002   ← 어긋남
```

## 2. 먼저 할 일: issue 에 방향 질문

#1514 본문에 "allocate IDs across all specs in table metadata so generated IDs cannot collide between specs" 라고 적혀 있습니다. 즉 의도된 설계입니다. `issue.md` 의 질문으로 시작하고, 반론의 근거로 다음 세 가지를 듭니다.
1. 스펙 문구: "assigned sequentially starting at 1000 in the reference implementation"
2. Java 코드: `PartitionSpec.Builder.nextFieldId()` 가 spec 마다 999 에서 시작
3. manifest 에 이미 적힌 ID 와 맞아야 데이터를 제대로 읽을 수 있음. v1 에서는 spec 끼리 ID 가 겹치는 것이 원래 동작

## 3. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `table/metadata.go` | 2255-2345 | `assignMissingPartitionFieldIDs(FromMetadata)`: metadata JSON 을 `map[string]json.RawMessage` 로 열어서 빠진 `field-id` 를 채운 뒤 다시 직렬화 |
| 같은 파일 | `rawPartitionSpec` | spec 의 fields 를 `[]map[string]json.RawMessage` 로 들고 있는 임시 구조체 |
| `manifest.go` | `getFieldIDMap`, `dataFile.Partition()` | manifest Avro 스키마의 field-id 로 partition 값을 찾음 |
| `table/metadata_internal_test.go` | | #1514 테스트(`TestAssignMissingPartitionFieldIDsAcrossSpecs` 등) |

Go 개념 메모:
- **`json.RawMessage`** 는 "아직 해석하지 않은 JSON 바이트"입니다. 구조 전체를 struct 로 정의하지 않고 필요한 키만 꺼내 고칠 때 씁니다.
- `field["field-id"] = rawFieldID` 처럼 **map 은 참조 타입**이라, `spec.Fields` 안의 map 을 고치면 `specs` 를 다시 직렬화할 때 그대로 반영됩니다(slice 안의 map 을 복사하지 않음).
- `slices.ContainsFunc(s, func(x T) bool {...})` 는 Go 1.21+ 표준 라이브러리 함수로, 조건을 만족하는 원소가 하나라도 있는지 검사합니다.

## 4. 수정

`table/metadata.go` `assignMissingPartitionFieldIDsFromMetadata` (`slices` 는 이미 import 되어 있음)

```diff
 	lastAssignedID := iceberg.PartitionDataIDStart - 1
 	lastPartitionIDSet := false
+	legacyAssigned := false
 	if rawLastPartitionID, ok := metadata["last-partition-id"]; ok {
 		...
 	}
 
+	// v1 specs that predate partition field IDs were numbered per spec from
+	// PartitionDataIDStart by position (Java's PartitionSpec.Builder), and the
+	// manifests written for them carry those IDs. Restore them per spec rather
+	// than allocating table-wide, or partition tuples are read under the wrong IDs.
+	var formatVersion int
+	_ = json.Unmarshal(metadata["format-version"], &formatVersion)
+
 	missingFields := make([]map[string]json.RawMessage, 0)
 	for _, spec := range specs {
+		if formatVersion == 1 && !slices.ContainsFunc(spec.Fields, func(f map[string]json.RawMessage) bool {
+			_, ok := f["field-id"]
+
+			return ok
+		}) {
+			for i, field := range spec.Fields {
+				id := iceberg.PartitionDataIDStart + i
+				rawFieldID, err := json.Marshal(id)
+				if err != nil {
+					return nil, err
+				}
+				field["field-id"] = rawFieldID
+				lastAssignedID = max(lastAssignedID, id)
+				legacyAssigned = true
+			}
+
+			continue
+		}
 		for _, field := range spec.Fields {
 			...
 		}
 	}
 
-	if len(missingFields) == 0 {
+	if len(missingFields) == 0 && !legacyAssigned {
 		return b, nil
 	}
```

`lastAssignedID` 를 갱신해 두면 같은 metadata 에 섞여 있는 "일부만 빠진" spec 이 레거시 ID 와 겹치지 않습니다. `last-partition-id` 도 아래 기존 코드에서 함께 갱신됩니다.

## 5. 테스트

`table/metadata_internal_test.go`(`package table`) 에 추가합니다. `bytes` import 가 필요할 수 있습니다.

```go
const legacyV1TwoSpecs = `{
  "format-version": 1,
  "table-uuid": "9c12d441-03fe-4693-9a96-a0705ddf69c1",
  "location": "s3://bucket/t",
  "last-updated-ms": 1602638573590,
  "last-column-id": 2,
  "schema": {"type": "struct", "schema-id": 0, "fields": [
    {"id": 1, "name": "x", "required": true, "type": "long"},
    {"id": 2, "name": "y", "required": true, "type": "long"}]},
  "partition-specs": [
    {"spec-id": 0, "fields": [{"name": "x", "transform": "identity", "source-id": 1}]},
    {"spec-id": 1, "fields": [
      {"name": "x", "transform": "identity", "source-id": 1},
      {"name": "y", "transform": "identity", "source-id": 2}]}
  ],
  "default-spec-id": 1,
  "properties": {},
  "snapshots": []
}`

func TestLegacyV1PartitionFieldIDsPerSpec(t *testing.T) {
	meta, err := ParseMetadataString(legacyV1TwoSpecs)
	require.NoError(t, err)

	spec1 := meta.PartitionSpecByID(1)
	assert.Equal(t, []int{1000, 1001}, []int{spec1.Field(0).FieldID, spec1.Field(1).FieldID})

	// A manifest written for spec 1 by the reference implementation carries
	// partition struct field IDs 1000 (x) and 1001 (y).
	sc := meta.CurrentSchema()
	javaSpec1 := iceberg.NewPartitionSpecID(1,
		iceberg.PartitionField{SourceIDs: []int{1}, FieldID: 1000, Name: "x", Transform: iceberg.IdentityTransform{}},
		iceberg.PartitionField{SourceIDs: []int{2}, FieldID: 1001, Name: "y", Transform: iceberg.IdentityTransform{}})
	df, err := iceberg.NewDataFileBuilder(javaSpec1, iceberg.EntryContentData, "s3://bucket/t/f.parquet",
		iceberg.ParquetFile, map[int]any{1000: int64(10), 1001: int64(20)}, nil, nil, 1, 1)
	require.NoError(t, err)
	snap := int64(1)
	var buf bytes.Buffer
	mf, err := iceberg.WriteManifest("m.avro", &buf, 1, javaSpec1, sc, snap,
		[]iceberg.ManifestEntry{iceberg.NewManifestEntry(iceberg.EntryStatusADDED, &snap, nil, nil, df.Build())})
	require.NoError(t, err)
	entries, err := iceberg.ReadManifest(mf, bytes.NewReader(buf.Bytes()), true)
	require.NoError(t, err)

	rec := GetPartitionRecord(entries[0].DataFile(), spec1.PartitionType(sc))
	assert.Equal(t, int64(10), rec.Get(0))
	assert.Equal(t, int64(20), rec.Get(1))
}
```

수정 전 결과(2026-10-24 실행):

```text
spec 1 field IDs: [1001 1002] (Java: [1000 1001])
partition tuple read through parsed spec 1: x=20 y=<nil> (want x=10 y=20)
```

## 6. 실행

```bash
go test -count=1 -run 'TestLegacyV1PartitionFieldIDsPerSpec|TestAssignMissingPartitionFieldIDs|TestParseMetadataBytesAssignsMissing' -v ./table/
go test . ./table/...
make lint
git commit -s -m "fix(table): number legacy v1 partition field IDs per spec"
```

## 7. 주의

- 이 PR 은 **#1514 작성자와 리뷰어를 멘션해서** 의견을 받는 게 좋습니다. 리뷰가 길어질 수 있으니 다른 쉬운 항목을 먼저 머지하고 진행하세요.
- 실제 Java 0.x 시절 v1 메타데이터 샘플이 있으면 설득력이 크게 올라갑니다. `testdata/` 나 Java 저장소의 오래된 테스트 리소스(`TableMetadataParserTest`, `TestTableMetadata` 의 legacy JSON)를 찾아보세요.
- "일부 필드만 ID 가 빠진" v1 spec 은 Java 가 거부하는 입력입니다. 이번 수정에서는 #1514 동작을 그대로 두고, 범위 밖이라고 PR 에 명시하세요.
