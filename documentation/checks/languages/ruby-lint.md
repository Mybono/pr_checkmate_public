# RuboCop

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · **RuboCop** · [PHP CS Fixer](php-format.md)

---

## Overview

Lints changed Ruby files with [RuboCop](https://rubocop.org/) (`rubocop --format simple <files>`).
RuboCop covers both style/lint rules and layout, so it doubles as the formatting signal for Ruby —
there is no separate Ruby format check. Glob: `*.rb`, `*.rake`.

The changed-file list is passed straight to RuboCop, so — unlike [Go Vet](go-lint.md) or
[Clippy](rust-lint.md), which analyse the whole module or crate — this check genuinely limits its
analysis to the files in the diff. RuboCop uses the repository's own `.rubocop.yml` when one is
present, the same way it would outside pr-checkmate.

| Property | Value |
|---|---|
| Display name | `RuboCop` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate ruby-lint` |
| Config key | `ruby` |
| Toolchain | Runner — requires the `rubocop` gem on the runner's `PATH`; not bundled |
| Source | `src/core/checks/languages/ruby-lint.ts` |

## When it applies

Both conditions must hold:

1. `ruby.enabled` is not `false`
2. Ruby is detected in the repository

Ruby is detected by the presence of a `Gemfile` or `.rubocop.yml` file, or at least one tracked `.rb`
or `.rake` file.

In a pull request the file set is the diff between base and head SHA, filtered to `*.rb`/`*.rake`;
outside a PR context it falls back to every tracked file matching those globs. Either way the list
honours `ignoreDirs`. No Ruby files in scope is a `pass`, not a `skip`.

If the `rubocop` binary itself is missing from the runner, the check returns `skip` rather than `fail`
or `warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `ruby.enabled` | boolean | `true` | Set `false` to skip the check |

There is no other per-check configuration — RuboCop reads its own `.rubocop.yml`, if present, the same
way it would outside pr-checkmate.

As with every check, the universal severity override also applies:

```json
{ "severity": { "RuboCop": "error" } }
```

### Example

```json
{
  "ruby": { "enabled": false }
}
```

## Disabling

```json
{ "ruby": { "enabled": false } }
```

Or remove it from the run without touching the `ruby` block:

```json
{ "severity": { "RuboCop": "off" } }
```

## Notes

- A missing `rubocop` binary is detected on the `execa` result (`code: 'ENOENT'`), not via
  `try`/`catch` — execa v9 with `reject: false` resolves rather than throws when the binary is absent.
- Output is the union of stdout and stderr, split into non-blank lines. The log prints the first 10,
  then `... and N more line(s)`.
- The returned summary prefers RuboCop's own summary line (matched with `/offense/i`, e.g. `"3 files
  inspected, 5 offenses detected"`); if no such line is found, it falls back to a bare line count, e.g.
  `12 line(s) of output`.
- This check never writes files — RuboCop's `--autocorrect`/`-A` mode is not invoked. Fixing findings
  is manual, or via your own `rubocop -A` outside pr-checkmate.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · **RuboCop** · [PHP CS Fixer](php-format.md)
