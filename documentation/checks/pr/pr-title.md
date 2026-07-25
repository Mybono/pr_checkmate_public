# PR Title

[Checks Index](../INDEX.md) · [Commitlint](commitlint.md) · [PR Body](pr-body.md) · [PR Size](pr-size.md) · **PR Title**

---

## Overview

Checks that the pull request **title** follows
[Conventional Commits](https://www.conventionalcommits.org/), and warns when it does not.

This matters more than it looks on a repository that squash-merges: the PR title becomes the commit
message on the default branch, so a title like `fixed stuff` is what release tooling reads. Tools
such as `standard-version` derive the next version number and the changelog from those messages — an
unconventional title silently drops the change out of the changelog.

The accepted shape is:

```text
<type>[(optional scope)][!]: <description>
```

| Part | Accepted values |
|---|---|
| `type` | `feat` · `fix` · `docs` · `style` · `refactor` · `test` · `chore` · `perf` · `ci` · `build` · `revert` |
| scope | optional, any text in parentheses — `feat(parser):` |
| `!` | optional breaking-change marker — `feat!:` or `feat(api)!:` |
| description | required, must be non-empty after the colon and space |

| Property | Value |
|---|---|
| Display name | `PR Title` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate pr-title` |
| Config key | none |
| Source | `src/core/checks/pr/pr-title-lint.ts` |

## When it applies

`GITHUB_EVENT_PATH` is set **and** the file it points to exists — in other words, the check runs only
inside GitHub Actions. It reads the PR title from the event payload rather than from git, because the
title is GitHub state and not part of the repository.

On GitLab, Bitbucket, or a local run it does not apply and is omitted from the report entirely.
Within GitHub Actions it still returns `skip` when the event has no `pull_request` object — for
example on a `push` trigger.

## Configuration

This check has **no config key**. The accepted type list is fixed in the source and cannot be
extended through `pr-checkmate.json`.

If you need a different type list, use [Commitlint](commitlint.md) instead: it validates commit
messages, accepts a configurable `types` array, and honours a local `commitlint.config.*` file.

| Key | Effect here |
|---|---|
| `severity` | The only way to downgrade, promote, or disable it |

## Disabling

There is no `enabled` flag:

```json
{ "severity": { "PR Title": "off" } }
```

To make an unconventional title fail the run — reasonable when squash-merge feeds your changelog:

```json
{ "severity": { "PR Title": "error" } }
```

## Notes

- The offending title is written to the log verbatim, so the report shows what was rejected.
- The check is title-only. It does not inspect the commits inside the PR — that is
  [Commitlint](commitlint.md)'s job. On a squash-merge repository the title is the one that matters;
  on a merge-commit repository the individual commits are.
- An unreadable or unparseable event payload returns `skip('event unreadable')`, never a failure.

---

[Checks Index](../INDEX.md) · [Commitlint](commitlint.md) · [PR Body](pr-body.md) · [PR Size](pr-size.md) · **PR Title**
