# Agents and Sub-Agents

OpenClaw supports three organizational patterns. Pick the right one; mixing them produces security holes.

## Pattern 1 — Single agent, single workspace

The default. One agent, one workspace directory, one gateway binding per channel. Good for personal assistants and single-tenant installs.

Defined in `~/.openclaw/openclaw.json`:

```jsonc
{
  "agents": {
    "main": {
      "workspace": "/home/<user>/.openclaw/workspace",
      "models": { "primary": "openrouter/anthropic/claude-sonnet-4-5" },
      "maxConcurrent": 4
    }
  }
}
```

## Pattern 2 — Multiple peer agents, same install

Multiple agents on the same gateway, each with its own workspace. Different channels can route to different agents (e.g., a management Telegram bot and a customer-facing WhatsApp bot).

```jsonc
{
  "agents": {
    "manager":   { "workspace": "/home/<user>/workspaces/manager",   ... },
    "customer":  { "workspace": "/home/<user>/workspaces/customer",  ... }
  },
  "channels": {
    "telegram": { "agent": "manager", ... },
    "whatsapp": { "agent": "customer", ... }
  }
}
```

**Trust boundary**: Peer agents in the same gateway share the OS process and all env vars. They are NOT a security boundary between each other. Use this pattern only when all agents serve the same trust zone.

## Pattern 3 — Multiple gateways (true isolation)

For multi-tenant SaaS or any case where agents must not be able to read each other's state, run multiple OpenClaw processes. Each one has its own `OPENCLAW_CONFIG_PATH`, `OPENCLAW_STATE_DIR`, port, and workspace root.

```bash
OPENCLAW_CONFIG_PATH=/etc/openclaw/tenant-a.json \
OPENCLAW_STATE_DIR=/var/lib/openclaw/tenant-a \
openclaw serve --port 18790
```

Each instance is a separate systemd service, a separate user (recommended), or even a separate host. This is the only OpenClaw-blessed way to host mutually untrusted tenants.

## Sub-agents — spawning, not peer-running

Sub-agents are different from peer agents. A sub-agent is a *background run* spawned from a parent session. Session key pattern:

```
agent:<agentId>:subagent:<uuid>
```

They run in isolated sessions, post a completion event back to the parent, and by default do NOT get session tools. They're for:

- Parallelizing research / long tool-calls.
- Cost control (parent uses an expensive model; sub-agents use a cheap one).
- Keeping the main session transcript clean.

### Spawning a sub-agent

Two ways:

**1. User-facing slash command** (in a thread-supporting channel):
```
/subagents spawn main "Summarize the last 7 days of changelog entries and propose release notes"
```

**2. Tool call from the parent agent**: the parent invokes `sessions_spawn` with parameters:

- `agentId` (required) — which agent to spawn as
- `prompt` (required) — the task for the sub-agent
- `model?` — override the default model (invalid values are skipped, falls back with a warning)
- `thinking?` — override thinking level
- `runTimeoutSeconds?` — defaults to `agents.defaults.subagents.runTimeoutSeconds` or 0 (no timeout)
- `thread?` — default false; true = request channel thread binding
- `sandbox?` — optional sandboxing config

### Depth model — who gets what tools

OpenClaw strictly gates session tools by depth:

| Depth | Role | Has `sessions_spawn`? | Other session tools? |
|-------|------|----------------------|---------------------|
| 1 (orchestrator, `maxSpawnDepth >= 2`) | Main agent with permission to spawn | ✅ Yes | Yes: `sessions_list`, `sessions_history`, `subagents` |
| 1 (non-spawning, `maxSpawnDepth == 1`) | Main agent without spawn permission | ❌ No | No session tools |
| 2 (worker) | Sub-agent | ❌ No (always denied at depth 2) | No |

**Practical implication**: Before writing a SOUL.md that tells the agent "spawn a sub-agent to research X", verify the agent is configured as an orchestrator. Otherwise the tool call will be denied and the agent will thrash.

