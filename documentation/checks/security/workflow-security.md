# Workflow Security

[Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · **Workflow Security**

---

## Overview

Scans added lines in `.github/workflows/*.yml` and `*.yaml` for the high-frequency GitHub Actions
footguns — script injection and supply-chain exposure.

Workflow files deserve their own check because a mistake there is not scoped to the application: it
runs with repository secrets and write tokens on CI infrastructure.

### What it looks for

| Finding | Severity | Why |
|---|---|---|
| Untrusted `github.event.*` input interpolated into an expression | error | A PR title, body, or branch name is attacker-controlled. Reaching a `run:` shell, `${{ }}` interpolation becomes **script injection** |
| `${{ github.head_ref }}` interpolation | warn | Same vector via the branch name — pass it through an `env:` var instead of interpolating into a script |
| Action not pinned to a full 40-character commit SHA | warn | A mutable tag can be moved to malicious code after review |
| `pull_request_target` trigger | warn | Runs with repository secrets in the context of an untrusted PR |

| Property | Value |
|---|---|
| Display name | `Workflow Security` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate workflow-security` |
| Config key | `workflowSecurity` |
| Source | `src/core/checks/security/workflow-security.ts` |

## When it applies

Both conditions must hold:

1. `workflowSecurity.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

Only files under `.github/workflows/` are scanned. GitLab CI, CircleCI and other platforms' config
files are not covered here — [Sensitive Files](sensitive-files.md) flags changes to them, without
inspecting their contents.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `workflowSecurity.enabled` | boolean | `true` | Set `false` to skip the check |
| `workflowSecurity.ignore` | string[] | `[]` | Finding **labels** to mute, matched as case-insensitive substrings |

### Examples

Stop warning about unpinned actions, because you rely on Dependabot to update tags:

```json
{
  "workflowSecurity": { "ignore": ["not pinned"] }
}
```

Accept `pull_request_target` because your workflow deliberately needs it and never checks out PR code:

```json
{
  "workflowSecurity": { "ignore": ["pull_request_target"] }
}
```

## Disabling

```json
{ "workflowSecurity": { "enabled": false } }
```

Given the blast radius of a workflow compromise, the opposite is often the better move — make findings
blocking:

```json
{ "severity": { "Workflow Security": "error" } }
```

## Fixing the injection findings

The safe pattern for untrusted input is an `env:` binding, so the value never reaches the shell as code:

```yaml
# flagged — the title is interpolated into the script
- run: echo "Reviewing ${{ github.event.pull_request.title }}"

# safe — the value arrives as an environment variable
- run: echo "Reviewing $TITLE"
  env:
    TITLE: ${{ github.event.pull_request.title }}
```

For actions, pin to a full commit SHA and keep the tag as a trailing comment:

```yaml
- uses: actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8 # v5.0.0
```

## Notes

- **This check never fails the run** on its own. The error/warn tags control the log icon and summary
  wording; the outcome is always `warn` when anything is found. Promote it with
  `severity: { "Workflow Security": "error" }`.
- The SHA-pinning rule matches `uses: owner/repo@ref` where `ref` is not exactly 40 hex characters.
  Local actions (`./.github/actions/…`) and Docker references do not match the pattern.
- Unlike [Diff Security](diff-security.md), lines are matched **as-is** — YAML comments are not
  stripped, so a commented-out workflow line can still be reported.
- The log shows at most **3 examples per finding label**; matched lines are trimmed to 120 characters.
- The inline `pr-checkmate-ignore` directive works here too, and is the narrowest way to accept one
  specific line.

---

[Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · **Workflow Security**
