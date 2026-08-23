# ♟️ PR CheckMate

[![npm](https://img.shields.io/npm/v/pr-checkmate?label=npm&color=CB3837&logo=npm&logoColor=white)](https://www.npmjs.com/package/pr-checkmate)
[![Downloads](https://img.shields.io/npm/dm/pr-checkmate?color=blue)](https://www.npmjs.com/package/pr-checkmate)
![License](https://img.shields.io/badge/license-Proprietary-lightgrey)
[![Security scan](https://img.shields.io/badge/security-SBOM_%2B_VirusTotal-2ea44f)](https://github.com/Mybono/pr_checkmate_public/releases/latest)
![ESLint](https://img.shields.io/badge/ESLint-10-4B32C3?logo=eslint&logoColor=white)
![Prettier](https://img.shields.io/badge/Prettier-3-F7B93E?logo=prettier&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-6-3178C6?logo=typescript&logoColor=white)
![cspell](https://img.shields.io/badge/cspell-10-4285F4)
![jscpd](https://img.shields.io/badge/jscpd-5-blue)
[![API Docs](https://img.shields.io/badge/API_docs-TypeDoc-9600FF?logo=readthedocs&logoColor=white)](https://pr-checkmate-docs.pages.dev)
[![Container](https://img.shields.io/badge/ghcr.io-pr--checkmate-2496ED?logo=docker&logoColor=white)](https://github.com/users/Mybono/packages/container/package/pr-checkmate)

> A security-first PR gate for CI: secret scanning, dependency vulnerability checks, and
> a CycloneDX SBOM plus an independent
> [VirusTotal scan](https://github.com/Mybono/pr_checkmate_public/releases/latest) of the
> exact published tarball on every release — plus 53 code-quality checks across 11
> languages, bundled. One npm package, one `pr-checkmate.json`, no per-language toolchain
> setup for the bundled ones.

Language is auto-detected. Checks run in parallel phases, results post as one PR comment
that updates in place, and formatter fixes are committed back to the branch.

## Quickstart

**1. Install.**

```bash
npm install --save-dev pr-checkmate
```

**2. Set it up.** `init` detects your languages and writes `pr-checkmate.json` plus
`.github/workflows/pr-checkmate.yml`. Safe to re-run — it updates the config rather than
replacing it, and never overwrites a workflow that already exists.

```bash
npx pr-checkmate init
```

**3. Run.**

```bash
npx pr-checkmate all
```

**Scope.** In CI it reviews the pull request diff; locally, every tracked file. `--full`
forces the whole repository anywhere. Only PR Size, Missing Tests and Lockfile Drift need a
diff and drop out of a full run.

### CI only — no install

Two files, taken straight from the public repository. Nothing goes into your
`package.json`.

- **[pr-checkmate-workflow.yml](https://github.com/Mybono/pr_checkmate_public/blob/main/pr-checkmate-workflow.yml)** → put in `.github/workflows/`.
  Runs the container, image tag already pinned.
- **[pr-checkmate.json](https://github.com/Mybono/pr_checkmate_public/blob/main/pr-checkmate.json)** → put in the repository root. Every check
  with its default value and a comment.

Commit both and it runs on the next pull request. Both are regenerated on every release.

[Running in CI →](https://github.com/Mybono/pr_checkmate_public/blob/main/documentation/ci-setup.md) — the generated workflow line by
line, which three lines are load-bearing, adopting it on a codebase that is not clean
yet, and the setup step each runner-dependency language needs.

## Language Support

Bundled languages need nothing on the runner. Runner-dependency languages need their
tool installed; the check skips gracefully when the tool is absent.

| Language | Checks | Toolchain |
|---|---|---|
| TypeScript / JavaScript | ESLint, Prettier, `tsc --noEmit` | Bundled |
| Python | Ruff (lint + format), mypy/pyright (types) | Bundled — Ruff, and pyright for types; the runner's own mypy is preferred when it has one |
| C++ | clang-format | Bundled |
| Swift | SwiftLint | `swiftlint` on the runner |
| Kotlin / Java | ktlint | `ktlint` on the runner |
| Go | `go vet`, gofmt | gofmt bundled; `go vet` needs the Go toolchain on the runner |
| Rust | `cargo clippy`, `cargo fmt` | Rust toolchain on the runner |
| C# | `dotnet format` | .NET SDK on the runner |
| Ruby | RuboCop | `rubocop` on the runner |
| PHP | PHP-CS-Fixer | `php-cs-fixer` on the runner |
| Shell | ShellCheck | `shellcheck` on the runner |

Secret scanning (gitleaks) and the universal git-diff checks are bundled and run for
every language.

## Checks

54 checks, run in parallel within each phase:

| Group | Count | What it covers |
|---|---|---|
| Universal git-diff | 18 | Secret scan, diff security, GitHub Actions and Dockerfile hardening, migration safety, sensitive files, custom rules, and the rest — language-agnostic and bundled |
| Per-language | 17 | Lint, format and type-check for the languages above |
| Dependencies | 7 | Unused/missing, circular, outdated, licence compliance, `npm audit`, and two vulnerability scanners |
| Quality | 7 | Duplicate code, dead code, spellcheck, markdown, YAML, broken symlinks, config validation |
| PR hygiene | 4 | Size, title, body, commit-message convention |

[Check reference →](https://github.com/Mybono/pr_checkmate_public/blob/main/documentation/checks/INDEX.md) — every check has its own
page: when it runs, every config key it reads, the defaults, how to turn it off.

## Configuration

One file, deep-merged over the defaults, so it holds only your deviations. The `severity`
map keys any check by the name it shows in the report:

```jsonc
{
  "severity": {
    "PR Size": "error",   // oversized PRs now fail the run
    "Spellcheck": "off"
  },
  "prSize": { "maxFiles": 40, "maxLines": 800 }
  // Skip a single line by appending `// pr-checkmate-ignore` to it.
}
```

`ignoreDirs`, `duplicate.ignore` and `deadCode.ignoreFiles` REPLACE their defaults rather
than extending them.

[Config guide →](https://github.com/Mybono/pr_checkmate_public/blob/main/documentation/config-guide.md) · [Annotated full config →](https://github.com/Mybono/pr_checkmate_public/blob/main/pr-checkmate.json)

## Programmatic API

Besides the CLI, PR CheckMate ships a typed API for embedding the same gates in your own
tooling. Import `runChecks`, `defineCheck`, and the outcome helpers, then run the
registry or your own checks. The full reference is at the
[API docs site](https://pr-checkmate-docs.pages.dev).

```ts
import { runChecks } from 'pr-checkmate';

const report = await runChecks({ cwd: process.cwd(), reporter: 'github' });
if (!report.ok) process.exit(1);
```

`runChecks` never throws on violations and never calls `process.exit`; the caller
decides. `report.ok` is `true` when nothing failed, and warnings and skips do not
affect it.

## License

LicenseRef Proprietary © 2025-2026 Artur Polishchuk. Free to install and run via the
published npm package (see [LICENSE](LICENSE) for the exact grant); the source is not
open-source and is not licensed for copying, modification, or redistribution.
