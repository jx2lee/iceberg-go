# 수정 가이드: 대소문자 무시 모드에서 identifier 컬럼 rename 실패

난이도: ★★☆☆☆ (수정은 3줄. map 의 key 가 무엇인지 따라가는 게 핵심)

## 1. 문제를 한 줄로

identifier 목록(map)의 key 는 스키마에 적힌 철자(`"id"`)인데, rename 할 때는 사용자가 입력한 철자(`"ID"`)로 찾습니다. 그래서 찾지 못하고, rename 후에는 없어진 이름 `"id"` 를 identifier 로 들고 있게 됩니다.

## 2. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `table/update_schema.go` | 209-226 | `initIdentifierFieldNames`: `FindColumnName(id)` 로 **스키마 철자**를 key 로 넣음 |
| 같은 파일 | 420-475 | `updateColumn`: 469줄에서 `fullName`(**사용자 철자**)으로 조회 |
| 같은 파일 | 397 | `deleteColumn`: 같은 패턴 |
| 같은 파일 | `findField` | `caseSensitive` 에 따라 `FindFieldByName` / `FindFieldByNameCaseInsensitive` 사용 |

Go 개념 메모:
- `map[string]struct{}` 는 Go 의 관용적인 **set** 입니다. `struct{}` 는 크기가 0 인 타입이라 메모리를 쓰지 않습니다. 넣을 때는 `m[k] = struct{}{}`, 확인할 때는 `_, ok := m[k]`.
- `name, ok := u.schema.FindColumnName(field.ID)` 처럼 **두 번째 반환값 `ok`** 로 존재 여부를 알려 주는 함수가 많습니다. 여기서는 `field` 를 방금 찾았으니 항상 `true` 라서 `_` 로 버려도 됩니다.

## 3. 수정

`table/update_schema.go` `updateColumn` 의 rename 블록 끝

```diff
-		if _, ok := u.identifierFieldNames[fullName]; ok {
-			delete(u.identifierFieldNames, fullName)
+		// identifierFieldNames is keyed by the schema's own spelling, which may
+		// differ from fullName when matching case-insensitively
+		canonicalName, _ := u.schema.FindColumnName(field.ID)
+		if _, ok := u.identifierFieldNames[canonicalName]; ok {
+			delete(u.identifierFieldNames, canonicalName)
 			u.identifierFieldNames[newFullName] = struct{}{}
 		}
```

`deleteColumn`(397줄)도 같은 방식으로:

```diff
-	delete(u.identifierFieldNames, fullName)
+	if canonicalName, ok := u.schema.FindColumnName(field.ID); ok {
+		delete(u.identifierFieldNames, canonicalName)
+	}
```

생각해 볼 점: `newFullName` 은 부모 경로가 사용자 철자라서 `"ADDRESS.town"` 처럼 섞일 수 있습니다. 중첩 identifier 를 대소문자 무시 모드로 rename 하는 경우까지 맞추려면 `newFullName` 도 canonical 부모 경로 + 새 이름으로 만드는 게 정확합니다. 리뷰에서 지적받을 수 있으니 테스트를 하나 더 넣어 확인해 보세요.

## 4. 테스트

`table/update_schema_test.go`(`package table`) 에 추가합니다.

```go
func TestCaseInsensitiveRenameIdentifierColumn(t *testing.T) {
	sch := iceberg.NewSchemaWithIdentifiers(1, []int{1},
		iceberg.NestedField{ID: 1, Name: "id", Type: iceberg.PrimitiveTypes.Int32, Required: true},
		iceberg.NestedField{ID: 2, Name: "data", Type: iceberg.PrimitiveTypes.String},
	)
	md, err := NewMetadata(sch, nil, UnsortedSortOrder, "", nil)
	require.NoError(t, err)
	tbl := New([]string{"t"}, md, "", nil, nil)

	sc, err := NewUpdateSchema(tbl.NewTransaction(), false, false).
		RenameColumn([]string{"ID"}, "key").
		Apply()
	require.NoError(t, err)
	assert.Equal(t, []int{1}, sc.IdentifierFieldIDs)
}
```

수정 전 결과(2026-10-24 실행):

```text
Error: Received unexpected error: identifier field not found: id
--- FAIL: TestCaseInsensitiveRenameIdentifierColumn
```

## 5. 실행

```bash
go test -count=1 -run TestCaseInsensitiveRenameIdentifierColumn -v ./table/
go test ./table/...
make lint
git commit -s -m "fix(table): keep identifier fields on a case-insensitive rename"
```

## 6. 주의

- 같은 함수의 `update-schema-rename-mutates-path` 와 줄이 가깝습니다. 그쪽을 먼저 머지한 뒤 이 브랜치를 rebase 하세요.
- Java 의 `SchemaUpdate.renameColumn` 도 identifier set 을 사용자 이름으로 다룹니다. 그래서 "Java parity" 가 아니라 "Go 의 caseSensitive=false 모드 안에서의 일관성" 문제로 설명하는 게 정확합니다.
