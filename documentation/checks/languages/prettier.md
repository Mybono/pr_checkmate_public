# Prettier

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · **Prettier** · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Checks (or rewrites) formatting of changed TypeScript/JavaScript/JSON/Markdown files with
Prettier, bundled so no local install is required.

**Your own Prettier config wins.** If the repository contains any of these files, they are
passed straight to Prettier and pr-checkmate's own defaults are not merged in at all:

```text
.prettierrc · .prettierrc.json · .prettierrc.js · prettier.config.js
```

Only when none of those exists does the check write a temporary `.prettierrc.temp.json`
combining pr-checkmate's bundled defaults with the `prettier.rules` block from
`pr-checkmate.json` (see [Configuration](#configuration)), and pass it via `--config`.

Ignore rules follow a similar precedence: the project's own `.prettierignore` wins, then its
`.gitignore`, and only if neither exists does the check write a temporary ignore file from the
shared default ignore list. Whichever mode was used, the temporary file (config and/or ignore)
is deleted in a `finally` block.

`ctx.write` decides the run mode: `--write` rewrites files in place, `--check` (the default)
only reports which files are not formatted.

| Property | Value |
|---|---|
| Display name | `Prettier` |
| Phase | `format` |
| CLI command | `npx pr-checkmate prettier` |
| Config key | `prettier` |
| Toolchain | Bundled |
| Source | `src/core/checks/languages/prettier.ts` |

## When it applies

`ctx.languages` includes `typescript` or `javascript`. There is no `enabled` flag on this
check; use `severity: { "Prettier": "off" }` to switch it off.

Target files are `*.ts`, `*.tsx`, `*.js`, `*.jsx`, `*.json`, `*.md` — a broader glob than
[ESLint](eslint.md)'s, since Prettier also formats JSON and Markdown — restricted to the
configured `sourcePath`(s). In a PR, only changed files (added/copied/modified/renamed) are
checked; outside a PR, every matching tracked file is. If no target files are found, the check
passes without invoking Prettier.

## Configuration

The runtime config key is `prettier.rules` — read straight from `pr-checkmate.json` on disk,
the same key `npx pr-checkmate init` writes — not `prettier.config`, which appears in the
`PRCheckMateConfig` TypeScript interface but is not consulted by this check.

| Key | Type | Default | Meaning |
|---|---|---|---|
| `prettier.rules` | object | `{ "semi": true, "singleQuote": true, "trailingComma": "all", "bracketSpacing": true, "arrowParens": "avoid", "printWidth": 100 }` | Prettier options, merged over the defaults one key at a time |

Only consulted when the repository has none of its own Prettier config files — see
[Overview](#overview).

### Example

```json
{
  "prettier": {
    "rules": {
      "printWidth": 120,
      "singleQuote": false
    }
  }
}
```

## Disabling

```json
{ "severity": { "Prettier": "off" } }
```

Or demote failed formatting to a non-blocking note (it already only ever returns `warn`, never
`fail` — see [Notes](#notes)):

```json
{ "severity": { "Prettier": "error" } }
```

## Notes

- This check never returns `fail`. In `--check` mode, unformatted files produce `warn` with the
  message `unformatted files (run with write to fix)`; only `severity: { "Prettier": "error" }`
  turns that into a build-breaking outcome.
- `ctx.write` failing outright (Prettier itself erroring during `--write`) also reports `warn`,
  not `fail`.
- Reads and writes go through the current Node binary invoking Prettier's bundled CJS entry
  point, and Prettier itself runs with `stdio: 'inherit'`, so its own file-by-file output
  appears directly in the CI log.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) · **Prettier** · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
