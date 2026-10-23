# 수정 가이드: void 필드 이름으로 RenameField 하면 엉뚱한 필드가 바뀜

난이도: ★★☆☆☆ (재귀 호출의 인자 두 개를 바로잡고, return 을 없애는 작업)

## 1. 문제를 한 줄로

`RenameField("a", "b")` 를 했는데 `b` 가 지워진(void) partition 필드의 이름이면, Go 는 **`a` 를 `a_<voidID>` 로** 바꾸고 끝납니다. Java 는 **void 필드 `b` 를 `b_<voidID>` 로 치운 뒤 `a → b`** 를 적용합니다.

## 2. 배경: v1 의 void partition 필드

v1 테이블에서는 partition 필드를 지워도 spec 에서 빠지지 않습니다. `void` transform(항상 null)으로 남고 이름도 그대로 유지됩니다. 그래서 같은 이름을 다시 쓰려면 void 필드의 이름을 먼저 `<이름>_<fieldId>` 로 바꿔 비켜 줘야 합니다. `addField` 는 이 처리가 되어 있습니다(Java 225-233줄과 같음).

## 3. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `table/update_spec.go` | 433-458 | `renameField`: 문제 코드 |
| 같은 파일 | 307- | `addField`: 같은 상황을 제대로 처리하는 참고 코드 |
| `table/update_spec_test.go` | 64 | `testPartitionedTable`: `id_identity`(1000), `street_void`(1001, void) 를 가진 테이블 |
| Java `core/.../BaseUpdatePartitionSpec.java` | 282-301 | 정답 동작 |

Go 개념 메모:
- `updateSpecOp` 는 `func() error` 타입입니다. `us.renameField(a, b)` 는 **함수를 돌려줄 뿐 실행하지 않으므로**, 끝에 `()` 를 한 번 더 붙여야 실행됩니다. 그래서 코드가 `us.renameField(...)()` 모양입니다.
- `existingField, exists := us.nameToField[newName]` 다음 줄의 `existingField.Transform.(iceberg.VoidTransform)` 는 키가 없을 때 0 값 struct 의 nil `Transform` 에 대한 type assertion 입니다. nil 인터페이스에 대한 `x.(T)` 의 두 값 형태는 panic 없이 `ok=false` 를 줍니다. 그래서 `exists &&` 와 함께 써도 안전합니다.

## 4. 수정

```diff
 		existingField, exists := us.nameToField[newName]
 		_, isVoidTransform := existingField.Transform.(iceberg.VoidTransform)
 		if exists && isVoidTransform {
-			return us.renameField(
-				name,
-				fmt.Sprintf("%s_%d", name, existingField.FieldID),
-			)()
+			// rename the old deleted field that is being replaced by the new field
+			if err := us.renameField(
+				newName,
+				fmt.Sprintf("%s_%d", newName, existingField.FieldID),
+			)(); err != nil {
+				return err
+			}
 		}
```

세 가지가 바뀝니다:
1. 첫 인자 `name` → `newName` (void 필드를 치움)
2. 접미사 기준도 `newName`
3. `return` 을 없애서 원래 요청한 rename 이 이어서 실행되게 함

## 5. 테스트

`table/update_spec_test.go`(`package table_test`) 에 추가합니다.

```go
func TestRenameFieldOntoVoidField(t *testing.T) {
	us := table.NewUpdateSpec(testPartitionedTable.NewTransaction(), true)
	_, _, err := us.RenameField("id_identity", "street_void").BuildUpdates()
	require.NoError(t, err)

	newSpec, err := us.Apply()
	require.NoError(t, err)
	assert.Equal(t, "street_void", newSpec.FieldsBySourceID(1)[0].Name)
	assert.Equal(t, "street_void_1001", newSpec.FieldsBySourceID(5)[0].Name)
}
```

수정 전 결과(2026-10-24 실행):

```text
new spec: [ 1000: id_identity_1001: identity(1)   1001: street_void: void(5) ]
Error: Not equal: expected: "street_void"  actual: "id_identity_1001"
```

## 6. 실행

```bash
go test -count=1 -run TestRenameFieldOntoVoidField -v ./table/
go test ./table/...
make lint
git commit -s -m "fix(table): move the void field aside when renaming onto its name"
```

## 7. 주의

- 파일 위쪽에 같은 이름의 기존 테스트가 있는지 먼저 확인하세요: `grep -n "Void" table/update_spec_test.go`. 혹시 기존 테스트가 "잘못된 동작"을 고정하고 있으면 그 테스트도 같이 고쳐야 하고, PR 본문에 그 사실을 적어야 합니다(2026-10-24 기준 수정 후 `./table/...` 전체 통과이므로 그런 테스트는 없음).
