---
name: openclaw-manager
description: Administer an OpenClaw installation — workspaces, skills, sub-agents, config files, and the gateway service. Use whenever the user wants to inspect, modify, or extend OpenClaw on this host.
metadata: {"skillVersion": "1.1", "lastVerifiedOpenClawVersion": "unset"}
---

# OpenClaw Manager

You are operating on a machine that has **OpenClaw** installed. OpenClaw is a file-driven AI agent platform that runs as a local Node.js gateway on port `18789`. Every agent is a directory of Markdown files; every skill is a directory with a `SKILL.md`; every tool is a documented capability surfaced by the gateway.

Your job with this skill loaded: **act as a careful administrator of the OpenClaw installation**. Read before you write. Never invent file paths or command names — OpenClaw has specific conventions and breaking them silently corrupts agents.

## Trigger phrases

Activate this skill when the conversation mentions any of:

- OpenClaw, אופןקלאו, "the bot on the server"
- Paths: `~/.openclaw`, `~/.agents`, `<workspace>/skills`, `openclaw.json`, `models.json`
- Workspace files: `SOUL.md`, `IDENTITY.md`, `USER.md`, `BOOTSTRAP.md`, `AGENTS.md`, `TOOLS.md`, `HEARTBEAT.md`
- Commands: `openclaw serve`, `openclaw configure`, `openclaw skills`, `openclaw doctor`, `/subagents`, `sessions_spawn`
- Gateway port `18789`
- Intents: "add a tool to the agent", "give the agent a new skill", "edit the agent's personality", "create a sub-agent", "fix the bot"
- Hebrew equivalents: להוסיף סוכן, לערוך את הסוכן, לבנות סקיל, תקן את הבוט, לשנות את האישיות של הסוכן

## The five things to get right

1. **Workspace structure is canonical.** Agents live in per-agent workspace directories with a fixed set of core Markdown files. Do not rename, merge, or invent these files. See `references/workspace-structure.md`.

2. **Skill loading order matters.** OpenClaw loads skills from multiple locations with strict precedence: `<workspace>/skills` > `<workspace>/.agents/skills` > `~/.agents/skills` > `~/.openclaw/skills` > bundled. A skill with the same name in a higher-precedence location silently shadows a lower one. See `references/skills-authoring.md`.

3. **Hot reload is real — restarts are usually wrong.** OpenClaw watches config and workspace files and reloads most changes automatically. Do not run `systemctl restart` or `openclaw gateway --force` unless you have a concrete reason. See `references/cli-and-reload.md`.

4. **Session tools are depth-gated.** Only orchestrator agents (depth 1 with `maxSpawnDepth >= 2`) can spawn sub-agents. Leaf agents cannot. Do not assume `sessions_spawn` is available. See `references/agents-and-subagents.md`.

5. **Secrets are everywhere and must stay there.** `openclaw.json`, `auth-profiles.json`, and `models.json` contain API keys and bot tokens. Never print them, never commit them, never include them in artifacts or messages. See `references/safety-rules.md`.

## Before any OpenClaw action — the mandatory pre-flight

Run this discovery sequence the first time in a session and whenever you're unsure what you're looking at. **Skip it only for pure read/explain questions** ("what does SOUL.md do?") — anything that will write, restart, or install requires the full sequence.

```bash
# 1. Which OpenClaw version — commands and flags change between versions.
#    Compare against this skill's frontmatter metadata.lastVerifiedOpenClawVersion.
#    If anything below contradicts `openclaw <cmd> --help`, trust the help.
openclaw --version

# 2. Find the OpenClaw config and state directories
ls -la ~/.openclaw/ 2>/dev/null
ls -la ~/.agents/ 2>/dev/null
echo "OPENCLAW_CONFIG_PATH=$OPENCLAW_CONFIG_PATH"
echo "OPENCLAW_STATE_DIR=$OPENCLAW_STATE_DIR"

# 3. Check the service (try user scope first — OpenClaw installs as a user service by default)
systemctl --user status openclaw-gateway 2>/dev/null | head -20
# Fall back to system scope only if user scope has nothing
systemctl status openclaw-gateway 2>/dev/null | head -20

# 4. See what's actually running
ps -eo pid,user,cmd | grep -i openclaw | grep -v grep

# 5. List the workspaces on this machine
find ~ -maxdepth 5 -name "workspace-*" -type d 2>/dev/null
find ~ -maxdepth 5 -name "SOUL.md" 2>/dev/null | head -20
```

Do not proceed until you know (a) the OpenClaw version, (b) where the config lives, (c) which user owns the running process, (d) which workspaces exist.

## Version compatibility

This skill has a `metadata.lastVerifiedOpenClawVersion` field in its frontmatter. Treat it as a soft contract:

- If `lastVerifiedOpenClawVersion` is `unset`, no one has formally tested this skill against a specific OpenClaw build on your host. Proceed, but prefer `openclaw <cmd> --help` over the commands in these references whenever they disagree.
- If the running version's major component matches `lastVerifiedOpenClawVersion`, you can trust the reference commands verbatim.
- If the major version differs (e.g., running `2.x` against a skill verified on `1.x`), flag this to the operator before making changes. CLI renames and flag changes between majors are the most common source of silent failure.

**Maintainer note**: after you test this skill against a specific OpenClaw version and confirm all referenced commands work, update the frontmatter to that version.

## Decision tree — what are you being asked to do?

| User intent | Go to |
|-------------|-------|
| "Edit the agent's personality / instructions / identity" | `references/workspace-structure.md` → section on SOUL.md / IDENTITY.md |
| "Add a skill / teach the agent to do X" | `references/skills-authoring.md` |
| "Create a new agent / workspace / sub-agent" | `references/agents-and-subagents.md` |
| "Change the model / add a provider / fix OpenRouter" | `references/config-files.md` |
| "Restart the bot / fix the gateway / service won't start" | `references/cli-and-reload.md` |
| "The bot leaked data / said the wrong thing / has wrong permissions" | `references/safety-rules.md` + `references/agents-and-subagents.md` |
| "The agent sends weird messages after finishing a task" | `references/agents-and-subagents.md` → NO_REPLY convention |
| "Install a community skill from ClawHub" | `references/skills-authoring.md` → Installing |
| "Debug — find out what the agent did / read the transcript" | `references/cli-and-reload.md` → Sessions & logs |

## Non-negotiable rules

The full rationale and exact procedures live in `references/safety-rules.md`. Do not cross these lines:

- **Backup before editing state.** `openclaw.json`, `models.json`, any workspace Markdown, any skill directory — timestamped backup first. No exceptions.
- **No secrets in Markdown.** They go in `auth-profiles.json` (chmod 600) or env vars referenced by name from `openclaw.json`.
- **No `rm -rf` inside `~/.openclaw/`, `~/.agents/`, or a workspace.** Rename to `.deleted-$(date +%s)` and let it sit for a week first.
- **No `git init` without a `.gitignore`** that excludes `auth-profiles.json`, `*.token`, `sessions/`, `.env`.
- **No restart reflex.** Try hot reload first. Restart only with a concrete reason — see `references/cli-and-reload.md`.

When in doubt on any of these, read `safety-rules.md`. Do not paraphrase from memory.

## Working style

Be terse. OpenClaw sessions are token-budgeted — the user pays for every line you write and every file you read. Follow the pattern the OpenClaw docs themselves recommend: read the specific reference you need, not the whole set; use scripts over prose for repetitive discovery; return `{success, path}` style minimal summaries after write operations.

When you're asked to make a change, state what you're about to change, do it, then show a three-line diff-style summary — not the whole file.
