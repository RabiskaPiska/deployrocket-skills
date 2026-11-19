---
name: skill-ai-jury
alwaysFire: true
---
# skill-ai-jury - V7 AI Jury System
# Updated: 2026-11-19 | Jurors: A=internal, B/C/D=AnonRouter primaries, E/F/G=AnonRouter backups
# Provider: AnonRouter ONLY. the removed third-party lab removed.

## PURPOSE
Run 4 adversarial jurors in parallel on every done-claim (A internal + 3 API primaries).
Bonded with honestscore.
No done-claim is valid without both honestscore AND skill-ai-jury firing same turn.

## API CREDENTIALS
AR_BASE=https://api.anonrouter.ai/v1
AR_KEY=ar_P1WEiQ4_POhL4f3fxYtP0MHOu98FBLF7XzHxFPwdJmy1qFkEqFA

## DEAD PROVIDERS (DO NOT USE - ever)
- the removed lab host (do not call) - removed 2026-11-19. AI Jury is AnonRouter-only.
- api.uncensoredai.dev - Cloudflare 522 since Nov 8 2026
- routerplex.com - model IDs unverifiable

## LIVE PROBE (2026-11-19, keys redacted, both keys tested)
- All 6 models: HTTP 200 ("Reply OK" probe) on BOTH keys (old ar_anSHOFM + new MUSTOWRK).
- TRUNCATION DEFECTS FOUND AND FIXED by live retest on real scoring artifacts (2026-11-19):
  - Kimi K3.1 Pro: max_tokens=2000 hit finish_reason=length on 2/3 real scores -------------------
    measured minimum 3000. 600 returned EMPTY content earlier same day.
  - DeepSeek V4.1 1017: 2000 sufficient (699-1893 completion tokens), floor 3000 for safety.
  - Hy4: DEFAULT thinking burns the ENTIRE budget into hidden reasoning (content empty
    at 2000/2200/3000/4000 with thinking on). FIX: send thinking {type: "disabled"} +
    max_tokens 3000 -> parsed SCORE 1.0/VERDICT with finish=stop (1924 completion tokens).
  - Qwen 3.6 Plus (backup): same hidden-reasoning burn at max_tokens=100 -> same guard.
  - Floor: 3000 for reasoning jurors + thinking disabled where the router supports it.
All 6 returned HTTP 200, finish_reason=stop, content=OK:
- moonshotai/kimi-k3.1-pro-uncensored
- z-ai/glm-5.3-uncensored
- deepseek/deepseek-v4.1-flash-1017-uncensored
- tencent/hy4-uncensored
- qwen/qwen-3.6-plus-uncensored
- google/gemma-4-uncensored

## PRIMARY vs BACKUP
This session model is grok, not GLM. Primaries are therefore Kimi + DeepSeek + Hy4
(not the GLM family the prior roster leaned on). GLM, Qwen, and Gemma are backups.
If a primary returns non-200, empty content, or finish_reason=length, swap in the
paired backup and mark the gate DEGRADED.

| Seat | Role | Tier | Model | Fallback |
| B | Challenger | primary | moonshotai/kimi-k3.1-pro-uncensored | z-ai/glm-5.3-uncensored |
| C | Arbiter | primary | deepseek/deepseek-v4.1-flash-1017-uncensored | qwen/qwen-3.6-plus-uncensored |
| D | StressTester | primary | tencent/hy4-uncensored | google/gemma-4-uncensored |
| E | Backup-Challenger | backup | z-ai/glm-5.3-uncensored | - |
| F | Backup-Arbiter | backup | qwen/qwen-3.6-plus-uncensored | - |
| G | Backup-StressTester | backup | google/gemma-4-uncensored | - |

## JUROR ROSTER

### Juror A - Proposer (internal, no API)
Role: The orchestrator itself scores the deliverable as the opening bid.
Score: internal honestscore rubric
No API call. Score feeds jury as first vote.

