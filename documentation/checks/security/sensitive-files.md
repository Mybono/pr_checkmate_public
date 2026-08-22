# Sensitive Files

[Docs](../../README.md) · [Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · **Sensitive Files** · [Workflow Security](workflow-security.md)

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
| Environment files (`.env`) | `.env`, `.env.*`, and the same in any directory — **except templates**, see below | ❌ error |
| Private keys and certificates | `.pem` `.key` `.p12` `.pfx` `.crt` `.cer` `.der` `.jks` `.keystore` | ❌ error |
| CI/CD pipeline config | `.github/workflows/`, `.gitlab-ci.yml`, `.circleci/`, `.travis.yml`, `Jenkinsfile`, `.buildkite/` | ⚠️ warn |
| Auth / authorization middleware | `middleware\|guards\|interceptors` paths containing `auth\|jwt\|session\|token\|oauth\|rbac`; also `auth.ts`, `jwt.js`, `session.mjs` | ⚠️ warn |
| Infrastructure and deployment | `docker-compose.yml`, `Dockerfile*`, `terraform/`, `*.tf`, `kubeconfig`, `helm/` | ⚠️ warn |
| Package manager lockfiles | `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml` | ⚠️ warn |

A file can land in more than one category and is then reported under each.

**Committed templates are not secrets.** A path ending in `.example`, `.sample`, `.template`, `.dist`
or `.defaults` is never reported as an environment file — `.env.example` is what the `.env` convention
tells you to commit, and a repository without one is the unusual case. Flagging it would mean the
check fires on the recommended practice, and a gate that fires on correct code is a gate people switch
off. You do not need `ignorePaths` for it.

The exclusion matches the **end** of the path only, so a real file named to sort next to the template
is still caught: `.env.example` is clean, `.env.example.local` is not.

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
| `sensitiveFileGuard.ignorePaths` | glob[] | `[]` | Files this check never looks at |
| `excludePaths` (top level) | glob[] | `[]` | Applies here too — the one global exclusion this check honours |
| `sensitiveFileGuard.failOn` | `error` \| `warn` \| `never` | `"error"` | What a finding rated `error` does to the run |
| `sensitiveFileGuard.severityOverrides` | map | `{}` | Re-level one category by its label |

**`sourcePath` and `ignoreDirs` are deliberately NOT applied.** A committed `.env` inside `dist/`
is still a committed secret, and a private key outside `sourcePath` is still a private key, so this
check looks everywhere git tracks. `excludePaths` is the exception: that is you saying "do not look
here" in as many words.

Reach for `ignorePaths` when a single file is a false positive — a `.pem` used as a test fixture, say.
(A committed `.env.example` needs nothing: templates are excluded by default.) Reach for
`severityOverrides` when a whole category is wrong for your
repository. The distinction matters: muting "Private keys and certificates" to excuse one fixture
also excuses the next real key somebody commits, while a path exclusion does not.

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
- If the diff cannot be read the check returns `skip('diff unavailable')`, and a file list git cannot
  produce gives `skip('git unavailable')`. Neither is a pass: a repository we failed to read is not a
  repository without secrets in it.
