# 수정 가이드: 문자열 → decimal 변환이 조용히 반올림함

난이도: ★★★☆☆ (코드는 짧지만 동작 범위를 메인테이너와 먼저 합의해야 함)

## 1. 문제를 한 줄로

`decimal(9,2)` 컬럼에 `'1.555'` 로 필터를 걸면, Go 는 이 값을 `1.56` 으로 **반올림해서** 비교합니다. 그래서 `d > '1.555'` 가 `d = 1.56` 인 행을 놓칩니다.

## 2. 먼저 할 일: issue 에 방향 질문

`issue.md` 의 (a)/(b) 질문을 올리고 답을 기다립니다. 아래 수정은 (b) 기준입니다. (a) 로 정해지면 4-2 를 보세요.

## 3. Go 개념

- **`math/big.Rat`**: 임의 정밀도 유리수입니다. `new(big.Rat).SetString("1.555")` 는 정확히 1555/1000 을 만듭니다. 파싱에 실패하면 `(nil, false)` 를 돌려줍니다.
- **`decimal128.Num`**: arrow-go 의 128-bit 정수입니다. decimal 은 "정수(unscaled) × 10^-scale" 로 저장합니다. 1.56 (scale 2) = 156.
- 정확히 같은지 확인하는 방법: `unscaled / 10^scale` 을 `big.Rat` 으로 만들어 원래 문자열의 `big.Rat` 과 `Cmp` 합니다. `Cmp` 결과가 0 이면 같은 값입니다.

## 4. 수정

`literals.go` `StringLiteral.To` 의 `case DecimalType:` (`math/big` 은 이미 import 되어 있음)

### 4-1. 안 (b): 손실이 생기는 변환만 거부

```diff
 	case DecimalType:
 		n, err := decimal128.FromString(string(s), int32(t.precision), int32(t.scale))
 		if err != nil {
 			return nil, fmt.Errorf("%w: casting '%s' to %s - %s",
 				ErrBadCast, s, typ, err.Error())
 		}
+		// FromString rounds extra fractional digits; a rounded bound changes
+		// predicate results, so reject it.
+		exact, ok := new(big.Rat).SetString(string(s))
+		got := new(big.Rat).SetFrac(n.BigInt(), new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(t.scale)), nil))
+		if !ok || exact.Cmp(got) != 0 {
+			return nil, fmt.Errorf("%w: casting '%s' to %s - value does not fit scale %d",
+				ErrBadCast, s, typ, t.scale)
+		}
 
 		return DecimalLiteral{Val: n, Scale: t.scale}, nil
```

### 4-2. 안 (a): Java 와 똑같이(scale 이 정확히 일치해야 함)

문자열에서 소수점 아래 자릿수를 세어 `t.scale` 과 비교합니다. 지수 표기(`1.5e1`)가 들어올 수 있으면 직접 세기보다 `big.Rat` 를 쓰는 편이 안전합니다. 이 경우 기존 테스트 중 `"1.5"` 같은 값을 쓰는 것이 있는지 `grep -n 'DecimalTypeOf' literals_test.go exprs_test.go` 로 먼저 확인하세요.

## 5. 테스트

`literals_test.go`(`package iceberg_test`) 에 추가합니다.

```go
func TestStringToDecimalRejectsRounding(t *testing.T) {
	typ := iceberg.DecimalTypeOf(9, 2)

	_, err := iceberg.NewLiteral("1.555").To(typ)
	assert.ErrorIs(t, err, iceberg.ErrBadCast)

	// exact values, including ones with fewer fractional digits, still convert
	for _, in := range []string{"1.55", "1.5", "-3", "1.550"} {
		_, err := iceberg.NewLiteral(in).To(typ)
		assert.NoError(t, err, in)
	}
}
```

`"1.550"` 은 자릿수는 많지만 값의 손실이 없으므로 안 (b) 에서는 통과합니다. 안 (a) 라면 이 줄은 에러 쪽으로 옮깁니다.

수정 전 결과(2026-10-24, 필터 평가까지 포함한 재현 테스트):

```text
expected scale-mismatch error, got 1.56
1.56 > 1.555 evaluated to false
1.56 == 1.555 evaluated to true
```

## 6. 실행

```bash
go test -count=1 -run TestStringToDecimal -v .
go test . ./table/...
make lint
git commit -s -m "fix(literals): reject strings that lose digits when cast to decimal"
```

## 7. 주의

- 정밀도 초과(`"12345678.9"` → `decimal(9,2)`)는 `FromString` 이 이미 에러를 냅니다. 이번 수정 범위가 아닙니다.
- 필터 바인딩이 실패하면 사용자는 에러를 보게 됩니다. 조용히 틀린 결과를 내는 것보다 낫다는 게 PR 의 논리이니, 본문에 이 점을 분명히 적으세요.