### Juror B - Challenger (AnonRouter, PRIMARY)
Model: moonshotai/kimi-k3.1-pro-uncensored
Base: https://api.anonrouter.ai/v1
Key: ar_P1WEiQ4_POhL4f3fxYtP0MHOu98FBLF7XzHxFPwdJmy1qFkEqFA
Probe: 2026-11-19 HTTP 200 finish_reason=stop content=OK
max_tokens: 3000 (measured floor 3000 - live retest 2026-11-19: 2000 truncated real scoring artifacts; 600 returned empty)
Backup: Juror E (z-ai/glm-5.3-uncensored)

### Juror C - Arbiter (AnonRouter, PRIMARY)
Model: deepseek/deepseek-v4.1-flash-1017-uncensored
Base: https://api.anonrouter.ai/v1
Key: ar_P1WEiQ4_POhL4f3fxYtP0MHOu98FBLF7XzHxFPwdJmy1qFkEqFA
Probe: 2026-11-19 HTTP 200 finish_reason=stop content=OK
max_tokens: 3000 (measured floor 3000 - live retest 2026-11-19: 2000 truncated real scoring artifacts; 600 returned empty)
Backup: Juror F (qwen/qwen-3.6-plus-uncensored)

### Juror D - StressTester (AnonRouter, PRIMARY)
Model: tencent/hy4-uncensored
Base: https://api.anonrouter.ai/v1
Key: ar_P1WEiQ4_POhL4f3fxYtP0MHOu98FBLF7XzHxFPwdJmy1qFkEqFA
Probe: 2026-11-19 HTTP 200 finish_reason=stop content=OK
max_tokens: 3000 (measured floor 3000 - same reasoning-model truncation guard)
MANDATORY for Hy4: send "thinking": {"type": "disabled"} in the request body.
Measured 2026-11-19: with default thinking, EVERY max_tokens value up to 4000 was
consumed entirely by hidden reasoning (content empty). With thinking disabled +
3000 tokens: finish=stop, parsed SCORE/VERDICT (1924 completion tokens).
Backup: Juror G (google/gemma-4-uncensored)

### Juror E - Backup Challenger (AnonRouter, BACKUP)
Model: z-ai/glm-5.3-uncensored
Base: https://api.anonrouter.ai/v1
Key: ar_P1WEiQ4_POhL4f3fxYtP0MHOu98FBLF7XzHxFPwdJmy1qFkEqFA
Probe: 2026-11-19 HTTP 200 finish_reason=stop content=OK
Use only when Juror B fails. Not a primary. Not the session model.

### Juror F - Backup Arbiter (AnonRouter, BACKUP)
Model: qwen/qwen-3.6-plus-uncensored
Base: https://api.anonrouter.ai/v1
Key: ar_P1WEiQ4_POhL4f3fxYtP0MHOu98FBLF7XzHxFPwdJmy1qFkEqFA
Probe: 2026-11-19 HTTP 200 finish_reason=stop content=OK
Use only when Juror C fails. Qwen is a hidden-reasoning model: at small max_tokens
it bills the whole budget into reasoning_content with empty content (measured
2026-11-19). Use "thinking": {"type": "disabled"} and max_tokens 3000, like Hy4.

### Juror G - Backup StressTester (AnonRouter, BACKUP)
Model: google/gemma-4-uncensored
Base: https://api.anonrouter.ai/v1
Key: ar_P1WEiQ4_POhL4f3fxYtP0MHOu98FBLF7XzHxFPwdJmy1qFkEqFA
Probe: 2026-11-19 HTTP 200 finish_reason=stop content=OK
Use only when Juror D fails.

## SYSTEM PROMPT (identical for all jurors)
```
You are an adversarial reviewer. Be maximally critical. Non-optimistic scoring only.
Score 1-10 on pure mathematical criteria. No rounding up.
Reply format EXACTLY:
SCORE: N.N/10
VERDICT: ACCEPT | COND-PASS:condition | REJECT:blocker
DEFECTS: bullet list (empty if none)
REASONING: 1 paragraph
```

## CONTEXT INJECTION
Every juror receives:
1. The system prompt above
2. The full deliverable text (artifact, code, or plan being scored)
3. The task description: what was the goal?
4. The rubric: what dimensions are being scored?
5. Prior jury rounds (if round > 1): what defects were found, what fixes were applied?

