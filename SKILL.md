---
name: openclaw-manager
description: Administer an OpenClaw installation — workspaces, skills, sub-agents, config files, and the gateway service. Use whenever the user wants to inspect, modify, or extend OpenClaw on this host.
metadata: {"skillVersion": "1.2", "lastVerifiedOpenClawVersion": "unset"}
---

# OpenClaw Manager

You are operating on a machine that has **OpenClaw** installed — a file-driven AI agent platform where every agent is a directory of Markdown files, every skill is a directory with a `SKILL.md`, and every tool is a documented capability surfaced by the gateway.

**Common defaults on a typical install** (verify on the live host before acting — none are universal):
- The gateway is a local Node.js process listening on port `18789` (configurable via `openclaw.json` or `--port`).
- Config and state live under `~/.openclaw/`; the master config is `~/.openclaw/openclaw.json` (location can be overridden by the install's env/config).
- On Linux the service usually runs as a user systemd unit named `openclaw-gateway`. On macOS it's typically a launchd agent; on Windows a scheduled task; it also ships as a Docker container and can run in the foreground (`openclaw gateway …`).

Your job with this skill loaded: **act as a careful administrator of this specific OpenClaw installation**. Read before you write. Do not guess file paths, service names, or commands — verify against `openclaw <cmd> --help` and the live filesystem for each action.

## When to activate this skill

Engage admin mode when the user is asking you to **inspect, modify, troubleshoot, or extend** an OpenClaw install — not for every mention of OpenClaw.

Typical triggers:

- Editing a workspace file (`SOUL.md`, `IDENTITY.md`, `USER.md`, `AGENTS.md`, `TOOLS.md`, `BOOTSTRAP.md`, `HEARTBEAT.md`, `MEMORY.md`)
- Editing `openclaw.json`, `models.json`, or `auth-profiles.json`
- Authoring, installing, or debugging a skill (including ClawHub installs)
- Adding/removing agents or sub-agents; changing `maxSpawnDepth`, tool allow/deny lists, channel bindings
- Troubleshooting the gateway — service not responding, channel not binding, reloads not landing, transcripts to read
- Hebrew equivalents: להוסיף סוכן, לערוך את הסוכן, לבנות סקיל, תקן את הבוט, לשנות את האישיות של הסוכן

Do **not** enter full admin mode for pure conceptual questions ("what is OpenClaw?", "what does SOUL.md do in general?"). A direct answer from the references is enough; skip pre-flight.

## The five things to get right

1. **Workspace structure is canonical.** Agents live in per-agent workspace directories with a fixed set of core Markdown files. Do not rename, merge, or invent these files. See `references/workspace-structure.md`.

2. **Skill loading order matters.** OpenClaw loads skills from multiple locations with strict precedence: `<workspace>/skills` > `<workspace>/.agents/skills` > `~/.agents/skills` > `~/.openclaw/skills` > bundled. A skill with the same name in a higher-precedence location silently shadows a lower one. See `references/skills-authoring.md`.

3. **Hot reload is real — restarts are usually wrong.** OpenClaw watches config and workspace files and reloads most changes automatically. Do not run `systemctl restart` or `openclaw gateway --force` unless you have a concrete reason. See `references/cli-and-reload.md`.

4. **Session tools are depth-gated.** Only orchestrator agents (depth 1 with `maxSpawnDepth >= 2`) can spawn sub-agents. Non-spawning and depth-2 agents cannot. Do not assume `sessions_spawn` is available. See `references/agents-and-subagents.md`.

5. **Secrets are everywhere and must stay there.** `openclaw.json`, `auth-profiles.json`, and `models.json` contain API keys and bot tokens. Never print them, never commit them, never include them in artifacts or messages. See `references/safety-rules.md`.

## Pre-flight — adaptive, not ceremonial

Match the depth of discovery to the blast radius of the task.

**Tier 0 — read/explain only.** No discovery. Answer from the references, do not touch the host.
Examples: "what does SOUL.md do?", "explain skill precedence".

**Tier 1 — targeted edit with a known path.** Minimal verification only.
Examples: editing a specific Markdown file the user named, bumping one field in a config the user pointed at.
Before the edit, confirm: the target file exists at the given path; if it's JSON, the current contents parse; if it's a workspace Markdown, the workspace is registered in the install's `openclaw.json`. No service probing, no filesystem-wide scans.

**Tier 2 — runtime, service, install, or "it's broken".** Run the discovery block below before risky changes.
Examples: restarting the gateway, installing a skill, rotating auth, adding an agent, debugging silent bots.

The commands below are **examples for a common Linux / user-systemd install**. Adapt them to the host: launchd on macOS, scheduled task on Windows, `docker` for containerized installs, or `ps` / `openclaw gateway ... --verbose` for foreground runs. Do not assume the exact paths, unit names, or workspace names — verify each time.

```bash
# 1. Version — CLI changes between OpenClaw versions. If anything below disagrees with
#    `openclaw <cmd> --help` on this host, trust the help.
openclaw --version

# 2. Config and state (defaults shown; the install may override via openclaw.json or env).
ls -la ~/.openclaw/ 2>/dev/null
ls -la ~/.agents/ 2>/dev/null

# 3. Service — common default is a user systemd unit named `openclaw-gateway`. Verify:
systemctl --user status openclaw-gateway 2>/dev/null | head -20
systemctl status openclaw-gateway 2>/dev/null | head -20   # system-scope fallback
# On macOS:   launchctl list | grep -i openclaw
# On Windows: schtasks /query /fo LIST | findstr /i openclaw
# In Docker:  docker ps --format '{{.Names}} {{.Image}}' | grep -i openclaw
# Foreground: ps -eo pid,user,cmd | grep -i openclaw | grep -v grep

# 4. Live process (whatever the install mode)
ps -eo pid,user,cmd | grep -i openclaw | grep -v grep

# 5. Workspaces — prefer the config as the source of truth over filesystem guessing.
python3 - <<'PY' 2>/dev/null
import json, os
cfg = os.path.expanduser("~/.openclaw/openclaw.json")
if os.path.exists(cfg):
    d = json.load(open(cfg))
    for name, agent in d.get("agents", {}).items():
        print(name, "→", agent.get("workspace", "<unset>"))
PY
# Fallback only if the config path differs on this install:
find ~ -maxdepth 6 -name "SOUL.md" 2>/dev/null | head
```

Before a risky change, confirm on this host: version, config location, which user owns the running process, and the workspace path of the agent you're about to touch. If any is ambiguous, resolve it first — do not push forward in parallel with the change.

## Version compatibility

`metadata.lastVerifiedOpenClawVersion` in the frontmatter is a soft hint, not an authority:

- **`unset`**: no formal verification on this host. Treat the live CLI and the current `openclaw.json` schema as ground truth — `openclaw <cmd> --help` overrides anything in these references when they disagree.
- **Matches the running major version**: the reference commands can be used as written, but still sanity-check paths and unit names on the host.
- **Differs materially from the running version**: warn the operator before any risky change. Renamed commands and reshaped flags are the most common silent-failure pattern across majors.

Never claim high confidence in a command when this skill's verified version differs from what's installed.

## Decision tree — what are you being asked to do?

| User intent | Go to |
|-------------|-------|
| **"Set up a new bot from scratch"** (customer ask) | **`references/walkthrough.md` — START HERE, Phase 0 is mandatory** |
| "Edit the agent's personality / instructions / identity" | `references/workspace-structure.md` → section on SOUL.md / IDENTITY.md |
| "Add a skill / teach the agent to do X" | `references/skills-authoring.md` |
| "Create a new agent / workspace / sub-agent" | `references/agents-and-subagents.md` |
| "Change the model / add a provider / fix OpenRouter" | `references/config-files.md` |
| "The bot is costing too much / switch to a cheaper model / check cache" | `references/cost-and-models.md` |
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

Be terse. Every open reference and every file read costs tokens on each turn.

- **Read only the specific reference the decision tree points to.** Not the whole set.
- **Verify before edit.** Resolve the actual file path, the actual config schema, and the actual service name on this host first. Assumed defaults are guesses until confirmed.
- **Prefer targeted inspection over filesystem-wide scans.** `cat <known-path>` beats `find ~`; `openclaw agents list` beats grepping for `SOUL.md`.
- **After a write, summarize in compact diff form** — path + what changed + reload status. Do not dump the full file unless asked.
- **Do not restart by reflex.** Most config and all workspace Markdown edits hot-reload. Restart only with a named cause (port change, token rotation, gateway confirmed wedged via logs).
- **When unsure, ask — don't guess a destructive path.** An extra clarifying question is cheaper than a corrupted workspace.
