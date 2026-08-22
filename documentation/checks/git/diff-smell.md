# Diff Smells

[Docs](../../README.md) · [Checks Index](../INDEX.md) · [Custom Rules](custom-rules.md) · **Diff Smells** · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)

---

## Overview

A quick advisory sweep of added JavaScript/TypeScript lines for the everyday leftovers of development:
debug output, a forgotten `debugger`, and unresolved comment markers.

| Finding | Matches |
|---|---|
| `console.*()` | `console.log`, `.error`, `.warn`, `.debug`, `.info` — in live code, not in a comment or a string |
| `debugger statement` | the `debugger` keyword in live code |
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

`diffSmell.enabled` is not `false`.

A diff range is *not* required: with none available the whole tree is reviewed instead of the check
dropping out, on the same reasoning as the security checks — a leftover `debugger` is leftover whether
it arrived in this PR or an earlier one.

Only `*.ts`, `*.tsx`, `*.js`, and `*.jsx` files are scanned. There is no equivalent for Python, Go, or
any other language here — use [Leftover Debug](leftover-debug.md) and [TODO/FIXME](todo-fixme.md) for
those.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `diffSmell.enabled` | boolean | `true` | Set `false` to skip the check |
| `diffSmell.ignorePaths` | string[] | `[]` | Globs excluded from scanning entirely |

The pattern list itself is fixed in the source: there is no `ignore`, no severity key of its own, and
no way to keep some findings while dropping others.

### Example

`ignorePaths` exists for the files where `console.*` is the output channel rather than a leftover —
build scripts, and config files loaded by another tool's process, where the project logger is not
reachable:

```json
{
  "diffSmell": {
    "enabled": true,
    "ignorePaths": ["scripts/**", ".eslintrc.js"]
  }
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

- If git cannot produce the diff, the check reports `skip — git unavailable` rather than a pass, and
  one unreadable language set is enough to skip the whole check. See
  [When git cannot answer](../concepts.md#when-git-cannot-answer).
- **`console.*` and `debugger` are matched in live code only.** Comments are stripped and string
  literals are blanked before matching, so a `foo(); // console.log(x)` trailing comment and the
  `'no-debugger': 'error'` line in an ESLint config are both left alone — the latter used to be
  reported as a leftover `debugger` in every repo that configures the rule.
- The `TODO`/`FIXME`/`HACK`/`XXX` patterns are scanned with comment handling **off**, since they live
  in comments by definition. They match **only `//` line comments**, so a `# TODO` in a shell script or
  a `/* TODO */` block comment is not reported here.
- The summary is compact: `3× console.*(), 1× TODO comment`.
- The log shows at most **3 examples per label**; matched lines are trimmed to 100 characters.
- The `pr-checkmate-ignore` directive is honoured, since the check reads its lines through the shared
  `reviewLines` helper.
- With a diff range only added lines are scanned; without one every tracked JS/TS file is reviewed.
