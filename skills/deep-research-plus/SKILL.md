---
name: deep-research-plus
description: Orchestration playbook for large multi-source research that needs parallel subagents and a cited, auditable report. Invoke when a question spans many independent sub-topics, needs cross-verified claims, or the user asks for a thorough/comprehensive research report. For a quick lookup, just search directly; for the bundled harness, the /deep-research skill may already fit — use this when you want the orchestration recipe and report schema below.
---

# Deep Research (Plus)

A recipe for fanning out research across subagents and synthesizing a verifiable report. This complements the bundled `/deep-research` skill — reach for that first for a standard run; use this when you want explicit control over fan-out and the report format.

## When to fan out (match scale to the question)

| Question shape | Approach |
|---|---|
| Single fact, 1–3 angles | Search + WebFetch directly in this turn. Don't spawn agents. |
| Comparison or 4–9 sub-topics | A handful of parallel subagents (Task / Workflow), one focused brief each. |
| Broad multi-axis investigation | 10+ subagents, clearly divided, orchestrator votes on claims and synthesizes. |

Subagents are worth it for **read-heavy, bounded, summarizable** work — each returns a conclusion, not a file dump. They are not free: a multi-agent run costs several times the tokens of a single agent, so don't fan out for work one context can hold.

## Give every subagent a complete brief

Vague briefs make agents duplicate work and leave gaps. Each one gets:
- **Objective** — one specific question, no ambiguity.
- **Output format** — findings + sources (with tier) + confidence + open questions.
- **Boundaries** — don't investigate adjacent topics; return raw findings, not recommendations.
- **Source guidance** — prefer official/primary 2025–2026 sources; skip pre-2025 unless it's the authoritative standard.
- **Stopping rule** — stop when sub-claims have independent corroboration and more searches stop adding signal. **No fixed quota** — "scour endlessly" and "stop too early" are both failure modes; converge on evidence, not a number.

## Cross-verify before synthesizing (vote on claims)

When agents return, build a claim ledger and tier each claim:
- **Verified** — 3+ independent, high-quality sources agree → use freely.
- **Probable** — 2 sources, or one solid secondary → use with a caveat.
- **Single-source** — flag as uncertain; it goes in Open Questions, never as a confident assertion.
- **Contradicted** — resolve before delivery, or report the disagreement explicitly.

Independent retrieval paths are the point: if one hallucination propagated across several agents, divergent sourcing catches it.

## Source tiers

| Tier | What | Examples |
|---|---|---|
| **S** | Official primary, current version | vendor docs/release notes, NIST NVD, MITRE CVE/CWE, RFCs, standards bodies |
| **A** | Maintainer-authored | maintainer GitHub issues/PRs with resolution, official CHANGELOG, recent official talks |
| **B** | High-signal community | high-vote Stack Overflow (2025–2026), recognized expert blogs, developer-consensus threads |
| **C** | Lower-signal | named-author personal blogs, tutorials — cite only when nothing better covers the angle, and label it |

Build conclusions on S+A. Use B for consensus checks. Discard anything older than 2025 unless it's still the authoritative reference.

## Report schema (for hand-off / long-lived docs)

```markdown
# [Topic] — Research Report
**Date · Confidence (with rationale) · Sources consulted**

## Executive summary
≤150 words, outcome first — what the reader gets if they read nothing else.

## Findings (per sub-topic)
- Claim · Verification (Verified / Probable / Single-source) · Sources [n] · Caveats

## Synthesis & recommendations
What the verified findings collectively mean. Ranked, actionable, with trade-offs.

## Risks & open questions
Single-source claims, unresolved contradictions, things to confirm before acting.

## References
Numbered, full URLs, with tier.

## Appendix: evidence ledger
Claim | Source URL | Verbatim quote | Tier — this is what makes the report auditable.
```

Keep headers literal (downstream agents pattern-match them), citations numeric with full URLs in References, code fences language-tagged, no emojis. If you run out of budget, deliver what you have with gaps marked — never silently truncate.
