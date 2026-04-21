# OpenClaw Config Files

Three JSON files hold the installation's configuration. All of them contain secrets in production.

## Where they live

```
~/.openclaw/
├── openclaw.json          # master config — agents, channels, gateway, models
├── agents/
│   └── <agent-id>/
│       └── agent/
│           ├── models.json         # per-agent model provider list
│           └── auth-profiles.json  # per-agent channel auth (bot tokens, webhooks)
└── state/                 # runtime state — do not hand-edit
```

Override the config location with `OPENCLAW_CONFIG_PATH`. Override state with `OPENCLAW_STATE_DIR`. For multi-tenant setups, set these per service unit.

## `openclaw.json` — anatomy

Top-level keys you'll see:

```jsonc
{
  "agents": { ... },        // agent definitions — see agents-and-subagents.md
  "channels": { ... },      // telegram, whatsapp, discord, slack, signal
  "gateway": { ... },       // port, bind mode, auth token, trusted proxies
  "commands": { ... },      // native commands, restart behavior, skill handling
  "skills": { ... },        // skill load paths and entries
  "logging": { ... }        // log levels, destinations
}
```

### `gateway` block

```jsonc
"gateway": {
  "port": 18789,
  "mode": "local",           // local | remote
  "bind": "loopback",        // loopback | all | custom
  "customBindHost": "0.0.0.0",
  "auth": {
    "mode": "token",         // token | none (none is dev-only, never prod)
    "token": "<REDACTED>"
  },
  "trustedProxies": ["127.0.0.1", "::1"],
  "controlUi": {
    "allowedOrigins": ["https://your-control-ui.example.com"]
  }
}
```

**Security notes**:
- `bind: "loopback"` is the safe default. Use `"all"` or a custom host only when the gateway must be reachable from other machines, AND you trust the network, AND the `auth.token` is strong.
- `auth.mode: "none"` disables authentication entirely. Only acceptable on a single-user dev machine behind a firewall. Never on a VPS.
- `controlUi.allowedOrigins` controls which domains can use the browser-based control UI. Do not put `*` here in production.

### `channels` block

Each channel has its own sub-block. Telegram example:

```jsonc
"telegram": {
  "enabled": true,
  "dmPolicy": "allowlist",       // allowlist | open | pairing
  "allowFrom": [123456789, 987654321],  // Telegram user IDs
  "groupPolicy": "allowlist",
  "groupAllowFrom": [-100123456789],    // Telegram group IDs (with -100 prefix)
  "botToken": "<REDACTED>",
  "streaming": "partial"         // off | partial | full
}
```

**Policy fields are security-critical**:
- `dmPolicy: "open"` means anyone who finds the bot can DM it. This usually leaks data and costs money.
- `dmPolicy: "allowlist"` + `allowFrom: []` means *everyone is blocked*. If the bot is silent, check this first.
- `groupPolicy: "allowlist"` + empty `groupAllowFrom` silently drops every group message. OpenClaw's doctor warns about this at startup — do not ignore the warning.

### `commands` block

```jsonc
"commands": {
  "native": "auto",           // auto | on | off
  "nativeSkills": "auto",
  "restart": true,
  "ownerDisplay": "raw"
}
```

- `native: "off"` disables built-in slash commands. Do this for hardened customer-facing bots.
- `restart: false` disables the `/restart` slash command for users.

## `models.json` — per-agent providers

```jsonc
{
  "providers": {
    "openrouter": {
      "baseUrl": "https://openrouter.ai/api/v1",
      "api": "openai-completions",
      "apiKey": "OPENROUTER_API_KEY",    // env var reference, not the literal key
      "models": [
        { "id": "anthropic/claude-sonnet-4-5", "name": "Claude Sonnet 4.5",
          "input": ["text", "image"], "contextWindow": 200000, "maxTokens": 8192,
          "cost": { "input": 3, "output": 15, "cacheRead": 0.3, "cacheWrite": 3.75 } },
        { "id": "anthropic/claude-opus-4-5", "name": "Claude Opus 4.5", ... },
        { "id": "google/gemini-2.5-pro-preview", ... },
        { "id": "auto", "name": "OpenRouter Auto", ... }
      ]
    }
  }
}
```

