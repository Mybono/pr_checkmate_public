# Go Format

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · **Go Format** · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Formats changed Go files with `gofmt`. Check mode is read-only (`gofmt -l`, which lists files whose
formatting differs from `gofmt`'s output); when run with `write`, it rewrites those files in place
with `gofmt -w`, mirroring [Prettier](prettier.md), [C++ Format](cpp-format.md), and the other format
checks — so CI can auto-commit the fix.

Unlike [Go Vet](go-lint.md), which always vets the whole module, this check genuinely operates only on
the changed `*.go` files.

| Property | Value |
|---|---|
| Display name | `Go Format` |
| Phase | `format` |
| CLI command | `npx pr-checkmate go-format` |
| Config key | `go` |
| Toolchain | Runner — requires the Go toolchain (`gofmt`, bundled with `go`) on the runner's `PATH`; not bundled with pr-checkmate |
| Source | `src/core/checks/languages/go-format.ts` |

## When it applies

Both conditions must hold:

1. `go.enabled` is not `false`
2. Go is detected in the repository

Go is detected by the presence of a `go.mod` file or at least one tracked `.go` file.

In a pull request the file set is the diff between base and head SHA, filtered to `*.go`; outside a
PR context it falls back to every tracked `.go` file. Either way the list honours `ignoreDirs`. No Go
files in scope is a `pass`, not a `skip`.

If the `gofmt` binary itself is missing from the runner, the check returns `skip` rather than `fail`
or `warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `go.enabled` | boolean | `true` | Set `false` to skip **both** Go checks |

**`go.enabled` is shared** between this check and [Go Vet](go-lint.md) — there is a single `go` block
in `pr-checkmate.json` covering both the lint and the format sub-checks, not one key per check. Turning
it off disables both at once; use `severity` to disable only one of the two.

As with every check, the universal severity override also applies, keyed by each check's own display
name:

```json
{ "severity": { "Go Format": "off" } }
```

### Example

Disable auto-formatting entirely while keeping [Go Vet](go-lint.md) active:

```json
{
  "go": { "enabled": true },
  "severity": { "Go Format": "off" }
}
```

## Disabling

```json
{ "go": { "enabled": false } }
```

This also disables [Go Vet](go-lint.md). To disable only this check:

```json
{ "severity": { "Go Format": "off" } }
```

## Notes

- A missing `gofmt` binary is detected on the `execa` result (`code: 'ENOENT'`), not via
  `try`/`catch` — execa v9 with `reject: false` resolves rather than throws when the binary is absent.
- Without `write`, the outcome is `warn` — `"N file(s) need formatting (run with write to fix)"` — and
  the first 10 offending paths are logged.
- With `write`, `gofmt -w` is run against exactly the files `gofmt -l` flagged (not the full target
  list). If the rewrite itself fails, the check still returns `warn`, with the rewrite's stderr logged
  — a formatter that can't write is advisory, never blocking.
- A successful rewrite returns `pass('reformatted N file(s)')`.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · **Go Format** · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
