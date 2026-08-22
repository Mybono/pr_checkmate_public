# C++ Format

[Docs](../../README.md) · [Checks Index](../INDEX.md) · [ESLint](eslint.md) · [TypeScript](typecheck.md) ·
[Prettier](prettier.md) · [Ruff Lint](python-lint.md) · [Ruff Format](python-format.md) ·
[Python Types](python-typecheck.md) · **C++ Format** · [SwiftLint](swift-lint.md) · [ktlint](kotlin-lint.md) ·
[Go Vet](go-lint.md) · [Go Format](go-format.md) · [Clippy](rust-lint.md) · [Rustfmt](rust-format.md) ·
[C# Format](csharp-format.md) · [RuboCop](ruby-lint.md) · [PHP CS Fixer](php-format.md) ·
[ShellCheck](shellcheck.md)

---

## Overview

Formats (or checks the formatting of) C/C++ source and header files with clang-format, bundled
as a self-contained WASM CLI (`@wasm-fmt/clang-format`) so it runs on a bare Node image with no
system `clang`/LLVM install.

Style resolution: a `.clang-format` file at the repository root always wins (`--style=file`) so
this check never overrides an established project style. Otherwise it uses
`cpp.clangFormat.style` (default `LLVM`).

Detection is a `--dry-run --Werror` pass first, so nothing is mutated in check mode; the
offending-file count comes from parsing clang-format's stderr for
`file:line:col: error|warning:` lines (falling back to "every target file" if that parse finds
nothing). Only when `ctx.write` is true does it run again with `-i` to rewrite files in place.

Recognised extensions: `*.c`, `*.cc`, `*.cpp`, `*.cxx`, `*.c++`, `*.h`, `*.hh`, `*.hpp`, `*.hxx`
— `.h` is included even though it's ambiguous with plain C, since clang-format handles both.

| Property | Value |
|---|---|
| Display name | `C++ Format` |
| Phase | `format` |
| CLI command | `npx pr-checkmate cpp-format` |
| Config key | `cpp` |
| Toolchain | Bundled |
| Source | `src/core/checks/languages/cpp-format.ts` |

## When it applies

Both of the following must hold:

1. `cpp.enabled` is not `false`
2. `ctx.languages` includes `cpp` — detected via a `CMakeLists.txt` / `compile_commands.json` /
   `conanfile.txt` / `conanfile.py` marker file, or at least one tracked `.cpp`/`.cc`/`.cxx`/
   `.c++`/`.hpp`/`.hh`/`.hxx` file (plain `.c`/`.h` files don't count toward language detection,
   but are still linted once C++ is detected some other way)

File discovery goes through the shared target resolver, like every other language check: in a PR
it diffs `baseSha..headSha` (added/copied/modified/renamed only) for the extensions above; outside
a PR it lists every tracked matching file. `sourcePath`, `ignoreDirs` and `excludePaths` all apply.

This check used to run its own `git` call and honour none of them, which is worth knowing if you
are upgrading: on a repository vendoring third-party C++ it scanned the vendored tree too — 403
files instead of 56 in the case that found it — and in `write` mode clang-format was rewriting
sources the repository does not own. If you added a `cpp.enabled: false` or a `severity` override
to work around that, it is no longer needed.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `cpp.enabled` | boolean | `true` | Set `false` to skip the check |
| `cpp.clangFormat.style` | string | `"LLVM"` | clang-format style name — `LLVM`, `Google`, `Chromium`, `Mozilla`, `WebKit`, `Microsoft`, `GNU`, or an inline `{...}` style string. Only used when the repo has no `.clang-format` file |

### Example

```json
{
  "cpp": {
    "clangFormat": { "style": "Google" }
  }
}
```

## Disabling

```json
{ "cpp": { "enabled": false } }
```

Or:

```json
{ "severity": { "C++ Format": "off" } }
```

## Notes

- A `.clang-format` file always beats `cpp.clangFormat.style` — commit one if the project needs
  anything beyond the seven named presets.
- Bundled as `@wasm-fmt/clang-format`; no system `clang` or LLVM toolchain is needed on the
  runner.
- `git ls-files`/`git diff` failing (e.g. git unusable) is reported as `skip('git unavailable')`;
  an empty file list is reported as `pass('no C/C++ files changed')`.
- In check mode (`ctx.write` false), unformatted files return
  `warn('<N> file(s) need formatting (run with write to fix)')`; the `-i` rewrite pass itself
  failing also returns `warn`, never `fail`.
