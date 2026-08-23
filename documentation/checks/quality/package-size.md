# Package Size

[Docs](../../README.md) · [Checks Index](../INDEX.md) · [Broken Symlinks](symlinks.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · [Dead Code](dead-code.md) · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · **Package Size** · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Reports how much the published npm package grew since its last release.

```text
⚠️ Package Size: 1.07 MB → 1.17 MB (+9.6%) vs 2.1.2:
  dist/core/checks/languages/external-checks.js +0.01 MB (new)
  dist/core/install/prepare.js +0.01 MB (new)
```

**Compared against your previous release, not against a threshold you had to
invent.** The question is whether the package is growing unnoticed, and a limit
nobody configured is a limit nobody trusts. The files behind the growth are named
too: a percentage on its own is a finding you cannot act on.

| Property | Value |
|---|---|
| Display name | `Package Size` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate package-size` |
| Config key | `packageSize` |
| Toolchain | Bundled — uses `npm pack`, nothing to install |
| Source | `src/core/checks/quality/package-size.ts` |

## When it applies

`packageSize.enabled` is not `false`, **and** `package.json` names a package that
is not `private`. A private package is never published, so there is nothing to
compare against and no consumer paying for the size.

`skip` — never `pass` — when the registry cannot answer or nothing has been
published under that name yet. A green row for a comparison that did not happen
would say the opposite of the truth.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `packageSize.enabled` | boolean | `true` | Set `false` to skip the check |
| `packageSize.maxGrowthPercent` | number | `20` | Growth above this, as a percentage of the previous release's unpacked size, earns a row |
| `packageSize.compareWith` | string | `"latest"` | dist-tag or exact version to compare against |

```json
{ "packageSize": { "maxGrowthPercent": 10, "compareWith": "next" } }
```

## Disabling

```json
{ "packageSize": { "enabled": false } }
```

Or `{ "severity": { "Package Size": "off" } }`.

## Notes

- **Advisory by design.** Growth is usually earned — a language profile added, a
  WASM binary landed. Make it a gate with
  `severity: { "Package Size": "error" }` if your repository wants one.
- **Nothing is written to your working tree.** `npm pack --dry-run` describes the
  tarball without producing one, for the current tree and for the published
  version alike.
- **`--ignore-scripts`, always.** `npm pack` runs `prepare` and `prepack`, which in
  your repository are arbitrary commands — often a full build. A check that
  executes them has become a build step with side effects nobody asked for.
- Sizes are **unpacked** on both sides, which is what the registry reports for a
  published version and what a consumer's `node_modules` actually costs.
