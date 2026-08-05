# Spellcheck

[Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · **Spellcheck** · [YAML Lint](yaml-lint.md)

---

## Overview

Runs [cspell](https://cspell.org/) over source, Markdown, and JSON files in scope. Like
[Markdown](markdown-lint.md), it's bundled — no runner dependency — and always runs regardless of
detected languages.

The bundled dictionary ships **multi-language** out of the box: English plus French, Russian,
Ukrainian, Hebrew, and Spanish (`@cspell/dict-fr-fr`, `@cspell/dict-ru_ru`, `@cspell/dict-uk-ua`,
`@cspell/dict-he`, `@cspell/dict-es-es`), on top of thousands of project-specific technical terms,
brand names, and API identifiers already accumulated in the bundled word list — SDK names, exchange
names, ffmpeg flags, and the like.

**Your own cspell config wins.** If the repository has `cspell.json`, `.cspell.json`, or
`cspell.config.json`, the check runs directly against it and ignores the bundled config and any
`cspell` block in `pr-checkmate.json` entirely.

| Property | Value |
|---|---|
| Display name | `Spellcheck` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate spellcheck` |
| Config key | `spellcheck` |
| Source | `src/core/checks/quality/spellcheck.ts` |

## When it applies

Always — `applies()` returns `true` unconditionally, with no `enabled` flag. The check still returns
`skip('git unavailable')` if git can't be queried, and `pass('no files to spellcheck')` (not a skip)
when nothing in scope matches.

Files considered: `*.ts *.tsx *.js *.jsx *.md *.mdx *.json`, scoped to `sourcePath` and filtered
through the standard ignore-directory list.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `cspell.words` | string[] | `[]` | Project vocabulary to accept, **appended** to the bundled word list |
| `cspell.ignorePaths` | string[] (globs) | `[]` | Extra paths to skip, **appended** to the bundled ignore list |
| `cspell.ignoreRegExpList` | string[] (regex sources) | `[]` | Extra content patterns to exclude from spellchecking, **appended** to the bundled list |

### Example

```json
{
  "cspell": {
    "words": ["acmecorp", "myinternalservice"],
    "ignorePaths": ["fixtures/**"]
  }
}
```

## Disabling

There is no `enabled` flag — use the universal severity override:

```json
{ "severity": { "Spellcheck": "off" } }
```

## Notes

- **Config-key naming.** The check reads a top-level **`cspell`** block from `pr-checkmate.json` (the
  actual JSON key, matching the underlying tool's own name and the `init`-generated config), not a
  `spellcheck` block — despite the check's display name and this doc's config-key column both being
  `spellcheck`. `pr-checkmate.json` also accepts a differently-shaped `spellcheck: { words,
  ignoreWords, ignorePaths }` block (recognized by [Config Validation](config-validation.md) as a
  known key, and present in the config's type definitions), but it is not consulted by this check —
  use `cspell.words` / `cspell.ignorePaths` as shown above.
- The bundled `cspell.json` also sets `language: "en,ru,uk,he,fr,es"` and pre-imports all five
  non-English dictionaries; these are fixed as part of the bundled default and aren't controlled by a
  `pr-checkmate.json` key.
- Bundled dictionary import paths are resolved absolutely at run time (via Node module resolution or
  relative to the bundled config), so they work regardless of the client's working directory or how
  the dependency tree is hoisted.
- Built-in `ignoreRegExpList` already excludes URLs and 7–40 character hex strings (commit SHAs,
  hashes) from spellchecking, on top of anything added via `cspell.ignoreRegExpList`.
- Runs with `stdio: 'inherit'`; exit code `0` is a pass, anything else is reported as
  `spelling issues found`, with per-word detail living in the inherited cspell output rather than the
  check's own summary.
- A temporary merged config (when no local cspell config exists) is written to
  `<cwd>/.cspell.temp.json` and removed in a `finally` block regardless of outcome.

---

[Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · **Spellcheck** · [YAML Lint](yaml-lint.md)
