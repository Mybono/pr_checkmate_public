# Merge Conflict

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · **Merge Conflict** · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)

---

## Overview

Detects unresolved git conflict markers committed into the diff. This is the least ambiguous failure in
the whole tool — a committed `<<<<<<<` is never intentional, and it usually means a file was resolved by
hand and one hunk was missed.

Recognised markers, each exactly seven characters at the start of an added line:

| Marker | Meaning |
|---|---|
| `<<<<<<<` | Start of the conflict (optionally followed by a ref label) |
| `\|\|\|\|\|\|\|` | Base section, present in `diff3` conflict style |
| `=======` | Separator, must stand alone on the line |
| `>>>>>>>` | End of the conflict (optionally followed by a ref label) |

Reported with file and line number, so the location is directly clickable in CI output.

| Property | Value |
|---|---|
| Display name | `Merge Conflict` |
| Phase | `blocking` |
| CLI command | none — runs only as part of a full run |
| Config key | `mergeConflict` |
| Source | `src/core/checks/git/merge-conflict.ts` |

## When it applies

Both conditions must hold:

1. `mergeConflict.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

The file set is resolved the same way as every other check — delta-aware, honouring `sourcePath` and
`ignoreDirs`. Conflicts are language-agnostic, so the glob is `*`; the scoping still keeps build,
vendor, and report directories out.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `mergeConflict.enabled` | boolean | `true` | Set `false` to skip the check |
| `mergeConflict.severity` | `error` \| `warn` | `"error"` | `error` fails the run; `warn` advises |

This is one of the few checks whose default severity is `error`, which is why it sits in the `blocking`
phase.

### Example

Downgrade to a warning — rarely appropriate, but useful in a repository that legitimately commits
conflict-marker fixtures:

```json
{
  "mergeConflict": { "severity": "warn" }
}
```

A better answer for that case is to keep the check strict and exclude the fixture directory:

```json
{
  "ignoreDirs": ["node_modules", "dist", "tests/fixtures/conflicts"]
}
```

## Disabling

```json
{ "mergeConflict": { "enabled": false } }
```

## Notes

- **A bare `=======` alone is not reported.** Seven equals signs are also a valid Markdown setext
  heading underline, so a separator counts only when the *same file* also contains an angle marker
  (`<<<<<<<` or `>>>>>>>`). Angle and base markers are always unambiguous and always reported.
- Line numbers are the **new-file** numbers, tracked by parsing each `@@` hunk header and counting added
  lines. Removed lines do not advance the counter.
- The summary lists the first **5** locations as `path:line`, then `(+N more)`. Every hit is written to
  the log individually.
- Severity controls both the log level and the outcome: `error` logs with ❌ and returns `fail`; `warn`
  logs with ⚠️ and returns `warn`.
- `resolveScopedTargetFiles` returning `null` (git unavailable) produces `skip('git unavailable')`; an
  empty file set passes.
- Only **added** lines are examined, so a conflict marker that already existed on the base branch is not
  reported by this check.

---

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · **Merge Conflict** · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)
