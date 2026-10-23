# literals: string cast to decimal silently rounds extra fractional digits

- Issue: 없음 (2026-10-24 검색: PR "decimal string scale", "decimal FromString", "StringLiteral", issue "decimal scale literal", "decimal round" → #2092(DecimalLiteral→Int, 해결됨), #1028, #1129(manifest decimal precision, open) 는 다른 문제)
- Verified against main `7768540`
- **먼저 방향 질문**: Java 는 scale 이 정확히 같지 않으면 모두 거부합니다("1.5" → decimal(9,2) 도 거부). Go 에서 그대로 따르면 지금 되는 "1.5" 가 깨집니다. 어느 쪽으로 할지 메인테이너에게 먼저 물어보세요.

## Issue 본문 초안

### Apache Iceberg version

main (development)

### Please describe the bug 🐞

`StringLiteral.To(DecimalType)` (`literals.go:1040-1047`) calls `decimal128.FromString(s, precision, scale)`, which **rounds** a string with more fractional digits than the column scale and returns no error. `"1.555"` cast to `decimal(9,2)` becomes `1.56`.

A rounded bound changes predicate results:

| filter on `d decimal(9,2)` | bound as | row `d = 1.56` |
|---|---|---|
| `d > '1.555'` | `d > 1.56` | not matched (should match) |
| `d = '1.555'` | `d = 1.56` | matched (should not) |

The JSON expression parser (`ParseExpr`, e.g. REST scan planning) passes decimal literals as strings, so it hits this path.

Java's `StringLiteral.to(DECIMAL)` returns `null` (binding fails) unless the string's scale equals the type's scale:

```java
BigDecimal decimal = new BigDecimal(value().toString());
if (scale == decimal.scale()) { return new DecimalLiteral(decimal); }
return null;
```

### Question

Two options:

- (a) **Match Java exactly**: reject any scale mismatch. This also rejects `"1.5"` for `decimal(9,2)`, which works today, so it is a behaviour change for callers.
- (b) **Reject only lossy casts**: allow `"1.5"` → `1.50`, reject `"1.555"` → `1.56`. This fixes the wrong results without breaking existing callers, but it is more permissive than Java.

I lean towards (b). Which would you prefer?

### Repro

```text
=== RUN   TestStringToDecimalRejectsRounding
    expected scale-mismatch error, got 1.56
    1.56 > 1.555 evaluated to false
    1.56 == 1.555 evaluated to true
--- FAIL
```
