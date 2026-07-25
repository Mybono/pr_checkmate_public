# Coverage

[Checks Index](../INDEX.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · **Coverage** · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Warns when no coverage report is found on disk. It does **not** run your tests, does **not** parse a
coverage percentage, and does **not** enforce a coverage threshold — it only checks that *something*
produced a report, so a broken `--coverage` flag or a test step that silently stopped collecting
coverage doesn't go unnoticed.

Because coverage reports are almost always git-ignored, the check probes the working tree with `fs`,
not `git` — it looks for a file at a known path and confirms it has non-zero size.

| Property | Value |
|---|---|
| Display name | `Coverage` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate coverage` |
| Config key | `coverage` |
| Source | `src/core/checks/quality/coverage.ts` |

## When it applies

`coverage.enabled` is **exactly `true`**. Like [Vuln Scan](../dependencies/vuln-scan.md), this is an
opt-in check gated with `=== true`, not `!== false` — omitting the key leaves it off, since a
presence-only warning would be pure noise on every repo that doesn't collect coverage at all.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `coverage.enabled` | boolean | `false` | Must be `true` for the check to run |
| `coverage.severity` | `'error' \| 'warn'` | `warn` | `warn` advises; `error` fails the run |
| `coverage.reportPath` | string \| string[] | built-in probe list (see below) | Explicit report location(s). **Replaces** the default list rather than adding to it |

### The built-in probe list

When `reportPath` is unset, the check looks for any of these, in order, and stops at the first one
found:

```text
coverage/lcov.info                              # JS/TS (lcov), C++
lcov.info
coverage/coverage-final.json                     # istanbul
coverage/coverage-summary.json
coverage/cobertura-coverage.xml                  # Cobertura (JS, Python)
coverage/clover.xml
coverage.xml                                     # Python (pytest-cov / coverage.py)
coverage.out                                     # Go
build/reports/jacoco/test/jacocoTestReport.xml   # Kotlin/Java (Gradle)
target/site/jacoco/jacoco.xml                    # Java (Maven)
```

### Example

Enable the check with a non-standard report path:

```json
{
  "coverage": {
    "enabled": true,
    "reportPath": "reports/coverage/lcov.info"
  }
}
```

`reportPath` also accepts an array — the check passes the moment any one of them is found:

```json
{
  "coverage": {
    "enabled": true,
    "reportPath": ["packages/api/coverage/lcov.info", "packages/web/coverage/lcov.info"]
  }
}
```

## Disabling

Omit the key, or set it explicitly:

```json
{ "coverage": { "enabled": false } }
```

To make a missing report fail the run rather than warn:

```json
{ "coverage": { "severity": "error" } }
```

## Notes

- A found report only needs `size > 0` — the check never opens or parses it, so it cannot detect a
  report that exists but reflects 0% coverage, or one that's stale from a previous run.
- The summary lists up to 3 of the probed locations as examples when nothing is found, so the log
  gives a hint about where it looked without dumping the entire list.
- `coverage.severity` is this check's **own** severity key, independent of the universal
  `severity: { "Coverage": … }` override — both work, and either is enough to promote a missing report
  to a failure.

---

[Checks Index](../INDEX.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · **Coverage** · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)
