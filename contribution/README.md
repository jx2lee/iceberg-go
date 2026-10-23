# iceberg-go 기여 후보 (2026-10-24, main `7768540`)

두 경로로 찾았습니다.

- **A. 코드에서 직접 찾은 결함**: upstream 에 issue 가 없습니다. main 에서 실패하는 Go 테스트로 재현했고, 수정을 적용하면 그 테스트와 해당 패키지 전체 테스트가 통과하는 것까지 확인했습니다.
- **B. open issue 중 링크된 PR 이 없는 것**: 거의 다 이미 담당자가 있거나, main 에서 이미 고쳐졌거나, 메인테이너가 방향을 정해야 하는 상태였습니다(아래 표).

각 폴더에는 세 파일이 있습니다.
- `issue.md`: 올릴 issue 초안과 재현 결과
- `pr.md`: 브랜치·제목·본문 초안
- `guide.md`: 수정 방법(Go 초심자용)

재현 테스트 코드는 `guide.md` 안에 있습니다. 수정 코드는 이 브랜치에 커밋하지 않았고 `guide.md` 의 코드 블록으로만 남겼습니다.

## A. 코드에서 찾은 결함 (모두 issue 없음, 2026-10-24 기준 중복 PR 없음)

| # | 폴더 | 영역 | 난이도 | 재현 | 비고 |
|---|---|---|---|---|---|
| 1 | `update-schema-rename-mutates-path` | table/UpdateSchema | ★ | 단위 테스트 2개 실패 | Go slice aliasing. 1줄 수정. **Go 공부용으로 첫 번째 추천** |
| 2 | `rest-rename-view-missing-namespace` | catalog/rest | ★ | httptest 로 실패 확인 | `RenameTable` 은 #2173 에서 이미 고침. 같은 수정을 view 에 적용 |
| 3 | `rest-update-view-commit-state-unknown` | catalog/rest | ★ | httptest 로 실패 확인 | `CommitTable` 의 5xx 매핑을 그대로 옮기면 됨. 2번과 묶어서 PR 하나로 내도 됨 |
| 4 | `strict-notin-nan-upper-bound` | table/evaluators | ★ | 단위 테스트 실패 | NaN guard 1개 누락(`VisitNotEqual` 에는 있음) |
| 5 | `update-schema-case-insensitive-rename-identifier` | table/UpdateSchema | ★★ | 단위 테스트 실패 | 1번과 같은 함수. PR 은 따로 |
| 6 | `view-builder-readd-existing-version` | view | ★★ | 단위 테스트 2개 실패 | Java `addVersionInternal` 과 동작이 다름 |
| 7 | `update-spec-rename-onto-void-field` | table/UpdateSpec | ★★ | 단위 테스트 실패 | 엉뚱한 필드의 이름이 바뀜(에러 없음) |
| 8 | `string-to-timestamp-ns-overflow` | literals | ★★ | 단위 테스트 실패 | #1533 이 놓친 string 경로 |
| 9 | `snapshot-summary-fabricated-totals` | table/snapshots | ★★ | 단위 테스트 2개 실패 | Java·PyIceberg 와 동작이 다름 |
| 10 | `string-to-decimal-rounding` | literals | ★★★ | 단위 테스트 실패 | Java 보다 느슨하게 고칠지 정해야 함 → **issue 에 방향 질문 먼저** |
| 11 | `v1-legacy-partition-field-ids` | table/metadata | ★★★★ | 단위 테스트 실패 | #1514 의 설계 의도와 부딪힘 → **issue 에 방향 질문 먼저** |

검증(2026-10-24, main `7768540`):
- 각 `guide.md` 의 테스트 코드를 그대로 대상 `_test.go` 에 붙였을 때 `go vet` 통과. main 에서는 새 테스트 14개가 모두 FAIL.
- 각 `guide.md` 의 수정 코드를 그대로 적용하면 위 테스트가 모두 통과하고, `go test . ./table/... ./view/... ./catalog/...` 도 전부 통과.
- `make lint` 는 돌리지 않았으니 PR 전에 실행하세요.

