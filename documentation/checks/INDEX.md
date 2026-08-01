# Check Reference

**Checks Index** · [Dependencies](#dependencies) · [Git & Diff](#git--diff) · [Languages](#languages) · [Pull Request](#pull-request) · [Quality](#quality) · [Security](#security) · [Config Guide](../README.md) · [Authoring Checks](../authoring-checks.md)

---

## Overview

PR CheckMate ships **51 checks**. Every one of them is documented in its own file below: what it
does, when it runs, every key it reads from `pr-checkmate.json`, the defaults, and how to turn it
off.

All configuration lives in a single `pr-checkmate.json` at the repository root. It is deep-merged
with the built-in defaults, so a config only needs the keys it wants to change — an absent key keeps
its default, and an empty `pr-checkmate.json` (`{}`) behaves exactly like no file at all.

## Getting a config file

How you obtain the file depends on how you run PR CheckMate. For deciding what to put in it — which
keys are worth setting, and how to tell a real finding from noise — see
[Writing your `pr-checkmate.json`](../README.md).

### If you install the package

```bash
npm install --save-dev pr-checkmate
npx pr-checkmate init
```

`init` writes a `pr-checkmate.json` tailored to the languages it detects in your repository, and is
safe to re-run — it updates the existing file rather than replacing it.

### If you only run it in CI

Many projects never add PR CheckMate to their `package.json` and instead invoke it on the fly:

```yaml
- run: npx pr-checkmate
```

That works, but there is no `node_modules` to generate a config from, so these projects get the
defaults and have no way to tune them. Download the template from the public repository instead:

```bash
curl -fsSL -o pr-checkmate.json \
  https://raw.githubusercontent.com/Mybono/pr_checkmate_public/main/pr-checkmate.json
```

Commit the file to your repository root and edit it there. Every check is listed in it with its real
default value and a short comment, so enabling, disabling, or re-tuning a check is a one-line change.

If you want editor autocompletion and inline validation, the schema is published alongside it and the
template already points at it:

```json
{
  "$schema": "https://raw.githubusercontent.com/Mybono/pr_checkmate_public/main/pr-checkmate.schema.json"
}
```

Both files are refreshed on every release, so the template never drifts from the version of
PR CheckMate you are running.

### Turning a check off

Two mechanisms, and the second works for every check:

```json
{
  "outdatedDeps": { "enabled": false },
  "severity": { "Duplicate Code": "off" }
}
```

`enabled: false` is per-check and only exists where a check defines it. `severity: { "<name>": "off" }`
removes any check from the run by its display name — see [the full list](#dependencies) below, or the
`//check-names` array inside the template.

## How checks are grouped

Checks run in three **phases**, strictly in order. Within a phase every check runs in parallel.

| Phase           | Runs   | An unexpected exception is graded as | Why this order                                                             |
| --------------- | ------ | ------------------------------------ | -------------------------------------------------------------------------- |
| `blocking`      | first  | `fail`                               | Cheap, high-confidence gates fail fast                                     |
| `informational` | second | `warn`                               | A crash in advisory tooling should not stop a merge                        |
| `format`        | last   | `warn`                               | Formatters may **write** files, so they run after everything has read them |

**A phase does not decide whether a check can fail the run.** The run's verdict is
`ok: summary.failed === 0` — _any_ check returning `fail`, in _any_ phase, fails it. Several
`informational` checks do exactly that when a rule is configured at `severity: "error"`
(for example [Banned Imports](dependencies/banned-imports.md)). What the phase actually controls is
execution order, and how an _unexpected exception_ is graded.

A check reports one of four outcomes: `pass`, `warn`, `fail`, or `skip` — the last is used when a
required tool is absent, so a missing toolchain never fails a PR. A check whose `applies()` returns
`false` is omitted from the report entirely rather than listed as skipped, keeping single-language
repos free of irrelevant rows.

## Review scope

What a run looks at is decided once, before any check executes, in this order:

| Scope  | When                         | What is reviewed                                                                               |
| ------ | ---------------------------- | ---------------------------------------------------------------------------------------------- |
| Full   | `--full` on any command      | Every tracked file. Overrides everything below, including CI variables.                        |
| Staged | `npx pr-checkmate precommit` | The staged changes (index vs `HEAD`).                                                          |
| Delta  | CI, pull request             | The PR diff — base SHA from the GitHub `pull_request` event or Bitbucket's destination branch. |
| Delta  | CI, no pull request          | `HEAD^`..`HEAD` — a push or branch build.                                                      |
| Full   | **no CI detected**           | Every tracked file.                                                                            |

A human at a terminal gets a whole-project verdict; a build gets the diff, which is the point of a
PR gate. Before this was explicit, a local run fell back to `HEAD^` and reviewed only the last
commit — `pr-checkmate all` could report a clean project with a `debugger` committed earlier.

The chosen scope is printed on the `[buildContext]` line (`scope=full (no CI detected)`,
`scope=delta (origin/main, PR build)`), because a wrong scope is otherwise invisible: a run that
reviewed one commit and one that reviewed the repository both just print passes. Any base that
cannot be resolved — a shallow clone without the base branch, a first commit with no `HEAD^` —
degrades to a full scan rather than diffing against a missing ref and reviewing nothing.

Checks that only make sense on a diff (`PR Size`, `Diff Security`, `Merge Conflict`, `Custom
Rules`, `Banned Imports`, the git-diff family — 16 in total) gate their own `applies()` on the
presence of a range, so a full-scan run omits them from the report entirely.

## Configuring any check

Three mechanisms apply to every check, on top of each check's own keys:

```jsonc
{
  "$schema": "./node_modules/pr-checkmate/pr-checkmate.schema.json",

  // 1. Scope — which directories checks look at
  "sourcePath": "src",
  "ignoreDirs": ["node_modules", "dist", "vendor"],

  // 2. Severity override — keyed by the check's DISPLAY NAME, not its file name
  "severity": {
    "Duplicate Code": "warn",
    "Outdated Deps": "off",
  },

  // 3. Per-check keys — see each check's page
  "prSize": { "maxFiles": 40, "maxLines": 800 },
}
```

- **`severity`** re-scopes any check without touching its own config. The key is the check's display
  name exactly as it appears in the tables below:
  - `"error"` — promotes a `warn` outcome to `fail`
  - `"warn"` — demotes a `fail` outcome to `warn`
  - `"off"` — removes the check from the run before it executes; a universal off-switch that works
    even for checks with no `enabled` flag of their own

  It translates only between `warn` and `fail` — a `pass` or `skip` is never altered.

- **`ignoreDirs`** _replaces_ the default ignore list rather than extending it — the generated
  config contains the full default list so it can be edited directly.
- Unknown top-level keys are reported by [Config Validation](quality/config-validation.md), which
  catches typos instead of silently ignoring them.

## Suppressing a single line

Every check that scans added diff lines honours an inline directive. Put `pr-checkmate-ignore`
anywhere on the line — usually in a trailing comment — and that line is removed from the diff before
any content check sees it:

```ts
const hash = createHash('md5'); // pr-checkmate-ignore — non-cryptographic cache key
```

```python
subprocess.run(cmd, shell=True)  # pr-checkmate-ignore
```

The directive is applied centrally in `diffAddedLines`, so it works for
[Diff Security](security/diff-security.md), [Workflow Security](security/workflow-security.md),
[TODO/FIXME](git/todo-fixme.md), [Leftover Debug](git/leftover-debug.md),
[Banned Imports](dependencies/banned-imports.md), [Custom Rules](git/custom-rules.md) and every other
diff-scanning check — without each of them implementing it.

Two related rules worth knowing:

- **Commented-out code is not flagged.** Code-targeting checks match against the line with its
  trailing `//` or `#` comment stripped, so `// eval(x)` is not reported as an `eval()` call. Quote
  state is tracked, so a `#` inside `"#fff"` or a `//` inside a URL is preserved.
- **`<check>.ignore` filters finding labels, not file paths — except for [YAML Lint](quality/yaml-lint.md).**
  For `diffSecurity`, `workflowSecurity`, and `dockerfileSecurity`, each `ignore` entry is matched as a
  **case-insensitive substring of the finding's label**. So `"ignore": ["md5"]` mutes the
  `MD5 (weak hash …)` finding everywhere, and does _not_ mean "skip files named md5". `yamlLint.ignore`
  is the one exception: it's matched as a glob against the **file path**, filtering which YAML files get
  linted at all — see [YAML Lint](quality/yaml-lint.md) for details.

## Running checks locally

```bash
npx pr-checkmate                    # every check, posts to the PR when in CI
npx pr-checkmate all                # same as above
npx pr-checkmate all --full         # every check over every tracked file
npx pr-checkmate precommit          # staged changes only, never writes files
npx pr-checkmate <command>          # a single check — see the tables below
npx pr-checkmate lint spellcheck    # several checks in one pass
npx pr-checkmate lint --full        # a single check, whole repository
```

`all`, `init` and `precommit` take no other arguments — passing any is an error, so a
run can never look successful for work it skipped.

`--full` is the only flag, and it composes with every command because scope is independent of
which checks run — see [Review scope](#review-scope). Outside CI it is already the default, so
it is only needed to force a whole-repo review inside a build. A misspelled flag aborts the run
rather than falling back to the diff, and `precommit --full` is refused as contradictory.

Five checks have no dedicated CLI command and run only as part of a full run: `Grype Scan`,
`Merge Conflict`, `ktlint`, `SwiftLint`, and `Dead Code`.

---

## Dependencies

Manifest, lockfile, licence, and vulnerability checks.

| Check                                            | Phase         | CLI command      | Config key           |
| ------------------------------------------------ | ------------- | ---------------- | -------------------- |
| [Banned Imports](dependencies/banned-imports.md) | informational | `banned-imports` | `bannedImports`      |
| [Circular Deps](dependencies/circular-deps.md)   | **blocking**  | `circular`       | `circularDeps`       |
| [Dependencies](dependencies/dependencies.md)     | **blocking**  | `deps`           | `dependency`         |
| [Grype Scan](dependencies/grype-scan.md)         | informational | —                | `grypeScan`          |
| [License Check](dependencies/license-check.md)   | **blocking**  | `license`        | `licenseCheck`       |
| [NPM Audit](dependencies/npm-audit.md)           | **blocking**  | `npm-audit`      | `security.npm-audit` |
| [Outdated Deps](dependencies/outdated-deps.md)   | informational | `outdated`       | `outdatedDeps`       |
| [Vuln Scan](dependencies/vuln-scan.md)           | informational | `vuln-scan`      | `vulnScan`           |

## Git & Diff

Checks that read the diff itself rather than the code's meaning.

| Check                                   | Phase         | CLI command      | Config key      |
| --------------------------------------- | ------------- | ---------------- | --------------- |
| [Custom Rules](git/custom-rules.md)     | informational | `custom-rules`   | `customRules`   |
| [Diff Smells](git/diff-smell.md)        | informational | `diff-smell`     | `diffSmell`     |
| [Large Files](git/large-files.md)       | informational | `large-files`    | `largeFiles`    |
| [Leftover Debug](git/leftover-debug.md) | informational | `leftover-debug` | `leftoverDebug` |
| [Lockfile Drift](git/lockfile-drift.md) | informational | `lockfile-drift` | `lockfileDrift` |
| [Merge Conflict](git/merge-conflict.md) | **blocking**  | —                | `mergeConflict` |
| [Missing Tests](git/missing-tests.md)   | informational | `missing-tests`  | `missingTests`  |
| [TODO/FIXME](git/todo-fixme.md)         | informational | `todo-fixme`     | `todoFixme`     |

## Languages

Linters, formatters, and type checkers. Language is auto-detected; a check skips when its language
is absent. Bundled tools need nothing on the runner — runner-dependency tools skip gracefully when
the binary is missing.

| Check                                         | Phase         | CLI command        | Config key           | Toolchain |
| --------------------------------------------- | ------------- | ------------------ | -------------------- | --------- |
| [ESLint](languages/eslint.md)                 | **blocking**  | `lint`             | `lint`               | Bundled   |
| [TypeScript](languages/typecheck.md)          | **blocking**  | `typecheck`        | `typecheck`          | Bundled   |
| [Prettier](languages/prettier.md)             | format        | `prettier`         | `prettier`           | Bundled   |
| [Ruff Lint](languages/python-lint.md)         | informational | `python-lint`      | `python.ruff.lint`   | Bundled   |
| [Ruff Format](languages/python-format.md)     | format        | `python-format`    | `python.ruff.format` | Bundled   |
| [Python Types](languages/python-typecheck.md) | informational | `python-typecheck` | `python.typecheck`   | Runner    |
| [C++ Format](languages/cpp-format.md)         | format        | `cpp-format`       | `cpp`                | Bundled   |
| [SwiftLint](languages/swift-lint.md)          | informational | —                  | `swift`              | Runner    |
| [ktlint](languages/kotlin-lint.md)            | informational | —                  | `kotlin`             | Runner    |
| [Go Vet](languages/go-lint.md)                | informational | `go-lint`          | `go`                 | Runner    |
| [Go Format](languages/go-format.md)           | format        | `go-format`        | `go`                 | Runner    |
| [Clippy](languages/rust-lint.md)              | informational | `rust-lint`        | `rust`               | Runner    |
| [Rustfmt](languages/rust-format.md)           | format        | `rust-format`      | `rust`               | Runner    |
| [C# Format](languages/csharp-format.md)       | format        | `csharp-format`    | `csharp`             | Runner    |
| [RuboCop](languages/ruby-lint.md)             | informational | `ruby-lint`        | `ruby`               | Runner    |
| [PHP CS Fixer](languages/php-format.md)       | format        | `php-format`       | `php`                | Runner    |

## Pull Request

Checks on the PR's metadata rather than its code.

| Check                          | Phase         | CLI command  | Config key   |
| ------------------------------ | ------------- | ------------ | ------------ |
| [Commitlint](pr/commitlint.md) | informational | `commitlint` | `commitlint` |
| [PR Body](pr/pr-body.md)       | informational | `pr-body`    | `prBody`     |
| [PR Size](pr/pr-size.md)       | informational | `pr-size`    | `prSize`     |
| [PR Title](pr/pr-title.md)     | informational | `pr-title`   | —            |

## Quality

Cross-language quality gates.

| Check                                             | Phase         | CLI command         | Config key         |
| ------------------------------------------------- | ------------- | ------------------- | ------------------ |
| [Case Collision](quality/case-collision.md)       | informational | `case-collision`    | `caseCollision`    |
| [Config Validation](quality/config-validation.md) | informational | `config-validation` | `configValidation` |
| [Coverage](quality/coverage.md)                   | informational | `coverage`          | `coverage`         |
| [Dead Code](quality/dead-code.md)                 | informational | —                   | `deadCode`         |
| [Duplicate Code](quality/duplicate-code.md)       | informational | `duplicate`         | `duplicate`        |
| [License Header](quality/license-header.md)       | informational | `license-header`    | `licenseHeader`    |
| [Markdown](quality/markdown-lint.md)              | informational | `markdown`          | `markdownlint`     |
| [Spellcheck](quality/spellcheck.md)               | informational | `spellcheck`        | `spellcheck`       |
| [YAML Lint](quality/yaml-lint.md)                 | **blocking**  | `yaml-lint`         | `yamlLint`         |

## Security

Secret scanning and diff-level security review.

| Check                                                  | Phase         | CLI command           | Config key           |
| ------------------------------------------------------ | ------------- | --------------------- | -------------------- |
| [Security Scan](security/security-scan.md)             | **blocking**  | `security`            | `security.gitleaks`  |
| [Diff Security](security/diff-security.md)             | informational | `diff-security`       | `diffSecurity`       |
| [Dockerfile Security](security/dockerfile-security.md) | informational | `dockerfile-security` | `dockerfileSecurity` |
| [Migration Safety](security/migration-safety.md)       | informational | `migration-safety`    | `migrationSafety`    |
| [Sensitive Files](security/sensitive-files.md)         | informational | `sensitive-files`     | `sensitiveFileGuard` |
| [Workflow Security](security/workflow-security.md)     | informational | `workflow-security`   | `workflowSecurity`   |

---

**Checks Index** · [Dependencies](#dependencies) · [Git & Diff](#git--diff) · [Languages](#languages) · [Pull Request](#pull-request) · [Quality](#quality) · [Security](#security) · [Config Guide](../README.md) · [Authoring Checks](../authoring-checks.md)
