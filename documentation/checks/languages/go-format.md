# Go Format

[Docs](../../README.md) · [Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) ·
[Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) ·
[Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) ·
[ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · **Go Format** · [Clippy](rust-lint.md) ·
[Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) ·
[PHP CS Fixer](php-format.md) · [ShellCheck](shellcheck.md)

---

## Overview

Formats changed Go files with gofmt. Check mode is read-only; when run with `write`, it rewrites the
files in place, mirroring [Prettier](prettier.md), [C++ Format](cpp-format.md), and the other format
checks — so CI can auto-commit the fix.

**No Go toolchain required.** The check prefers the runner's own `gofmt` when there is one, and falls
back to a bundled WASM build of gofmt (`@wasm-fmt/gofmt`, ~285 KB) otherwise. That order is
deliberate: gofmt's output is a function of the Go release, and the release a repository agrees with is
the one installed beside it — reaching for the bundled copy first would mean reformatting a Go 1.18
tree by current rules in `write` mode, with this check and the repository's own CI rewriting the same
files in opposite directions.

Unlike [Go Vet](go-lint.md), which always vets the whole module, this check genuinely operates only on
the changed `*.go` files.

| Property | Value |
|---|---|
| Display name | `Go Format` |
| Phase | `format` |
| CLI command | `npx pr-checkmate go-format` |
| Config key | `go` |
| Toolchain | Bundled (`@wasm-fmt/gofmt`), with the runner's own `gofmt` preferred when present |
| Source | `src/core/checks/languages/go-format.ts` |

## When it applies

Both conditions must hold:

1. `go.enabled` is not `false`
2. Go is detected in the repository

Go is detected by the presence of a `go.mod` file or at least one tracked `.go` file.

In a pull request the file set is the diff between base and head SHA, filtered to `*.go`; outside a
PR context it falls back to every tracked `.go` file. Either way the list honours `ignoreDirs`. No Go
files in scope is a `pass`, not a `skip`.

There is no runner dependency to skip for. A `skip` is reached only when git cannot list files, or in
the pathological case where the runner has no `gofmt` **and** the bundled formatter could not be
loaded — which means a partial install of pr-checkmate itself.

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

- An absent system `gofmt` is detected on the `execa` result (`code: 'ENOENT'`), not via
  `try`/`catch` — execa v9 with `reject: false` resolves rather than throws when the binary is absent.
  That result is what selects the bundled route; it is not an error condition.
- On the system route, `gofmt -l` lists the files whose formatting differs, and `gofmt -w` rewrites
  exactly those (not the full target list). On the bundled route each file is formatted in memory and
  compared, because the WASM build is a library and reports nothing about what it changed.
- Without `write`, the outcome is `warn` — `"N file(s) need formatting (run with write to fix)"` — and
  the first 10 offending paths are logged. Nothing on disk is touched.
- If a rewrite fails, the check still returns `warn`, with the failure logged — a formatter that
  can't write is advisory, never blocking.
- A file gofmt cannot **parse** is logged and skipped, not counted as needing formatting: gofmt parses
  before it formats, so that is a syntax error and [Go Vet](go-lint.md)'s finding to report.
- A successful rewrite returns `pass('reformatted N file(s)')`.
- The bundled formatter is fetched as the `go` install profile when it is not already present. `go
  vet` is deliberately **not** part of that profile — the Go toolchain is not ours to ship, so
  [Go Vet](go-lint.md) remains a runner dependency.
