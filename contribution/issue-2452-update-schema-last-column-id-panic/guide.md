# 수정 가이드: last-column-id 가 너무 작으면 UpdateSchema 가 panic

난이도: ★☆☆☆☆ (이미 있는 검증 함수를 한 번 호출하면 됨)

> 상태: 리포터(@Shruthirdy)가 이슈를 올리면서 직접 고치겠다고 했고, 12분 뒤 PR #2453 을 열었습니다. 이 가이드는 학습과 리뷰용입니다.

## 1. 문제를 한 줄로

메타데이터의 `last-column-id` 가 실제로 쓰이는 필드 id 보다 작으면, 컬럼을 추가할 때 이미 있는 id 가 다시 배정되고 `Apply()`/`Commit()` 이 error 를 돌려주지 않고 panic 합니다.

## 2. 배경: last-column-id

Iceberg 는 컬럼마다 고유한 정수 id 를 붙입니다. `last-column-id` 는 "지금까지 배정한 가장 큰 id" 를 기록하는 카운터입니다. 새 컬럼은 `last-column-id + 1` 을 받습니다. 정상 메타데이터라면 이 값은 모든 스키마의 최대 필드 id 보다 크거나 같습니다.

손상된 메타데이터(`x`=1, `y`=2, `z`=3 인데 `last-column-id` 가 2)는 파싱 단계에서 걸러지지 않습니다. 그래서 테이블은 정상적으로 열리고, 첫 스키마 변경에서 새 컬럼 `w` 가 id 3 을 받아 `z` 와 충돌합니다.

## 3. 코드 읽기 (main `2e69b8a`)

| 파일 | 줄 | 무엇 |
|---|---|---|
| `table/update_schema.go` | 138-181 | `NewUpdateSchema`: 생성자. 실패하면 `u.err` 에 저장하고 `u` 를 그대로 돌려줌 |
| 같은 파일 | 175 | `u.lastColumnID = meta.LastColumnID()`: 검증 없이 카운터를 세팅 |
| 같은 파일 | 246-249 | `assignNewColumnID`: `lastColumnID++` 후 반환 |
| 같은 파일 | 1097, 1127 | `Apply`: 마지막에 `iceberg.NewSchema` 호출 |
| `schema.go` | 95-96 | `checkDuplicateFieldIDs` 가 실패하면 `panic(err)` |
| `table/metadata.go` | 1313-1327 | `validateLastColumnID`: 바로 이 경우를 `ErrInvalidMetadata` 로 알려 주는 기존 함수 |
| `table/transaction.go` | 676 | `Transaction.UpdateSchema`: `NewUpdateSchema` 를 부르는 유일한 곳 |

Go 개념 메모:
- **error 와 panic**: Go 에서 예상할 수 있는 실패는 `error` 값으로 돌려줍니다. `panic` 은 프로그램 버그용이고, 잡지 않으면 프로세스가 죽습니다. 손상된 메타데이터는 외부 입력이므로 `error` 로 돌려줘야 합니다.
- **지연된 에러(`u.err`)**: `UpdateSchema` 는 `AddColumn(...).Apply()` 처럼 메서드를 체인으로 이어 부르는 빌더입니다. 생성자에서 에러를 바로 돌려줄 수 없으니 `u.err` 에 넣어 두고, `Apply`·`BuildUpdates`·`Commit` 이 시작할 때 `u.err` 를 확인해 돌려줍니다. nil transaction 처리(159줄)가 같은 방식입니다.
- **같은 패키지의 비공개 메서드**: `validateLastColumnID` 는 소문자로 시작하므로 `table` 패키지 안에서만 보입니다. `update_schema.go` 도 `table` 패키지라 그대로 호출할 수 있습니다. `meta` 는 `*MetadataBuilder` 입니다(`txnMeta()` 반환값).
- **`require.NotPanics`**: 테스트 안에서 panic 이 나면 테스트 실패로 바꿔 줍니다. 이게 없으면 테스트 바이너리 전체가 죽습니다.

## 4. 수정

`table/update_schema.go`, `NewUpdateSchema` 안

```diff
 	u.lastColumnID = meta.LastColumnID()
+	// A last-column-id below an id already in use would hand out a duplicate
+	// id, and iceberg.NewSchema panics on duplicates in Apply.
+	if err := meta.validateLastColumnID(); err != nil {
+		u.err = err
+
+		return u
+	}
 
 	for _, opt := range opts {
```

## 5. 테스트

`table/update_schema_test.go`(`package table`) 끝에 추가합니다. import 에 `"strings"` 를 더해야 합니다(goimports 가 자동으로 넣어 줍니다).

```go
func TestUpdateSchemaLastColumnIDBelowHighestFieldID(t *testing.T) {
	// highest field id is 3 ("z"), but last-column-id says 2,
	// so a new column would be assigned id 3 again.
	meta, err := ParseMetadataString(strings.Replace(ExampleTableMetadataV2,
		`"last-column-id": 3`, `"last-column-id": 2`, 1))
	require.NoError(t, err)
	require.Equal(t, 2, meta.LastColumnID())

	tbl := New([]string{"db", "t"}, meta, "", nil, nil)
	update := func() *UpdateSchema {
		return tbl.NewTransaction().UpdateSchema(true, false).
			AddColumn([]string{"w"}, iceberg.PrimitiveTypes.Int32, "", false, nil)
	}

	var applyErr, commitErr error
	require.NotPanics(t, func() { _, applyErr = update().Apply() })
	require.NotPanics(t, func() { commitErr = update().Commit() })
	assert.ErrorIs(t, applyErr, ErrInvalidMetadata)
	assert.ErrorIs(t, commitErr, ErrInvalidMetadata)
}
```

## 6. 실행

```bash
go test -count=1 -run TestUpdateSchemaLastColumnIDBelowHighestFieldID -v ./table/
go test ./table/...
go vet ./table/...
make lint
git commit -s -m "fix(table): return an error from UpdateSchema when last-column-id is too low"
```

## 7. 수정 전/후 결과 (2026-11-06, main `2e69b8a`)

수정 전:

```text
--- FAIL: TestUpdateSchemaLastColumnIDBelowHighestFieldID (0.00s)
        	Error:      	func (assert.PanicTestFunc)(0x1059d0020) should not panic
        	            		Panic value:	invalid schema: multiple fields for id 3: z and w
```

수정 후:

```text
=== RUN   TestUpdateSchemaLastColumnIDBelowHighestFieldID
--- PASS: TestUpdateSchemaLastColumnIDBelowHighestFieldID (0.00s)
ok  	github.com/apache/iceberg-go/table
```

`go test ./table/...` 전체 통과, `go vet ./table/...` 통과.

## 8. 주의

- **이미 PR #2453 이 있습니다.** 같은 수정으로 PR 을 또 열면 중복입니다. 하고 싶다면 #2453 에서 재현과 테스트 결과를 남기는 리뷰 정도가 적당합니다.
- 이 수정은 `last-column-id` 를 자동으로 고치지 않고 에러만 돌려줍니다. `max(last-column-id, 최대 필드 id)` 로 보정하는 방법도 있지만, 이슈가 기대하는 동작은 에러이고 Java 도 이 값을 보정하지 않습니다.
- 파싱(`ParseMetadataString`)에서 막는 방법도 생각할 수 있습니다. 하지만 그러면 이런 메타데이터를 가진 테이블은 읽기조차 안 됩니다. 이슈도 스키마 변경 시점의 에러를 요청하고 있습니다.
