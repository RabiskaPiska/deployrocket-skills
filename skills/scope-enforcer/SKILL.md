---
name: scope-enforcer
description: Deny-by-default scope enforcement as CODE for pentest and scoped-engagement sessions - the B02 master-table gap closed. A PreToolUse hook matches every Bash/tool call (exact + wildcard) against scope.yaml (in-scope hosts, CIDRs, domains, URL paths, filesystem paths, processes); out-of-scope calls are BLOCKED BEFORE the call fires; blocked attempts are logged to an append-only audit; read-only allowlist rules exist for recon; and a daily self-attack (feed the gate an out-of-scope call, prove it blocks) keeps the guard real. Closes the biggest pentest-loop gap - prompts alone are bypassable, and one out-of-scope curl from an agent is an incident. alwaysFire variant for pentest sessions (fires on every tool call), plus a lighter alwaysFire workspace mode (deny writes outside the workspace).
alwaysFire: true
---

# scope-enforcer - deny-by-default scope gate as CODE

You are running **scope-enforcer**. Every tool call in this session passes
through a mechanical scope gate BEFORE it fires. The gate reads scope.yaml,
matches the call (exact + wildcard), and blocks anything out of scope with
a logged, auditable refusal. Prompts ask agents to stay in scope; the hook
makes staying in scope the only executable path.

**Why this exists (the measured failure class):** the master table calls
scope enforcement "the single biggest pentest-loop gap (B02, P2, DR2/DR9)"
- the loop's prompts say "stay in scope" and nothing enforces it. An agent
that misparses a target, follows an out-of-scope redirect, or fat-fingers
a CIDR can touch a system nobody authorized. Every serious 2026 agent
framework has converged on the same answer: mechanical enforcement at the
tool layer (deny rules, hook gates), not prompt discipline. This skill is
the procedure + the config schema + the self-test.

**What this skill is NOT:**

- NOT a replacement for the scope document. scope.yaml is DERIVED from the
  signed scope document (burp-operator rule: never operate without one);
  the hook enforces, the document authorizes.
- NOT a sandbox (see sandbox-isolation for runtime isolation). The gate
  says "this call may not run"; it does not contain a compromised call.
- NOT optional in pentest sessions. A session that fires a tool call
  without the gate loaded is out of compliance - the gates line reports
  `scope: gate not loaded` as a finding, not a footnote.

---

## 0) HARD RULES (survive compaction)

1. **Deny-by-default.** Anything not matched by scope.yaml is BLOCKED. An
   allowlist default ("allow unless denied") is the failure mode this
   skill exists to kill - the internet is bigger than your allowlist.
2. **The gate is code, not prose.** PreToolUse hook, deterministic match,
   no model judgment in the enforcement path. Model judgment lives in the
   RULES (what goes in scope.yaml), the hook enforces them mechanically.
3. **Fail-closed on error, fail-open never.** If scope.yaml is missing,
   unreadable, or the hook crashes: BLOCK the call (pentest sessions) -
   a broken scope gate must not become a permit-all. (Workspace mode may
   choose fail-open for reads; pentest mode never does.)
4. **Blocked attempts are logged, not hidden.** Every block writes one
   append-only audit line (ts, session, tool, args, rule that denied).
   The audit is the difference between a guard and a rumor.
5. **Recon gets a read-only lane.** Pure read-only probes (DNS lookups,
   HTTP GET to in-scope hosts, cert transparency) can be allowed via a
   `read_only: true` rule class - but the rule must still be written.
   "Reads are harmless" is how subdomain sweeps leak into production
   panels.
6. **Self-attack is mandatory.** A gate that has never blocked an
   out-of-scope call does not exist. Daily (and at install): feed it a
   deliberate out-of-scope call and PROVE the block + the audit line.
7. **Scope changes are file changes with provenance.** scope.yaml edits
   cite the scope document version; a mid-engagement expansion without a
   scope update is a violation the audit shows.
8. **No secrets in the hook path.** scope.yaml contains hosts and paths,
   never credentials; the hook never receives or logs secrets (redaction
   patterns from session-memory capture apply to its audit log too).
9. **One scope.yaml per engagement, hash-locked.** The gate logs the file
   hash at load; a mid-session change is visible in the audit (and
   requires the TOTP gate for destructive-class expansions).
10. **Workspace mode (non-pentest): deny writes outside the workspace.**
    The same hook, a simpler ruleset: protects the owner's file tree from
    agent drift. Both modes ship with this skill; a session runs one.

