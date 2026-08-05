# Writing your `pr-checkmate.json`

**Config Guide** · [Check Reference](checks/INDEX.md) · [Authoring Checks](authoring-checks.md)

---

This is the manual for tuning PR CheckMate to _your_ repository. It covers the method — how to decide
what belongs in the file and what does not.

For the meaning of an individual key, go to [the check reference](checks/INDEX.md): every check has
its own page listing each key it reads, the default, and how to turn it off. This guide answers the
question that comes first: which keys should you be setting at all?

## The one rule

A config is a list of **deviations from the defaults**, each with a written reason.

The file is deep-merged with the built-in defaults, so an absent key keeps its default and `{}`
behaves exactly like no file at all. Every line you add is therefore a claim that your repository is
different from the norm in a specific way. If you cannot say why a line is there, delete it — a
default you re-state by hand is a default that silently stops tracking ours when it changes.

The corollary: do not start from a full list of every key. Start from an empty file and let the tool
tell you what needs changing.

## Step 1 — Run it before you configure it

```bash
npx pr-checkmate all --full
```

You cannot tune a report you have not seen. Guessing produces configs that mute real findings and
leave the noisy ones on.

Two things to know before you run it:

- **`all` writes files.** Every check that can fix what it finds does, and that starts in the first
  phase, not just in the `format` phase at the end: `eslint --fix` is a blocking check,
  `markdownlint --fix` is informational, and Prettier, Ruff Format and clang-format come last.
  Nothing is committed — the CLI never calls `git commit`, and the auto-commit step lives in the
  GitHub Actions template (`src/config/pr-checkmate-workflow.yml`) — but your working tree changes.
  Run on a clean tree, or on a scratch clone, so `git status` afterwards tells you exactly which
  tool touched what. `npx pr-checkmate precommit` never writes, if you need a read-only look.