### The NO_REPLY convention

When a sub-agent completion event arrives after the parent has already sent its final answer, the correct response is the exact silent token `NO_REPLY` (or `no_reply`). Not "thanks", not a summary, not a new message — the literal token. OpenClaw suppresses the outbound message.

If you see an agent replying with garbage to its own sub-agent's late completion, the fix is to add to SOUL.md:
```
If you receive a sub-agent completion event after you have already replied to the user, respond with the exact literal token: NO_REPLY
```

### Orchestration pattern — start once, wait for event

The OpenClaw docs are explicit: **do not build poll loops** around `sessions_list`, `sessions_history`, `/subagents list`, or `exec sleep`. Spawn the child once, then wait for the completion event. Polling wastes tokens and creates race conditions.

Bad pattern (do not write skills that instruct this):
```
1. sessions_spawn("research task")
2. Loop: sessions_list; if done, break; else sleep 10s
```

Good pattern:
```
1. sessions_spawn("research task")
2. Wait. The event will arrive.
```

If the parent needs to do other work while waiting, `sessions_yield` pauses the parent cleanly until a child event arrives.

## Session tools — the full set

Available to orchestrator agents:

- `sessions_list` — list active and recent sessions
- `sessions_history` — read the transcript of a session (by key)
- `session_status` — check if a specific session is still running
- `sessions_send` — send a message into another session
- `sessions_spawn` — spawn a new sub-agent session
- `sessions_yield` — pause the current agent until a child event arrives
- `subagents` — status/stop operations on sub-agents

**Security**: Bearer token access to the gateway HTTP API grants *all* of these tools, effectively operator-level control. Never share the gateway token across trust zones, and never embed it in a SKILL.md or SOUL.md.

## Per-agent tool allow/deny lists

Configure in `openclaw.json`:

```jsonc
{
  "agents": {
    "customer-bot": {
      "tools": {
        "deny": ["exec", "filesystem_write", "sessions_spawn"],
        "allow": ["http_get", "web_search"]
      }
    }
  }
}
```

Use this aggressively. A public-facing Discord or WhatsApp bot should almost never have `exec` or filesystem write access. A personal assistant can.

## Creating a new workspace (full procedure)

1. Pick a path outside the default workspace: `mkdir -p /home/<user>/workspaces/<name>`.
2. Create the seven canonical Markdown files (see `workspace-structure.md`). Start IDENTITY.md, SOUL.md, USER.md; leave others empty until you need them.
3. Register the agent in `~/.openclaw/openclaw.json` under `agents.<name>`:
   ```jsonc
   "<name>": {
     "workspace": "/home/<user>/workspaces/<name>",
     "models": { "primary": "openrouter/anthropic/claude-sonnet-4-5" },
     "tools": { "deny": [] },
     "maxConcurrent": 4,
     "maxSpawnDepth": 1
   }
   ```
4. Bind a channel (or a session key) to this agent. If you skip this step, the agent exists but no one can talk to it.
5. Backup the config, then save. OpenClaw hot-reloads the config.
6. Verify: `openclaw agents list` should show the new agent.
7. Test with a minimal prompt through the bound channel.

## Common pitfalls

- **Two agents, one bot token**: Telegram (and most messaging platforms) accept only one polling connection per bot. If two gateways try to poll the same bot, they fight. Use a separate bot per gateway or bind the bot to one gateway only.
- **Sub-agent assumed to have filesystem access**: Sub-agents inherit the parent's tool denylist by default, but explicit `deny` rules at the sub-agent level are often stricter. Check.
- **Session key reuse**: Don't manually set a sub-agent's session key to a flat value — OpenClaw writes role/control scope into session metadata at spawn, and a flat key can accidentally regain orchestrator privileges on restore.
- **`service_role` key in an agent**: Using a Supabase `service_role` JWT in an agent bypasses all RLS. The agent can read every tenant's data. Use `anon` key + a per-tenant JWT instead, every time.
