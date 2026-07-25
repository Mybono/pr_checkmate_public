# NPM Audit

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · **NPM Audit** · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)

---

## Overview

Runs `npm audit --json` and fails the run on vulnerabilities in **direct** dependencies at or above
a severity threshold.

The word *direct* is the whole design of this check. `npm audit` reports the entire transitive tree,
but a consumer cannot fix a vulnerability buried three levels down in someone else's package — only
the dependencies declared in their own `package.json`. So this check keeps `isDirect` findings and
discards the rest. Two further exclusions apply:

- **pr-checkmate itself** is skipped. It installs into the consumer's `node_modules`, so its own
  dependency tree shows up in their audit. Failing someone's PR for a vulnerability pr-checkmate
  introduced would be unfixable by them.
- **Packages in `ignore`** never block, but are still logged as warnings — an exception is recorded,
  not silently dropped.

| Property | Value |
|---|---|
| Display name | `NPM Audit` |
| Phase | `blocking` |
| CLI command | `npx pr-checkmate npm-audit` |
| Config key | `security.npm-audit` |
| Source | `src/core/checks/dependencies/npm-audit.ts` |

## When it applies

Both conditions must hold:

1. `security.npm-audit.enabled` is not `false`
2. a `package.json` exists at the repository root

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `security.npm-audit.enabled` | boolean | `true` | Set `false` to skip the check entirely |
| `security.npm-audit.auditLevel` | `info` \| `low` \| `moderate` \| `high` \| `critical` | `"moderate"` | Minimum severity that counts. Anything below it is ignored |
| `security.npm-audit.ignore` | string[] | `[]` | Package names that never block, even when direct and at/above `auditLevel`. Still reported as warnings |

Severity is ranked `info < low < moderate < high < critical`, and the comparison is inclusive — the
default `moderate` blocks on `moderate`, `high`, and `critical`.

### Examples

Raise the bar so only high and critical findings block:

```json
{
  "security": {
    "npm-audit": { "auditLevel": "high" }
  }
}
```

Accept a known advisory in one package while keeping the check active everywhere else:

```json
{
  "security": {
    "npm-audit": {
      "auditLevel": "moderate",
      "ignore": ["markdownlint-cli2"]
    }
  }
}
```

## Disabling

```json
{ "security": { "npm-audit": { "enabled": false } } }
```

Or, without touching the check's own config, demote it so a finding warns instead of failing:

```json
{ "severity": { "NPM Audit": "warn" } }
```

## Fixing a finding rather than suppressing it

When `npm audit` proposes a fix that is a **major downgrade** (`fixAvailable.isSemVerMajor: true`),
check whether a patched version of the *transitive* package exists first. A direct dependency is
often flagged only because it pins a vulnerable sub-dependency, and an `overrides` entry fixes it
without losing the newer major:

```json
{
  "overrides": {
    "markdownlint-cli2": { "js-yaml": "5.2.2" }
  }
}
```

Verify with `npm ls <package> --all`, which marks substituted versions as `overridden`. This keeps
the dependency current instead of rolling it back, and is preferable to adding the package to
`ignore`.

## Notes

- The check reads `npm audit --json` with `reject: false`: npm exits non-zero when it finds
  vulnerabilities but still emits usable JSON on stdout.
- For ecosystems other than npm, use [Vuln Scan](vuln-scan.md) (osv-scanner) or
  [Grype Scan](grype-scan.md).

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · **NPM Audit** · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)
