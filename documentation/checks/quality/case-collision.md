# Case Collision

[Checks Index](../INDEX.md) · **Case Collision** · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Flags tracked files whose paths differ only by case — `Foo.ts` next to `foo.ts`. Such pairs coexist
fine on Linux's case-sensitive filesystem, but collide on macOS, Windows, and many Docker layers,
silently breaking a checkout or a build the moment someone works from one of those.

This is a **whole-repo scan**, not a diff check — it looks at every tracked file, not just the ones
touched by the PR, because a collision introduced weeks ago and only now colliding with a new file is
exactly the kind of thing delta-only scanning would miss.

| Property | Value |
|---|---|
| Display name | `Case Collision` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate case-collision` |
| Config key | `caseCollision` |
| Source | `src/core/checks/quality/case-collision.ts` |

## When it applies

`caseCollision.enabled` is not `false`. There is no language or tool prerequisite — the check reads
`git ls-files` directly and needs nothing installed.

The file list is scoped to `sourcePath` and filtered through `ignoreDirs`, same as every other check.
If the repository has no tracked files under that scope, the check returns `skip('no tracked files')`.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `caseCollision.enabled` | boolean | `true` | Set `false` to skip the check |

That is the entire configuration surface — there is no severity key, ignore list, or threshold.

### Example

```json
{
  "caseCollision": { "enabled": false }
}
```

## Disabling

```json
{ "caseCollision": { "enabled": false } }
```

Or remove it from the run entirely by display name:

```json
{ "severity": { "Case Collision": "off" } }
```

## Notes

- Files are grouped by lower-cased path. Any group with more than one distinct original path is a
  collision, and every member of that group is reported, joined with `↔`.
- This always finds a **pass/warn**, never a **fail** — there is no `severity` key promoting a
  collision to an error, so the universal `severity: { "Case Collision": "error" }` override is the
  only way to make it block a merge.
- Uses `listTrackedFiles`, the same whole-repo (non-delta) helper as checks like circular-dependency
  detection that need the full picture rather than just the PR's changed files.
- The check only compares paths that are still tracked by git — an untracked file on disk cannot
  collide with anything from the check's point of view.

---

[Checks Index](../INDEX.md) · **Case Collision** · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)
