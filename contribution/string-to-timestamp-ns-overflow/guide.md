# 수정 가이드: 문자열 → timestamp_ns 변환이 범위를 넘으면 조용히 값이 돌아감

난이도: ★★☆☆☆ (작은 헬퍼 함수 하나. 경계값 테스트를 꼼꼼히)

## 1. 문제를 한 줄로

`int64` 나노초로 표현할 수 있는 시간은 약 1677~2262년입니다. 그 밖의 문자열을 `timestamp_ns` 로 바꾸면, 에러가 나는 대신 **엉뚱한 연도로 값이 돌아갑니다**(1600년 → 2184년).

## 2. Go 개념

- `time.Time.UnixNano()` 문서: *"The result is undefined if the Unix time in nanoseconds cannot be represented by an int64 (a date before the year 1678 or after 2262)."* Go 는 정수 overflow 에서 panic 하지 않고 **wrap-around** 합니다.
- 범위 안인지 확인하는 간단한 방법은 **왕복(round-trip) 비교**입니다. `ns := tm.UnixNano()` 를 `time.Unix(0, ns)` 로 되돌렸을 때 원래 `tm` 과 같지 않으면 overflow 가 난 것입니다.
- `time.Time` 은 `==` 대신 **`Equal`** 로 비교해야 합니다(`==` 는 timezone·monotonic clock 까지 비교).
- `fmt.Errorf("%w: ...", ErrBadCast, ...)` 의 `%w` 는 에러를 **감싸서(wrap)** 호출자가 `errors.Is(err, iceberg.ErrBadCast)` 로 확인할 수 있게 합니다.

## 3. 코드 읽기

| 파일 | 줄 | 무엇 |
|---|---|---|
| `literals.go` | 938- | `StringLiteral.To`: 타입별 `switch` |
| 같은 파일 | 1014-1031 | `TimestampNsType`, `TimestampTzNsType` case: 고칠 곳 |
| 같은 파일 | 836-841 | `TimestampLiteral.To` 의 #1533 범위 검사 (에러 메시지 스타일 참고) |
| `literals_test.go` | | 기존 `StringLiteral` 변환 테스트들 |

## 4. 수정

두 case 의 return 을 헬퍼 호출로 바꿉니다.

```diff
 	case TimestampNsType:
 		...
-		return TimestampNsLiteral(TimestampNano(tm.UTC().UnixNano())), nil
+		return timestampNsFromTime(tm, s)
 	case TimestampTzNsType:
 		...
-		return TimestampNsLiteral(TimestampNano(tm.UTC().UnixNano())), nil
+		return timestampNsFromTime(tm, s)
```

`StringLiteral.To` 아래에 헬퍼를 추가합니다.

```go
// timestampNsFromTime rejects instants outside the int64 nanosecond range,
// where UnixNano silently wraps.
func timestampNsFromTime(tm time.Time, s StringLiteral) (Literal, error) {
	ns := tm.UnixNano()
	if !time.Unix(0, ns).Equal(tm) {
		return nil, fmt.Errorf("%w: timestamp '%s' is outside the nanosecond range", ErrBadCast, s)
	}

	return TimestampNsLiteral(TimestampNano(ns)), nil
}
```

`tm.UTC()` 를 뺀 이유: Unix 시간은 timezone 과 무관해서 `UnixNano()` 결과가 같습니다. 원래 코드와 결과도 같습니다.

## 5. 테스트

`literals_test.go`(`package iceberg_test`) 에 추가합니다. 경계값을 넣어서 범위 안의 값은 계속 통과하는지도 확인합니다.

```go
func TestStringToTimestampNsOutOfRange(t *testing.T) {
	for _, tc := range []struct {
		typ iceberg.Type
		in  string
	}{
		{iceberg.PrimitiveTypes.TimestampNs, "1600-01-01T00:00:00"},
		{iceberg.PrimitiveTypes.TimestampNs, "2500-01-01T00:00:00"},
		{iceberg.PrimitiveTypes.TimestampTzNs, "1600-01-01T00:00:00+00:00"},
		{iceberg.PrimitiveTypes.TimestampTzNs, "2500-01-01T00:00:00+00:00"},
	} {
		_, err := iceberg.NewLiteral(tc.in).To(tc.typ)
		assert.ErrorIs(t, err, iceberg.ErrBadCast, "%s -> %s", tc.in, tc.typ)
	}

	// boundaries of the int64 nanosecond range still convert
	for _, in := range []string{"1677-09-21T00:12:43.145224192", "2262-04-11T23:47:16.854775807"} {
		lit, err := iceberg.NewLiteral(in).To(iceberg.PrimitiveTypes.TimestampNs)
		require.NoError(t, err, in)
		assert.NotNil(t, lit)
	}
}
```

수정 전 결과(2026-10-24, 같은 내용의 재현 테스트):

```text
timestamp_ns: expected out-of-range error, got 2184-07-20 23:34:33.709551616
timestamptz_ns: expected out-of-range error, got 2184-07-20 23:34:33.709551616
1970-01-01 < 1600-01-01 evaluated to true
```

## 6. 실행

```bash
go test -count=1 -run TestStringToTimestampNsOutOfRange -v .
go test .
make lint
git commit -s -m "fix(literals): reject out-of-range strings cast to timestamp nanos"
```

## 7. 주의

- 경계 문자열 두 개는 `math.MinInt64`/`math.MaxInt64` 나노초에 해당하는 값입니다. 한 나노초만 넘겨도(`...192` → `...191`) 에러가 나는지 한 줄 더 넣으면 리뷰어가 좋아합니다.
- `DateLiteral`→ns 나 `TimestampLiteral`→ns 처럼 다른 경로에도 같은 문제가 없는지 `grep -n "UnixNano\|ToNanos" literals.go` 로 확인해 보세요.