- **`--full` matters.** Outside CI a full scan is already the default, but pass it anyway to be
  explicit: you want a whole-repository picture now, not a diff. The 16 diff-only checks (PR Size,
  Diff Security, the git-diff family) drop out of a full run — plan for those separately, see
  [Step 4](#step-4--the-checks-a-full-run-cannot-show-you).

## Step 2 — Sort every row of the report into one of five buckets

The summary is the input to your config. Each row is one of these, and only the last two are worth
configuring:

| Bucket                | What it looks like                                                                               | What to do                                                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| **True finding**      | A real bug, typo, weak hash, or vulnerable dependency                                            | Fix the code. Do not touch the config                                                                         |
| **Not applicable**    | A check for a language you do not have                                                           | Nothing — it never registered. Absent rows need no config                                                     |
| **Missing toolchain** | `⏭️ mypy/pyright not installed`, `⏭️ grype not installed`                                        | Nothing. A skip never fails a PR. Only add a key if you want it to stay off _after_ someone installs the tool |
| **Wrong target**      | The finding is about generated, vendored, or tool-owned files                                    | Scope the check — an `ignore`/`allow`/`ignorePaths` list                                                      |
| **Wrong for my repo** | The check is sound but the convention it enforces is not yours, or its tool conflicts with yours | Turn it off with a measured reason, or re-tune its threshold                                                  |

Two questions resolve most rows:

1. **Did a human write this file?** Generated changelogs, SBOMs, knowledge-graph exports, lockfiles,
   vendored samples and committed build reports are not review material. Scope them out.
2. **Would this row appear on every PR?** A check that always fires teaches reviewers to ignore the
   whole report. Either make it actionable or turn it off — leaving it noisy is the one outcome with
   no upside.

## Step 3 — Apply the smallest deviation that fixes it

### Scoping out files

Reach for a per-check `ignore` before anything global. `ignoreDirs` **replaces** the entire default
ignore list (62 entries: `node_modules`, `dist`, `coverage`, `__pycache__`, `.venv`, `Pods`, …), so
setting it to `["vendor"]` silently un-ignores everything else. Use it only if you are prepared to
restate the full list.

Ignore lists are not all the same shape. This is the part that bites:

| Key                                                                           | Matched against                                     | Merges with the default?                                                                                        |
| ----------------------------------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `yamlLint.ignore`                                                             | File path (glob)                                    | No default                                                                                                      |
| `symlinks.ignore`                                                             | File path (glob)                                    | No default                                                                                                      |
| `markdownlint.ignores`                                                        | File path (glob)                                    | **Appends** — `node_modules/**`, `dist/**`, `coverage/**`, `graphify-out/**`, `.claude/**` are already excluded |
| `cspell.words` / `cspell.ignorePaths`                                         | Word / path (glob)                                  | **Appends** to the bundled dictionary and ignore list                                                           |
| `largeFiles.allow`                                                            | File path (glob)                                    | **Appends** — every common lockfile is already exempt                                                           |
| `missingTests.ignore`                                                         | File path (glob)                                    | **Replaces** — restate the defaults you still want                                                              |
| `duplicate.ignore`                                                            | File path (glob)                                    | **Replaces** (defaults exclude tests and mocks)                                                                 |
| `deadCode.ignoreFiles`                                                        | **Regex**, not glob                                 | **Replaces**                                                                                                    |
| `commitlint.ignores`                                                          | **Regex** against the subject                       | **Replaces** — you lose the `^Release\b` and `^chore\(release\):` skips                                         |
| `diffSecurity.ignore`, `workflowSecurity.ignore`, `dockerfileSecurity.ignore` | **The finding's label**, case-insensitive substring | No default                                                                                                      |
| `dependency.ignore`, `outdatedDeps.ignore`, `security.npm-audit.ignore`       | Package name                                        | No default                                                                                                      |

The last two rows are the reason the table exists: `"diffSecurity": { "ignore": ["md5"] }` mutes the
`MD5 (weak hash)` finding everywhere and does not mean "skip files named md5" — see
[Suppressing a single line](checks/INDEX.md#suppressing-a-single-line) for the full rule. And the
`cspell` block only reaches repositories that have no `cspell.json`, `.cspell.json` or
`cspell.config.json` of their own. With one of those present, yours wins entirely and the block in
`pr-checkmate.json` is ignored.

### Turning a check off

```json
{
  "outdatedDeps": { "enabled": false },
  "severity": { "Duplicate Code": "off" }
}
```

`enabled: false` exists only where a check defines it. `severity: { "<name>": "off" }` works for
every check, keyed by the display name from the summary, and it removes the check _before it runs_.
That last part matters for the three checks that write files but have no `enabled` flag of their own
— ESLint, Prettier and Markdown — where an `off` entry is the only way to stop them touching your
tree. Prefer `severity` when you are switching something off because of a conflict, and keep the
reason next to it.

`severity` also re-grades without disabling: `"error"` promotes a warning to a failure, `"warn"`
demotes a failure to advisory. Use `"warn"` when a blocking check is telling you something true that
you cannot fix this quarter — you keep the signal and unblock the queue.

### Re-tuning a threshold

Measure, do not guess. `prSize` defaults to 50 files / 1000 lines; before raising it, look at what
your merged PRs actually contain:

```bash
for c in $(git log --merges -20 --format=%H); do
  git diff --shortstat $(git rev-parse $c^1) $c
done
```

If the number is inflated by generated files committed into the branch by a pipeline step, say so in
the comment and raise the line limit. The file count usually stays meaningful even when the line
count does not.

## Step 4 — The checks a full run cannot show you

Sixteen checks need a diff and are absent from a `--full` report, so reason about them from your
history instead of waiting for the first PR to be noisy:

| Check                | Ask                                                                                                 | Typical config                                 |
| -------------------- | --------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| **PR Size**          | What do merged PRs actually measure?                                                                | `prSize.maxFiles` / `maxLines`                 |
| **Large Files**      | Does a pipeline step commit big generated artifacts into the branch?                                | `largeFiles.allow`                             |
| **Missing Tests**    | Which directories legitimately ship without unit tests?                                             | `missingTests.ignore`                          |
| **Commitlint**       | What share of the last 300 non-merge commits is Conventional?                                       | `severity: { "Commitlint": "off" }` below ~90% |
| **Migration Safety** | Do migrations live outside the default globs, and is every tracked `*.sql` file really a migration? | `migrationSafety.paths`                        |
| **Diff Security**    | Which labels are false positives in your stack?                                                     | `diffSecurity.ignore`                          |
| **PR Body**          | Do your PRs carry ticket ids?                                                                       | `prBody.requireTicket`                         |

`migrationSafety.paths` replaces the defaults (`**/migrations/**`, `**/migrate/**`,
`**/db/migrate/**`, `**/versions/**`, `**/*.sql`), so restate the ones you still want. It is the one
key on this page that the published schema does not describe — it works, but your editor will not
autocomplete it, and Config Validation only inspects top-level keys so it will not flag a typo
inside the block either.

A one-line measurement beats an opinion. For Commitlint:

```bash
git log --no-merges -300 --format=%s |
  grep -cE '^(build|chore|ci|docs|feat|fix|perf|refactor|revert|style|test)(\(.+\))?!?: '
```

## Step 5 — Re-run and check it landed

```bash
npx pr-checkmate all --full
git status --porcelain   # must be empty, or you know exactly which tool wrote what
```

You are done when:

- **Config Validation is green.** It reports unknown top-level keys, which is how you catch a typo
  like `largeFile` that would otherwise be silently ignored.
- **Nothing rewrote your repository.** A formatter that touches dozens of files on a clean tree is
  misconfigured, not helpful.
- **Every remaining `❌` is something you intend to fix**, not something you have decided to live
  with. Anything in the second category belongs at `severity: "warn"` with a reason.
- **Every remaining `⚠️` is true signal.** If a warning will appear on every PR unchanged, scope it
  or switch it off.

## Comment your reasons in the file

Keys beginning with `//` are inert: nothing reads them, and Config Validation skips them the way it
skips `$schema`. That makes the config its own changelog. Use them — a scoping decision without a
reason becomes unremovable, because nobody later can tell whether it was load-bearing:

```jsonc
{
  "//yamlLint": "Hard error by default and doing real work here — SAM templates pass, because CloudFormation tags (!Ref, !Sub) are recognised as application-specific rather than defects. Only iac/slo-template.yaml is excluded: it declares five top-level `Metadata:` keys to hang YAML anchors off. Duplicate map keys are invalid YAML, so the parser is right, but the shape is deliberate.",
  "yamlLint": { "ignore": ["iac/slo-template.yaml"] },
}
```

A good reason states the measurement (`2026 of 2565 unknown words came from graphify-out/graph.json`),
the blast radius (`a --fix run rewrote 653 lines of a generated changelog`), and the way back
(`re-enable after bumping prettier to ^3 and reformatting once`).

## Gotchas worth knowing before you start

- **ESLint uses your setup; Prettier does not.** The ESLint check prefers your repository's own
  `eslint` binary and config, so a project pinned to ESLint 8 with `.eslintrc.*` keeps working,
  provided the CI step runs `npm install` **before** PR CheckMate. With no `node_modules` it falls
  back to the bundled ESLint 10, which cannot read `.eslintrc.*`, and the check degrades to a
  warning rather than a red gate. The Prettier check always runs the bundled Prettier 3: it honours
  your `.prettierrc` and your `.prettierignore` (or `.gitignore`, if you have no `.prettierignore`),
  but not your pinned version. If your repo enforces Prettier 2 elsewhere, the two will fight over
  ternaries and long assignments — switch one of them off rather than committing both.
- **Fixers run in write mode in a normal run.** That is intended, since fixes land instead of being
  reported, but it means anything generated must be scoped out or the run will fight its generator.
- **A missing tool is a skip, never a failure.** ktlint, SwiftLint, RuboCop, PHP-CS-Fixer, mypy,
  grype and the Go/Rust/.NET toolchains are runner dependencies. You do not need to disable them for
  a runner that lacks them — set the key only to keep them off deliberately once the tool arrives.
- **A language you do not have needs no config**, but a single stray file can conscript one: one
  tracked `.rs` file makes the repo "Rust" and registers Clippy and Rustfmt. Prefer dropping the
  stray artifact from git over disabling the language.
- **`sourcePath` narrows checks; the bloat and secret guards ignore it on purpose** so a large asset
  or a leaked key outside `src/` is still caught.

## Worked example

A JavaScript + Python AWS SAM monorepo, after one full run. Every entry traces to something the run
printed:

```jsonc
{
  "$schema": "https://raw.githubusercontent.com/Mybono/pr_checkmate_public/main/pr-checkmate.schema.json",

  "//severity": [
    "Prettier — repo pins prettier ^2 and enforces it in a pre-commit CI step; the check always runs bundled Prettier 3, which reformatted jest.config.js and one handler into v3 style. Left on, the two tools fight. Re-enable after bumping prettier to ^3.",
    "Commitlint — 67 of the last 300 non-merge commits are Conventional (22%). Warning on ~78% of PRs trains reviewers to ignore the report.",
  ],
  "severity": { "Prettier": "off", "Commitlint": "off" },

  "//yamlLint": "SAM templates pass — CFN tags are not treated as defects. Only iac/slo-template.yaml is excluded: five deliberate duplicate `Metadata:` keys holding YAML anchors.",
  "yamlLint": { "ignore": ["iac/slo-template.yaml"] },

  "//markdownlint": "Runs with --fix, so generated docs must be excluded or it fights its generator: a trial run rewrote 653 lines of the 297 KB CHANGELOG.md. demo-apps/ is vendored.",
  "markdownlint": { "ignores": ["**/CHANGELOG.md", "demo-apps/**"] },

  "//prSize": "maxFiles keeps the default 50 — PRs are 7-17 files. maxLines raised from 1000 because a pipeline step commits the regenerated changelog and graph into the branch: 1220-2958 lines per PR, of which 1177-2660 is that churn and under 300 is human.",
  "prSize": { "maxLines": 4000 },

  "//python": "Ruff Lint reports 16 genuine issues — keep it. Format is off: a full run reformatted 24 of 25 tracked .py files, so every PR would carry unrelated churn. Typecheck is off: no config, no annotations, no checker in the runner image.",
  "python": { "typecheck": false, "ruff": { "lint": true, "format": false } },

  "//security": "Dev-only tooling whose sole remedy is a major bump; none of it ships in a Lambda bundle. Still reported as warnings. fast-xml-parser is deliberately absent — a runtime dependency with a high-severity advisory and a non-major fix, so it stays blocking.",
  "security": { "npm-audit": { "ignore": ["eslint", "jest", "sequelize-cli", "sqlite3"] } },
}
```

Note what is **not** in it: no `ignoreDirs`, no restated defaults, no entries for ktlint or SwiftLint
(absent languages), and no blanket `npm-audit` disable — the one genuinely exploitable dependency is
left blocking on purpose.

## Checklist

- [ ] Ran `all --full` on a clean tree and read the whole summary
- [ ] `git status` reviewed — every write was expected
- [ ] Each config entry traces to a specific line of that output
- [ ] Each entry has a `//` comment with the measurement and the way back
- [ ] Nothing restates a default; nothing scopes out code a human wrote
- [ ] Reasoned about the diff-only checks from git history
- [ ] Re-ran: Config Validation green, remaining findings all actionable

---

**Config Guide** · [Check Reference](checks/INDEX.md) · [Authoring Checks](authoring-checks.md)
