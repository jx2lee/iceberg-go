# Empty string partition value makes filtered scans skip the whole manifest

- Issue: https://github.com/apache/iceberg-go/issues/2407 (opened 2026-11-03 by @PDGGK, open, no assignee, no comments)
- Status check (2026-11-06): `gh pr list --state all --search "2407"`, `--author PDGGK`, "empty string partition", "empty string bound" → no PR for this issue. **But the reporter wrote at the end of the issue: "I have a fix and a regression test ready and would like to work on this."** So it is effectively claimed by the reporter.
- Verified against main `2e69b8a` (2026-11-06)

## 재현 요약

String identity 파티션 값 중 가장 작은 값이 `""` 이면 manifest list 의 `lower_bound` 가 null 로 기록됩니다(`contains_null=false` 와 함께). 스펙상 null lower bound 는 "모든 값이 null" 이라는 뜻이라서 manifest evaluator 가 그 필드에 대한 모든 조건에서 manifest 를 통째로 건너뜁니다.

이슈 본문의 end-to-end 테스트(hadoop catalog, `{"", "us"}` append 후 `region = v` scan)를 2e69b8a 에서 실행한 결과:

```text
repro_test.go:47: region = "us": 0 rows
repro_test.go:47: region = "": 0 rows
```

수정 후에는 둘 다 `1 rows`.

패키지 단위 최소 재현(`manifest_test.go`, `TestManifestListKeepsEmptyPartitionBound`):

```text
--- FAIL: TestManifestListKeepsEmptyPartitionBound (0.00s)
    --- FAIL: TestManifestListKeepsEmptyPartitionBound/string (0.00s)
        manifest_test.go:5050:
            	Error:      	Expected value not to be nil.
            	Test:       	TestManifestListKeepsEmptyPartitionBound/string
            	Messages:   	pass 0: empty lower bound written as null
FAIL
FAIL	github.com/apache/iceberg-go	0.549s
```

(`binary` 서브테스트는 main 에서도 통과합니다. `[]byte("")` 는 nil 이 아니라서입니다. 회귀 방지용으로 둡니다.)

## 원인 (2e69b8a 기준)

- `literals.go:1079-1085` `StringLiteral.MarshalBinary` 는 `unsafe.Slice(unsafe.StringData(s), len(s))` 를 돌려줍니다. `s == ""` 이면 nil slice 입니다.
- `manifest.go:1609-1633` `partitionFieldStats.toSummary` 가 그 결과의 포인터를 그대로 저장합니다(`:1618-1619` lower, `:1624-1625` upper).
- twmb/avro 는 `*[]byte` 가 nil slice 를 가리키면 `["null","bytes"]` union 의 null 로 씁니다. #1309 / #1464 (`ensurePartitionList`, `manifest.go:309`) 와 같은 인코더 동작입니다.
- 읽는 쪽 `table/evaluators.go:393` (`VisitEqual`), `:470`, `:490`, `:519` 등은 `LowerBound == nil` 이면 `rowsCannotMatch`. Java `ManifestEvaluator` 도 동일(`lowerBound() == null` → `ROWS_CANNOT_MATCH`).
- `toSummary` 호출 경로는 `partitionSummaryStats.summaries` (`manifest.go:1718`) → `ManifestWriter.ToManifestFile` (`manifest.go:1878`) 하나뿐입니다.
- Java `PartitionSummary.toSummary` 는 `min != null ? Conversions.toByteBuffer(type, min) : null` 이라서 `""` 이면 길이 0 ByteBuffer(null 아님)가 기록됩니다.
- 읽은 뒤 다시 쓰는 경로(상속 manifest)는 문제없습니다. 길이 0 bytes 를 읽으면 nil 이 아닌 빈 slice 가 되고 테스트의 두 번째 pass 가 이를 확인합니다.

## Claim 댓글 초안 (게시하지 말 것)

리포터가 이미 수정과 테스트를 갖고 있다고 밝혔으므로 지금 claim 하는 건 맞지 않습니다. 리포터 PR 이 1~2주 안 올라오지 않을 때만 아래처럼 남깁니다.

> Hi @PDGGK, are you still planning to open a PR for this? If you no longer have time, I'd like to work on this. I have a fix in `partitionFieldStats.toSummary` (keep a present bound as a zero-length slice instead of nil, matching Java's `PartitionSummary`) and a manifest-list round-trip test. Happy to review yours instead if it's coming.
