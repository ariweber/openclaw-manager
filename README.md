# openclaw-manager

A skill that teaches an AI agent how to **safely administer an [OpenClaw](https://github.com/ariweber/openclaw) installation** — editing agent workspaces, authoring skills, configuring sub-agents, managing `openclaw.json`, and troubleshooting the gateway service.

OpenClaw is a file-driven AI agent platform that runs as a local Node.js gateway on port `18789`. Every agent is a directory of Markdown files (`SOUL.md`, `IDENTITY.md`, `USER.md`, …), every skill is a directory with a `SKILL.md`, and every tool is a documented capability surfaced by the gateway. Because state lives in files, a careless edit can silently corrupt an agent — this skill encodes the conventions an administrator must follow.

## What this skill does

When loaded, it trains the host agent (Claude Code, an OpenClaw agent itself, or any compatible runtime) to:

- Use the **canonical workspace layout** and never invent, rename, or merge core files.
- Respect **skill loading precedence** (`<workspace>/skills` > `<workspace>/.agents/skills` > `~/.agents/skills` > `~/.openclaw/skills` > bundled).
- Prefer **hot reload over restarts** — most Markdown and config edits are picked up automatically.
- Understand that **session tools are depth-gated** — only orchestrator agents can spawn sub-agents.
- Keep **secrets out of Markdown files** — `auth-profiles.json` and `models.json` are the only right places for keys.

The skill ships with six reference documents covering every area of OpenClaw administration:

| File | Covers |
|------|--------|
| `SKILL.md` | Entry point — pre-flight discovery, decision tree, non-negotiable rules. |
| `references/workspace-structure.md` | The seven core Markdown files and what belongs in each. |
| `references/skills-authoring.md` | Writing, installing, and debugging skills; ClawHub workflow. |
| `references/cli-and-reload.md` | `openclaw` CLI, systemd service, hot-reload semantics, log inspection. |
| `references/agents-and-subagents.md` | Agent config, sub-agent spawning, depth limits, trust boundaries. |
| `references/config-files.md` | `openclaw.json`, `models.json`, `auth-profiles.json` — structure and safe edits. |
| `references/safety-rules.md` | Destructive-op guards, secret handling, tenancy isolation. |

## Installation

### Option 1 — Install into an OpenClaw workspace (recommended)

From the machine where OpenClaw is installed:

```bash
# Download the packaged skill
curl -L -o openclaw-manager.zip \
  https://github.com/ariweber/openclaw-manager/raw/main/openclaw-manager.zip

# Extract into your workspace's skills directory
unzip openclaw-manager.zip -d <workspace>/skills/

# Confirm OpenClaw picked it up
openclaw skills list | grep openclaw-manager
```

Replace `<workspace>` with the workspace path from `openclaw.json` (e.g. `~/.openclaw/workspace` for a single-agent install, or `/home/<user>/workspaces/<name>` for a per-tenant setup).

No restart needed — OpenClaw hot-reloads skill directories. If the skill doesn't appear, run `openclaw skills list` and check the output for reasons it was rejected (usually a `requires` mismatch).

### Option 2 — Install globally for all workspaces on the host

```bash
mkdir -p ~/.openclaw/skills
unzip openclaw-manager.zip -d ~/.openclaw/skills/
```

Any workspace on this host that does not have a higher-precedence copy will load this one.

### Option 3 — Use as a Claude Code skill

If you want Claude Code (the CLI) to load this skill when you work on OpenClaw hosts:

```bash
mkdir -p ~/.claude/skills
unzip openclaw-manager.zip -d ~/.claude/skills/
```

Restart Claude Code. The skill auto-triggers on any mention of OpenClaw, `~/.openclaw`, `SOUL.md`, `openclaw serve`, gateway port `18789`, or their Hebrew equivalents (אופןקלאו, להוסיף סוכן, etc.).

### Option 4 — Install from source (this repo directly)

```bash
git clone https://github.com/ariweber/openclaw-manager.git
cp -r openclaw-manager <workspace>/skills/
```

The repo layout is already a valid skill directory — `SKILL.md` at the root plus a `references/` subdirectory. The loose `.md` files at the repo root are the same reference files; the `references/` layout inside the zip is what OpenClaw expects.

## Verifying it loaded

```bash
openclaw skills list
```

You should see an `openclaw-manager` entry with its source path and an `eligible: true` flag. If it shows `eligible: false`, the output will print the reason (most common: the skill's `requires` clause didn't match your environment).

To see the skill's entry point:

```bash
openclaw skills show openclaw-manager
```

## Using it

Once installed, the skill activates automatically whenever the loaded agent is asked to do OpenClaw administrative work — you do not invoke it by name. Trigger phrases include anything about:

- Editing an agent's personality, identity, or behavior (`SOUL.md`, `IDENTITY.md`, `USER.md`)
- Adding, installing, or debugging skills
- Creating a new agent or sub-agent
- Changing models or rotating auth tokens
- Restarting or diagnosing the gateway
- Hebrew equivalents: להוסיף סוכן, לערוך את הסוכן, לבנות סקיל, תקן את הבוט

The skill's first act in any session is a **pre-flight discovery sequence** — it finds the config path, checks which user owns the running gateway, and lists the workspaces on the host — before making any change.

## Updating

```bash
# From a workspace install
rm -rf <workspace>/skills/openclaw-manager
curl -L https://github.com/ariweber/openclaw-manager/raw/main/openclaw-manager.zip \
  | bsdtar -xvf - -C <workspace>/skills/
```

OpenClaw will hot-reload the new version within a couple of seconds. Confirm with `openclaw skills list`.

## Repo contents

```
.
├── SKILL.md                        # Skill entry point
├── agents-and-subagents.md         # Reference (flat copy)
├── cli-and-reload.md               # Reference (flat copy)
├── config-files.md                 # Reference (flat copy)
├── safety-rules.md                 # Reference (flat copy)
├── skills-authoring.md             # Reference (flat copy)
├── workspace-structure.md          # Reference (flat copy)
└── openclaw-manager.zip            # Packaged skill (SKILL.md + references/)
```

The flat `.md` files at the repo root are for easy reading on GitHub. The `.zip` is the installable bundle — it contains the same files but organised in the layout OpenClaw expects (`SKILL.md` at the top, references under `references/`).

## License

MIT.
