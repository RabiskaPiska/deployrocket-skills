---
name: result-classifier
description: Classifies any probe, task, or agent output into one of four buckets: HIT, FREE, BAD, RETRY. Trigger on: after any probe run completes, after any agent returns output, after any task completion claim, when deciding what to do with a tool result, "classify this output", "is this a real finding". Prevents misclassification of false positives as hits and prevents discarding recoverable failures as permanent bad results.
---

# Result Classifier

## Trigger

Fire this skill AFTER any of these events:
- A subagent or seat returns its result
- A probe tool (nuclei, sqlmap, subfinder, burp, etc.) finishes a run
- A web-research agent returns findings
- A build or test run exits
- A deploy completes (or appears to complete)
- You are about to write a result into a report, brain note, or summary

Do NOT fire for trivial single-step tool calls (a `read` of a file, a `grep` that returned expected results). Fire when the output represents the conclusion of a meaningful unit of work.

## Instructions

### Phase 1 — Read the Raw Output

Before classifying, read at minimum:
- The last 50 lines of any log file
- The full return value of the agent/tool call
- Any exit code or status field

Never classify from a summary another agent wrote — read the raw output yourself.

### Phase 2 — Apply the Four-Bucket Test

Evaluate the output against each bucket's criteria IN ORDER. Stop at the first match.

---

#### Bucket 1: HIT

**Definition:** The output contains a confirmed, actionable, verifiable positive result.

**Criteria (ALL must be true):**
- The finding is specific (names a file, endpoint, CVE, exact value — not "something might be wrong")
- The finding is reproducible (you can describe the exact steps to reproduce it)
- The finding is NOT a known false-positive pattern (see BAD bucket)
- At least one of: you verified it manually, a second tool confirmed it, or the output explicitly states confirmed=true

**Examples:**
- Nuclei found a CVE-2024-XXXX with a matching HTTP response body
- SQLi probe returned a different response for `1=1` vs `1=2` AND extracted a DB version string
- Build succeeded with exit code 0 AND the expected output file exists and is non-empty
- Subdomain takeover confirmed: CNAME points to unclaimed S3 bucket AND S3 returns NoSuchBucket

**Action on HIT:**
1. Document with full evidence (tool output snippet, URL, payload, response)
2. Assign severity (if security: CVSS score; if build: blocker/non-blocker)
3. Write to the relevant brain note / report immediately (not "later")
4. If part of a multi-seat run: return to orchestrator as a confirmed result

---

#### Bucket 2: FREE

**Definition:** The output is a valid, useful secondary finding — not the primary target, but genuinely worth keeping.

**Criteria:**
- The result is real (not a false positive) but outside the original task scope
- It provides future value (e.g., an extra subdomain, an unrelated open port, a deprecated API endpoint)
- It does NOT require immediate action

**Examples:**
- Recon found a subdomain not in scope for this engagement but worth noting for later
- A build warning (not error) about a deprecated dependency
- A tool returned a low-severity informational finding alongside the primary results
- Research surfaced a tangentially related CVE that affects a different component

**Action on FREE:**
1. Log to a FINDINGS-FREE.md or the brain's relevant workspace note
2. Do NOT include in the primary report or deliverable
3. Do NOT block the current task on it
4. Tag with `[FREE]` and date so it can be reviewed later

---

#### Bucket 3: BAD

**Definition:** The output is a false positive, error, or genuinely useless result. Do not waste time on it.

