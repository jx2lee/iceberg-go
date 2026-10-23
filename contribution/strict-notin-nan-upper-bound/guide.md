# 수정 가이드: strict NotIn 이 NaN 상한을 무시함

난이도: ★☆☆☆☆ (if 문 하나. 개념 이해가 더 중요)

## 1. 문제를 한 줄로

파일 통계의 상한(upper bound)이 NaN 이면 "이 통계는 믿을 수 없다"로 처리해야 하는데, `NotIn` 은 하한만 검사합니다. 그래서 NaN 상한이 들어오면 "모든 행이 조건을 만족한다"고 잘못 답합니다.

## 2. 배경: inclusive vs strict 평가기

파일마다 컬럼의 min(lower)·max(upper) 통계가 있습니다. 평가기는 이 통계만 보고 파일 단위로 판단합니다.

| 평가기 | 질문 | 틀리면 |
|---|---|---|
| inclusive | "이 파일에 조건을 만족하는 행이 **있을 수도** 있나?" | 읽어야 할 파일을 건너뜀 |
| strict | "이 파일의 **모든** 행이 조건을 만족하나?" | 행을 하나씩 보지 않고 파일 전체를 지움 → **데이터 손실** |

그래서 strict 는 애매하면 무조건 `rowsMightNotMatch`(false) 를 돌려줘야 합니다.

`x NOT IN (7, 8)` 의 strict 로직: 값 목록에서 `[lower, upper]` 범위 밖에 있는 값을 지웁니다(어차피 파일에 없을 값이니까). 남는 값이 없으면 "파일의 어떤 행도 7·8 이 아니다" → must match.

NaN 상한일 때: Go 의 `cmp.Compare` 는 NaN 을 **가장 작은 값**으로 취급합니다. 그래서 "7 > NaN, 8 > NaN, 둘 다 상한보다 크니 범위 밖" → 둘 다 지움 → must match. 실제 행에는 7.0 이 있는데도 이렇게 답합니다.

## 3. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `table/evaluators.go` | 1690-1730 | `VisitNotEqual`: 하한·상한 모두 `m.isNan(...)` 검사 (모범 답안) |
| 같은 파일 | 1772-1810 | `VisitNotIn`: 하한만 검사 |
| 같은 파일 | 104 | `removeBoundCheck`: 타입별 제네릭 비교 |
| `table/evaluators_test.go` | 1283 | `mockDataFile`: 통계를 직접 넣어 만드는 테스트용 가짜 DataFile |

Go 개념 메모:
- `iceberg.Float64Literal(7).MarshalBinary()` 는 통계 bound 를 저장하는 바이트 형식(single-value serialization)을 만듭니다. `mockDataFile.lowerBounds` 에 이 바이트를 넣습니다.
- `newStrictMetricsEvaluator(schema, expr, caseSensitive, includeEmptyFiles)` 는 `func(iceberg.DataFile) (bool, error)` 를 돌려줍니다. 이처럼 **함수를 값으로 돌려주는** 스타일이 이 패키지에 많습니다.

## 4. 수정

`table/evaluators.go` `VisitNotIn` 의 upper bound 블록

```diff
 	if upperBoundBytes := m.upperBounds[fieldID]; upperBoundBytes != nil {
 		upperBound, err := iceberg.LiteralFromBytes(field.Type, upperBoundBytes)
 		if err != nil {
 			panic(err)
 		}
 
+		if m.isNan(upperBound) {
+			// NaN indicates unreliable bounds.
+			return rowsMightNotMatch
+		}
+
 		values = removeBoundCheck(upperBound, values, -1)
```

## 5. 테스트

`table/evaluators_test.go`(`package table`) 끝에 추가합니다. `math` 는 이미 import 되어 있습니다.

```go
func TestStrictNotInNaNUpperBound(t *testing.T) {
	sc := iceberg.NewSchema(1, iceberg.NestedField{ID: 1, Name: "x", Type: iceberg.PrimitiveTypes.Float64, Required: true})
	lo, _ := iceberg.Float64Literal(7).MarshalBinary()
	hi, _ := iceberg.Float64Literal(math.NaN()).MarshalBinary()
	// rows {7.0, NaN}: the 7.0 row does not satisfy x NOT IN (7.0, 8.0)
	df := &mockDataFile{
		count: 2, valueCounts: map[int]int64{1: 2}, nullCounts: map[int]int64{1: 0},
		lowerBounds: map[int][]byte{1: lo}, upperBounds: map[int][]byte{1: hi},
	}

	for _, expr := range []iceberg.BooleanExpression{
		iceberg.NotIn(iceberg.Reference("x"), 7.0, 8.0),
		iceberg.NotIn(iceberg.Reference("x"), 7.0), // binds to NotEqual
	} {
		eval, err := newStrictMetricsEvaluator(sc, expr, true, false)
		require.NoError(t, err)
		mustMatch, err := eval(df)
		require.NoError(t, err)
		assert.Falsef(t, mustMatch, "%s: strict evaluator claimed every row matches", expr)
	}
}
```

수정 전 결과(2026-10-24 실행):

```text
NotIn(term=Reference(name='x'), {[7 8]}): strict evaluator claimed every row matches
--- FAIL: TestStrictNotInNaNUpperBound
```

## 6. 실행

```bash
go test -count=1 -run TestStrictNotInNaNUpperBound -v ./table/
go test ./table/...
make lint
git commit -s -m "fix(table): treat a NaN upper bound as unreliable in strict NotIn"
```

## 7. 주의

- 리뷰어가 "NaN bound 는 스펙상 쓰면 안 되는 값 아니냐"고 물을 수 있습니다. 같은 파일의 다른 visitor(`VisitNotEqual`, `VisitGreaterThan` 등)가 이미 "NaN indicates unreliable bounds" 로 방어하고 있으니 일관성을 맞추는 수정이라고 답하면 됩니다.
- 비슷한 누락이 더 있는지 확인: `grep -n "removeBoundCheck\|isNan" table/evaluators.go` 로 bound 를 읽는 곳마다 NaN 검사가 있는지 봅니다(inclusive 쪽은 "건너뛰지 않는" 방향이 안전하므로 기준이 다릅니다).
