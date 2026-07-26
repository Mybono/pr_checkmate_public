# Dependencies

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · **Dependencies** · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)

---

## Overview

Runs [knip](https://knip.dev) to compare what the code imports against what `package.json` declares,
and reports the two ways those can disagree:

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
| `dependency.ignore` | string[] | `[]` | Dependency names, or globs over the whole name, that knip should not consider |

### Examples

Silence packages that knip cannot see being used — a common case for tooling loaded by
configuration rather than by an `import` statement:

```json
{
  "dependency": {
    "ignore": ["@types/*", "ts-node", "eslint-plugin-*"]
  }
}
```

The list becomes knip's `ignoreDependencies`, with each entry compiled to a pattern anchored to the
whole dependency name. That anchoring matters: knip reads a value containing `*` as an unanchored
regular expression, so a raw `eslint-*` would also silence `eslint` itself. Written as documented
here, `eslint-plugin-*` covers `eslint-plugin-import` and leaves `eslint` reported.

## Disabling

```json
{ "dependency": { "enabled": false } }
```

Or demote the missing-dependency failure to a warning:

```json
{ "severity": { "Dependencies": "warn" } }
```

## Notes

- `dependency.ignore` is written to a temporary `.knip.temp.json` **only when non-empty**, and never
  when the repository already has a knip config (`knip.json`, `knip.config.ts`, a `knip` key in
  `package.json`, …). A repo that configures knip itself keeps that file as the single source of
  truth; pr-checkmate does not layer its own config on top. When both exist the check logs that the
  `ignore` list was skipped, so a list that has no effect never looks like a broken check.
- The bundled knip binary is executed with the current Node binary rather than through `npx`, which
  would exit 127 when knip is not on `PATH`.
- The check reads knip's JSON report rather than trusting the exit code. knip writes that report as a
  single line, so the report is found by scanning stdout's lines from the end for the last one that
  parses and carries `issues`. Anything else on stdout is noise — knip imports the project's own
  config files while analyzing (a `wdio.conf.ts`, say), so a dotenv banner or a top-level
  `console.log` in a config module is printed ahead of the report, braces and all.
- When no report can be found, the check returns `skip`. It never fails on unparseable output: a
  tool that misbehaved is not a defect in the client's manifest, and blocking a PR for it reports our
  problem as theirs.
- Only knip's `dependencies` and `unlisted` issue types are read. Unused **dev**Dependencies,
  `binaries` and `unresolved` paths are deliberately ignored: CLI tools and shell commands
  legitimately have no import to find, so reporting them would be noise.
- False positives on "unused" are common for packages consumed indirectly (build plugins, type-only
  packages, CLI tools invoked from npm scripts). That is why unused only warns.

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · **Dependencies** · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)
