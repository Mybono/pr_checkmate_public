# Leftover Debug

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · **Leftover Debug** · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)

---

## Overview

Flags debug statements left in added Python and Kotlin/Java lines — the non-JS/TS counterpart of the
debug-output half of [Diff Smells](diff-smell.md). JS/TS `console.*`/`debugger` calls are already
covered there, so this check deliberately stays out of those file globs to avoid reporting the same
line twice.

| Language | Matches |
|---|---|
| Python | `print(`, `pprint(`, `breakpoint(`, `pdb.set_trace(` |
| Kotlin / Java | `System.out.println(` / `System.err.println(`, and bare `println(` |

The bare `println(` pattern deliberately excludes the qualified `System.out.println(`/`System.err.println(`
form so a qualified call isn't counted twice.

| Property | Value |
|---|---|
| Display name | `Leftover Debug` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate leftover-debug` |
| Config key | `leftoverDebug` |
| Source | `src/core/checks/git/leftover-debug.ts` |

## When it applies

All of the following must hold:

1. `leftoverDebug.enabled` is not `false`
2. a diff range is available (`baseSha` is set)
3. the repository's detected languages include Python and/or Kotlin

The check runs its Python scan and its Kotlin/Java scan independently — a Python-only repo only ever
sees the Python patterns, and vice versa. A repo with neither language never runs at all.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `leftoverDebug.enabled` | boolean | `true` | Set `false` to skip the check |
| `leftoverDebug.severity` | `error` \| `warn` | `"warn"` | `error` fails the run; `warn` advises |

There is no `ignore` or pattern override — the pattern list is fixed in the source.

### Example

```json
{
  "leftoverDebug": { "enabled": true, "severity": "warn" }
}
```

Promote to a blocking gate:

```json
{
  "leftoverDebug": { "severity": "error" }
}
```

## Disabling

```json
{ "leftoverDebug": { "enabled": false } }
```

Or via the universal override:

```json
{ "severity": { "Leftover Debug": "off" } }
```

## Notes

- **Matching is against the comment-stripped line.** A `print(x)` inside a `#` comment, or a `println`
  after a `//`, is not reported — only real code is matched.
- The `pr-checkmate-ignore` directive is honoured, so a deliberate leftover can opt out inline.
- Findings are grouped by label and the log shows at most **3 examples per label**; matched lines are
  trimmed to 100 characters.
- The summary is compact: `2× print(), 1× breakpoint()`.
- Only added lines are scanned, so a `print()` call that predates the PR is not reported.
- A crash while reading the diff (e.g. git unavailable) is reported as `skip('diff unavailable')`
  rather than a failure.

---

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · **Leftover Debug** · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)
