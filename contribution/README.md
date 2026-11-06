# iceberg-go 기여 후보 (2026-11-06 갱신, main `2e69b8a`)

처음 정리한 것은 2026-10-24(main `7768540`)이고, 그 사이 upstream 에 커밋이 131개 들어와서 다시 확인했습니다.

- **A. 코드에서 직접 찾은 결함**: upstream 에 issue 가 없습니다. 최신 main 에서 실패하는 Go 테스트로 재현했고, 수정을 적용하면 그 테스트와 해당 패키지 전체 테스트가 통과하는 것까지 확인했습니다.
- **B. open issue 중 링크된 PR 이 없는 것**: 2026-11-06 기준으로 새로 조사했습니다.

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

## B. open issue 중 PR 이 링크되지 않은 것 (2026-11-06 확인)

조사 시점 기준: open issue 64개, open PR 52개. 착수 전에 issue 에 다른 사람의 댓글이 새로 달렸는지 다시 보세요.

### 지금 잡을 수 있는 것

| 이슈 | 종류 | 상태 | 난이도 |
|---|---|---|---|
| #2452 UpdateSchema Apply/Commit 이 `last-column-id` < 기존 field ID 일 때 panic | bug | 2026-11-06 등록. 댓글·PR 없음. 작성자(Shruthirdy)가 직접 고칠 수도 있으니 먼저 물어보기 | 쉬움 |
| #2407 빈 문자열 partition 값 때문에 필터 scan 이 manifest 전체를 건너뜀 | bug | 2026-11-03 등록, 재현 포함. 댓글·PR 없음 | 보통 |
| #2425 embedded Arrow schema 에서 dictionary 타입 컬럼에 row filter 가 실패 | bug | 담당자 없음. #2427 은 equality delete 쪽만 고침. 작성자(hcrosse)가 근처 작업(#2442, #2445) 중이라 겹칠 수 있음 | 보통 |
| #701 Glue `updateTable` 이 큰 schema 에서 실패 | bug | 메인테이너가 원인(`constructTableInput` 이 과거 컬럼까지 직렬화)을 짚어 둠. 이전 draft #769 는 닫힘. 담당자 없음. 실제 Glue 없이 단위 테스트로 크기 검증 가능할지 확인 필요 | 보통 |
| #2371 changelog 파일을 순서를 지키며 병렬로 읽기 | perf | #2170 리뷰의 후속. 담당자 없음 | 보통 |
| #2370 snapshot 이 delete 를 추가할 때 changelog planning 이 모든 data manifest 를 읽음 | perf | #2169 리뷰의 후속. 담당자 없음 | 보통~어려움 |
| #1535 CI 에서 Docker pull 재시도 | ci | 2026-08-02 에 claim 됐지만 그 뒤 PR 없음. 먼저 물어보기 | 쉬움 |
| #1793 SQLITE_BUSY flaky test | test | claim 이 둘(8월, 10-23)인데 PR 없음. 메인테이너가 `busy_timeout` 방식을 제안해 둠. 아직 하는 사람이 있는지 불확실 | 쉬움 |

### close 요청만 하면 되는 것

| 이슈 | 상태 |
|---|---|
| #2063 TruncateTransform UTF-8 | open 이지만 #1198·#1223 으로 v0.7.0 에서 해결됨(댓글로 확인). close 요청 |
| #2131 `NOT(x<5)` 와 NaN | open 이지만 #2355(2026-11-04 머지)가 연결돼 있어 사실상 해결 |
| #1868 DV supersession 검사 | open 이지만 #2054(10-14 머지)가 모든 commit 에서 검사한다고 함. 해결 여부 확인 후 close 요청 |

### 제외

| 이슈 | 이유 |
|---|---|
| #1847, #2132, #2215, #2181 | 이전 목록에 있던 것. 각각 #1861, #2369, #2352, #2417 로 닫힘 |
| #2082 CLI `--output` | eneskeles 가 담당 중(9/30~, PR 없음). 다른 사람도 손을 든 상태 |
| #2195 DV replay | richiebachala 가 작성자 승인을 받고 진행 중 |
| #1685, #1687 UpdateSchema nested | draft PR #1688 이 08-08 이후 멈춰 있음. 하고 싶으면 badalprasadsingh 에게 먼저 물어보기 |
| #2393, #2406, #2311, #2160 | 이미 claim 됨 |
| #1119, #2388, #2446, #2339 | open PR 있음(#1128, #2389·#2390, #2447, #2354) |
| #2416 | arrow-go#1448 에 막혀 있음 |
| #860, #538 | 조사 난이도가 높거나 iceberg-go 문제가 아니라고 정리됨 |
| 나머지 | 기능 요청·tracking·epic 이거나 메인테이너 담당 |

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
