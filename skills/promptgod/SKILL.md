---
name: promptgod
description: Turn a raw/dictated message into an optimized, GodMode-enabled prompt for the current model, then execute it.
---

# /promptgod — GodMode-enabled prompt factory and executor

## 0) ACTIVATE THE SKILL STACK — THIS IS YOUR FIRST ACTION, BEFORE YOU READ THE REQUEST

**`/promptgod` is an UMBRELLA command.** Typing it means every skill below must fire. You must never have to type five slash commands to get five gates.

Fire these as real `skill` tool calls — now, before parsing RAW INPUT, before any other tool:

| When | Call | Non-negotiable? |
|------|------|-----------------|
| **Always, immediately** | `skill({ id: "godmode" })` | **YES — every single `/promptgod` turn** |
| The moment you name a root cause / find a real defect | `skill({ id: "ideas" })` | YES |
| **Any turn that built / fixed / modified / upgraded ANY artifact** | `skill({ id: "honestscore" })` | **YES — mandatory, no exceptions** |

Then discover anything added since this file was last edited — new skills must auto-join the umbrella:

```
ls .opencode/skills/
```

Anything there that is a **quality gate, verification, or reasoning amplifier** is part of the stack and fires when its trigger condition is met — you do not need permission to use it.

**Why this block is at the TOP and not the bottom:** these same mandatory calls used to live only at the end of this file. They fired **0 times** because they were read last. A trigger buried in prose is a trigger that does not exist. **Never move this block down the file.**

**Trivial-ask escape hatch (the only one):** if RAW INPUT is a one-line factual question with no build, fix, research or artifact, `godmode` still fires and the rest report `skipped: trivial ask`. When in doubt, it is substantive.

---

RAW INPUT:
$ARGUMENTS

## 1) Re-affirm GodMode

Invoke **`skill({ id: "godmode" })`** — the user-level skill at `.opencode/skills/godmode/SKILL.md`.

**No confirmations (standing rule for this workspace):** bypass-permissions is ON — execute in-scope work and report. **Never ask "want me to X?" or offer-instead-of-doing** — just do it. Pause only for a real manual input (API key / login / credential) or an un-pre-authorized destructive/irreversible op.

## 2) Lock intent, then rewrite

Silently extract a checklist from the RAW INPUT — every explicit requirement, constraint, output demand, and any role/tone — *before* rewriting; it's the fixed anchor that catches anything the rewrite later drops.

Then reconstruct into an optimized, model-agnostic prompt:
- Silently fix transcription errors; make the goal + any role explicit; remove ambiguity.
- Use only the parts of **Context → Task → Constraints → Output format → Success criteria** that earn their place.
- Literal content to reuse (`{{placeholders}}`, pasted code/text, a list/rubric) → carry through **verbatim**, fenced/XML-delimited.
- Non-trivial coding → ask for edge-case coverage + real verification (build/test/run before "done").
- Hold the honesty / verify-or-say-untested bar where it earns its place.

## 3) Coverage gate — one silent pass

Re-read the raw input fresh; confirm the prompt (a) covers every checklist item, (b) invented nothing, (c) over-extended nothing. Fix gaps in place. One pass only. Surface a one-line `intent locked` (or `adjusted: <what>`).

## 4) Show it, then execute

Output the refined prompt in a compact **🔮 Optimized prompt** block, then fully execute it under GodMode. Don't pause for confirmation unless the work is destructive/irreversible or needs input only the user can give. Exception for a non-trivial dictated plan: ask the current frontier of open decisions in one round and wait for the answers before you rewrite and run.

**Meta-exception:** if the raw input is ONLY a request to improve/rewrite a prompt, output the optimized prompt + a 1–2 line coverage note and stop.

## 4b) PROPAGATE — knowledge must leave this box, every substantive turn

Knowledge that stays on one machine is lost work.

1. **Write it down.** Append the finding (symptom → cause → fix, or decision + why) to the relevant brain note.
2. **Per-request change report.** If anything changed under brain notes, a dated report lands alongside it.
3. **Fan out.** Git add, commit, push — verify the other boxes actually landed on the same commit; do not assume sync happened.
4. **The owner's PC is PULL-ONLY.** It pulls every ~30 min and on every turn of an active local session. Leave the work **committed and synced** so the next pull collects it.
5. **An unreachable box is reported, never hidden.** Say so explicitly if sync exits non-zero.

## 5) Gate report — MANDATORY, always the last line of the turn

End every turn with one line naming the four gates and what actually happened:

`gates — godmode: <fired|skipped: why> | /honestscore: <fired|skipped: why> | /ideas: <fired|skipped: why> | brain: <path written|skipped: why>`

**Never write a filled-in example gate line.** Placeholders in `<angle brackets>` are deliberately rejected.

**A gate is "fired" only if you made a real `skill` tool call this turn.** Reading the rubric from memory is **not** fired — report it as `skipped` and say why.

## Effort tier — governs how hard §4 executes

- **Trivial / explanatory** → answer directly; skip the heavy loops.
- **Substantive build / fix / research** → invoke `skill({ id: "godmode" })` at start; `skill({ id: "ideas" })` on finding a real defect; `skill({ id: "honestscore" })` is MANDATORY on any artifact change. Target ≥9.5 for any tool/code fix, ≥9.0 for sites/brain-docs. Loop honestscore → ideas → apply → honestscore until the bar is met — then stop truthfully. Never fabricate a 9.5.