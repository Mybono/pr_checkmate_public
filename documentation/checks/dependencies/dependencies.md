# Dependencies

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · **Dependencies** · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)

---

## Overview

Runs [depcheck](https://github.com/depcheck/depcheck) to compare what the code imports against what
`package.json` declares, and reports the two ways those can disagree:

| Finding | Outcome | Why it matters |
|---|---|---|
| **Missing** — imported but not declared | `fail` | The build works locally only because a transitive dependency happens to provide it. It breaks the moment that package changes its own tree |
| **Unused** — declared but never imported | `warn` | Dead weight in the install and in the vulnerability surface, but harmless to correctness |

Missing dependencies are checked first and short-circuit: when any are found the check fails and
unused ones are not reported in the same run.

| Property | Value |
|---|---|
| Display name | `Dependencies` |
| Phase | `blocking` |
| CLI command | `npx pr-checkmate deps` (alias `dependencies`) |
| Config key | `dependency` |
| Source | `src/core/checks/dependencies/dependency-check.ts` |

## When it applies

Both conditions must hold:

1. `dependency.enabled` is not `false`
2. a `package.json` exists at the repository root

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `dependency.enabled` | boolean | `true` | Set `false` to skip the check |
| `dependency.ignore` | string[] | `[]` | Dependency names or globs depcheck should not consider |

### Examples

Silence packages that depcheck cannot see being used — a common case for tooling loaded by
configuration rather than by an `import` statement:

```json
{
  "dependency": {
    "ignore": ["@types/*", "ts-node", "eslint-plugin-*"]
  }
}
```

Globs are passed straight through to depcheck's `--ignores`.

## Disabling

```json
{ "dependency": { "enabled": false } }
```

Or demote the missing-dependency failure to a warning:

```json
{ "severity": { "Dependencies": "warn" } }
```

## Notes

- `dependency.ignore` is forwarded as `--ignores` **only when non-empty**. This is deliberate: the
  depcheck CLI flag overrides any `.depcheckrc` in the repository, so passing an empty list would
  silently disable a project's own depcheck configuration. Leave the key unset and your
  `.depcheckrc` continues to apply.
- The bundled depcheck binary is executed with the current Node binary rather than through `npx`,
  which would exit 127 when depcheck is not on `PATH`.
- depcheck exits non-zero when it finds issues but still writes JSON to stdout, so the check reads
  the output rather than trusting the exit code. If no JSON can be located at all it returns `skip`;
  if JSON is present but unparseable it returns `fail`.
- False positives on "unused" are common for packages consumed indirectly (build plugins, type-only
  packages, CLI tools invoked from npm scripts). That is why unused only warns.

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · **Dependencies** · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)
