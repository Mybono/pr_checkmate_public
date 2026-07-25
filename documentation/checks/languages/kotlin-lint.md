# ktlint

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · **ktlint** · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Lints changed Kotlin files with [ktlint](https://pinterest.github.io/ktlint/), reporting style and
formatting violations. Glob: `*.kt`, `*.kts`.

The changed-file list is passed straight to ktlint (`ktlint --reporter=plain <files>`), so — unlike
[Go Vet](go-lint.md) or [Clippy](rust-lint.md), which analyse the whole module or crate — this check
genuinely limits its analysis to the files in the diff.

| Property | Value |
|---|---|
| Display name | `ktlint` |
| Phase | `informational` |
| CLI command | none — runs only as part of a full run |
| Config key | `kotlin` |
| Toolchain | Runner — requires the `ktlint` binary on the runner's `PATH`; not bundled |
| Source | `src/core/checks/languages/kotlin-lint.ts` |

## When it applies

Both conditions must hold:

1. `kotlin.enabled` is not `false`
2. Kotlin is detected in the repository

Kotlin is detected by the presence of a marker file (`build.gradle`, `build.gradle.kts`,
`settings.gradle`, `settings.gradle.kts`, or `AndroidManifest.xml`) or at least one tracked `.kt`,
`.kts`, or `.java` file.

In a pull request the file set is the diff between base and head SHA, filtered to the Kotlin globs;
outside a PR context it falls back to every tracked file matching those globs. Either way the list
honours `ignoreDirs`. No Kotlin files in scope is a `pass`, not a `skip`.

If the `ktlint` binary itself is missing from the runner, the check returns `skip` rather than `fail`
or `warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `kotlin.enabled` | boolean | `true` | Set `false` to skip the check |

There is no other per-check configuration — ktlint reads its own `.editorconfig` rules, if present, the
same way it would outside pr-checkmate.

As with every check, the universal severity override also applies:

```json
{ "severity": { "ktlint": "warn" } }
```

### Example

```json
{
  "kotlin": { "enabled": false }
}
```

## Disabling

```json
{ "kotlin": { "enabled": false } }
```

Or remove it from the run without touching the `kotlin` block:

```json
{ "severity": { "ktlint": "off" } }
```

## Notes

- There is no dedicated CLI command for this check (`npx pr-checkmate ktlint` does not exist) — it
  only runs as part of a full run (`npx pr-checkmate` / `npx pr-checkmate all`).
- A missing binary is detected on the `execa` result (`code: 'ENOENT'`), not via `try`/`catch` — execa
  v9 with `reject: false` resolves rather than throws when the binary is absent.
- Output is the union of stdout and stderr, trimmed; an exit code of `0` or empty output is a `pass`.
- On violations, the log prints the first 10 lines, then `... and N more`; the returned summary is a
  bare issue count, e.g. `3 issue(s)`.
- This check never writes files — ktlint's autocorrect mode is not invoked. Fixing findings is manual.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · **ktlint** · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
