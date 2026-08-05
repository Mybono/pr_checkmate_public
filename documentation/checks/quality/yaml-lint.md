# YAML Lint

[Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · **YAML Lint**

---

## Overview

Parses every changed `.yml`/`.yaml` file with the [`yaml`](https://eemeli.org/yaml/) package —
bundled, no runner dependency, always available like [Markdown](markdown-lint.md) and
[Spellcheck](spellcheck.md) — and reports the parser's own errors and warnings: syntax errors,
duplicate map keys, tabs used as indentation, and other structural problems. Multi-document files
(`---`-separated) have each document parsed and reported independently.

Unlike every other check in this category, **YAML Lint is `blocking`**, not `informational` — a YAML
syntax error is cheap to detect and unambiguous, so it runs first and, by default, fails the run
rather than merely warning.

| Property | Value |
|---|---|
| Display name | `YAML Lint` |
| Phase | `blocking` |
| CLI command | `npx pr-checkmate yaml-lint` |
| Config key | `yamlLint` |
| Source | `src/core/checks/quality/yaml-lint.ts` |

## When it applies

`yamlLint.enabled` is not `false` — it's **on by default**. There is no language or tool prerequisite;
it needs nothing installed beyond the bundled `yaml` package.

Target files come from the normal delta/full-scan resolution, but with one deliberate exception: the
global `ignoreDirs` exclusion for `.github`, `.gitlab`, and `.circleci` is un-ignored specifically for
this check (`keepDirs`), because those directories hold the CI/workflow YAML a linter most needs to
see, even though they're excluded from every other check by default.

Returns `skip('git unavailable')` if git can't be queried, `skip('no YAML files')` when nothing
matches, or `skip('all YAML files ignored')` if every match is filtered out by `yamlLint.ignore`.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `yamlLint.enabled` | boolean | `true` | Set `false` to skip the check |
| `yamlLint.severity` | `'error' \| 'warn'` | `error` | `error` fails the run (default); `warn` downgrades findings to advisory |
| `yamlLint.ignore` | string[] (globs) | `[]` | Files/paths excluded from linting |

### Example

Downgrade to advisory and skip a generated Helm chart directory:

```json
{
  "yamlLint": {
    "severity": "warn",
    "ignore": ["charts/**", "**/*.generated.yml"]
  }
}
```

## Disabling

```json
{ "yamlLint": { "enabled": false } }
```

To keep it running but stop it from blocking a merge:

```json
{ "yamlLint": { "severity": "warn" } }
```

Or remove it from the run entirely by display name:

```json
{ "severity": { "YAML Lint": "off" } }
```

## Notes

- **Application-specific tags are not defects.** A problem the parser reports as
  `TAG_RESOLVE_FAILED` — "Unresolved tag: `!Ref`" — means it met a tag whose schema belongs to
  another application, and YAML is extensible there by design. CloudFormation and SAM templates
  (`!Ref`, `!Sub`, `!GetAtt`, `!Equals`, `!If`, `!ImportValue`), Home Assistant (`!include`) and
  Ansible (`!vault`) all rely on it, so those problems are filtered out. Before that, a single
  valid CloudFormation template produced 557 findings and, because this check is blocking, failed
  the run. Real defects — syntax errors, duplicate keys, tab indentation — are unaffected.
- `yaml`'s `parseAllDocuments` never throws — it collects problems on each document instead, so one
  malformed file can never abort the whole check or hide problems in files parsed after it.
- Findings are listed with `file:line:col` when position information is available; up to **10** are
  logged individually, with `... and N more` appended beyond that — the same convention as
  [License Header](license-header.md) and [Config Validation](config-validation.md).
- `yamlLint.ignore` filters by **file path** (standard glob matching), which is different from the
  `<check>.ignore` convention described in the [Checks Index](../INDEX.md#configuring-any-check) for
  checks like `diffSecurity` and `workflowSecurity`, where `ignore` matches a *finding label* as a
  substring instead. For YAML Lint, `ignore` means "skip this file," full stop.
- Both `yamlLint.severity` and the universal `severity: { "YAML Lint": … }` override achieve the same
  result; `yamlLint.severity` is this check's own dedicated key, in addition to the universal
  mechanism every check supports.

---

[Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · **YAML Lint**
