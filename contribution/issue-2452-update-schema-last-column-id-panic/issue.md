# UpdateSchema Apply/Commit panics when last-column-id < existing field ID

- Issue: https://github.com/apache/iceberg-go/issues/2452 (opened 2026-11-06 by @Shruthirdy)
- Status check (2026-11-06): no assignee, no comments. The issue body says "I already have a fix for this and would be happy to open a PR". The reporter opened **PR #2453** (`fix(table): return error from UpdateSchema when last-column-id is bel…`, OPEN, no reviews yet) 12 minutes later. It uses the same approach as the fix below (`validateLastColumnID()` in `NewUpdateSchema`).
  → **이미 리포터가 claim 하고 PR 을 열었음. 새로 기여할 여지는 없고, 할 수 있는 건 #2453 리뷰 정도.**
- Verified against main `2e69b8a` (2026-11-06)

## Repro

`ExampleTableMetadataV2` has fields `x`=1, `y`=2, `z`=3. Change `"last-column-id": 3` to `2`.
`ParseMetadataString` accepts it. Adding a column with `UpdateSchema` gives the new column id 3 again, and `Apply()` panics:

```text
--- FAIL: TestUpdateSchemaLastColumnIDBelowHighestFieldID (0.00s)
    update_schema_test.go:2701:
        	Error:      	func (assert.PanicTestFunc)(0x1059d0020) should not panic
        	            		Panic value:	invalid schema: multiple fields for id 3: z and w
        	            	github.com/apache/iceberg-go.NewSchemaWithIdentifiers(...)
        	            		schema.go:96
        	            	github.com/apache/iceberg-go.NewSchema(...)
        	            		schema.go:74
        	            	github.com/apache/iceberg-go/table.(*UpdateSchema).Apply(...)
        	            		table/update_schema.go:1127
```

## Root cause (on `2e69b8a`)

- `table/update_schema.go:175` — `NewUpdateSchema` seeds `u.lastColumnID = meta.LastColumnID()` and does not check it against the field ids that are already in use.
- `table/update_schema.go:246-249` — `assignNewColumnID` hands out `lastColumnID+1` = 3, which `z` already uses.
- `table/update_schema.go:1127` — `Apply` calls `iceberg.NewSchema`, and `schema.go:95-96` panics on duplicate ids (`checkDuplicateFieldIDs`).
- `table/metadata.go:1313` — `MetadataBuilder.validateLastColumnID()` already detects this case and returns `ErrInvalidMetadata`. Today only `SetFormatVersion` (`metadata.go:1284`) calls it.

`NewUpdateSchema` is reached only through `Transaction.UpdateSchema` (`table/transaction.go:676`), so one check in `NewUpdateSchema` covers every path.

Java (`SchemaUpdate`) also copies `base.lastColumnId()` without checking it. There, a duplicate id makes the `Schema` constructor throw an exception. The caller gets an exception, which is Java's normal error path. In Go, the equivalent is returning an `error`.

## Claim comment draft (do not post — #2453 already exists)

> I'd like to work on this. I can reproduce it on main (`2e69b8a`): `Apply()` panics with `invalid schema: multiple fields for id 3: z and w`. Running the existing `MetadataBuilder.validateLastColumnID()` in `NewUpdateSchema` and storing the error in `u.err` turns it into `ErrInvalidMetadata` from `Apply()`/`Commit()`.

Since the reporter already opened #2453, a review comment there fits better, for example:

> Reproduced on `2e69b8a` and confirmed this change fixes it; `go test ./table/...` passes locally. LGTM.
