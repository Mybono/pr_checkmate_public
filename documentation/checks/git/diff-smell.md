# Diff Smells

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · **Diff Smells** · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)

---

## Overview

A quick advisory sweep of added JavaScript/TypeScript lines for the everyday leftovers of development:
debug output, a forgotten `debugger`, and unresolved comment markers.

| Finding | Matches |
|---|---|
| `console.*()` | `console.log`, `.error`, `.warn`, `.debug`, `.info` — not inside a `//` comment |
| `debugger statement` | the `debugger` keyword |
| `TODO comment` | a `// TODO` line comment |
| `FIXME comment` | a `// FIXME` line comment |
| `HACK comment` | a `// HACK` line comment |
| `XXX marker` | a `// XXX` line comment |

All findings are warnings; the check cannot fail a run on its own.

### How it differs from the adjacent checks

Three checks overlap here, deliberately, and you may want only one of them:

| Check | Scope | Can block? |
|---|---|---|
| **Diff Smells** | JS/TS only, both debug output *and* comment markers, one combined report | no |
| [Leftover Debug](leftover-debug.md) | debug statements across several languages | yes, via `severity` |
| [TODO/FIXME](todo-fixme.md) | comment markers across several languages, with an optional ticket requirement | yes, via `severity` |

If you run the other two, this one is largely redundant for a JS/TS repository — switching it off is a
reasonable way to cut duplicate reporting.

| Property | Value |
|---|---|
| Display name | `Diff Smells` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate diff-smell` |
| Config key | `diffSmell` |
| Source | `src/core/checks/git/diff-smell.ts` |

## When it applies

Both conditions must hold:

1. `diffSmell.enabled` is not `false`
2. a diff range is available (`baseSha` is set)

Only `*.ts`, `*.tsx`, `*.js`, and `*.jsx` files are scanned. There is no equivalent for Python, Go, or
any other language here — use [Leftover Debug](leftover-debug.md) and [TODO/FIXME](todo-fixme.md) for
those.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `diffSmell.enabled` | boolean | `true` | Set `false` to skip the check |

That is the whole surface. The pattern list is fixed in the source: there is no `ignore`, no severity
key of its own, and no way to keep some findings while dropping others.

### Example

```json
{
  "diffSmell": { "enabled": true }
}
```

To keep debug detection but stop the comment markers being reported twice, disable this check and rely
on the two focused ones:

```json
{
  "diffSmell": { "enabled": false },
  "leftoverDebug": { "enabled": true, "severity": "error" },
  "todoFixme": { "enabled": true, "requireTicket": true }
}
```

## Disabling

```json
{ "diffSmell": { "enabled": false } }
```

Or promote every smell to a blocking failure:

```json
{ "severity": { "Diff Smells": "error" } }
```

## Notes

- **`console.*` inside a comment is not reported.** The pattern requires the line to not begin with `//`
  after the diff marker. It is a prefix guard, not a full comment parser — a trailing comment such as
  `foo(); // console.log(x)` still matches.
- The `TODO`/`FIXME`/`HACK`/`XXX` patterns match **only `//` line comments**, so a `# TODO` in a shell
  script or a `/* TODO */` block comment is not reported here.
- The summary is compact: `3× console.*(), 1× TODO comment`.
- The log shows at most **3 examples per label**; matched lines are trimmed to 100 characters.
- The `pr-checkmate-ignore` directive is honoured, since the check reads the diff through the shared
  `diffAddedLines` helper.
- Only added lines are scanned, so existing `console.log` calls are not reported.

---

[Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · **Diff Smells** · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)
