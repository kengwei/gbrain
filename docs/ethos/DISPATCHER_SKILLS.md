---
type: pattern
title: "Dispatcher Skills for Token-Constrained Harnesses"
subtitle: "When the Stock 29-Skill Install Eats Your Whole Context Window"
authors:
  - kengwei
  - "Claude Code (session 2026-04-21)"
created: 2026-04-21
updated: 2026-04-21
tags: [gbrain, gstack, skills, local-models, prompt-engineering, context-window, hermes]
status: draft-v1
prior: "Thin Harness, Fat Skills"
---

# Dispatcher Skills for Token-Constrained Harnesses

The stock gBrain install copies 29 skills into the agent's skills directory.
For Claude Desktop or OpenClaw-class harnesses with 200K+ context windows,
that's fine — skill descriptions loaded at prompt time add a few thousand
tokens and leave plenty of headroom.

For local models, it's a blocker.

Google Gemma 4 26B loaded via Ollama defaults to a 4,096-token context. The
Hermes Agent system prompt alone — before you've typed a single word — is
~12,500 tokens of tool definitions, personality, and skill descriptions.
Every skill you install adds another 100–500 tokens. 29 × 300 ≈ 8,700 tokens
just in skill metadata, loaded every turn, before any content.

The math doesn't work. You can bump the context window to 128K and pay for
it in RAM, or you can keep the install and have no room for the actual
conversation.

There's a better option.

## The pattern

**Install one dispatcher skill that wraps the `gbrain` CLI, with a table of
pointers to the full per-topic skills.** The dispatcher is always in the
system prompt. The deep skills live on disk and are loaded on demand via
the file toolset when the task matches.

```
~/.hermes/skills/gbrain/
  SKILL.md           ← 147 lines. Dispatcher. Always loaded.
                       Describes: brain philosophy, filing rules,
                       CLI cheatsheet, "when to write vs query",
                       [topic → full skill path] table.

~/gbrain/skills/
  query/SKILL.md     ← 400 lines. Loaded ONLY when the agent decides
  ingest/SKILL.md    ← 350 lines. it needs deep guidance on that topic.
  enrich/SKILL.md    ← 280 lines. Read via file toolset, not injected.
  briefing/SKILL.md  ← 220 lines.
  maintain/SKILL.md  ← 190 lines.
  ... (24 more)
```

The dispatcher's job is to teach the agent three things:

1. **The brain's philosophy** — what's worth writing, what's not, what
   conventions to follow. The stuff that rarely changes and applies to
   every interaction.
2. **The CLI primitives** — `gbrain query`, `gbrain put`, `gbrain
   extract`, etc. Enough to do the common cases inline.
3. **Where to look for more** — a two-column table mapping task types to
   full skill paths, so the agent can load the right deep skill when the
   dispatcher's inline guidance isn't enough.

The dispatcher is thin; the deep skills stay fat. This is **thin harness,
fat skills applied recursively** — the skills layer gets its own thin
harness in token-constrained environments.

## When to use this pattern

The 29-skill stock install works when:

- Context window is ≥ 200K (Claude Desktop, Claude Code, GPT-5 class)
- Skill descriptions loaded at prompt time are a rounding error on cost
- The harness's skill-matching logic benefits from seeing all skills at
  once (Claude Code's "description is the resolver" pattern)

The dispatcher pattern is the right move when:

- Context window is ≤ 32K and you're running locally (Gemma, Qwen, Llama,
  Mistral, DeepSeek on Ollama / LM Studio / vLLM)
- Each turn's token cost is non-trivial (self-hosted inference, API with
  strict budget, anything billed per-token)
- The agent's conversation topic is narrow within any single session —
  you don't flip between 15 different brain operations in one chat
- You accept manual-load latency for deep skills (one extra round-trip
  via the file toolset when a deep skill is needed)

The trade-off is explicit: **you lose automatic skill matching at prompt
time, you gain ~8K of reclaimed context**. For local models, that ratio
is almost always worth it.

## Anatomy of a dispatcher SKILL.md

