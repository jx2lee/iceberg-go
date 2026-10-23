Branch: `fix/view-builder-reuse-base-version`

Title:

```
fix(view): reuse an existing base version instead of adding a duplicate
```

Body:

```markdown
## Description

Fixes #<issue>

`addVersion` found an identical existing version, but only returned early when
that version had been added in the same builder. A version from the base
metadata was appended again, so `Build` failed with `duplicate version-id`.
Re-applying an unchanged definition, or reverting to an earlier one, therefore
failed.

Match Java's `ViewMetadata.Builder.addVersionInternal`: always return the
existing ID, and set `lastAddedVersionID` only when the version was added in
this builder.

## Testing

- `TestSetCurrentVersionReusesBaseVersion`: same definition on top of a base
  is a no-op (one version, no changes).
- `TestSetCurrentVersionRevertsToEarlierVersion`: reverting makes version 1
  current again without adding a version.

Both fail on main and pass with this change; `go test ./view/...` passes.
```

Checklist: `make lint`, `go test ./view/...`, `git commit -s`.
