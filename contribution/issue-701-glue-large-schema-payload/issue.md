# Glue UpdateTable fails with "Payload size of request exceeded limit" for a large schema

- Issue: https://github.com/apache/iceberg-go/issues/701 (opened 2026-01-27 by @vrecan, OPEN, no assignee, no labels)
- Status check (2026-11-06):
  - Maintainer-side comment by @laskoviymishka confirms the bug: `constructTableInput` always writes every column (current + historical) into `StorageDescriptor.Columns`.
  - **PR #769** (`fix(glue): add ability for glue to skip StorageDescriptor cols`, @laskoviymishka) added a `glue.schema-columns` property (default `true`). @zeroshade's only concern: "I don't want us to be creating an entirely new catalog property if there might already exist a standard / common usage for this." He asked to start a thread on the Iceberg dev mailing list. On 2026-05-20 the author said "Silence on mail list :(" and "Need to bring this to one of the syncs". The PR was converted to draft and **closed by its author on 2026-10-13** with no merge.
  - No other open claim or PR. Searches `701`, `glue schema`, `glue columns`: only #769 is about this; #2113 (perf of column building), #1522 (keep column comments), #1562 (column order) touched the same code but do not change the payload size.
  - Java side: apache/iceberg#7584 (issue), #11334 and #12664 (`glue.non-current-fields-disabled` PRs) are all closed without merge. Nothing newer found.
  → **코드보다 "새 catalog property 를 써도 되나"라는 합의가 막힌 이슈. 메인테이너 확인 없이 PR 부터 열면 #769 와 같은 이유로 멈춘다.**
- Verified against main `2e69b8a` (2026-11-06)

## Repro (offline, no AWS)

The test builds a real `glue.Client` from the AWS SDK with a fake `HTTPClient` that records the serialized request body and never sends it. So the byte count is exactly what the SDK would put on the wire for `UpdateTable`.

Schema: 3600 top-level `string` columns named `ecs_field_name_00000` … (the reporter has ~3600 fields).

```text
=== RUN   TestGlueTableInputPayloadLargeSchema
    glue_test.go:3599: columns=3600 payload=546378 bytes
    glue_test.go:3600:
        	Error:      	"546378" is not less than "409600"
        	Test:       	TestGlueTableInputPayloadLargeSchema
--- FAIL: TestGlueTableInputPayloadLargeSchema (0.02s)
```

Other measurements with the same helper (main `2e69b8a`):

| Schema | Glue columns sent | Request bytes |
|---|---:|---:|
| 100 flat columns | 100 | 15,210 |
| 1,000 flat columns | 1,000 | 151,111 |
| 3,600 flat columns | 3,600 | 546,311 |
| 1,800 current columns + 1,800 renamed in an older schema | 3,600 | 547,004 |
| 60 structs × 60 leaves (3,600 leaves, nested) | 60 | 91,349 |

About 150 bytes per top-level column. Historical schemas add their dropped/renamed columns on top of the current ones.

Limits:
- AWS does not document the size limit behind `InvalidInputException: Payload size of request exceeded limit`. AWS users report a 400 KB limit on a table's schema ([re:Post](https://repost.aws/questions/QUvbjcqsavTF633n47J02MMw/feature-request-increase-aws-glue-catelog-400-kb-schema-size-limit)). The test uses 400 KiB only as that reported value, not as an official quota.
- Documented per-column limits ([API_Column](https://docs.aws.amazon.com/glue/latest/webapi/API_Column.html)): `Name` ≤ 255, `Comment` ≤ 255, `Type` ≤ 131072, parameter value ≤ 512000.

## Root cause (on `2e69b8a`)

- `catalog/glue/glue.go:1216-1236` — `constructTableInput` always sets `StorageDescriptor.Columns: schemasToGlueColumns(...)` (line 1228). There is no way to turn it off.
- `catalog/glue/schema.go:32-62` — `schemasToGlueColumns` adds every current column (`iceberg.field.current=true`) and then every column of every older schema whose name is not already present (`iceberg.field.current=false`).
- Callers: `glue.go:340` (CreateTable), `:490` (S3 Tables commit), `:536` (RegisterTable), `:590` (CommitTable → UpdateTable), `:610` (CommitTable → CreateTable). Every commit resends the full column list even when the schema did not change.

## Java / PyIceberg comparison

The Go code already does what Java and PyIceberg do. This is a gap in all three, not a Go-only bug.

- Java `IcebergToGlueConverter.setTableInputInformation` ([4a1513f](https://github.com/apache/iceberg/blob/4a1513f1f220cc6adcd96093b64d6ab43e0808f1/aws/src/main/java/org/apache/iceberg/aws/glue/IcebergToGlueConverter.java#L245-L290)) always sets `storageDescriptor.columns(toColumns(...))`. `toColumns` (L346-L366) adds current columns with `iceberg.field.current=true`, then **still adds** non-current columns with `iceberg.field.current=false`. Non-current columns are marked, not removed. No property to skip them.
- PyIceberg `_to_columns` ([glue.py](https://github.com/apache/iceberg-python/blob/27a5162749bb670d00b23b8360869410b6e5b3a1/pyiceberg/catalog/glue.py#L225-L256)) does the same.

Dropping only non-current columns would not help the reporter: they say all ~3600 fields are in the current schema.

## Claim comment draft (do not post yet)

```markdown
Hi, I'd like to pick this up, but I want to check the direction first since #769 was closed.

I reproduced the size offline by serializing the `UpdateTable` request with the
real AWS SDK client and a fake HTTP client: 3600 top-level string columns give a
546 KB request, and history adds dropped/renamed columns on top. Java and
PyIceberg build the column list the same way, so there is no existing
cross-implementation behaviour to copy.

The code change is small (an opt-in property, default unchanged, that leaves
`StorageDescriptor.Columns` empty; the table is still loaded from the metadata
file). The open question from #769 is the property itself. Would you accept it
as a Go catalog property (keeping the `glue.schema-columns` name from #769), or
should I first raise it again on dev@ / in a community sync? Happy to go either
way.
```
