# 수정 가이드: dictionary 컬럼에 row filter 를 걸면 에러가 남

난이도: ★★☆☆☆ (함수 하나 추가. Arrow 메모리 관리(Retain/Release)를 정확히 맞추는 게 핵심)

## 1. 문제를 한 줄로

Parquet 파일에 저장된 Arrow 스키마가 컬럼을 dictionary 타입으로 기록해 두면, 그 컬럼에 필터(`s = "a"`)를 걸었을 때 arrow-go 가 "utf8 이어야 하는데 dictionary 다"라며 에러를 냅니다.

## 2. 배경

- PyIceberg 로 pandas `Categorical` 컬럼을 append 하면 파일의 `ARROW:schema` 에 `dictionary<values=string, indices=int8>` 이 기록됩니다. iceberg-go 는 이 파일을 읽을 때 컬럼을 dictionary 배열로 받습니다.
- 스캔 pipeline 은 "파일 batch 읽기 → (삭제 처리) → **필터** → projection(`ToRequestedSchema`)" 순서입니다. projection 은 dictionary 를 풀어 주지만, 필터는 그 전에 실행되므로 dictionary 를 그대로 봅니다.
- 필터 식은 iceberg 스키마에서 만들어져서 `s` 가 `utf8` 이라고 가정합니다. arrow-go 는 실행 전에 컬럼 타입을 비교하는데, `large_utf8` 만 예외로 봐 주고 dictionary 는 거부합니다.

## 3. 코드 읽기 (main `2e69b8a`)

| 파일 | 줄 | 무엇 |
|---|---|---|
| `table/arrow_scanner.go` | 1651-1667 | `filterRecords`: batch 에 substrait 식을 실행해 mask 를 만들고 `compute.Filter` 로 행을 거름 |
| 같은 파일 | 2839-2840 | 필터를 pipeline 에 추가 |
| 같은 파일 | 2859 | 그 뒤에 `ToRequestedSchema` (여기서 dictionary 가 풀림) |
| `table/arrow_filter_plan.go` | 63 | 스캔 쪽 `filterRecords` 호출 |
| `table/transaction.go` | 2735-2777 | `prepareBatchFilter`: 또 하나의 `filterRecords` 호출 (`:2775`) |
| arrow-go v18.8.0 `arrow/compute/exprs/exec.go` | 525-536 | 에러를 내는 타입 비교 |
| `table/arrow_scanner.go` | 614-638 | `filePathValueType`/`filePathValues`: position delete 의 dictionary 처리 (참고용) |

`filterRecords` 를 부르는 곳은 위 두 곳뿐입니다(`grep -rn "filterRecords" --include='*.go' table/`). 그래서 `filterRecords` 한 곳만 고치면 스캔과 `prepareBatchFilter` 가 모두 고쳐집니다.

#2427(equality delete 의 dictionary 수정)은 `equality_delete_reader.go` 의 키 인코더에 dictionary case 를 넣은 것입니다. 행마다 키 바이트를 만드는 코드라 필터 경로에 가져다 쓸 수 있는 helper 는 아닙니다.

## 4. Go / Arrow 개념 메모

- **Dictionary 배열**: 같은 값이 반복되는 컬럼을 "값 목록(dictionary) + 각 행의 번호(indices)"로 저장하는 방식입니다. `["a","b","a"]` 는 dictionary `["a","b"]` 와 indices `[0,1,0]` 이 됩니다. 타입 표기는 `dictionary<values=utf8, indices=int8>` 입니다. 값을 꺼내려면 indices 로 dictionary 를 찾아야 하므로, 일반 `utf8` 배열과 타입이 다릅니다.
- **dictionary 풀기(decode)**: `compute.CastArray(ctx, arr, compute.SafeCastOptions(valueType))` 로 dictionary 배열을 값 타입 배열(`utf8` 등)로 바꿀 수 있습니다. arrow-go 의 cast 에 dictionary → 값 타입 kernel 이 있습니다.
- **Retain/Release**: Arrow 배열은 참조 카운트로 메모리를 관리합니다. 만든 쪽(또는 `Retain` 한 쪽)이 `Release` 해야 합니다. `array.NewRecordBatch` 는 넘긴 컬럼을 스스로 `Retain` 하므로, 우리가 cast 로 만든 배열은 record 를 만든 뒤 우리가 `Release` 해도 됩니다.
- **`defer` in loop**: 루프 안의 `defer` 는 루프가 아니라 함수가 끝날 때 실행됩니다. 여기서는 일부러 그 동작을 써서 `NewRecordBatch` 가 Retain 한 뒤에 release 되게 합니다. 컬럼 수만큼만 쌓이므로 문제없습니다.
- **`memory.NewCheckedAllocator`**: 테스트에서 할당량을 추적합니다. `defer mem.AssertSize(t, 0)` 가 release 누락을 잡아 줍니다.

