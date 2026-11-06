# Row filters fail on columns stored as dictionary types in the embedded Arrow schema

- Issue: https://github.com/apache/iceberg-go/issues/2425 (opened by @hcrosse, open, no assignee, no comments)
- Status check (2026-11-06): `gh pr list --state all --search "2425"` → nothing. `--search "dictionary"` → the reporter's related PRs:
  - #2427 `fix(table): match equality deletes on dictionary columns` (merged 2026-11-04, closes #2426). Touches only `table/equality_delete_reader.go` (`makeColEncoder` dictionary case). Equality-delete key encoding, not row filters. Its helper encodes per-row keys, so there is nothing to reuse for the filter path.
  - #2442 `perf(table): match single-column equality deletes per dictionary entry` (open). Only `table/equality_delete_reader.go` + a test. Performance, equality deletes only.
  - #2445 `perf(table): keep dictionary entry results across batches` (draft, stacked on #2442). Same files, same scope.
  - None of them touch `filterRecords` / the row-filter path. **No overlap, this issue is not fixed or claimed.** The issue text says "I have not tested either change", and the reporter did not say they would fix it. Because the reporter is active on dictionary work, comment before opening a PR.
- Verified against main `2e69b8a` (2026-11-06)

## 재현 요약

이슈 본문의 PyIceberg 재현은 실행하지 않았습니다. 대신 같은 코드 경로를 두 가지로 확인했습니다.

1. 패키지 단위 최소 재현 (`table/transaction_internal_test.go`, `TestPrepareBatchFilterOnDictionaryColumn`): `dictionary<int8, utf8>` 컬럼이 있는 batch 에 `s = "a"` 필터를 `prepareBatchFilter` 로 적용합니다.

```text
--- FAIL: TestPrepareBatchFilterOnDictionaryColumn (0.00s)
    transaction_internal_test.go:85:
        	Error Trace:	.../table/transaction_internal_test.go:85
        	Error:      	Received unexpected error:
        	            	invalid: referenced field %!s(<nil>) was dictionary<values=utf8, indices=int8, ordered=false>, but should have been utf8
        	Test:       	TestPrepareBatchFilterOnDictionaryColumn
FAIL
FAIL	github.com/apache/iceberg-go/table	0.922s
```

(main 에서는 이 뒤에 `checked_allocator.go: LEAK of 64 bytes` 줄이 몇 개 더 나옵니다. arrow-go 의 에러 경로가 참조한 컬럼 datum 을 release 하지 않아서이고, 수정 후에는 에러가 나지 않으므로 사라집니다.)

2. end-to-end (커밋하지 않은 임시 테스트): `pqarrow.WithStoreSchema()` 로 dictionary 컬럼이 있는 Parquet 파일을 쓰고, `AddDataFilesByPath` 후 `Scan(WithRowFilter(EqualTo(s, "a"))).ToArrowTable` 을 실행했습니다. main 에서는 이슈와 같은 에러, 수정 후에는 `rows=2` 이고 결과 컬럼 타입은 `utf8` 입니다.

## 원인 (2e69b8a 기준)

- `table/arrow_scanner.go:1651-1667` `filterRecords` 가 파일에서 읽은 batch 를 그대로 `exprs.ExecuteScalarExpression(ctx, rec.Schema(), ...)` (`:1656`) 에 넘깁니다.
- 필터는 projection 전에 걸립니다. `table/arrow_scanner.go:2839-2840` 에서 pipeline 에 추가되고, `ToRequestedSchema`(dictionary 를 풀어 주는 곳)는 `:2859` 에서 그 뒤에 실행됩니다.
- substrait 식은 iceberg 스키마 기준이라 `s` 를 `utf8` 로 기대합니다(`table/arrow_filter_plan.go:396`, `table/transaction.go:2764` 의 `substrait.ConvertExpr`).
- arrow-go v18.8.0 `arrow/compute/exprs/exec.go:525-536` 가 필드 참조 타입을 비교합니다. `large_utf8`/`large_binary` 만 예외로 받고 dictionary 는 거부합니다.
- `filterRecords` 를 부르는 곳은 두 군데입니다: `table/arrow_filter_plan.go:63` (scan) 와 `table/transaction.go:2775` (`prepareBatchFilter`, `transaction_delete.go:589` 에서 사용). 둘 다 이 함수를 거치므로 여기 한 곳만 고치면 됩니다.

## Claim 댓글 초안 (게시하지 않음)

```text
I'd like to work on this. I reproduced it on main (2e69b8a) with a small test
that runs a string filter over a batch whose column is dictionary<int8, utf8>,
and with a scan over a Parquet file written with the stored Arrow schema.

My plan is to cast top-level dictionary columns to their value type only for
evaluating the predicate in filterRecords, then filter the original batch, so
the output schema does not change and projection still decodes as before.
This doesn't overlap with #2442/#2445, which only touch equality deletes.
@hcrosse, let me know if you were already planning to fix this yourself.
```