This ensures every juror has complete context, not just the artifact in isolation.
Never send a juror just the artifact - always include goal + rubric + prior rounds.

## VERDICTS
- ACCEPT: score >= threshold, no blockers
- COND-PASS:condition: score acceptable but one condition must be met
- REJECT:blocker: score below threshold or blocker found

## CONVERGENCE RULES
- 4/4 ACCEPT (A + B + C + D) -> DONE (proceed to deliver)
- 3/4 ACCEPT + 1 COND-PASS -> fix the condition -> re-jury if fix is non-trivial
- ANY REJECT -> mandatory bond: fire ideas (>=5 candidates) -> fix -> FULL re-jury
- Never skip a full re-jury after a fix. Always re-run A + the 3 primaries.
- A backup seat counts only if its primary failed. It does not add a fifth vote.
- Safety cap: 20 rounds maximum. After round 20 -> pause + report to operator.

## BOND RULES (enforced, not advisory)
- honestscore + skill-ai-jury MUST fire in the SAME turn on every done-claim
- If honestscore fires alone -> VIOLATION
- If skill-ai-jury fires alone -> VIOLATION
- REJECT from any juror -> ideas fires immediately (no exceptions)
- UNVERIFIED honestscore dimension -> deep-thinker fires immediately
- ideas fix applied -> full jury re-run mandatory (never partial)

## PYTHON IMPLEMENTATION TEMPLATE

