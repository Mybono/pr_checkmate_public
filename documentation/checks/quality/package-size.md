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

**Where it looks.** At `publishConfig.registry` when your manifest names one, and
at the default registry otherwise. npm reads that field for `publish` but not for
a fetch, so a package on GitHub Packages or a company mirror has to be asked for
explicitly — otherwise the answer is a 404 about the wrong host.

`skip` — never `pass` — when there is nothing to compare against. The reason says
which of three things happened, because they are not the same problem:

| Reason | What it means | What to do |
|---|---|---|
| `nothing published under this name yet` | 404. Nobody has published it | Nothing. The row starts comparing after your first release |
| `no credentials for <host>` | 401 or 403 — the registry has the package and will not show it to a stranger; an unpublished name answers 404, not 401 | See below, or `packageSize.enabled: false` |
| `could not read the published version` | Anything else — a proxy, a timeout, wording we do not recognise | Usually transient |

### Reading a private registry

The row says only what happened; the fix does not fit in a report line. For
GitHub Packages in Actions it is three things, and all three are needed — npm
reads the token through a reference in `.npmrc`, so the variable alone does
nothing:

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: 24
    registry-url: https://npm.pkg.github.com   # writes the .npmrc
    scope: '@your-org'
- run: npx pr-checkmate all
  env:
    NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

For any other private registry, an `.npmrc` with `//host/:_authToken=…` does the
same job. If you would rather not wire credentials for an advisory check, switch
it off — the row is not worth a token you did not want to issue.

A green row for a comparison that did not happen would say the opposite of the
truth, and one sentence covering all three would send half the readers looking
for the wrong thing.

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
