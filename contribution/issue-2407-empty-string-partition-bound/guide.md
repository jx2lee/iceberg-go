# 수정 가이드: 빈 문자열 파티션 값이 manifest 를 통째로 건너뛰게 함

난이도: ★☆☆☆☆ (nil 검사 두 줄. 원인 추적이 더 중요)

> 먼저 볼 것: 이슈 작성자(@PDGGK)가 "수정과 테스트가 준비돼 있다"고 이미 밝혔습니다. 직접 PR 을 올리기 전에 issue.md 의 상태 확인과 댓글 초안 설명을 읽어 주세요.

## 1. 문제를 한 줄로

String 파티션의 최솟값이 `""` 이면 manifest list 에 `lower_bound` 가 null 로 저장되고, 평가기가 이를 "값이 전부 null" 로 읽어서 그 manifest 의 모든 행을 scan 결과에서 빠뜨립니다.

## 2. 배경: partition summary

manifest list 는 manifest 파일마다 파티션 필드별 요약(`FieldSummary`)을 갖습니다.

| 필드 | 뜻 |
|---|---|
| `contains_null` | null 파티션 값이 있나 |
| `lower_bound` / `upper_bound` | null 이 아닌 값 중 최소/최대 (바이트로 직렬화) |

`lower_bound` 가 null 이면 "null 이 아닌 값이 하나도 없다"는 뜻입니다. 그래서 `region = 'us'` 같은 조건은 그 manifest 에서 절대 맞을 수 없다고 판단하고 건너뜁니다. 실제로는 `""` 와 `"us"` 가 있는데도요.

## 3. 코드 읽기 (main `2e69b8a`)

