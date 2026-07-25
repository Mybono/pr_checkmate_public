# Circular Deps

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · **Circular Deps** · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)

---

## Overview

Detects import cycles in TypeScript and JavaScript using [dpdm](https://github.com/acrazing/dpdm)
and fails the run when it finds any. A cycle means module `A` imports `B` which (directly or through
a chain) imports `A` again.

Cycles are worth blocking because their symptoms appear far from their cause: a module evaluates
before its dependency has finished loading, so an import that is a valid function at runtime is
`undefined` during module evaluation. The resulting `TypeError` points at the consumer, not at the
cycle.

| Property | Value |
|---|---|
| Display name | `Circular Deps` |
| Phase | `blocking` |
| CLI command | `npx pr-checkmate circular` |
| Config key | none |
| Source | `src/core/checks/dependencies/circular-deps.ts` |

## When it applies

TypeScript **or** JavaScript is among the detected languages. It scans git-tracked `*.ts`, `*.tsx`,
`*.js`, and `*.jsx` files within `sourcePath`, and passes when no such files exist.

## Configuration

This check has **no dedicated config key**. Its behaviour is shaped only by the global scoping
options:

| Key | Effect here |
|---|---|
| `sourcePath` | Restricts which directories are scanned |
| `ignoreDirs` | Directory names excluded from the file list |
| `severity` | The only way to downgrade or disable it |

### Example

Analyse only application code, leaving generated clients out of the graph:

```json
{
  "sourcePath": "src",
  "ignoreDirs": ["node_modules", "dist", "generated"]
}
```

## Disabling

There is no `enabled` flag. Use the universal switch:

```json
{ "severity": { "Circular Deps": "off" } }
```

To keep the report but stop it blocking merges:

```json
{ "severity": { "Circular Deps": "warn" } }
```

## Notes

- Runs `npx dpdm --no-warning --no-tree --circular <files>`. The full cycle listing from dpdm is
  written to the log so each cycle can be traced.
- dpdm's *success* message also contains the word "circular", so the check does not grep for it
  loosely. It matches only dpdm's two failure shapes: a `Circular Dependencies:` header, or a
  numbered entry such as `[1] src/a.ts -> src/b.ts`.
- Not delta-aware. Cycles are a property of the whole import graph, so a full scan of the tracked
  file set is the only meaningful analysis — a cycle can be created by a one-line change in a file
  the PR never touched.
- Type-only cycles (`import type`) are erased at compile time and are harmless at runtime, but dpdm
  analyses the source graph and may still report them. If this is noisy for your codebase, demote the
  check to `warn`.

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · **Circular Deps** · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · [License Check](license-check.md) · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)