---

## 1) WHEN IT FIRES

| Mode | Trigger | Enforcement point |
|------|---------|-------------------|
| Pentest (primary) | session flagged `pentest: true` (or scope.yaml exists in run dir) | PreToolUse on Bash, and on any tool whose args contain a URL/host |
| Workspace (light) | every non-pentest session with the hook installed | PreToolUse on Write/Edit + Bash file-touching commands: deny outside workspace |
| Always-on assert | every pentest turn start | a check that the hook is loaded (log line + hash); if not, first action = load it |

Fires on EVERY tool call in pentest mode (that is the point - mechanical,
zero context cost, no skill invocation needed per call).

---

## 2) scope.yaml - THE SCHEMA

```yaml
# scope.yaml - signed scope doc <ref> - generated <date> - hash at load
meta:
  engagement: "client-x-pentest-2026-11"
  scope_doc: "docs/scopes/2026-11-10-signed.md"
  fail_mode: "closed"          # pentest mode
  log: "audit-scope.log"

in_scope:
  domains:                     # suffix match: subdomains included
    - "client-x.com"
    - "clientx-cdn.net"
  hosts:                       # exact names + resolved IPs pinned at start
    - "app.client-x.com"
    - "203.0.113.10"
  cidrs:
    - "203.0.113.0/24"
  url_paths:                   # optional path allowlist (prefix match)
    - "/api/v1"
    - "/admin"
  ports: [80, 443, 8080, 8443]
  fs_paths:                    # local dirs tools may touch
    - "./recon"
    - "./findings"

read_only_extra:               # read-only rules (still scoped)
  - "crt.sh"                   # cert transparency lookups
  - "dns.google"               # DNS resolution only

allow_commands:                # exact command prefixes always allowed
  - "nmap .*203\\.0\\.113"     # regex, must keep the in-scope CIDR in args
  - "httpx -l ./recon/hosts.txt"
  - "python (tools|scripts)/"

deny_commands:                 # never, even if otherwise in scope
  - "rm\\s+-rf\\s+/"
  - "sshpass"
  - ".*--upload-file.*\\.env"

excluded_from_scope:           # explicitly out (blocks + logs attempts)
  - "mail.client-x.com"        # third-party hosted, contract excludes
```

Rules:

- `domains` match by suffix (foo.client-x.com in, client-x.com.evil.io NOT
  - suffix matching is dot-anchored, a classic bypass is exact-match
  domains; the matcher anchors the dot).
- `hosts` pins resolved IPs at engagement start (recon seat writes them);
  later DNS surprises (a host resolving OUT of scope) surface as blocked
  calls - which is the point.
- `allow_commands` are regexes matched against the full command string;
  they must be conservative (a regex that matches everything is a
  permit-all and a Critical audit finding).
- The file is ASCII, versioned in the run dir, and its sha256 is logged at
  every hook load (drift detection).

---

## 3) THE HOOK - enforcement path

PreToolUse hook (OpenCode hooks or Claude Code PreToolUse), pseudo-contract
(the reference script ships as scope_gate.py, ~200 lines, stdlib only):

```
stdin: tool event JSON (tool name, args/command)
1. load scope.yaml (cached by sha); if load fails -> BLOCK + audit
2. if tool in {Write, Edit}: resolve target path; allow iff inside
   scope.fs or read-only-evaluated; else BLOCK
3. if tool == Bash: extract network targets (domains/IPs in the command
   string) and file paths:
     - every host token must match in_scope.domains/hosts/CIDRs
     - ports matched if present
     - allow_commands / deny_commands regex checked first (deny wins)
   - no target extractable + not allowlisted -> BLOCK (deny-by-default)
4. BLOCK -> exit code 2 with reason on stderr + audit line:
   {"ts","session","tool","args","decision":"block","rule":"<which>"}
5. ALLOW -> audit line (decision:"allow") only for pentest mode
   (full audit trail; workspace mode logs blocks only)
```

Extraction classes handled by the gate:

- domains in URLs (`https://app.client-x.com/...`), hostnames in commands,
  dotted-quad IPs, CIDR targets in nmap-style args.
- file paths (absolute, relative, ~-expanded).
- a deny-pattern wins over any allow (order: deny_commands -> allow ->
  in_scope match -> default deny).

---

## 4) WORKSPACE MODE (non-pentest always-fire)

