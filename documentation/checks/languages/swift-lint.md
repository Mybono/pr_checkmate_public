# SwiftLint

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · **SwiftLint** · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · [ShellCheck](shellcheck.md)

---

## Overview

Lints Swift files by shelling out to the `swiftlint` binary (`swiftlint lint --quiet --path
<cwd>`). SwiftLint is not bundled with pr-checkmate — it must already be on the runner's `PATH`.

Changed-file detection (`resolveTargetFiles`) is delta-aware, but it is only used as a **gate**:
if there are no changed Swift files, the check passes immediately without invoking `swiftlint`
at all. When it does run, `swiftlint` is invoked with `--path <cwd>` rather than the individual
target file list, so it always lints the **whole working directory**, not just the files that
changed — a real difference from the delta-mode behavior described for most checks in the
[Checks Index](../INDEX.md#delta-mode).

| Property | Value |
|---|---|
| Display name | `SwiftLint` |
| Phase | `informational` |
| CLI command | none — only runs as part of a full run |
| Config key | `swift` |
| Toolchain | Runner (`swiftlint`, pre-installed on `macos-latest`) |
| Source | `src/core/checks/languages/swift-lint.ts` |

## When it applies

Both of the following must hold:

1. `swift.enabled` is not `false`
2. `ctx.languages` includes `swift` — detected via a `Package.swift` / `Package.resolved`
   marker file, or at least one tracked `.swift` file

The `*.swift` target list used for the gate above is resolved through the plain (not
source-path-scoped) target resolver — `sourcePath` does not affect this check either way, since
`swiftlint` itself always scans the whole `cwd` once triggered.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `swift.enabled` | boolean | `true` | Set `false` to skip the check |

There is nothing else to configure through pr-checkmate — SwiftLint's own rule configuration
(`.swiftlint.yml`) is entirely up to the project.

## Disabling

```json
{ "swift": { "enabled": false } }
```

Or:

```json
{ "severity": { "SwiftLint": "off" } }
```

## Notes

- No dedicated CLI command — SwiftLint only runs as part of a full `npx pr-checkmate` /
  `npx pr-checkmate all` run, alongside [Merge Conflict](../git/merge-conflict.md), `ktlint`, and
  `Dead Code`.
- On GitHub-hosted `ubuntu-latest` runners `swiftlint` is absent by default; the check reports
  `skip('swiftlint not installed')` rather than failing. On `macos-latest` it comes
  pre-installed.
- Absence is detected via the child process's `code: 'ENOENT'` on the result object, not a
  try/catch — execa v9 with `reject: false` resolves rather than throws when the binary itself
  can't be found.
- Output lines are filtered to those containing `: warning:` or `: error:`; up to 10 are logged,
  then `... and N more`. A non-zero exit with no parseable output is still treated as `pass` —
  empty output is trusted over the raw exit code.
- Always reports `warn` when issues are found, never `fail` — promote it with
  `severity: { "SwiftLint": "error" }` once a team is ready to gate on it.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · **SwiftLint** · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · [ShellCheck](shellcheck.md)
