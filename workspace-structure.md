# OpenClaw Workspace Structure

Every OpenClaw agent is a directory. The directory name is the workspace name. Inside it are Markdown files (core agent definition) and subdirectories (skills, state, sessions).

## Canonical workspace layout

```
<workspace-root>/
├── BOOTSTRAP.md          # One-shot startup script. Deleted after first run.
├── IDENTITY.md           # Name, persona, tone, emoji style. Stable across sessions.
├── SOUL.md               # System prompt / behavioral rules. The "constitution".
├── USER.md               # Who the agent is serving. Phone numbers, roles, preferences.
├── AGENTS.md             # Rules for working INSIDE this workspace (file conventions, co-agents).
├── TOOLS.md              # Environment notes: SSH targets, cameras, external APIs available.
├── HEARTBEAT.md          # Periodic tasks. Cron-like but natural-language.
├── TASKS.json            # (optional) Structured task queue, shared across sub-agents.
├── SHARED_KNOWLEDGE.json # (optional) Facts the agent has learned and wants to persist.
├── skills/               # Per-workspace skills (highest precedence)
│   └── <skill-name>/
│       └── SKILL.md
├── .agents/
│   └── skills/           # Project-level skills, apply before workspace skills
├── sessions/             # Transcripts and state. NEVER hand-edit. NEVER delete casually.
├── inboxes/              # (optional multi-agent pattern) messages arriving to this agent
├── outboxes/             # (optional multi-agent pattern) messages sent by this agent
└── memory/               # (optional) long-term memory files the agent curates itself
```

## The seven core Markdown files — what goes where

Strict separation matters. Putting persona in SOUL.md or rules in IDENTITY.md makes the agent confused. Respect the contract:

### BOOTSTRAP.md
- **Purpose**: A one-time script the agent runs on its very first turn. Used to fetch secrets from a vault, announce itself, register with a central control plane, etc.
- **Lifecycle**: The agent is expected to delete this file after successfully executing it. If you see a BOOTSTRAP.md in a long-running workspace, something went wrong on first start — investigate before deleting.
- **Edit when**: Setting up a new workspace, rotating credentials that need to be re-fetched, or bootstrapping a restored workspace.

### IDENTITY.md
- **Purpose**: Who the agent *is*. Name, voice, emoji usage, greeting style, language register. Short — usually under 40 lines.
- **Does NOT contain**: Capabilities, rules, prohibitions, tool descriptions. Those go in SOUL.md.
- **Edit when**: Renaming the agent, changing its language default (e.g., Hebrew → English), adjusting its tone.

### SOUL.md
- **Purpose**: The system prompt. Behavioral rules, prohibitions, decision trees, output format contracts. This is the longest and most important file.
- **Structure convention**: Most production SOUL.md files have sections like `# תפקיד` / `# Role`, `# פורמט תשובה` / `# Response Format`, `# מחיקות — חסום` / `# Destructive Ops — Blocked`, `# הרשאות` / `# Permissions`.
- **Edit when**: Changing what the agent is allowed to do, fixing a behavioral bug (agent leaked data, agent deleted something it shouldn't have, agent replied in the wrong format).
- **Critical**: Never put API keys, tokens, or connection strings here. This file is read by the LLM every turn — secrets in SOUL.md are secrets in every prompt.

### USER.md
- **Purpose**: Who the agent serves. Names, phone numbers, roles (`manager`, `staff`, `read-only`), relationships.
- **Format convention**: Usually a table or a list of entries with `{name, phone, role, notes}`.
- **Edit when**: Adding an employee who should be able to command the agent, changing role assignments, removing access.
- **Security**: This file IS the access-control policy for the agent. Treat it as code, not config. Review changes like a PR.

### AGENTS.md
- **Purpose**: Rules for how the agent behaves with *other files in this workspace*. "Always append to CHANGELOG before editing", "never touch sessions/", "sub-agent outputs go in outboxes/subagent-name.md".
- **Edit when**: Adding new workspace conventions, onboarding a sub-agent into the workspace.

### TOOLS.md
- **Purpose**: Environment documentation. "The camera is at rtsp://...", "SSH to the NAS with key ~/.ssh/nas", "the Supabase URL is `https://xxx.supabase.co`". Describes *what's available*, not *what to do with it*.
- **Edit when**: Adding infrastructure (new server, new API endpoint), removing decommissioned infra.
- **Security**: URLs and hostnames are fine here. Keys and tokens are not.

### HEARTBEAT.md
- **Purpose**: Periodic/scheduled behavior. "Every morning at 07:00 summarize yesterday's tasks", "if no activity for 24h, send a check-in".
- **Edit when**: Adding/removing scheduled behavior.

## Multiple agents on one machine

Each agent has its own workspace. The `openclaw.json` field `agents.<name>.workspace` points to it. Common patterns on a single host:

```
~/.openclaw/workspace/                        # default workspace (single-agent setup)
/home/user/workspaces/ansh-cars-manager/      # per-tenant workspace
/home/user/workspaces/ansh-cars-bot/          # second agent for same tenant
/home/user/workspaces/other-tenant-manager/   # isolated workspace for another customer
```

**Never share a workspace between agents that have different trust levels.** OpenClaw's security model treats a workspace as a single trust boundary. A sub-agent that shares a workspace with its parent can read every file the parent can read.

## Sessions and state — hands off

Two directories you should almost never edit by hand:

- **`<workspace>/sessions/`** — transcript storage. OpenClaw manages it. If you need to inspect, read files; do not modify.
- **`~/.openclaw/state/`** and **`~/.openclaw/agents/<name>/sessions/`** — active session state. Corrupting this breaks live conversations.

If a session is misbehaving, the correct fix is almost always to *end the session* via the gateway, not to touch the files. See `cli-and-reload.md`.

## Editing checklist — before you save any workspace file

1. Backup: `cp FILE FILE.bak.$(date +%s)` (OpenClaw does not version these files).
2. Read the whole file — do not skim. Many workspaces have inline `<!-- -->` comments with edit conventions.
3. If the file has a frontmatter section (some do), preserve it exactly.
4. After save: if the agent is running, watch for the hot-reload line in logs (`journalctl --user -u openclaw-gateway -f | grep -i reload`). Most Markdown edits reload in under 2 seconds.
5. Test with a minimal prompt in the channel (Telegram/WhatsApp/CLI) — never assume the reload "probably worked".

## Common mistakes to avoid

- **Inventing a file** because it "seems like it should exist". If `HEARTBEAT.md` isn't there, the agent doesn't do periodic tasks. Don't create one unless the user asks.
- **Putting prohibitions in IDENTITY.md** — they belong in SOUL.md. IDENTITY is about voice and presentation, not behavior.
- **Letting SOUL.md grow past 500 lines.** Split into references and point to them from SOUL. A 2000-line SOUL costs tokens on every turn.
- **Forgetting `company_id` / tenancy fields** when the agent interacts with a multi-tenant database. These are usually defined in TOOLS.md or in a dedicated `TENANCY.md`. Missing them = data leaks between customers.
