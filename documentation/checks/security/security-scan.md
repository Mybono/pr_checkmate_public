# Security Scan

[Docs](../../README.md) · [Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · **Security Scan** · [Sensitive Files](sensitive-files.md) · [Workflow Security](workflow-security.md)

---

## Overview

Secret scanning via a bundled [gitleaks](https://github.com/gitleaks/gitleaks) wrapper. This is the
one check in the security category that **fails** the run: a committed credential is not an advisory
matter, and unlike a style issue it cannot be fixed after the fact — once pushed, the secret must be
rotated.

Language-agnostic, so it runs on every repository regardless of stack. Findings are printed to the
run log as the scanner produces them, and the outcome names how many there were:

```text
❌ Security Scan: 3 potential secret(s)
```

No report file is written into your repository. (Earlier versions said results went to
`gitleaks-report.html`; the check never wrote one — that flag belongs to a separate npm script — so
the line has been corrected rather than the file added.)

The outcome depends on gitleaks' exit code, and the distinction matters:

| Exit code | Meaning | Outcome |
|---|---|---|
| `0` | No secrets found | `pass` |
| `1` | Secrets found | **`fail`** |
| anything else | Scanner or infrastructure error | `warn` |

A crashing scanner produces a warning rather than a failure, so tooling trouble never blocks an
unrelated PR — while a real finding always does. A scanner that cannot even be located (the bundled
binary unresolvable under an unusual install layout) is a `skip` for the same reason: the check
resolves it inside a `try`, so a resolution error can no longer surface as a failed blocking check.

Exit code `1` alone is not enough to call something a finding, because the **wrapper** exits `1` when
it fails to start the scanner at all — and that used to be reported as a leaked secret. The two are
now told apart by what was written to stderr: a wrapper failure names itself there, a real finding
does not. So a container where gitleaks cannot run reports `warn` naming the cause, instead of a
blocking `fail` naming a secret that does not exist.

The most common cause of that was a read-only `HOME`. Running the image as
`--user $(id -u):$(id -g)` — which is the documented way to keep file ownership sane — leaves `HOME`
pointing at a directory the container user cannot write, and the scanner needs somewhere to put its
own state. The check now redirects `HOME` to a writable directory for the duration of the scan when
the real one cannot be written to, so the recommended invocation and the scanner no longer contradict
each other.

| Property | Value |
|---|---|
| Display name | `Security Scan` |
| Phase | `blocking` |
| CLI command | `npx pr-checkmate security` |
| Config key | `security.gitleaks` |
| Source | `src/core/checks/security/security.ts` |

## When it applies

`security.gitleaks.enabled` is not `false`. It needs no diff range, and the scan scope follows the
run's: with a range the scanner runs in its `ci` mode over the changed commits, and without one
(a local run, or `--full`) it walks the repository's commit history instead. The distinction is
printed as `[Security Scan]: scanning the diff range` / `scanning full history`, so the log always
says which one happened. A history scan is fast — roughly a second over 700 commits — but
`security.gitleaks.depth` caps it for a repository with a long past.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `security.gitleaks.enabled` | boolean | `true` | Set `false` to skip the check |
| `security.gitleaks.version` | string | `"8.30.0"` | Which gitleaks binary version to use |
| `security.gitleaks.depth` | integer | unset (whole history) | Caps the history scan used when there is no diff range to the last N commits. Ignored in a PR run |

**Your own gitleaks config wins.** If the repository contains any of these, it is used and the bundled
ruleset is not passed at all:

```text
.gitleaks.toml · gitleaks.toml · .gitleaks/config.toml
```

This is the mechanism for allow-listing a false positive — add a gitleaks
[allowlist rule](https://github.com/gitleaks/gitleaks#configuration) to your own `gitleaks.toml`.
There is no `ignore` key in `pr-checkmate.json` for this check.

### Examples

Pin a different gitleaks version:

```json
{
  "security": {
    "gitleaks": { "version": "8.28.0" }
  }
}
```

Allow-list a test fixture, in your own `gitleaks.toml`:

```toml
[allowlist]
  description = "test fixtures contain deliberately fake keys"
  paths = ['''tests/fixtures/.*''']
```

## Disabling

```json
{ "security": { "gitleaks": { "enabled": false } } }
```

Or demote a real finding to a warning — rarely a good idea for secrets:

```json
{ "severity": { "Security Scan": "warn" } }
```

## Notes

- **Why the version is pinned.** Left unpinned, the scanner queries the GitHub API for the "latest"
  gitleaks release on every run. That adds latency and times out entirely in locked-down or offline CI.
  The default `8.30.0` is the same version the scanner would otherwise fall back to.
- gitleaks runs with `stdio: 'inherit'`, so its full output appears in the CI log rather than being
  captured and condensed.
- `BASE_SHA` and `HEAD_SHA` are passed through the environment when a diff range is available, which is
  how `--diff-mode ci` scopes the scan to the pull request.
- The bundled binary is executed with the current Node binary rather than through `npx`.
- A finding means the secret is in the git history, not merely in the working tree. Removing the line
  in a follow-up commit does not remove the secret — rotate it.
