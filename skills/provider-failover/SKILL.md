---
name: provider-failover
description: Eligibility-policy provider failover for fleet LLM sessions - retry-safe/deadline/capability/quota gates, a shared retry budget across providers, per-provider circuit breakers, gradual restore with canary probes, and the config-plane swap procedure (0600 key files, {file:} refs, pre-swap backups, all config roots). Generalizes cortex-router beyond CortexAI and closes the stress-test finding that the rendered bootstrap bundle ships ZERO fallback providers. Fires on: provider error/timeout/quota-exhaustion, model selection at session start, canary failure, "swap the provider", "who is primary now", deploy of any bundle that names models, and the recurring insufficient_credits seat-killer.
alwaysFire: true
---

# provider-failover -- Eligibility-Policy Failover for Fleet LLM Providers

## 0. PURPOSE

Two measured failures motivate this skill:

1. MON1 (fleet monitoring run): 0/10 tasks completed on BOTH boxes because the
   provider was down -- and nothing detected it live. The outage was only diagnosed
   post-hoc ("harness exonerated" was written about a harness that had silently
   produced nothing).
2. The stress-tester verdict on the bootstrap bundle: "abliteratedllm is a single
   point of failure: bootstrap bundle renders zero fallback providers; U-QUOTA
   untested; recurring insufficient_credits killed subagent seats on both boxes"
   (runlog-J4-spotcheck). Zero fallback is not a rare event -- it is the rendered
   default when nobody runs this skill.

cortex-router solves runtime routing for CortexAI endpoints (primary/fallback with a
circuit breaker in code). This skill owns the rest of the surface: the eligibility
POLICY for what counts as an acceptable failover target, the shared retry budget so
failover cannot multiply cost, per-provider breakers where no code exists, the
config-plane SWAP procedure (which was executed by hand at least three times in one
campaign: CFG-SWAP, PAT rotation, juror pinning), and the canary probe that turns
"provider down" from a post-mortem finding into a 60-second detection.

## 1. WHEN TO FIRE

Fire immediately on ANY of:

- A provider call fails: timeout, 5xx, empty completion, quota error
  (insufficient_credits), repeated malformed output (thinking-retry class).
- Session start / model selection: before the first real call, know the primary AND
  the fallback order (never render a bundle with a single provider).
- User or orchestrator says: "swap the provider", "rotate the key", "who is
  primary", "provider is dead", "fail over".
- A canary probe fails or a heartbeat reports 0/10-with-no-exceptions (the MON1
  signature: zero completions everywhere means infrastructure, not tasks).
- Deploying any bundle/config that names providers: verify the fallback set is
  non-empty BEFORE the deploy ships (the rendered-bundle gate).
- Scheduled: canary probes every N minutes on long autonomous runs (cadence in 6.3).

### Always-fire justification

Provider outages and quota exhaustion break every task type (deploys, proofs,
harvests, config work) and fail silently -- the RGAP1 always-fire test. The campaign
evidence: provider-swap executed 3+ times with identical steps (CFG x7, T5 x6, C3 x6,
verdict-adjudication x6, rotation x5, final-report x7), one entire monitoring run
zeroed by an undetected outage, and a formal adversarial finding that the shipped
bundle had no fallback at all. Every session touches a provider; every session
therefore needs the eligibility policy resident, which is why this is always-fire
rather than situational.

## 2. THE MODEL

### 2.1 Provider registry (declare it per session, before first call)

```
| field          | meaning                                          |
|----------------|--------------------------------------------------|
| id             | provider label (e.g. glm, abliteratedllm, anon)  |
| endpoint       | base URL + auth style (Bearer vs Basic)          |
| capability     | models served, context window, tool-call support |
| quota_state    | live: ok | low | exhausted | unknown             |
| deadline_class | can it finish a long call? (streaming? timeout?) |
| retry_safety   | is a retry idempotent here (no double-charge, no dup side-effect) |
| breaker        | CLOSED | OPEN_UNTIL(<ts>) | HALF_OPEN(canary%)   |
```

Unknown quota_state is treated as LOW (conservative), never as ok. "Unknown" as a
pass is the same defect class as "could not measure" = pass.

### 2.2 Eligibility policy (the four gates, checked in order)

A provider is ELIGIBLE for a call only if ALL pass:

1. RETRY-SAFE: the call can be retried without duplicating a side-effect. Writes,
   sends, and paid generations need idempotency keys or idempotent semantics; pure
   reads/completions always pass.
