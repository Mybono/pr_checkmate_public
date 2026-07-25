# Rustfmt

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · **Rustfmt** · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Formats Rust code with `cargo fmt`, run rather than a bare `rustfmt` invocation so the crate's
`edition` (from `Cargo.toml`) is respected. Check mode is read-only (`cargo fmt --check`); when run
with `write`, it rewrites in place with `cargo fmt`, mirroring [Prettier](prettier.md),
[Go Format](go-format.md), and the other format checks — so CI can auto-commit the fix.

Like [Clippy](rust-lint.md), `cargo fmt` operates over the whole crate rather than a file list. The
changed-file list (`*.rs`) only gates **whether** the check runs at all.

| Property | Value |
|---|---|
| Display name | `Rustfmt` |
| Phase | `format` |
| CLI command | `npx pr-checkmate rust-format` |
| Config key | `rust` |
| Toolchain | Runner — requires the Rust toolchain (`cargo`, which bundles `rustfmt`) on the runner's `PATH`; not bundled |
| Source | `src/core/checks/languages/rust-format.ts` |

## When it applies

Both conditions must hold:

1. `rust.enabled` is not `false`
2. Rust is detected in the repository

Rust is detected by the presence of a `Cargo.toml` file or at least one tracked `.rs` file.

In a pull request the file set is the diff between base and head SHA, filtered to `*.rs`; outside a PR
context it falls back to every tracked `.rs` file. Either way the list honours `ignoreDirs`. No Rust
files in scope is a `pass`, not a `skip`, and `cargo fmt` is never invoked.

If the `cargo` binary is missing from the runner, the check returns `skip` rather than `fail` or
`warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `rust.enabled` | boolean | `true` | Set `false` to skip **both** Rust checks |

**`rust.enabled` is shared** between this check and [Clippy](rust-lint.md) — there is a single `rust`
block in `pr-checkmate.json` covering both the lint and the format sub-checks, not one key per check.
Turning it off disables both at once; use `severity` to disable only one of the two.

As with every check, the universal severity override also applies, keyed by each check's own display
name:

```json
{ "severity": { "Rustfmt": "off" } }
```

### Example

Disable auto-formatting entirely while keeping [Clippy](rust-lint.md) active:

```json
{
  "rust": { "enabled": true },
  "severity": { "Rustfmt": "off" }
}
```

## Disabling

```json
{ "rust": { "enabled": false } }
```

This also disables [Clippy](rust-lint.md). To disable only this check:

```json
{ "severity": { "Rustfmt": "off" } }
```

## Notes

- A missing `cargo` binary is detected on the `execa` result (`code: 'ENOENT'`), not via
  `try`/`catch` — execa v9 with `reject: false` resolves rather than throws when the binary is absent.
- Because `cargo fmt --check` inspects the whole crate rather than the changed-file list, a
  pre-existing formatting drift elsewhere in the crate surfaces here too, not just drift introduced by
  the PR.
- Without `write`, the outcome is `warn` — `"Rust code needs formatting (run with write to fix)"`.
- With `write`, `cargo fmt` (no `--check`) is run over the whole crate again. If that rewrite itself
  fails, the check still returns `warn`, with the rewrite's stderr logged — a formatter that can't
  write is advisory, never blocking.
- A successful rewrite returns `pass('reformatted Rust code')`.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · **Rustfmt** · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