## 5. 수정

`table/arrow_scanner.go`. 필터 평가용 batch 만 dictionary 를 풀고, 실제로 거르는 건 원본 batch 입니다. 그래야 출력 스키마가 바뀌지 않고, 뒤의 projection 이 하던 대로 동작합니다.

```diff
@@ func filterRecords(ctx context.Context, recordFilter expr.Expression) recProcess
 	return func(rec arrow.RecordBatch) (arrow.RecordBatch, error) {
 		defer rec.Release()
 
-		input := compute.NewDatumWithoutOwning(rec)
-		mask, err := exprs.ExecuteScalarExpression(ctx, rec.Schema(), recordFilter, input)
+		evalRec, err := decodeDictionaryColumns(ctx, rec)
+		if err != nil {
+			return nil, err
+		}
+		defer evalRec.Release()
+
+		mask, err := exprs.ExecuteScalarExpression(ctx, evalRec.Schema(), recordFilter,
+			compute.NewDatumWithoutOwning(evalRec))
 		if err != nil {
 			return nil, err
 		}
 		defer mask.Release()
 
+		input := compute.NewDatumWithoutOwning(rec)
+
 		result, err := compute.Filter(ctx, input, mask, *compute.DefaultFilterOptions())
 		if err != nil {
 			return nil, err
@@
 	}
 }
 
+// decodeDictionaryColumns returns rec with its top-level dictionary columns
+// cast to their value types. The substrait executor rejects a dictionary
+// column whose expression expects the value type, for example
+// dictionary<values=utf8> for a string predicate. The caller must release the
+// returned batch.
+func decodeDictionaryColumns(ctx context.Context, rec arrow.RecordBatch) (arrow.RecordBatch, error) {
+	var (
+		fields []arrow.Field
+		cols   []arrow.Array
+	)
+	for i, f := range rec.Schema().Fields() {
+		dt, ok := f.Type.(*arrow.DictionaryType)
+		if !ok {
+			continue
+		}
+		if cols == nil {
+			fields = slices.Clone(rec.Schema().Fields())
+			cols = slices.Clone(rec.Columns())
+		}
+		decoded, err := compute.CastArray(ctx, rec.Column(i), compute.SafeCastOptions(dt.ValueType))
+		if err != nil {
+			return nil, err
+		}
+		defer decoded.Release()
+		fields[i].Type = dt.ValueType
+		cols[i] = decoded
+	}
+
+	if cols == nil {
+		rec.Retain()
+
+		return rec, nil
+	}
+
+	md := rec.Schema().Metadata()
+
+	return array.NewRecordBatch(arrow.NewSchema(fields, &md), cols, rec.NumRows()), nil
+}
+
 type arrowScan struct {
```

`slices`, `array`, `compute` 는 이미 import 되어 있어 import 변경은 없습니다. dictionary 컬럼이 없으면 `rec.Retain()` 만 하고 그대로 돌려주므로 기존 경로에 추가 할당이 없습니다.

## 6. 테스트

`table/transaction_internal_test.go`(`package table`) 에서 `TestPrepareBatchFilterRejectsTransformedTerms` 바로 아래에 붙입니다. import 블록에 세 줄을 추가합니다.

```go
	"github.com/apache/arrow-go/v18/arrow"
	"github.com/apache/arrow-go/v18/arrow/array"
	"github.com/apache/arrow-go/v18/arrow/memory"
```

