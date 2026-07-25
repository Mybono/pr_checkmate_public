# Vuln Scan

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · **Vuln Scan**

---

## Overview

Cross-language dependency vulnerability scan via
[osv-scanner](https://github.com/google/osv-scanner), which reads lockfiles across ecosystems — npm,
PyPI, Go, crates.io, Maven and others — and matches them against the
[OSV database](https://osv.dev). It exists to cover everything [NPM Audit](npm-audit.md) cannot,
since that check is npm-only.

This is the one dependency check that is **opt-in**: it runs only when explicitly enabled.

| Property | Value |
|---|---|
| Display name | `Vuln Scan` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate vuln-scan` |
| Config key | `vulnScan` |
| Source | `src/core/checks/dependencies/vuln-scan.ts` |

## When it applies

`vulnScan.enabled` is **exactly `true`**. Unlike every other check in this category, omitting the key
leaves it off — the gate is `=== true`, not `!== false`.

`osv-scanner` is a *runner dependency*. When the binary is absent the check returns
`skip('osv-scanner not installed')`, so enabling it on a runner that lacks the tool is safe but
pointless.

Install it on a GitHub Actions runner with:

```yaml
- name: Install osv-scanner
  run: |
    curl -sSfL -o /usr/local/bin/osv-scanner \
      https://github.com/google/osv-scanner/releases/latest/download/osv-scanner_linux_amd64
    chmod +x /usr/local/bin/osv-scanner
```

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `vulnScan.enabled` | boolean | `false` | Must be `true` for the check to run |

That is the entire surface. There is no severity threshold and no ignore list — if you need either,
use [Grype Scan](grype-scan.md), which offers `minSeverity` and `ignore`.

### Example

```json
{
  "vulnScan": { "enabled": true }
}
```

## Disabling

Omit the key, or set it explicitly:

```json
{ "vulnScan": { "enabled": false } }
```

To make findings fail the run rather than warn:

```json
{ "severity": { "Vuln Scan": "error" } }
```

## Notes

- Runs `osv-scanner -r .` recursively from the repository root.
- Exit code `0` means no vulnerabilities and the check passes. Any other exit code is treated as
  findings, and the output is filtered to lines mentioning `CVE-`, `GHSA-`, `vulnerab`, or `advisor`.
- The log lists at most **10** matching lines, then `... and N more`.
- Unlike [NPM Audit](npm-audit.md) and [Grype Scan](grype-scan.md), this check does **not** filter to
  direct dependencies — it reports whatever osv-scanner finds, including transitive packages you
  cannot fix directly. Expect more noise, and correspondingly broader coverage.
- The count in the summary is a count of matching output *lines*, not of distinct vulnerabilities.

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · **Vuln Scan**
