# Authoring a new check

Every check in PR CheckMate resolves the files it looks at through one place:
`src/core/targets.ts`. A recent check skipped those
helpers, called `git diff` directly, and would have scanned build and vendor
directories while ignoring a client's `sourcePath`. This guide records the
patterns that avoid that, so the next check gets file scoping right the first
time.

## Folder layout

Checks live under `../src/core/checks/`, grouped into category folders by
purpose:

- `git/` — merge-conflict, leftover-debug, todo-fixme, large-files,
  lockfile-drift, missing-tests, diff-smell
- `security/` — diff-security, sensitive-file-guard, migration-safety, security
- `dependencies/` — dependency-check, npm-audit, outdated-deps, circular-deps,
  license-check
- `quality/` — duplicate-check, dead-code, spellcheck, markdown-lint, coverage
- `pr/` — pr-body, pr-size-check, pr-title-lint, commitlint
- `languages/` — lint, prettier, typecheck, python-lint, python-format,
  python-typecheck, ruff-shared, cpp-format, swift-lint, kotlin-lint, go-lint,
  go-format, rust-lint, rust-format, csharp-format, ruby-lint, php-format

A new check goes in the folder that matches what it does. The shared helper
`findings.ts` stays at the checks root, since every category imports from it.

Because check files sit one level deeper than the old flat layout, their imports
reach one more directory up. A check in a category folder imports `../../types`,
`../../targets`, `../../../utils`, and `../../../config`, and reaches the shared
helper as `../findings`. The existing files in each folder show the exact depth.

## Anatomy of a check

A check is an object that satisfies the `Check` interface in
`src/core/types.ts`:

```typescript
export interface Check {
  name: string;
  phase: Phase;
  applies(ctx: CheckContext): boolean | Promise<boolean>;
  run(ctx: CheckContext): Promise<CheckOutcome>;
}
```

`applies(ctx)` decides whether the check is relevant to this repo and run.
`run(ctx)` does the work and returns an outcome. Build the object with
`defineCheck({ ... })` for full type inference, or annotate it as `Check`
directly like the existing checks do.

The context passed to both methods is fixed and shared. No check reads
`process.cwd()` or `process.env` on its own:

```typescript
export interface CheckContext {
  cwd: string;
  languages: Language[];
  config: PRCheckMateConfig;
  write: boolean;
  baseSha?: string;
  headSha?: string;
}
```

### Phases

`Phase` is `'blocking' | 'informational' | 'format'`. The orchestrator in
`src/core/run-checks.ts` runs the phases strictly
in order and runs the checks within a phase in parallel.

- `blocking`: a `fail` here sets `report.ok = false` and fails the run. Use it
  for gates that must stop a merge (merge conflicts, type errors, secrets).
- `informational`: only ever warns or skips, never blocks. Use it while a check
  is being rolled out, or for advisory findings (diff smells, sensitive files).
- `format`: runs last because formatters may write files. Use it for checks that
  reformat and commit code back (Prettier, clang-format, Ruff format).

The distinction is enforced at two points. In the registry, blocking failures
flip `report.ok`. In `run-checks.ts`, a throw is translated to `fail` in a
blocking check and to `warn` in the other two phases.

### Outcomes

Return one of the constructors from `types.ts`:

