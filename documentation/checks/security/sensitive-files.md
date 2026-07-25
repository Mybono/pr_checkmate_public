# Sensitive Files

[Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · **Sensitive Files** · [Workflow Security](workflow-security.md)

---

## Overview

Flags **which files the PR touches**, not what is in them. Some paths deserve a second reviewer no
matter how small the diff — credentials, CI configuration, auth middleware, infrastructure.

This is a routing signal rather than a defect detector: nothing here is necessarily wrong, but a
one-line change to `.github/workflows/release.yml` or `middleware/auth.ts` should not slip through on
the strength of being one line.

### Categories

| Category | Matches | Logged as |
|---|---|---|
| Environment files (`.env`) | `.env`, `.env.*`, and the same in any directory | ❌ error |
| Private keys and certificates | `.pem` `.key` `.p12` `.pfx` `.crt` `.cer` `.der` `.jks` `.keystore` | ❌ error |
| CI/CD pipeline config | `.github/workflows/`, `.gitlab-ci.yml`, `.circleci/`, `.travis.yml`, `Jenkinsfile`, `.buildkite/` | ⚠️ warn |
| Auth / authorization middleware | `middleware\|guards\|interceptors` paths containing `auth\|jwt\|session\|token\|oauth\|rbac`; also `auth.ts`, `jwt.js`, `session.mjs` | ⚠️ warn |
| Infrastructure and deployment | `docker-compose.yml`, `Dockerfile*`, `terraform/`, `*.tf`, `kubeconfig`, `helm/` | ⚠️ warn |
| Package manager lockfiles | `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml` | ⚠️ warn |

A file can land in more than one category and is then reported under each.

| Property | Value |
|---|---|
| Display name | `Sensitive Files` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate sensitive-files` |
| Config key | `sensitiveFileGuard` |
| Source | `src/core/checks/security/sensitive-file-guard.ts` |

## When it applies

Both conditions must hold:

1. `sensitiveFileGuard.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

Files are collected with `--diff-filter=ACMRD`, so **deletions count too** — removing a private key or
a workflow is as noteworthy as adding one.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `sensitiveFileGuard.enabled` | boolean | `true` | Set `false` to skip the check |

That is the whole surface. The category list and its patterns are fixed in the source, and unlike
[Diff Security](diff-security.md) there is **no `ignore` key** — a category cannot be muted
individually.

### Example

```json
{
  "sensitiveFileGuard": { "enabled": true }
}
```

If a category is consistently noisy for your repository — lockfile changes on every dependency bump,
for instance — the available options are to switch the check off entirely or to accept the warning,
since it never blocks a merge.

## Disabling

```json
{ "sensitiveFileGuard": { "enabled": false } }
```

Or make touching a sensitive path a hard gate that requires a deliberate override:

```json
{ "severity": { "Sensitive Files": "error" } }
```

## Notes

- **This check never fails the run on its own.** `.env` files and private keys are logged with ❌ and
  counted as `N sensitive file(s) changed`, while the rest report as `N notable file(s) changed` — but
  the outcome is `warn` either way. Promote it with `severity` if you want it blocking.
- It does not read file contents. A committed `.env` full of real credentials is caught by
  [Security Scan](security-scan.md), which greps for secrets; this check only observes that the file
  was touched.
- Occurrence counts are suppressed in the log (`showCount: false`) because listing the file paths is
  the useful output, not how many matched a category.
- `.gitignore` is not consulted. A file that is tracked *and* ignored still appears in the diff and is
  still reported.
- If the diff cannot be read the check returns `skip('diff unavailable')`.

---

[Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · **Sensitive Files** · [Workflow Security](workflow-security.md)
