# Migrating

<!-- docs-nav -->
[Docs](README.md) · [Config guide](config-guide.md) · [Check reference](checks/INDEX.md) · [How checks work](checks/concepts.md) · [Running in CI](ci-setup.md) · **Migrating** · [Architecture](architecture-and-approach.md) · [Authoring a check](authoring-checks.md)

---

Two upgrades changed what a green build means. Read the section for the version you are coming from;
if you are on 1.x, read both, bottom one first.

## To 3.0

**No API or config keys changed.** Every key you have keeps working, and `runChecks`, `defineCheck`
and the outcome helpers are untouched. What changed is that two checks report where they used to be
silent, so a build can turn red on code you did not touch.

| What | Why | What to do |
|---|---|---|
| **Workflow Security fires on `--full`** | It was blind on the whole-tree path — `.github` is on the default ignore list, so it reviewed nothing and reported a pass. Measured on 28 repositories: five gained findings, 8 to 54 each, almost all "action not pinned to a full commit SHA" | Nothing, unless you promoted it to `error`. Findings are real; pin the actions or demote with `severity` |
| **Python Types runs** | It needed mypy or pyright on the runner and skipped almost everywhere. pyright is bundled now | Nothing, unless you promoted it. Where your project's dependencies are not installed it skips and says so |
| **Security Scan names a number** | The row read `potential secrets detected` for one secret and for thirty-nine | Only if you match that string in CI: it is now `39 potential secret(s)` |
| **First run on a Python repo downloads 19 MB** | pyright ships as the `pytypes` install profile rather than in the tarball | Nothing. Air-gapped: pre-warm with `npx pr-checkmate install --all`, or it skips with a reason |

Two changes only ever produce **fewer** findings, and need no action: `.env.template` and its
siblings are no longer treated as secrets, and `excludePaths` now reaches the three checks that
built their own file list (Sensitive Files, License Header, Lockfile Drift).

The rollback is the same one line as below:

```json
{ "severityDefaults": { "failOn": "warn" } }
```

## To 2.0

A finding rated `error` now fails the run. That is the whole release as far as your pipeline is
concerned; everything else in it is a lever for controlling that one sentence. Until now
`reportSeverityBuckets` ended in `warn()` no matter what was in the error bucket, so an
`eval(userInput)` landing in a diff printed a red row, posted it to the PR, and exited 0. Six checks
report through that function — Diff Security, Workflow Security, Dockerfile Security, Sensitive
Files, Banned Imports and Custom Rules — and for all six the default is now `failOn: "error"`. If
your current report has a red row under any of them and a green build, that build turns red on the
first PR after you upgrade. If it does not, you can upgrade and come back to this page only when
something surprises you.

Worth knowing before you start: all six are diff-only checks, so `pr-checkmate all --full` will not
show you what is about to block. A full run now lists them as skips instead. The honest preview is
one pull request against a branch nobody merges.

### The one-line rollback

```json
{ "severityDefaults": { "failOn": "warn" } }
```

Every pattern-based check goes back to advisory. Findings are still detected, still logged, still in
the PR comment — they no longer set the exit code. No version pin, no per-check edits, and nothing
else in the release depends on this key.

Two things about it:

- It is a fallback, not an override. A check that sets its own `failOn` wins, and so does the
  top-level `severity` map.
- `failOn: "never"` also exists and is the wrong tool here. It passes the check outright; `warn`
  keeps the row visible, and that row is what tells you when the check is ready to be promoted back.

### Recommended migration

Staged, in the order that keeps the queue moving:

1. **Upgrade and change nothing.** You need to see the real report before you tune it, and a
   pre-emptive `severityDefaults` hides exactly the output you are looking for.
2. **Open one throwaway PR and read the summary.** Every check that changed reviews a diff, so this
   is the only way to see the new failures. Note which rows are red and how many findings sit behind
   each one.
3. **Set `severityDefaults` to `warn`** if that list is longer than a day's work. Nothing is muted;
   the gate is off while you clear the backlog.
