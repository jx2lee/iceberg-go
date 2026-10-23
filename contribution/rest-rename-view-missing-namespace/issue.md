# rest: RenameView reports a missing destination namespace as ErrNoSuchView

- Issue: 없음 (2026-10-24 검색: PR "rename view namespace", "NoSuchNamespaceException error type" → #2173(table 만), #1461(RenameView 추가)뿐. issue "rename view NoSuchNamespace" 결과 없음)
- 관련: PR #2173 이 `RenameTable` 의 같은 문제를 고침. sibling 인 `RenameView` 는 빠짐
- Verified against main `7768540`

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

`RenameView` maps every 404 from `POST /v1/views/rename` to `catalog.ErrNoSuchView` (`catalog/rest/rest.go:2820-2824`).

The REST spec's rename 404 covers both a missing source view and a missing destination namespace. The two are distinguished by `error.type`. Java's `ErrorHandlers.ViewErrorHandler` maps a 404 with `type=NoSuchNamespaceException` to `NoSuchNamespaceException`.

#2173 fixed exactly this for `RenameTable` by adding `withErrorTypeOverride(map[string]error{errTypeNoSuchNamespace: catalog.ErrNoSuchNamespace})`, but `RenameView` was not updated. A caller renaming a view into a namespace that does not exist gets `ErrNoSuchView`, which points at the source view instead of the destination namespace.

### Repro

Server returns `404 {"error":{"type":"NoSuchNamespaceException","message":"Namespace does not exist: missing"}}` for `/v1/views/rename`:

```text
--- FAIL: TestRestCatalog/TestRenameView404MissingNamespace
    Error: Target error should be in err chain:
           expected: "namespace does not exist"
           in chain: "NoSuchNamespaceException: Namespace does not exist: missing"
                     "view does not exist"
```

The fix is the same one-line option used for `RenameTable`.
