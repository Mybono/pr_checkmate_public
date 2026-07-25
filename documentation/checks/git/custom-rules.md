# Custom Rules

[Checks Index](../INDEX.md) · **Custom Rules** · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)

---

## Overview

Runs **your own** regular expressions against the added lines of the diff. This is the escape hatch
for conventions no built-in check covers — a deprecated internal API, a forbidden logging call, a
naming rule, an import path that must not be used outside one package.

It is also the only check where you decide whether a finding blocks: a rule with
`"severity": "error"` returns `fail` and **fails the run**, despite the check sitting in the
`informational` phase.

| Property | Value |
|---|---|
| Display name | `Custom Rules` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate custom-rules` |
| Config key | `customRules` |
| Source | `src/core/checks/git/custom-rules.ts` |

## When it applies

Both conditions must hold:

1. at least one entry in `customRules`
2. a diff range is available (`baseSha` is set)

There is no `enabled` flag — the check is inert until you write a rule.

## Configuration

`customRules` is an array of rule objects.

| Key | Type | Required | Default | Meaning |
|---|---|---|---|---|
| `name` | string | **yes** | — | Rule identifier. Used as the report label when `message` is absent |
| `pattern` | string | **yes** | — | Regex source, matched against each added line |
| `message` | string | no | the `name` | What the report shows — use it to explain the fix |
| `severity` | `error` \| `warn` | no | `"warn"` | `error` **fails the run**; anything else warns |
| `include` | string[] | no | all files | Globs the rule is limited to |
| `exclude` | string[] | no | none | Globs the rule skips |
| `flags` | string | no | none | Regex flags. `g` and `y` are **stripped** — see Notes |

A rule missing `name` or `pattern` is silently dropped. A rule whose `pattern` is not a valid regex is
skipped with a warning in the log, and the rest of the rules still run.

### Examples

Forbid a deprecated internal helper, blocking the merge:

```json
{
  "customRules": [
    {
      "name": "no-legacy-http",
      "pattern": "from ['\"]@acme/http-legacy['\"]",
      "message": "@acme/http-legacy is deprecated — use @acme/http",
      "severity": "error"
    }
  ]
}
```

Scope a rule to one area and exempt its tests:

```json
{
  "customRules": [
    {
      "name": "no-console-in-server",
      "pattern": "console\\.(log|debug)\\(",
      "message": "use the structured logger in server code",
      "include": ["src/server/**"],
      "exclude": ["**/*.test.ts"],
      "severity": "error"
    }
  ]
}
```

Case-insensitive match via `flags`:

```json
{
  "customRules": [
    { "name": "no-todo-assignee", "pattern": "todo\\s*\\(", "flags": "i" }
  ]
}
```

Remember `pattern` is a regex *source inside JSON*, so every backslash must be doubled: `\\.`, `\\s`,
`\\(`.

## Disabling

Remove the rules, or keep them and stop them blocking:

```json
{ "severity": { "Custom Rules": "warn" } }
```

To switch the check off while leaving the rules in the file:

```json
{ "severity": { "Custom Rules": "off" } }
```

## Notes

- **`g` and `y` flags are stripped deliberately.** `RegExp.test()` on a global or sticky regex advances
  `lastIndex` between calls, so the same rule would match on one line and silently skip the next. They
  are removed so a rule behaves per-line regardless of what you pass.
- **`include`/`exclude` are matched against the file path** parsed from the diff header, so a rule can
  be scoped without listing paths in `sourcePath`.
- Findings are grouped by `message`, and each entry is reported as `path: line content`, trimmed to 100
  characters. The log shows at most **3 examples per rule**.
- Only added lines are scanned — pre-existing violations are grandfathered in, so a new rule does not
  demand a repo-wide cleanup before the next PR can merge.
- The `pr-checkmate-ignore` directive is honoured, so a single deliberate line can opt out.
- Matching is line-by-line. A pattern cannot span multiple lines, and there is no AST awareness: a
  match inside a string literal or a comment counts.

---

[Checks Index](../INDEX.md) · **Custom Rules** · [Diff Smells](diff-smell.md) · [Large Files](large-files.md) · [Leftover Debug](leftover-debug.md) · [Lockfile Drift](lockfile-drift.md) · [Merge Conflict](merge-conflict.md) · [Missing Tests](missing-tests.md) · [TODO/FIXME](todo-fixme.md)
