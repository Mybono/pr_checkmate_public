# Grype Scan

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · **Grype Scan** · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)

---

## Overview

Cross-ecosystem dependency vulnerability scan via Anchore's [grype](https://github.com/anchore/grype).
Reports only vulnerabilities in **direct** dependencies, and warns rather than fails.

grype resolves versions from lockfiles, so its raw output covers the whole transitive tree — plus
pr-checkmate's own bundled dependencies, since it installs into your `node_modules`. Neither is
something you can fix, so the check narrows the findings to packages declared in your own manifests.
Everything dropped is **counted and logged**, so the suppression is visible rather than silent.

Ecosystems whose manifests can be parsed for direct dependencies:

| grype artifact type | Ecosystem |
|---|---|
| `npm` | npm |
| `python` | Python |
| `go-module` | Go |
| `rust-crate` | Rust |
| `gem` | Ruby |
| `php-composer` | PHP |

A finding in any other ecosystem is suppressed, because there is no manifest to establish whether it
is direct.

| Property | Value |
|---|---|
| Display name | `Grype Scan` |
| Phase | `informational` |
| CLI command | none — runs only as part of a full run |
| Config key | `grypeScan` |
| Source | `src/core/checks/dependencies/grype-scan.ts` |

## When it applies

`grypeScan.enabled` is not `false` — the check is **on by default**.

`grype` is a *runner dependency*: it must be installed on the machine. When the binary is absent the
check returns `skip('grype not installed')`, so leaving it enabled on a runner without grype costs
nothing.

Install it on a GitHub Actions runner with:

```yaml
- name: Install grype
  run: curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
```

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `grypeScan.enabled` | boolean | `true` | Set `false` to skip the check |
| `grypeScan.minSeverity` | `negligible` \| `low` \| `medium` \| `high` \| `critical` | `"medium"` | Minimum severity that is reported. Inclusive |
| `grypeScan.ignore` | string[] | `[]` | Suppress by **CVE/GHSA id** or by **package name**. Case-insensitive |
| `grypeScan.timeoutMs` | number | `120000` | Cap on the scan. On timeout the check skips rather than fails |

`ignore` accepts either kind of entry in the same array — each value is matched against both the
vulnerability id and the artifact name.

### Examples

Only surface high and critical issues:

```json
{
  "grypeScan": { "minSeverity": "high" }
}
```

Suppress one advisory by id and one package wholesale:

```json
{
  "grypeScan": {
    "minSeverity": "medium",
    "ignore": ["CVE-2025-1234", "GHSA-xxxx-yyyy-zzzz", "some-package"]
  }
}
```

## Disabling

```json
{ "grypeScan": { "enabled": false } }
```

Or universally, which also works when the key is absent:

```json
{ "severity": { "Grype Scan": "off" } }
```

To make findings fail the run instead of warning:

```json
{ "severity": { "Grype Scan": "error" } }
```

## Notes

- Runs `grype dir:. -o json -q` in the repository root, with every directory in `ignoreDirs`
  (`node_modules`, `dist`, `build`, `coverage`, …) passed as `--exclude`. This is not a cosmetic
  filter: cataloguing an installed `node_modules` took **355s** on this repository versus **0.8s**
  without it, for the same findings. grype reads dependency versions from lockfiles, so nothing
  reportable is lost — and every package found *inside* `node_modules` would have been suppressed
  anyway as third-party (see above).
- `timeoutMs` backstops the run. An advisory check must never be able to look like a hung commit,
  which is what an uncapped directory scan does on a repository with dependencies installed.
- The log lists at most **10** findings, then `... and N more`. The summary count is the full total.
- Because grype scans the directory rather than a diff, this check is not delta-aware: it reports the
  current state of your dependencies regardless of what the PR changed.
- Overlaps with [NPM Audit](npm-audit.md) for npm and with [Vuln Scan](vuln-scan.md) (osv-scanner)
  for everything else. Running more than one is reasonable — they use different vulnerability
  databases — but expect duplicate findings.

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · **Grype Scan** · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)
