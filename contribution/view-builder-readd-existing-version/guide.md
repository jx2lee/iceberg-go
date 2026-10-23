# 수정 가이드: view MetadataBuilder 가 base 에 있던 version 을 중복 추가함

난이도: ★★☆☆☆ (수정은 몇 줄. 대신 builder 가 base 와 updates 를 어떻게 구분하는지 이해해야 함)

## 1. 문제를 한 줄로

"이 version 은 이미 있다"는 것까지는 알아내는데, 그것이 **이번 builder 에서 추가한 것**일 때만 멈춥니다. **원래 metadata(base)에 있던 것**이면 한 번 더 추가합니다. 그 결과 version-id 1 이 두 개가 되어 `Build()` 가 실패합니다.

## 2. 배경: builder 패턴

```
base metadata ──MetadataBuilderFromBase──▶ builder ──AddVersion/SetCurrentVersion...──▶ Build() ──▶ 새 metadata + changes(updates)
```

- `b.versionList` / `b.versionsById`: base 의 version + 이번에 추가한 version
- `b.updates`: **이번에** 일어난 변경만 담음 (REST 로 보낼 update 목록)
- `b.lastAddedVersionID`: `SetCurrentVersionID(LastAddedID)` 처럼 "방금 추가한 것"을 가리키는 특수값을 풀 때 씀

Java 의 규칙: 이미 있으면 **항상 기존 ID 를 반환**하고, `lastAddedVersionID` 는 이번 builder 에서 추가한 경우에만 그 ID, 아니면 `nil`.

## 3. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `view/metadata_builder.go` | 114-132 | `SetCurrentVersion`: schema 추가 → `addVersion` → `SetCurrentVersionID` |
| 같은 파일 | 147-200 | `addVersion`: 문제의 early-return 이 for 문 **안**에만 있음 |
| 같은 파일 | 505-516 | `reuseOrCreateNewVersionID`: `Equals` 가 같으면 기존 ID 반환(ID·timestamp 는 비교하지 않음) |
| `view/metadata_builder_test.go` | 35-70 | `newTestBuilder`, `newTestVersionWithSQL`, `newTestSchema` 헬퍼 |

Go 개념 메모:
- `upd.(*addViewVersionUpdate)` 는 **type assertion** 입니다. 인터페이스 값 `upd` 의 실제 타입이 `*addViewVersionUpdate` 인지 확인하고, `ok` 로 결과를 받습니다.
- `b.lastAddedVersionID` 의 타입은 `*int64` 입니다. Go 에는 "없음"을 나타내는 Optional 타입이 없어서 `nil` 포인터로 표현합니다. `&newVersionID` 는 지역 변수의 주소인데, Go 는 escape analysis 로 이 변수를 heap 에 옮기므로 안전합니다.

## 4. 수정

```diff
 	// Check if this version was added in an update already
 	if _, ok := b.versionsById[newVersionID]; ok {
+		b.lastAddedVersionID = nil
 		for _, upd := range b.updates {
 			if vvu, ok := upd.(*addViewVersionUpdate); ok && vvu.Version.VersionID == newVersionID {
 				b.lastAddedVersionID = &newVersionID
 
-				return newVersionID, nil
+				break
 			}
 		}
+
+		return newVersionID, nil
 	}
```

주석도 "Reuse an existing version; only a version added by this builder counts as last added" 처럼 바꿔 주세요.

## 5. 테스트

`view/metadata_builder_test.go`(`package view`) 에 추가합니다.

```go
func TestSetCurrentVersionReusesBaseVersion(t *testing.T) {
	base, err := newTestBuilder().
		SetLoc("location").
		AddSchema(newTestSchema(0)).
		AddVersion(newTestVersionWithSQL(1, LastAddedID, "select 1")).
		SetCurrentVersionID(LastAddedID).
		Build()
	require.NoError(t, err)

	b, err := MetadataBuilderFromBase(base.Metadata)
	require.NoError(t, err)
	// e.g. CREATE OR REPLACE VIEW with an unchanged definition
	res, err := b.SetCurrentVersion(newTestVersionWithSQL(1, 0, "select 1"), newTestSchema(0)).Build()
	require.NoError(t, err)

	ids := []int64{}
	for _, v := range res.Metadata.Versions() {
		ids = append(ids, v.VersionID)
	}
	assert.Equal(t, []int64{1}, ids)
	assert.Empty(t, res.Changes)
}

func TestSetCurrentVersionRevertsToEarlierVersion(t *testing.T) {
	base, err := newTestBuilder().
		SetLoc("location").
		AddSchema(newTestSchema(0)).
		AddVersion(newTestVersionWithSQL(1, LastAddedID, "select 1")).
		AddVersion(newTestVersionWithSQL(2, LastAddedID, "select 2")).
		SetCurrentVersionID(2).
		Build()
	require.NoError(t, err)

	b, err := MetadataBuilderFromBase(base.Metadata)
	require.NoError(t, err)
	res, err := b.SetCurrentVersion(newTestVersionWithSQL(3, 0, "select 1"), newTestSchema(0)).Build()
	require.NoError(t, err)
	assert.Equal(t, int64(1), res.Metadata.CurrentVersionID())
	assert.Len(t, res.Metadata.Versions(), 2)
}
```

수정 전 결과(2026-10-24 실행): 두 테스트 모두 `invalid view metadata: duplicate version-id 1` 로 FAIL. 수정 후에는 `go test ./view/...` 전체가 통과합니다.

## 6. 실행

```bash
go test -count=1 -run 'TestSetCurrentVersion' -v ./view/
go test ./view/...
make lint
git commit -s -m "fix(view): reuse an existing base version instead of adding a duplicate"
```

## 7. 주의

- `lastAddedVersionID = nil` 이 핵심입니다. 이게 없으면 직전 `AddVersion` 의 값이 남아서 `SetCurrentVersionID(LastAddedID)` 가 엉뚱한 version 을 가리킬 수 있습니다. 리뷰어가 물어보면 Java `addVersionInternal` 의 동작을 근거로 답하면 됩니다.
- `udf/metadata_builder.go` 에도 비슷한 builder 가 있습니다. 같은 패턴인지 한번 보고, 같으면 issue 에 "udf 에도 같은 구조가 있다"고 적어 두세요(같은 PR 에 넣지는 말 것).
