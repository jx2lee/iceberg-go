# iceberg-go 기여 후보 (2026-11-06 갱신, main `2e69b8a`)

처음 정리한 것은 2026-10-24(main `7768540`)이고, 그 사이 upstream 에 커밋이 131개 들어와서 다시 확인했습니다.

- **A. 코드에서 직접 찾은 결함**: upstream 에 issue 가 없습니다. 최신 main 에서 실패하는 Go 테스트로 재현했고, 수정을 적용하면 그 테스트와 해당 패키지 전체 테스트가 통과하는 것까지 확인했습니다.
- **B. 기존 open issue 재현**: 링크된 PR 이 없는 bug issue 4건을 재현하고 폴더를 만들었습니다.
- **C. 지금 기여할 수 있는 open issue**: 쉬운 순으로 정리했습니다.

각 폴더에는 세 파일이 있습니다.
- `issue.md`: 올릴 issue 초안과 재현 결과
- `pr.md`: 브랜치·제목·본문 초안
- `guide.md`: 수정 방법(Go 초심자용)

재현 테스트 코드는 `guide.md` 안에 있습니다. 수정 코드는 이 브랜치에 커밋하지 않았고 `guide.md` 의 코드 블록으로만 남겼습니다.

## 2026-11-06 갱신에서 바뀐 것

