Branch: `fix/string-decimal-no-rounding`

Title (안 (b) 기준):

```
fix(literals): reject strings that lose digits when cast to decimal
```

Body:

```markdown
## Description

Fixes #<issue>

`StringLiteral.To(DecimalType)` used `decimal128.FromString`, which rounds
extra fractional digits, so `d > '1.555'` on `decimal(9,2)` was bound as
`d > 1.56` and dropped rows equal to 1.56.

Return `ErrBadCast` when the parsed value is not exactly equal to the string.
A string with fewer fractional digits than the scale (`'1.5'` for scale 2) is
still accepted, as before; Java rejects that case too, see the discussion in
the issue.

## Testing

`TestStringToDecimalRejectsRounding` fails on main and passes with this
change; existing decimal literal tests still pass.
```

Checklist: `make lint`, `go test .`, `go test ./table/...`(필터 경로 확인), `git commit -s`.
