# 수정 가이드: UpdateView 의 5xx 를 ErrCommitStateUnknown 으로

난이도: ★☆☆☆☆ (`CommitTable` 의 map 을 그대로 옮기기)

## 1. 문제를 한 줄로

table commit 은 5xx 를 "커밋 됐는지 모름(`ErrCommitStateUnknown`)"으로 돌려주는데, view commit 은 그냥 서버 에러로 돌려줍니다.

왜 중요한가: "모름"과 "실패"는 호출자의 대응이 다릅니다. 실패면 정리하고 재시도하면 되지만, 모름이면 실제로 커밋됐을 수 있으니 metadata 를 다시 읽어서 확인해야 합니다. 이걸 잘못 구분하면 이미 커밋된 결과를 지울 수 있습니다.

## 2. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `catalog/rest/rest.go` | 1843-1851 | `CommitTable`: 4개 5xx → `ErrCommitStateUnknown` (모범 답안) |
| 같은 파일 | 2677-2709 | `UpdateView`: 2703 줄 map 에 404/409 만 있음 |
| `catalog/rest/rest_test.go` | 4276-4320 | `TestCommitTableErrCommitStateUnknown`: 테스트도 이걸 복사해서 수정 |
| 같은 파일 | 3720 | `TestUpdateViewLabels`: view commit 경로(`/v1/namespaces/<ns>/views/<name>`) 예시 |

Go 개념 메모:
- `map[int]error{...}` 는 HTTP status → sentinel error 매핑표입니다. `doPost` 가 응답 코드로 이 표를 찾아 에러를 감쌉니다.
- `r.Run(name, func(){...})` 는 testify suite 의 **subtest** 입니다. `-run 'TestRestCatalog/TestUpdateViewErrCommitStateUnknown/502'` 처럼 하나만 돌릴 수 있습니다.

## 3. 수정

```diff
 	ret, err := doPost[payload, viewResponse](ctx, r.baseURI, path,
 		payload{Identifier: restIdentifier, Requirements: requirements, Updates: updates}, r.cl,
-		map[int]error{http.StatusNotFound: catalog.ErrNoSuchView, http.StatusConflict: ErrCommitFailed})
+		map[int]error{
+			http.StatusNotFound:            catalog.ErrNoSuchView,
+			http.StatusConflict:            ErrCommitFailed,
+			http.StatusInternalServerError: ErrCommitStateUnknown,
+			http.StatusBadGateway:          ErrCommitStateUnknown,
+			http.StatusServiceUnavailable:  ErrCommitStateUnknown,
+			http.StatusGatewayTimeout:      ErrCommitStateUnknown,
+		})
```

## 4. 테스트

`catalog/rest/rest_test.go` 의 `TestCommitTableErrCommitStateUnknown` 아래에 추가합니다.

```go
func (r *RestCatalogSuite) TestUpdateViewErrCommitStateUnknown() {
	var statusCode int

	r.mux.HandleFunc("/v1/namespaces/ns/views/view", func(w http.ResponseWriter, req *http.Request) {
		w.WriteHeader(statusCode)
		json.NewEncoder(w).Encode(map[string]any{
			"error": map[string]any{
				"message": http.StatusText(statusCode),
				"type":    "ServerException",
				"code":    statusCode,
			},
		})
	})

	cat, err := rest.NewCatalog(context.Background(), "rest", r.srv.URL)
	r.Require().NoError(err)

	for _, code := range []int{
		http.StatusInternalServerError,
		http.StatusBadGateway,
		http.StatusServiceUnavailable,
		http.StatusGatewayTimeout,
	} {
		statusCode = code
		r.Run(strconv.Itoa(code), func() {
			_, err := cat.UpdateView(context.Background(), table.Identifier{"ns", "view"}, nil, nil)
			r.ErrorIs(err, rest.ErrCommitStateUnknown,
				"%d should return ErrCommitStateUnknown, got: %v", code, err)
		})
	}
}
```

수정 전 결과(2026-10-24 실행): 4개 subtest 가 모두 FAIL 이고, 수정 후에는 통과합니다.

## 5. 실행

```bash
go test -count=1 -run 'TestRestCatalog/TestUpdateView' -v ./catalog/rest/
go test ./catalog/rest/...
make lint
git commit -s -m "fix(catalog/rest): return ErrCommitStateUnknown for UpdateView 5xx"
```

## 6. 주의

- `ErrCommitStateUnknown` 은 `rest` 패키지의 sentinel 입니다(`catalog` 패키지가 아님). 테스트에서는 `rest.ErrCommitStateUnknown` 으로 씁니다.
- view 의 다른 커밋 경로(예: `CreateView` 의 5xx)도 같은 의미인지 Java `ErrorHandlers` 를 보고 PR 범위를 정하세요. Java 에서도 create 는 `CommitStateUnknown` 이 아니므로 `UpdateView` 만 고치면 됩니다.