Same hook, simpler ruleset, shipped as the default:

```yaml
mode: workspace
in_scope:
  fs:
    - "C:/Users/rabim/OneDrive/Desktop/Claude recent backup/<category>"
    - "<workspace>"
    - "C:/Users/rabim/AppData/Local/Temp/opencode"
deny:
  fs:
    - "C:/Windows/**"
    - "C:/Program Files/**"
  commands:
    - "rm -rf /"
    - "git push --force"
    - "curl .*(\\|\\s*sh)"
```

Enforces the standing placement rules mechanically (files never loose on
the Desktop, never inside his other Desktop folders) - the rule that has
been violated by habit despite being written in four configs, because
prompts do not enforce; hooks do.

---

## 5) THE PROCEDURE (per session, pentest mode)

**S1. Scope intake.** The signed scope document lands in the run dir.
Generate scope.yaml from it (the mapping is mechanical: every named host,
domain, CIDR, path, port). The generation is reviewed by the operator
(the model) against the signed doc line by line - a translation error here
is a scope violation with extra steps.

**S2. Install the hook.** Wire the PreToolUse hook into the ACTIVE config
root (per-root rule - verify per root). Log the scope.yaml sha256 + hook
load confirmation to the runlog.

**S3. Self-attack (the proof).** Before the first probe:

```
# attempted out-of-scope call (MUST be blocked):
curl -s https://not-in-scope.example.com --max-time 3
# expected: BLOCKED + audit line. If it executed -> fix, do not proceed.
```

Also attack the bypasses deliberately (Section 6 test list).

**S4. Run the engagement.** Every tool call passes the gate. Blocked
attempts: audit + (if the agent persists) a prompt-level reminder; three
blocks on the same intent = the scope is wrong, not the agent - stop and
re-read the signed doc.

**S5. Scope changes.** Edit scope.yaml only with a citation to the signed
doc (or owner message); re-log the new sha; re-run the self-attack.

**S6. Close-out.** The audit log IS part of the evidence bundle: every
blocked attempt and every allowed class is in the report appendix -
"we touched only what scope.yaml allowed, here is the log".

---

## 6) VERIFICATION COMMANDS (the gate on the gate)

```
# V1. Hook loaded in the ACTIVE config root
#    evidence: hook log line with scope sha + timestamp at session start
#    (per-root check - the config-root law applies)

# V2. Deny-by-default proven (self-attack)
#    out-of-scope curl -> BLOCKED + audit line exists
tail -1 <run-dir>/scope-audit.jsonl | grep '"decision": "block"'

# V3. Suffix-anchoring (the dot-anchoring bypass)
#    attempt "evil-client-x.com" -> BLOCKED (not a suffix of client-x.com)
#    attempt "client-x.com.evil.com" -> BLOCKED

# V4. Deny overrides allow
#    a deny_commands match inside an allowed domain -> BLOCKED

# V5. Fail-closed proven
#    temporarily rename scope.yaml -> next call BLOCKED (not silently allowed)

# V6. Read-only class actually read-only
#    a POST to an allowlisted-crt.sh-like rule -> BLOCKED (rule was GET-only)

# V7. Audit integrity
#    every block has a line; append-only (no rewrites); count matches gate refusals
```

V3 and V5 are the two the fleet had missing once: a suffix-bypass in the
first matcher, and a fail-open default. Both were found by self-attack,
not by reading the code - attack your own gate.

---

## 7) KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Out-of-scope call executed | hook not loaded in active config root (per-root law) | verify per config root; the V2 self-attack at session start |
| Everything blocked (gate too tight) | deny-by-default with no rules authored | author rules from the signed doc; recon read_only class first |
| Suffix bypass (evil-client-x.com allowed) | naive substring match | dot-anchored suffix match; V2 in the self-attack |
| Gate silent on failure (allowed on crash) | fail-open default in hook | fail-closed; V5 proves it |
| Audit log missing blocks | hook blocks but skips logging | block path and audit line are the same code path; audit failure = block (fail-closed) |
| Agent "worked around" the gate (used python to fetch) | command allowlist too narrow, agent routed around | allowlist the TOOL not the bypass; python one-liners doing network I/O go through the same matcher; if the agent routes around scope, that is a scope-discipline incident, logged |
| scope.yaml edited mid-session silently | no sha logging | log sha at load; drift = re-run self-attack + note in report |
| IPv6 target missed | matcher classes only handle v4 | normalize addresses (v4+v6) before matching; add v6 CIDR support to the matcher |

