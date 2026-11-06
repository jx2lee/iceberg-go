# 수정 가이드: Glue UpdateTable 이 큰 스키마에서 payload 한도 초과로 실패

난이도: ★★☆☆☆ (코드는 if 문 하나와 인자 하나. 다만 새 property 를 넣어도 되는지 메인테이너와 먼저 합의해야 함)

## 1. 문제를 한 줄로

Glue 카탈로그는 커밋할 때마다 Iceberg 컬럼 전체(현재 + 과거 스키마)를 Glue 테이블의 `StorageDescriptor.Columns` 에 다시 써 보냅니다. 컬럼이 3600개쯤 되면 요청이 Glue 한도를 넘어 매번 실패하는데, 이걸 끌 방법이 없습니다.

## 2. 배경

- Glue 테이블은 Iceberg 테이블을 가리키는 "포인터"입니다. 진짜 스키마는 S3 의 metadata JSON 에 있고, Glue 의 `Parameters["metadata_location"]` 이 그 파일을 가리킵니다.
- `StorageDescriptor.Columns` 는 Athena 콘솔 같은 도구가 보여 주는 정보용 사본입니다. Java 코드 주석에도 "informational purpose" 라고 적혀 있습니다.
- 컬럼 하나는 요청 JSON 에서 약 150바이트입니다(이름, 타입, 파라미터 3개). 3600개면 약 546KB 입니다. AWS 는 한도를 공식 문서에 적지 않았고, 사용자들이 보고한 값은 약 400KB 입니다.
- Java, PyIceberg 도 똑같이 전부 보냅니다. Java 쪽에서 "과거 컬럼 빼기" 옵션 PR(#11334, #12664)이 두 번 나왔지만 둘 다 머지되지 않고 닫혔습니다.
- iceberg-go 에서는 #769 가 `glue.schema-columns` 를 제안했습니다. 메인테이너(zeroshade)는 "표준이 아닌 새 property 를 만드는 게 맞나, dev 메일링 리스트에서 논의해 보라"고 했고, 답이 없어서 작성자가 2026-10-13 에 닫았습니다.

## 3. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `catalog/glue/glue.go` | 77-79 | `SkipArchive` property 상수. 새 상수를 여기 옆에 둡니다 |
| 같은 파일 | 1216-1236 | `constructTableInput`: `TableInput` 을 만들고 1228줄에서 컬럼을 항상 채움 |
| 같은 파일 | 340, 490, 536, 590, 610 | `constructTableInput` 을 부르는 곳 다섯 군데(CreateTable, S3 Tables 커밋, RegisterTable, CommitTable 의 update/create) |
| `catalog/glue/schema.go` | 32-62 | `schemasToGlueColumns`: 현재 스키마 컬럼 → 과거 스키마 컬럼 순서로 이름 중복 없이 모음 |
| 같은 파일 | 125-140 | `fieldToGlueColumn`: 컬럼 하나에 `iceberg.field.id/optional/current` 파라미터를 붙임 |
| `catalog/glue/glue_test.go` | 377 | `TestGlueConstructTableInputPreservesExistingColumnComments`: `table.NewMetadata` → `table.New` 로 테스트용 테이블을 만드는 예시 |

Go 개념 메모:
- `c.props.GetBool(key, default)` 는 카탈로그 생성 때 받은 문자열 property 를 bool 로 읽습니다. 값이 없거나 파싱이 안 되면 default 를 돌려줍니다. `SkipArchive` 도 같은 방식입니다.
- AWS SDK v2 클라이언트는 `glue.Options.HTTPClient` 에 `Do(*http.Request) (*http.Response, error)` 메서드만 있으면 아무 타입이나 받습니다(인터페이스). 테스트에서는 요청 body 길이만 재고 에러를 돌려주는 가짜를 넣어서, 네트워크 없이 "실제로 보낼 바이트 수"를 잽니다.
- 기존 `mockGlueClient` 는 `*glue.UpdateTableInput` 구조체만 받기 때문에 직렬화된 크기를 알 수 없습니다. 그래서 이 테스트는 진짜 SDK 클라이언트 + 가짜 HTTP 클라이언트를 씁니다.
- `RetryMaxAttempts: 1` 을 주지 않으면 SDK 가 전송 에러를 재시도하면서 테스트가 느려집니다.

## 4. 수정

`catalog/glue/glue.go`

```diff
 	SkipArchive        = "glue.skip-archive"
 	SkipArchiveDefault = true
 
+	// Write Iceberg schema columns to the Glue StorageDescriptor. Set to false
+	// to omit them when a very large schema exceeds Glue's request payload
+	// limit. The table stays readable through its metadata file, but tools that
+	// read columns from Glue (Athena DDL views, Lake Formation column grants)
+	// will not see them.
+	SchemaColumns        = "glue.schema-columns"
+	SchemaColumnsDefault = true
+
```

```diff
-func constructTableInput(tableName string, staged *table.Table, previousGlueTable *types.Table) *types.TableInput {
+func constructTableInput(tableName string, staged *table.Table, previousGlueTable *types.Table, withColumns bool) *types.TableInput {
 	...
 		StorageDescriptor: &types.StorageDescriptor{
 			Location: aws.String(staged.Location()),
-			Columns:  schemasToGlueColumns(staged.Metadata(), existingColumns),
 		},
 	}
+	if withColumns {
+		tableInput.StorageDescriptor.Columns = schemasToGlueColumns(staged.Metadata(), existingColumns)
+	}
```

호출하는 다섯 곳은 모두 마지막 인자만 추가합니다.

```diff
-		TableInput:   constructTableInput(tableName, staged.Table, nil),
+		TableInput:   constructTableInput(tableName, staged.Table, nil, c.props.GetBool(SchemaColumns, SchemaColumnsDefault)),
```

기존 테스트 세 곳(`glue_test.go` 369, 405, 417줄)은 `constructTableInput(..., true)` 로 바꿉니다.

전체 diff (main `2e69b8a` 기준):

```diff
diff --git a/catalog/glue/glue.go b/catalog/glue/glue.go
index 4e65ac4..9284257 100644
--- a/catalog/glue/glue.go
+++ b/catalog/glue/glue.go
@@ -78,6 +78,14 @@ const (
 	SkipArchive        = "glue.skip-archive"
 	SkipArchiveDefault = true
 
+	// Write Iceberg schema columns to the Glue StorageDescriptor. Set to false
+	// to omit them when a very large schema exceeds Glue's request payload
+	// limit. The table stays readable through its metadata file, but tools that
+	// read columns from Glue (Athena DDL views, Lake Formation column grants)
+	// will not see them.
+	SchemaColumns        = "glue.schema-columns"
+	SchemaColumnsDefault = true
+
 	AccessKeyID     = "glue.access-key-id"
 	SecretAccessKey = "glue.secret-access-key"
 	SessionToken    = "glue.session-token"
@@ -337,7 +345,7 @@ func (c *Catalog) CreateTable(ctx context.Context, identifier table.Identifier,
 	_, err = c.glueSvc.CreateTable(ctx, &glue.CreateTableInput{
 		CatalogId:    c.catalogId,
 		DatabaseName: aws.String(database),
-		TableInput:   constructTableInput(tableName, staged.Table, nil),
+		TableInput:   constructTableInput(tableName, staged.Table, nil, c.props.GetBool(SchemaColumns, SchemaColumnsDefault)),
 	})
 	if err != nil {
 		if isAlreadyExistsException(err) {
@@ -487,7 +495,7 @@ func (c *Catalog) commitS3TablesTable(ctx context.Context, database, tableName s
 	_, err = c.glueSvc.UpdateTable(ctx, &glue.UpdateTableInput{
 		CatalogId:    c.catalogId,
 		DatabaseName: aws.String(database),
-		TableInput:   constructTableInput(tableName, staged.Table, allocated.Table),
+		TableInput:   constructTableInput(tableName, staged.Table, allocated.Table, c.props.GetBool(SchemaColumns, SchemaColumnsDefault)),
 		VersionId:    allocated.Table.VersionId,
 		SkipArchive:  aws.Bool(c.props.GetBool(SkipArchive, SkipArchiveDefault)),
 	})
@@ -533,7 +541,7 @@ func (c *Catalog) RegisterTable(ctx context.Context, identifier table.Identifier
 	_, err = c.glueSvc.CreateTable(ctx, &glue.CreateTableInput{
 		CatalogId:    c.catalogId,
 		DatabaseName: aws.String(database),
-		TableInput:   constructTableInput(tableName, tbl, nil),
+		TableInput:   constructTableInput(tableName, tbl, nil, c.props.GetBool(SchemaColumns, SchemaColumnsDefault)),
 	})
 	if err != nil {
 		if isAlreadyExistsException(err) {
@@ -587,7 +595,7 @@ func (c *Catalog) CommitTable(ctx context.Context, identifier table.Identifier,
 		_, err = c.glueSvc.UpdateTable(ctx, &glue.UpdateTableInput{
 			CatalogId:    c.catalogId,
 			DatabaseName: aws.String(database),
-			TableInput:   constructTableInput(tableName, staged.Table, currentGlueTable),
+			TableInput:   constructTableInput(tableName, staged.Table, currentGlueTable, c.props.GetBool(SchemaColumns, SchemaColumnsDefault)),
 			// use `VersionId` to implement optimistic locking
 			VersionId:   currentGlueTable.VersionId,
 			SkipArchive: aws.Bool(c.props.GetBool(SkipArchive, SkipArchiveDefault)),
@@ -607,7 +615,7 @@ func (c *Catalog) CommitTable(ctx context.Context, identifier table.Identifier,
 		_, err = c.glueSvc.CreateTable(ctx, &glue.CreateTableInput{
 			CatalogId:    c.catalogId,
 			DatabaseName: aws.String(database),
-			TableInput:   constructTableInput(tableName, staged.Table, nil),
+			TableInput:   constructTableInput(tableName, staged.Table, nil, c.props.GetBool(SchemaColumns, SchemaColumnsDefault)),
 		})
 		if err != nil {
 			if isAlreadyExistsException(err) {
@@ -1213,7 +1221,7 @@ func constructParameters(staged *table.Table, previousGlueTable *types.Table) ma
 	return parameters
 }
 
-func constructTableInput(tableName string, staged *table.Table, previousGlueTable *types.Table) *types.TableInput {
+func constructTableInput(tableName string, staged *table.Table, previousGlueTable *types.Table, withColumns bool) *types.TableInput {
 	var existingColumns []types.Column
 	if previousGlueTable != nil && previousGlueTable.StorageDescriptor != nil {
 		existingColumns = previousGlueTable.StorageDescriptor.Columns
@@ -1225,9 +1233,11 @@ func constructTableInput(tableName string, staged *table.Table, previousGlueTabl
 		Parameters: constructParameters(staged, previousGlueTable),
 		StorageDescriptor: &types.StorageDescriptor{
 			Location: aws.String(staged.Location()),
-			Columns:  schemasToGlueColumns(staged.Metadata(), existingColumns),
 		},
 	}
+	if withColumns {
+		tableInput.StorageDescriptor.Columns = schemasToGlueColumns(staged.Metadata(), existingColumns)
+	}
 
 	if comment, ok := descriptionProperty(staged.Properties()); ok {
 		tableInput.Description = aws.String(comment)
diff --git a/catalog/glue/glue_test.go b/catalog/glue/glue_test.go
index e33d19a..6e95e85 100644
--- a/catalog/glue/glue_test.go
+++ b/catalog/glue/glue_test.go
@@ -23,6 +23,7 @@ import (
 	"errors"
 	"fmt"
 	"io"
+	"net/http"
 	"net/url"
 	"os"
 	"path/filepath"
@@ -366,7 +367,7 @@ func TestGlueConstructDescriptionCompatibility(t *testing.T) {
 				nil,
 			)
 
-			tableInput := constructTableInput("test_table", staged, nil)
+			tableInput := constructTableInput("test_table", staged, nil, true)
 			require.Equal(t, tt.expected, aws.ToString(tableInput.Description))
 			require.NotContains(t, tableInput.Parameters, PropsKeyDescription)
 			require.NotContains(t, tableInput.Parameters, legacyPropsKeyDescription)
@@ -402,7 +403,7 @@ func TestGlueConstructTableInputPreservesExistingColumnComments(t *testing.T) {
 		},
 	}
 
-	input := constructTableInput("test_table", staged, previous)
+	input := constructTableInput("test_table", staged, previous, true)
 	columns := make(map[string]types.Column, len(input.StorageDescriptor.Columns))
 	for _, column := range input.StorageDescriptor.Columns {
 		columns[aws.ToString(column.Name)] = column
@@ -414,7 +415,7 @@ func TestGlueConstructTableInputPreservesExistingColumnComments(t *testing.T) {
 
 	withoutStorageDescriptor := &types.Table{}
 	for _, previous := range []*types.Table{nil, withoutStorageDescriptor} {
-		input := constructTableInput("test_table", staged, previous)
+		input := constructTableInput("test_table", staged, previous, true)
 		for _, column := range input.StorageDescriptor.Columns {
 			require.Nil(t, column.Comment)
 		}
@@ -3526,3 +3527,81 @@ func TestGlueCommitTableUsesCatalogAwsConfig(t *testing.T) {
 	assertRecordedConfig(t, awsCfg, recorded)
 	mockGlueSvc.AssertExpectations(t)
 }
+
+// bodySizeRecorder is an HTTP client for the real Glue SDK client that records
+// the serialized request body size and never sends anything over the network.
+type bodySizeRecorder struct{ size int }
+
+func (r *bodySizeRecorder) Do(req *http.Request) (*http.Response, error) {
+	body, err := io.ReadAll(req.Body)
+	if err != nil {
+		return nil, err
+	}
+	r.size = len(body)
+
+	return nil, errors.New("request recorded, not sent")
+}
+
+// updateTablePayloadSize returns the exact number of bytes the AWS SDK would
+// send for an UpdateTable call carrying the given TableInput.
+func updateTablePayloadSize(t *testing.T, input *types.TableInput) int {
+	t.Helper()
+
+	recorder := &bodySizeRecorder{}
+	client := glue.New(glue.Options{
+		Region:           "us-east-1",
+		Credentials:      aws.AnonymousCredentials{},
+		HTTPClient:       recorder,
+		RetryMaxAttempts: 1,
+	})
+	_, err := client.UpdateTable(context.Background(), &glue.UpdateTableInput{
+		DatabaseName: aws.String("test_database"),
+		TableInput:   input,
+	})
+	require.Error(t, err)
+	require.NotZero(t, recorder.size)
+
+	return recorder.size
+}
+
+func TestGlueTableInputPayloadLargeSchema(t *testing.T) {
+	// Issue #701: about 3600 schema fields make UpdateTable fail with
+	// "Payload size of request exceeded limit". Glue does not document the
+	// limit; 400 KB is the value reported by AWS users for the table schema.
+	const reportedGlueSchemaLimit = 400 * 1024
+
+	fields := make([]iceberg.NestedField, 3600)
+	for i := range fields {
+		fields[i] = iceberg.NestedField{
+			ID:   i + 1,
+			Name: fmt.Sprintf("ecs_field_name_%05d", i),
+			Type: iceberg.StringType{},
+		}
+	}
+	metadata, err := table.NewMetadata(
+		iceberg.NewSchema(0, fields...),
+		nil,
+		table.UnsortedSortOrder,
+		"s3://test-bucket/test_table",
+		nil,
+	)
+	require.NoError(t, err)
+	staged := table.New(
+		TableIdentifier("test_database", "test_table"),
+		metadata,
+		"s3://test-bucket/test_table/metadata/v2.metadata.json",
+		nil,
+		nil,
+	)
+
+	// Default behaviour matches Java and PyIceberg: every column is sent.
+	withColumns := constructTableInput("test_table", staged, nil, true)
+	require.Len(t, withColumns.StorageDescriptor.Columns, len(fields))
+	require.Greater(t, updateTablePayloadSize(t, withColumns), reportedGlueSchemaLimit)
+
+	// glue.schema-columns=false keeps the request small.
+	withoutColumns := constructTableInput("test_table", staged, nil, false)
+	require.Empty(t, withoutColumns.StorageDescriptor.Columns)
+	require.Equal(t, "s3://test-bucket/test_table", aws.ToString(withoutColumns.StorageDescriptor.Location))
+	require.Less(t, updateTablePayloadSize(t, withoutColumns), reportedGlueSchemaLimit)
+}
```

## 5. 테스트

`catalog/glue/glue_test.go`(`package glue`) 끝에 붙입니다. import 에 `"net/http"` 를 추가해야 합니다. 나머지(`context`, `errors`, `fmt`, `io`, `iceberg`, `table`, `aws`, `glue`, `types`, `require`)는 이미 있습니다.

```go
// bodySizeRecorder is an HTTP client for the real Glue SDK client that records
// the serialized request body size and never sends anything over the network.
type bodySizeRecorder struct{ size int }

func (r *bodySizeRecorder) Do(req *http.Request) (*http.Response, error) {
	body, err := io.ReadAll(req.Body)
	if err != nil {
		return nil, err
	}
	r.size = len(body)

	return nil, errors.New("request recorded, not sent")
}

// updateTablePayloadSize returns the exact number of bytes the AWS SDK would
// send for an UpdateTable call carrying the given TableInput.
func updateTablePayloadSize(t *testing.T, input *types.TableInput) int {
	t.Helper()

	recorder := &bodySizeRecorder{}
	client := glue.New(glue.Options{
		Region:           "us-east-1",
		Credentials:      aws.AnonymousCredentials{},
		HTTPClient:       recorder,
		RetryMaxAttempts: 1,
	})
	_, err := client.UpdateTable(context.Background(), &glue.UpdateTableInput{
		DatabaseName: aws.String("test_database"),
		TableInput:   input,
	})
	require.Error(t, err)
	require.NotZero(t, recorder.size)

	return recorder.size
}

func TestGlueTableInputPayloadLargeSchema(t *testing.T) {
	// Issue #701: about 3600 schema fields make UpdateTable fail with
	// "Payload size of request exceeded limit". Glue does not document the
	// limit; 400 KB is the value reported by AWS users for the table schema.
	const reportedGlueSchemaLimit = 400 * 1024

	fields := make([]iceberg.NestedField, 3600)
	for i := range fields {
		fields[i] = iceberg.NestedField{
			ID:   i + 1,
			Name: fmt.Sprintf("ecs_field_name_%05d", i),
			Type: iceberg.StringType{},
		}
	}
	metadata, err := table.NewMetadata(
		iceberg.NewSchema(0, fields...),
		nil,
		table.UnsortedSortOrder,
		"s3://test-bucket/test_table",
		nil,
	)
	require.NoError(t, err)
	staged := table.New(
		TableIdentifier("test_database", "test_table"),
		metadata,
		"s3://test-bucket/test_table/metadata/v2.metadata.json",
		nil,
		nil,
	)

	// Default behaviour matches Java and PyIceberg: every column is sent.
	withColumns := constructTableInput("test_table", staged, nil, true)
	require.Len(t, withColumns.StorageDescriptor.Columns, len(fields))
	require.Greater(t, updateTablePayloadSize(t, withColumns), reportedGlueSchemaLimit)

	// glue.schema-columns=false keeps the request small.
	withoutColumns := constructTableInput("test_table", staged, nil, false)
	require.Empty(t, withoutColumns.StorageDescriptor.Columns)
	require.Equal(t, "s3://test-bucket/test_table", aws.ToString(withoutColumns.StorageDescriptor.Location))
	require.Less(t, updateTablePayloadSize(t, withoutColumns), reportedGlueSchemaLimit)
}
```

수정 전에 문제를 보려면, 위 테스트 마지막 부분을 아래처럼 바꿔 main 에서 돌립니다(인자 3개짜리 원래 시그니처).

```go
	input := constructTableInput("test_table", staged, nil)
	size := updateTablePayloadSize(t, input)
	t.Logf("columns=%d payload=%d bytes", len(input.StorageDescriptor.Columns), size)
	require.Less(t, size, reportedGlueSchemaLimit)
```

## 6. 실행

```bash
go test -count=1 -run TestGlueTableInputPayloadLargeSchema -v ./catalog/glue/
go test ./catalog/glue/...
go vet ./catalog/glue/...
make lint
git commit -s -m "fix(catalog/glue): add glue.schema-columns to skip StorageDescriptor columns"
```

## 7. 결과

수정 전(2026-11-06, main `2e69b8a`, 위의 "수정 전" 버전):

```text
=== RUN   TestGlueTableInputPayloadLargeSchema
    glue_test.go:3599: columns=3600 payload=546378 bytes
    glue_test.go:3600:
        	Error:      	"546378" is not less than "409600"
--- FAIL: TestGlueTableInputPayloadLargeSchema (0.02s)
```

수정 후:

```text
=== RUN   TestGlueTableInputPayloadLargeSchema
--- PASS: TestGlueTableInputPayloadLargeSchema (0.01s)
PASS
ok  	github.com/apache/iceberg-go/catalog/glue
```

`go test ./catalog/glue/...` 통과, `go vet` 통과.

## 8. 주의

- **PR 부터 열지 마세요.** #769 는 코드가 아니라 "표준에 없는 property" 문제로 멈췄습니다. issue.md 의 claim 댓글로 메인테이너에게 먼저 방향을 묻고, 답을 받은 뒤 진행합니다. 이름을 바꾸라고 하면 상수 값만 바꾸면 됩니다.
- 400KB 는 AWS 공식 수치가 아닙니다. PR 설명에서도 "보고된 값"이라고 적고, 테스트도 한도를 정확히 맞히는 용도가 아니라 크기 차이를 보여 주는 용도로 둡니다.
- "과거 컬럼만 빼자"는 대안은 Java 와도 다르고, 리포터에게도 도움이 안 됩니다(3600개가 모두 현재 스키마에 있음).
- `false` 로 두면 `UpdateTable` 이 기존 Glue 컬럼을 지웁니다(UpdateTable 은 TableInput 전체를 교체). Lake Formation 컬럼 권한이나 Athena 컬럼 목록을 쓰는 사용자는 영향을 받습니다. #769 는 이를 막으려고 `glue.lakeformation-enabled` 검사를 따로 넣었는데, 그건 property 를 하나 더 만드는 것이라 여기서는 뺐습니다. 리뷰에서 요청하면 추가합니다.
- 중첩 struct 로 된 스키마는 컬럼 수가 적어서(3600 leaf → 60 컬럼, 91KB) 같은 문제가 덜 납니다. 대신 struct 하나의 `Type` 문자열이 131072자를 넘으면 별도 검증 에러가 날 수 있는데, 이 이슈 범위는 아닙니다.
