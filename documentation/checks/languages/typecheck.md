# TypeScript

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · **TypeScript** · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)

---

## Overview

Type-checks the project with `tsc --noEmit`. The compiler is resolved from the project's own
`node_modules` first, so a `tsconfig.json` is compiled by the TypeScript version it was written
for, falling back to the version bundled with pr-checkmate.

Unlike every other check in this category, TypeScript is never scoped to the diff. `tsc` needs
the whole program graph to resolve types correctly — a change to one file's exported type can
break a distant, unmodified consumer — so this check always runs `tsc --noEmit` against the
entire project, in a PR or not.

| Property | Value |
|---|---|
| Display name | `TypeScript` |
| Phase | `blocking` |
| CLI command | `npx pr-checkmate typecheck` |
| Config key | `typecheck` |
| Toolchain | Bundled |
| Source | `src/core/checks/languages/typecheck.ts` |

## When it applies

All of the following must hold:

1. `typecheck.enabled` is not `false`
2. `ctx.languages` includes `typescript`
3. a `tsconfig.json` file exists at the repository root

At run time there is one more gate, checked after `applies()`: `node_modules` must exist in the
project. `tsc` needs every installed `@types` package and everything listed in `tsconfig.json`'s
`types` to resolve; running it via `npx pr-checkmate` without a prior install produces a wall of
cryptic `TS2688` "cannot find type definition file" errors that look like code problems but are
really a missing `npm ci`. This check reports `skip` with that exact suggestion instead of
failing the PR over a setup gap.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `typecheck.enabled` | boolean | `true` | Set `false` to skip the check |

There is nothing else to configure — `tsc` reads the project's own `tsconfig.json`, which
pr-checkmate never modifies.

## Disabling

```json
{ "typecheck": { "enabled": false } }
```

Or:

```json
{ "severity": { "TypeScript": "off" } }
```

## Notes

- No delta mode: this is the one language check that always does a full-project scan, even
  inside a PR with a `baseSha`/`headSha` range available.
- A missing `node_modules` produces `skip`, not `fail` — a missing install step never fails a
  PR by itself.
- The failure summary counts `error TS\d+` occurrences in `tsc`'s combined stdout/stderr; the
  full output is logged via `logger.warn` before the count is computed.
- `tsc` runs with `reject: false`. A non-zero exit whose output contains no `error TS` diagnostic is
  not a type error — `tsc` was killed (an out-of-memory on a large project), crashed, or never
  started — so it is reported as `tsc failed to run (exit N)` at `warn`, not as a failure. Blocking a
  PR with `0 type error(s)` named nothing the author could fix.

---

[Checks Index](../INDEX.md) · [ESLint](eslint.md) · **TypeScript** · [Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · [Python Types](python-typecheck.md) · [C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md)
