# Banned Imports

[Checks Index](../INDEX.md) · **Banned Imports** · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)

---

## Overview

Blocks specific packages from entering the codebase. Scans the **added lines of the diff** for a
banned package arriving through any of four routes:

| Route | Example |
|---|---|
| ES import | `import x from 'moment'` |
| CommonJS require | `require('moment')` |
| Bare side-effect import | `import 'moment'` |
| Manifest dependency entry | `"moment": "^2.30.0"` in `package.json` |

Sub-paths count too — banning `lodash` also catches `import get from 'lodash/get'`. Because only
**added** lines are examined, existing usage is grandfathered in: the rule stops the problem
spreading without demanding an immediate repo-wide migration.

| Property | Value |
|---|---|
| Display name | `Banned Imports` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate banned-imports` |
| Config key | `bannedImports` |
| Source | `src/core/checks/dependencies/banned-imports.ts` |

> Despite the `informational` phase, a ban declared with `"severity": "error"` returns `fail` and
> **does fail the run**. The phase governs execution order, not whether the check can block — see
> [How checks are grouped](../INDEX.md#how-checks-are-grouped).

## When it applies

Both conditions must hold:

1. at least one entry in `bannedImports`
2. a diff range is available (`baseSha` is set)

There is no `enabled` flag: the check is inert until you configure a ban. Outside a PR context there
is no diff, so it does not run at all.

## Configuration

`bannedImports` is an array of objects.

| Key | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `name` | string | **yes** | — | Package name to ban. Matched literally (regex-escaped), plus any sub-path |
| `message` | string | no | `banned dependency: <name>` | Explanation shown in the report — use it to name the approved alternative |
| `severity` | `error` \| `warn` | no | `"warn"` | `error` fails the run; anything else warns |

Any value other than the exact string `"error"` is treated as `warn`.

### Example

```json
{
  "bannedImports": [
    {
      "name": "moment",
      "message": "moment is in maintenance mode — use date-fns instead",
      "severity": "error"
    },
    {
      "name": "lodash",
      "message": "prefer native array/object methods; import lodash-es per-function if unavoidable",
      "severity": "warn"
    },
    { "name": "request" }
  ]
}
```

`request` above has no `severity`, so it warns, and no `message`, so the report reads
`banned dependency: request`.

## Disabling

Remove the entries, or drop the whole key. To keep the bans configured but stop them failing the
run:

```json
{ "severity": { "Banned Imports": "warn" } }
```

Or switch the check off completely:

```json
{ "severity": { "Banned Imports": "off" } }
```

## Notes

- Findings are grouped by `message`, and the log prints up to **3 example lines** per group so a
  large PR does not flood the report. The count in the summary is the full total.
- Each matched line is truncated to 100 characters in the log.
- When the diff cannot be read the check returns `skip('diff unavailable')` rather than failing.
- Detection is line-based pattern matching, not import resolution. A dynamic
  `await import(pkgName)` built from a variable is not caught.

---

[Checks Index](../INDEX.md) · **Banned Imports** · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)
