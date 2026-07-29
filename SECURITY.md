# Security Policy

## Supported Versions

Only the latest version of `pr-checkmate` published to npm is supported with
security fixes. Please upgrade to the latest version before reporting an
issue.

## Reporting a Vulnerability

If you believe you've found a security vulnerability in `pr-checkmate`
(including in the CLI, the bundled checks, or the GitHub Actions workflow
template it ships), please report it privately — do not open a public GitHub
issue.

- **Preferred:** use [GitHub's private vulnerability reporting](https://github.com/Mybono/pr_checkmate_public/security/advisories/new)
  on this repository ("Security" tab → "Report a vulnerability").
- **Alternative:** email <stringmymail@gmail.com> with the details.

Please include:

- The `pr-checkmate` version affected.
- Steps to reproduce, or a minimal `pr-checkmate.json` / PR diff that
  triggers the issue.
- The potential impact (e.g. arbitrary code execution, secret exposure,
  privilege escalation in the CI workflow it runs in).

## Response Process

We aim to acknowledge reports within 5 business days, and to ship a fix or a
mitigation plan within 30 days for confirmed critical issues. We'll credit
reporters in the release notes unless anonymity is requested.

## Scope Notes

`pr-checkmate` runs inside your CI pipeline and, depending on how it's
wired up, may hold `pull-requests: write` / `contents: write` permissions
and read the contents of your PR diffs (including the secret-scanning
check). Vulnerabilities that could let a crafted PR or repository escalate
those permissions or exfiltrate data are in scope and treated as high
priority.