2. DEADLINE: provider can plausibly complete inside the task's deadline (a provider
   that streams at 2 tok/s does not take a 10-minute deadline shard).
3. CAPABILITY: the model on that provider supports what the call needs (context
   size, tool calls, JSON mode, vision). A model that silently degrades (drops
   tools, truncates) is INELIGIBLE, not a fallback.
4. QUOTA: quota_state is not exhausted, and spending it does not breach the run's
   declared budget (pairs with skill-autonomous-run-budget).

Order matters: retry-safety before deadline before capability before quota -- a cheap
provider you cannot safely retry is worse than an expensive one you can.

### 2.3 Shared retry budget (failover must not multiply cost)

```
BUDGET_total_retries   = 6 per task (hard cap, all providers combined)
BUDGET_per_provider    = 2 consecutive failures -> breaker opens
BUDGET_wall_clock      = declared in the run's budget YAML; breaker at 80%
Rule: a failover chain may never spend more total attempts than
      BUDGET_total_retries, no matter how many providers are eligible.
```

Why shared, not per-provider: the measured failure mode ($59/3h) was per-seat
runaways that nobody aggregated. A per-provider budget alone lets 3 providers x 6
retries = 18 silent attempts. The budget is GLOBAL to the turn/run.

### 2.4 Circuit breaker (per provider)

```
CLOSED      normal; failures counted per consecutive-failure, not lifetime
OPEN        after 2 consecutive failures (per provider); stay OPEN for
            COOLDOWN = 5 min (first trip), 15 min (second), 15 min + halve
            traffic (third trip in an hour = provider demoted for the session)
HALF_OPEN   after cooldown: canary probes only (see 6); N consecutive canary
            passes restore CLOSED, gradually (7)
State is per config-root per box (fleet roots do not share breaker memory --
the dual-root law applies here too: flip breakers in EVERY root or the state lies)
```

## 3. THE FAILOVER PROCEDURE (runtime path)

```
3.1 Failure observed (error, timeout, or canary trip):
    - classify the error first (result-classifier: HIT / FREE / BAD / RETRY):
      BAD (auth 401, model-not-found) -> do NOT retry the same provider; go to 3.3
      RETRY (timeout, 5xx, empty completion)  -> one retry, then 3.3
      QUOTA (402/insufficient_credits)        -> do NOT retry; go 3.3
3.2 Retry ONCE on the same provider only if retry-safe AND the error was transient
    (timeout/5xx). Never retry a QUOTA or AUTH failure.
3.3 Mark the provider's breaker: 2 consecutive fails -> OPEN_UNTIL(now+cooldown).
3.4 Pick the next provider by the eligibility policy (2.2) against the registry.
3.5 Spend from the SHARED retry budget; log each attempt: provider, error class,
    latency. A failover without a log line is invisible to the post-mortem.
3.6 If NO provider is eligible: STOP the run (fail loud), emit the outage signal
    (fleet-monitor beat, "PROVIDER-POOL-EXHAUSTED"), and write the state_file with
    resume instructions. NEVER fall back to "answer from memory" for a task that
    was defined as needing live calls.
3.7 After restore: gradual restore (7) -- never snap back to full traffic on the
    first good response.
```

## 4. THE CONFIG-PLANE SWAP (change who is primary; the campaign's repeated op)

This is the generalized provider-swap from RGAP1 candidate 6 (CFG x7, T5 x6, C3 x6,
final-report x7). Steps, in order:

```
4.1 INVENTORY the current state first (read-only):
    - grep -r "provider" <each config root>            # find every root
    - list every {file:} key reference and its 0600 key file
    - record current primary + the exact jsonc blocks you are about to change
4.2 BACKUP every file you will touch:
      cp config.jsonc config.jsonc.pre-swap-$(date +%s)
      cp /path/keyfile /path/keyfile.pre-swap-$(date +%s) --Archive mode 0600
4.3 WRITE the new provider config to ALL roots on ALL boxes (dual-root law):
    - key material NEVER inline: write key to file with mode 0600, reference as {file: ...}
    - apply to ~/.config/opencode AND the dotroot on dual-root boxes
    - one JSONC round-trip parse per root before declaring anything
4.4 PROVE the swap with live evidence, not config diffs:
      HTTP 200 on GET /v1/models with the new key
      one REAL completion round-trip with the chosen model (a models-list 200
      does NOT prove the completion route works -- provider no-route failures
      are silent on some gateways)
    A swap without a completion probe is UNVERIFIED, never "done".
4.5 RESTART or hot-reload whatever consumes the config; re-verify through the
    consumer (a healthy endpoint behind a stale consumer is a false green).
4.6 CANARY: send one cheap real task through before re-pointing crews.
4.7 RECORD: what changed, old values (for rollback), new md5s, in the runlog.
```

