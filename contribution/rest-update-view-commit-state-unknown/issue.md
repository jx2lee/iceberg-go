# rest: UpdateView maps 5xx to server errors instead of ErrCommitStateUnknown

- Issue: 없음 (2026-10-24 검색: PR "UpdateView CommitStateUnknown", "ErrCommitStateUnknown" → #1832, #754 (무관). issue "view commit state unknown" 결과 없음)
- Verified against main `7768540`

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

`CommitTable` and `CommitTransaction` map 500/502/503/504 to `ErrCommitStateUnknown` (`catalog/rest/rest.go:1843-1851`, `~1933`), because after those responses the commit may or may not have been applied.

`UpdateView` (`catalog/rest/rest.go:2701-2703`) only maps 404 and 409. A 5xx comes back as `ErrServerError`/`ErrServiceUnavailable`, so a caller cannot tell an ambiguous view commit from a definite failure, and may retry or clean up as if nothing was written.

- REST spec `replaceView` lists `CommitStateUnknown` responses for 500/502/504, the same as `updateTable`.
- Java `ErrorHandlers.ViewCommitErrorHandler` throws `CommitStateUnknownException` for 500, 502, 503 and 504.

### Repro

```text
--- FAIL: TestRestCatalog/TestUpdateViewErrCommitStateUnknown/500
--- FAIL: TestRestCatalog/TestUpdateViewErrCommitStateUnknown/502
--- FAIL: TestRestCatalog/TestUpdateViewErrCommitStateUnknown/503
--- FAIL: TestRestCatalog/TestUpdateViewErrCommitStateUnknown/504
```