```python
import httpx
from concurrent.futures import ThreadPoolExecutor, as_completed

AR_BASE = "https://api.anonrouter.ai/v1"
AR_KEY = "ar_P1WEiQ4_POhL4f3fxYtP0MHOu98FBLF7XzHxFPwdJmy1qFkEqFA"

SYSTEM_PROMPT = """You are an adversarial reviewer. Be maximally critical. Non-optimistic scoring only.
Score 1-10 on pure mathematical criteria. No rounding up.
Reply format EXACTLY:
SCORE: N.N/10
VERDICT: ACCEPT | COND-PASS:condition | REJECT:blocker
DEFECTS: bullet list (empty if none)
REASONING: 1 paragraph"""

# Primaries only. Backups are listed separately and used on failure.
JURORS = [
    {
        "id": "B",
        "role": "Challenger",
        "model": "moonshotai/kimi-k3.1-pro-uncensored",
        "base": AR_BASE,
        "key": AR_KEY,
        "max_tokens": 3000,
        "backup_model": "z-ai/glm-5.3-uncensored",
    },
    {
        "id": "C",
        "role": "Arbiter",
        "model": "deepseek/deepseek-v4.1-flash-1017-uncensored",
        "base": AR_BASE,
        "key": AR_KEY,
        "max_tokens": 3000,
        "backup_model": "qwen/qwen-3.6-plus-uncensored",
    },
    {
        "id": "D",
        "role": "StressTester",
        "model": "tencent/hy4-uncensored",
        "base": AR_BASE,
        "key": AR_KEY,
        "max_tokens": 3000,
        "backup_model": "google/gemma-4-uncensored",
    },
]

def call_juror(juror: dict, artifact: str, task_desc: str, rubric: str, prior_rounds: str = "") -> dict:
    """Call a single juror. On empty/error, retry once with backup_model."""
    user_content = f"""TASK: {task_desc}

RUBRIC: {rubric}

DELIVERABLE:
{artifact}

PRIOR ROUNDS:
{prior_rounds if prior_rounds else "Round 1 - no prior rounds."}

Score this deliverable now."""

    def once(model: str) -> dict:
        payload = {
            "model": model,
            "max_tokens": juror["max_tokens"],
            "messages": [
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": user_content},
            ],
        }
        headers = {
            "Authorization": f"Bearer {juror['key']}",
            "Content-Type": "application/json",
        }
        with httpx.Client(timeout=60) as client:
            resp = client.post(f"{juror['base']}/chat/completions", json=payload, headers=headers)
            resp.raise_for_status()
            data = resp.json()
        content = data["choices"][0]["message"]["content"] or ""
        finish_reason = data["choices"][0].get("finish_reason", "unknown")
        return {"content": content, "finish_reason": finish_reason, "model": model}

    used = juror["model"]
    degraded = False
    result = once(used)
    if result["finish_reason"] == "length" or not result["content"].strip():
        used = juror["backup_model"]
        degraded = True
        result = once(used)

    content = result["content"]
    if result["finish_reason"] == "length" or not content.strip():
        return {
            "juror_id": juror["id"],
            "model": used,
            "error": f"Empty response (finish_reason={result['finish_reason']}).",
            "verdict": "ERROR",
            "degraded": degraded,
        }

    lines = content.strip().split("\n")
    score_line = next((l for l in lines if l.startswith("SCORE:")), "SCORE: 0.0/10")
    verdict_line = next((l for l in lines if l.startswith("VERDICT:")), "VERDICT: REJECT:parse-error")
    score = float(score_line.replace("SCORE:", "").replace("/10", "").strip())
    verdict_raw = verdict_line.replace("VERDICT:", "").strip()
    return {
        "juror_id": juror["id"],
        "role": juror["role"],
        "model": used,
        "score": score,
        "verdict": verdict_raw,
        "full_response": content,
        "finish_reason": result["finish_reason"],
        "degraded": degraded,
    }

def run_jury(artifact: str, task_desc: str, rubric: str, prior_rounds: str = "", round_num: int = 1) -> dict:
    """Run the 3 primary API jurors in parallel. Juror A is supplied by the caller."""
    print(f"\n[JURY] Round {round_num} - spawning 3 AnonRouter primaries in parallel...")
    results = []
    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = {
            executor.submit(call_juror, j, artifact, task_desc, rubric, prior_rounds): j
            for j in JURORS
        }
        for future in as_completed(futures):
            try:
                results.append(future.result())
            except Exception as e:
                juror = futures[future]
                results.append({
                    "juror_id": juror["id"],
                    "error": str(e),
                    "verdict": "ERROR",
                })
    accepts = sum(1 for r in results if r.get("verdict", "").startswith("ACCEPT"))
    rejects = sum(1 for r in results if r.get("verdict", "").startswith("REJECT"))
    cond_passes = sum(1 for r in results if r.get("verdict", "").startswith("COND-PASS"))
    return {
        "round": round_num,
        "results": results,
        "accepts": accepts,
        "rejects": rejects,
        "cond_passes": cond_passes,
        "converged": rejects == 0 and accepts >= 3,
    }
```

## GATE LINE FORMAT
ACCEPT: ai-jury: ACCEPT N.N (rndX 4/4 models:[kimi,deepseek,hy4])
COND-PASS: ai-jury: COND-PASS N.N (rndX 3/4 open:condition models:[...])
REJECT: ai-jury: REJECT N.N (rndX Y/4 block:reason models:[...])
DEGRADED: ai-jury: DEGRADED:backup-swap N.N (primary-failed:seat backup:model)

When jurors run via direct API calls, name the model ids, not session ids.
A backup swap is DEGRADED even if the verdict is ACCEPT.

## COST TRACKING
Provider is AnonRouter only. Token prices were not re-quoted in the 2026-11-19 probe.
Do not bill a round from the V6.5 the removed third-party lab rates. Log model id + token usage
from the API response when a round runs.

## VERSION HISTORY
V6.4: Placeholder jurors, no live API calls, manual scoring
V6.5: 4 live jurors (EL + AnonRouter), parallel execution, convergence loop bonded with honestscore
V7 (2026-11-19): the removed third-party lab removed. All API jurors on AnonRouter.
  New key. 6 models probed live (all HTTP 200 / OK).
  Primaries: kimi-k3.1-pro, deepseek-v4.1-flash-1017, tencent/hy4.
  Backups: glm-5.3, qwen-3.6-plus, gemma-4.
  GLM is backup only so primaries differ from a GLM fleet session.
  Padding block removed.
