# License Check

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · **License Check** · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)

---

## Overview

Fails the run when a **production** dependency ships under a copyleft licence that would impose
source-disclosure obligations on a closed-source product.

The forbidden list is fixed in the check and is not configurable:

| Forbidden |
|---|
| `GPL-2.0` · `GPL-3.0` · `AGPL-3.0` · `LGPL-2.0` · `LGPL-2.1` · `LGPL-3.0` |

Matching is a substring test against the reported licence string, so compound expressions such as
`(MIT OR GPL-3.0)` are caught. Where a package declares several licences they are joined with ` OR `
before matching.

Only production dependencies are inspected (`license-checker --production`) — `devDependencies` are
build-time tooling and do not ship, so their licences are out of scope.

| Property | Value |
|---|---|
| Display name | `License Check` |
| Phase | `blocking` |
| CLI command | `npx pr-checkmate license` |
| Config key | `licenseCheck` |
| Source | `src/core/checks/dependencies/license-check.ts` |

## When it applies

A `package.json` exists at the repository root. There is no `enabled` flag — use
`severity: { "License Check": "off" }` to switch it off.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `licenseCheck.allowPackages` | string[] | pr-checkmate's own cspell dictionaries | Package names exempted regardless of licence |
| `licenseCheck.allowLicenses` | string[] | `[]` | Licence substrings to accept even when they appear on the forbidden list |

Both lists are **appended** to the defaults rather than replacing them, so adding an entry never
silently drops the built-in exemptions.

### Two exemptions you get for free

- **`allowPackages` defaults** already contain `@cspell/dict-es-es`, `@cspell/dict-fr-fr`,
  `@cspell/dict-he`, `@cspell/dict-ru_ru`, and `@cspell/dict-uk-ua`. Some cspell dictionaries ship
  under copyleft terms, but they are *pr-checkmate's* dependencies, not yours.
- **pr-checkmate's entire dependency closure** is resolved at runtime and skipped. Because
  pr-checkmate installs into your `node_modules`, its transitive tree appears in your licence report;
  you can neither fix nor be responsible for it.

### Examples

Exempt a vendor package you have a commercial licence for:

```json
{
  "licenseCheck": {
    "allowPackages": ["some-vendor-sdk"]
  }
}
```

Accept LGPL-3.0 across the project — for instance because you only link dynamically:

```json
{
  "licenseCheck": {
    "allowLicenses": ["LGPL-3.0"]
  }
}
```

`allowLicenses` is checked **before** the forbidden list, so this genuinely overrides the ban.

## Disabling

```json
{ "severity": { "License Check": "off" } }
```

To keep it visible but non-blocking:

```json
{ "severity": { "License Check": "warn" } }
```

## Notes

- The bundled `license-checker` binary is resolved and executed with the current Node binary rather
  than through `npx`, which would exit 127 when the tool is not on `PATH`.
- If `license-checker` is unavailable, or emits output that is not valid JSON, the check returns
  `skip` — a broken tool never fails someone's PR.
- Licence data comes from each package's own metadata. A package that declares its licence
  incorrectly will be classified incorrectly; this check is not a legal audit.

---

[Checks Index](../INDEX.md) · [Banned Imports](banned-imports.md) · [Circular Deps](circular-deps.md) · [Dependencies](dependencies.md) · [Grype Scan](grype-scan.md) · **License Check** · [NPM Audit](npm-audit.md) · [Outdated Deps](outdated-deps.md) · [Vuln Scan](vuln-scan.md)