```yaml
---
name: gbrain
description: |
  Shared Postgres-backed knowledge brain. Accessible via the gbrain CLI
  (no MCP server). Use gbrain to look up context before responding to
  questions, and to write new knowledge after learning something worth
  remembering. Invoke via the terminal toolset.
version: 1.0.0
triggers:
  - "what do I know about"
  - "look up in the brain"
  - "remember this"
  - "save this to gbrain"
metadata:
  hermes:
    tags: [Knowledge, Memory, Brain]
---

# gbrain Skill (CLI-driven)

## Architecture (2 sentences max)

Shared Supabase; multiple agents read/write the same pages. Writes go
through `gbrain put`; reads via `gbrain query` or `gbrain search`.

## The 80% CLI (inline — always available)

- `gbrain query "..."` — hybrid vector + keyword search
- `gbrain search "..."` — keyword-only (faster, no Ollama needed)
- `gbrain put <slug> < file.md` — create/update a page
- `gbrain stats` — sanity check: pages, chunks, link coverage

## When to write (philosophy, not procedure)

- Decisions made with rationale
- Factual discoveries about the user or their world
- Person/company profiles as they're learned

Don't write: acknowledgments, things outdated in 24h, duplicate content.

## When to load a deep skill

If the task is non-trivial, read the matching file with the read_file tool:

| Topic                              | Read                                            |
|------------------------------------|-------------------------------------------------|
| Deep query with citation chaining  | `~/gbrain/skills/query/SKILL.md`                |
| Meeting / article / tweet ingest   | `~/gbrain/skills/ingest/SKILL.md`               |
| Enriching existing pages from web  | `~/gbrain/skills/enrich/SKILL.md`               |
| Morning briefings                  | `~/gbrain/skills/briefing/SKILL.md`             |
| Brain health, orphans, stale pages | `~/gbrain/skills/maintain/SKILL.md`             |
| Filing conventions (slugs, tags)   | `~/gbrain/skills/_brain-filing-rules.md`        |

Those authors cover edge cases this dispatcher omits for brevity.
```

That's the whole dispatcher. ~80 lines of prose, 6 rows of table, 1 CLI
cheatsheet. It teaches the agent enough to handle the common case inline
and signals where to dig when the case isn't common.

## Installation recipe

```bash
# 1. Generate skill docs for the target host (applies tool-name rewrites)
cd ~/gbrain && bun run gen:skill-docs --host <host>

# 2. Hand-author the dispatcher SKILL.md for your agent
# (use the anatomy above as a template; ~100 lines)
cat > ~/.<host>/skills/gbrain/SKILL.md <<'EOF'
---
name: gbrain
description: ...
...
EOF

# 3. Skip the 28 generated full-skill directories. Leave them in
#    ~/gbrain/skills/ where the dispatcher's table points.
```

The manual step is hand-authoring the dispatcher from the anatomy above.
~100 lines, one-time cost per deployment.

## What this pattern is NOT

- **Not a replacement for full skills.** The deep skills still exist and
  still do the real work. This is about *when* they're loaded, not
  *whether* they're loaded.
- **Not "prompt compression."** No summarization, no information loss in
  the deep skills. The dispatcher is a router, not a compressor.
- **Not a workaround.** It's a deliberate architecture choice for a
  specific class of deployment. Wide-context harnesses should stick with
  the full install — the automatic skill-matching is worth the tokens.

## Observed trade-offs in practice (n=1)

Single reference deployment: Hermes Agent running Gemma 4 26B on Ollama,
128K context after model reload, 8 active toolsets of 26, 27 of 83 total
skills disabled. **One data point, not a survey.** Numbers below are
directionally useful; individual deployments will vary with model choice,
context window, and which toolsets are active.

- **Context at conversation start**: ~12.5K / 131K (9.5%) — enough
  headroom for 50+ substantive turns on this deployment.
- **Deep-skill-load cost**: one extra round-trip when the agent reads a
  full SKILL.md. The file is already on disk, so latency is sub-second.
  Tokens consumed scale with the skill loaded, not with the total install.
- **Miss rate on skill selection**: low so far on this deployment. The
  dispatcher's topic table is the fallback when inline CLI guidance isn't
  enough, and the agent reliably picks the right entry. More data points
  from other token-constrained deployments would strengthen (or complicate)
  this claim — contributions welcome.

## Open questions

- **How does this interact with gBrain's auto-activation triggers?** The
  frontmatter `triggers:` list in the dispatcher covers the common cases;
  deep skills don't get triggers unless they're loaded, which means the
  dispatcher becomes the single surface area for the agent's first-touch
  skill selection. Whether that's a feature or a bug depends on whether
  the dispatcher's triggers correctly fire for the 27 sub-skill cases.
- **Should the dispatcher be auto-generated from the full skill set?**
  A future `bun run gen:dispatcher-skill` that reads every SKILL.md,
  extracts the topic / one-line description / path, and emits a
  templated dispatcher would remove the hand-authoring step. Worth
  doing if this pattern sees adoption.

## Related

- [THIN_HARNESS_FAT_SKILLS.md](./THIN_HARNESS_FAT_SKILLS.md) — the
  underlying principle this pattern recursively applies
- [MARKDOWN_SKILLS_AS_RECIPES.md](./MARKDOWN_SKILLS_AS_RECIPES.md) — why
  the deep skills are worth loading on demand in the first place
- Reference implementation: `~/.hermes/skills/gbrain/SKILL.md` on a
  Hermes Agent running Gemma 4 locally (the deployment that surfaced
  the need for this pattern).
