# probe-runner

## Purpose
Automates the execution and scoring of routing probes against the fleet's skill/agent system. A probe fires a test prompt, observes which agent responds, which skills load, which vault tools are offered, and whether the result is correct. The probe-runner aggregates results across the 500-probe matrix, flags double-routes and misfires, and produces a scored report.

## Trigger Conditions
- User says: "run probes", "fire these agents", "test the routing", "run the probe matrix", "check routing accuracy"
- After any change to AGENTS.md, a skill's trigger-condition section, or the fleet prefix
- Before a major fleet deployment (pre-flight routing gate)
- Weekly maintenance item (routing regression check)

---

## Probe Format

Each probe is a single record:

```json
{
  "probe_id": "P-0042",
  "category": "security",
  "input": "test for SQL injection on the login form",
  "expected_agent": "pentest",
  "expected_skills": ["sqli-specialist", "pentest"],
  "expected_vault_tools": ["sqli-surgeon", "sqli-shapeshifter"],
  "must_NOT_fire": ["general", "explore"],
  "transcript_required": ["T1", "T2", "T3"]
}
```

Fields:
- `probe_id` — unique identifier (P-0001 … P-0500)
- `category` — routing category (see matrix below)
- `input` — the verbatim prompt sent to the agent
- `expected_agent` — the agent name that MUST handle this probe
- `expected_skills` — skill IDs that MUST appear in the session's skill stack
- `expected_vault_tools` — vault tool names that MUST be offered (present in tool list, not necessarily called)
- `must_NOT_fire` — agent names that must NOT handle this probe (misfire list)
- `transcript_required` — which transcript tiers to capture (T1=summary, T2=skill-load events, T3=full tool-call log)

---

## The 500-Probe Matrix Structure

Probes are distributed across 10 routing categories, 50 probes each:

| Category ID | Category name | Primary agent(s) | Key skills tested |
|---|---|---|---|
| C01 | SQL injection | pentest | sqli-specialist, sqli-fp-rules |
| C02 | XSS / CSP | pentest | xss-specialist, csp-bypass |
| C03 | Active Directory | pentest | active-directory-attack |
| C04 | Cloud attack | pentest | cloud-attack |
| C05 | Recon / surface mapping | pentest | recon-specialist, subdomain-enum |
| C06 | Web design / UI | site-fixer, site-judge | refero-design, impeccable |
| C07 | Library docs lookup | general | context7, gsap-core |
| C08 | Codebase exploration | explore | (direct tools) |
| C09 | Maintenance / fleet health | general | maintenance-scheduler, mcp-health-checker |
| C10 | Research / deep-dive | general | deep-research-plus, subagents |

Each category has 50 probes spanning:
- 10 canonical / happy-path probes (clear intent, expected to route cleanly)
- 15 boundary probes (ambiguous intent, edge-case phrasing)
- 15 adversarial probes (designed to confuse routing — wrong keywords, mixed domains)
- 10 regression probes (historically misrouted; must stay correct after fixes)

---

## Transcript Tiers

| Tier | What it captures | When to use |
|---|---|---|
| T1 | Agent name fired, top-3 skills loaded, final answer present (y/n), latency | All 500 probes |
| T2 | Full skill-load event log, vault tool offer list, skill trigger reason | Sample of 50 probes (10%) or any FAIL |
| T3 | Complete tool-call sequence, all parameters, all raw outputs | Any double-route or misfire |

---

## Running Probes

### Single probe (manual)
```
Skill("probe-runner")
PROBE: P-0042
INPUT: "test for SQL injection on the login form"
```
The runner fires the input, observes the agent, captures T1+T2, and scores.

### Batch run (automated, using subagents)
```
Skill("subagents")
# Spawn N subagents (floor 5, ceiling 30 per subagents skill)
# Assign each a slice of the 500-probe matrix
# Each subagent returns a T1 row per probe
# Orchestrator merges and scores
```

Recommended batch sizes: 25 probes per subagent × 20 subagents = 500 probes in one wave.

### Category-only run
```
run probes category:C01
```
Fires only the 50 C01 probes. Useful after editing sqli-specialist or sqli-fp-rules.

---

## Scoring a Probe

Each probe gets a binary score:

| Condition | Score |
|---|---|
| Correct agent fired AND all expected_skills loaded AND no must_NOT_fire agent present | **PASS** |
| Wrong agent fired (misfire) | **FAIL — misfire** |
| Correct agent but missing ≥1 expected_skill | **FAIL — incomplete skill load** |
| Correct agent + skills, but a must_NOT_fire agent also fired | **FAIL — double-route** |
| No agent fired / timeout | **FAIL — no response** |

Score the batch:
```
accuracy = PASS_count / total_probes
```
Target: **≥96% accuracy** across the full 500-probe matrix before any fleet-wide prefix deploy.