```go
// TestPrepareBatchFilterOnDictionaryColumn covers files whose embedded Arrow
// schema stores a column as a dictionary, as PyIceberg does for a pandas
// Categorical column. See https://github.com/apache/iceberg-go/issues/2425.
func TestPrepareBatchFilterOnDictionaryColumn(t *testing.T) {
	mem := memory.NewCheckedAllocator(memory.DefaultAllocator)
	defer mem.AssertSize(t, 0)

	schema := iceberg.NewSchema(1,
		iceberg.NestedField{ID: 1, Name: "id", Type: iceberg.PrimitiveTypes.Int64, Required: true},
		iceberg.NestedField{ID: 2, Name: "s", Type: iceberg.PrimitiveTypes.String},
	)

	dictType := &arrow.DictionaryType{IndexType: arrow.PrimitiveTypes.Int8, ValueType: arrow.BinaryTypes.String}
	arrSchema := arrow.NewSchema([]arrow.Field{
		{Name: "id", Type: arrow.PrimitiveTypes.Int64},
		{Name: "s", Type: dictType, Nullable: true},
	}, nil)

	bldr := array.NewRecordBuilder(mem, arrSchema)
	defer bldr.Release()
	bldr.Field(0).(*array.Int64Builder).AppendValues([]int64{1, 2, 3}, nil)
	sb := bldr.Field(1).(*array.BinaryDictionaryBuilder)
	for _, v := range []string{"a", "b", "a"} {
		require.NoError(t, sb.AppendString(v))
	}
	rec := bldr.NewRecordBatch()

	filter, err := prepareBatchFilter(iceberg.EqualTo(iceberg.Reference("s"), "a"), schema, true)
	require.NoError(t, err)

	filtered, err := filter(context.Background(), rec)
	require.NoError(t, err)
	defer filtered.Release()

	assert.True(t, arrow.TypeEqual(dictType, filtered.Column(1).DataType()),
		"filter must keep the input column type")
	assert.Equal(t, []int64{1, 3}, filtered.Column(0).(*array.Int64).Int64Values())
}
```

`filter` 가 `rec` 의 소유권을 가져가므로(내부에서 Release) 테스트에서 `rec.Release()` 를 따로 부르지 않습니다.

## 7. 실행

```bash
go test -count=1 -run TestPrepareBatchFilterOnDictionaryColumn -v ./table/
go test -count=1 ./table/...
go vet ./table/
make lint
git commit -s -m "fix(table): evaluate row filters on dictionary columns"
```

## 8. 수정 전/후 결과 (2026-11-06, main `2e69b8a`)

수정 전:

```text
--- FAIL: TestPrepareBatchFilterOnDictionaryColumn (0.00s)
    transaction_internal_test.go:85:
        	Error:      	Received unexpected error:
        	            	invalid: referenced field %!s(<nil>) was dictionary<values=utf8, indices=int8, ordered=false>, but should have been utf8
FAIL	github.com/apache/iceberg-go/table	0.922s
```

수정 후:

```text
=== RUN   TestPrepareBatchFilterOnDictionaryColumn
--- PASS: TestPrepareBatchFilterOnDictionaryColumn (0.00s)
ok  	github.com/apache/iceberg-go/table	0.913s
```

`go test ./table/...` 전체 통과, `go vet ./table/`, `golangci-lint run ./table/` (0 issues), goimports 차이 없음.

임시 end-to-end 테스트(커밋 안 함, `pqarrow.WithStoreSchema()` 로 dictionary 컬럼 파일 작성 → `AddDataFilesByPath` → 필터 스캔)도 수정 전에는 같은 에러, 수정 후에는 2행이 나왔습니다.

## 9. 주의

- 수정 전 실패 출력에 `checked_allocator.go: LEAK of 64 bytes` 줄이 몇 개 더 붙습니다. arrow-go 의 에러 경로(`exec.go:534`)가 참조한 컬럼을 release 하지 않아서 생기는 것이고, 이 PR 범위가 아닙니다. 수정 후에는 에러가 안 나므로 사라집니다.
- 필터에 쓰이지 않는 dictionary 컬럼도 풀립니다. 필터가 어떤 컬럼을 참조하는지 substrait 식에서 꺼내는 건 코드가 커집니다. 리뷰어가 성능을 물으면 "dictionary 컬럼이 있는 batch 만 해당되고, 참조 컬럼만 푸는 최적화는 따로 할 수 있다"고 답하면 됩니다.
- struct/list/map 안에 들어 있는 dictionary 는 풀지 않습니다. PyIceberg 재현은 top-level 컬럼이라 이걸로 충분합니다. PR 본문에 한 줄로 적어 두었습니다.
- arrow-go 쪽에서 고치는 방법(이슈에서 언급된 apache/arrow-go#440 처럼 타입 비교에 dictionary 예외 추가)도 있습니다. iceberg-go 에서 바로 고칠 수 있고 arrow-go 릴리스를 기다리지 않아도 되므로 이쪽을 택했다고 설명하면 됩니다.
- 이슈 작성자(@hcrosse)가 dictionary 관련 PR(#2427, #2442, #2445)을 계속 올리고 있습니다. 세 PR 모두 equality delete 만 건드려서 겹치지 않지만, PR 전에 issue.md 의 claim 댓글을 먼저 남기는 게 좋습니다.
- #2410(dictionary 컬럼을 유지하는 scan 옵션 제안)이 들어가면 이 경로를 타는 경우가 많아집니다. 그 경우에도 이 수정이 그대로 적용됩니다.
