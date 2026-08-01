# Why PR CheckMate is built the way it is

**Published:** 2026-08-01. This document is dated and additive: later updates are
appended as new dated sections, not rewrites of what's here.

This is not the README. The README is an onboarding document — how to install and run
the tool. This is an engineering rationale document: the problem I set out to solve,
the alternatives I considered, why I rejected them, and the concrete tradeoffs I made
instead. It's written so it can be cited on its own, without requiring a walkthrough of
the README or the source.

## The problem: per-language CI setup doesn't scale

A repository with N languages needs, in the naive case, N separate toolchains
installed on the CI runner, N separate lint/format configs, and N separate workflow
steps kept in sync as tool versions move. Multiply by M runner images (Ubuntu, macOS,
self-hosted) and K supported versions of each toolchain, and the actual maintenance
surface is closer to N × M × K than N. In practice this shows up as: a Python check
that silently no-ops because `ruff` isn't on the runner image this month, a Go step
that breaks because the pinned Go version drifted from what `go.mod` declares, or a
security scanner that never got wired in because it needed its own install step and
nobody had the time.

The common answer is per-language GitHub Actions (`actions/setup-go`,
`actions/setup-python`, etc.) composed by hand in every repo, or a monolithic
container that bundles everything.

## Alternatives considered

- **`super-linter`** — a single Docker container bundling dozens of linters. Solves the
  "which tool do I install" problem by shipping a large fixed image, but the image
  itself becomes the maintenance surface: repos take on whatever version of every
  linter the container ships, on its release cadence, not theirs. Configuration is
  spread across per-linter config files layered inside the container's expected
  paths, and delta-only (changed-files-only) behavior has to be opted into and
  configured per linter rather than being one runtime concept applied uniformly.
- **`mega-linter`** — the same Docker-bundling approach as `super-linter`, with a
  larger tool set and its own flavor system to cut image size per stack. Same
  underlying tradeoff: you inherit a container's release cycle and its per-linter
  configuration surface instead of one config file.
- **`lint-staged` + `husky`** — solves delta mode well (it is fundamentally a
  changed-files tool) but is JS/TS-first: extending it to Python, Go, C++, etc. means
  writing and maintaining that wiring yourself, per repo. It also runs at commit time
  on the developer's machine, not as an independent CI gate — a `--no-verify` commit
  or a hook a contributor never installed bypasses it entirely.
- **Hand-rolled per-language GitHub Actions steps** — the default when a team has no
  bundling tool at all. Full control, and also full ownership of every toolchain
  version and every workflow diff, repeated in every repository that needs it.

PR CheckMate's answer: bundle the tools themselves (ESLint, Prettier, Ruff,
clang-format, gitleaks) as WASM/JS dependencies of one npm package, so the languages
that are bundled need nothing installed on the runner at all — no container, no
per-language setup step, no version drift between what the repo expects and what the
runner happens to have. Languages that need a native toolchain (Go, Rust, C#, Ruby,
PHP, Swift, Kotlin/Java) still need that toolchain on the runner, exactly as they
would with any other approach — bundling only removes the tools that *can* run without
one. One config file (`pr-checkmate.json`) covers every check, bundled or not, instead
of one config surface per linter.

## The security architecture: verifying the artifact, not just the source

Most CI security tooling scans source code in the repository. That leaves a gap: the
step between "source looks clean" and "the thing a consumer's `npm install` actually
pulls down" — the publish step — is unverified. A compromised publish credential, a
build-time dependency substitution, or a maintainer's compromised machine can all
produce a tarball that doesn't match what's in git, and source-only scanning would
never see it.

PR CheckMate's release process instead verifies the published artifact itself: every
version ships a CycloneDX SBOM generated from the actual published npm tarball, plus
an independent VirusTotal scan of that same tarball, both attached to the GitHub
release. That is a narrower, more specific guarantee than "the source was scanned" —
it's "this exact file, the one your lockfile will resolve to, was scanned by a party
other than the maintainer." Combined with the bundled gitleaks secret scan (every diff,
every PR) and dependency vulnerability scanning (`npm audit`, grype by default,
osv-scanner opt-in), the tool covers three distinct points in the supply chain: what
gets committed, what gets depended on, and what gets published — rather than only the
first.

This maps directly onto the software supply chain transparency goals in Executive
Order 14028 and NIST's Secure Software Development Framework (SSDF): an SBOM per
release and independent verification of the published artifact are exactly the kind of
provenance those frameworks ask for, applied here at the scale of a single npm
package's release process rather than an enterprise supply chain program.

## Concrete numbers

Two real, dated measurements rather than a marketing estimate:

- **Delta mode in production CI.** PR #153 ("Minor update", 6 files changed,
  +1323/−7 lines) — the `PR CheckMate (self-check)` job's "Run pr-checkmate on itself"
  step ran in **15 seconds**, in GitHub Actions, 2026-08-01
  (run [30697824260](https://github.com/Mybono/pr_checkmate/actions/runs/30697824260)).
  This is the tool checking only the changed files, in its own dogfooding CI.
- **Full-project scan, locally.** The same package's own ~700-file repository,
  scanned end to end with `pr-checkmate all --full` on 2026-08-01: **~18 seconds**
  wall clock, 22 checks passed, 4 warnings, 0 failures, 3 skipped (language checks
  with no matching files).

These two numbers are not a controlled A/B comparison — one ran on a GitHub-hosted
runner, the other on a local machine, and repo size wasn't held constant between them.
They're reported separately, with their actual context, rather than combined into a
single "N× faster" claim that the underlying data doesn't support. A proper controlled
comparison (same machine, same commit, delta vs. `--full`) is an open item — see below.

## Known limitations

- WASM/JS builds of a linter can lag behind that linter's native release — a rule
  added upstream this month may not be available in the bundled version yet. This is
  monitored per-tool but not on a fixed SLA; it's the direct tradeoff for not
  requiring a native toolchain install.
- Bundling covers the languages listed as "Bundled" in the README; everything else
  still needs its native toolchain on the runner, same as any other approach.

## Open items for a future dated update

- A controlled delta-vs-full timing comparison (same commit, same machine) is not yet
  captured — the two numbers above are real but come from different environments.
- A direct, numeric comparison against `super-linter`/`mega-linter` (e.g., cold-start
  time, image/install size, config lines needed for an equivalent check set) would
  strengthen the "Alternatives considered" section beyond the qualitative comparison
  above.