---

## 8) ANTI-PATTERNS

- ALLOW-EVERYTHING-PLUS-DENY: inverting to a blocklist. The blocklist loses
  to every target you forgot. Deny-by-default is the design.
- THE SILENT BLOCK: refusing calls without an audit line or a surfaced
  reason - the agent retries blindly and burns budget. Blocks state the
  rule name and the expected fix.
- SCOPE.CYAML-AS-DOCUMENTATION: a scope.yaml nobody loaded in the active
  config root. The self-attack at session start exists because the most
  common failure is the hook never being wired.
- THE WIDE REGEX: `allow_commands: [".*"]` to stop the nagging. That is
  the gate off. Wide rules need a written justification + expiry.
- PROMPT-ONLY SCOPE: restating scope in the brief and calling it
  enforcement. Briefs carry the WHY; the hook carries the enforcement.
- AUDIT-THEATER: logging allows only. The blocks are the evidence the
  gate works; the allows are the evidence of what was touched. Both.

---

## 9) WORKED EXAMPLES

### 9.1 The engagement that stayed in scope (the point of the skill)

Signed scope: client-x.com + one CDN host + 203.0.113.0/24, ports 80/443/
8080/8443, no mail. scope.yaml generated; hook installed; self-attack
blocks an out-of-scope curl (rule: not-in-in_scope-domains).

