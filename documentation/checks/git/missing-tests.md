# Missing Tests

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · **Missing Tests** · [TODO/FIXME](todo-fixme.md)

---

## Overview

A heuristic nudge, not a coverage tool: it flags a PR that **adds** a new source file but touches no
test file anywhere in scope. It cannot know whether the right test was written — only that *some* test
file changed alongside the new code. For line-level coverage of changed code, see
[Coverage](../quality/coverage.md) instead.

Source files are recognised across every supported language (`.ts`/`.tsx`/`.js`/`.jsx`/`.mjs`/`.cjs`,
`.py`, `.kt`/`.kts`/`.java`, `.swift`, `.cpp`/`.cc`/`.cxx`/`.c`/`.h`/`.hpp`, `.go`, `.rb`). A file is
recognised as a test by common per-language conventions: `*.test.ts`/`*.spec.jsx`, an `__tests__/` or
`test(s)/` directory, `test_*.py`/`*_test.py`/`*_test.go`, or `*Test.kt`/`*Spec.swift` and similar.

| Property | Value |
|---|---|
| Display name | `Missing Tests` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate missing-tests` |
| Config key | `missingTests` |
| Source | `src/core/checks/git/missing-tests.ts` |

## When it applies

Both conditions must hold:

1. `missingTests.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

The file set **is** scoped by `sourcePath` and `ignoreDirs`, the same way linting is — a file outside
the configured source tree is never counted as "new source" that needs a test.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `missingTests.enabled` | boolean | `true` | Set `false` to skip the check |
| `missingTests.severity` | `error` \| `warn` | `"warn"` | `error` fails the run; `warn` advises |
| `missingTests.ignore` | string[] | see below | Globs for source files that legitimately ship without a test of their own. Setting this **replaces** the defaults |

Default `ignore`:

```json
["**/*.d.ts", "**/index.ts", "**/index.tsx", "**/index.js", "**/index.jsx", "**/*.config.*", "**/migrations/**", "**/migrate/**", "**/__init__.py"]
```

Barrel files, type declarations, config files, migrations, and Python package markers are exempt by
default, since none of them typically carry meaningful logic to test.

### Example

```json
{
  "missingTests": {
    "enabled": true,
    "severity": "warn",
    "ignore": ["**/*.d.ts", "**/index.ts", "**/*.generated.ts"]
  }
}
```

Promote to a blocking gate:

```json
{
  "missingTests": { "severity": "error" }
}
```

## Disabling

```json
{ "missingTests": { "enabled": false } }
```

Or via the universal override:

```json
{ "severity": { "Missing Tests": "off" } }
```

## Notes

- **Only *added* files count as needing a test; modified files do not.** A source file changed (but
  not newly added) never triggers a finding on its own — the check is about new code arriving with no
  test coverage, not about existing code losing it.
- **Any test file touched anywhere in scope satisfies the check for the whole PR** — modifying an
  existing test, not just adding a new one, is enough. The check does not verify that the touched test
  actually exercises the new file; it's a heuristic, not a coverage cross-reference.
- The `pr-checkmate.json` template ships a shorter `ignore` list than the code's built-in default —
  it omits `**/migrations/**`, `**/migrate/**`, and `**/__init__.py`. Since the config key is present
  (not omitted) in the generated file, that shorter list is what actually applies for anyone using the
  template as their starting config; the fuller 9-entry list above only applies with no `missingTests.ignore`
  key at all.
- Findings are capped at **5 shown**, then `... and N more`, each logged as
  `new source without tests: path`.
- The summary is `${N} new source file(s) added without any test changes`.
- A crash while reading the diff (e.g. git unavailable) is reported as `skip('diff unavailable')`
  rather than a failure.

---

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · **Missing Tests** · [TODO/FIXME](todo-fixme.md)
