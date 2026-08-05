# Broken Symlinks

[Checks Index](../INDEX.md) · **Broken Symlinks** · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Flags tracked symlinks whose target does not exist — a link left pointing at a file that was renamed,
moved, or deleted without updating the link itself. That kind of broken link fails a checkout or a
build the moment something tries to follow it, regardless of which PR introduced it.

`fs.existsSync` follows a symlink to its target and returns `false` for exactly the broken case, which
is indistinguishable from "no file here at all" without a separate check that a link is actually
present. That's why this check can't reuse the normal delta/full-scan file resolution every other
check uses (`resolveTargetFiles` filters out anything that fails `fs.existsSync` — which would silently
drop the broken links this check exists to find) and instead pairs `fs.lstatSync` (is this a symlink?)
with `fs.existsSync` (does it resolve?) itself.

| Property | Value |
|---|---|
| Display name | `Broken Symlinks` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate symlinks` |
| Config key | `symlinks` |
| Source | `src/core/checks/quality/symlinks.ts` |

## When it applies

`symlinks.enabled` is not `false` — it's **on by default**. There is no language or tool prerequisite;
it needs nothing installed beyond Node's own `fs` module.

Uses `listTrackedFiles`, the same whole-repo (non-delta) helper as [Case Collision](case-collision.md)
and circular-dependency detection — checks that need the full picture rather than just the PR's changed
files. No tracked files at all is a `skip('no tracked files')`, not a `pass`.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `symlinks.enabled` | boolean | `true` | Set `false` to skip the check |
| `symlinks.severity` | `'error' \| 'warn'` | `warn` | `warn` advises (default); `error` fails the run |
| `symlinks.ignore` | string[] (globs) | `[]` | Links intentionally left broken, excluded from the check |

### Example

Promote to a gate, and exempt a fixture that deliberately tests another tool's own broken-symlink
handling:

```json
{
  "symlinks": {
    "severity": "error",
    "ignore": ["test/fixtures/broken-link"]
  }
}
```

## Disabling

```json
{ "symlinks": { "enabled": false } }
```

Or remove it from the run without touching the `symlinks` block:

```json
{ "severity": { "Broken Symlinks": "off" } }
```

## Notes

- `symlinks.ignore` filters by **file path** (standard glob matching against the symlink's own
  repo-relative path), the same convention [YAML Lint](yaml-lint.md) uses — not the finding-label
  substring match that `diffSecurity`/`workflowSecurity`/`dockerfileSecurity` use. See the
  [Checks Index](../INDEX.md#suppressing-a-single-line) for that distinction.
- Both `symlinks.severity` and the universal `severity: { "Broken Symlinks": … }` override achieve the
  same result; `symlinks.severity` is this check's own dedicated key, in addition to the universal
  mechanism every check supports.
- A symlink whose target exists but is itself a broken link one hop further down (a chain) is followed
  transparently by `fs.existsSync`, so only the final, unresolvable end of a chain is reported.

---

[Checks Index](../INDEX.md) · **Broken Symlinks** · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)
