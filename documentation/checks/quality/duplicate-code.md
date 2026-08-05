# Duplicate Code

[Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · **Duplicate Code** · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Runs [jscpd](https://github.com/kucherenko/jscpd) — copy/paste detection — over the PR's changed
JS/TS files, scoped to `sourcePath`, to catch duplicated blocks large enough to warrant extracting a
shared function or module.

Certain paths are excluded **regardless of configuration**: `node_modules/**`, `dist/**`, `.git/**`,
and common config-file globs (`*.config.js`, `.eslintrc.*`, `jest.config.*`, `playwright.config.*`,
and similar). Config files are data, not logic, and are commonly shipped in more than one format
(e.g. `.eslintrc` alongside a flat config) — their repetition is expected, not a code smell, so
scanning them would only produce noise no `duplicate.ignore` entry could reasonably fix.

| Property | Value |
|---|---|
| Display name | `Duplicate Code` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate duplicate` |
| Config key | `duplicate` |
| Source | `src/core/checks/quality/duplicate-check.ts` |

## When it applies

`duplicate.enabled` is not `false`, **and** the repository's detected languages include `typescript`
or `javascript`. In delta mode the check only scans changed files (`*.ts`, `*.tsx`, `*.js`, `*.jsx`)
under `sourcePath`; outside a PR context it falls back to every matching file. It returns
`skip('git unavailable')` if git can't be queried, or `skip('no relevant files')` when nothing matches.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `duplicate.enabled` | boolean | `true` | Set `false` to skip the check |
| `duplicate.minLines` | number | `5` | Minimum consecutive lines for a match to count as a clone |
| `duplicate.minTokens` | number | `50` | Minimum token count for a match to count as a clone |
| `duplicate.ignore` | string[] (globs) | test files/folders (see below) | Paths excluded from detection, **in addition to** the always-on base ignores. Setting this **replaces** the default list |
| `duplicate.formats` | string[] | source-code formats (see below) | jscpd language formats to scan. Setting this **replaces** the default list |

### Default `ignore`

```json
["**/__tests__/**", "**/__mocks__/**", "**/*.test.*", "**/*.spec.*", "**/test/**", "**/tests/**"]
```

Test setup legitimately repeats itself (fixtures, boilerplate assertions), so it's excluded by
default. These are appended to the always-on base ignores, not a replacement for them.

### Default `formats`

```json
["javascript", "typescript", "jsx", "tsx", "python", "kotlin", "swift", "c", "cpp"]
```

Data formats — YAML, JSON, Markdown — are intentionally **not** included by default, since repetition
in config or content files is rarely meaningful duplication. Add them back explicitly if you do want
that coverage.

### Example

Loosen the detection threshold and re-include YAML:

```json
{
  "duplicate": {
    "minLines": 10,
    "minTokens": 80,
    "formats": ["javascript", "typescript", "jsx", "tsx", "yaml"]
  }
}
```

## Disabling

```json
{ "duplicate": { "enabled": false } }
```

Or:

```json
{ "severity": { "Duplicate Code": "off" } }
```

## Notes

- The check name is `Duplicate Code` and the doc/CLI slug is `duplicate` / `duplicate-code.md`, but
  the underlying source file is named `duplicate-check.ts` — a naming mismatch worth knowing if you go
  looking for it.
- jscpd v5 appends promotional/funding lines to its console output; the check strips any line
  mentioning those markers before it reaches your terminal.
- The clone count in the summary is parsed from jscpd's own `N clones found` console line via regex,
  not from a JSON reporter — if that text format ever changes upstream, the count could read as `0`
  even when jscpd found clones.
- If jscpd itself is unavailable or exits with an unreadable result, the check returns
  `skip('jscpd unavailable')` rather than failing the run.
- There is no `duplicate.severity` key — the finding is always a `warn`. Use the universal
  `severity: { "Duplicate Code": "error" }` override to make duplication block a merge.

---

[Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · **Duplicate Code** · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)
