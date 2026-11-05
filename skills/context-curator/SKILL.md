---
name: context-curator
description: Fills agent context windows with ONLY the necessary material from the tool vault and knowledge base before execution. Trigger on: "load context for", "prep agent", "what does the agent need to know", before any multi-step agent task that references the vault or knowledge base. Anti-junk specialist — prevents irrelevant chunks from polluting context and causing tool-hallucination or dropped instructions.
---

# Context Curator

## Trigger

Fire this skill BEFORE loading any agent that needs to reference the tool vault, Obsidian brain, or knowledge base. Also fires when an agent output shows signs of hallucinated tool names, wrong vault paths, or missed instructions — those are symptoms of dirty context.

## Instructions

### Phase 1 — Define the Context Budget

1. Identify the model's usable context window (subtract system prompt + skill overhead; typical safe budget: 80 k tokens for Opus 4.8, 160 k for Fable 5).
2. Set a hard ceiling: **no single knowledge-base load exceeds 20% of the budget** (≈16 k / 32 k tokens respectively).
3. Never load "the full brain" or "all vault tools" — progressive disclosure only.

### Phase 2 — Query the Vault (Source Priority 1)

1. Open `_INDEX.md` (vault root) to get workspace list.
2. Navigate ONLY to the workspace matching the task domain:
   - web-dev task → `web-dev/knowledge/`
   - pentest task → `cybersec/knowledge/`
   - SEO task → `seo/knowledge/`
   - LLC/ops task → `llc/knowledge/`
3. Read the workspace index note first (≤500 tokens). Extract tool names, pattern slugs, and file refs relevant to the task.
4. Open individual notes only when they match 2+ task keywords. Stop after 3 notes or 8 k tokens, whichever comes first.

**Relevance scoring (fast heuristic):**
- Title contains a task keyword → score +2
- Note has a `## Tools` or `## Scripts` section → score +1
- Note was written within last 30 days → score +1
- Load notes scoring ≥3 first; skip notes scoring 0.

### Phase 3 — Query the Knowledge Base (Source Priority 2)

1. Only reach the knowledge base if the vault note is absent or scored <3.
2. Identify the knowledge-base path for the domain (same structure as vault).
3. Load the most-specific applicable note (e.g., `web-dev/knowledge/DESIGN-SYSTEM-NOTES.md`), not a directory listing.
4. Hard cap: 4 k tokens from this source.

### Phase 4 — Web Search (Source Priority 3)

1. Only reach web search if:
   - The task depends on a version/CVE/API that may have drifted since training.
   - The vault and knowledge base returned nothing useful.
2. Search for the SPECIFIC missing fact (e.g., "framer-motion 11 breaking changes") — not the whole topic.
3. WebFetch the authoritative page; extract only the relevant section (≤2 k tokens).

### Phase 5 — Assemble the Context Package

Produce a compact, ordered context block with this structure:

```
## Context Package — <task label>

### Vault Excerpts
<tool names, pattern slugs, command snippets — no padding>

### Knowledge Base Excerpts
<only the parts that answer the task's sub-questions>

### Live Web Facts (if any)
<versioned claim + source URL + date fetched>

### Excluded (why)
<list anything you considered and dropped, with one-line reason>
```

4. Paste the context package into the agent prompt BEFORE the task instructions — never after.

### Anti-Junk Rules (never break)

- Never include "related but might be useful" material. Useful = directly answers a sub-question in THIS task.
- Never paste a full file when a 10-line excerpt suffices.
- Never include headers, frontmatter, or wikilinks from notes — strip them before pasting.
- If you are uncertain whether a chunk is relevant, **exclude it**. A missing fact causes one retrieval. An irrelevant chunk causes tool-hallucination for the entire run.

## References

- Vault root: `C:\Users\rabim\OneDrive\Desktop\AI Stuff\agency-brain` (PC) / `/srv/dr/brains/` (VPS)
- VPS knowledge base: `/srv/dr/brains/deployrocket-central-brain/`
- Related skills: `tool-dispatcher` (decides which tool to use), `fleet-monitor` (heartbeat), `godmode` (overall task orchestration)
- Anthropic guidance: context window budget management per `claude-code` prompting docs 2026

## Scripts

```bash
# Estimate token count of a note before loading (rough: 1 token ≈ 4 chars)
wc -c "$NOTE_PATH" | awk '{printf "~%d tokens\n", $1/4}'

# List vault workspace notes by modification date (newest first)
ls -lt "$VAULT_WORKSPACE/knowledge/" | head -20
```