- `pass(details?)`: the check ran and found nothing wrong.
- `warn(details)`: an advisory finding. Never blocks.
- `fail(details)`: a real violation. Any `fail` fails the run regardless of
  phase — phase does not decide blocking. See
  [Client-configurable severity](#client-configurable-severity).
- `skip(details)`: the check could not run (for example git was unavailable).

`run` must not throw to signal a violation. Return `fail` or `warn` instead. A
thrown error is caught by the orchestrator and recorded as a `fail` (blocking)
or `warn` (informational and format), which is a fallback for unexpected
crashes, not a control-flow mechanism.

## The golden rule: resolve files through targets.ts

Never call `git diff` or `git ls-files` directly to build the list of files a
check operates on. Always go through the helpers in
`src/core/targets.ts`.

Those helpers give every check the same guarantees for free:

- Delta mode via `ctx.baseSha`: when a diff range is present, only changed files
  are scanned; otherwise the helpers fall back to the full tracked file set.
- `ctx.headSha ?? 'HEAD'` for the head of the range, never a hardcoded `'HEAD'`.
- `--diff-filter=ACMR`, so only Added, Copied, Modified, and Renamed files come
  back. Deletions are dropped, so every returned path exists on disk and is safe
  to read.
- `ignoreDirs` exclusion, so build, dependency, tooling, and report directories
  (`dist`, `build`, `vendor`, `reports`, and the rest of the curated default)
  are never scanned, even when committed.
- The client's `sourcePath`, so scanning stays inside the configured
  subdirectory.

This matters most in client CI repos, where PR CheckMate runs as an installed
package against code it does not own. A client may set `sourcePath` to `"src"`,
or commit generated directories that should never be linted. A check that
shells out to `git diff` on its own bypasses all of that: it scans the whole
diff, reads files under ignored directories, and treats a narrowed `sourcePath`
as if it were the repo root. Resolving through `targets.ts` is the only way a
new check inherits the same scoping as every existing one.

## Which helper to use

Pick the resolver by the kind of check you are writing.

### File-list check

Lint, format, and file-name checks that operate on whole files. Use
`resolveTargetFiles(ctx, globs)`, or `resolveScopedTargetFiles(ctx, globs,
sourcePaths)` when the check should honour `sourcePath`. Both return `null` when
git is unusable (the caller should `skip`) and `[]` when nothing matches.

The Ruff lint check in
`src/core/checks/languages/python-lint.ts`
is the model:

```typescript
const targets = await resolveTargetFiles(ctx, PY_GLOBS);
if (targets === null) return skip('git unavailable');
if (targets.length === 0) return pass('no Python files changed');
```

### Diff-content check scoped to a language

Checks that scan added lines for patterns within a set of language file globs.
Use `diffAddedLines(ctx, globs)` from
`src/core/checks/findings.ts`, which returns
the raw `git diff --unified=0 --diff-filter=ACMR` output for those globs. Pair it
with `reportSeverityBuckets` and the `FindingBucket` type to group results by
severity.

The diff-security check in
`src/core/checks/security/diff-security.ts`
passes explicit language globs:

```typescript
const stdout = await diffAddedLines(ctx, ['*.ts', '*.tsx', '*.js', '*.jsx', '*.mjs', '*.cjs']);
```

The diff-smell check in
`src/core/checks/git/diff-smell.ts`
follows the same shape with its own glob set.

### Language-agnostic content check

Checks whose patterns apply to any file type, like the merge-conflict scan.
Resolve the scoped file list first, then run the content diff limited to exactly
those files. The merge-conflict check in
`src/core/checks/git/merge-conflict.ts`
is the reference:

```typescript
const files = await resolveScopedTargetFiles(ctx, ['*'], getSourcePaths());
if (files === null) return skip('git unavailable');
if (files.length === 0) return pass();

const { stdout } = await execa(
  'git',
  ['diff', '--unified=0', ctx.baseSha as string, ctx.headSha ?? 'HEAD', '--', ...files],
  { cwd: ctx.cwd },
);
```

`['*']` matches every path because conflicts are language-agnostic, and scoping
still keeps ignored and out-of-scope directories out. The content diff then uses
`ctx.headSha ?? 'HEAD'`, never a bare `'HEAD'`. Do not skip the resolve step and
diff the whole range directly. That is the mistake this guide exists to prevent.

### Whole-repo analysis

Checks that need the full file graph rather than the diff, like
circular-dependency detection. Use `listTrackedFiles(ctx, globs, sourcePaths)`,
which lists all tracked files (never delta) and still applies `ignoreDirs` and
`sourcePath`. Because it uses `git ls-files`, untracked directories such as
`node_modules` are excluded by construction.

## The applies convention

`applies(ctx)` gates on a config toggle plus the relevant condition. The toggle
check uses `!== false` so a check stays on unless a client explicitly disables
it in `pr-checkmate.json`.

Delta-only checks also require a diff range. The merge-conflict and diff checks
gate on `Boolean(ctx.baseSha)`:

```typescript
applies(ctx: CheckContext): boolean {
  return ctx.config.mergeConflict?.enabled !== false && Boolean(ctx.baseSha);
}
```

Language checks gate on the detected language set instead:

```typescript
applies(ctx: CheckContext): boolean {
  return (
    ctx.config.python?.enabled !== false &&
    ctx.config.python?.ruff?.lint !== false &&
    ctx.languages.includes('python')
  );
}
```

A check whose `applies` returns false is omitted from the report entirely rather
than listed as skipped, so single-language repos are not filled with noise.

## Client-configurable severity

Each check returns its own default severity: `fail()` for a real violation,
`warn()` for advice. That default decides whether a fresh run blocks, and the
report's outcome is `report.ok = summary.failed === 0` in
`src/core/run-checks.ts`, so a single `fail()`
from any check fails the run. `phase` only orders execution and decides how a
*thrown* error is classified (`fail` in blocking, `warn` elsewhere). A check
returning `fail()` from the informational phase blocks exactly as one in the
blocking phase does.

On top of that default, a client can override the severity of any check by name
without touching check code.

### The universal severity map

`pr-checkmate.json` takes a top-level `severity` map keyed by a check's display
name (its `name` field, not the file or CLI slug). It accepts three values:

- `"error"` — treat the check's findings as blocking; fail the run.
- `"warn"` — treat the check's findings as advisory; do not block.
- `"off"` — disable the check entirely; it never runs.

```jsonc
{ "severity": { "Merge Conflict": "warn", "ESLint": "error", "Spellcheck": "off" } }
```

The map layers on top of whatever a check returns. `applySeverity` in
`src/core/run-checks.ts` remaps the outcome:
`error` promotes a returned `warn` to `fail`, `warn` downgrades a returned `fail`
to `warn`, and `pass`/`skip` are never changed. `off` is handled one step
earlier: `runChecks` filters any check whose name maps to `"off"` out of the
selected set, so it is excluded before it can run. Both work for every check,
old and new, because they act on the check's name and outcome rather than its
internals. This is the recommended way to control severity per repo.

The field is typed as `severity?: Record<string, 'error' | 'warn' | 'off'>` in
`src/interfaces/index.ts`.

### Per-check severity fields

The newer diff-based checks still expose their own `<check>.severity: 'error' |
'warn'` field next to `enabled`, for example
`mergeConflict?: { enabled?: boolean; severity?: 'error' | 'warn' }`. These
continue to work, but the universal `severity` map is the preferred way to set
severity for any check.

Inside a check, resolve the level once with
`severityEmitter(ctx.config.<check>?.severity, <fallback>)` from
`src/core/checks/findings.ts`. It returns a
`log` function, an `icon`, and an `emit` constructor already matched to the
resolved level: `logger.error` / ❌ / `fail` for `error`, and `logger.warn` /
⚠️ / `warn` for `warn`. Call `sev.log(...)` for per-finding logging and
`return sev.emit(summary)` for the outcome, so logging and the outcome stay
consistent at whatever level the check chose. The fallback encodes the default:
merge conflicts pass `'error'`, while leftover debug, lockfile drift, and large
files pass `'warn'`.

```typescript
const sev = severityEmitter(ctx.config.mergeConflict?.severity, 'error');
for (const h of hits) sev.log(`[Merge Conflict]: ${sev.icon} ${h.file}:${h.line}`);

return sev.emit(summary);
```

Whichever level a check settles on, the universal `severity` map still has the
final say, so a client can override it by name.

## Delta versus full scan

When `ctx.baseSha` is set, the file helpers restrict results to the diff range.
When it is absent, `resolveTargetFiles` and `resolveScopedTargetFiles` fall back
to scanning all tracked files, while `listTrackedFiles` always scans everything.

Diff-content checks have no full-repo equivalent: `diffAddedLines` needs a base
SHA to compute added lines, and scanning added lines across an entire repo has no
meaning. That is why these checks gate on `Boolean(ctx.baseSha)` in `applies`
and skip cleanly outside a PR context.

## Wiring a new check

1. Create the check file in the category folder that matches its purpose:
   `src/core/checks/<category>/<name>.ts` (`git`, `security`, `dependencies`,
   `quality`, `pr`, or `languages`). Export the `Check` object and use the
   deeper import paths a category folder needs (`../../types`, `../../targets`,
   `../../../utils`, `../../../config`, `../findings`).
2. Add its config type to `PRCheckMateConfig` in
   `src/interfaces/index.ts`. Follow the existing
   toggle shape, for example `mergeConflict?: { enabled?: boolean }`.
3. Register it in the correct phase group in
   `src/core/registry.ts`. The registry is ordered
   by phase (blocking, then informational, then format); put the import and the
   `REGISTRY` entry under the matching comment.
4. If the check should be runnable on its own, add a mapping to `COMMAND_TO_CHECK`
   in `src/cli.ts`, for example
   `'merge-conflict': 'Merge Conflict'`. The value must match the check's `name`
   exactly. Without a mapping the check still runs as part of `all`.
5. Update the README language and check tables through the doc-writer flow. Do
   not edit them by hand as part of the code change.

## How it runs in CI

The generated workflow in
`src/config/pr-checkmate-workflow.yml`
checks out with `fetch-depth: 0`, which gives git the full history that delta
mode needs to compute a diff range. The default run is `npx pr-checkmate all`,
which in `src/cli.ts` calls `runChecks` with the `github`
reporter and `write: true`. That posts a summary comment on the PR and lets
format-phase checks write their fixes, which a follow-up step commits and pushes
back to the branch.

Individual commands (`npx pr-checkmate lint`, `npx pr-checkmate diff-security`,
and so on) run a single check with the `stdout` reporter instead.

## A correct check skeleton

This skeleton lives in a category folder (for example `src/core/checks/git/`),
so the imports reach the right depth:

```typescript
import { getSourcePaths } from '../../../config';
import { resolveScopedTargetFiles } from '../../targets';
import { Check, CheckContext, pass, skip } from '../../types';
import { severityEmitter } from '../findings';

export const exampleCheck: Check = {
  name: 'Example',
  phase: 'blocking',

  applies(ctx: CheckContext): boolean {
    return ctx.config.example?.enabled !== false && Boolean(ctx.baseSha);
  },

  async run(ctx: CheckContext) {
    // Resolve files the canonical way: delta-aware, honouring ignoreDirs and sourcePath.
    const files = await resolveScopedTargetFiles(ctx, ['*'], getSourcePaths());
    if (files === null) return skip('git unavailable');
    if (files.length === 0) return pass();

    const violations = await inspect(ctx, files); // your logic here
    if (violations.length === 0) return pass();

    // Level comes from config; error blocks, warn advises. Logging follows it.
    const sev = severityEmitter(ctx.config.example?.severity, 'error');
    for (const v of violations) sev.log(`[Example]: ${sev.icon} ${v}`);

    return sev.emit(`${violations.length} violation(s)`);
  },
};
```

Add `example?: { enabled?: boolean; severity?: 'error' | 'warn' }` to
`PRCheckMateConfig`, register
`exampleCheck` under the matching phase in the registry, and the check inherits
the same file scoping as every other one.
