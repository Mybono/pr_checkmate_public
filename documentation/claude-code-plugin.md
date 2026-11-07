# The Claude Code plugin

<!-- docs-nav -->
[Docs](README.md) · [Config guide](config-guide.md) · [Check reference](checks/INDEX.md) · [How checks work](checks/concepts.md) · [Running in CI](ci-setup.md) · [Migrating](migrating.md) · [Architecture](architecture-and-approach.md) · [Authoring a check](authoring-checks.md) · [AI agents](ai-agents.md) · **Claude Code plugin**

---

The plugin is the shortest way to give Claude Code the gate. One command installs two things: the
[MCP server](ai-agents.md) and a skill that tells the agent when to use it. It adds nothing to
your `package.json`.

```text
/plugin install pr-checkmate --marketplace Mybono/pr_checkmate_public
```

Needs Claude Code 2.1.275 or later for the one-step form. On older versions, add the marketplace
first with `/plugin marketplace add Mybono/pr_checkmate_public`, then run
`/plugin install pr-checkmate@pr-checkmate-marketplace`.

## What you get

**The MCP server.** The plugin starts it with `npx -y pr-checkmate@<version> mcp`. The version is
fixed in each release of the plugin, so an update of the plugin is the only thing that changes
which code runs. The first start downloads the package, which can take a few seconds. The tools
are the ones described in [AI agents](ai-agents.md#the-tools).

**The skill.** `/pr-checkmate:pr-checkmate`, and the agent also picks it up by itself when you
start work in a repository. It does three things:

1. Finds out what is available: the MCP tools, the CLI in the project, or nothing.
2. If the package is missing, explains what it is and asks before installing it. It never
   installs on its own. Setting up a repository, `suggest_config` and then `write_config`, also
   waits for your answer.
3. Tells the agent which tool fits which moment: `get_rules` before writing, `check_changes`
   after a piece of work, `scan_secrets` before touching credentials, `check_dependency` before
   adding a package, `verify` once at the end. It does not choose checks by language, because the
   package already detects languages.

## What Claude Code asks you

Claude Code asks before the first call of each tool, naming the plugin and the server, for example
`plugin:pr-checkmate:mcp — Check Changes Tool`. Answer once per tool, or allow the whole server
in `permissions.allow`. Plugin tools carry the plugin name in their identifier, so the rule is
different from the one for a server added with `claude mcp add`:

```json
{ "permissions": { "allow": ["mcp__plugin_pr-checkmate_mcp"] } }
```

The plugin does not install the commit hook. That stays in `agent-setup`, where you agree to it
on its own.

## Plugin or `agent-setup`?

| | Plugin | `claude mcp add` and `agent-setup` |
|---|---|---|
| Agents | Claude Code | Claude Code, Cursor, any MCP client |
| Install | one command | two commands, once and per repository |
| Package in your project | not needed to start | needed for `agent-setup` |
| Instructions in `AGENTS.md` or `CLAUDE.md` | no, the skill carries them | yes |
| Commit gate hook | no | yes |
| Updates | `/plugin update`, or automatic if you turn it on | `npm update` |

Use the plugin for the quickest start in Claude Code. Use `agent-setup` for other agents, or when
you want the commit gate. Do not use both in one project: the tools would be listed twice.

## Without the plugin

The skill is a single file, `skills/pr-checkmate/SKILL.md`. Any agent that follows the `SKILL.md`
standard and can run shell commands can use it, with the CLI instead of the MCP tools. The
git-commit hook and the instructions block are specific to Claude Code.

## Updating and removing

```text
/plugin update pr-checkmate@pr-checkmate-marketplace
/plugin uninstall pr-checkmate@pr-checkmate-marketplace
```

Removing the plugin removes the server and the skill. A `pr-checkmate.json` or workflow that the
skill helped you create stays in your repository.
