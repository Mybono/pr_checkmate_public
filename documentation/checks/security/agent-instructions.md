# Agent Instructions

<!-- cspell:ignore windsurfrules -->

[Docs](../../README.md) · [Checks Index](../INDEX.md) · [Diff Security](diff-security.md) · [Dockerfile Security](dockerfile-security.md) · [Migration Safety](migration-safety.md) · [Security Scan](security-scan.md) · [Sensitive Files](sensitive-files.md) · [Workflow Security](workflow-security.md) · **Agent Instructions**

---

## Overview

Scans the files that tell an AI coding agent what to do, or what it is allowed to do, for
instructions that were put there to be missed by the person reviewing them.

These files are code in every sense that matters. An agent reads them and acts, with the
developer's shell and the developer's credentials. A pull request that edits one changes what
every contributor's agent does on its next run — and the diff is prose, which review tends to skim.
That makes it the cheapest place in a repository to plant an instruction, and the least reviewed.

Matched files:

```text
AGENTS.md · CLAUDE.md · GEMINI.md (any directory) · .cursorrules · .windsurfrules · .cursor/rules/**
.github/copilot-instructions.md · .claude/commands/** · .claude/agents/** · .claude/skills/**
.claude/settings.json · .mcp.json
```

### What it looks for

| Finding | Level | Why it matters |
|---|---|---|
| Invisible Unicode characters (zero-width, bidirectional overrides, the tag block) | `error` | A model reads them; a reviewer cannot see them. Each has been used to hide an instruction in text that looks harmless |
| Telling the agent to hide something from the user | `error` | An instruction that depends on the human not knowing is not a convention |
| Sending credentials or secrets to a URL | `error` | Exfiltration, spelled out in a file the agent will obey |
| `"defaultMode": "bypassPermissions"` in project settings | `error` | Every contributor's agent would run without asking |
| "Ignore previous instructions" phrasing | `warn` | The standard prompt-injection opener. A warning only: a file that *discusses* injection says it too |
| `curl … \| sh` in an instruction | `warn` | The agent will run what it is told to run |
| `--no-verify`, `--dangerously-skip-permissions` | `warn` | Tells the agent to skip a safety check |
| `"enableAllProjectMcpServers": true` | `warn` | Auto-approves every MCP server the repository declares |
| `Bash(*)` / `Write(*)` permission | `warn` | Permission for every command or file write |
| MCP server over plain HTTP (not localhost) | `warn` | Tool traffic readable and alterable on the network |
| MCP server fetched with `npx -y` and no pinned version | `warn` | The next publish runs on every contributor's machine |

A byte-order mark is not reported: editors put one at the top of ordinary files.

| Property | Value |
|---|---|
| Display name | `Agent Instructions` |
| Phase | `informational` |
| CLI command | `npx pr-checkmate agent-instructions` |
| Config key | `agentInstructions` |
| Source | `src/core/checks/security/agent-instructions.ts` |

## When it applies

`agentInstructions.enabled` is not `false`. No diff range is required: a hidden instruction is as
live in a file that arrived last year as in one added today, so a whole-repository run reads these
files too. Files are read as written — a leading `#` is a heading here, not a comment.

## Configuration

| Key | Type | Default | Meaning |
|---|---|---|---|
| `agentInstructions.enabled` | boolean | `true` | Set `false` to skip the check |
| `agentInstructions.ignore` | string[] | `[]` | Finding **labels** to mute, matched as case-insensitive substrings |
| `agentInstructions.ignorePaths` | string[] | `[]` | Files to exclude, as globs |
| `agentInstructions.failOn` | `error` \| `warn` \| `never` | `error` | What a finding rated `error` does to the run |
| `agentInstructions.severityOverrides` | object | `{}` | Re-level one finding by label substring; `"off"` drops it |

### Examples

A repository that documents prompt injection and quotes the phrase:

```json
{ "agentInstructions": { "ignorePaths": ["docs/security/prompt-injection.md"] } }
```

Accept an unpinned MCP server while keeping every other rule:

```json
{ "agentInstructions": { "ignore": ["no pinned version"] } }
```

## Disabling

```json
{ "agentInstructions": { "enabled": false } }
```

Or make every finding blocking:

```json
{ "severity": { "Agent Instructions": "error" } }
```