## B. open issue 중 PR 이 링크되지 않은 것 (2026-10-24 확인)

| 이슈 | 상태 | 할 수 있는 것 |
|---|---|---|
| #1847 manifest partition logical-type panic | main 에서 이미 해결(`8d9b813`, #1861). issue 와 연결이 안 돼 있음 | "fixed by #1861" 댓글로 close 요청 |
| #2063 TruncateTransform UTF-8 | #1198·#1223 으로 이미 해결. 댓글로도 확인됨 | close 요청 |
| #2131 `NOT(x<5)` 와 NaN | PR 없음. Revanth14 가 (a)/(b) 안을 내고 메인테이너 답을 기다리는 중 | 방향이 정해진 뒤에 진행. Java 도 같은 문제가 있어서 설계 결정이 필요함 |
| #2132 `NotIn` 과 NULL | #2131 과 같은 상황 | 같음 |
| #2082 CLI `--output` | eneskeles 가 담당 중(assignee) | 제외 |
| #2215 DV summary | akritishakya 가 담당(2026-10-20) | 제외 |
| #2195 DV replay | 작성자(badalprasadsingh)가 방향까지 적어 둠 | 제외 |
| #1685, #1687 UpdateSchema nested | #1688 이 open | 제외 |
| #1792, #2152, #2154, #1929, #2043, #1433 | 기능 요청이나 제안 | 제외(결함 아님) |
| #701 Glue payload 크기 | 실제 Glue 환경 필요. PR #769 는 닫힘 | 제외 |
| #2181 vended credential | 설계 논의 중 | 제외 |

## 검토했지만 제외한 후보 (코드 리뷰 중)

- REST 이름의 `/`·`%` escape: #2074·#2104 로 해결됨.
- REST `LoadTable`/`DropTable` 이 404 `NoSuchNamespaceException` 을 `ErrNoSuchTable` 로 매핑: SQL catalog 도 같아서 의도된 동작으로 봄.
- SQL catalog 의 namespace prefix 매칭(`a` vs `ab`): escape 와 재필터링이 되어 있어 문제없음.
- v2/v3 에서 partition field ID 재사용, 참조되지 않은 snapshot 을 ExpireSnapshots 가 지우는 문제: 이전 커밋에서는 재현됐지만 `7768540` 에서는 이미 고쳐짐.
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
- 체크: `make lint`, `go test ./<pkg>/...`. 공개 API 가 바뀌면 `dev/check-api-compat.sh` 도 돌립니다(이번 11건은 모두 비공개 함수 수정이라 해당 없음).
- 테스트는 기존 `_test.go` 파일 중 맞는 곳에 넣습니다. 새 파일을 만들면 Apache 라이선스 헤더를 붙여야 합니다.

## Go 빠른 참고 (가이드 전체에서 쓰는 것)

| 개념 | 한 줄 설명 |
|---|---|
| `go test -run TestName ./table/` | 패키지 하나에서 이름이 정규식에 맞는 테스트만 실행. `-count=1` 을 붙이면 캐시를 쓰지 않음 |
| `package table` vs `package table_test` | 같은 폴더의 테스트 파일. 앞쪽은 비공개(소문자) 함수에 접근할 수 있고, 뒤쪽은 외부 사용자처럼 공개 API 만 씀 |
| `require.X` / `assert.X` (testify) | `require` 는 실패하면 즉시 테스트를 중단하고, `assert` 는 기록만 하고 계속 진행 |
| `errors.Is(err, ErrX)` | `fmt.Errorf("%w", ...)` 로 감싼 에러 사슬 안에 `ErrX` 가 있는지 확인 |
| slice `s[a:b]` | 새 배열이 아니라 같은 배열의 "창". 여기에 `append` 하면 원본이 바뀔 수 있음(1번 가이드) |
| map 조회 `v, ok := m[k]` | 키가 없으면 `ok == false`, `v` 는 0 값 |
