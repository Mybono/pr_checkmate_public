# Markdown

[Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · **Markdown** · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Runs [markdownlint-cli2](https://github.com/DavidAnson/markdownlint-cli2) over every `.md`/`.mdx`
file in scope. It is bundled — no runner dependency, no toolchain to install — and, like
[Spellcheck](spellcheck.md), always runs regardless of detected languages, since Markdown shows up in
every kind of repository.

**Your own markdownlint config wins.** If the repository has any of `.markdownlint.json`,
`.markdownlint.jsonc`, `.markdownlint.yaml`, or `.markdownlint.yml`, the check passes no `--config` at
all and lets `markdownlint-cli2` auto-discover it — at that point `markdownlint.rules` in
`pr-checkmate.json` is **ignored entirely**.

Only when none of those exists does it fall back to a bundled ruleset, merged with any
`markdownlint.rules` you've configured:

| Bundled rule | Setting |
|---|---|
| `MD013` (line length) | `line_length: 500` |
| `MD024` (duplicate headings) | `siblings_only: true` |
| `MD033` (inline HTML) | `false` (disabled) |
| `MD060` | `false` (disabled) |

| Property | Value |
|---|---|
| Display name | `Markdown` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate markdown` |
| Config key | `markdownlint` |
| Source | `src/core/checks/quality/markdown-lint.ts` |

## When it applies

Always — `applies()` returns `true` unconditionally, with no `enabled` flag to check. The check still
returns `skip('git unavailable')` if git can't be queried, or `skip('no markdown files')` when there
are no `.md`/`.mdx` files in scope (delta in a PR, full scan otherwise).

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `markdownlint.rules` | object | `{}` | markdownlint rule overrides, **merged** on top of the bundled defaults above. Only used when the repo has no local `.markdownlint.*` file |
| `markdownlint.ignores` | string[] (globs) | `[]` | Extra paths to skip, **appended** to the built-in ignore list |

### Built-in ignore list

```text
node_modules/** · dist/** · coverage/** · graphify-out/** · .claude/**
```

`markdownlint-cli2` has no `--ignore-path` flag, so every exclusion — built-in or from
`markdownlint.ignores` — is expressed internally as a negated glob (`#pattern`). A bare directory name
also gets a second `#name/**` glob generated for it, so excluding `docs/generated` also excludes
everything inside it.

### Example

Disable the line-length rule and skip a generated docs folder:

```json
{
  "markdownlint": {
    "rules": { "MD013": false },
    "ignores": ["docs/generated/**"]
  }
}
```

## Disabling

There is no `enabled` flag — use the universal severity override:

```json
{ "severity": { "Markdown": "off" } }
```

To keep it visible but non-blocking (its default is already `warn`, so this is only relevant if you
promoted it):

```json
{ "severity": { "Markdown": "warn" } }
```

## Notes

- Unlike most `informational`-phase checks, this one can **write files**: when run with `--fix` (the
  package's format-on-write mode), it passes `--fix` through to `markdownlint-cli2`, which rewrites
  auto-fixable violations in place. It still runs in the `informational` phase rather than `format`,
  so plan around that if you rely on phase ordering.
- When merged rules are needed, they're written to a temporary file in the OS temp directory named
  `.markdownlint-merged-<pid>.json`, and deleted in a `finally` block regardless of outcome.
- Runs with `stdio: 'inherit'`, so `markdownlint-cli2`'s own per-violation output appears directly in
  the CI log.
- Exit code `0` is a pass and `1` is reported as `markdown lint issues found`, with no per-rule
  breakdown in the check's own summary — the detail lives in the inherited log output.
- Exit code `2` means `markdownlint-cli2` could not run at all (an unreadable config, a bad glob). It
  is reported as `markdownlint failed to run (exit 2)` rather than as lint findings, so a tooling
  problem is never attributed to the client's markdown — which matters when the check is promoted
  with `severity: { "Markdown": "error" }`.

---

[Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · **Markdown** · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)