4. **Promote checks back one at a time** with the `severity` map, cheapest first. A check whose
   findings are already at zero costs nothing to promote and is one fewer hole. Each promotion is a
   line with a reason next to it, the same as every other entry in the file.

The end state is `severityDefaults` deleted and no `severity` entries left. If a check never gets
there, that is a decision worth writing down rather than a config line to leave behind.

```jsonc
{
  "$schema": "https://raw.githubusercontent.com/Mybono/pr_checkmate_public/main/pr-checkmate.schema.json",

  "//severityDefaults": "Upgraded on a six-year-old service. First PR after the bump went red on 34 Diff Security findings, none of them introduced by that PR. Advisory until the backlog is clear; every check below has been promoted back individually. Delete this key once diffSecurity follows.",
  "severityDefaults": { "failOn": "warn" },

  "//severity": [
    "Sensitive Files — promoted immediately. Its error-rated categories are .env files and private keys, we ship neither, so the gate costs nothing and closes the worst hole.",
    "Workflow Security — promoted after pinning the four unpinned actions. Only the script-injection rule is error-rated, and we had none.",
  ],
  "severity": { "Sensitive Files": "error", "Workflow Security": "error" },

  "//diffSecurity": "Still advisory. ignorePaths: test/fixtures holds deliberately malicious payloads for the parser tests and trips eval() and both SQL rules on every run. severityOverrides: the SQL template-literal rule fires on the query builder, where the interpolated value is a column name from a closed set — demoted rather than dropped, so a genuinely new one is still visible. disablePatterns: MD5 is used for cache keys, never for authentication.",
  "diffSecurity": {
    "ignorePaths": ["test/fixtures/**", "**/*.generated.ts"],
    "severityOverrides": { "SQL query with template literal": "warn" },
    "disablePatterns": ["MD5"],
  },

  "//run": "onDegradedScope raised to error: a shallow clone silently drops 16 checks, and a green build that reviewed nothing is worse than a red one that says so.",
  "run": { "onDegradedScope": "error" },
}
```

### If your CI turns red

| Symptom                                                                           | Cause                                                                                                                                                                                  | Fix                                                                                                                                                      |
| --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A check that warned yesterday fails today, with the same findings                 | `failOn` now defaults to `error`, and an error-rated finding fails the run                                                                                                             | Fix the findings, or `severityDefaults: { "failOn": "warn" }` while you work through them                                                                |
| Sensitive Files blocks on a `.pem` fixture or a committed `.env.test`             | `Environment files (.env)` and `Private keys and certificates` are the two error-rated categories                                                                                      | `sensitiveFileGuard.severityOverrides` to demote the category, or `ignore` to drop it. Both match the category label, not the path                       |
| A `Scan Scope` row saying `N diff-only check(s) did not run — set fetch-depth: 0` | Shallow clone. The PR base commit is not in it, no diff range can be built, and the run silently fell back to full                                                                     | `fetch-depth: 0` on `actions/checkout`. The regenerated workflow already sets it. `run.onDegradedScope: "off"` only if scanning everything is deliberate |
| Sixteen checks now appear as skips in a full run                                  | Diff-only checks used to be dropped from the report entirely; they now say why they did not run                                                                                        | Nothing — that is the point. `run.reportSkipped: "never"` restores the old silence                                                                       |
| More `eval()` and prototype-pollution hits than the same code produced before     | The detectors got stricter: `eval()` is caught on lines that also contain a string or template literal, and `obj['__proto__']` and bracket-form `constructor.prototype` are caught now | `diffSecurity.ignorePaths` for test fixtures and generated files                                                                                         |
| A rule fires on a construct that is safe in your codebase                         | Same as above — a pattern that used to be suppressed by a quote elsewhere on the line now matches                                                                                      | `diffSecurity.severityOverrides: { "<label substring>": "warn" }` to demote, `"off"` to drop it, or `disablePatterns` to stop the rule running           |
| A whole line's worth of findings appears in TypeScript files with private fields  | Comment stripping is language-aware now. `#` no longer reads as a comment in C-family files, so `this.#x = …` stops truncating the rest of the line                                    | Nothing to configure — those findings were always there. `diffSecurity.commentHandling: "legacy"` reverts if you need the old behaviour temporarily      |
| The PR summary comment never arrives                                              | The step needs `GITHUB_TOKEN` in its `env`; Actions does not put it there on its own                                                                                                   | Add `env: GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}` to the step, and keep `permissions: pull-requests: write`                                           |
| Prettier fixes stop landing on the branch                                         | The generated workflow no longer commits and pushes formatting changes, and its `contents` permission is `read`                                                                        | Run the formatter locally, or re-add the commit step and the write permission yourself                                                                   |