Rollback = restore the .pre-swap backups, restart, re-probe. The backup step (4.2)
is what makes rollback possible at all; skipping it has cost full restarts before.

## 5. CANARY PROBES (detect the outage before the work does)

```
5.1 Probe (every 10-15 min on long runs; on-demand otherwise):
    GET  <base>/v1/models   with auth -> expect 200 and the expected model id present
    POST <base>/v1/chat/completions -> 1-token prompt, max_tokens=1
                                       -> expect 200 and non-empty choice text
5.2 Frequency: session start (mandatory), every 10-15 min on long runs, immediately
    after any 5xx/quota error, and before every deploy of a rendered bundle.
5.3 Classification (result-classifier): 200 + plausible body = HIT; 200 + empty/
    wrong shape = BAD (treat as failure); timeout/5xx/402 = RETRY-then-DEAD.
5.4 Two consecutive canary failures = provider OPEN; emit a fleet-monitor beat with
    the probe evidence (status code + latency), not "it seems slow".
5.5 The MON1 lesson, encoded: a scheduled batch that returns 0/N with ZERO errors
    is an outage signature, not a quiet success. Any 0/N result triggers a manual
    canary before the run is believed.
```

## 6. VERIFICATION COMMANDS (paste output, never summarize)

```bash
# 6.1 Canary: models list
curl -sS -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $(cat /path/key)" \
  https://<provider>/v1/models

# Canary: real completion (the only proof the route works)
curl -sS -H "Authorization: Bearer $(cat /path/key)" -H "Content-Type: application/json" \
  -d '{"model":"<model-id>","max_tokens":8,"messages":[{"role":"user","content":"ping"}]}' \
  https://<provider>/v1/chat/completions | python3 -c "import json,sys; r=json.load(sys.stdin); print(r['choices'][0]['message']['content'][:40])"

# Config validity after a swap (per root, per the dual-root law):
python3 -c "import json5,json;print(len(json5.load(open('config.jsonc'))))"  # or jsonc parser of choice
grep -n "provider" ~/.config/opencode/config.jsonc | head
# Law-grep the 7 invariants from the config law on EVERY root you touched.

# Key file hygiene:
stat -c "%a %n" /path/keyfile        # expect 600
grep -r "api_key" ~/.config/opencode/config.jsonc   # expect {file:...} refs, not literals

# Breaker state (if persisted):
cat /tmp/provider-breakers.json 2>/dev/null || echo "no breaker state (all CLOSED)"
```

Windows equivalents: `curl.exe -sS -o NUL -w "%{http_code}"` for status checks;
`Get-Content`, `Get-Acl` for key-file permission checks (expect Administrator-only
read on Windows equivalents of 0600).

## 7. GRADUAL RESTORE (HALF_OPEN discipline)

```
On first successful canary after cooldown:
  - route 10% of new calls (or 1 canary seat) to the restored provider
  - after 3 consecutive clean canaries: 50%
  - after 3 more clean minutes: 100%, breaker CLOSED
Skip-ahead is FORBIDDEN for providers that tripped twice in one hour; those serve
only canary probes for the rest of the session.
Rationale: the measured failure pattern was flapping (works, dies, works) -- a snap
back to full traffic re-hides the flap for exactly the length of the next long run.
```

## 8. KNOWN-FAILURE CHECKLIST

| Symptom | Root cause (measured) | Fix |
|---|---|---|
| 0/10 tasks complete on both boxes, no errors logged | Provider dead; consumers fail silently post-hoc | Canary at start (5.1) + the 0/N rule (5.5); fail the run FAST instead of exonerating a dead harness |
| Bundle deployed with zero fallback | Nobody rendered the fallback set before deploy | Pre-deploy check (9.1): >= 2 eligible providers or the deploy BLOCKS |
| insufficient_credits mid-shard, seat dies | Quota gate absent from eligibility | QUOTA gate (2.2-4); mark provider low after first 402, do not retry |
| Swap "worked" but completions 404 | no-route silent failure on that provider | The completion probe in 6 (models-list 200 is NOT proof) |
| Auth fails after key rotation on one box only | Dual-root drift: one root updated, one stale | dual-root law: every change to ALL roots; JSONC round-trip + law-grep per root |
| Failover triples the cost | Per-provider retries without a shared budget | Shared retry budget (2.3); global counter in the ledger |
| Provider recovers, then dies again in 20 min | Snap-back at 100% on first success | Gradual restore (this section, 7) |
| Key file readable group/world | Copied without preserving mode | chmod 600 (or Windows ACL) at write time; verify with stat/Get-Acl every swap |
| "Provider healthy" but every reply truncated | Deadline class mismatch (model/stream mismatch) | DEADLINE gate; probe with a deadline-shaped prompt, not max_tokens=1 only |

