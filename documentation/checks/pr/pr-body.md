# PR Body

[Checks Index](../INDEX.md) · [Commitlint](commitlint.md) · **PR Body** · [PR Size](pr-size.md) · [PR Title](pr-title.md)

---

## Overview

Validates the pull request **description**: that it is long enough to be a description at all, and
that it references a ticket.

Both problems it catches are the same problem a year later — a change on the default branch with no
recorded reason. The ticket reference is what connects the diff to the discussion that produced it.

Accepted ticket references:

| Form | Example |
|---|---|
| GitHub closing keyword | `Closes #123` · `Fixes #45` · `Resolves #7` (case-insensitive, singular or plural) |
| JIRA-style key | `PROJ-456` — uppercase letters/digits, a hyphen, then digits |

| Property | Value |
|---|---|
| Display name | `PR Body` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate pr-body` |
| Config key | `prBody` |
| Source | `src/core/checks/pr/pr-body.ts` |

## When it applies

Both conditions must hold:

1. `prBody.enabled` is not `false`
2. `GITHUB_EVENT_PATH` is set and the file exists — the check runs only inside GitHub Actions, because
   the description is GitHub state rather than repository content

On other CI platforms it is omitted from the report. Inside GitHub Actions it returns `skip` when the
event carries no `pull_request` object.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `prBody.enabled` | boolean | `true` | Set `false` to skip the check |
| `prBody.minLength` | number | `20` | Minimum description length in characters, after trimming |
| `prBody.requireTicket` | boolean | `true` | Require a ticket reference |

### Examples

Ask for a substantial description but drop the ticket requirement — sensible for a repository with no
issue tracker:

```json
{
  "prBody": { "minLength": 80, "requireTicket": false }
}
```

Keep the ticket requirement and accept a one-line description:

```json
{
  "prBody": { "minLength": 1, "requireTicket": true }
}
```

## Disabling

```json
{ "prBody": { "enabled": false } }
```

Or promote it to a hard gate:

```json
{ "severity": { "PR Body": "error" } }
```

## Notes

- **An empty description reports only the length problem.** The ticket rule is evaluated as
  `requireTicket && trimmed && …` — it is skipped entirely when the body is empty, so a blank PR gets
  one clear message instead of two overlapping ones.
- The description is trimmed before measuring, so whitespace and newlines do not count toward
  `minLength`.
- Both problems can be reported together when the body is short *and* has no ticket; they are joined
  by a semicolon in the summary.
- `minLength` counts characters, not words — a 20-character description satisfies the default while
  saying very little. Raise it if you want the check to have real force.

---

[Checks Index](../INDEX.md) · [Commitlint](commitlint.md) · **PR Body** · [PR Size](pr-size.md) · [PR Title](pr-title.md)