Two notes on the workflow rows. `pr-checkmate init` never overwrites an existing
`.github/workflows/pr-checkmate.yml`, so none of those changes reach you until you delete the file
and re-run it — including the `GITHUB_TOKEN` line, which is why a comment that never appeared will
keep not appearing. And `init` now asks whether a failed check should block the pull request; the
advisory answer writes a `continue-on-error` into the generated workflow, which is a blunter
instrument than `severityDefaults` and disarms the gate for secrets too.

### Reference: what each knob does

| Key                            | Reads it                                                                       | Values                                         | Effect                                                                                       |
| ------------------------------ | ------------------------------------------------------------------------------ | ---------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `severityDefaults.failOn`      | Top level                                                                      | `error` (default), `warn`, `never`             | Fallback for any pattern check that sets no `failOn` of its own                              |
| `<check>.failOn`               | `diffSecurity`, `workflowSecurity`, `dockerfileSecurity`, `sensitiveFileGuard` | `error` (default), `warn`, `never`             | What a non-empty error bucket does to the run, for that check alone                          |
| `<check>.severityOverrides`    | The same four                                                                  | label substring → `error`, `warn`, `off`       | Re-levels one finding. Case-insensitive; the longest matching substring wins; `off` drops it |
| `<check>.ignorePaths`          | `diffSecurity`, `workflowSecurity`, `dockerfileSecurity`                       | Globs                                          | Whole files skipped before anything is matched                                               |
| `diffSecurity.disablePatterns` | `diffSecurity`                                                                 | Label substrings                               | The built-in rule never runs, rather than running and having its findings discarded          |
| `diffSecurity.extraPatterns`   | `diffSecurity`                                                                 | `{ label, pattern, severity?, view?, flags? }` | Project rules with the same comment handling and grouping as the built-ins                   |
| `diffSecurity.commentHandling` | `diffSecurity`                                                                 | `language` (default), `legacy`, `off`          | `legacy` strips `#` and `//` everywhere; `off` scans commented-out code too                  |
| `run.reportSkipped`            | Top level                                                                      | `auto` (default), `always`, `never`            | Whether diff-only checks that got no diff range appear in the report as skips                |
| `run.onDegradedScope`          | Top level                                                                      | `warn` (default), `error`, `off`               | The `Scan Scope` row when a CI run wanted a diff range and could not build one               |

Four things the table cannot say in a cell:

- **Order of application inside a check is `ignore` → `severityOverrides` → `failOn`.** A finding
  dropped by `ignore` is gone before an override could re-level it.
- **The `severity` map is applied last**, by the orchestrator, and outranks everything above. It
  keeps working exactly as it did: `"error"` promotes, `"warn"` demotes, `"off"` removes the check
  before it runs.
- **Banned Imports and Custom Rules read `severityDefaults` and nothing else.** They have no
  per-check `failOn`, because the severity of a rule is already yours — you wrote it.
- **`extraPatterns` regexes are anchored on the added-line marker for you**, `g` and `y` flags are
  stripped, and a malformed pattern is logged and skipped rather than taking the scan down with it.
  Use `view: "code"` when you are looking for a call rather than for text inside a string.

The published schema also lists `ignorePaths` under `sensitiveFileGuard`. The check does not read
it; use `ignore`, which matches the category label.

---

[Config Guide](README.md) · [Check Reference](checks/INDEX.md) · [Authoring Checks](authoring-checks.md) ·
**Migrating**
