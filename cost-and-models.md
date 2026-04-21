# Cost and Model Selection

When an operator says *"the bot is costing too much"*, *"למה זה עולה כל כך?"*, or asks about switching to a cheaper model — this is the reference.

## The three places cost becomes visible

### 1. OpenRouter dashboard (per-request ground truth)

If your `models.json` uses OpenRouter, the authoritative per-request cost lives at <https://openrouter.ai/activity>. Every call is logged with input tokens, output tokens, cached tokens, and the dollar amount. Filter by API key to isolate this install's traffic.

This is the only place you see cache-hit savings reflected in dollars. Start here whenever cost is the question.

### 2. OpenClaw gateway logs

Per-turn token counts are logged by the gateway:

```bash
journalctl --user -u openclaw-gateway --since "1 hour ago" \
  | grep -iE "tokens|usage|cost"
```

These lines show `input`, `output`, and (where supported) `cache_read` / `cache_write` counts per LLM call. They do **not** show dollar amounts — those come from the provider.

### 3. Session transcripts

Each session under `<workspace>/sessions/` (or `~/.openclaw/state/sessions/`) contains the full turn-by-turn record, often with token counts attached. Useful for answering "which session cost the most" or "which tool call ballooned the context".

```bash
ls -lt <workspace>/sessions/ | head
# Read the most expensive ones:
grep -E '"(input|output)_tokens"' <session-file> | head
```

## Model picker — what to use where

The rule of thumb: use the cheapest model that reliably completes the task. Overspending on a strong model for deterministic or narrow work is the most common cost leak.

| Task type | Recommended tier |
|-----------|-----------------|
| Orchestrator agent (SOUL.md logic, multi-tool reasoning) | Sonnet, or Opus for complex workflows |
| Sub-agent doing one deterministic task | Haiku |
| Summarization / extraction from a doc | Haiku |
| Tool-call routing (decide which tool, forward args) | Haiku — nearly as good as Sonnet here |
| Code generation with real stakes | Sonnet minimum, Opus when context is large |
| Code review / architecture discussion | Opus |
| Hebrew / bilingual chat with mild reasoning | Sonnet |

For sub-agents specifically, set the model explicitly in the `sessions_spawn` call — do NOT let sub-agents inherit the orchestrator's expensive model by default.

## Switching a model

### For one agent

Edit `~/.openclaw/openclaw.json`:

```jsonc
"agents": {
  "<bot-name>": {
    "models": {
      "primary": "openrouter/anthropic/claude-haiku-4-5"   // was claude-sonnet-4-5
    }
  }
}
```

Save atomically (see `config-files.md` → Safe edit workflow). OpenClaw hot-reloads. Test with a real prompt on the channel — benchmarks from the provider do not predict how your specific SOUL.md will behave.

### For a specific sub-agent task

Override per spawn:

```json
{ "agentId": "<bot-name>", "prompt": "...", "model": "openrouter/anthropic/claude-haiku-4-5" }
```

Invalid model IDs are silently skipped and fall back to the agent's default with a log warning. Grep for `model fallback` in the journal if a spawn didn't land on the cheap model.

### Adding a new provider entry

If the cheaper model isn't in `models.json`, add it:

```jsonc
{ "id": "anthropic/claude-haiku-4-5", "name": "Claude Haiku 4.5",
  "input": ["text", "image"], "contextWindow": 200000, "maxTokens": 8192,
  "cost": { "input": 0.25, "output": 1.25, "cacheRead": 0.03, "cacheWrite": 0.3 } }
```

Verify cost numbers against the provider's current pricing — they change. Stale cost estimates in `models.json` affect nothing at runtime (OpenRouter bills what it bills) but can mislead dashboards and decisions.

## Prompt caching — the biggest lever

Anthropic and OpenAI both support prompt caching: repeated prefixes are charged at a fraction of the normal input rate (~10% on Anthropic). For a long SOUL.md read on every turn, this is the difference between paying full price 100 times a day and paying cache-read price 99 of them.

### How to tell if caching is working

- OpenRouter dashboard: look at the `Cached` column per request. Zero on every turn = caching not active.
- Gateway logs: lines with `cache_read` > 0 = hitting cache.

### What kills cache hit rate

- **Changing SOUL.md** every few minutes. Each edit invalidates the cached prefix; the next turn pays full price.
- **Putting timestamps or rotating content in the prefix.** A SOUL.md that injects `Today is 2026-04-21` on every turn has a cache-key that changes daily. Move volatile content to the suffix or to a tool call.
- **Random skill loading order.** If the skills metadata block sorts differently each session (e.g., directory scan order on different filesystems), the cache key shifts.

### What to aim for

On a stable agent with a static SOUL.md and well-ordered skills, cache-read ratio should sit above 80% after the first few turns of a session. Below 50% sustained = something in the prefix is churning; investigate.

## Cost investigation checklist — "the bill is high"

1. **Pull the last 7 days from OpenRouter.** Sort by cost descending. Which agent? Which session?
2. **Open the top session's transcript.** Count turns. Are there obvious loops (same tool called 20 times)?
3. **Check cache-read ratio on that session.** Low ratio = prefix churn, not task volume.
4. **Check model ID.** Is the orchestrator on Opus when Sonnet would do? Are sub-agents inheriting Opus by default?
5. **Check `maxConcurrent` and `maxSpawnDepth`.** A runaway orchestrator spawning 10 Opus sub-agents per task will quintuple the bill silently.
6. **Check skill bloat.** `openclaw skills list` — every loaded skill adds to the prefix on every turn. If 30 skills are loaded and 3 are used, the other 27 are paid-for dead weight. Narrow the `description` fields so they only trigger when actually relevant.

## When to stop optimizing

Cost optimization has diminishing returns. If you've:
- Switched to the right model tier for the task,
- Cache-read ratio is >80%,
- No orphaned sub-agents looping,
- Skill list is narrow,

…and the bill is still too high, the honest answer to the operator is that the task itself is expensive. More aggressive cuts (harsher summarization, smaller context windows, cheaper tier) will degrade quality. Surface the tradeoff; don't hide it.
