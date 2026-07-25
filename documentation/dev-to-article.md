# PR CheckMate: one install, every PR check, six languages

Every mechanical review nitpick — formatting, linting, typos, secrets, licenses — runs automatically on every PR, before a human looks at it. One npm package, one CI step, no per-repo toolchain setup.

[npmjs.com/package/pr-checkmate](https://www.npmjs.com/package/pr-checkmate)

## Languages (auto-detected)

TypeScript, JavaScript, Python, C++, Swift, Android/Kotlin.

Python and C++ run via bundled WebAssembly — no Python or clang on the runner.

## Checks

- Lint: ESLint (+ SonarJS), Ruff, SwiftLint, ktlint
- Format (auto-fixes and commits back): Prettier, clang-format, Ruff
- Types: `tsc --noEmit`
- Security: secret scan, diff security scan, npm audit
- Dependencies: unused/missing, circular, outdated, license (blocks GPL/AGPL/LGPL)
- Quality: duplicate code, dead code, spellcheck (6 languages)
- PR hygiene: size, title, body, commitlint

Full list is in the README on the npm page.

## Setup

```bash
npm install --save-dev pr-checkmate
npx pr-checkmate init
npx pr-checkmate all
```

`init` detects your languages and writes `pr-checkmate.json` plus a ready GitHub Actions workflow.

## Configuration

Everything lives in one file: `pr-checkmate.json`. Enable or disable any check, tune its rules, ignore paths. Full option list in the README.

## CI/CD

```yaml
name: PR CheckMate
on:
  pull_request:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: actions/setup-node@v4
        with: { node-version: 24 }
      - run: npm ci
      - run: npx pr-checkmate all
```

## Why

- One install, any stack
- Only changed files checked in a PR — fast
- Checks run in parallel
- Auto-fixes formatting and commits back
- Posts one PR summary comment, updated in place
- Every check toggleable per repo

[npmjs.com/package/pr-checkmate](https://www.npmjs.com/package/pr-checkmate)