Mid-engagement, recon finds `dev.client-x.com` resolving to a DIFFERENT
provider (a forgotten staging box). The agent's follow-up probe fires -
BLOCKED (host pinned by the signed doc's IP list; the new IP is out).
Operator checks the signed doc: the staging box is NOT in scope. The
finding is reported as "discovered out-of-scope asset - expansion needs
client sign-off"; the gate held the line at the exact moment an agent
with momentum would have poked it. That block line in the audit is what
makes the engagement defensible.

### 9.2 The bypass attempts (self-attack suite, run at install)

```
evil-client-x.com        -> BLOCK (not dot-anchored to scope)
client-x.com.evil.com    -> BLOCK (suffix match anchors left)
203.0.113.10             -> ALLOW (in CIDR)
203.0.114.10             -> BLOCK
crt.sh lookup            -> ALLOW (read_only rule)
GET app.client-x.com/admin -> ALLOW (path in scope)
POST app.client-x.com/api -> BLOCK (ports/paths rule: GET-only on /api)
python -c "urlopen('http://internal.host')" -> BLOCK (target parse hit)
```

Two real catches in one drill: the suffix anchoring and the python
urlopen route (the agent-routes-around class). The gate was fixed and
re-attacked BEFORE the engagement started - not after the first incident.

### 9.3 Workspace mode in action (non-pentest)

A web-dev session runs a build script with a bug: `rm -rf "$OUT/*"`
where $OUT is empty -> the command expands to `rm -rf /*`-adjacent paths.
The deny rule (`rm -rf /`) matches the string and BLOCKS before exec -
the standing deny rule made mechanical. Separately, a session trying to
write `C:/Users/rabim/Desktop/notes.txt` (loose on the Desktop - the
banned placement) is blocked with the rule name; the agent reroutes to
the category folder. The prompt rule existed for months; the hook
enforces it on day one.

---

## 10) GATES-LINE CONTRACT

Slot in the per-task extension (pentest sessions):

```
scope: gate active (sha ab12*, N rules, blocks 3/allowed 142)
scope: BLOCKED-itself (self-attack failed -> fix before proceeding)
scope: skipped: workspace mode (no scope.yaml; workspace deny active)
```

- `dim-security` cites the gate: an engagement report whose session log
  shows zero out-of-scope executions plus the audit trail is direct
  security evidence. A pentest report WITHOUT the scope audit trail is
  capped per the honestscore Unknown rule - the evidence either exists in
  the audit file or the dimension is capped.
- The gate never emits its own gate line; it feeds the parent's.

---

## 11) INTEGRATION MAP

| Skill | Relationship |
|-------|--------------|
| pentest / pentest-oracle | scope intake at engagement start generates scope.yaml from the signed doc |
| burp-operator | the bridge's requests pass the same gate (proxy-backed observability); out-of-scope target approvals are denied at the hook |
| adapter-spec | adapters run under workspace mode by default; pentest adapters opt into the scope gate via their config |
| session-monitor-auto | a rising block-rate with no agent adaptation (same block class 3+ beats) = the scope or the agent is wrong; monitor flags |
| secret-rotation / secrets-leak-hunter | deny_commands protect .env reads; the gate is the PreToolUse leg of the secrets defense |
| skill-totp-gate | destructive ops inside scope still require the TOTP gate - the scope gate is a boundary, not a safety interlock |
| subdomain-enum | read_only rules for the enumeration pipeline (subfinder, crt.sh, dnsx) |

---

## 12) OPEN ITEMS (honest limits)

- The matcher is regex-based over command strings - not a parse. An
  obfuscated target (base64 in a script file the command reads) escapes
  string matching; the mitigation is allow-listing script files and
  reviewing them as findings evidence, not trusting command text alone.
- IPv6 + DNS rebinding are known gaps (Section 7); hosts pinning mitigates
  rebinding for pinned hosts, not for unpinned domains.
- No per-user identity in the audit (single-operator fleet); multi-tenant
  audit needs the event-sourced run state (D01) to attach identity.

---

## APPENDIX A - THE ONE-CARD OPERATING SUMMARY

```
GENERATE  scope.yaml from the SIGNED scope doc (mechanical mapping)
INSTALL   PreToolUse hook in the ACTIVE config root (per-root law)
PROVE     self-attack: out-of-scope call MUST block + audit line
          suffix-anchor probe | deny-overrides-allow | fail-closed test
ENFORCE   every tool call: match domains/hosts/CIDR/paths/ports/fs
          deny_commands first -> allowlist -> else BLOCK (logged)
LOG       append-only audit: ts, session, tool, args, decision, rule
CHANGE    scope edits cite the signed doc; re-log sha; re-attack gate
DEGRADE   never fail-open in pentest mode; broken gate = blocked calls
SELF      2 flat beats with the same block class = wrong scope or
          wrong agent - escalate, do not rubber-pass
NEVER     blocklist-default | wide regex without justification |
          silent blocks | prompt-only enforcement
```

---

## APPENDIX B - REFERENCE scope.yaml (annotated, drop-in shape)

```yaml
meta:
  engagement: "client-x-pentest-2026-11"
  signed_doc: "scope-2026-11-08-signed.md"   # the authority
  sha_at_load: "<filled by hook>"            # drift detection
  fail_mode: closed                          # pentest mode never fails open

in_scope:
  domains:        # DOT-ANCHORED suffix match
    - "client-x.com"
    - "clientx-cdn.net"
  hosts:          # pinned name->ip at recon time (rebinding guard)
    - { name: "app.client-x.com", ip: "203.0.113.10" }
  cidrs:
    - "203.0.113.0/24"
  url_paths:      # optional prefix allowlist
    - "/"
    - "/api"
    - "/admin"
  ports: [80, 443, 8080, 8443]

read_only:        # GET-only, no body, safe methods class
  domains: ["crt.sh", "dns.google"]
  tools: ["whois", "dig", "host", "subfinder", "dnsx", "httpx"]

allow_commands:   # regex against the full command string
  - "^nmap .*(203\\.0\\.113|app\\.client-x\\.com)"
  - "^(curl|httpx) .*https://(app\\.)?client-x\\.com"

deny_commands:    # win over allow, always
  - "rm\\s+-rf\\s+/"
  - "curl[^|]*\\|\\s*(ba)?sh"          # curl|sh class
  - "(cat|less|more|head|tail).*\\.env"
  - "git\\s+push.*--force"

excluded:         # explicitly out - blocks AND logs attempts
  hosts: ["mail.client-x.com"]
  note: "third-party hosted mail; contract excludes"
```

The annotations are the spec: dot-anchored suffixes, pinned hosts, a
read-only class, deny-first ordering, and an explicit excluded list (so
"out of scope" is a positive statement, not an absence).

---

## APPENDIX C - THE HOOK CONTRACT (scope_gate.py, exact)

```
in:  stdin JSON event {tool, tool_input, session_id, cwd}
out: exit 0 = allow | exit 2 + stderr reason = BLOCK (PreToolUse deny)
     audit line on EVERY decision:
       {"ts","session","tool","args(<200ch)","decision","rule"}

order of evaluation:
  1. load scope.yaml by path from env SCOPE_YAML (cache by sha)
     load failure -> BLOCK (fail-closed) + audit rule="scope-load-fail"
  2. deny_commands regex match -> BLOCK rule="deny:<pattern-name>"
  3. read_only class match (tool + args in read-only sets) -> ALLOW
  4. extract targets (domains, IPv4/IPv6, CIDRs, URLs, fs paths)
     zero targets + command not in allow_commands -> BLOCK rule="no-match"
     any target outside in_scope -> BLOCK rule="out-of-scope:<which>"
  5. fs paths: outside scope.fs (workspace mode: outside workspace)
     -> BLOCK rule="fs-out-of-scope"
  6. else ALLOW rule="in-scope"

performance: <5ms cached, no network, stdlib only, fail-closed.
```

The audit line on allows is deliberate: at engagement close, the allowed
set is the definitive "what we touched" - the report's scope-compliance
section is generated from it, not reconstructed from memory.

---

## APPENDIX D - SELF-ATTACK SUITE (the install gate, all must pass)

Run at install and re-run after any scope change:

```
T1  out-of-scope domain    curl https://notinscope.example  -> BLOCK
T2  suffix anchor left     curl https://client-x.com.evil.com -> BLOCK
T3  suffix anchor right    curl https://evil-client-x.com -> BLOCK
T4  in-CIDR allowed        nmap 203.0.113.10 -> ALLOW
T5  out-CIDR blocked       nmap 203.0.114.10 -> BLOCK
T6  pinned-host drift      app.client-x.com resolving to a NEW ip
                           -> BLOCK (pin mismatch) [if pinning on]
T7  deny beats allow       curl https://app.client-x.com | sh -> BLOCK
T8  read-only class        dig client-x.com -> ALLOW
T9  write-class probe      curl -X POST .../api -> BLOCK (GET-only rule)
T10 fail-closed            mv scope.yaml scope.yaml.bak; any call -> BLOCK
                           (restore after)
T11 audit integrity        every decision above has exactly one audit line
T12 obfuscation probe      python -c "open('x').read()" fetching a URL
                           string from a file -> BLOCK (no-match rule)
```

T10 and T12 are the two most skipped and the two that matter most: T10
proves fail-closed (the gate that fails open is a permit), T12 proves the
deny-by-default spine works when string matching cannot parse the command.
A gate passing T1-T9 but failing T12 will eventually be routed around -
by the agent, innocently, the first time a one-liner needs a lookup.

Record the suite results in the runlog: one line, all 12 outcomes.

---

## APPENDIX E - SCOPE-CHANGE PROCEDURE (mid-engagement expansion)

Scope expansions happen (new subdomain authorized mid-test); the procedure
keeps the gate honest:

1. The client's written authorization lands in the run dir (email/quote
   archived as a file - the audit trail needs the artifact).
