# Commitlint

[Checks Index](../INDEX.md) · **Commitlint** · [PR Body](pr-body.md) · [PR Size](pr-size.md) · [PR Title](pr-title.md)

---

## Overview

Runs [commitlint](https://commitlint.js.org/) over every commit in the pull request range and warns
when a message does not follow [Conventional Commits](https://www.conventionalcommits.org/).

Unlike [PR Title](pr-title.md), which inspects the single squash-merge title, this check validates the
individual commits — the ones that land on the default branch when a repository uses merge commits or
rebase merges.

**Your own commitlint config wins.** If the repository contains any of the standard config files, the
check uses it and ignores the `commitlint` block in `pr-checkmate.json`:

```text
commitlint.config.cjs · commitlint.config.js · commitlint.config.mjs · commitlint.config.ts
.commitlintrc · .commitlintrc.json · .commitlintrc.yaml · .commitlintrc.yml
.commitlintrc.js · .commitlintrc.cjs
```

Only when none of those exists does it generate a temporary config from your `pr-checkmate.json`
settings, extending `@commitlint/config-conventional`.

| Property | Value |
|---|---|
| Display name | `Commitlint` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate commitlint` |
| Config key | `commitlint` |
| Source | `src/core/checks/pr/commitlint.ts` |

## When it applies

`commitlint.enabled` is not `false`. Unlike the other checks in this category it does not require a
GitHub event, because it reads git history rather than PR metadata.

The range is `--from <baseSha> --to <headSha>`. Outside a PR context it falls back to `HEAD~1..HEAD`,
so a local run validates just the most recent commit.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `commitlint.enabled` | boolean | `true` | Set `false` to skip the check |
| `commitlint.types` | string[] | `feat`, `fix`, `docs`, `chore`, `ci`, `refactor`, `perf`, `style`, `revert`, `test`, `build` | Allowed commit types (`type-enum`) |
| `commitlint.ignores` | string[] | `["^Release\\b", "^chore\\(release\\):"]` | **Regex sources** for messages to skip. Matched case-insensitively |

Both `types` and `ignores` are **ignored entirely** when the repository has its own commitlint config.

### Why `ignores` has those defaults

commitlint already skips literal `Merge …` and `Revert …` messages, but not GitHub's squash-merge
titles. A release PR merged as `Release 15 (#114)` is not a conventional commit and never will be, so
the two default patterns keep automated release commits from producing permanent warnings.

### Examples

Add a project-specific type and skip dependabot commits:

```json
{
  "commitlint": {
    "types": ["feat", "fix", "docs", "chore", "ci", "refactor", "perf", "test", "build", "deps"],
    "ignores": ["^Release\\b", "^chore\\(release\\):", "^build\\(deps\\)"]
  }
}
```

Remember these are regex *sources* inside JSON, so a backslash must be escaped — `\\b`, `\\(`.

## Disabling

```json
{ "commitlint": { "enabled": false } }
```

Or promote violations to a failure:

```json
{ "severity": { "Commitlint": "error" } }
```

## Notes

- The generated temporary config is a **CJS module**, not JSON, because commitlint requires `ignores`
  to hold predicate *functions* — something JSON cannot express. It is written to the OS temp
  directory, named with the current process id, and deleted in a `finally` block so a crash cannot
  leave it behind.
- `@commitlint/config-conventional` is located through Node module resolution rather than by path,
  which keeps it working regardless of how the dependency tree is hoisted.
- commitlint runs with `stdio: 'inherit'`, so its full report appears directly in the CI log. The
  check's own summary is the short `commit message violations found`.
- The bundled `@commitlint/cli` binary is executed with the current Node binary rather than through
  `npx`, which would fail when commitlint is not on `PATH`.

---

[Checks Index](../INDEX.md) · **Commitlint** · [PR Body](pr-body.md) · [PR Size](pr-size.md) · [PR Title](pr-title.md)
