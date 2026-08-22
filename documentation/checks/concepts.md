# How checks work

<!-- docs-nav -->
[Docs](../README.md) · [Config guide](../config-guide.md) · [Check reference](../checks/INDEX.md) · **How checks work** · [Running in CI](../ci-setup.md) · [Migrating](../migrating.md) · [Architecture](../architecture-and-approach.md) · [Authoring a check](../authoring-checks.md)

---

## Getting a config file

`npx pr-checkmate init` writes one for the languages it detects, and is safe to re-run — it updates
the file rather than replacing it. With no `node_modules` to generate from (a pipeline that calls us
through `npx` or the container), download the template instead:
[Running in CI →](../ci-setup.md).

To switch a check off: `severity: { "<display name>": "off" }` works for every one of them, and some
also take `enabled: false`. Which to prefer, and how to re-grade rather than disable, is in the
[config guide](../config-guide.md#turning-a-check-off).

## How checks are grouped

Checks run in three **phases**, strictly in order. Within a phase every check runs in parallel.

| Phase           | Runs   | An unexpected exception is graded as | Why this order                                                             |
| --------------- | ------ | ------------------------------------ | -------------------------------------------------------------------------- |
| `blocking`      | first  | `fail`                               | Cheap, high-confidence gates fail fast                                     |
| `informational` | second | `warn`                               | A crash in advisory tooling should not stop a merge                        |
| `format`        | last   | `warn`                               | Formatters may **write** files, so they run after everything has read them |

**A phase does not decide whether a check can fail the run.** The run's verdict is
`ok: summary.failed === 0` — _any_ check returning `fail`, in _any_ phase, fails it. Several
`informational` checks do exactly that when a rule is configured at `severity: "error"`
(for example [Banned Imports](dependencies/banned-imports.md)). What the phase actually controls is
execution order, and how an _unexpected exception_ is graded.

A check reports one of four outcomes: `pass`, `warn`, `fail`, or `skip` — the last is used when a
required tool is absent, so a missing toolchain never fails a PR. A check whose `applies()` returns
`false` is omitted from the report entirely rather than listed as skipped, keeping single-language
repos free of irrelevant rows.

## Review scope

What a run looks at is decided once, before any check executes, in this order:

| Scope  | When                         | What is reviewed                                                                               |
| ------ | ---------------------------- | ---------------------------------------------------------------------------------------------- |
| Full   | `--full` on any command      | Every tracked file. Overrides everything below, including CI variables.                        |
| Staged | `npx pr-checkmate precommit` | The staged changes (index vs `HEAD`).                                                          |
| Delta  | CI, pull request             | The PR diff — base SHA from the GitHub `pull_request` event or Bitbucket's destination branch. |
| Delta  | CI, no pull request          | `HEAD^`..`HEAD` — a push or branch build.                                                      |
| Full   | **no CI detected**           | Every tracked file.                                                                            |

A human at a terminal gets a whole-project verdict; a build gets the diff, which is the point of a
PR gate.

The chosen scope is printed on the `[buildContext]` line (`scope=full (no CI detected)`,
`scope=delta (origin/main, PR build)`), because a wrong scope is otherwise invisible: a run that
reviewed one commit and one that reviewed the repository both just print passes. Any base that
cannot be resolved — a shallow clone without the base branch, a first commit with no `HEAD^` —
degrades to a full scan rather than diffing against a missing ref and reviewing nothing.

Checks that only make sense on a diff (`PR Size`, `Diff Security`, `Merge Conflict`, `Custom
Rules`, `Banned Imports`, the git-diff family — 16 in total) gate their own `applies()` on the
presence of a range, so a full-scan run omits them from the report entirely.

### When git cannot answer

Every scoped check asks git which files to review. That question can fail — a shallow clone missing
the base commit, a repository owned by another user so git refuses it, a checkout that is not a
repository at all — and the failure looks nothing like a clean project.

Such a check reports **`⏭️ skip — git unavailable`**, not `pass`: a pass says "we looked and found
nothing", a skip says "we could not look". Reported as a pass, an unreadable repository produced a
green gate over code nobody examined.

Where a check reviews several file sets — one per language — one failed set skips the whole check.

Consequences for a run:

- A skip is neutral: it does not fail the gate, and `severity` cannot make it fail.
- A skip **is** visible in the summary line (`⏭️ 3 skipped`), which is the point.
- If skips appear where you expect results, the repository is the thing to look at — commonly a
  shallow CI clone (`fetch-depth: 0` fixes it) or a container running as a different user than the
  one that owns the checkout.

## Configuring any check

Three mechanisms apply to every check, on top of each check's own keys:

```jsonc
{
  "$schema": "./node_modules/pr-checkmate/pr-checkmate.schema.json",

  // 1. Scope — which directories checks look at
  "sourcePath": "src",
  "ignoreDirs": ["node_modules", "dist", "vendor"],

  // 2. Severity override — keyed by the check's DISPLAY NAME, not its file name
  "severity": {
    "Duplicate Code": "warn",
    "Outdated Deps": "off",
  },

  // 3. Per-check keys — see each check's page
  "prSize": { "maxFiles": 40, "maxLines": 800 },
}
```

- **`severity`** re-scopes any check without touching its own config. The key is the check's display
  name exactly as it appears in the tables below:
  - `"error"` — promotes a `warn` outcome to `fail`
  - `"warn"` — demotes a `fail` outcome to `warn`
  - `"off"` — removes the check from the run before it executes; a universal off-switch that works
    even for checks with no `enabled` flag of their own

  It translates only between `warn` and `fail` — a `pass` or `skip` is never altered.

- **`ignoreDirs`** _replaces_ the default ignore list rather than extending it — the generated
  config contains the full default list so it can be edited directly.
- Unknown top-level keys are reported by [Config Validation](quality/config-validation.md), which
  catches typos instead of silently ignoring them.

## Suppressing a single line

Every check that scans added diff lines honours an inline directive. Put `pr-checkmate-ignore`
anywhere on the line — usually in a trailing comment — and that line is removed from the diff before
any content check sees it:

```ts
const hash = createHash('md5'); // pr-checkmate-ignore — non-cryptographic cache key
```

```python
subprocess.run(cmd, shell=True)  # pr-checkmate-ignore
```

The directive is applied centrally in `diffAddedLines`, so it works for
[Diff Security](security/diff-security.md), [Workflow Security](security/workflow-security.md),
[TODO/FIXME](git/todo-fixme.md), [Leftover Debug](git/leftover-debug.md),
[Banned Imports](dependencies/banned-imports.md), [Custom Rules](git/custom-rules.md) and every other
diff-scanning check — without each of them implementing it.

Two related rules worth knowing:

- **Commented-out code is not flagged.** Code-targeting checks match against the line with its
  trailing `//` or `#` comment stripped, so `// eval(x)` is not reported as an `eval()` call. Quote
  state is tracked, so a `#` inside `"#fff"` or a `//` inside a URL is preserved.
- **`<check>.ignore` filters finding labels, not file paths — except for [YAML Lint](quality/yaml-lint.md).**
  For `diffSecurity`, `workflowSecurity`, and `dockerfileSecurity`, each `ignore` entry is matched as a
  **case-insensitive substring of the finding's label**. So `"ignore": ["md5"]` mutes the
  `MD5 (weak hash …)` finding everywhere, and does _not_ mean "skip files named md5". `yamlLint.ignore`
  is the one exception: it's matched as a glob against the **file path**, filtering which YAML files get
  linted at all — see [YAML Lint](quality/yaml-lint.md) for details.

## Running checks locally

```bash
npx pr-checkmate                    # every check, posts to the PR when in CI
npx pr-checkmate all                # same as above
npx pr-checkmate all --full         # every check over every tracked file
npx pr-checkmate precommit          # staged changes only, never writes files
npx pr-checkmate <command>          # a single check — see the tables below
npx pr-checkmate lint spellcheck    # several checks in one pass
npx pr-checkmate lint --full        # a single check, whole repository
```

`all`, `init` and `precommit` take no other arguments — passing any is an error, so a
run can never look successful for work it skipped.

`--full` is the only flag, and it composes with every command because scope is independent of
which checks run — see [Review scope](#review-scope). Outside CI it is already the default, so
it is only needed to force a whole-repo review inside a build. A misspelled flag aborts the run
rather than falling back to the diff, and `precommit --full` is refused as contradictory.

Five checks have no dedicated CLI command and run only as part of a full run: `Grype Scan`,
`Merge Conflict`, `ktlint`, `SwiftLint`, and `Dead Code`.

---

<!-- docs-nav -->
[Docs](../README.md) · [Config guide](../config-guide.md) · [Check reference](../checks/INDEX.md) · **How checks work** · [Running in CI](../ci-setup.md) · [Migrating](../migrating.md) · [Architecture](../architecture-and-approach.md) · [Authoring a check](../authoring-checks.md)
