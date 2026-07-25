# Dockerfile Security

[Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · **Dockerfile Security** · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · [Workflow Security](workflow-security.md)

---

## Overview

Scans added lines of changed Dockerfiles for the common build-time security smells. Container builds
are a supply-chain surface: an unpinned base image or a `curl … | sh` step means the artifact you ship
today is not the one you reviewed.

Matched files:

```text
**/Dockerfile · **/Dockerfile.* · **/*.Dockerfile
```

### What it looks for

| Finding | Why it matters |
|---|---|
| `FROM …:latest` | The base image is mutable — pin a version or a digest |
| `USER root` | The container runs privileged; drop to a non-root user |
| `curl`/`wget` piped to `sh`/`bash` | Executes whatever the server returns today, unreviewed and unverified |
| `ADD https://…` | Fetches over the network with no checksum — prefer `COPY`, or verify a digest |
| `ENV`/`ARG` holding `PASSWORD`, `SECRET`, `TOKEN`, `API_KEY`, `ACCESS_KEY` | Values baked into `ENV` persist in the image layers and in `docker history` |
| `--insecure` / `--no-check-certificate` | Disables TLS verification during the build |
| `sudo` inside `RUN` | Unnecessary in a container build, and usually a sign that `USER` was switched too early |

All seven are tagged `warn`; none is tagged `error`.

| Property | Value |
|---|---|
| Display name | `Dockerfile Security` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate dockerfile-security` |
| Config key | `dockerfileSecurity` |
| Source | `src/core/checks/security/dockerfile-security.ts` |

## When it applies

Both conditions must hold:

1. `dockerfileSecurity.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

The check passes immediately when the PR touches no Dockerfile. `ignoreDirs` applies to the file list;
`sourcePath` does not restrict it, since Dockerfiles usually sit at the repository root.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `dockerfileSecurity.enabled` | boolean | `true` | Set `false` to skip the check |
| `dockerfileSecurity.ignore` | string[] | `[]` | Finding **labels** to mute, matched as case-insensitive substrings |

The pattern list and the Dockerfile globs themselves are fixed in the source and cannot be extended
through configuration.

### Examples

Accept `:latest` in a development image while keeping every other rule:

```json
{
  "dockerfileSecurity": { "ignore": ["latest"] }
}
```

Mute both root-user findings:

```json
{
  "dockerfileSecurity": { "ignore": ["USER root", "sudo inside RUN"] }
}
```

For a single deliberate line, the inline directive is narrower:

```dockerfile
RUN curl -sSfL https://example.com/install.sh | sh  # pr-checkmate-ignore — vendor has no release artifact
```

## Disabling

```json
{ "dockerfileSecurity": { "enabled": false } }
```

Or promote the findings to blocking:

```json
{ "severity": { "Dockerfile Security": "error" } }
```

## Notes

- **This check never fails the run.** Every pattern is `warn`, and the outcome is `warn` when anything
  matches. Use `severity` to make it blocking.
- The `pr-checkmate-ignore` directive is honoured by this check's own scanner, which reads the raw
  `git diff` rather than going through the shared `diffAddedLines` helper. The behaviour is the same.
- Dockerfile comments are **not** stripped before matching, so a commented-out `FROM …:latest` can
  still be reported.
- The log shows at most **3 examples per finding label**; matched lines are trimmed to 120 characters.
- `resolveTargetFiles` returning `null` (git unavailable) produces `skip('git unavailable')`; a failing
  diff produces `skip('diff unavailable')`. Neither fails the PR.
- This is a heuristic linter, not [hadolint](https://github.com/hadolint/hadolint). It covers the
  frequent security mistakes, not Dockerfile best practice in general.

---

[Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · **Dockerfile Security** · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · [Workflow Security](workflow-security.md)
