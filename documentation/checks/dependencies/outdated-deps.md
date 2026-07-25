# Outdated Deps

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · **Outdated Deps** · [Vuln Scan](vuln-scan.md)

---

## Overview

Runs `npm outdated --json` and warns about dependencies that are a **full major version** behind the
latest release. Minor and patch drift is counted and logged as information, but never warns —
otherwise the check would fire on nearly every PR in an active project and stop being read.

| Gap to latest | Outcome |
|---|---|
| Major version behind | `warn`, one line per package |
| Minor or patch behind | `pass`, with an informational count |
| Up to date | `pass` |

| Property | Value |
|---|---|
| Display name | `Outdated Deps` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate outdated` |
| Config key | `outdatedDeps` |
| Source | `src/core/checks/dependencies/outdated-deps.ts` |

## When it applies

Both conditions must hold:

1. `outdatedDeps.enabled` is not `false`
2. a `package.json` exists at the repository root

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `outdatedDeps.enabled` | boolean | `true` | Set `false` to skip the check |
| `outdatedDeps.ignore` | string[] | `[]` | Package names dropped before counting, so they never warn |
| `outdatedDeps.timeoutMs` | number | `30000` | Cap on how long `npm outdated` may take. On timeout the check skips |

> `timeoutMs` is read by the check but is **not** currently declared in
> `pr-checkmate.schema.json`. It works, but your editor will not autocomplete it and
> [Config Validation](../quality/config-validation.md) validates only top-level keys, so a typo
> inside `outdatedDeps` goes unnoticed.

### Examples

Pin two packages deliberately behind their latest major and stop them warning:

```json
{
  "outdatedDeps": {
    "ignore": ["eslint", "@types/node"]
  }
}
```

Allow longer for a large dependency tree on a slow runner:

```json
{
  "outdatedDeps": { "timeoutMs": 60000 }
}
```

## Disabling

```json
{ "outdatedDeps": { "enabled": false } }
```

Or promote it to a blocking gate, if being current is a hard requirement:

```json
{ "severity": { "Outdated Deps": "error" } }
```

## Notes

- **Why a timeout exists.** `npm outdated` queries the registry for every dependency and can run for
  minutes on a large tree. An advisory check must never hold up CI, so it is capped and *skips* on
  timeout rather than failing.
- **Baseline selection.** npm's `current` field is only populated when the package is actually
  installed in `node_modules`. In CI — where pr-checkmate is often invoked via `npx` without an
  `npm ci` for the target repo — `current` is absent, so the check falls back to `wanted`: the
  version the `package.json` range resolves to, which npm computes from the registry without a local
  install. Entries with neither field are skipped rather than treated as major `0`, which would
  report every package as wildly out of date.
- The log lists every major-behind package as `name: baseline → latest`.
- Unparseable JSON is treated as `pass`, not as a failure — this check is advisory and must not break
  a PR because of a registry hiccup.

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · **Outdated Deps** · [Vuln Scan](vuln-scan.md)
