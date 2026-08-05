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

> PR CheckMate is a security-first PR gate: secret scanning, dependency
> vulnerability checks, and a signed SBOM + independent VirusTotal scan on
> every release — plus 53 code-quality checks across 12 languages bundled in
> for free, running in CI on every pull request. The bundled languages need no
> per-language toolchain setup.

Every release is published with a CycloneDX SBOM and an independent
[VirusTotal scan report](https://github.com/Mybono/pr_checkmate_public/releases/latest)
of the exact npm tarball, attached to that version's
[GitHub release notes](https://github.com/Mybono/pr_checkmate_public/releases/latest).

## Why PR CheckMate

Most teams wire up a different tool for each concern, in every repo, per language:
ESLint or Ruff or SwiftLint for lint, Prettier or `gofmt` or `rustfmt` for format,
gitleaks for secrets, `npm audit` or grype for dependency risk — each with its own
install step, its own config file, its own version to keep pinned. PR CheckMate
replaces that whole stack with one npm package and one `pr-checkmate.json`: lint,
format, type-check, secret scanning, dependency risk, and SBOM/VirusTotal-verified
releases, for all 11 supported languages, configured in one place instead of stitched
together tool by tool, language by language. You add one step to a GitHub Actions
workflow and everything runs on every PR.

- Secret scanning on every diff. gitleaks runs on every PR, bundled, no setup.
- Verifiable releases. Every version ships a CycloneDX SBOM and an independent
  VirusTotal scan of the exact published tarball, attached to the GitHub release.
- Dependency risk visibility. License compliance, `npm audit`, and osv-scanner/grype
  vulnerability scans.

Also included, bundled the same way:

- One package, any stack. Language is auto-detected from the repo.
- Bundled tooling. ESLint, Prettier, Ruff, clang-format, and gitleaks run inside the
  package (WASM/JS), so clients install nothing for those languages.
- Delta mode. A PR run checks only the changed files; outside CI the default is every tracked
  file. `--full` forces that anywhere, and a run that cannot resolve its base falls back to it.
- Parallel phases. Checks run concurrently within each phase: blocking, then
  informational, then format.
- Reports in the PR. Results post as a single comment that updates in place, and
  formatter fixes are committed back to the branch.
- Tunable. Any check can be re-scoped, downgraded, or disabled in `pr-checkmate.json`.

## Security & Supply Chain

- Secret scanning. gitleaks runs on every diff, bundled, no setup.
- Verifiable releases. Every version ships a CycloneDX SBOM and an independent
  [VirusTotal scan](https://github.com/Mybono/pr_checkmate_public/releases/latest) of
  the exact published npm tarball, attached to that version's GitHub release notes.
- Dependency risk. License compliance, `npm audit`, a cross-language vulnerability scan
  (osv-scanner, opt-in), and a direct-dependency vulnerability scan (grype, on by
  default).
- CI/CD hardening. GitHub Actions workflow and Dockerfile security checks run as part
  of the universal git-diff checks below.

## Language Support

Bundled languages need nothing on the runner. Runner-dependency languages need their
tool installed; the check skips gracefully when the tool is absent.

| Language | Checks | Toolchain |
|---|---|---|
| TypeScript / JavaScript | ESLint, Prettier, `tsc --noEmit` | Bundled |
| Python | Ruff (lint + format), mypy/pyright (types) | Ruff bundled; types need mypy or pyright on the runner |
| C++ | clang-format | Bundled |
| Swift | SwiftLint | `swiftlint` on the runner |
| Kotlin / Java | ktlint | `ktlint` on the runner |
| Go | `go vet`, `gofmt` | Go toolchain on the runner |
| Rust | `cargo clippy`, `cargo fmt` | Rust toolchain on the runner |
| C# | `dotnet format` | .NET SDK on the runner |
| Ruby | RuboCop | `rubocop` on the runner |
| PHP | PHP-CS-Fixer | `php-cs-fixer` on the runner |
| Shell | ShellCheck | `shellcheck` on the runner |

Secret scanning (gitleaks) and the universal git-diff checks are bundled and run for
every language.

## Quickstart

Install:

```bash
npm install --save-dev pr-checkmate
```

Scaffold `pr-checkmate.json` and `.github/workflows/pr-checkmate.yml` for the detected
languages:

```bash
npx pr-checkmate init
```

Run all checks:

```bash
npx pr-checkmate all
```

Scope depends on where the command runs. In CI it reviews the pull request diff, taking the
base SHA from the GitHub `pull_request` event or Bitbucket's destination branch; a CI build
with no PR attached falls back to `HEAD^`. Locally, with no CI environment detected, it
reviews every tracked file. It used to fall back to `HEAD^` there too, so a local run saw
only the last commit and would report a clean project with a `debugger` committed the day
before. `--full` forces the whole-repo review anywhere, including inside a CI build, and it
composes with any command:

```bash
npx pr-checkmate all --full
npx pr-checkmate lint --full
```

The 16 checks that only make sense on a diff (PR Size, Diff Security, Merge Conflict, Custom
Rules, Banned Imports, the git-diff family) drop out of the report on a full scan. The
[per-check reference](https://github.com/Mybono/pr_checkmate_public/blob/main/documentation/checks/INDEX.md)
spells out the rules under "Review scope".

### Running in CI without installing

Plenty of repos never add pr-checkmate to `package.json` and call it on the fly in CI
instead (`npx pr-checkmate`). That works, but there is no `node_modules` to generate a
config from, so the run silently falls back to the defaults with no file to tune.
`npx pr-checkmate init` is no help either, since the package is not kept around.

Pull the published template instead:

```bash
curl -fsSL -O https://raw.githubusercontent.com/Mybono/pr_checkmate_public/main/pr-checkmate.json
```

Commit the file to the repository root and edit it there. Every check appears in it with
its real default value and a short comment, so switching a check on or off, or changing
one of its thresholds, is a one-line edit. The JSON schema is published alongside it for
editor autocompletion, and the template already references it:

```text
https://raw.githubusercontent.com/Mybono/pr_checkmate_public/main/pr-checkmate.schema.json
```

Both files are refreshed on every release, so the template never drifts from the
installed version.

## GitHub Actions

Every workflow follows the same shape: checkout, set up Node, install, run. The
TypeScript setup is the baseline:

```yaml
name: PR CheckMate
on:
  pull_request:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@v4
        with: { node-version: 24 }
      - run: npm ci
      - run: npx pr-checkmate all
```

For a runner-dependency language, add its toolchain setup step before the run: Go with
`actions/setup-go`, Rust with `dtolnay/rust-toolchain`, C# with `actions/setup-dotnet`,
Ruby with `ruby/setup-ruby` (plus `gem install rubocop`), PHP with
`shivammathur/setup-php` (tools: `php-cs-fixer`), Kotlin by fetching the `ktlint`
binary. Swift's `swiftlint` is pre-installed on `macos-latest`, and `shellcheck` is
pre-installed on `ubuntu-latest` — neither needs a setup step. Anything missing is
skipped, not failed.

## Checks

53 checks, grouped by category and run in parallel within each phase:

- Universal git-diff (18) — language-agnostic and bundled: merge-conflict markers,
  secret scan, diff security (JS/TS/Python/Go/Kotlin), diff smells, GitHub Actions and
  Dockerfile security, case-collision, custom rules, banned imports, license headers,
  leftover debug lines, lockfile drift, large/binary files, TODO/FIXME, missing tests,
  coverage presence, DB migration safety, and sensitive-file guard.
- Dependencies (7) — unused/missing, circular, outdated, license compliance, npm audit,
  a cross-language vuln scan (osv-scanner, opt-in), and a direct-dependency vuln scan
  (grype, on by default).
- Quality (7) — duplicate code, dead code, multi-language spellcheck, markdown lint,
  YAML lint, broken symlinks, config validation.
- PR hygiene (4) — size, title, body, and commit-message convention.
- Per-language (17) — lint, format, and type-check for the languages listed above.

Each check has its own page in the
[per-check reference](https://github.com/Mybono/pr_checkmate_public/blob/main/documentation/checks/INDEX.md),
covering when it runs, the config keys it reads, and how to turn it off.

## Configuration

Everything lives in `pr-checkmate.json`. Run `npx pr-checkmate init` to scaffold the
full file for your detected languages.

- Severity map. The one universal control. Key any check by its display name (as it
  appears in the report) and set `"error"` to fail the run, `"warn"` for advisory, or
  `"off"` to disable it. This works for every check, old or new.
- Per-check options. For example `prSize.maxFiles`/`maxLines`, `largeFiles.maxSizeKb`
  and `allow`, `prBody.minLength`, `duplicate.minLines`, `missingTests.ignore`, or
  `<check>.ignore` to mute a single finding. Any check also takes `enabled: false`.
- `customRules`. Your own regex rules run against the diff; a rule with
  `severity: "error"` blocks the run. Fields: `name`, `pattern`, `message?`,
  `severity?`, `include?`, `exclude?`.
- `bannedImports` forbids specific dependencies or imports. `licenseHeader` requires a
  header on newly added source files (opt-in).
- Inline suppression. Append `// pr-checkmate-ignore` to any line to skip it.

```jsonc
{
  // Promote, downgrade, or disable ANY check by its display name.
  "severity": {
    "PR Size": "error",   // oversized PRs now fail the run
    "Spellcheck": "off"   // turn a check off entirely
  },
  "prSize": { "maxFiles": 40, "maxLines": 800 },
  "customRules": [
    { "name": "no-fixme", "pattern": "FIXME", "severity": "error" }
  ]
  // Skip a single line by appending `// pr-checkmate-ignore` to it.
}
```

For the complete list of keys each check reads, along with their defaults, see the
[per-check reference](https://github.com/Mybono/pr_checkmate_public/blob/main/documentation/checks/INDEX.md).

## PR Summary Comment

After `npx pr-checkmate all`, a summary table is posted to the PR and updated in place
on later pushes, so there are no duplicate comments. Format fixes (Prettier, Ruff,
clang-format, and the rest) are committed straight back to the branch. Posting needs
the `pull-requests: write` permission, which the generated workflow already includes.

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