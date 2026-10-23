# 수정 가이드: RenameColumn 이 호출자의 path slice 를 덮어씀

난이도: ★☆☆☆☆ (1줄. 대신 Go slice 의 핵심 함정을 배우기 좋은 문제)

## 1. 문제를 한 줄로

`append(path[:len(path)-1], newName)` 이 **새 slice 를 만들지 않고 `path` 의 마지막 칸을 덮어씁니다.** 그 결과 사용자가 넘긴 `[]string{"address","city"}` 가 `{"address","town"}` 으로 바뀝니다.

## 2. Go 개념: slice 는 "배열을 보는 창"

slice 는 내부적으로 `(포인터, 길이 len, 용량 cap)` 세 값을 가집니다.

```go
path := []string{"address", "city"} // len=2, cap=2, 배열 [address, city]
prefix := path[:1]                  // len=1, cap=2  ← 같은 배열을 가리킴!
out := append(prefix, "town")       // cap 이 남아 있으니 새 배열을 만들지 않고 배열[1] 에 씀
fmt.Println(path)                   // [address town]  ← 원본이 바뀜
```

`append` 는 용량이 남아 있으면 **제자리에 씁니다.** 용량이 부족할 때만 새 배열을 할당합니다. 그래서 "다른 slice 의 일부를 잘라 append" 하는 코드는 원본을 망가뜨릴 수 있습니다.

해결책:
- `slices.Clone(path[:len(path)-1])`: 복사본을 만든 뒤 append (이 저장소 스타일)
- 또는 `path[:len(path)-1:len(path)-1]`: 3-index slice 로 cap 을 len 과 같게 만들어 append 가 새 배열을 할당하도록 강제. 동작은 하지만 읽기 어려움

## 3. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `table/update_schema.go` | 547-557 | `RenameColumn`: 바로 실행하지 않고 `u.ops` 에 **클로저(closure)** 로 저장. `path` 변수를 그대로 캡처함 |
| 같은 파일 | 420-475 | `updateColumn`: 450줄이 문제의 `append` |
| 같은 파일 | `Apply`, `BuildUpdates` | 호출할 때마다 `u.ops` 를 처음부터 다시 실행 |

Go 개념 메모:
- **클로저**: `func() error { return u.updateColumn(path, ...) }` 는 `path` 를 "복사"하지 않고 같은 slice 헤더를 들고 있습니다. 배열이 바뀌면 다음 실행 때 바뀐 값을 봅니다.
- 그래서 `Apply()` 를 두 번 부르면 두 번째 실행은 `{"address","town"}` 으로 `findField` 를 하게 되고 `field not found` 가 납니다.

## 4. 수정

`table/update_schema.go:450` (`slices` 는 이미 import 되어 있음)

```diff
-		newFullName := strings.Join(append(path[:len(path)-1], update.Name.Val), ".")
+		newFullName := strings.Join(append(slices.Clone(path[:len(path)-1]), update.Name.Val), ".")
```

같은 패턴이 다른 곳에도 있는지 확인하기:

```bash
grep -rn 'append(path\[:' --include='*.go' . | grep -v _test.go
```

(2026-10-24 기준 이 한 곳뿐)

## 5. 테스트

`table/update_schema_test.go`(`package table`) 끝에 추가합니다. `testMetadata` 는 같은 파일 52줄에 정의되어 있습니다.

```go
func TestRenameColumnDoesNotMutatePath(t *testing.T) {
	tbl := New([]string{"id"}, testMetadata, "", nil, nil)

	path := []string{"address", "city"}
	sc, err := NewUpdateSchema(tbl.NewTransaction(), true, false).
		RenameColumn(path, "town").
		UpdateColumn(path, ColumnUpdate{Doc: iceberg.Optional[string]{Valid: true, Val: "the town"}}).
		Apply()
	require.NoError(t, err)
	assert.Equal(t, []string{"address", "city"}, path)

	f, ok := sc.FindFieldByName("address.town")
	require.True(t, ok)
	assert.Equal(t, "the town", f.Doc)
}

func TestRenameColumnApplyThenBuildUpdates(t *testing.T) {
	tbl := New([]string{"id"}, testMetadata, "", nil, nil)
	us := NewUpdateSchema(tbl.NewTransaction(), true, false).
		RenameColumn([]string{"address", "city"}, "town")

	_, err := us.Apply()
	require.NoError(t, err)
	_, _, err = us.BuildUpdates()
	require.NoError(t, err)
}
```

수정 전 결과(2026-10-24 실행):

```text
path after Apply: [address town]
Error: Received unexpected error: field not found: address.town
--- FAIL: TestRenameColumnDoesNotMutatePath
Error: Received unexpected error: field not found: address.town
--- FAIL: TestRenameColumnApplyThenBuildUpdates
```

## 6. 실행

```bash
go test -count=1 -run 'TestRenameColumn' -v ./table/
go test ./table/...
make lint
git commit -s -m "fix(table): do not overwrite the caller's path in UpdateSchema rename"
```

## 7. 주의

- 같은 함수의 identifier 처리 버그(`update-schema-case-insensitive-rename-identifier`)는 **별도 PR** 로 냅니다(CONTRIBUTING: one issue per PR). 이 PR 이 먼저 머지되면 그쪽은 rebase 만 하면 됩니다.