---

## Detecting Double-Routes

A double-route occurs when two agents both claim to handle the same probe. Detection:

1. In the T1 transcript, look for more than one `agent_fired` entry per probe_id.
2. In T3, look for skill-load events from two distinct agent namespaces within the same session turn.

When a double-route is detected:
1. Capture T3 for that probe.
2. Identify which trigger condition in each agent's SKILL.md matches the input.
3. Add a negative example to the lower-priority agent's trigger section: `NOT for: <specific phrasing>`.
4. Re-run the probe to confirm resolution.
5. Log to `_ops/DOUBLE-ROUTE-LOG.md`: probe_id, input, both agents, root cause, fix applied.

---

## Detecting Misfires

A misfire is when the wrong agent handles a probe — e.g., `general` fires on a security probe instead of `pentest`.

Detection:
1. `agent_fired != expected_agent` in T1.
2. Or: `agent_fired` is in `must_NOT_fire` list.

Resolution:
1. Read the misfired agent's SKILL.md trigger section.
2. Identify the keyword/phrase that caused the wrong routing.
3. Either: add a `NOT for:` exclusion to the misfired agent, OR strengthen the correct agent's trigger keywords.
4. Re-run; confirm fix.
5. Move the probe to the regression bucket so it stays in future runs.

---

## Probe Result Table Format

After a run, emit:

```
PROBE RUN RESULTS — 2026-10-31T11:00:00Z
  Matrix        : FULL 500 (or "C01 only: 50")
  Runner        : 20 subagents × 25 probes
  Duration      : 4m 38s

  Category  | Total | PASS | FAIL-misfire | FAIL-skill | FAIL-dbl | FAIL-noResp | Accuracy
  ----------|-------|------|--------------|------------|----------|-------------|--------
  C01 SQL   |  50   |  49  |      0       |     1      |    0     |      0      |  98.0%
  C02 XSS   |  50   |  48  |      1       |     1      |    0     |      0      |  96.0%
  C03 AD    |  50   |  50  |      0       |     0      |    0     |      0      | 100.0%
  C04 Cloud |  50   |  47  |      2       |     1      |    0     |      0      |  94.0% ⚠️
  C05 Recon |  50   |  50  |      0       |     0      |    0     |      0      | 100.0%
  C06 Design|  50   |  49  |      0       |     0      |    1     |      0      |  98.0%
  C07 Docs  |  50   |  50  |      0       |     0      |    0     |      0      | 100.0%
  C08 Explore| 50   |  50  |      0       |     0      |    0     |      0      | 100.0%
  C09 Maint |  50   |  49  |      0       |     0      |    1     |      0      |  98.0%
  C10 Deep  |  50   |  50  |      0       |     0      |    0     |      0      | 100.0%
  ----------|-------|------|--------------|------------|----------|-------------|--------
  TOTAL     | 500   | 492  |      3       |     3      |    2     |      0      |  98.4% ✅

  GATE: ✅ PASS (≥96% required; 98.4% achieved)
  Failures to investigate: P-0112 (C04 misfire), P-0113 (C04 misfire), P-0214 (C02 misfire)
                           P-0089 (C01 skill), P-0156 (C02 skill), P-0267 (C04 skill)
                           P-0301 (C06 double), P-0445 (C09 double)
```

---

## Probe Library Storage

Probe definitions live at:
- VPS: `/srv/dr/brains/probe-library/probes-C01.json` … `probes-C10.json`
- PC: `C:\Users\rabim\OneDrive\Desktop\AI Stuff\agency-brain\probe-library\`
- GitHub: `deployrocket-central-brain/probe-library/`

New probes are added to the library (never deleted — moved to `archived/` if retired). The regression bucket (`probes-regression.json`) is append-only.

---

## Quick-Run Shortcuts

| Command | What it runs |
|---|---|
| `run probes full` | All 500 probes, 20 subagents |
| `run probes category:C01` | 50 C01 probes only |
| `run probes regression` | Regression bucket only (~50 probes) |
| `run probes boundary` | All 150 boundary probes across categories |
| `run probes adversarial` | All 150 adversarial probes |
| `run probes failed` | Re-run only the FAILs from the last run |
| `probe P-0042` | Single probe with T1+T2+T3 |

---

## Notes
- Probes are read-only against production systems — they test routing logic only, not live external targets.
- Never include real credentials, real domain names, or real CVE exploit payloads in probe inputs; use placeholder names (`example.com`, `TARGET`, `REDACTED`).
- The 500-probe target is the full-matrix gate. Weekly maintenance uses the 50-probe regression bucket only (faster, catches regressions without the full cost).
- Add a probe to the regression bucket whenever a misfire or double-route is fixed — so the fix stays fixed.
