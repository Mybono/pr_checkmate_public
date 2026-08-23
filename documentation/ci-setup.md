# Running in CI

<!-- docs-nav -->
[Docs](README.md) · [Config guide](config-guide.md) · [Check reference](checks/INDEX.md) · [How checks work](checks/concepts.md) · **Running in CI** · [Migrating](migrating.md) · [Architecture](architecture-and-approach.md) · [Authoring a check](authoring-checks.md)

---

## Without installing anything

Two files from the public repository, and nothing in your `package.json`:

- **[pr-checkmate-workflow.yml](https://github.com/Mybono/pr_checkmate_public/blob/main/pr-checkmate-workflow.yml)** → `.github/workflows/`.
  Runs the container, image tag already pinned.
- **[pr-checkmate.json](https://github.com/Mybono/pr_checkmate_public/blob/main/pr-checkmate.json)** → repository root. Every check with its
  default and a comment; one-line edits from there. The
  [schema](https://raw.githubusercontent.com/Mybono/pr_checkmate_public/main/pr-checkmate.schema.json) it references gives editors autocompletion.

Both are regenerated on every release.

### Why the container rather than `npx`

A cold `npx` downloads 547 MB and takes about five minutes **on every job**, because there
is no `node_modules` to reuse. The image is 161 MB, cached by the runner, and carries the
binaries some checks would otherwise skip for — ShellCheck among them.

```bash
docker run --rm -v "$PWD:/repo" --user "$(id -u):$(id -g)" \
  ghcr.io/mybono/pr-checkmate:3 all
```

`--user` keeps files the formatters rewrite owned by you rather than by root. The image
carries SLSA provenance and an SBOM in the registry:

```bash
docker buildx imagetools inspect ghcr.io/mybono/pr-checkmate:3 \
  --format '{{ json .Provenance }}'
```

[All tags →](https://github.com/users/Mybono/packages/container/package/pr-checkmate)

## GitHub Actions

`init` generates this workflow; it is reproduced here so nothing about it is a
surprise. One job runs the whole registry — which checks run is decided in
`pr-checkmate.json`, not by copying YAML.

```yaml
name: PR CheckMate
on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  pr-checkmate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4.4.0
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020 # v4.4.0
        with: { node-version: 24, cache: 'npm' }
      - run: npm ci
      - name: Run PR CheckMate
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx pr-checkmate all
```

Three lines are load-bearing:

- `fetch-depth: 0` — without the full history the PR base is not in the clone, no
  diff range can be built, and every diff-based check skips itself. The run says so
  (`Scan Scope: … set fetch-depth: 0`) rather than reporting a clean pass.
- `GITHUB_TOKEN` — Actions does not put it in the environment on its own. Without it
  the checks still run, but the summary comment never reaches the PR.
- no `continue-on-error` — a gate that cannot fail the job is not a gate.

## Air-gapped and cached runners

Only one tool is fetched at run time — pyright, for Python Types. Everything else ships in the
package. The keys below are described in the [config guide](config-guide.md#tooling-that-is-fetched-not-shipped);
this is where to put them.

**The container needs none of this.** It carries pyright already and runs with fetching disabled.

Running through `npx` instead, persist `~/.cache/pr-checkmate` or the download repeats every job.
The generated GitHub workflow already does. Elsewhere:

```yaml
# .gitlab-ci.yml
cache:
  key: "pr-checkmate-$CI_JOB_IMAGE"
  paths: [.cache/pr-checkmate]
variables:
  PR_CHECKMATE_CACHE_DIR: $CI_PROJECT_DIR/.cache/pr-checkmate
```

```yaml
# bitbucket-pipelines.yml
definitions:
  caches:
    pr-checkmate: ~/.cache/pr-checkmate
```

**Closed network.** Bake the cache while you still have one, then forbid every fetch:

```bash
npx pr-checkmate install --all      # into ~/.cache/pr-checkmate, or PR_CHECKMATE_CACHE_DIR
```

```jsonc
{ "install": { "offline": true } }
```

A read-only cache baked into an image is fine — it is read, never written. If something is
missing anyway, the affected check skips and says which setting stopped it; add
`"onMissingTool": "error"` when a check that cannot run should redden the build instead.

## Adopting it on an existing codebase

The instinct on an existing codebase is to soften the workflow. Do the opposite: keep
the gate real and relax individual checks in `pr-checkmate.json`, so whatever you have
already made clean stays enforced.

```jsonc
{
  // everything advisory to begin with…
  "severityDefaults": { "failOn": "warn" },
  // …except the ones you are ready to enforce
  "severity": {
    "Security Scan": "error",
    "Diff Security": "error",
    "Duplicate Code": "off"
  }
}
```

Promote checks one at a time as the codebase catches up. `pr-checkmate init` also
offers an advisory workflow if you would rather start there, and it writes a banner
explaining how to leave that mode. Either way, `init` never overwrites a workflow file
that already exists.

For a runner-dependency language, add its toolchain setup step before the run: Go with
`actions/setup-go`, Rust with `dtolnay/rust-toolchain`, C# with `actions/setup-dotnet`,
Ruby with `ruby/setup-ruby` (plus `gem install rubocop`), PHP with
`shivammathur/setup-php` (tools: `php-cs-fixer`), Kotlin by fetching the `ktlint`
binary. Swift's `swiftlint` is pre-installed on `macos-latest`, and `shellcheck` is
pre-installed on `ubuntu-latest` — neither needs a setup step. Anything missing is
skipped, not failed.
