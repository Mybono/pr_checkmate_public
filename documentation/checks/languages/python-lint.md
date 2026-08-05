# Ruff Lint

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · **Ruff Lint** · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · [ShellCheck](shellcheck.md)

---

## Overview

Lints changed Python files with [Ruff](https://docs.astral.sh/ruff/), bundled as a self-contained
WASM module (`@astral-sh/ruff-wasm-nodejs`) so it runs on a bare Node image with no Python or
`pip` install. Informational by design, matching how formatting checks roll out — it reports
issues without failing the build until a team promotes it via `severity`.

Each target file is read and passed to `workspace.check(content)`, which reports diagnostics
using Ruff's **built-in default rule set** (roughly pyflakes + pycodestyle) — the workspace is
always created with `Workspace.defaultSettings()`, so a project's own `ruff.toml` or
`pyproject.toml [tool.ruff]` is not read; rule selection is not currently configurable through
pr-checkmate.

| Property | Value |
|---|---|
| Display name | `Ruff Lint` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate python-lint` |
| Config key | `python.ruff.lint` |
| Toolchain | Bundled |
| Source | `src/core/checks/languages/python-lint.ts` |

## When it applies

All of the following must hold:

1. `python.enabled` is not `false`
2. `python.ruff.lint` is not `false`
3. `ctx.languages` includes `python` — detected via a `pyproject.toml` / `setup.py` /
   `setup.cfg` / `requirements.txt` / `Pipfile` marker file, or at least one tracked `.py`/`.pyi`
   file

Target files are `*.py`, resolved through the plain (not source-path-scoped) target resolver —
**`sourcePath` does not narrow which Python files are linted**; only delta mode (changed files
in a PR) and `ignoreDirs` do. If no target files are found, the check passes without loading the
Ruff workspace.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `python.enabled` | boolean | `true` | Set `false` to disable every Python check (lint, format, and types together) |
| `python.ruff.lint` | boolean | `true` | Set `false` to skip just this check, leaving [Ruff Format](python-format.md) and [Python Types](python-typecheck.md) running |

### Example

Disable just this check while keeping Ruff formatting and type checks on:

```json
{
  "python": {
    "ruff": { "lint": false }
  }
}
```

## Disabling

```json
{ "python": { "ruff": { "lint": false } } }
```

Or:

```json
{ "severity": { "Ruff Lint": "off" } }
```

## Notes

- Shares its Ruff workspace setup with [Ruff Format](python-format.md) (`ruff-shared.ts`), so
  lint and format always run against identical Ruff settings, and the WASM module is required
  lazily so it never loads in a non-Python repo.
- Up to 5 diagnostics per file are logged individually as `path:line CODE message`, with
  `... and N more` for the rest; every affected file still counts toward the summary regardless
  of how many of its diagnostics were printed.
- A file that was renamed or removed between the diff being computed and this check reading it
  is silently skipped rather than erroring.
- Always reports `warn` when issues are found, never `fail` — promote it with
  `severity: { "Ruff Lint": "error" }` once a team is ready to gate on it.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · **Ruff Lint** · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · [ShellCheck](shellcheck.md)
