# Go Vet

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · **Go Vet** · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Vets Go code with `go vet ./...`, reporting suspicious constructs (unreachable code, wrong `Printf`
verbs, misuse of sync primitives, and the rest of `go vet`'s built-in analysers).

`go vet` operates per-package rather than per-file, so it always runs over the whole module
(`./...`). The changed-file list (`*.go`) only gates **whether** the check runs at all — if nothing
Go changed, it passes without invoking `go vet`. This is different from [Go Format](go-format.md),
which does act only on the changed files.

| Property | Value |
|---|---|
| Display name | `Go Vet` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate go-lint` |
| Config key | `go` |
| Toolchain | Runner — requires the Go toolchain (`go`) on the runner's `PATH`; not bundled |
| Source | `src/core/checks/languages/go-lint.ts` |

## When it applies

Both conditions must hold:

1. `go.enabled` is not `false`
2. Go is detected in the repository

Go is detected by the presence of a `go.mod` file or at least one tracked `.go` file.

In a pull request the file set is the diff between base and head SHA, filtered to `*.go`; outside a
PR context it falls back to every tracked `.go` file. Either way the list honours `ignoreDirs`. No Go
files in scope is a `pass`, not a `skip`, and `go vet` is never invoked.

If the `go` binary itself is missing from the runner, the check returns `skip` rather than `fail` or
`warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `go.enabled` | boolean | `true` | Set `false` to skip **both** Go checks |

**`go.enabled` is shared** between this check and [Go Format](go-format.md) — there is a single `go`
block in `pr-checkmate.json` covering both the lint and the format sub-checks, not one key per check.
Turning it off disables both at once; there is no way to keep one enabled and the other disabled via
this key (use `severity` for that instead, see below).

As with every check, the universal severity override also applies, keyed by each check's own display
name:

```json
{ "severity": { "Go Vet": "warn" } }
```

### Example

Keep the Go toolchain checks enabled but treat `go vet` findings as advisory only, while leaving
[Go Format](go-format.md) at its default severity:

```json
{
  "go": { "enabled": true },
  "severity": { "Go Vet": "warn" }
}
```

## Disabling

```json
{ "go": { "enabled": false } }
```

This also disables [Go Format](go-format.md). To disable only this check:

```json
{ "severity": { "Go Vet": "off" } }
```

## Notes

- A missing `go` binary is detected on the `execa` result (`code: 'ENOENT'`), not via `try`/`catch` —
  execa v9 with `reject: false` resolves rather than throws when the binary is absent.
- `go vet` writes diagnostics (and build errors) to stderr; stdout is appended after it. Both are
  combined, split into lines, and empty lines dropped.
- On findings, the log prints the first 10 diagnostic lines, then `... and N more`; the returned
  summary is a bare line count, e.g. `4 diagnostic line(s)`.
- Because `go vet` runs `./...`, a build failure anywhere in the module — not just in files touched by
  the PR — surfaces here as a diagnostic.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · **Go Vet** · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
