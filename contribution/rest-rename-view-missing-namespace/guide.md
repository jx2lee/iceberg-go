# 수정 가이드: REST RenameView 가 namespace 없음을 view 없음으로 보고함

난이도: ★☆☆☆☆ (이미 고쳐진 sibling 코드를 그대로 따라 쓰는 작업)

## 1. 문제를 한 줄로

`RenameTable` 은 404 응답의 `error.type` 을 보고 "table 없음"과 "namespace 없음"을 구분합니다(#2173). 그런데 `RenameView` 는 그 작업이 안 돼 있어서 404 를 모두 "view 없음"으로 보고합니다.

## 2. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `catalog/rest/rest.go` | 2167-2210 | `RenameTable`: 모범 답안. 주석까지 그대로 참고 |
| 같은 파일 | 2791-2830 | `RenameView`: 고칠 곳 |
| 같은 파일 | 423-430 | `withErrorTypeOverride`: 응답 body 의 `error.type` 문자열 → Go 에러 매핑 옵션 |
| `catalog/rest/scan_planning.go` | 100 | `errTypeNoSuchNamespace = "NoSuchNamespaceException"` 상수 |
| `catalog/rest/rest_test.go` | 2148-2240 | `RenameTable` 의 404 테스트들. 테스트도 이걸 따라 작성 |
| 같은 파일 | 3871 | `TestRenameView404`: 기존 view 테스트 |

Go 개념 메모:
- **sentinel error**: `var ErrNoSuchView = errors.New("view does not exist")` 처럼 패키지 수준에 선언해 둔 에러 값. 호출자는 `errors.Is(err, catalog.ErrNoSuchView)` 로 종류를 판별합니다.
- **functional options**: `doPost(..., allowNoContent(), withErrorTypeOverride(...))` 처럼 `func(*reqConfig)` 를 가변 인자(`...reqOption`)로 받아 설정을 하나씩 적용하는 패턴. 옵션을 하나 추가하는 것으로 끝납니다.
- `doPost[payload, any](...)` 의 대괄호는 **제네릭** 타입 인자입니다(요청 body 타입, 응답 타입).

## 3. 수정

`catalog/rest/rest.go` `RenameView` 안

```diff
 	_, err = doPost[payload, any](ctx, r.baseURI, path, payload{Source: src, Destination: dst}, r.cl,
 		map[int]error{
 			http.StatusNotFound: catalog.ErrNoSuchView,
 			http.StatusConflict: catalog.ErrViewAlreadyExists,
-		}, allowNoContent())
+		}, allowNoContent(), withErrorTypeOverride(map[string]error{errTypeNoSuchNamespace: catalog.ErrNoSuchNamespace}))
```

`RenameTable` 위에 있는 주석("The spec's rename 404 covers both...")을 짧게 붙여 주면 리뷰어가 의도를 바로 알 수 있습니다.

## 4. 테스트

`catalog/rest/rest_test.go` 의 `TestRenameView404` 아래에 추가합니다(`RestCatalogSuite` 메서드. `r.mux` 에 핸들러를 등록하고, `r.srv.URL` 로 catalog 를 만듭니다).

```go
func (r *RestCatalogSuite) TestRenameView404MissingNamespace() {
	r.mux.HandleFunc("/v1/views/rename", func(w http.ResponseWriter, req *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusNotFound)
		json.NewEncoder(w).Encode(map[string]any{"error": errorResponse{
			Message: "Namespace does not exist: missing",
			Type:    "NoSuchNamespaceException",
			Code:    404,
		}})
	})

	cat, err := rest.NewCatalog(context.Background(), "rest", r.srv.URL, rest.WithOAuthToken(TestToken))
	r.Require().NoError(err)

	_, err = cat.RenameView(context.Background(),
		catalog.ToIdentifier("example", "source"), catalog.ToIdentifier("missing", "destination"))
	r.ErrorIs(err, catalog.ErrNoSuchNamespace)
	r.NotErrorIs(err, catalog.ErrNoSuchView)
}
```

수정 전 결과(2026-10-24 실행): `--- FAIL: TestRestCatalog/TestRenameView404MissingNamespace`. 수정 후 통과.

suite 메서드만 골라 실행하는 방법:

```bash
go test -count=1 -run 'TestRestCatalog/TestRenameView' -v ./catalog/rest/
```

`TestRestCatalog` 는 `suite.Run(t, new(RestCatalogSuite))` 를 부르는 진입 함수이고, `/` 뒤가 메서드 이름입니다.

## 5. 실행

```bash
go test ./catalog/rest/...
make lint
git commit -s -m "fix(catalog/rest): report a missing namespace on RenameView 404"
```

## 6. 주의

- `rest-update-view-commit-state-unknown` 과 같은 파일이고 둘 다 작은 수정이라, 메인테이너가 괜찮다고 하면 PR 하나로 묶어도 됩니다. 기본은 issue 하나에 PR 하나입니다.
- `RenameTable` 테스트 중 `TestRenameTableNoSuchNamespaceTypeIsStatusGated` 는 "type override 는 매핑된 status 에만 적용된다"는 것을 고정합니다. view 에도 같은 테스트를 하나 더 넣으면 리뷰가 빨라집니다.
