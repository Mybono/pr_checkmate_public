# Config Validation

[Checks Index](../INDEX.md) · [Case Collision](case-collision.md) · **Config Validation** · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Validates `pr-checkmate.json` itself, so a typo in the config file doesn't fail silently. It checks
two things:

- **Unknown top-level keys** — a key like `mergeConfict` (missing the `l`) is never applied, and
  without this check that typo just does nothing forever with no warning.
- **Invalid `severity` values** — every entry in the `severity` map must be `error`, `warn`, or `off`;
  anything else is flagged.

The check reads the **raw file from disk**, not the merged-with-defaults config every other check
operates on. That distinction matters: it reports what you actually wrote, not what PR CheckMate
computed after filling in defaults — a merged config would never show an unknown key since defaults
don't contain typos.

`//`-prefixed comment keys and `$schema` are ignored, since both are expected, meaningful content in a
`pr-checkmate.json` rather than mistakes.

| Property | Value |
|---|---|
| Display name | `Config Validation` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate config-validation` |
| Config key | `configValidation` |
| Source | `src/core/checks/quality/config-validation.ts` |

## When it applies

`configValidation.enabled` is not `false`. If the repository has no `pr-checkmate.json` at all, the
check returns `skip('no pr-checkmate.json')` — there is nothing to validate when a project runs on
defaults alone.

If the file exists but isn't valid JSON, the check reports that directly (`pr-checkmate.json is not
valid JSON: <parser message>`) rather than trying to validate keys in a file it couldn't parse.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `configValidation.enabled` | boolean | `true` | Set `false` to skip the check |

There is no severity key of its own and no way to tune which keys are "known" — the recognized key set
is fixed in the check's implementation (`KNOWN_CONFIG_KEYS`) and kept in sync with every configurable
check as new ones are added.

### Example

```json
{
  "configValidation": { "enabled": false }
}
```

## Disabling

```json
{ "configValidation": { "enabled": false } }
```

Or:

```json
{ "severity": { "Config Validation": "off" } }
```

## Notes

- Up to **10** issues are logged individually; the returned summary quotes the first **3**, followed
  by `…` if there are more.
- This check is what makes typo'd keys visible at all — `severity: { "Config Validation": "off" }`
  turns that feedback off along with everything else, which is rarely what you want even on a
  project that disables most other checks.
- Because it reads the file directly rather than through the normal config loader, it is unaffected
  by how any other check merges or normalizes its own settings — it only sees what's literally on
  disk.

---

[Checks Index](../INDEX.md) · [Case Collision](case-collision.md) · **Config Validation** · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)
