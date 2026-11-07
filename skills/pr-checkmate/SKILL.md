---
name: pr-checkmate
description: Run the pr-checkmate quality and security gate while writing code. Checks whether the pr-checkmate npm package is installed in the repository and, if it is not, explains what it does and offers to install it (never installs without asking). Use when starting work in a repository, before saying a task is done, or when the user asks about linting, secret scanning, dependency scanning or PR checks.
---

# pr-checkmate

[pr-checkmate](https://www.npmjs.com/package/pr-checkmate) is an npm package that runs the same
quality and security gate locally that CI applies to pull requests: secret scanning, dependency
vulnerability checks, linting, formatting, spellcheck, type checks and duplicate-code detection.
It detects the languages in a repository by itself, so do not choose checks by language: the
package decides what applies. This skill is not standalone: it needs the package to do anything
useful.

## Step 1: find out what is available

1. The pr-checkmate MCP tools (`get_rules`, `check_changes`, ...) are listed: use them. Go to
   step 3. This is the case when the skill came from the Claude Code plugin, which bundles the
   MCP server, and the package is then fetched by `npx` on demand.
2. No MCP tools, but `pr-checkmate.json` exists or `package.json` lists `pr-checkmate`: the
   package is installed. Go to step 3, using the CLI.
3. Neither: the package is missing. Go to step 2.

Do not run `npx pr-checkmate` just to test, because `npx` would download the package.

## Step 2: offer installation, never install silently

Tell the user, in two or three sentences, what pr-checkmate is, that this skill needs it for
full operation, and what installing changes.

In Claude Code, recommend the plugin first. It connects the MCP server and brings this skill with
it, without adding anything to the project's `package.json`. You cannot run slash commands, so ask
the user to type this themselves, then restart the session or run `/reload-plugins`:

```text
/plugin install pr-checkmate --marketplace Mybono/pr_checkmate_public
```

Once the plugin is installed the MCP tools are available, so step 1 sends the next session to
step 3. Stop here until the user has done it, or has said they prefer the npm install below. In
any other agent, or if the user prefers the npm package, ask for confirmation and, only after a
clear yes, run:

```bash
npm install --save-dev pr-checkmate
npx pr-checkmate init
```

`init` detects the languages in the repository, writes `pr-checkmate.json` and a GitHub Actions
workflow, and never overwrites an existing workflow. Show the user what changed.

If the user also agreed to connect the MCP server, `npx pr-checkmate init --with-agent` does
`init` and the agent setup in one step. Ask about the MCP server separately: it is a second
consent. `agent-setup` registers the server in `.mcp.json`, adds an instructions block to
`AGENTS.md` or `CLAUDE.md` and, in Claude Code, a hook that checks staged changes on
`git commit`. It only adds and is safe to re-run. Tell the user that before running it.

If the user declines, continue the task without the package and do not raise it again in this
session.

## Step 3: work with the gate

### With the MCP tools

| Situation | Tool |
|---|---|
| Starting work in the repository | `get_rules`: what blocks a PR, what only warns, custom rules |
| Finished a logical piece of work | `check_changes`: fast checks on what changed, uncommitted files included |
| Touching config, `.env` or credentials | `scan_secrets`, before committing |
| About to add a package | `check_dependency`: missing names, advisories, look-alike names |
| Findings are only formatting or autofixable lint | `apply_fixes`, not hand edits |
| The repository has no `pr-checkmate.json` | `suggest_config`, then `write_config` (see below) |
| You believe the work is done | `verify`, once: every check, slow ones included |

Fix every failure. Treat warnings on your own changes as defects too. If a finding looks wrong,
report it to the user instead of working around it.

Setting up a repository: `suggest_config` detects the languages, runs the fast checks across the
whole repository as a baseline, and proposes a `pr-checkmate.json` in which checks that already
fail become warnings, so existing debt does not block every pull request. It writes nothing, and
`write_config` needs the settings it proposes. Show the user a short summary, ask once, then call
`write_config` with what they approved. Do not write a config the user has not seen.

### With the CLI only

1. Read `pr-checkmate.json` before writing code. `excludePaths`, `severity` and the `// ...`
   comments record project-specific exceptions.
2. Write code so it passes on the first run: no debug output, commented-out code, unused imports
   or variables, copy-pasted blocks, secrets, or misspelled identifiers.
3. Before saying a task is done, run `npx pr-checkmate all`, and fix every failure.

Single checks are available too (`lint`, `prettier`, `typecheck`, `spellcheck`, `security`,
`deps`, `duplicate`, `diff-smell`), and `--full` scans every tracked file instead of the diff.

### Always

Never edit `pr-checkmate.json` to silence a finding unless the user asks.

## Limits

An agent can skip a check, so a clean local run does not replace CI, which stays the authority.
