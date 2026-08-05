# C# Format

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · **C# Format** · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · [ShellCheck](shellcheck.md)

---

## Overview

Formats and verifies C# code with `dotnet format`, which also runs the project's configured analyzers
and whitespace/style rules — not just brace placement. Check mode is read-only
(`dotnet format --verify-no-changes`); when run with `write`, it rewrites in place with
`dotnet format`, mirroring [Prettier](prettier.md), [Go Format](go-format.md), and the other format
checks — so CI can auto-commit the fix.

`dotnet format` operates on the project or solution — it needs a prior `dotnet restore` to resolve the
project's dependencies — so it always runs over the whole project rather than a file list. The
changed-file list (`*.cs`) only gates **whether** the check runs at all — if nothing C# changed, it
passes without invoking `dotnet format`.

| Property | Value |
|---|---|
| Display name | `C# Format` |
| Phase | `format` |
| CLI command | `npx pr-checkmate csharp-format` |
| Config key | `csharp` |
| Toolchain | Runner — requires the .NET SDK (`dotnet format`) on the runner's `PATH`; not bundled |
| Source | `src/core/checks/languages/csharp-format.ts` |

## When it applies

Both conditions must hold:

1. `csharp.enabled` is not `false`
2. C# is detected in the repository

C# is detected by the presence of a `*.csproj` or `*.sln` file at the repository root (a glob marker,
since C# projects have no single fixed marker filename) or at least one tracked `.cs` file.

In a pull request the file set is the diff between base and head SHA, filtered to `*.cs`; outside a PR
context it falls back to every tracked `.cs` file. Either way the list honours `ignoreDirs`. No C#
files in scope is a `pass`, not a `skip`, and `dotnet format` is never invoked.

If the `dotnet` binary (the .NET SDK) is missing from the runner, the check returns `skip` rather than
`fail` or `warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `csharp.enabled` | boolean | `true` | Set `false` to skip the check |

There is no other per-check configuration — `dotnet format` reads the project's own `.editorconfig`
and analyzer settings, the same way it would outside pr-checkmate.

As with every check, the universal severity override also applies:

```json
{ "severity": { "C# Format": "off" } }
```

### Example

```json
{
  "csharp": { "enabled": false }
}
```

## Disabling

```json
{ "csharp": { "enabled": false } }
```

Or remove it from the run without touching the `csharp` block:

```json
{ "severity": { "C# Format": "off" } }
```

## Notes

- A missing `dotnet` binary is detected on the `execa` result (`code: 'ENOENT'`), not via
  `try`/`catch` — execa v9 with `reject: false` resolves rather than throws when the binary is absent.
- Because `dotnet format --verify-no-changes` needs a restored project, a repository whose dependencies
  were never restored on the runner can surface as a formatting failure even when the code itself is
  correctly formatted — restore the project before running pr-checkmate in CI.
- Without `write`, the outcome is `warn` —
  `"C# code needs formatting (run with write to fix)"`.
- With `write`, `dotnet format` (no `--verify-no-changes`) is run over the whole project again. If that
  rewrite itself fails, the check still returns `warn`, with the rewrite's stderr logged — a formatter
  that can't write is advisory, never blocking.
- A successful rewrite returns `pass('reformatted C# code')`.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · **C# Format** · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · [ShellCheck](shellcheck.md)
