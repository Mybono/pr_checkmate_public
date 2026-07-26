# Circular Deps

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · **Circular Deps** · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)

---

## Overview

Detects import cycles in TypeScript and JavaScript using [dpdm](https://github.com/acrazing/dpdm).
A cycle means module `A` imports `B` which (directly or through a chain) imports `A` again.

Cycles are worth knowing about because their symptoms appear far from their cause: a module evaluates
before its dependency has finished loading, so an import that is a valid function at runtime is
`undefined` during module evaluation. The resulting `TypeError` points at the consumer, not at the
cycle.

Findings are **advisory by default** (`severity: "warn"`). Most codebases carry long-standing cycles
through barrel files, so a repo gets to see its own before anything gates on them — set
`circularDeps.severity: "error"` to fail the run instead.

| Property | Value |
|---|---|
| Display name | `Circular Deps` |
| Phase | `blocking` (findings are `warn` by default — see Configuration) |
| CLI command | `npx pr-checkmate circular` |
| Config key | `circularDeps` |
| Source | `src/core/checks/dependencies/circular-deps.ts` |

dpdm ships **bundled with pr-checkmate**, so nothing is required on the runner. When the project
being checked has its own `dpdm`, that copy is used instead — the same order `ESLint` and
`TypeScript` follow for eslint and tsc.

## When it applies

TypeScript **or** JavaScript is among the detected languages, and `circularDeps.enabled` is not
`false`. It scans git-tracked `*.ts`, `*.tsx`, `*.js`, and `*.jsx` files within `sourcePath`, and
passes when no such files exist.

## Configuration

| Key | Default | Effect |
|---|---|---|
| `circularDeps.enabled` | `true` | `false` removes the check from the run |
| `circularDeps.severity` | `"warn"` | `"error"` fails the run on any cycle |
| `circularDeps.ignore` | `[]` | Globs over repo-relative paths kept out of the analysis, and dropped from any cycle they appear in |
| `circularDeps.timeoutMs` | `120000` | Cap on the analysis; on timeout the check skips |

These global scoping options apply as well:

| Key | Effect here |
|---|---|
| `sourcePath` | Restricts which directories are scanned |
| `ignoreDirs` | Directory names excluded from the file list |
| `severity` | Universal per-check override, including `off` |

### Example

Analyse only application code, tolerate cycles through generated clients and barrel files, and gate
merges on everything that remains:

```json
{
  "sourcePath": "src",
  "ignoreDirs": ["node_modules", "dist", "generated"],
  "circularDeps": {
    "severity": "error",
    "ignore": ["**/*.generated.ts", "src/**/index.ts"]
  }
}
```

## Disabling

```json
{ "circularDeps": { "enabled": false } }
```

Or with the universal switch:

```json
{ "severity": { "Circular Deps": "off" } }
```

Findings are already advisory by default; to gate merges on them set
`circularDeps.severity: "error"` or `{ "severity": { "Circular Deps": "error" } }`.

## Notes

- Cycles are read from dpdm's JSON report (`-o`), whose `circulars` array is the authoritative list —
  not from its console output. Every cycle is written to the log as `a.ts -> b.ts` so it can be
  traced; the first ten are listed, with a count of the rest.
- When dpdm cannot produce a report — an unresolvable binary, a tsconfig it cannot read, a timeout —
  the check **skips**. It never passes on a failed analysis (that would hide a check that did not
  run) and never fails on one (that would block a PR for a tooling problem).
- Not delta-aware. Cycles are a property of the whole import graph, so a full scan of the tracked
  file set is the only meaningful analysis — a cycle can be created by a one-line change in a file
  the PR never touched. On a repo whose file list would overflow the OS argument limit, the scan
  falls back to `sourcePath` globs, which dpdm expands itself over the same tree.
- Type-only cycles (`import type`) are erased at compile time and are harmless at runtime, but dpdm
  analyses the source graph and may still report them. Keep the check at its default `warn` if that
  is noisy for your codebase, or list the files in `circularDeps.ignore`.

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · **Circular Deps** · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)
