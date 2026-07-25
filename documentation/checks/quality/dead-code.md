# Dead Code

[Checks Index](../INDEX.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · **Dead Code** · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)

---

## Overview

Runs [ts-unused-exports](https://github.com/pzavolinsky/ts-unused-exports) against the project's
`tsconfig.json` to find exports that nothing in the project imports — the TypeScript-aware equivalent
of dead code, since an unused export can't be caught by a linter operating file-by-file.

This is a **whole-project analysis**, not a diff check: `ts-unused-exports` needs the full module
graph to know whether an export is actually unused anywhere, so it always analyzes the entire
`tsconfig.json` project rather than just the files touched by the PR.

| Property | Value |
|---|---|
| Display name | `Dead Code` |
| Phase | `informational` |
| CLI command | — (runs only as part of a full run) |
| Config key | `deadCode` |
| Source | `src/core/checks/quality/dead-code.ts` |

## When it applies

`deadCode.enabled` is not `false`, **and** `typescript` is among the languages detected in the
repository. Unlike most checks in this category, TypeScript is a hard prerequisite rather than an
optional narrowing — a repo with no TypeScript never runs this check at all, and it's one of five
checks with no dedicated CLI command (the others are
[Grype Scan](../dependencies/grype-scan.md), [Merge Conflict](../git/merge-conflict.md), ktlint, and
SwiftLint).

If `applies()` passes but there is no `tsconfig.json` at the repository root, the check returns
`skip('no tsconfig.json found')` at run time.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `deadCode.enabled` | boolean | `true` | Set `false` to skip the check |
| `deadCode.ignoreFiles` | string[] (regex sources) | `["/__tests__/", "/__mocks__/", "/__mock__/", "/tests?/", "\\.test\\.", "\\.spec\\."]` | Files exempted from the *report*. **Replaces** the default list |

### Example

Also exempt a generated directory from the report:

```json
{
  "deadCode": {
    "ignoreFiles": [
      "/__tests__/",
      "/__mocks__/",
      "/__mock__/",
      "/tests?/",
      "\\.test\\.",
      "\\.spec\\.",
      "/generated/"
    ]
  }
}
```

Remember these are regex *sources*, so a literal backslash needs escaping (`\\.`) inside the JSON
string, and setting the key replaces the whole default list rather than adding to it.

## Disabling

```json
{ "deadCode": { "enabled": false } }
```

Or:

```json
{ "severity": { "Dead Code": "off" } }
```

## Notes

- `ignoreFiles` entries are regex sources matched against **reported file paths**, not glob patterns,
  and not `ts-unused-exports`' own `--ignoreFiles` flag. That distinction is deliberate: `ts-unused-
  exports`' native `--ignoreFiles` drops a file from *analysis* entirely, so its imports stop counting
  as usage — exempting a barrel or entry point that way would cascade and flag everything it
  re-exports as unused too. Filtering the *output* instead keeps every usage edge intact and only
  hides the exempted files' own findings.
- Defaults exempt test and mock files, since their exports are typically consumed only by the test
  runner (`jest.mock`, dynamic imports) — usage `ts-unused-exports` has no way to see.
- An invalid regex in `ignoreFiles` is skipped with a warning rather than aborting the whole check.
- If `ts-unused-exports` itself is unavailable or exits abnormally with no usable output, the check
  returns `skip('ts-unused-exports unavailable')` rather than failing the run.
- The tool prints one summary line (`N modules with unused exports`) followed by one line per file in
  `path: sym1, sym2` form; the check parses that format directly rather than using a JSON reporter.

---

[Checks Index](../INDEX.md) · [Case Collision](case-collision.md) · [Config Validation](config-validation.md) · [Coverage](coverage.md) · **Dead Code** · [Duplicate Code](duplicate-code.md) · [License Header](license-header.md) · [Markdown](markdown-lint.md) · [Spellcheck](spellcheck.md) · [YAML Lint](yaml-lint.md)
