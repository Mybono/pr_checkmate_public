# Lockfile Drift

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · **Lockfile Drift** · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)

---

## Overview

Flags a manifest changed in the diff without its matching lockfile also being updated — the state
where CI would resolve dependency versions the author never actually locked.

| Manifest | Lockfile(s) |
|---|---|
| `package.json` | `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml` |
| `pyproject.toml` | `poetry.lock`, `uv.lock` |
| `requirements.in` | `requirements.txt` |
| `go.mod` | `go.sum` |
| `Cargo.toml` | `Cargo.lock` |
| `Gemfile` | `Gemfile.lock` |

Matched by basename, so it works the same way inside a monorepo sub-package as at the repo root.

| Property | Value |
|---|---|
| Display name | `Lockfile Drift` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate lockfile-drift` |
| Config key | `lockfileDrift` |
| Source | `src/core/checks/git/lockfile-drift.ts` |

## When it applies

Both conditions must hold:

1. `lockfileDrift.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

The changed-file list is **not** scoped to `sourcePath` — manifests and lockfiles typically live at the
repository root, outside any configured source tree, and scoping would hide the very files the check
needs to see.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `lockfileDrift.enabled` | boolean | `true` | Set `false` to skip the check |
| `lockfileDrift.severity` | `error` \| `warn` | `"warn"` | `error` fails the run; `warn` advises |

That is the whole surface — the manifest/lockfile pairing table is fixed in the source.

### Example

```json
{
  "lockfileDrift": { "enabled": true, "severity": "warn" }
}
```

Promote to a blocking gate:

```json
{
  "lockfileDrift": { "severity": "error" }
}
```

## Disabling

```json
{ "lockfileDrift": { "enabled": false } }
```

Or via the universal override:

```json
{ "severity": { "Lockfile Drift": "off" } }
```

## Notes

- **`package.json` is content-aware; every other ecosystem is not.** For `package.json`, the check
  parses `dependencies`/`devDependencies`/`peerDependencies`/`optionalDependencies` from both the base
  and head versions and only flags drift when that dependency data actually differs — a `version` or
  `scripts` bump alone does not trigger it. The other manifests (`pyproject.toml`, `go.mod`,
  `Cargo.toml`, `Gemfile`, `requirements.in`) have no such parser here, so *any* change to them is
  treated as needing a lockfile update.
- **A manifest with no committed lockfile is silently skipped** — if none of a manifest's paired
  lockfiles exist in the repo, there is nothing to drift, so no finding is produced.
- A newly added manifest (no base version to diff against) counts as changed.
- If reading the head manifest fails (e.g. it was deleted in this diff), it is treated as changed so
  the drift still surfaces.
- The summary is `${N} manifest(s) changed without a lockfile update`, and each finding is logged as
  `path changed but lockfile[s] not updated`.
- A crash while reading the diff (e.g. git unavailable) is reported as `skip('diff unavailable')`
  rather than a failure.

---

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · **Lockfile Drift** · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)
