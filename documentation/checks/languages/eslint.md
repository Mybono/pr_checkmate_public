# ESLint

[Checks Index](../INDEX.md) · **ESLint** · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Lints changed TypeScript/JavaScript files with ESLint. The binary is resolved from the
project's own `node_modules` first (so a repo pinned to ESLint 8 with a legacy `.eslintrc.*`
keeps working), falling back to the ESLint bundled with pr-checkmate.

The resolved binary's major version decides how it is invoked: ESLint 9+ uses flat config and
ignores `--ext` (extensions come from the config's `files` glob); ESLint 8 and earlier is
launched with `--ext .ts,.tsx,.js,.jsx`, which flat config would silently misinterpret.

**Your own ESLint config wins.** If the repository contains any of these files, they are used
as-is and pr-checkmate's own rule set is not applied at all:

```text
eslint.config.mjs · eslint.config.js · .eslintrc.js · .eslintrc.json · .eslintrc
```

Only when none of those exists does the check pass `--config` pointing at pr-checkmate's own
bundled config (`eslint.config.mjs` for ESLint ≥9, `.eslintrc.js` for ESLint <9). That bundled
config file loads `pr-checkmate.json` itself and merges its own default rule set with the
`eslint.rules` / `eslint.ignorePatterns` blocks (see [Configuration](#configuration) below).

| Property | Value |
|---|---|
| Display name | `ESLint` |
| Phase | `blocking` |
| CLI command | `npx pr-checkmate lint` |
| Config key | `lint` |
| Toolchain | Bundled |
| Source | `src/core/checks/languages/lint.ts` |

## When it applies

`ctx.languages` includes `typescript` or `javascript` — detected via a `tsconfig.json` /
`package.json` marker file, or at least one tracked `.ts`/`.tsx`/`.js`/`.jsx`/`.mjs`/`.cjs`
file. There is no `enabled` flag on this check; use `severity: { "ESLint": "off" }` to switch
it off.

Target files are `*.ts`, `*.tsx`, `*.js`, `*.jsx`, restricted to the configured `sourcePath`(s).
In a PR, only files changed between `baseSha` and `headSha` are linted (added/copied/
modified/renamed only); outside a PR context every matching tracked file is scanned. If no
target files are found, the check passes without invoking ESLint.

`ctx.write` (auto-fix mode, e.g. `precommit`) adds `--fix`. The shared default ignore list
(`IGNORE_PATTERNS`) is always applied via `--ignore-pattern`, on top of either config path.

## Configuration

pr-checkmate's own ESLint config is generated from a top-level `eslint` block — **not** the
`lint` key the `PRCheckMateConfig` interface and this table's "Config key" row name — because
the bundled `eslint.config.mjs`/`.eslintrc.js` load `pr-checkmate.json` directly off disk as a
separate process and read `eslint.rules`/`eslint.ignorePatterns`. `npx pr-checkmate init`
writes exactly this shape, so treat `eslint` as the key that actually takes effect:

| Key | Type | Default | Meaning |
|---|---|---|---|
| `eslint.rules` | object | pr-checkmate's built-in rule set (`no-console: warn`, `eqeqeq: error`, `@typescript-eslint/no-floating-promises: error`, `prettier/prettier: error`, … — run `init` to see and edit the full list) | Rule overrides, merged over the defaults one rule at a time |
| `eslint.ignorePatterns` | string[] | `[]` | Glob patterns appended to the built-in ignores (`node_modules`, `dist`, `.git`, `build`, `coverage`, `.next`) |

Both keys are read only when the repository has none of its own ESLint config files — see
[Overview](#overview).

### Example

```json
{
  "eslint": {
    "rules": {
      "no-console": "off",
      "@typescript-eslint/no-explicit-any": "error"
    },
    "ignorePatterns": ["**/generated/**"]
  }
}
```

## Disabling

```json
{ "severity": { "ESLint": "off" } }
```

To keep it visible but non-blocking:

```json
{ "severity": { "ESLint": "warn" } }
```

## Notes

- The consumer's `typescript`-adjacent tooling is never touched by this check directly — only
  the ESLint binary and config are resolved this way; type errors are [TypeScript](typecheck.md)'s
  job.
- ESLint runs with `stdio: 'inherit'`, so its full report (including autofix summary when
  `--fix` is used) appears directly in the CI log.
- `--ext` is only ever passed on the legacy (ESLint <9) path — passing it under flat config
  would make ESLint fall back to its default JS parser for explicitly-listed files, silently
  breaking TypeScript linting.
- Exit code `1` means lint errors and is the only outcome that fails the run. Exit `2` is ESLint's
  fatal error — a config it cannot load, an unresolvable plugin, a parser it cannot construct — and
  is reported as `eslint failed to run (exit 2)` at `warn`, noting whether the config in play was
  the client's or pr-checkmate's. An incompatibility between our bundled config and a client's
  toolchain is not a defect in their code and must not gate their PR.

---

[Checks Index](../INDEX.md) · **ESLint** · [TypeScript](typecheck.md) · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
