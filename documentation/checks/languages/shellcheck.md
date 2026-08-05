# ShellCheck

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · **ShellCheck**

---

## Overview

Lints changed shell scripts with [ShellCheck](https://www.shellcheck.net/)
(`shellcheck -f gcc <files>`). Glob: `*.sh`, `*.bash`, `*.ksh` — matched by extension only, the same
convention every other language check in this table uses; an extensionless script identified only by
its shebang is not picked up.

The changed-file list is passed straight to ShellCheck, so — like [RuboCop](ruby-lint.md) and unlike
[Go Vet](go-lint.md) or [Clippy](rust-lint.md), which analyse the whole module or crate — this check
genuinely limits its analysis to the files in the diff. `-f gcc` produces one line per finding, ending
in a `[SCxxxx]` code, which both the log and the reported count read directly.

| Property | Value |
|---|---|
| Display name | `ShellCheck` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate shellcheck` |
| Config key | `shellcheck` |
| Toolchain | Runner — requires the `shellcheck` binary on the runner's `PATH`; not bundled |
| Source | `src/core/checks/languages/shellcheck.ts` |

## When it applies

`shellcheck.enabled` is not `false` — it's **on by default**. There is no language-detection gate:
the check simply resolves the target-file list, and an empty list (no shell scripts changed) is a
`pass`, not a `skip`.

In a pull request the file set is the diff between base and head SHA, filtered to
`*.sh`/`*.bash`/`*.ksh`; outside a PR context it falls back to every tracked file matching those
globs. Either way the list honours `ignoreDirs`.

If the `shellcheck` binary itself is missing from the runner, the check returns `skip` rather than
`fail` or `warn` — per the general rule that anything missing is skipped, not failed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `shellcheck.enabled` | boolean | `true` | Set `false` to skip the check |
| `shellcheck.minSeverity` | `'error' \| 'warning' \| 'info' \| 'style'` | ShellCheck's own default (every level) | Passed as `-S`; only findings at or above this level are reported |
| `shellcheck.ignore` | string[] | `[]` | ShellCheck codes to suppress (e.g. `"SC2086"`), passed as `-e` |

There is no dedicated `shellcheck.severity` key — findings always `warn` by default. As with every
check, the universal severity override still applies to promote it to a gate:

```json
{ "severity": { "ShellCheck": "error" } }
```

### Example

Only report warnings and above, and suppress a code the project has accepted:

```json
{
  "shellcheck": {
    "minSeverity": "warning",
    "ignore": ["SC2086"]
  }
}
```

## Disabling

```json
{ "shellcheck": { "enabled": false } }
```

Or remove it from the run without touching the `shellcheck` block:

```json
{ "severity": { "ShellCheck": "off" } }
```

## Notes

- A missing `shellcheck` binary is detected on the `execa` result (`code: 'ENOENT'`), not via
  `try`/`catch` — execa v9 with `reject: false` resolves rather than throws when the binary is absent.
- Output is the union of stdout and stderr, split into non-blank lines. The log prints the first 10,
  then `... and N more`. The reported summary is a plain line count (`N issue(s) in M file(s)`),
  since `-f gcc` gives one line per finding rather than a parseable summary line.
- This check never writes files — ShellCheck's own `--format=diff` autofix suggestions are not
  applied. Fixing findings is manual, or via `shellcheck -f diff` outside pr-checkmate.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · **ShellCheck**
