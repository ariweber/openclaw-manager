# OpenClaw Skills — Authoring and Installing

A skill is a directory with a `SKILL.md` at its root. The SKILL.md has YAML frontmatter (`name`, `description` required) and Markdown body. Optionally: a `scripts/` directory with executables, a `references/` directory with docs loaded on demand, an `assets/` directory.

OpenClaw uses the **AgentSkills** spec. Same format as Claude's own skills.

## Loading order and precedence — memorize this

When OpenClaw starts a session, it loads skills from these locations, in this order (higher wins on name collision):

1. `<workspace>/skills/` — **highest precedence, per-workspace**
2. `<workspace>/.agents/skills/` — project-level, applied before workspace skills
3. `~/.agents/skills/` — personal, cross-workspace on this machine
4. `~/.openclaw/skills/` — managed/local, visible to all agents on this machine
5. Bundled skills — shipped with the OpenClaw install
6. `skills.load.extraDirs` entries in `~/.openclaw/openclaw.json` — lowest precedence

**Practical implication**: if you drop a skill named `git-workflow` into `<workspace>/skills/` and a community skill named `git-workflow` exists in `~/.openclaw/skills/`, the workspace one wins — silently. No warning, no merge. This is the #1 cause of "my skill changes aren't taking effect" bugs.

## SKILL.md — minimal format

```markdown
---
name: skill-name
description: One sentence on what it does. Include trigger phrases. Be specific about WHEN to use.
---

# Skill Name

Instructions for the agent. Markdown. Keep under 500 lines.

## Workflow
1. Read X
2. Call Y
3. Return Z
```

**Frontmatter rules (important — OpenClaw's embedded parser is strict):**
- `name` and `description` are required.
- Only single-line frontmatter values are supported. Do not use YAML block literals (`|` or `>`).
- `metadata` (if used) must be a single-line JSON object.
- No comments inside the frontmatter block.

## The description is the trigger

OpenClaw (like Claude) picks skills by matching the user's request against the `description`. Good descriptions:

- State both **what** the skill does AND **when** to use it.
- Include specific phrases the user is likely to say (in the user's actual language — Hebrew triggers if users speak Hebrew).
- Err on the side of being slightly pushy ("Use this skill whenever X is mentioned") to combat under-triggering.
- Are under 2–3 sentences. Longer descriptions dilute the signal.

Bad:
```
description: Utility for file operations.
```

Good:
```
description: Use whenever the user asks to read, write, move, or diff files in the car-inventory workspace, including Hebrew phrases like "תראה לי את הקובץ" or "תשמור את זה". Handles markdown, JSON, and CSV. Returns minimal {path, bytes_written} summaries.
```

## Progressive disclosure — the token-budget pattern

OpenClaw loads skills in three tiers:

1. **Metadata** (name + description) — always in context. Keep tight.
2. **SKILL.md body** — loaded when the skill triggers. Aim for under 500 lines; 200 is better.
3. **Bundled resources** (references/, scripts/, assets/) — loaded only when the skill itself reads them.

Put only the "always-needed" instructions in SKILL.md. Move long references, per-domain specifics, and rare-case procedures into `references/` files and link to them by path:

```markdown
For AWS-specific deployment, read references/aws.md.
For GCP, read references/gcp.md.
```

The agent reads only what it needs.

## Scripts beat prose

If a step is deterministic — "list all cars in the database", "count files matching X", "zip the logs folder" — write a shell or Python script in `scripts/` and tell the agent to run it. A 20-line script replaces 500 tokens of prose instructions AND is reliable.

```
my-skill/
├── SKILL.md         → "Run scripts/inventory.sh. Parse output."
└── scripts/
    └── inventory.sh
```

The agent executes, receives compact output, and only then reasons about it. This is the single biggest token optimization available.

## Minimize tool output

If your skill invokes a gateway tool or external API, wrap the call so only the *needed* fields come back. Don't return an entire Supabase response object if you only need `{ok: true, id}`. Tokens spent on unused fields are tokens you pay for every turn the context stays alive.

## File layout examples

Simple skill (most common):
```
git-safe-commit/
└── SKILL.md
```

Skill with helper script:
```
db-snapshot/
├── SKILL.md
└── scripts/
    └── snapshot.sh
```

Skill spanning multiple domains:
```
cloud-deploy/
├── SKILL.md              (workflow + picker: which cloud?)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Skill with assets (e.g., templates):
```
report-generator/
├── SKILL.md
├── scripts/
│   └── build.py
└── assets/
    └── report-template.docx
```

## Installing skills

### From ClawHub (the community registry)

```bash
# From inside the workspace you want to install into
openclaw skills install <slug>
```

This downloads a ClawHub skill folder into `<workspace>/skills/`. OpenClaw's built-in dangerous-code scanner runs before installation. Critical findings block the install unless you pass `--dangerous`.

**Security**: Treat third-party skills as untrusted code. Read the SKILL.md and every script before enabling. If it spawns a shell, calls an external URL, or reads files outside the workspace — review twice.

### From a local directory

Just copy it:
```bash
cp -r ~/my-drafts/my-skill ~/.openclaw/skills/
# or for workspace-only scope:
cp -r ~/my-drafts/my-skill <workspace>/skills/
```

OpenClaw picks it up on the next session. No restart needed.

### From a git repo

```bash
git clone https://example.com/my-team/private-skills.git ~/private-skills
# Then point skills.load.extraDirs at it in ~/.openclaw/openclaw.json:
# "skills": { "load": { "extraDirs": ["/home/<user>/private-skills"] } }
```

This gives lowest precedence, which is what you usually want for a shared team registry (so workspace overrides still work).

## Overriding a bundled skill

Create a skill with the same `name` in a higher-precedence location. For example, to patch a bundled skill named `web-research` without forking the OpenClaw install:

```bash
cp -r /path/to/bundled/web-research ~/.openclaw/skills/web-research
# Edit ~/.openclaw/skills/web-research/SKILL.md
```

Your version now wins. When OpenClaw updates, the bundled version updates too, but your override stays in place. Revisit periodically to rebase your changes.

## Common authoring mistakes

- **SKILL.md too long.** Anything over 500 lines should be split. If you're approaching 300, start planning the split.
- **Description too generic.** "Helps with files" — triggers on everything, triggers usefully on nothing.
- **Multi-line frontmatter values.** Will silently fail to parse with OpenClaw's embedded parser. Keep values on one line.
- **Instructing the agent to "remember X across turns".** Skills don't persist state. Use the workspace's memory files or `SHARED_KNOWLEDGE.json` for that.
- **Forgetting that the skill is shadowed.** If you edit a bundled skill and your workspace has an override, nothing changes in practice. Always check with `openclaw skills list` (or inspect the precedence chain manually).

## Verifying a skill was loaded

```bash
openclaw skills list
# Should show your skill with its source directory and "eligible" status.
```

If it's listed but not "eligible", the filter rejected it — usually because a required binary isn't installed or an env var is missing. Check the `requires` field in the skill's SKILL.md frontmatter (if any) and the OpenClaw skills log output.
