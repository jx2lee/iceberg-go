# 수정 가이드: snapshot summary 가 부모에 없던 total-* 값을 만들어 냄

난이도: ★★☆☆☆ (5줄. "없음"과 "0"을 구분하는 연습)

## 1. 문제를 한 줄로

부모 snapshot summary 에 `total-records` 가 **없으면** Go 는 0 으로 간주하고 이번 커밋의 값만 더해서 씁니다. Java·PyIceberg 는 **없으면 쓰지 않습니다**(모르는 값을 지어내지 않음).

## 2. 배경

snapshot summary 는 `map[string]string` 입니다. `added-records` 는 이번 커밋의 변화량이고, `total-records` 는 누적값입니다(부모의 total + added − deleted).

| 상황 | Java | Go(현재) |
|---|---|---|
| 부모 snapshot 없음(첫 커밋) | 모든 total 을 0 에서 시작 | 같음 (`previous == nil` 분기) |
| 부모는 있는데 total 키가 없음 | 그 total 은 쓰지 않음 | 0 으로 보고 씀 ← 버그 |
| 부모 total 이 숫자가 아님 | 쓰지 않음 | 0 으로 보고 씀 ← 버그 |
| 계산 결과가 음수 | 쓰지 않음 | 쓰지 않음(같음). 하지만 다음 커밋에서 0 부터 다시 시작 ← 버그 |

## 3. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `table/snapshots.go` | 716-756 | `updateSnapshotSummaries`. 728-736 이 `previous == nil` 일 때 0 기본값, 738-746 이 `updateTotals` 클로저 |
| `table/snapshot_producers.go` | 1365 | `rebaseSummary`: 호출하는 곳 |
| `iceberg.Properties.GetInt64` | | 키가 없거나 파싱에 실패하면 기본값을 돌려주는 헬퍼. 이 helper 가 "없음"을 0 으로 뭉개는 원인 |

Go 개념 메모:
- `prev, ok := previous[totalProp]` 처럼 map 조회의 두 번째 값으로 "키 존재 여부"를 알 수 있습니다. `GetInt64(k, 0)` 는 이 정보를 버립니다.
- `strconv.ParseInt(s, 10, 64)` 는 `(int64, error)` 를 돌려줍니다. Go 에는 예외가 없어서, 파싱 실패도 `err != nil` 로 확인합니다.
- `updateTotals := func(...) {...}` 는 함수 안에 정의한 **클로저**입니다. `sum`, `previous` 를 캡처하고 있어서 인자로 넘기지 않아도 됩니다.

## 4. 수정

`strconv` 는 이미 import 되어 있습니다.

```diff
 	updateTotals := func(totalProp, addedProp, removedProp string) {
-		newTotal := previous.GetInt64(totalProp, 0)
+		// Like Java's SnapshotProducer.updateTotal, only carry a total forward
+		// when the parent summary has it.
+		prev, ok := previous[totalProp]
+		if !ok {
+			return
+		}
+		newTotal, err := strconv.ParseInt(prev, 10, 64)
+		if err != nil {
+			return
+		}
 		newTotal += sum.Properties.GetInt64(addedProp, 0)
 		newTotal -= sum.Properties.GetInt64(removedProp, 0)
```

## 5. 테스트

`table/snapshots_internal_test.go`(`package table`) 에 추가합니다.

```go
func TestSummaryTotalsMissingInParent(t *testing.T) {
	// parent summary written by a writer that did not track totals
	previous := iceberg.Properties{"added-records": "10"}

	sum, err := updateSnapshotSummaries(Summary{
		Operation:  OpAppend,
		Properties: iceberg.Properties{"added-records": "5", "added-data-files": "1", "added-files-size": "100"},
	}, previous)
	require.NoError(t, err)
	for _, k := range []string{totalRecordsKey, totalDataFilesKey, totalFileSizeKey, totalDeleteFilesKey} {
		assert.NotContains(t, sum.Properties, k)
	}
}

func TestSummaryTotalNotResurrectedAfterInvalidation(t *testing.T) {
	// parent says 1 data file but the commit deletes 2: the total is dropped
	s1, err := updateSnapshotSummaries(Summary{
		Operation:  OpDelete,
		Properties: iceberg.Properties{"deleted-data-files": "2"},
	}, iceberg.Properties{totalDataFilesKey: "1"})
	require.NoError(t, err)
	require.NotContains(t, s1.Properties, totalDataFilesKey)

	// the next commit must not restart the dropped total from 0
	s2, err := updateSnapshotSummaries(Summary{
		Operation:  OpAppend,
		Properties: iceberg.Properties{"added-data-files": "1"},
	}, s1.Properties)
	require.NoError(t, err)
	assert.NotContains(t, s2.Properties, totalDataFilesKey)
}
```

`assert.NotContains(t, map, key)` 는 map 에 그 key 가 없는지 확인합니다.

수정 전 결과(2026-10-24 실행):

```text
total-records=5 fabricated; parent had no such total
total-data-files=1 fabricated; parent had no such total
total-files-size=100 fabricated; parent had no such total
total-delete-files=0 fabricated; parent had no such total
total-data-files=1 resurrected after being invalidated
```

## 6. 실행

```bash
go test -count=1 -run 'TestSummaryTotal' -v ./table/
go test ./table/...
make lint
git commit -s -m "fix(table): do not invent snapshot totals missing from the parent summary"
```

## 7. 주의

- 관련된 작은 차이: 부모 snapshot 은 있는데 `Summary` 가 nil 이면 Go 는 `previous == nil` 로 넘겨서 0 기본값을 씁니다. Java 는 이때 빈 map 을 써서 total 을 쓰지 않습니다. 호출부(`snapshot_producers.go` 에서 부모 summary 를 꺼내는 곳)를 보고, 같은 PR 에 넣을지 리뷰어에게 물어보세요. 기본은 이번 PR 범위 밖입니다.
- #2215(DV 를 position delete 로 셈)는 같은 파일을 건드릴 수 있습니다. 담당자(akritishakya)의 PR 과 충돌하지 않게, PR 을 올리기 전에 rebase 하세요.
