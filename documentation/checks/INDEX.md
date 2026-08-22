# Check Reference

<!-- docs-nav -->
[Docs](../README.md) · [Config guide](../config-guide.md) · **Check reference** · [How checks work](../checks/concepts.md) · [Running in CI](../ci-setup.md) · [Migrating](../migrating.md) · [Architecture](../architecture-and-approach.md) · [Authoring a check](../authoring-checks.md)

---

## Overview

PR CheckMate ships **53 checks**. Every one of them is documented in its own file below: what it
does, when it runs, every key it reads from `pr-checkmate.json`, the defaults, and how to turn it
off.

All configuration lives in a single `pr-checkmate.json` at the repository root. It is deep-merged
with the built-in defaults, so a config only needs the keys it wants to change — an absent key keeps
its default, and an empty `pr-checkmate.json` (`{}`) behaves exactly like no file at all.

> **How checks work** — scope, git failures, configuring, silencing a single
> line, running locally, and getting a config file in the first place: all of it is
> in [Concepts](concepts.md). This page is the catalogue.

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
| [Python Types](languages/python-typecheck.md) | informational | `python-typecheck` | `python.typecheck`   | Bundled   |
| [C++ Format](languages/cpp-format.md)         | format        | `cpp-format`       | `cpp`                | Bundled   |
| [SwiftLint](languages/swift-lint.md)          | informational | —                  | `swift`              | Runner    |
| [ktlint](languages/kotlin-lint.md)            | informational | —                  | `kotlin`             | Runner    |
| [Go Vet](languages/go-lint.md)                | informational | `go-lint`          | `go`                 | Runner    |
| [Go Format](languages/go-format.md)           | format        | `go-format`        | `go`                 | Bundled   |
| [Clippy](languages/rust-lint.md)              | informational | `rust-lint`        | `rust`               | Runner    |
| [Rustfmt](languages/rust-format.md)           | format        | `rust-format`      | `rust`               | Runner    |
| [C# Format](languages/csharp-format.md)       | format        | `csharp-format`    | `csharp`             | Runner    |
| [RuboCop](languages/ruby-lint.md)             | informational | `ruby-lint`        | `ruby`               | Runner    |
| [PHP CS Fixer](languages/php-format.md)       | format        | `php-format`       | `php`                | Runner    |
| [ShellCheck](languages/shellcheck.md)         | informational | `shellcheck`       | `shellcheck`         | Runner    |

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
| [Broken Symlinks](quality/symlinks.md)            | informational | `symlinks`           | `symlinks`          |
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

**Checks Index** · [Concepts](concepts.md) · [Dependencies](#dependencies) · [Git & Diff](#git--diff) · [Languages](#languages) · [Pull Request](#pull-request) · [Quality](#quality) · [Security](#security) · [Config Guide](../config-guide.md)