| 파일 | 줄 | 무엇 |
|---|---|---|
| `literals.go` | 1079-1085 | `StringLiteral.MarshalBinary`: 문자열 바이트를 복사 없이 slice 로 반환. `""` 이면 nil |
| `manifest.go` | 70-73 | `FieldSummary`: `LowerBound *[]byte` 는 Avro `["null","bytes"]` union |
| `manifest.go` | 1609-1633 | `partitionFieldStats.toSummary`: min/max 를 바이트로 바꿔 포인터 저장 (수정 위치) |
| `manifest.go` | 1718, 1878 | `summaries()` → `ManifestWriter.ToManifestFile`: `toSummary` 의 유일한 호출 경로 |
| `manifest.go` | 309 | `ensurePartitionList`: 같은 인코더 동작(#1309)을 다른 필드에서 고친 선례 |
| `table/evaluators.go` | 389-418 | `manifestEvalVisitor.VisitEqual`: `LowerBound == nil` 이면 `rowsCannotMatch` |
| `manifest_test.go` | 515 | `TestPartitionFieldStatsAcceptsConvertibleValues`: `newPartitionFieldStat` 사용 예 |

Go 개념 메모:
- **nil slice 와 빈 slice**: `var b []byte` 는 nil, `[]byte{}` 는 nil 이 아닌 길이 0 slice 입니다. `len` 은 둘 다 0 이고 대부분의 코드에서 똑같이 동작하지만, `b == nil` 로 구분됩니다. twmb/avro 는 이 차이를 보고 nil 이면 union 의 null 쪽을 씁니다.
- **`unsafe.StringData("")`**: 빈 문자열의 데이터 포인터는 nil 일 수 있고, `unsafe.Slice(nil, 0)` 은 nil slice 를 돌려줍니다.
- **`*[]byte`**: Avro optional 필드를 Go 에서 포인터로 표현합니다. 포인터가 nil 이면 null, 아니면 값. 그런데 포인터가 nil slice 를 가리키는 경우도 인코더가 null 로 씁니다. 이번 버그의 핵심입니다.

## 4. 수정

`manifest.go` `partitionFieldStats.toSummary`

```diff
 	if p.min != nil {
 		lit = NewLiteral(*p.min)
 		lb, _ := lit.MarshalBinary()
+		// "" marshals to a nil slice, and the encoder writes a pointer to a
+		// nil slice as null. A null bound means "all values are null", so
+		// keep a present bound non-nil.
+		if lb == nil {
+			lb = []byte{}
+		}
 		lowerBound = &lb
 	}
 
 	if p.max != nil {
 		lit = NewLiteral(*p.max)
 		ub, _ := lit.MarshalBinary()
+		if ub == nil {
+			ub = []byte{}
+		}
 		upperBound = &ub
 	}
```

왜 `StringLiteral.MarshalBinary` 를 고치지 않나: `MarshalBinary` 는 data file 통계, 표현식 직렬화 등 여러 곳에서 쓰입니다. 그쪽은 map 값이나 일반 bytes 필드라 nil 이어도 길이 0 으로 잘 써집니다. 문제는 "optional bytes 에 포인터로 넣는" 곳 하나뿐이라서 그 자리에서 고치는 게 범위가 가장 좁습니다. Java 의 `PartitionSummary.toSummary` 도 min 이 있으면 길이 0 ByteBuffer 를 넣습니다.

## 5. 테스트

`manifest_test.go`(`package iceberg`) 끝에 추가합니다. 필요한 import(`bytes`, `assert`, `require`)는 이미 있습니다.

```go
func TestManifestListKeepsEmptyPartitionBound(t *testing.T) {
	for _, typ := range []PrimitiveType{PrimitiveTypes.String, PrimitiveTypes.Binary} {
		t.Run(typ.String(), func(t *testing.T) {
			stats, err := newPartitionFieldStat(typ)
			require.NoError(t, err)
			require.NoError(t, stats.update(""))
			require.NoError(t, stats.update("us"))

			seqNum := int64(1)
			list := []ManifestFile{NewManifestBuilder(2, "s3://bucket/m0.avro", 100, 0, 1).
				Partitions([]FieldSummary{stats.toSummary()}).AddedFiles(1).Build()}

			// write, read back, then rewrite the inherited entry and read again
			for i := range 2 {
				var buf bytes.Buffer
				require.NoError(t, WriteManifestList(2, &buf, 1, nil, &seqNum, 0, list))
				list, err = ReadManifestList(&buf)
				require.NoError(t, err)

				s := list[0].Partitions()[0]
				require.NotNilf(t, s.LowerBound, "pass %d: empty lower bound written as null", i)
				assert.Empty(t, *s.LowerBound)
				assert.Equal(t, []byte("us"), *s.UpperBound)
			}
		})
	}
}
```

두 번째 pass 는 "읽어 온 manifest 를 다음 커밋에서 그대로 다시 쓰는" 경로를 확인합니다. `binary` 는 main 에서도 통과하지만 같은 문제가 다른 타입으로 번지지 않는지 지키는 용도입니다.

## 6. 실행

```bash
go test -count=1 -run TestManifestListKeepsEmptyPartitionBound -v .
go vet . ./table/
go test ./ ./table/...
make lint
git commit -s -m "fix(manifest): write empty string partition bounds as zero-length bytes"
```

## 7. 수정 전/후 결과 (2026-11-06, main `2e69b8a`)

수정 전:

```text
--- FAIL: TestManifestListKeepsEmptyPartitionBound (0.00s)
    --- FAIL: TestManifestListKeepsEmptyPartitionBound/string (0.00s)
        manifest_test.go:5050:
            	Error:      	Expected value not to be nil.
            	Messages:   	pass 0: empty lower bound written as null
FAIL	github.com/apache/iceberg-go	0.549s
```

수정 후:

```text
--- PASS: TestManifestListKeepsEmptyPartitionBound (0.00s)
    --- PASS: TestManifestListKeepsEmptyPartitionBound/string (0.00s)
    --- PASS: TestManifestListKeepsEmptyPartitionBound/binary (0.00s)
ok  	github.com/apache/iceberg-go	0.642s
```

이슈 본문의 end-to-end scan 도 `region = "us"`, `region = ""` 각각 0행에서 1행으로 바뀝니다. `go test ./ ./table/...` 와 `go vet . ./table/` 모두 통과했습니다.

## 8. 주의

- 이 이슈는 리포터가 직접 고치겠다고 했습니다. 먼저 PR 을 올리면 예의에 어긋나니, 리포터 PR 이 올라오면 리뷰로 참여하는 게 좋습니다.
- 이미 null 로 잘못 기록된 기존 테이블은 이 수정으로 고쳐지지 않습니다. 새로 쓰는 manifest 부터 맞게 기록됩니다. 리뷰어가 물으면 "기존 manifest 는 `RewriteManifests` 처럼 manifest 를 새로 쓰는 작업을 거쳐야 바로잡힌다"고 답하면 됩니다.
- 리뷰어가 `MarshalBinary` 쪽에서 고치자고 하면 4절의 이유(호출처가 많고 문제는 optional 포인터 한 곳)를 설명하면 됩니다.
