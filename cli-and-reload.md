# OpenClaw CLI, Service, and Reload

## Getting to a known state — the `--help` path

OpenClaw's command surface changes between versions. When unsure, check the installed version's help first:

```bash
openclaw --version
openclaw --help
openclaw <subcommand> --help
```

Do not assume commands from this reference are available without checking. If the help output contradicts what's written here, trust the help.

## The commands that actually matter

### Service lifecycle

OpenClaw installs as a **user systemd service by default**. Not a system service. This catches people constantly.

```bash
# Check status (user scope — this is the right one for most installs)
systemctl --user status openclaw-gateway

# Start / stop / restart (user scope)
systemctl --user start openclaw-gateway
systemctl --user stop openclaw-gateway
systemctl --user restart openclaw-gateway

# Enable on boot (requires linger for the user)
loginctl enable-linger <username>    # allows user services to start at boot
systemctl --user enable openclaw-gateway
```

If `systemctl --user` returns "Unit not found", the service isn't installed. Install it:
```bash
openclaw gateway install --force
```

If it installed under `/home/<user>/.config/systemd/user/openclaw-gateway.service` and still isn't found, you may need:
```bash
systemctl --user daemon-reload
```

### Running without systemd (foreground)

```bash
openclaw serve               # foreground, default config
openclaw serve --port 18790  # alternate port
```

Use this for debugging. For production, prefer the systemd unit — it handles restarts, logging, and lingering.

### Restart semantics — `gateway --force`

OpenClaw's equivalent to "restart the gateway process":

```bash
openclaw gateway --force
```

This kills the running gateway and starts a new one with the current on-disk config. Use it when:
- You changed `gateway.port`, `gateway.bind`, or `gateway.auth.token` (these don't hot-reload).
- You rotated an environment variable.
- The gateway is wedged and you've confirmed hot reload isn't picking up your change.

Do NOT use it as a reflex after any edit. Most edits hot-reload. Unnecessary restarts drop active sessions.

### Config management

```bash
openclaw configure                     # interactive top-level
openclaw configure --section model     # configure a model provider
openclaw configure --section telegram  # rotate Telegram bot token
openclaw configure --section whatsapp
openclaw configure --section gateway
```

Each section gives an interactive prompt. The changes are written to `~/.openclaw/openclaw.json` and hot-reloaded where possible.

### Onboarding a new install

```bash
openclaw onboard                              # interactive, prompts for everything
openclaw onboard --auth-choice openai-api-key
openclaw onboard --auth-choice openai-codex   # ChatGPT/Codex OAuth flow
```

Only run this on a fresh install. On a working system, `onboard` can overwrite existing configs.

### Skills commands

```bash
openclaw skills list                 # show all loaded skills with sources and eligibility
openclaw skills install <slug>       # install from ClawHub into <workspace>/skills
openclaw skills install <slug> --dangerous   # bypass the safety scanner (rarely appropriate)
openclaw skills show <name>       # show the SKILL.md and frontmatter for one skill
openclaw skills remove <name>     # remove a skill from the workspace
```

`openclaw skills list` is the single most useful diagnostic. It shows:
- Which skills loaded
- From which directory (precedence in action)
- Whether they passed the `requires` filter (eligibility)
- Why an ineligible one was rejected

### Agents commands

```bash
openclaw agents list               # list configured agents and their workspace paths
openclaw agents show <id>          # details for one agent (models, tools, channels)
openclaw agents restart <id>       # restart one agent without touching the gateway
```

### Doctor — the pre-flight check

```bash
openclaw doctor
```

Runs a battery of health checks. The output is worth reading fully after any significant config change. Common warnings and what they mean:

- **"groupPolicy is 'allowlist' but groupAllowFrom is empty — all group messages will be silently dropped"** → exactly what it says. Add group IDs or change the policy.
- **"gateway bind is 'all' with auth.mode 'none'"** → the gateway is exposed to the network without auth. Fix immediately.
- **"Model provider has no apiKey"** → the env var referenced by `models.json` isn't set in the systemd unit environment.

## Sessions and logs

### Live logs

```bash
journalctl --user -u openclaw-gateway -f              # follow
journalctl --user -u openclaw-gateway --since "5 minutes ago"
journalctl --user -u openclaw-gateway -n 200          # last 200 lines
```

Filter for specific events:
```bash
journalctl --user -u openclaw-gateway -f | grep -iE "error|warn|reload"
```

### Session inspection

Transcripts live under each agent's workspace in `sessions/`, and active session state under `~/.openclaw/state/`. To inspect without corrupting:

```bash
# List recent sessions
ls -lt ~/.openclaw/state/sessions/ 2>/dev/null | head

# Or per-workspace
ls -lt /home/<user>/workspaces/<n>/sessions/ | head

# Read a transcript (they're usually JSON or JSONL)
cat <session-file> | python3 -m json.tool | less
```

Never `rm` a session file while the gateway is running. End the session properly first, via the gateway's HTTP API or a slash command in the bound channel.

### Finding what the agent actually did

The transcript is the ground truth. If a user reports "the agent said X", find the session and read it. Do not rely on the agent's self-report of its own past turns — model memory drifts.

## Debugging checklist — "the bot isn't responding"

1. **Is the gateway running?**
   ```bash
   systemctl --user status openclaw-gateway
   ps -eo pid,user,cmd | grep openclaw | grep -v grep
   ```

2. **Is the port listening?**
   ```bash
   ss -tlnp | grep 18789
   ```

3. **Did the last reload succeed?**
   ```bash
   journalctl --user -u openclaw-gateway --since "10 minutes ago" | grep -iE "error|fail|reload"
   ```

4. **Is the channel enabled and the user allowlisted?**
   ```bash
   python3 -c "import json; c=json.load(open('/home/<user>/.openclaw/openclaw.json')); print(json.dumps(c['channels']['telegram'], indent=2))"
   ```

5. **Is there a conflicting second instance?**
   - Two processes for the same bot token fight for Telegram's long-poll. Only one wins per bot.
   - Check: `ps -ef | grep openclaw` across all users on the host.
   - Check: other machines on the same Tailscale / VPN running a competing gateway with the same token.

6. **Is the model provider reachable and billed?**
   ```bash
   curl -H "Authorization: Bearer $OPENROUTER_API_KEY" https://openrouter.ai/api/v1/models | head
   ```

7. **Does `openclaw doctor` flag anything?**

If all seven pass and the bot still doesn't respond, check the agent-level logs:
```bash
journalctl --user -u openclaw-gateway -f
# ...then send a test message. Every inbound event should log.
```

Silent inbound = channel problem (auth, allowlist, network).
Logged inbound, silent outbound = agent problem (SOUL.md bug, tool error, model failure).

## The "don't do this" list

- **`pkill -9 openclaw`** — kills the process mid-write; can corrupt state. Use `systemctl stop` or `openclaw gateway --force`.
- **Editing files under `~/.openclaw/state/` or `sessions/`** — active state. Read-only.
- **Running `openclaw onboard` on a working install** — overwrites config.
- **Rotating an env var without restarting** — OpenClaw doesn't re-read env after start. Restart the systemd unit.
- **`openclaw skills install <slug> --dangerous` without reading the skill** — the flag exists for emergencies; treat it like `rm -rf /` in terms of caution.
