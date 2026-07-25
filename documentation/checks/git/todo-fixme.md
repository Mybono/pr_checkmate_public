# TODO/FIXME

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · **TODO/FIXME**

---

## Overview

Flags leftover `TODO`/`FIXME`/`XXX`/`HACK` markers added in Python and Kotlin/Java comments. JS/TS
markers are already handled by [Diff Smells](diff-smell.md), so this check deliberately stays out of
those file globs to avoid reporting the same line twice (the same split used by
[Leftover Debug](leftover-debug.md)).

A marker only counts when it appears directly after a comment opener — `# TODO: …` or `// TODO: …` — so
the word "TODO" appearing elsewhere mid-comment is not matched.

Optionally, the check can require that a marker reference a ticket, so an untracked TODO is flagged but
one with a linked ticket (`#123`, or JIRA-style `PROJ-456`) is not.

| Property | Value |
|---|---|
| Display name | `TODO/FIXME` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate todo-fixme` |
| Config key | `todoFixme` |
| Source | `src/core/checks/git/todo-fixme.ts` |

## When it applies

All of the following must hold:

1. `todoFixme.enabled` is not `false`
2. a diff range is available (`baseSha` is set)
3. the repository's detected languages include Python and/or Kotlin

The Python scan and the Kotlin/Java scan run independently — a Python-only repo only ever sees the
`#`-comment scan. A repo with neither language never runs at all.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `todoFixme.enabled` | boolean | `true` | Set `false` to skip the check |
| `todoFixme.severity` | `error` \| `warn` | `"warn"` | `error` fails the run; `warn` advises |
| `todoFixme.requireTicket` | boolean | `false` | When `true`, a marker that links a ticket (`#123` / `PROJ-456`) is not flagged |
| `todoFixme.keywords` | string[] | `["TODO", "FIXME", "XXX", "HACK"]` | Marker keywords to scan for. Setting this **replaces** the defaults |

### Example

```json
{
  "todoFixme": {
    "enabled": true,
    "severity": "warn",
    "requireTicket": false,
    "keywords": ["TODO", "FIXME", "XXX", "HACK"]
  }
}
```

Require every marker to reference a ticket, and treat an untracked one as a blocking failure:

```json
{
  "todoFixme": {
    "requireTicket": true,
    "severity": "error"
  }
}
```

## Disabling

```json
{ "todoFixme": { "enabled": false } }
```

Or via the universal override:

```json
{ "severity": { "TODO/FIXME": "off" } }
```

## Notes

- **Keyword matching is case-insensitive**, but findings are bucketed under the **uppercased** keyword
  regardless of how it was written in the source — `// todo: fix` is reported under the `TODO` label.
- **`requireTicket` checks the whole line for a ticket reference, not just the text after the
  keyword.** A line containing both the marker and a ticket ID anywhere in it is exempted, even if the
  ticket ID isn't adjacent to the marker.
- The `pr-checkmate-ignore` directive is honoured, so a single deliberate marker can opt out.
- Findings are grouped by keyword, and the log shows at most **3 examples per label**; matched lines
  are trimmed to 100 characters.
- The summary is compact — `2× TODO, 1× FIXME` — with `(no ticket reference)` appended whenever
  `requireTicket` is on.
- Only added lines are scanned, so an existing untracked `TODO` is not reported.
- A crash while reading the diff (e.g. git unavailable) is reported as `skip('diff unavailable')`
  rather than a failure.

---

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · **TODO/FIXME**
