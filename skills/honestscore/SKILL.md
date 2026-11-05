---
name: honestscore
description: Score a deliverable (code / brain doc / website) on the honest, non-optimistic 1–10 rubric — subtractive, evidence-gated, per-dimension, adversarial; code gets performance + concurrency dimensions and a measurable 9.7 bar for small code changes. Use before claiming "done" or inside any "loop until ≥X".
---

# honestscore — Honest Rubric Scoring

You are running the **Honest Rubric**. Score the target the user names — **blind and non-optimistic**: treat it as an anonymous third-party submission against the spec, never "your own work"; a real 8 is a good result. **Fluency is not evidence.**

**Step 0 — commit first, then look.** Before judging the artifact, write down in one or two lines what a correct/complete version *would* contain. Then diff the artifact against that.

**Step 0b — ask, don't tell.** Restate every load-bearing claim as a **question** before evaluating it — "Does it render offline?" not "It renders offline."

## The ONE scoring algorithm (follow in order)

1. **Split into applicable dimensions:** correctness · completeness · robustness/edge-cases · security · UX-visual · docs · **performance** · **concurrency**. A fixed list cannot score a task whose real failure mode sits outside these eight, so name that missing axis and score it too before trusting the number.
   - **performance** is applicable — never N/A — for any artifact whose purpose includes speed or about which ANY speed claim is made. Evidence tiers: before/after that passes the domain's statistical rule, command + output pasted → no cap; a single timing run, a profile screenshot or a traced hot path → ≤8.0; "should be faster" → ≤6.0.
   - **concurrency** is applicable for code with threads, async tasks, goroutines, shared mutable state or locks: a clean run of the language's race tooling lifts its cap; none → Unknown ≤6.0. **N/A is a claim you must defend:** write a one-line justification for each, and **N/A is forbidden for any axis the artifact plausibly exercises**. An axis you simply *cannot* verify is **Unknown → ≤6.0, never excluded**.

2. **Each applicable dimension = 10 − its own defect ledger.** Point costs: **cosmetic 0.1–0.3** · **moderate 0.4–0.8** · **serious 1–2**. No ledger → no score.

3. **Cap each dimension by evidence you PASTE, not evidence you hold.** Zero matching tool results → that dimension is capped at 6.0 arithmetically. Recorded run / passing automated test with output shown → no cap · reviewed screenshot or traced code-path → ≤9 · partial evidence → ≤7.0 · reasoning-only → ≤6.0.

4. **Overall = the MIN across applicable dimensions.** Consequence: **≥9 requires *every* applicable axis independently evidenced.** Report all three: `MIN (the score) / mean / the per-dimension vector`.

5. **Binary must-have gates are hard caps** (never averaged in): each is MET / NOT-MET. Any NOT-MET → cap **8.5**; a *critical* gate NOT-MET → cap **4.0**.

6. **Final number** = continuous to one decimal; **never round up**; on a genuine borderline, resolve downward.

7. **De-correlated adversarial pass — a binary must-have.** Write your own number to a scratch file BEFORE the adversary returns. Then spawn a subagent given only the artifact + this rubric, tasked *"build the case this is NOT a 9"*, required to **execute/read** the artifact, not reason about it. **Final = the lower number.** Raising it afterwards requires citing **specific new evidence**. If you cannot run the pass, that is a NOT-MET gate → cap 7.0.

## Loop discipline (when used inside "loop until ≥X")

- Every iteration must clear at least one external pass/fail check produced outside the model. No external check ran → the iteration doesn't count.
- Every score increase must name the specific defect it closed.
- Cap at 2–3 iterations for documents and sites. Code-performance loops may run up to 5 rounds.
- **9.7 for small, specific code changes:** every applicable dimension ≥9.5 · MIN ≥9.7 · every must-have MET · performance evidenced by its statistical rule · concurrency evidenced when applicable · zero unrebutted findings from the de-correlated adversarial pass.

## Output

- Per applicable dimension: its defect ledger + **the pasted evidence + the tool-result count** + its capped score (mark N/A **with justification** or Unknown).
- The MIN, any binary-gate caps, and the honest final number (post-adversarial).
- **A "NOT verified" list** — where there is no evidence, there is no score.
- **Typed defect tickets for `/ideas`:** each remaining defect as `defect | point-cost | evidence-that-would-close-it`, biggest-lever first.

Never round up. A 9.2 is a 9.2.