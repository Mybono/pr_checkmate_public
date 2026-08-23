# Python Types

[Docs](../../README.md) · [Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) ·
[Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) · **Python Types** ·
[C++ Format](cpp-format.md) · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) · [Go Vet](go-lint.md) ·
[Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) · [C# Format](csharp-format.md) ·
[RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) · [ShellCheck](shellcheck.md)

---

## Overview

Type-checks changed Python files — the Python analogue of [TypeScript](typecheck.md)'s
`tsc --noEmit`.

**No Python install required.** Checker selection is: the runner's own `mypy` when it has one, our
bundled `pyright` otherwise. Pyright is a Node bundle carrying its own typeshed, so it needs neither
Python nor pip — which is what stopped `⏭️ mypy/pyright not installed` from being the answer on most
runners. mypy comes first because a repository that installed mypy configured it too (`mypy.ini`,
`[tool.mypy]`), and its settled opinion about its own code beats ours. Absence of mypy is detected via
`code: 'ENOENT'` on the result object (execa v9 with `reject: false` resolves rather than throws on a
missing binary), not a try/catch.

Set `python.typeChecker` to pin one. Pinning `"mypy"` on a runner without it is a `skip`, not a
silent fall back to pyright: naming a checker is a decision, and substituting the other one would hide
that it was never honoured.

**Dependencies must be installed.** Pyright resolves imports against what is on disk, and a CI job
that never ran `pip install` has nothing to resolve against. When any import cannot be resolved the
check returns `skip` naming the modules and telling you to `pip install` — exactly what
[TypeScript](typecheck.md) does with a missing `node_modules`, and for the same reason: nothing
downstream of an unresolved import can be trusted. A `@click.group()` whose `click` is absent makes
`main` a plain function, and then every `@main.command(...)` under it is reported as a type error. On
a real client repository that was nine findings out of fifteen (ticket 076) — a wall confident enough
that the honest reading of it is "we cannot answer yet".

Add whatever your project uses to the CI job — `pip install -e .`, `pip install -r
requirements.txt`, `uv sync` — and the check starts reporting. A repository with a pyright config of
its own (`pyrightconfig.json`, or a `[tool.pyright]` section in `pyproject.toml`) is exempt: having
written one is having an opinion about import resolution — a `venvPath`, or these rules downgraded
deliberately — and a skip would override the one client who said what they wanted.

| Property | Value |
|---|---|
| Display name | `Python Types` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate python-typecheck` |
| Config key | `python.typecheck` |
| Toolchain | `pyright`, **downloaded on first use** — 34 MB only Python repositories need. The runner's own `mypy` is preferred when present. Offline, cached or air-gapped runners: [Running in CI](../../ci-setup.md#air-gapped-and-cached-runners) |
| Source | `src/core/checks/languages/python-typecheck.ts` |

## When it applies

All of the following must hold:

1. `python.enabled` is not `false`
2. `python.typecheck` is not `false`
3. `ctx.languages` includes `python`

Target files are `*.py`, resolved through the plain (not source-path-scoped) target resolver —
**`sourcePath` does not narrow which files are type-checked**; only delta mode and `ignoreDirs`
do. If no target files are found, the check passes without invoking either type checker.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `python.enabled` | boolean | `true` | Set `false` to disable every Python check (lint, format, and types together) |
| `python.typecheck` | boolean | `true` | Set `false` to skip just this check |
| `python.typeChecker` | `"mypy"` \| `"pyright"` | unset — the runner's `mypy` if present, else our bundled `pyright` | Pin one checker. `"mypy"` on a runner without it is a `skip`, not a fall back |

### Example

Force pyright instead of the default mypy-first order:

```json
{
  "python": {
    "typeChecker": "pyright"
  }
}
```

## Disabling

```json
{ "python": { "typecheck": false } }
```

Or:

```json
{ "severity": { "Python Types": "off" } }
```

## Notes

- Nothing needs installing on the runner. Add `pip install mypy` to a setup step only if you want
  mypy's opinion (and your own mypy config) rather than pyright's.
- `pytypes` is a separate install profile from `python`, because pyright is 19 MB against Ruff's 12
  and the two answer different questions. `python.typecheck: false` therefore stops the download as
  well as the check — a language profile decided on file counts alone would have fetched it anyway.
- **mypy route:** output lines are filtered to those containing `error` or `warning`
  (case-insensitive), capped at 10 with `... and N more`. The summary prefers mypy's own
  "Found N errors" line, otherwise `<N> type issue(s)`.
- **pyright route:** `--outputjson` is parsed and `information`-level diagnostics dropped. If any
  `reportMissingImports` or `reportMissingModuleSource` survives, the check skips as described above
  instead of reporting; otherwise the summary is `<N> error(s), <M> warning(s)`. Line numbers are
  re-based to 1, since pyright counts from 0 and every other row in the report counts from 1.
- Always reports `warn` when issues are found, never `fail` — matching [Ruff Lint](python-lint.md)'s
  advisory-by-default design. Promote it with `severity: { "Python Types": "error" }` once a
  team is ready to gate on it.