2. scope.yaml is edited; the diff cites the authorization file in a
   comment; the new sha is logged by the hook on next load.
3. The self-attack suite re-runs (the changed rules are the new bypass
   surface - attack your own gate, again).
4. The monitor notes the change (beats.md line) so the report's timeline
   shows when scope changed and under whose authorization.

De-scoping (client pulls a host mid-test) follows the same path in
reverse - the host moves to `excluded`, and any cached session state
touching that host is flagged in the report.

No verbal expansions. "The client said it was fine" without a file is not
authorization - it is a finding for the report.

---

## APPENDIX F - WORKSPACE MODE DETAILS (non-pentest always-fire)

Workspace mode enforces the OWNER's standing file-placement rules
mechanically:

```
in_scope.fs:
  - "<workspace>"
  - "C:/Users/rabim/AppData/Local/Temp/opencode"       # scratch
  - "C:/Users/rabim/OneDrive/Desktop/Claude recent backup/<category>"
deny.fs:
  - "C:/Windows/**"
  - "C:/Program Files/**"
  - "C:/Users/rabim/Desktop/**"        # loose files + his folders
deny.commands:
  - "rm\\s+-rf"
  - "git\\s+push.*--force"
  - "curl[^|]*\\|\\s*(ba)?sh"
```

What changes vs pentest mode: fail_mode is `open-with-log` for READ
operations (a broken gate must not stop research) and `closed` for
writes/deletes (a broken gate must not allow writes outside the
workspace). The asymmetry is the point: reads are cheap and reversible;
writes to the owner's filing system are the measured failure.

The gate logs the same audit lines; the workspace audit doubles as the
"what did the session touch" record the CONTINUE-HERE and backup flows
consume.

---

