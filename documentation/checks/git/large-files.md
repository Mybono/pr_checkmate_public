# Large Files

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · **Large Files** · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)

---

## Overview

Flags files added, modified, or renamed in the diff that are either oversized or binary. It exists to
catch accidental bloat — a build artifact, a video, a database dump — committed where it shouldn't be,
before it permanently inflates the repository's history.

Two independent conditions can flag a file, and either is reported on its own: it can be **too large**
(bigger than the configured limit) and/or **binary** (as git itself classifies it via `--numstat`). A
file can trip both at once, in which case both reasons are shown together.

Lockfiles are exempt from the size limit by default — a 600 KB `package-lock.json` is expected, not
bloat. Their content is still covered by [Lockfile Drift](lockfile-drift.md) and the Sensitive Files
check.

| Property | Value |
|---|---|
| Display name | `Large Files` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate large-files` |
| Config key | `largeFiles` |
| Source | `src/core/checks/git/large-files.ts` |

## When it applies

Both conditions must hold:

1. `largeFiles.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

Unlike most checks in this group, the file set is **not** scoped to `sourcePath` — bloat can land
anywhere in the repo, including outside a configured source tree, so a root-level asset is exactly the
case a narrowed scope would miss. `ignoreDirs` (`node_modules`, `dist`, `vendor`, …) is still honoured,
since large generated files there are expected.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `largeFiles.enabled` | boolean | `true` | Set `false` to skip the check |
| `largeFiles.severity` | `error` \| `warn` | `"warn"` | `error` fails the run; `warn` advises |
| `largeFiles.maxSizeKb` | number | `500` | Files above this size (in KB) are flagged |
| `largeFiles.allow` | string[] | `[]` | Globs exempt from the size/binary check. **Appended** to the built-in lockfile allow-list, not a replacement for it |

### Example

```json
{
  "largeFiles": {
    "enabled": true,
    "severity": "warn",
    "maxSizeKb": 500,
    "allow": ["assets/**", "*.png"]
  }
}
```

Promote to a blocking gate for a repo that must never grow a binary blob:

```json
{
  "largeFiles": { "severity": "error" }
}
```

## Disabling

```json
{ "largeFiles": { "enabled": false } }
```

Or via the universal override:

```json
{ "severity": { "Large Files": "off" } }
```

## Notes

- **Lockfiles are always exempt from the size check**, regardless of `allow`: `package-lock.json`,
  `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb`, `npm-shrinkwrap.json`, `Cargo.lock`, `poetry.lock`,
  `uv.lock`, `Pipfile.lock`, `pdm.lock`, `Gemfile.lock`, `composer.lock`, `go.sum`, and
  `Package.resolved` are all matched by a built-in list before `allow` is even consulted.
- **Binary detection comes from `git diff --numstat`**, not a content sniff — a file is treated as
  binary when git reports `-`/`-` in the additions/deletions columns for it, which is exactly how git
  itself decides a file is binary.
- A file that matches `allow` is skipped entirely — neither the size nor the binary reason is reported
  for it, even if it is both huge and binary.
- If a file is renamed or removed between the diff snapshot and the on-disk stat, it is silently
  skipped rather than reported or failing the check.
- The summary is `${N} large/binary file(s) added`; each finding is logged individually as
  `path (reason[, reason])`, e.g. `assets/demo.mp4 (2048 KB > 500 KB, binary)`.
- Deleted files are never flagged — the underlying file list only includes added, copied, modified, and
  renamed paths.

---

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · **Large Files** · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)