## 9. RELATION TO EXISTING SKILLS (dedup)

- cortex-router: runtime routing/failover CODE for CortexAI/GateAI endpoints.
  provider-failover is the general skill: same breaker/restore mathematics, plus the
  eligibility policy and the config-plane swap cortex-router does not cover. Cortex
  deployments should still use cortex-router as the in-process implementation and
  this skill as the policy + swap layer.
- provider-swap-failover candidate (RGAP1 #6): this skill IS that candidate,
  promoted and generalized; it dedups by absorbing it.
- skill-circuit-breaker: cost/time breakers for RUNS. This skill's breakers are per
  PROVIDER. Both may fire in one run; their budgets are different axes.
- mcp-ops: MCP server health, not LLM providers. Different surface; both may matter
  in the same deploy.

## 10. EVIDENCE REQUIREMENTS (per failover event)

A failover or swap is "done" only with all of:

1. The eligibility table as it stood at decision time (which gates each provider
   passed/failed, with the probe output as evidence).
2. The shared-retry-budget ledger line (attempts per provider, total spend).
3. The config diff (pre-swap vs post-swap) + backup paths + their hashes.
4. The post-swap completion probe output (models 200 AND a real completion body).
5. The gate line naming what fired (see 8).

"Eligibility was obvious" without the table is an overclaim; the table is the work.

## 11. SELF-TEST (attack your own gate; ~10 min, no LLM spend)

```
11.1 Kill the primary endpoint (or point SERVER at a dead port) and fire the
     canary. EXPECT: two failures -> breaker OPEN in the registry file -> the
     failover picks the secondary. If the session continues silently on the dead
     provider, the gate is inert -- that is the MON1 defect reproduced; fix before
     trusting anything else.
11.2 Expend the shared budget deliberately (6 failures across providers). EXPECT:
     the run STOPS (fail-closed) with STOP reason in the ledger, not "one more
     retry" forever.
11.3 Restore the primary mid-cooldown and probe. EXPECT: HALF_OPEN gradual
     restore, not a snap to 100%.
11.4 Corrupt one root's config on a dual-root box (bad jsonc). EXPECT: the
     round-trip parse gate FAILS the swap; the other root's good state is the
     rollback.
```

A breaker that has never tripped in a test does not exist (measured-failure rule 2).

## 12. GATES LINE USAGE

Every turn running this skill ends with the canonical 14-field line (producer rule:
canonical field order, values never contain "|", not-fired = "skipped: <why>",
n/a needs a one-line why). The fields this skill most often carries:

```
gates -- godmode:activation: fired | promptgod: skipped: subagent turn | subagents: <N seats> | honestscore: <N.N> | ideas: fired | deep-thinker: skipped: runtime op, no research claim | dim-correctness: <N.N> | dim-completeness: <N.N> | dim-robustness: <N.N (fallback coverage)> | dim-security: <N.N (key perms)> | dim-performance: <N.N> | dim-concurrency: <N.N (per-seat breakers)> | dim-docs: <N.N> | brain: <files written>
```

Minimum to close a swap turn: registry table, backup hashes, both probe outputs.
Minimum to close a failover turn: the retry-budget ledger line + the restore state.

## 13. WORKED EXAMPLE (composite from the campaign's real incidents)

```
Session start, registry built:
  glm        quota=ok      retry=safe   deadline=ok  cap=ok    breaker=CLOSED
  abliteratedllm quota=low retry=safe   deadline=ok  cap=ok    breaker=CLOSED
  anonrouter quota=unknown retry=safe   deadline=ok  cap=tools? breaker=CLOSED
Turn 3: primary returns 522 twice -> breaker OPEN_UNTIL(+5min); shared budget -2.
Failover: abliteratedllm passes all four gates; canary completion OK; traffic moves.
Post-mortem line in ledger: provider=glm attempts=2 fail=consecutive-2 state=OPEN.
The MON1 difference: with this skill, the 0/10 night would have ended at minute 1
with "PROVIDER-POOL-EXHAUSTED" instead of post-hoc archaeology.
Config swap variant (PAT rotation): backups hashed, both roots patched, models 200,
completion probe returned "ok" -> swapped, seat re-pinned, md5 recorded, done.
```

## 14. VERSION / CHANGELOG

```
provider-failover v1.0 (2026-11-12)
- Authored from runlog-RGAP1 candidate #6 (provider-swap-failover), runlog-RGAP2
  (top-harness spend/kill-switch doctrine), runlog-J4-spotcheck (stress-tester
  finding: bundle rendered zero fallback providers), MON1 undetected-outage class.
- Generalizes cortex-router (runtime, CortexAI-specific) into a config+policy skill
  valid for any provider set; cortex-router remains the in-code router where present.
- Eligibility gates encode the measured silent-failure trio: quota (402), no-route,
  and deadline-sampled probes.
```

## 15. REGISTRY FILE FORMAT (persisted, per box, per root)

Registry is a file, not a mental model. Format (JSON, one per root):

```json
{
  "providers": {
    "glm": {
      "endpoint": "https://api.example.com",
      "auth": "bearer",
      "key_file": "/home/rabi/.keys/glm.key",
      "key_file_mode_ok": true,
      "models": ["glm-5.3-flash-abliterated"],
      "tool_calls": true,
      "context_window": 200000,
      "quota_state": "ok",
      "breaker": {"state": "CLOSED", "consecutive_failures": 0, "open_until": null},
      "last_probe": {"at": "2026-11-12T13:22:04Z", "models": 200, "completion": 200, "latency_ms": 640}
    },
    "abliteratedllm": {
      "quota_state": "low",
      "note": "402 seen 2026-11-12T09:14Z; halve traffic until next top-up",
      "breaker": {"state": "CLOSED", "consecutive_failures": 0}
    }
  },
  "shared_retry_budget": {"cap": 6, "used_this_task": 0},
  "declared_wall_clock_budget": "2h",
  "breaker_trip_at": "80%"
}
```

Rules:
- The registry is written BEFORE the first call of a session (2.1) and updated on
  every state change. A registry older than the config it describes is stale --
  rebuild it (it is derived state; rebuilding is cheap and safer than trusting).
- quota_state transitions: ok -> low on first 402/insufficient_credits; low ->
  exhausted on second; exhausted providers are ineligible for the rest of the
  session unless a top-up is confirmed by a successful canary, not by hope.

## 16. MULTI-BOX DEPLOYMENT OF A PROVIDER CHANGE

```
16.1 Enumerate boxes: VPS1, VPS2, PC (and any RC/live session boxes); enumerate
     config roots per box (dual-root law: ~/.config/opencode + dotroot where it
     exists; "VPS2 dotroot legitimately has no mcp section" -- absence of a section
     is not drift, but a DIFFERENT provider block in a root IS).
16.2 Stage the change as a script; scp it; run via ssh python3 file.py (never
     quoting-stacked bash -c over ssh -- the pkill-self-kill class of traps).
16.3 Apply per root with round-trip parse; collect per-root md5s.
16.4 Probe from EACH box (network paths differ -- the inter-DC outage proved a
     healthy endpoint from one host and a dead route from another simultaneously).
16.5 Record per-host results in the runlog table; any red cell = no rollout.
16.6 Only then re-point crews/seats; keep the old provider pinned as fallback in
     the rendered bundle (the J4 finding: fallback must be IN the render, not in
     the operator's head).
```

## 17. RENDERED-BUNDLE PRE-DEPLOY CHECK (blocks the deploy)

Before any bundle/bootstrap that renders provider config ships:

```bash
# 9.1 fallback-coverage gate: count DISTINCT providers in the rendered output
python3 - <<'EOF'
import json, sys
cfg = json.load(open('rendered-config.json'))   # the bundle's rendered provider block
providers = cfg_providers = cfg.get('providers') or cfg.get('provider', [])
if isinstance(providers, dict): providers = list(providers)
eligible = [p for p in providers if cfg[p].get('quota_state') in ('ok','low')]
print(f"providers={len(providers)} eligible={len(eligible)}")
if len(eligible) < 2:
    print("FALLBACK-COVERAGE-FAIL: bundle has zero/one fallback")
    sys.exit(2)
EOF
```

Exit 2 = deploy blocked. This is the direct closure of the stress-tester finding:
"bootstrap bundle renders zero fallback providers" becomes a red gate instead of a
verdict line.

## 18. OUTAGE SIGNALS AND ESCALATION

| Signal | Meaning | Action |
|---|---|---|
| PROVIDER-POOL-EXHAUSTED | no eligible provider | stop the run, state_file + fleet-monitor beat, page the operator only if a manual action is needed (top-up/credential) |
| PROVIDER-DEGRADED | breaker OPEN on primary, secondary serving | continue on secondary; log; no page |
| PROVIDER-FLAPPING | >= 2 trips in 1h | demote for session; canary-only; note in runlog |
| KEY-PERMS-FAIL | 0600/ACL check failed | fix perms BEFORE any call; keys readable by group = rotate, not chmod-and-forget |

Escalation inherits the heartbeat rule: an outage beat with no side-effect evidence
in two consecutive beats escalates itself (the monitor re-emits with REAL probe
output attached).

## 19. GLOSSARY

- eligibility gate: one of retry-safe / deadline / capability / quota checks that a
  provider must pass to receive traffic.
- shared retry budget: the global per-task cap on total attempts across providers.
- breaker: per-provider failure state machine (CLOSED / OPEN / HALF_OPEN).
- canary: the minimal live probe (models 200 + 1-token completion) that proves a
  route end-to-end.
- gradual restore: 10% -> 50% -> 100% traffic return after cooldown.
- demotion: a provider restricted to canary-only for the rest of the session.
- dual-root law: config changes apply to ALL config roots on ALL boxes or the change
  did not happen.

## 20. FAQ

Q: Is this just cortex-router with more prose?
A: No. cortex-router is runtime code for one provider family. This skill adds the
eligibility policy (the part that decides whether a fallback is ALLOWED), the shared
budget, the config-plane swap, and the pre-deploy fallback gate. Where cortex-router
runs, it implements this skill's breaker math in-process; where it does not, the
skill is the whole implementation.

Q: Why 2 consecutive failures, not 1?
A: One transient 522 during the campaign's real inter-DC flap (E1) was common; two
consecutive was the reliable "this is real" signal. Single-failure trip would have
thrashed the fleet during the measured 1h+ return-path outage.

Q: Why is the models-list 200 not enough?
A: Measured: provider gateways exist where /v1/models answers 200 but the completion
route has no binding for the requested model (provider.no-route, silent). Only a
completion round-trip proves the pair (provider, model) works.

Q: Where do breaker states persist?
A: Per root in the registry file. Fleet roots do not share memory; that is a feature
(blind-box independence) and a trap (a breaker flipped in one root is a lie in the
other -- the dual-root law again).

Q: The task must have live data and no provider is eligible.
A: Fail loud (3.6). "Answer from memory" for a live-data task is fabrication, the
worst outcome available. The state_file + beat is the honest terminal state.
```

## 21. PER-SEAT FAILOVER ON CREWS (concurrency field, concretely)

Crews multiply both the spend and the blast radius of a dead provider:

```
- Each seat starts with a registry COPY and reports breaker transitions in its beat.
- The orchestrator aggregates: 3+ seats tripping the same provider within one beat
  = immediate fleet-wide OPEN for that provider (do not wait for each seat to
  independently discover the outage -- that is the 0/10 night again).
- Seat retry attempts all draw from the shared budget; the orchestrator owns the
  counter, seats request from it (no seat may retry locally beyond its slice).
- Model pinning (juror pinning, per-seat provider pins from the campaign): a pin is
  an eligibility OVERRIDE and must be re-validated by canary at pin time; a pin to a
  provider whose breaker is OPEN fails the seat assignment, loudly.
```

## 22. RELATION TO DEPLOYS (blast-radius pairing)

- Before deploy: the fallback-coverage gate (section 17) blocks a bundle with < 2
  eligible providers.
- After deploy: first canary through the CONSUMER (4.5), then the 10% gradual
  restore if the deploy touched a previously-OPEN provider.
- skill-blast-radius-rollback owns the deploy watchdog and the rollback swap; this
  skill owns provider eligibility. Both gates must pass before a deploy is called
  live; a deploy that passed blast-radius checks but shipped zero fallback is the
  exact measured defect this pairing exists to block.

## 23. METRIC RULES (what this skill refuses to do)

- Never judge provider health by latency alone (a fast 402 is still dead).
- Never average success rates across models; a provider can be healthy for one
  model id and dead for another (per-model rows in the registry).
- Never let a single successful probe mark restore (gradual restore is mandatory).
- Never record "provider ok" from a stale registry entry; registry older than the
  last probe is UNKNOWN, and UNKNOWN is treated as low, not ok.
