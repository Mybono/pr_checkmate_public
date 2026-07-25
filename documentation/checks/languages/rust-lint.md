# Clippy

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · **Clippy** · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Lints Rust code with [Clippy](https://doc.rust-lang.org/clippy/) (`cargo clippy --quiet -- -D
warnings`), so any Clippy warning is treated as a build-breaking diagnostic in the check's own eyes,
regardless of what the run itself then does with that outcome.

Clippy compiles the whole crate, so it always runs over the crate as a whole rather than a file list.
The changed-file list (`*.rs`) only gates **whether** the check runs at all — if nothing Rust changed,
it passes without invoking `cargo clippy`. This mirrors [Go Vet](go-lint.md).

| Property | Value |
|---|---|
| Display name | `Clippy` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate rust-lint` |
| Config key | `rust` |
| Toolchain | Runner — requires the Rust toolchain (`cargo` + the `clippy` component) on the runner's `PATH`; not bundled |
| Source | `src/core/checks/languages/rust-lint.ts` |

## When it applies

Both conditions must hold:

1. `rust.enabled` is not `false`
2. Rust is detected in the repository

Rust is detected by the presence of a `Cargo.toml` file or at least one tracked `.rs` file.

In a pull request the file set is the diff between base and head SHA, filtered to `*.rs`; outside a PR
context it falls back to every tracked `.rs` file. Either way the list honours `ignoreDirs`. No Rust
files in scope is a `pass`, not a `skip`, and `cargo clippy` is never invoked.

If `cargo` (or the `clippy` component) is missing from the runner, the check returns `skip` rather than
`fail` or `warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `rust.enabled` | boolean | `true` | Set `false` to skip **both** Rust checks |

**`rust.enabled` is shared** between this check and [Rustfmt](rust-format.md) — there is a single
`rust` block in `pr-checkmate.json` covering both the lint and the format sub-checks, not one key per
check. Turning it off disables both at once; use `severity` to disable only one of the two.

As with every check, the universal severity override also applies, keyed by each check's own display
name:

```json
{ "severity": { "Clippy": "error" } }
```

### Example

Promote Clippy findings to a hard failure while leaving [Rustfmt](rust-format.md) at its default
severity:

```json
{
  "rust": { "enabled": true },
  "severity": { "Clippy": "error" }
}
```

## Disabling

```json
{ "rust": { "enabled": false } }
```

This also disables [Rustfmt](rust-format.md). To disable only this check:

```json
{ "severity": { "Clippy": "off" } }
```

## Notes

- A missing `cargo` binary is detected on the `execa` result (`code: 'ENOENT'`), not via
  `try`/`catch` — execa v9 with `reject: false` resolves rather than throws when the binary is absent.
- Clippy is invoked with `-D warnings`, so a plain warning-level lint already produces a non-zero exit
  code — Clippy itself, not just pr-checkmate's severity mapping, treats warnings as failures here.
- Only stderr lines matching `/\b(warning|error)\b/` are kept for the log and the count, filtering out
  Clippy's other build noise.
- On findings, the log prints the first 10 matching lines, then `... and N more`; the returned summary
  is a diagnostic count, e.g. `3 clippy diagnostic(s)`, falling back to `unknown clippy diagnostic(s)`
  if no line matched the pattern despite a non-zero exit.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · **Clippy** · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