**Criteria (ANY is sufficient):**
- The tool errored out (connection refused, timeout, invalid response) — this is NOT a finding
- The "finding" matches a known false-positive pattern (see list below)
- The result is a generic template match with no confirmed instance (nuclei matched a keyword but response body doesn't confirm exploitation)
- The output is a duplicate of a result already classified
- The agent hallucinated (cited a file that doesn't exist, a URL that returns 404, a line number that's wrong)

**Known false-positive patterns:**
- HTTP 200 with body "Not Found" (soft 404)
- Nuclei template match on a static text string in a non-exploitable context
- SQLi time-delay match where response time variance is >±500ms (network jitter)
- Subdomain "takeover" where CNAME points to an active service, not an unclaimed one
- Build "success" where the output binary/artifact does NOT exist on disk

**Action on BAD:**
1. Log briefly: what was tested, what came back, why it is BAD
2. Do NOT include in any report
3. If this was a probe: apply `/ideas` to design a better probe before retrying
4. If this was an agent result: note the hallucination pattern so the next agent prompt avoids it

---

#### Bucket 4: RETRY

**Definition:** The output is incomplete, ambiguous, or blocked — the task could succeed with a different approach.

**Criteria:**
- The tool ran but returned no results (empty output, 0 findings) — could be a config issue, not absence of finding
- The agent hit a rate limit, auth failure, or transient error
- The result is ambiguous (could be HIT or BAD but you cannot confirm without more data)
- The task ran against the wrong target, path, or parameter (scope error)

**Examples:**
- SQLmap returned 0 findings but you only tested GET params, not POST body
- Nuclei scan ran but skipped 80% of templates due to a tag filter misconfiguration
- Agent returned "I could not access the file" — file existed but path was wrong
- Research agent returned outdated results (2023 data for a 2026 question)

**Action on RETRY:**
1. Identify the SPECIFIC reason the attempt was incomplete (not "it didn't work")
2. Change ONE variable before retrying (different wordlist, correct path, POST instead of GET, updated query)
3. Maximum 2 retries on the same approach. Third attempt must use a fundamentally different method.
4. If still unresolved after 2 retries: escalate to BAD and log the gap

### Phase 3 — Write the Classification

Every classified result gets a one-line record in the run's output log:

```
[TIMESTAMP] [BUCKET] <short description> | evidence: <file/line/URL> | action: <taken>
```

Examples:
```
[14:32] [HIT] SQLi confirmed on /api/search?q= — DB version extracted | evidence: sqlmap-output.txt:L47 | action: written to report
[14:33] [FREE] Extra subdomain api-staging.target.com found | evidence: subfinder-out.txt:L12 | action: logged to FREE.md
[14:34] [BAD] Nuclei matched CVE-2024-1234 but response body shows patched version string | evidence: nuclei-out.txt:L88 | action: discarded
[14:35] [RETRY] Subfinder returned 0 results — ran without -all flag | evidence: empty output | action: re-run with -all
```

### Phase 4 — Aggregate Classification (multi-seat runs)

When classifying outputs from a full fan-out:

1. Collect all seat results
2. De-duplicate: same finding from two seats = one HIT, not two
3. Count by bucket: HIT/FREE/BAD/RETRY
4. If RETRY count > 30% of total: the task design is flawed — redesign before re-running
5. If BAD count > 50% of total: the probe/agent prompt is generating noise — invoke `/ideas` on the probe design

## References

- Related skills: `fleet-monitor` (detect stalls before classifying), `ideas` (design better probes on BAD/RETRY), `honestscore` (score final deliverable), `pentest` (security-specific classification guidance)
- False positive rules for SQLi: skill `sqli-fp-rules`
- Brain note destination: `cybersec/knowledge/` for security findings, `web-dev/knowledge/` for build findings

## Scripts

```bash
#!/bin/bash
# classify_quick.sh — emit a classification line from a tool output file
# Usage: bash classify_quick.sh <output_file> <bucket> "<description>"
TS=$(date +%H:%M)
FILE="$1"
BUCKET="$2"
DESC="$3"
LINES=$(wc -l < "$FILE" 2>/dev/null || echo "N/A")
echo "[$TS] [$BUCKET] $DESC | evidence: $FILE ($LINES lines) | classified: $(date -I)"
```

```python
# Aggregate bucket counts from a classification log
import re, sys
from collections import Counter

log = open(sys.argv[1]).read()
buckets = re.findall(r'\[(HIT|FREE|BAD|RETRY)\]', log)
counts = Counter(buckets)
total = sum(counts.values())
for b, n in counts.most_common():
    print(f"{b}: {n} ({100*n//total}%)")
print(f"TOTAL: {total}")
```