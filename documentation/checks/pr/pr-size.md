# PR Size

[Checks Index](../INDEX.md) · [Commitlint](commitlint.md) · [PR Body](pr-body.md) · **PR Size** · [PR Title](pr-title.md)

---

## Overview

Measures the pull request with `git diff --shortstat` and warns when it exceeds a recommended file
count or line count. Insertions and deletions are summed, so a change that rewrites 400 lines counts
as 800.

The wording in the report is deliberately advisory — `recommended < N` rather than "limit exceeded".
Some changes are legitimately large (a generated client, a dependency bump, a rename across a
package), and this check exists to prompt a conversation about splitting the work, not to block it.

| Property | Value |
|---|---|
| Display name | `PR Size` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate pr-size` |
| Config key | `prSize` |
| Source | `src/core/checks/pr/pr-size-check.ts` |

## When it applies

Both conditions must hold:

1. `prSize.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

Outside a PR context there is nothing to measure, so the check does not run.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `prSize.enabled` | boolean | `true` | Set `false` to skip the check |
| `prSize.maxFiles` | number | `50` | Warn above this many changed files |
| `prSize.maxLines` | number | `1000` | Warn above this many changed lines (insertions + deletions) |

Each threshold is reported independently — a PR can trip one, the other, or both, and every tripped
threshold appears in the summary.

### Examples

Tighten both thresholds for a team that favours small PRs:

```json
{
  "prSize": { "maxFiles": 20, "maxLines": 400 }
}
```

Keep the file limit but stop worrying about line count on a repo with large generated files:

```json
{
  "prSize": { "maxFiles": 30, "maxLines": 100000 }
}
```

## Disabling

```json
{ "prSize": { "enabled": false } }
```

Or make an oversized PR fail the run rather than warn:

```json
{ "severity": { "PR Size": "error" } }
```

## Notes

- The actual counts are always logged as information (`N files changed, M lines`), even when both
  thresholds pass — useful as a trend signal in the CI log.
- Parsing is regex-based against git's `--shortstat` output. A diff with no insertions or no deletions
  simply omits that clause and the missing value is treated as `0`.
- If the diff cannot be read the check returns `skip('diff unavailable')` rather than failing.
- Not affected by `sourcePath` or `ignoreDirs`: it measures the whole diff, since PR size is about
  reviewer load rather than which files are product code.

---

[Checks Index](../INDEX.md) · [Commitlint](commitlint.md) · [PR Body](pr-body.md) · **PR Size** · [PR Title](pr-title.md)
