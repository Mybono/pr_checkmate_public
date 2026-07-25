# Python Types

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · **Python Types** · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Type-checks changed Python files — the Python analogue of [TypeScript](typecheck.md)'s
`tsc --noEmit`. Unlike Ruff, neither `mypy` nor `pyright` ships with pr-checkmate; this is a
runner-dependency check that shells out to whichever binary is available, and skips gracefully
when neither is installed.

Checker selection: if `python.typeChecker` is set, only that binary is tried. Otherwise the
check tries `mypy` first, then `pyright` — the first one that actually exists on the runner
wins. Absence is detected via the child process's `code: 'ENOENT'` on the result object (execa
v9 with `reject: false` resolves rather than throws on a missing binary), not a try/catch.

| Property | Value |
|---|---|
| Display name | `Python Types` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate python-typecheck` |
| Config key | `python.typecheck` |
| Toolchain | Runner (needs `mypy` or `pyright` installed on the runner — not bundled) |
| Source | `src/core/checks/languages/python-typecheck.ts` |

## When it applies

All of the following must hold:

1. `python.enabled` is not `false`
2. `python.typecheck` is not `false`
3. `ctx.languages` includes `python`

Target files are `*.py`, resolved through the plain (not source-path-scoped) target resolver —
**`sourcePath` does not narrow which files are type-checked**; only delta mode and `ignoreDirs`
do. If no target files are found, the check passes without invoking either type checker.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `python.enabled` | boolean | `true` | Set `false` to disable every Python check (lint, format, and types together) |
| `python.typecheck` | boolean | `true` | Set `false` to skip just this check |
| `python.typeChecker` | `"mypy"` \| `"pyright"` | unset in code (tries `mypy`, then `pyright`); the shipped template pins `"mypy"` | Force one specific type checker instead of autodetecting |

### Example

Force pyright instead of the default mypy-first order:

```json
{
  "python": {
    "typeChecker": "pyright"
  }
}
```

## Disabling

```json
{ "python": { "typecheck": false } }
```

Or:

```json
{ "severity": { "Python Types": "off" } }
```

## Notes

- Install the checker on the CI runner before this check can run anything — e.g.
  `pip install mypy` or `pip install pyright` in a setup step. Without one, the check reports
  `skip` naming the tool it looked for (`<forced> not installed` or `mypy/pyright not
  installed`), never `fail`.
- Output lines are filtered to those containing `error` or `warning` (case-insensitive) before
  logging, capped at 10 with `... and N more`. The summary line prefers the checker's own
  "found N errors" line when present, otherwise falls back to `<N> type issue(s)`.
- Always reports `warn` when issues are found, never `fail` — matching [Ruff Lint](python-lint.md)'s
  advisory-by-default design. Promote it with `severity: { "Python Types": "error" }` once a
  team is ready to gate on it.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · **Python Types** · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