## APPENDIX G - EXTENDED TROUBLESHOOTING

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Hook not firing at all | hooks not enabled in the config root, or matcher regex wrong | verify per root (the per-root law); run the self-attack T1 as the session's first action |
| Every call blocked, even in-scope | scope.yaml path env not set / file unreadable | the hook's load-fail rule fires; check env + path per root |
| Blocked agent keeps retrying the same call | block reason not surfaced | the block stderr names the rule; the agent brief must say "read the refusal, do not retry blind" |
| Legit pipeline blocked by deny regex (curl|sh pattern on a safe chain) | deny patterns too broad | narrow the pattern (e.g. match `curl.*| *sh$`), document the change in scope.yaml comments, re-attack |
| Audit file grows unbounded | no rotation | rotate per engagement; archive closed engagements with the report |
| Two sessions, different scope.yaml versions | per-session run dirs with copied files | single source: SCOPE_YAML env points at the canonical file; copies are the drift failure class |
| Host pinning false-blocks after client DNS change | pin stale | re-run the recon pin step (logged), not a silent pin edit |

---

## APPENDIX G2 - THE COMPLIANCE REPORT SECTION (generated, not written)

At engagement close, the report's scope-compliance section is generated
from the audit log:

```
scope-compliance:
  gate: active (sha ab12* -> ef34* after 1 authorized expansion)
  self-attack suite: 12/12 pass (runlog line <ref>)
  decisions: allowed 1,412 | blocked 38 (all logged, rule names attached)
  expansions: 1 (2026-11-10, auth file scope-change-2.md)
  out-of-scope executions: 0   <- THE number that defends the engagement
```

the T-suite exists so "0 blocks" means "0 attempts out of scope", not
"nobody looked at the gate".

---

## APPENDIX H - WORKED EXAMPLE: THE GATE THAT CAUGHT ITS OWN AGENT

Mid-engagement, the recon seat runs:

```
subfinder -d client-x.com -silent | httpx -silent
```

In-scope so far. But httpx probes EVERY resolved subdomain - including
`dev.client-x.com` which the client's DNS now points at 198.51.100.7
(outside the pinned CIDR). The gate blocks the httpx leg
(rule=out-of-scope:ip 198.51.100.7). The seat surfaces the refusal; the
operator checks: the signed doc pins app/client/www only; `dev` is
out-of-scope infrastructure the client forgot about. Finding written
(subdomain takeover candidate + scope question), client asked, answer
archived, scope updated through the Appendix E procedure.

Without the gate, httpx would have probed the dev box - harmless-looking,
a contract violation, and (because nobody remembered it existed) a
possible alarm on the client's side. The gate did not make the agent
smarter; it made the unauthorized touch impossible. That is the entire
value proposition of enforcement over prompting.

---

## APPENDIX I - DENY-DEFAULT vs BLOCKLIST (the design lesson)

The fleet already had settings.json deny rules (rm -rf, force-push,
curl|sh). This gate is the GENERAL pattern they point at:

| settings.json deny rules | scope-enforcer gate |
|--------------------------|----------------------|
| static per-machine list | per-engagement rule file |
| blocklist | deny-by-default allowlist |
| no audit trail | append-only audit per decision |
| shell strings only | targets: domains/hosts/CIDR/fs/ports |

The lesson: when a blocklist keeps growing, flip to deny-by-default
with a rule file; the old patterns become deny_commands (short,
reviewed, justified). Enforcement is a boundary, not a habit.

---

## APPENDIX J - AGENT BRIEF UNDER THE GATE (three habits)

1. READ THE REFUSAL: a block names its rule; fix behavior or request a
   scope change - never retry blind (retries burn budget and log noise).
2. NO ROUTE-AROUNDS: python one-liners doing what a blocked command did
   are a violation (T12-class commands are blocked anyway); the honest
   path is the scope-change procedure (Appendix E).
3. SURFACE BLOCK SPIKES: three same-class blocks = wrong scope or wrong
   agent - surface it, stop, let the operator decide.

Spawn briefs include these three lines verbatim when the gate is on.

---

## APPENDIX K - COMPLIANCE NUMBER DEFENSE (why the audit log matters)

A pentest report defends itself with three numbers, all generated from
the audit file, none reconstructed from memory:

- out-of-scope executions: 0 (every decision logged, rule attached)
- self-attack suite: 12/12 at install and after every scope change
- expansions: N, each with its authorization artifact referenced

