# Using PR CheckMate from an AI agent

<!-- docs-nav -->
[Docs](README.md) · [Config guide](config-guide.md) · [Check reference](checks/INDEX.md) · [How checks work](checks/concepts.md) · [Running in CI](ci-setup.md) · [Migrating](migrating.md) · [Architecture](architecture-and-approach.md) · [Authoring a check](authoring-checks.md) · **AI agents**

---

CI tells you a pull request is wrong after the code is written. An agent can hear it
before. PR CheckMate ships an [MCP](https://modelcontextprotocol.io) server that gives a
coding agent the same gate CI applies, in two moments:

- **before writing** — `get_rules` says what this repository enforces, so the first draft
  already conforms;
- **while and after writing** — `check_changes` and `scan_secrets` run the checks on what
  the agent has changed so far, and `verify` runs the full gate once at the end, so it fixes
  findings in its own loop instead of in a review.

Two tools write files, `apply_fixes` and `write_config`; every other tool only reads. Nothing is staged, committed or posted.

## Install once, use everywhere

```bash
npm install -g pr-checkmate
claude mcp add --scope user pr-checkmate -- pr-checkmate mcp
```

The server is `pr-checkmate mcp`, a standard stdio MCP server, so any MCP client can run it
the same way. It reviews the repository the client was started in; a tool call can also
pass `cwd` to name another one.

### After installing

1. **Check the connection.** `claude mcp list` shows `pr-checkmate ✔ Connected`. Sessions that were
   already open do not pick the server up; start a new one.
2. **`claude: command not found`.** `claude` is the Claude Code CLI. The editor extension ships a
   copy that is not on your `PATH`; either install the CLI or run the extension's binary by its
   full path (it sits under `~/.vscode/extensions/anthropic.claude-code-*/resources/native-binary/`,
   and the path changes with the extension version).
3. **Permission prompts.** Claude Code asks before each MCP call unless the server is allowed.
   Add it to `permissions.allow` in `~/.claude/settings.json`:

   ```json
   "permissions": { "allow": ["mcp__pr-checkmate"] }
   ```

   That allows every tool, including the two that write files. To keep the agent read-only set
   `"mcp": { "readOnly": true }` in the project's `pr-checkmate.json`, or allow the tools one by
   one (`mcp__pr-checkmate__check_changes`, and so on).
4. **No guarantee.** The agent is told when to call each tool, and it can still skip them. For a
   gate it cannot skip, keep the CI check, or a Claude Code Stop hook that runs `pr-checkmate all`.

## Make a repository use it by default

```bash
pr-checkmate agent-setup
```

If the CLI is not on `PATH`, the server is registered through `npx -y pr-checkmate@<version> mcp`
(pinned to the version that wrote it, so a later publish does not run on every contributor's machine) instead, so the entry works either way, and the commit gate is skipped with a note: through
`npx` the hook would resolve the package on every command the agent runs. Install globally
and run `agent-setup` again to add it.

Until a project registers the server, the summary printed by `all` ends with an `AI AGENT:` line
naming the command. It is not printed in CI and goes away with `reporter.include.docsLink: false`.
The check looks only at the project's `.mcp.json`, so a server added at user level still gets the
line. `init` also adds `npx pr-checkmate agent-setup` to the `commands` block of an existing
`pr-checkmate.json`, and touches nothing else in it.

## Turn it off or narrow it

The `mcp` block of `pr-checkmate.json` decides what the server offers an agent working in that
project. It is read on every request, so an edit applies to the agent's next call without
restarting the server.

```json
"mcp": {
  "enabled": true,
  "disabledTools": [],
  "readOnly": false
}
```

| Key | Effect |
| --- | --- |
| `enabled: false` | The server still connects but offers no tools, resources or prompts, and tells the agent it is off. The `AI AGENT:` line in the run summary stops appearing. |
| `disabledTools` | Hides the named tools: `get_rules`, `check_changes`, `apply_fixes`, `scan_secrets`, `check_dependency`, `suggest_config`, `write_config`, `explain_config`, `verify`. The agent is told which ones it does not have. Unknown names are ignored. |
| `readOnly: true` | Hides `apply_fixes` and `write_config`, the two tools that write files. |

A `pr-checkmate.json` that is not valid JSON counts as no settings, so a typo never switches the
server off. The schema offers the tool names in an editor.

Registering a server only makes the tools exist. An agent that was never told to use them
mostly does not, so `agent-setup` adds three things, each covering a gap in the one before:

| Writes | What it does |
|---|---|
| `.mcp.json` | Registers the server for everyone who opens the repository |
| `AGENTS.md` / `CLAUDE.md` | A short block telling the agent *when* to call each tool. Both files are updated if both exist; `AGENTS.md` is created if neither does |
| `.claude/settings.json` | Claude Code hooks: the gate on `git commit`, and a quick security check after each file edit |

It only adds. An existing server entry is left as you wrote it, the instructions block is
refreshed in place between its markers, and a settings file it cannot parse is reported
instead of overwritten. Run it again any time; `--no-hook` skips the commit gate.

## The tools

| Tool | Use it |
|---|---|
| `get_rules` | Before writing code. Lists what blocks a pull request, what only warns, and the repository's own custom rules and banned imports — read from the live config, so it cannot drift from CI |
| `check_changes` | While working and before committing. Fast checks; returns the findings — fix every ❌ and call it again |
| `scan_secrets` | Before committing configuration, environment or credential files |
| `apply_fixes` | Instead of fixing style by hand. Applies Prettier and `eslint --fix` to the files the agent changed — and only those — then re-checks. The one tool that edits files |
| `check_dependency` | Before adding a package. Does it exist, does that version, known advisories (malware included), how new and how used, install scripts, look-alike names |
| `verify` | Once, when the work is believed done. Every check `all` runs, the slow ones included, on the agent's changes |
| `suggest_config` | Setting pr-checkmate up for a repository. Proposes a `pr-checkmate.json` from its languages and a whole-repository baseline. Writes nothing |
| `write_config` | After the user agrees. Validates, then merges: a value the user already set is never replaced |
| `explain_config` | Answering "what does this setting do". Reads the schema shipped with this version |

The same rules are available as the `pr-checkmate://rules` resource.

### Setting up pr-checkmate for a repository

Ask the agent to set it up, or run the `setup_pr_checkmate` prompt. It follows the adoption
path the docs recommend — a gate that is real from the first commit, without a first run that
is red for reasons nobody on the pull request caused:

1. `suggest_config` detects the languages, runs the fast checks across the whole repository and
   proposes a config: checks that **already fail** become warnings so existing debt does not
   block every PR; generated directories (`generated`, `vendor`…) are excluded; everything else
   keeps its defaults. A failing **Merge Conflict** or **Security Scan** is never proposed as a
   warning — a conflict marker is a file that does not run and a leaked secret is already
   exposed, so those are reported as things to fix.
2. The agent shows the proposal and waits.
3. `write_config` validates it first (an unknown key, a misspelt check name or a bad value is
   refused with a "did you mean"), then merges over `pr-checkmate.json`. A value the user already
   set is kept and listed; `overwrite: true` replaces it and only when the user asked.

Promote each warning back to `"error"` as its check reaches zero.

## What `check_changes` looks at

| `scope` | Reviews |
|---|---|
| `worktree` (default) | Every uncommitted change — staged, unstaged and new files. Nothing has to be staged |
| `staged` | What `git commit` would record |
| `branch` | Commits since the branch left the default branch |
| `full` | Every tracked file |

`worktree` is built from a scratch copy of the index, so the agent's staging area is left
exactly as it was and `.gitignore` is honoured.

By default only the fast checks run: nothing heavy and nothing that waits on a registry.
Pass `deep: true` — or call `verify` — for the rest: typecheck, dependency audit, Grype. Pass `checks: ["lint", "security"]` to run specific ones, by
command name or by the name the report shows.

## What the agent hears, and when

Findings carry a file and a line wherever the check knows one — `a.js:2  debugger statement` —
so the agent fixes the line instead of hunting for it. That covers the pattern checks, secrets
(located but never quoted), ESLint (file, line and rule), TypeScript (file, line and `TS` code)
and Prettier (the unformatted files). Other checks report a count or a short reason.
`pr-checkmate <check> --sarif` writes the same findings as SARIF 2.1.0. A check without locations
still gets one result with no location; GitHub code scanning shows only results that have one, so
those appear in editors and other viewers but not as alerts.

With the hooks installed the agent also hears about a file the moment it writes it: a quick,
pattern-only security review of that one file (SQL built by concatenation, `eval`, a conflict
marker, a banned import). Style and debug leftovers wait for `check_changes`, where they would
actually be a mistake; secrets wait for the commit gate, because the scanner starts a process.

## Prompts

The server also offers ready-made requests — slash commands in Claude Code:

| Prompt | Does |
|---|---|
| `prepare_pr` | `apply_fixes`, `verify`, fix what fails, report what is left |
| `fix_findings` | Security first, then every failed check, then small warnings |
| `explain_finding` | What a finding protects against and the smallest fix |
| `setup_pr_checkmate` | The guided setup above |

## The commit gate

An MCP tool is a request; a hook is a fact. When the agent runs `git commit`, the hook
reviews what is about to be recorded and, on a failure, blocks the commit and hands the
findings back as the reason. `git commit -a` is reviewed as the working tree, since it
stages as it commits.

The hook **fails open**. If PR CheckMate itself cannot run — a broken install, a payload it
does not understand — the commit goes through and the reason goes to stderr. A gate that
cannot run must not become the reason the agent cannot work.

## What this does and does not promise

Installing this makes conforming code the likely outcome. It does not make
`npx pr-checkmate all` find nothing, and it is worth knowing why:

- **The agent decides whether to call a tool.** The instructions block and the commit hook
  push hard, but an agent can still skip a step; `--no-verify` is not something a hook can
  stop in every client. The CI gate remains the authority.
- **`check_changes` is the fast subset.** It leaves out the heavy and network checks, which
  `all` runs. `verify` closes that gap — which is why the instructions ask for it at the end.
- **Scope differs.** The agent's tools review what it changed. A whole-repository run also
  reports problems that were there before the agent arrived.
- **`get_rules` names the checks, not every threshold.** The agent learns exact limits —
  a line length, a complexity ceiling — from findings, not before writing.
- **Findings are summaries**, as in CI: a check name and what it found, not file-and-line
  diagnostics. Running the named check on its own gives the detail.
- **First run.** A repository's first run can spend time fetching tooling on demand. Run
  `pr-checkmate install` once to have it ready.

The server runs with your own rights, like the CLI. The protocol is implemented in this
package, with no MCP SDK and no HTTP framework behind it.

## Does it work? Measure it

`scripts/agent-eval/run.js` runs the same agent on the same task twice, plain and with this
server installed, and counts the checks that fail afterwards and did not before the agent
started — old debt is not charged to it. Alternating arm order and `--runs` keep noise down.
The harness has been exercised against a stand-in agent that writes bad code in one arm and
clean code in the other, to show it can tell them apart. It has since been run against a real
model, and **there is still no effect to quote**: the first 64 runs were invalid (the agent also
loaded the user's own CLAUDE.md and Stop hook, so the plain arm was already gated, and the slow
checks did not run), and the clean pilot that followed has 12 runs, too few to separate an effect
from noise. The agent is now started with `--setting-sources project` so only the project's own
files reach it. See ticket 092 for the plan and the numbers.