**Best practice**: reference API keys by environment variable name (`"apiKey": "OPENROUTER_API_KEY"`), not by literal value. Set the env var in the systemd unit file or a sourced `.env`.

**Adding a model**: append to the `models` array. No restart needed — OpenClaw reloads on file change.

**Removing a model**: pull it from the array. If an agent's config still references it, OpenClaw falls back to the default with a warning.

## `auth-profiles.json` — per-channel credentials

```jsonc
{
  "telegram": {
    "botToken": "<REDACTED>",
    "webhookUrl": null
  },
  "whatsapp": {
    "session": "<REDACTED>"
  }
}
```

**File permissions**: `chmod 600 auth-profiles.json`. Always. Check with `ls -la` before and after editing; some editors reset permissions on save.

## Hot reload — what reloads and what doesn't

OpenClaw's file watcher reloads most config changes within seconds:

| Change | Hot reload? |
|--------|-------------|
| Workspace Markdown files (SOUL.md, IDENTITY.md, etc.) | ✅ Yes, within ~2s |
| `openclaw.json` — `channels.*.allowFrom` additions | ✅ Yes |
| `openclaw.json` — `agents.<n>.models` | ✅ Yes |
| `openclaw.json` — `agents.<n>.tools.deny/allow` | ✅ Yes |
| `openclaw.json` — adding a new agent | ✅ Usually |
| `openclaw.json` — `gateway.port` / `gateway.bind` | ❌ No — requires restart |
| `openclaw.json` — `gateway.auth.token` rotation | ❌ No — requires restart |
| Environment variables (`OPENROUTER_API_KEY`, bot tokens in env) | ❌ No — systemd restart |
| `models.json` — adding/removing a model | ✅ Yes |
| Adding a skill to `<workspace>/skills/` or `~/.openclaw/skills/` | ✅ Yes, on next session start |

**When in doubt**, watch the log:
```bash
journalctl --user -u openclaw-gateway -f | grep -iE "reload|watch|config"
```

You'll see lines like `config reloaded` or `skill registry updated` when hot reload fires.

## Safe edit workflow for any config file

```bash
# 1. Backup
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%s)

# 2. Validate the current JSON before you start
python3 -m json.tool ~/.openclaw/openclaw.json > /dev/null && echo "valid JSON"

# 3. Make the edit (atomic write via temp file)
cp ~/.openclaw/openclaw.json /tmp/openclaw.json.edit
# ... edit /tmp/openclaw.json.edit ...
python3 -m json.tool /tmp/openclaw.json.edit > /dev/null || { echo "JSON broken — aborting"; exit 1; }
mv /tmp/openclaw.json.edit ~/.openclaw/openclaw.json

# 4. Watch the reload land
journalctl --user -u openclaw-gateway --since "10 seconds ago" | tail -20

# 5. Smoke-test through the channel
```

**Never** pipe `>` directly into `openclaw.json`. A crashed pipe leaves a truncated file and the gateway can fail to restart. Always write to a temp file and `mv` atomically.

## Multi-tenant / multi-gateway configs

If you see multiple `openclaw.json` files under `/etc/openclaw/` or multiple state dirs under `/var/lib/openclaw/`, you're on a multi-gateway host. Each gateway:

- Has its own systemd unit (typically `openclaw-gateway-<tenant>.service`)
- Binds a different port
- Has its own `auth.token`
- Must not share a workspace with another gateway

Edit only the config for the specific tenant you're working on. Cross-tenant edits should go through whatever deployment tooling the install uses (Ansible, a custom CLI, etc.).
