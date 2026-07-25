# License Header

[Checks Index](../INDEX.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · **License Header** · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Requires a license or copyright header — matched by a regex you configure — within the first N lines
of every source file **newly added** by the PR. It never looks at pre-existing files, so adopting this
check mid-project doesn't retroactively flag your entire history; it only holds new files to the
standard going forward.

Checked extensions: `.ts .tsx .js .jsx .mjs .cjs .py .kt .java .go .rs .cs .rb .php .swift .c .cc .cpp
.h .hpp`.

This check is fully **opt-in**: it does nothing at all until a repository sets
`licenseHeader.pattern`.

| Property | Value |
|---|---|
| Display name | `License Header` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate license-header` |
| Config key | `licenseHeader` |
| Source | `src/core/checks/quality/license-header.ts` |

## When it applies

`licenseHeader.enabled` is **exactly `true`**, **and** a base SHA is available — i.e. the check only
runs inside a PR/delta context, since "newly added" is meaningless without a diff range to compare
against. Outside a PR context (a local run with no base SHA) `applies()` returns `false` and the check
is omitted from the report entirely, rather than appearing as a skip.

Even with `enabled: true`, the check still needs `licenseHeader.pattern` set — if it's absent, `run()`
returns `skip('no licenseHeader.pattern configured')`. An invalid regex in `pattern` is caught the same
way, as `skip('invalid licenseHeader.pattern')`.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `licenseHeader.enabled` | boolean | `false` | Must be `true` for the check to run |
| `licenseHeader.pattern` | string (regex source) | `""` (unset) | Pattern that must appear somewhere in the checked lines |
| `licenseHeader.lines` | number | `10` | How many lines from the top of each new file are checked |

### Example

Require a `Copyright <year> <company>` line somewhere in the first 5 lines of every new file:

```json
{
  "licenseHeader": {
    "enabled": true,
    "pattern": "Copyright \\d{4} Acme Inc\\.",
    "lines": 5
  }
}
```

Remember `pattern` is a regex *source* inside a JSON string, so backslashes need escaping (`\\d`,
`\\.`).

## Disabling

```json
{ "licenseHeader": { "enabled": false } }
```

Leaving `enabled` unset (the default) has the same effect, since it's opt-in.

To keep it configured but non-blocking:

```json
{ "severity": { "License Header": "warn" } }
```

(there is no `licenseHeader.severity` of its own — see Notes.)

## Notes

- "Newly added" is computed as `git diff --diff-filter=A <baseSha> <headSha ?? HEAD>` — a file that
  was *modified* but already existed before the PR is never checked, regardless of whether it has a
  header.
- A file listed as added by the diff but no longer readable on disk (renamed or removed again between
  the diff and the check running) is silently skipped rather than reported missing.
- The header pattern is tested against the **joined first N lines** as a single string, so a
  multi-line pattern (or one anchored with `^`/`$` per line via the `m` flag) works, but a plain
  substring pattern is usually enough.
- Missing files are logged individually, capped at 10, with `... and N more` appended beyond that —
  the same convention as [YAML Lint](yaml-lint.md) and [Config Validation](config-validation.md).
- There is no `licenseHeader.severity` key — a missing header is always a `warn`. Use the universal
  `severity: { "License Header": "error" }` override to make it block a merge.

---

[Checks Index](../INDEX.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · **License Header** · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)
