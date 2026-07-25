# Ruff Format

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · **Ruff Format** · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Formats changed Python files with Ruff's formatter, using the same bundled WASM workspace as
[Ruff Lint](python-lint.md) (no Python or `pip` install needed). Read-only in check mode —
`warn` when a file's formatted output differs from its current content — and rewrites files in
place when `ctx.write` is true, so CI can auto-commit the result, mirroring the
[Prettier](prettier.md) and [C++ Format](cpp-format.md) checks.

For each target file, `workspace.format(content)` is compared against the file's current
content. A syntax error makes Ruff's formatter throw; that is treated as a lint concern, not a
formatting failure, so the file is logged and skipped rather than counted as unformatted or
failing the run.

| Property | Value |
|---|---|
| Display name | `Ruff Format` |
| Phase | `format` |
| CLI command | `npx pr-checkmate python-format` |
| Config key | `python.ruff.format` |
| Toolchain | Bundled |
| Source | `src/core/checks/languages/python-format.ts` |

## When it applies

All of the following must hold:

1. `python.enabled` is not `false`
2. `python.ruff.format` is not `false`
3. `ctx.languages` includes `python`

Target files are `*.py`, resolved through the plain (not source-path-scoped) target resolver —
**`sourcePath` does not narrow which Python files are formatted**; only delta mode and
`ignoreDirs` do. If no target files are found, the check passes without loading the Ruff
workspace.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `python.enabled` | boolean | `true` | Set `false` to disable every Python check (lint, format, and types together) |
| `python.ruff.format` | boolean | `true` | Set `false` to skip just this check, leaving [Ruff Lint](python-lint.md) and [Python Types](python-typecheck.md) running |

### Example

Disable just this check while keeping Ruff linting and type checks on:

```json
{
  "python": {
    "ruff": { "format": false }
  }
}
```

## Disabling

```json
{ "python": { "ruff": { "format": false } } }
```

Or:

```json
{ "severity": { "Ruff Format": "off" } }
```

## Notes

- Shares its Ruff workspace setup with [Ruff Lint](python-lint.md) (`ruff-shared.ts`), so the
  two checks always agree on what "formatted" means.
- `ctx.write = false` (the default report-only mode): never mutates files, logs a warning
  listing how many files need formatting, and returns
  `<N> file(s) need formatting (run with write to fix)`.
- `ctx.write = true`: rewrites every unformatted file in place and returns `pass` with
  `reformatted <N> file(s)`.
- A file with a syntax error is logged as `Could not format <path> (syntax error?)` and skipped
  — it does not fail this check, since diagnosing the syntax error is [Ruff Lint](python-lint.md)'s
  job.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · **Ruff Format** · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
