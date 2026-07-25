# Diff Security

[Checks Index](../INDEX.md) · **Diff Security** · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · [Workflow Security](workflow-security.md)

---

## Overview

Pattern-matches **added diff lines** for dangerous constructs — injection vectors, weak cryptography,
disabled TLS verification, unsafe deserialization. It is a fast heuristic layer, not a substitute for a
real SAST tool, and it is deliberately scoped to what the PR *adds* so existing code does not drown the
signal.

JavaScript/TypeScript patterns always run. Language-specific sets are added only when that language is
detected, so a Python repository gets the Python rules without paying for the rest.

| Pattern set | Files scanned | When |
|---|---|---|
| JS/TS | `*.ts` `*.tsx` `*.js` `*.jsx` `*.mjs` `*.cjs` | always |
| Python | `*.py` | Python detected |
| Go | `*.go` | Go detected |
| Kotlin/Java | `*.kt` `*.kts` `*.java` | Kotlin detected |

### What it looks for

| Category | Examples |
|---|---|
| Code injection | `eval()`, `new Function()`, Python `exec()`, `pickle.load` |
| Command injection | `exec`/`spawn` with template literals or concatenation, `os.system()`, `subprocess(shell=True)`, `Runtime.exec()` |
| SQL injection | query built with template interpolation, string concatenation, `%` formatting, or `fmt.Sprintf` |
| XSS | `dangerouslySetInnerHTML`, `innerHTML` assignment, `document.write()` |
| Prototype pollution | `__proto__` mutation, `constructor.prototype` access |
| Weak crypto | MD5, SHA-1, DES, ECB mode, `createCipher`/`createDecipher` |
| Transport security | `verify=False`, `InsecureSkipVerify: true`, `ALLOW_ALL_HOSTNAME_VERIFIER`, plain `http://` calls |
| CORS | wildcard `origin: '*'` and `Access-Control-Allow-Origin: '*'` |

| Property | Value |
|---|---|
| Display name | `Diff Security` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate diff-security` |
| Config key | `diffSecurity` |
| Source | `src/core/checks/security/diff-security.ts` |

## When it applies

Both conditions must hold:

1. `diffSecurity.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `diffSecurity.enabled` | boolean | `true` | Set `false` to skip the check |
| `diffSecurity.ignore` | string[] | `[]` | Finding **labels** to mute, matched as case-insensitive substrings |

`ignore` filters by label text, **not** by file path. To find the label to mute, read it from the
report — for example `MD5 (weak hash — use sha256 or stronger)` is muted by any of `"md5"`,
`"weak hash"`, or the full string.

### Examples

Mute the two weak-hash findings project-wide, because MD5 is only used for cache keys:

```json
{
  "diffSecurity": { "ignore": ["md5", "sha-1"] }
}
```

Mute the CORS wildcard rule on a deliberately public API:

```json
{
  "diffSecurity": { "ignore": ["CORS wildcard"] }
}
```

For a one-off exception prefer the inline directive, which is narrower than muting a label everywhere:

```ts
const legacy = createHash('md5'); // pr-checkmate-ignore — matches upstream cache keys
```

## Disabling

```json
{ "diffSecurity": { "enabled": false } }
```

## Notes

- **This check never fails the run.** Patterns are tagged `error` or `warn` internally, which controls
  the log icon (❌ vs ⚠️) and the summary wording — but the outcome is always `warn` when anything is
  found. Use `severity: { "Diff Security": "error" }` to make findings blocking.
- **Comments are not scanned.** Each line is matched with its trailing `//` or `#` comment stripped, so
  commented-out code is not reported. The *original* line is what appears in the log, for context.
- The log shows at most **3 examples per finding label**, then `... and N more`. Matched lines are
  trimmed to 120 characters.
- Regex heuristics produce false positives. The SQL-concatenation rule, for instance, requires a whole
  SQL keyword inside a quoted string followed by concatenation — specifically so expressions like
  `insertions + deletions` are not flagged — but the general limitation stands.
- If the diff cannot be read the check returns `skip('diff unavailable')`.

---

[Checks Index](../INDEX.md) · **Diff Security** · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · [Workflow Security](workflow-security.md)