"Trust us, we stayed in scope" is the weak version. The gate makes the
strong version free: the log IS the compliance section. This is the same
evidence-first doctrine as every other gate in the fleet - the difference
is that here the evidence doubles as legal cover, so the standard is
zero-tolerance: one unlogged decision is a gate defect, not a footnote.

---

## APPENDIX L - FINAL SELF-CHECK (before any engagement close)

1. Did the self-attack suite run at install AND after every scope change
   (runlog line for each)?
2. Is every blocked decision logged with a rule name (sample 5, check)?
3. Is the allowed-set audit the source of the report "systems touched"
   section (generated, not remembered)?
4. Did any agent route around a block (grep the audit for the same class
   after a refusal)?
5. Are scope expansions each tied to an authorization artifact?

Any "no" = the compliance section cannot be written yet. The gate product
is not the blocking - it is the defensible record.

---

## APPENDIX M - WORKED EXAMPLE: WORKSPACE MODE STOPS THE CLASSIC

Dictated task: "fix the pricing table on the site" - while the session
cwd is a scratch dir, an early command runs:

  cp pricing.html C:/Users/rabim/Desktop/notes.html

The workspace gate blocks it: rule fs-out-of-scope (Desktop is deny).
The agent surfaces the refusal, reroutes to "Claude recent backup/web-dev/",
and the file lands in the owner filing system where he expects it.

The prompt version of this rule has existed on every box since 2026-08
("every file goes in Claude recent backup/<category>") and was violated
repeatedly by momentum. The hook version was violated zero times after
install - the same lesson as every hook in this fleet: prompts ask,
hooks enforce, and the cheapest enforcement is the one nobody has to
remember.

---

## APPENDIX N - PERFORMANCE AND FAILURE BUDGET

The gate sits on EVERY tool call, so its budget is strict:

- hot path: <5ms (sha-cached rules, regex only, no network, no subprocess)
- cold load: <100ms once per session
- crash policy: exit 2 (BLOCK) with rule=scope-load-fail - a broken
  gate costs a blocked call and one audit line, never a permission
- audit write: append-only, best-effort with 3 retries, then in-memory
  buffer flushed on next successful write (never dropped silently)

The budget is the reason the gate is a hook and not a skill: a skill
recalled per call costs tokens and discipline; a hook costs 5ms and
obeys arithmetic.

---

## APPENDIX O - METRICS (one line per session, pentest mode)

scope | <date> | engagement <id> | rules <N> | sha <ab12*>
| self-attack <12/12|failed> | decisions <A allow / B block>
| expansions <N> | out-of-scope-exec <0|count> | gate-crashes <N>

Signals: blocked-rate (a healthy engagement blocks a handful and every
block is a story; hundreds of blocks = mis-authored rules); gate-crashes
(nonzero = fail-closed working, but fix the cause before the next
session); out-of-scope-exec must be 0 always - any nonzero is an incident
with its own report.

These metrics feed the engagement report AND the fleet calibration file
(same line format as review/clarify/converge/analyze).

---

## APPENDIX P - FAQ (short, factual)

**Q: The agent needs an out-of-scope DNS lookup mid-analysis.**
The read_only class covers it (dns lookups allowed by rule). Anything
that touches a HOST still needs the boundary - reads about a host are
not reads of a host.

**Q: Client says "also test staging" on a call.**
Get it in writing (email file in the run dir), then Appendix E. A voice
authorization with no artifact is a report finding, not scope.

**Q: The gate blocks a legit allowlisted command because of quoting?**
Normalize the command (single quoting style in briefs), then fix the
rule regex if it was over-narrow - document in scope.yaml comments and
re-attack.

**Q: Is the gate active for MCP tool calls?**
Yes - PreToolUse covers the MCP tool namespace in both runtimes; the
matcher includes mcp tools by name prefix. The T-suite includes one MCP
call case for exactly this.

---

## APPENDIX Q - THE ONE-PARAGRAPH DOCTRINE

Scope enforcement is the difference between an agent that was ASKED to
behave and an agent that CANNOT misbehave. Every measured incident in
the campaign log where an agent touched something out of bounds shares
one shape: the rule existed in prose, the moment of violation was busy,
and nobody - agent or human - re-read the prose at that moment.
Mechanical enforcement has no such moment: the boundary fires at the
exact instant of the attempt, every time, with a record. That is why
this skill is a hook with a procedure attached, not a procedure with a
hook attached - and why the T-suite (prove the guard fails on its own
defect) is the real install gate, not the yaml.
