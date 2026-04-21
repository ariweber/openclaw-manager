# Walkthrough — Setting Up a New Customer Bot

This is the happy-path script for "a customer asked me to set up a new OpenClaw bot." Follow the phases in order. Do not skip Phase 0.

## Phase 0 — Ask before you build

When a customer (operator) says *"set up a bot for me"*, *"תקים לי בוט חדש"*, or equivalent, **do not start executing**. Stop and collect these five facts first. If any answer is missing, ask the customer. Do not guess.

### Question 1 — Which channel(s)?

> "Telegram, WhatsApp, or both?"

If the customer says "whatever works", default to Telegram (simpler setup, clearer allowlist model) — but confirm.

### Question 2 — Who gets access?

This is channel-specific. Ask the relevant set of questions.

**If Telegram:**
- *"What Telegram user IDs should be allowed to DM the bot?"* (Numeric IDs, not @usernames. If the customer doesn't know their ID, point them to `@userinfobot` or similar.)
- *"Any Telegram groups the bot should respond in? Share the group ID with the `-100` prefix (e.g., `-1001234567890`)."*
- *"In groups, should the bot reply to every message, or only when @-mentioned?"*

**If WhatsApp:**
- *"The bot authenticates AS a WhatsApp account — whose number will it use? A dedicated number or an existing one?"* (Using an existing personal number binds the bot to every chat that number already has. Almost always the wrong move. Recommend a dedicated number.)
- *"Which phone numbers (in international format, e.g., +972501234567) should the bot accept DMs from?"*
- *"Which WhatsApp groups should the bot operate in? Share the group JID (e.g., `...@g.us`)."*
- *"In groups, reply to every message, only when @-mentioned, or only when a specific keyword appears?"*
- *"What should happen when a message arrives from someone NOT on the allowlist? Silent drop, polite decline, or forward to the operator?"*

**If both**: ask both sets. The two allowlists are independent.

### Question 3 — Agent identity

- *"What's the bot's name?"*
- *"Primary language — Hebrew, English, or bilingual?"*
- *"Tone — formal, casual, terse, playful?"*

### Question 4 — Purpose

- *"In one sentence, what is the bot supposed to do for you?"*
- *"What is it explicitly NOT supposed to do?"* (The "not" list is usually more important — it becomes the prohibitions section in SOUL.md.)

### Question 5 — Trust level

- *"Who is the operator (full control) and who is a regular user?"*
- *"Is this bot allowed to run shell commands on the server, or is it pure conversation?"*
- *"Should the bot ever send money, delete data, or make irreversible changes? If yes — under what confirmation protocol?"*

Write the answers in a temp scratch file (`/tmp/bot-setup-$(date +%s).md`) before continuing. You will reference them repeatedly in phases 2–4.

---

## Phase 1 — Create the workspace

```bash
WORKSPACE=/home/<user>/workspaces/<bot-name>
mkdir -p "$WORKSPACE"
cd "$WORKSPACE"
```

Create the three mandatory Markdown files from the answers in Phase 0:

**IDENTITY.md** — from Q3:
```markdown
# <Bot Name>

Language: <Hebrew | English | bilingual>
Tone: <tone from Q3>
Greeting: "<one-line default greeting>"
Emoji usage: <none | minimal | frequent>
```

**USER.md** — from Q2 and Q5:
```markdown
# Users

| Name | Phone / Telegram ID | Role | Notes |
|------|---------------------|------|-------|
| <operator> | <id> | operator | full control, can instruct destructive ops |
| <user 2>   | <id> | user     | read-only |
```

**SOUL.md** — from Q4 and Q5. Start minimal:
```markdown
# Role
<one sentence from Q4 answer to "what is the bot for">

# Forbidden
<bullet list from Q4 answer to "what it must NOT do">
<bullet list from Q5: destructive ops policy>

# Response format
<brief — terse, structured, plain text by default>

# Permissions
- Operator (<name>, <id>) may instruct any allowed action.
- Other users in USER.md may <read-only | ask questions | ...>.
- Anyone not in USER.md: <behavior from Q2 "off-allowlist" answer>.
```

You do NOT need to create the other canonical files (BOOTSTRAP.md, AGENTS.md, TOOLS.md, HEARTBEAT.md) now. Add them only when the bot needs what they provide.

**Expected check**: `ls $WORKSPACE` shows exactly `IDENTITY.md USER.md SOUL.md`.

---

## Phase 2A — Bind Telegram (if requested in Q1)

### Provision the bot

1. Customer creates a bot via `@BotFather` on Telegram. Receives a token like `1234567890:AAH...`.
2. Customer sends you the token privately — **not in the session transcript**. You store it via:
   ```bash
   openclaw configure --section telegram
   ```
   Paste the token when prompted. OpenClaw writes it to `auth-profiles.json` (chmod 600).

### Register the agent and channel

Edit `~/.openclaw/openclaw.json` (backup first — `cp openclaw.json openclaw.json.bak.$(date +%s)`):

```jsonc
{
  "agents": {
    "<bot-name>": {
      "workspace": "/home/<user>/workspaces/<bot-name>",
      "models": { "primary": "openrouter/anthropic/claude-sonnet-4-5" },
      "maxConcurrent": 4,
      "maxSpawnDepth": 1
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "agent": "<bot-name>",
      "dmPolicy": "allowlist",
      "allowFrom": [<IDs from Q2>],
      "groupPolicy": "allowlist",
      "groupAllowFrom": [<group IDs from Q2>],
      "streaming": "partial"
    }
  }
}
```

Validate JSON and atomically save (see `config-files.md` → Safe edit workflow).

### Confirm reload

```bash
journalctl --user -u openclaw-gateway --since "30 seconds ago" | grep -iE "reload|telegram"
openclaw agents list | grep <bot-name>
openclaw doctor
```

`doctor` should print no warnings about `allowFrom: []` or missing tokens.

---

## Phase 2B — Bind WhatsApp (if requested in Q1)

> **Note**: WhatsApp channel config in OpenClaw is less uniform than Telegram across versions. Prefer the interactive configure command over hand-editing JSON:
>
> ```bash
> openclaw configure --section whatsapp
> ```
>
> This walks through the session / pairing process and writes the correct fields for your install's version. If you must hand-edit, read your install's current `openclaw.json` schema first — do not assume the fields from this walkthrough.

### Pairing (session creation)

WhatsApp does not use bot tokens. The bot pairs with a phone number via QR scan or pairing code — exactly like WhatsApp Web. Steps:

1. Confirm with the customer that they have the dedicated phone number available and the SIM/app active.
2. Run `openclaw configure --section whatsapp` and follow the interactive flow.
3. The operator scans the QR on the phone (or enters the pairing code). This creates a persistent session in `auth-profiles.json` — **this session file is now as sensitive as the WhatsApp account itself**. Chmod 600, never commit, never screenshot.

### Access policy fields

In your install's `openclaw.json`, the whatsapp channel block looks roughly like:

```jsonc
"whatsapp": {
  "enabled": true,
  "agent": "<bot-name>",
  "dmPolicy": "allowlist",
  "allowFrom": ["+972501234567", "+972502223344"],   // phone numbers from Q2
  "groupPolicy": "allowlist",
  "groupAllowFrom": ["1234567890@g.us"],              // group JIDs from Q2
  "groupReplyMode": "mention-only",                   // every | mention-only | keyword
  "offAllowlistPolicy": "silent"                      // silent | decline | forward-to-operator
}
```

**Field names differ between OpenClaw versions.** If any field is rejected on reload, run `openclaw configure --section whatsapp` and match the names it writes.

### Confirm reload

```bash
journalctl --user -u openclaw-gateway --since "30 seconds ago" | grep -iE "reload|whatsapp"
openclaw doctor
```

### WhatsApp-specific traps

- **Two gateways with the same WhatsApp session** = both disconnect immediately. One session per device, always.
- **Using a personal number instead of a dedicated one** = the bot sees every chat on that number. Strongly discourage.
- **Group JIDs change** if a group is re-created. If the bot stops responding in a group, verify the JID.
- **Business API vs session-based (Baileys-style)**: these are different integration modes with different auth-profiles.json shapes. Check `openclaw --version` and the install's docs.

---

## Phase 3 — Smoke test

Test from the channel. Do not declare the bot working based on log output alone.

**Telegram**:
1. Operator (from USER.md) sends `/start` or a plain "hi" to the bot.
2. Bot responds within 10s.
3. A second user NOT in `allowFrom` sends a message. Bot should NOT respond (if `dmPolicy: allowlist`).

**WhatsApp**:
1. Operator sends a DM to the bot's number. Bot responds.
2. Non-allowlisted number sends a DM. Behavior matches `offAllowlistPolicy` from Q2.
3. In an allowlisted group, operator sends a message matching the group-reply rule from Q2. Bot responds.

While testing, follow the logs:
```bash
journalctl --user -u openclaw-gateway -f
```

Every inbound message should produce a log line. Silent inbound = channel/auth problem. Logged inbound with no outbound = agent problem (SOUL.md, model, tool error).

---

## Phase 4 — Handoff to the operator

Send the customer:
1. The bot's public handle (Telegram @username, or WhatsApp number).
2. A one-paragraph description of what it does (derived from SOUL.md `# Role`).
3. The allowlist contents (so they can verify it matches their expectations).
4. Instructions for reaching you if the bot misbehaves: which log command, which workspace path.
5. A statement of what is NOT configured yet (e.g., "HEARTBEAT.md is empty — no scheduled behavior"), so the operator knows what they don't have.

Do NOT send secrets. Not the bot token, not the WhatsApp session file, not the gateway auth token.

---

## If anything fails

- Silent bot → `openclaw doctor`, then check the channel allowlist.
- 500 errors on reload → JSON was saved broken; restore from `.bak.*` file.
- Bot answers in the wrong language → fix IDENTITY.md, save, wait ~2s for hot reload, retry.
- Bot answers with garbage after tool calls → SOUL.md response-format section is missing or contradictory.

See also: `safety-rules.md` (the 4-question data-leak self-test before declaring done), `cli-and-reload.md` (debugging checklist).
