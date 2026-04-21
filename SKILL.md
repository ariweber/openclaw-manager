---
name: openclaw-manager
description: Use this skill WHENEVER working on a server or machine that has OpenClaw installed and the user wants to inspect, modify, or extend the OpenClaw installation — including editing agent workspaces (SOUL.md, IDENTITY.md, USER.md, BOOTSTRAP.md, AGENTS.md, TOOLS.md, HEARTBEAT.md), authoring or installing skills, configuring sub-agents, modifying `openclaw.json` or `models.json`, troubleshooting the gateway/systemd service, or spawning sessions. Trigger on any mention of OpenClaw, אופןקלאו, "the bot on the server", `~/.openclaw`, `~/.agents`, `<workspace>/skills`, `openclaw.json`, SOUL.md, BOOTSTRAP.md, `openclaw serve`, `openclaw configure`, `openclaw skills`, `/subagents`, `sessions_spawn`, or gateway port 18789. Also trigger when the user asks to "add a tool to the agent", "give the agent a new skill", "edit the agent's personality", "create a sub-agent", "fix the bot", or any Hebrew equivalent (להוסיף סוכן, לערוך את הסוכן, לבנות סקיל, תקן את הבוט). This skill teaches correct OpenClaw conventions — do not guess file locations, reload semantics, or tool names; consult this skill's references first.
---

# OpenClaw Manager

You are operating on a machine that has **OpenClaw** installed. OpenClaw is a file-driven AI agent platform that runs as a local Node.js gateway on port `18789`. Every agent is a directory of Markdown files; every skill is a directory with a `SKILL.md`; every tool is a documented capability surfaced by the gateway.

Your job with this skill loaded: **act as a careful administrator of the OpenClaw installation**. Read before you write. Never invent file paths or command names — OpenClaw has specific conventions and breaking them silently corrupts agents.

## The five things to get right

1. **Workspace structure is canonical.** Agents live in per-agent workspace directories with a fixed set of core Markdown files. Do not rename, merge, or invent these files. See `references/workspace-structure.md`.

2. **Skill loading order matters.** OpenClaw loads skills from multiple locations with strict precedence: `<workspace>/skills` > `<workspace>/.agents/skills` > `~/.agents/skills` > `~/.openclaw/skills` > bundled. A skill with the same name in a higher-precedence location silently shadows a lower one. See `references/skills-authoring.md`.

3. **Hot reload is real — restarts are usually wrong.** OpenClaw watches config and workspace files and reloads most changes automatically. Do not run `systemctl restart` or `openclaw gateway --force` unless you have a concrete reason. See `references/cli-and-reload.md`.

4. **Session tools are depth-gated.** Only orchestrator agents (depth 1 with `maxSpawnDepth >= 2`) can spawn sub-agents. Leaf agents cannot. Do not assume `sessions_spawn` is available. See `references/agents-and-subagents.md`.

5. **Secrets are everywhere and must stay there.** `openclaw.json`, `auth-profiles.json`, and `models.json` contain API keys and bot tokens. Never print them, never commit them, never include them in artifacts or messages. See `references/safety-rules.md`.

## Before any OpenClaw action — the mandatory pre-flight

Run this discovery sequence the first time in a session and whenever you're unsure what you're looking at:

```bash
# 1. Find the OpenClaw config and state directories
ls -la ~/.openclaw/ 2>/dev/null
ls -la ~/.agents/ 2>/dev/null
echo "OPENCLAW_CONFIG_PATH=$OPENCLAW_CONFIG_PATH"
echo "OPENCLAW_STATE_DIR=$OPENCLAW_STATE_DIR"

# 2. Check the service (try user scope first — OpenClaw installs as a user service by default)
systemctl --user status openclaw-gateway 2>/dev/null | head -20
# Fall back to system scope only if user scope has nothing
systemctl status openclaw-gateway 2>/dev/null | head -20

# 3. See what's actually running
ps -eo pid,user,cmd | grep -i openclaw | grep -v grep

# 4. List the workspaces on this machine
find ~ -maxdepth 5 -name "workspace-*" -type d 2>/dev/null
find ~ -maxdepth 5 -name "SOUL.md" 2>/dev/null | head -20
```

Do not proceed until you know (a) where the config lives, (b) which user owns the running process, (c) which workspaces exist.

## Decision tree — what are you being asked to do?

| User intent | Go to |
|-------------|-------|
| "Edit the agent's personality / instructions / identity" | `references/workspace-structure.md` → section on SOUL.md / IDENTITY.md |
| "Add a skill / teach the agent to do X" | `references/skills-authoring.md` |
| "Create a new agent / workspace / sub-agent" | `references/agents-and-subagents.md` |
| "Change the model / add a provider / fix OpenRouter" | `references/config-files.md` |
| "Restart the bot / fix the gateway / service won't start" | `references/cli-and-reload.md` |
| "The bot leaked data / said the wrong thing / has wrong permissions" | `references/safety-rules.md` + `references/agents-and-subagents.md` |
| "Install a community skill from ClawHub" | `references/skills-authoring.md` → Installing |
| "Debug — find out what the agent did / read the transcript" | `references/cli-and-reload.md` → Sessions & logs |

## Non-negotiable rules

- **Never `rm -rf` anything under `~/.openclaw/`, `~/.agents/`, or a workspace directory** without first moving it to a backup path (`mv foo foo.bak.$(date +%s)`). OpenClaw state is irreplaceable — sessions, memory, and skill installs all live there.
- **Never edit `openclaw.json` in place without a backup.** Always `cp openclaw.json openclaw.json.bak.$(date +%s)` first.
- **Never put secrets into a SKILL.md, SOUL.md, or any Markdown file.** Secrets belong in `auth-profiles.json` (chmod 600) or in `skills.entries.*.env` / `.apiKey` in `openclaw.json`.
- **Never commit `~/.openclaw/` or a workspace to git without a `.gitignore` that excludes `auth-profiles.json`, `*.token`, `sessions/`, and `.env`.**
- **Never claim a restart is needed without testing hot reload first.** Most config and all workspace Markdown edits are picked up automatically.

## Working style

Be terse. OpenClaw sessions are token-budgeted — the user pays for every line you write and every file you read. Follow the pattern the OpenClaw docs themselves recommend: read the specific reference you need, not the whole set; use scripts over prose for repetitive discovery; return `{success, path}` style minimal summaries after write operations.

When you're asked to make a change, state what you're about to change, do it, then show a three-line diff-style summary — not the whole file.