- **빠짐**: `update-schema-rename-mutates-path`(이전 1번). 다른 기여자가 #2333(2026-10-26 머지)으로 같은 수정을 넣었고, 테스트 이름(`TestRenameColumnDoesNotMutatePath`)까지 같습니다.
- **남은 10건은 모두 최신 main 에서 그대로 재현**되고, `guide.md` 의 수정도 그대로 적용됩니다. 각 문서의 줄 번호를 `2e69b8a` 기준으로 고쳤습니다.
- `string-to-decimal-rounding`: `iceberg.DecimalTypeOf` 가 `(type, error)` 를 반환하도록 바뀌어서 테스트 코드를 고쳤습니다.
- `rest-update-view-commit-state-unknown`: #2350 이 `CommitTable` 에 4xx → `ErrCommitRejected` 매핑을 추가했다는 점을 guide 에 적었습니다.
- 기존 issue 4건(#2425, #701, #2407, #2452)을 재현해서 `issue-<번호>-...` 폴더로 추가했습니다(B).

## A. 코드에서 찾은 결함 (모두 issue 없음, 2026-11-06 기준 중복 PR 없음)

| # | 폴더 | 영역 | 난이도 | 재현 | 비고 |
|---|---|---|---|---|---|
| 1 | `rest-rename-view-missing-namespace` | catalog/rest | ★ | httptest 로 실패 확인 | `RenameTable` 은 #2173 에서 이미 고침. 같은 수정을 view 에 적용. **첫 번째 추천** |
| 2 | `rest-update-view-commit-state-unknown` | catalog/rest | ★ | httptest 로 실패 확인 | `CommitTable` 의 5xx 매핑을 그대로 옮기면 됨. 1번과 묶어서 PR 하나로 내도 됨 |
| 3 | `strict-notin-nan-upper-bound` | table/evaluators | ★ | 단위 테스트 실패 | NaN guard 1개 누락(`VisitNotEqual` 에는 있음). 최근 NaN 관련 PR(#2355, #2418, #2432)도 이 부분은 건드리지 않음 |
| 4 | `update-schema-case-insensitive-rename-identifier` | table/UpdateSchema | ★★ | 단위 테스트 실패 | #2333 이 고친 함수와 같은 곳 |
| 5 | `view-builder-readd-existing-version` | view | ★★ | 단위 테스트 2개 실패 | Java `addVersionInternal` 과 동작이 다름 |
| 6 | `update-spec-rename-onto-void-field` | table/UpdateSpec | ★★ | 단위 테스트 실패 | 엉뚱한 필드의 이름이 바뀜(에러 없음) |
| 7 | `string-to-timestamp-ns-overflow` | literals | ★★ | 단위 테스트 실패 | #1533 이 놓친 string 경로 |
| 8 | `snapshot-summary-fabricated-totals` | table/snapshots | ★★ | 단위 테스트 2개 실패 | Java·PyIceberg 와 동작이 다름. #2352 가 같은 파일을 고쳤지만 이 로직은 그대로 |
| 9 | `string-to-decimal-rounding` | literals | ★★★ | 단위 테스트 실패 | Java 보다 느슨하게 고칠지 정해야 함 → **issue 에 방향 질문 먼저** |
| 10 | `v1-legacy-partition-field-ids` | table/metadata | ★★★★ | 단위 테스트 실패 | #1514 의 설계 의도와 부딪힘 → **issue 에 방향 질문 먼저** |

검증(2026-11-06, main `2e69b8a`):
- 각 `guide.md` 의 테스트 코드를 그대로 붙였을 때 `go vet` 통과. main 에서는 새 테스트 12개가 모두 FAIL.
- 각 `guide.md` 의 수정 코드를 모두 적용하면 위 테스트가 통과하고, `go test . ./table/... ./view/... ./catalog/...` 도 전부 통과.
- `make lint` 는 돌리지 않았으니 PR 전에 실행하세요.

## B. 기존 open issue 재현 (2026-11-06, main `2e69b8a`)

issue 가 이미 있으니 `issue.md` 에는 claim 댓글 초안(올리지 않음)과 재현 결과를 넣었습니다. 네 건 모두 직접 다시 돌려서 수정 전에는 FAIL, 수정 후에는 PASS 인 것을 확인했고, 네 수정을 함께 적용한 상태에서 `go test . ./table/... ./catalog/...` 도 전부 통과합니다.

| 폴더 | 이슈 | 난이도 | 재현 | 지금 가져갈 수 있나 |
|---|---|---|---|---|
| `issue-2425-row-filter-dictionary-column` | #2425 dictionary 컬럼에 row filter 를 걸면 실패 | ★★ | 단위 테스트 실패(`...was dictionary<values=utf8...>, but should have been utf8`) | **가능.** 댓글·담당자·PR 없음. 작성자(hcrosse)의 #2427·#2442·#2445 는 equality delete 쪽만 고쳐서 겹치지 않음. 작성자가 이 영역을 계속 작업 중이라 claim 댓글부터 |
| `issue-701-glue-large-schema-payload` | #701 Glue 에서 큰 schema 일 때 payload 한도 초과 | ★★ | 가짜 HTTP 클라이언트로 요청 크기를 쟀을 때 컬럼 3600개 = 546,378B > 400KB | **방향 질문 먼저.** Java·PyIceberg 도 같은 동작. 새 property(`glue.schema-columns`)를 둘지는 #769 리뷰에서 "dev@ 에서 논의하라"는 답을 받고 멈춘 상태. 400KB 는 공식 문서가 아니라 사용자 보고값 |
| `issue-2407-empty-string-partition-bound` | #2407 빈 문자열 partition 값 때문에 manifest 를 통째로 건너뜀 | ★ | 단위 테스트 실패(`empty lower bound written as null`) | **어려움.** 작성자가 본문에 "fix 와 regression test 가 준비돼 있고 직접 하겠다"고 적음. 1~2주 PR 이 없을 때만 물어보기. 공부용으로는 가장 좋음 |
| `issue-2452-update-schema-last-column-id-panic` | #2452 `last-column-id` 가 기존 field ID 보다 작으면 UpdateSchema 가 panic | ★ | 단위 테스트 panic | **불가.** 작성자가 같은 날 PR #2453 을 올림(open). 리뷰로만 참여 가능 |

## C. 지금 기여할 수 있는 open issue, 쉬운 순 (2026-11-06)

서브에이전트가 open issue 64건을 전부 보고, 고칠 코드 위치까지 확인해서 정리했습니다. 난이도 1 은 함수 하나에서 몇 줄 고치는 수준이고, 5 는 여러 패키지에 걸친 설계 작업입니다. B 에서 직접 재현한 것 말고는 제가 다시 확인하지 않았으니, 착수 전에 issue 를 다시 보세요.

| 순위 | 이슈 | 종류 | 난이도 | 고칠 곳 | 주의 |
|---|---|---|---|---|---|
| 1 | #1535 CI 에서 docker pull 재시도 | ci | 1 | `.github/workflows/go-integration.yml`, Makefile `integration-setup*` | 08-02 claim 이후 PR 없음(stale). CI 와 Makefile 중 어디에 넣을지 issue 가 묻고 있음 |
| 2 | #1015 `CommitNumRetriesDefault` 를 0 → 4 로 | feature | 1 | `table/properties.go:189` | 기본값 정책이라 메인테이너 판단 필요 |
| 3 | #2407 빈 문자열 partition bound | bug | 1 | `manifest.go` `toSummary` | B 참고. 작성자가 사실상 claim |
| 4 | #2425 dictionary 컬럼 row filter | bug | 2 | `table/arrow_scanner.go` `filterRecords` | B 참고. **지금 가장 추천** |
| 5 | #701 Glue 큰 schema | bug | 2 | `catalog/glue/glue.go` `constructTableInput` | B 참고. 설계 결정 필요 |
| 6 | #1086 v1/v2 metadata 의 v3 전용 필드 허용 | feature | 2 | `table/metadata.go` `versionScopedField` | dev@ 논의 대기 |
| 7 | #2371 changelog 파일 병렬 읽기 | perf | 3 | `table/changelog_reader.go` `Read` | 순서 유지와 limit 처리가 까다로움 |
| 8 | #2443 REST Idempotency-Key 지원 | feature | 3 | `catalog/rest/rest.go` `do`/`doPost` | `scan_planning.go` 의 기존 helper 재사용 가능. 5xx 재시도 범위 주의, 스펙 지식 필요 |
| 9 | #1097 Hadoop `RenameTable` | feature | 3 | `catalog/hadoop` | Java 는 일부러 구현하지 않음. 설계 결정 필요 |
| 10 | #1792 REST `snapshots=refs` 일 때 나머지 snapshot lazy-load | feature | 3 | `catalog/rest` `LoadTable` + table | 작성자가 08-26 "working on this" 이후 PR 없음(stale) |
| 11 | #2370 changelog planning 이 manifest 를 너무 많이 읽음 | perf | 4 | `table/incremental_changelog_deletes.go` | delete 적용 규칙을 알아야 하고 정확성 위험이 있음 |
| 12 | #2410 scan 결과에서 Parquet dictionary 인코딩 유지 | feature | 4 | reader·cast·filter 경로 전반 | 공개 API 설계 필요. hcrosse 가 작업 중인 영역 |
| 13 | #2344 정렬 compaction Phase 2(spill to disk) | feature | 4 | `table/sort_compaction.go` | Phase 1 작성자가 이어서 할 가능성이 큼 |
| 14 | #1262 Windows CI | ci | 5 | 여러 패키지 | 성격상 tracking 이라 쪼개서 진행해야 함 |

### close 요청만 하면 되는 것

| 이슈 | 근거 |
|---|---|
| #2063 TruncateTransform UTF-8 | #1198·#1223 으로 v0.7.0 에서 해결 |
| #2131 `NOT(x<5)` 와 NaN | #2355("Closes #2131")가 11-04 에 머지됐는데 자동으로 닫히지 않음 |
| #1868 DV supersession 검사 | #2054(10-14)가 모든 commit 에서 검사 |
| #2360 NaN value count | #2420(11-04)이 `nan_value_counts` 기록을 시작. double 도 포함하는지는 불확실 |
| #1793 SQLITE_BUSY flaky | #2036(10-01)이 재시도 횟수를 늘림. 그 뒤 재발했는지는 불확실 |
| #2428 OAuth2 manager export | #2433(11-05)으로 1번 제안은 끝남. 남은 GoogleAuthManager 는 별도 issue 로 나누면 됨 |

### 제외

| 이유 | 이슈 |
|---|---|
| open PR 있음 | #2452(#2453), #2446, #2388, #2339, #2040, #1929, #1685·#1687(#1688, 08-08 이후 멈춤), #1119, #1215·#1216 |
| 담당자 있음 | #2083, #2082, #2062, #1433, #1178, #1079, #989 |
| 최근 6주 안에 claim | #2406, #2393, #2311, #2195, #2160, #2152, #2416(arrow-go 릴리스 대기) |
| tracking·epic·질문 | #2411–#2415, #1691, #1289, #485, #183, #4 |
| 외부 요인 대기 | #2043(Java 쪽 안정화), #1880(Docker 이미지) |
| 설계 보류·우선순위 낮음 | #2154, #2044, #1149, #696, #399 |
| 버그 아님·재현 불가 | #538, #860 |
| 이전 목록에서 닫힌 것 | #1847, #2132, #2215, #2181 |

## 검토했지만 제외한 후보 (코드 리뷰 중)

- REST 이름의 `/`·`%` escape: #2074·#2104 로 해결됨.
- REST `LoadTable`/`DropTable` 이 404 `NoSuchNamespaceException` 을 `ErrNoSuchTable` 로 매핑: SQL catalog 도 같아서 의도된 동작으로 봄.
- SQL catalog 의 namespace prefix 매칭(`a` vs `ab`): escape 와 재필터링이 되어 있어 문제없음.
- v2/v3 에서 partition field ID 재사용, 참조되지 않은 snapshot 을 ExpireSnapshots 가 지우는 문제: 이전 커밋에서는 재현됐지만 `7768540` 에서는 이미 고쳐짐.
- `RenameColumn` 이 호출자의 path slice 를 바꾸는 문제(이전 1번): #2333 으로 해결.
- year/month transform 빠른 경로(#2263): ±80만 일 범위를 전수 비교했고 차이 없음.
- bucket hash, truncate(음수·decimal·UTF-8): 스펙과 일치.
- `ApplyNameMapping` 에서 `field-id` 가 없을 때의 nil deref: recover 되어 에러로 바뀜. Java 도 NPE 를 냄.

## 공통 주의

- **upstream 에 링크 걸지 않기.** 이 브랜치는 `origin`(jx2lee/iceberg-go) 에만 push 합니다. 커밋 메시지에 `apache/iceberg-go#123` 을 쓰면 upstream issue 타임라인에 "referenced" 가 남으니 쓰지 않습니다. 문서 본문의 링크는 괜찮습니다.
- CONTRIBUTING.md:
  - issue 에 "I'd like to work on this" 댓글을 달고 메인테이너가 확인해 주면 시작합니다.
  - A 그룹은 issue 가 없으니 `issue.md` 로 issue 를 먼저 올립니다.
  - **PR 은 한 번에 하나씩.** 이미 머지된 PR 이 있지만 리뷰 부담을 줄이려면 순서대로 내는 게 좋습니다.
- 커밋은 `git commit -s`(DCO). 제목은 `fix(table): ...` 처럼 소문자 conventional commits 로 씁니다.
- 체크: `make lint`, `go test ./<pkg>/...`. 공개 API 가 바뀌면 `dev/check-api-compat.sh` 도 돌립니다(이번 10건은 모두 비공개 함수 수정이라 해당 없음).
- 테스트는 기존 `_test.go` 파일 중 맞는 곳에 넣습니다. 새 파일을 만들면 Apache 라이선스 헤더를 붙여야 합니다.

## Go 빠른 참고 (가이드 전체에서 쓰는 것)

| 개념 | 한 줄 설명 |
|---|---|
| `go test -run TestName ./table/` | 패키지 하나에서 이름이 정규식에 맞는 테스트만 실행. `-count=1` 을 붙이면 캐시를 쓰지 않음 |
| `package table` vs `package table_test` | 같은 폴더의 테스트 파일. 앞쪽은 비공개(소문자) 함수에 접근할 수 있고, 뒤쪽은 외부 사용자처럼 공개 API 만 씀 |
| `require.X` / `assert.X` (testify) | `require` 는 실패하면 즉시 테스트를 중단하고, `assert` 는 기록만 하고 계속 진행 |
| `errors.Is(err, ErrX)` | `fmt.Errorf("%w", ...)` 로 감싼 에러 사슬 안에 `ErrX` 가 있는지 확인 |
| slice `s[a:b]` | 새 배열이 아니라 같은 배열의 "창". 여기에 `append` 하면 원본이 바뀔 수 있음(#2333 참고) |
| map 조회 `v, ok := m[k]` | 키가 없으면 `ok == false`, `v` 는 0 값 |
