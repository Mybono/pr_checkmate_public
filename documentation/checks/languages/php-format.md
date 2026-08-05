# PHP CS Fixer

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · **PHP CS Fixer** · [ShellCheck](shellcheck.md)

---

## Overview

Formats changed PHP files with [PHP-CS-Fixer](https://cs.symfony.com/) (`php-cs-fixer fix
--using-cache=no <files> --dry-run`). Check mode is read-only (`--dry-run`); when run with `write`, it
rewrites in place with the same command minus `--dry-run`, mirroring [Prettier](prettier.md),
[Go Format](go-format.md), and the other format checks — so CI can auto-commit the fix. Uses the
repository's own `.php-cs-fixer.php` config when present, the same way it would outside pr-checkmate.

The changed-file list is passed straight to PHP-CS-Fixer, so — unlike [Clippy](rust-lint.md) or
[C# Format](csharp-format.md), which analyse the whole crate or project — this check genuinely limits
its analysis to the files in the diff. Glob: `*.php`.

| Property | Value |
|---|---|
| Display name | `PHP CS Fixer` |
| Phase | `format` |
| CLI command | `npx pr-checkmate php-format` |
| Config key | `php` |
| Toolchain | Runner — requires the `php-cs-fixer` binary on the runner's `PATH`; not bundled |
| Source | `src/core/checks/languages/php-format.ts` |

## When it applies

Both conditions must hold:

1. `php.enabled` is not `false`
2. PHP is detected in the repository

PHP is detected by the presence of a `composer.json` file or at least one tracked `.php` file.

In a pull request the file set is the diff between base and head SHA, filtered to `*.php`; outside a
PR context it falls back to every tracked `.php` file. Either way the list honours `ignoreDirs`. No
PHP files in scope is a `pass`, not a `skip`.

If the `php-cs-fixer` binary itself is missing from the runner, the check returns `skip` rather than
`fail` or `warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `php.enabled` | boolean | `true` | Set `false` to skip the check |

There is no other per-check configuration — PHP-CS-Fixer reads its own `.php-cs-fixer.php`, if
present, the same way it would outside pr-checkmate.

As with every check, the universal severity override also applies:

```json
{ "severity": { "PHP CS Fixer": "off" } }
```

### Example

```json
{
  "php": { "enabled": false }
}
```

## Disabling

```json
{ "php": { "enabled": false } }
```

Or remove it from the run without touching the `php` block:

```json
{ "severity": { "PHP CS Fixer": "off" } }
```

## Notes

- A missing `php-cs-fixer` binary is detected on the `execa` result (`code: 'ENOENT'`), not via
  `try`/`catch` — execa v9 with `reject: false` resolves rather than throws when the binary is absent.
- The dry run always passes `--using-cache=no`, so a stale PHP-CS-Fixer cache on the runner cannot mask
  or fabricate a finding.
- Without `write`, the outcome is `warn` — `"PHP code needs formatting (run with write to fix)"`.
- With `write`, the same `fix --using-cache=no <files>` command is re-run without `--dry-run`. If that
  rewrite itself fails, the check still returns `warn`, with the rewrite's stderr logged — a formatter
  that can't write is advisory, never blocking.
- A successful rewrite returns `pass('reformatted PHP code')`.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · **PHP CS Fixer** · [ShellCheck](shellcheck.md)
