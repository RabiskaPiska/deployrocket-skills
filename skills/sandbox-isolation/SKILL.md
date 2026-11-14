---
name: sandbox-isolation
description: Per-seat sandbox isolation for untrusted or risky execution - the E2B-self-host / Microsandbox / Docker-per-seat pattern. Every adapter run, untrusted payload, scraped-file parse, or dependency install executes inside an ephemeral sandbox with CPU/mem/disk/network caps, no host filesystem access, pause/resume, and teardown on exit. Kills the "one bad seat kills the host" failure class and the shared-box cross-contamination class. Covers three tiers - Docker ephemeral containers (default), microVM runtimes (Microsandbox/E2B self-host, for hostile code), and plain process isolation with resource caps (when no container runtime exists) - plus the sandbox decision table for when NOT to sandbox. alwaysFire - fires whenever a task executes untrusted input, unreviewed dependencies, or fan-out worker fleets where seat isolation matters.
alwaysFire: true
---

# sandbox-isolation - per-seat isolation for untrusted execution

You are running **sandbox-isolation**. Before anything untrusted executes -
a scraped file, a cloned repo's scripts, a pip install, an adapter running a
third-party tool, a worker fleet whose seats you do not fully control - put
it in a sandbox: ephemeral, resource-capped, network-scoped, and destroyed
afterward. The host survives everything the sandbox sees.

**Why this exists (the failure class):** the fleet's workers share one box
per role; one bad seat - a dependency install that hooks a build, a
scraped document with a malicious macro, an adapter invoking a tool that
"temporarily" writes outside its dir - can kill the host or poison peers.
The master table (D11, P3) names per-seat sandbox isolation as a missing
infra layer; the 2026 ecosystem gives three buildable tiers (Docker
ephemeral, microVM self-host, process-level caps) and this skill makes the
choice mechanical instead of per-incident.

**What this skill is NOT:**

- NOT a pentest tool-safety certification. The scope gate (scope-enforcer)
  controls WHERE calls may point; the sandbox controls WHAT DAMAGE a call
  can do. Both apply in pentest sessions.
- NOT a VM provider recommendation. E2B hosted is fine for exploration;
  self-host (E2B open-source or Microsandbox) is the default because the
  fleet runs on VPS boxes it owns.
- NOT universal. Compiling a LaTeX PDF you wrote does not need a sandbox
  sandbox-theater; scraping an untrusted URL and parsing its bytes does.
  The decision table (Section 2) is the skill.

---

## 0) HARD RULES (survive compaction)

1. **Untrusted means sandboxed, no exceptions.** "Untrusted" = anything not
   authored by this session and not reviewed by a fresh-context pass:
   fetched files, cloned repos' code, model-generated code that will be
   EXECUTED (not just read), third-party adapters, dependency installs
   from non-pinned sources.
2. **Default tier: ephemeral Docker** (one container per task, `--rm`,
   read-only rootfs, capped CPU/mem/disk, no docker socket, tmpfs for
   scratch). MicroVM only for hostile-input runs; process-isolation only
   when no container runtime exists - and then the task is flagged.
3. **Network is a capability, not a default.** Sandboxes get the network
   profile their task needs (none / allowlist egress / full). Default:
   egress allowlist or none. A sandbox with full network is a remote
   workstation, not a sandbox.
4. **One process, one sandbox, one lifetime.** No reuse across tasks (the
   contamination class); teardown on exit is the rule - the filesystem of
   a finished sandbox is evidence, not a workspace (export artifacts
   explicitly, then destroy).
5. **Caps before work.** CPU, memory, disk, PID caps are set AT LAUNCH.
   A sandbox without caps is a container-shaped accident waiting to eat
   the host (the <=90% resource contract applies inside sandboxes too).
6. **Artifacts cross the boundary explicitly.** Copy results OUT to the
   run dir (with a scan for secrets/oversize), never mount the run dir
   read-write into the sandbox. Inputs in (read-only mount), artifacts
   out (copied, scanned, sized).
7. **Pause/resume is optional infra, teardown is not optional.** Long
   jobs may pause; nothing survives teardown except exported artifacts.
8. **The sandbox is disposable.** Any state that matters lives in the run
   dir / store, not in the container. A deleted sandbox must never lose
   work - if it would, the workflow is wrong, not the rule.
9. **Audit the boundary.** Every sandbox creation, cap set, network rule,
   and teardown is one log line (the same evidence doctrine as every
   gate). A sandbox nobody can prove existed is theater.
10. **Host hygiene stays on the host.** Secrets never enter sandboxes via
    env; if a task needs a credential, it gets a scoped token injected
    at run start and the sandbox is burned after.

---

## 1) WHEN IT FIRES

Always-fire on any of:

| Trigger | Tier |
|---------|------|
| Parsing files fetched from the web (PDFs, docs, archives) | Docker ephemeral, no egress |
| Running a cloned repo's code / examples / install scripts | Docker, egress allowlist |
| `pip install` / `npm install` of non-pinned or new deps | Docker, egress registry-only |
| Adapter invocation (fleet-forge adapters, tool wrappers) | Docker default |
| Pentest payload handling (generated by a target, or exploit code) | microVM (hostile class) |
| Untrusted model output executed as code (the LLM-writes-code case) | microVM or Docker + no egress |
| Worker fleets (>=3 seats) on one box | Docker per seat, shared network profile |
| Research with `curl | sh`-class temptation | BLOCK the pattern (scope gate) + sandbox the run |
| Pure read-only analysis of trusted local files | skip (`sandbox: skipped: trusted local`) |

---

## 2) THE DECISION TABLE - sandbox tier by risk

| Input/Task class | Tier | Network | Caps | Lifetime |
|------------------|------|---------|------|----------|
| Trusted local files, trusted code | none | - | - | - |
| Web-fetched file parse | Docker, RO rootfs | none | 1 CPU / 512M / 1G disk | minutes |
| New dependency install | Docker | registry egress only | 2 CPU / 2G | one install |
| Cloned repo code execution | Docker | allowlist | 2 CPU / 2G | one run |
| Adapter tool run | Docker | profile per adapter | NumCPU-1 | one run |
| Hostile/generated payload | microVM | deny + explicit holes | per-VM | one run |
| Crew worker fleet | Docker per seat | shared profile per role | per-seat caps | wave lifetime |
| Long-running scheduled job | Docker w/ named volume if state needed | allowlist | contract caps | managed |

The rule that collapses most debates: **if the task's inputs come from
outside the fleet, it gets a sandbox; the only question is which tier.**

---

## 3) THE PROCEDURE (step by step)

**S1. Classify.** Run the decision table. Record the tier choice + one-line
why in the run state (the report's environment section cites it).

**S2. Provision.** Tier 1 (default):

```
docker run --rm --name smem-<run>-<seat> \
  --cpus=2 --memory=2g --pids-limit=256 \
  --network <profile> \           # none | allowlist-network
  --read-only --tmpfs /tmp:size=512m \
  -v <run-dir>/in:/in:ro -v <run-dir>/out:/out \
  <image> <command>
```

- Images are pinned (digest, not tag) and built from the fleet's own
  Dockerfiles (dependency-advisory scans them).
- `in/` is read-only, `out/` is the only writable mount.

**S3. Execute + observe.** The sandbox's stdout/stderr streams to the run
log. A sandbox that lives past its task's beat budget is chased by the
watchdog like any child (pause -> inspect -> kill).

**S4. Export.** Copy artifacts from `out/` to the run dir; run the
artifact gate: size sanity, secret-pattern scan (session-memory redaction
patterns), file-type check. THEN the sandbox is destroyed
(`--rm` on exit; explicit `docker rm -f` on cleanup paths).

**S5. Record.** One audit line: sandbox id, image digest, caps, network
profile, duration, exit code, artifacts exported. The line is the
provenance: any finding produced inside the sandbox carries its sandbox id
in the evidence chain.

**S6. MicroVM tier (hostile inputs).** When the task handles hostile input
(payloads from a target, untrusted generated code), the same flow runs
inside a microVM (Microsandbox or E2B self-hosted): kernel-level isolation,
snapshot/restore for fast cold starts, and NO shared kernel with the host.
The workflow is identical; only the runtime differs. The tier escalation
rule: two container escapes or suspicious kernel-level behavior in the
same class -> escalate the tier for that task class permanently.

**S7. Process-tier fallback (no container runtime).** Windows boxes and
stripped VPSes: the job runs as a dedicated low-privilege user, under the
OS job controls (Windows Job Objects / `ulimit` + `systemd-run --scope` on
Linux), disk quota via a dedicated volume, network via a firewall profile.
Flag every such run in the report: `sandbox: process-tier (no container
runtime)` - honest degradation, same audit shape.

---

## 3) VERIFICATION COMMANDS

```
# V1. Runtime present and functional
docker run --rm hello-world        # (or: podman / microvm health check)
#    expect: clean exit; the pull itself happens ONCE at install, not per run

# V2. Caps actually applied
docker inspect smem-<id> --format '{{.HostConfig.Resources}}'
#    expect: the cpu/memory/pids caps you set (not host defaults)

# V3. Network profile enforced
docker run --rm --network none <img> python -c "import urllib.request; urllib.request.urlopen('https://example.com', timeout=3)"
#    expect: fails fast (no egress) - the guard proves itself by failing

# V4. Ephemeral proven
docker ps -a | grep smem-<run>     # after close: expect no leftovers

# V5. Artifact boundary
#    in/ was mounted RO (no write-back); out/ contents scanned
#    (sizes sane, no secrets, file list matches the task's expectation)

# V6. Teardown logged
grep "sandbox-close" <run-dir>/audit.log | tail -1
```

V2, V3 and V4 are the ones skipped in practice - a sandbox that inherited
host defaults, kept network, or left containers lying around is the
accident the skill exists to prevent, and all three are one-command checks.

---

## 5) KNOWN-FAILURE CHECKLIST

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Sandbox wrote to the host FS | bind-mounted run dir read-write | in/ RO, out/ copy-out only; V2-style mount check |
| Container left running after task | teardown not in the finally-path | teardown in the wrapper script's trap; nightly prune (`docker container prune --filter until=24h`) |
| Caps "didn't apply" | image default beats flag, or compose overrides | V2 check per launch; pin the runner script, not ad-hoc docker invocations |
| Sandbox needs secrets | env leakage habit | scoped, single-use tokens passed as files (RO), never env; sandbox burned after |
| Cold-start kills latency budget | pull-per-run | pre-pull pinned digests at session start; the pull IS the supply-chain check point (dependency-advisory) |
| Windows host, no containers | runtime absent | process-tier fallback (S6) + flag in the report; do not pretend |
| Sandbox escapes via kernel | shared kernel container tier | hostile class -> microVM tier; that is the tier's entire purpose |
| Artifacts lost on teardown | export step skipped | S4 export is part of teardown; the audit line lists artifact hashes |

---

## 6) ANTI-PATTERNS

- SANDBOX-THEATER: a container with host network, root, and a rw mount of
  $HOME. The costume of isolation with none of the property.
- THE ETERNAL SANDBOX: reuse across runs for "warmth". Contamination is
  cumulative; lifetime = one task.
- SECRETS-VIA-ENV: the sandbox sees the fleet's .env because env is
  inherited. Scope credentials per sandbox; burn after.
- TIER-SKIPPING: hostile payloads in the cheap tier because the microVM
  is "slow to boot". The tier table is mechanical; risk decides, not
  impatience.
- TEARDOWN-OPTIONAL: "I'll clean up later." Prune jobs + trap-based
  teardown; later never comes.
- DOCKER-AS-VM: running a full desktop image with 16G to run a python
  script. Caps exist to make waste visible.

---

## 7) WORKED EXAMPLES

### 7.1 The scraped-file parse (the most common case)

Task: "read this client PDF and extract the pricing". The PDF comes from
outside. Flow: fetched to run-dir/in/; sandbox launched (Tier 1, network
none, 1 CPU / 512M); parser runs; out/ gets extracted text + a file list;
teardown; audit line records the sandbox id + exit code. If the PDF carried
an exploit (parser CVEs are a real class), the blast radius was one dead
sandbox, not the session host. Cost: one pre-pulled image, ~2s overhead.

### 7.2 The untrusted dependency install

`uv add <new-package>` for a client project: registry-only egress sandbox,
install runs, lockfile exported. The install script class (postinstall
hooks reaching for the network or $HOME) hits the cap wall instead of the
host. Paired with dependency-advisory (advisory check) and
supply-chain-sentinel (SBOM), this closes the three legs: known-vuln,
build-time behavior, and runtime isolation.

### 7.3 Crew fan-out with per-seat caps

An 8-seat build: each seat's worker runs in its own container with 2 CPU /
4G caps and a shared no-egress-except-git profile. One seat runs away
(exploit loop, runaway allocation) - its container hits the cap, dies, the
monitor sees the exit, the scope is absorbed per the recovery ladder. The
host never felt it. Before this skill: the same runaway was the "one bad
seat kills the host" incident class (D11's own words).

### 7.4 Honest skip

Task: rewrite a README in the repo. Trusted files, no execution of outside
input. `sandbox: skipped: trusted local edit` - correct. Sandboxing a text
edit is theater and the gates line should say skip.

---

## 8) GATES-LINE CONTRACT

Slot in the per-task extension:

```
sandbox: docker (2 runs, 0 escapes, teardown ok)
sandbox: microvm (hostile class, 1 run)
sandbox: process-tier (no container runtime on host - flagged)
sandbox: skipped: trusted local edit
```

Feeds `dim-security` (isolation evidence: audit lines + V-checks) and
`dim-robustness` (the runaway-containment story). The teardown-and-audit
requirement is binary: runs without teardown records cap the security
dimension per the Unknown rule.

---

## 9) INTEGRATION MAP

| Skill | Relationship |
|-------|--------------|
| adapter-spec | adapters declare their sandbox needs (tier, caps, egress) in the adapter manifest; the gate enforces |
| scope-enforcer | scope gate decides WHAT calls may run; sandbox decides what damage they can do - both PreToolUse-adjacent, complementary |
| supply-chain-sentinel | SBOM/grype run on the sandbox images; installs happen inside sandboxes |
| dependency-advisory | new deps installed in registry-only sandboxes before entering the project |
| resource-governor | the <=90% contract applies per-seat; sandbox caps are its enforcement arm |
| dag-dispatcher | worker executors may point at sandboxed runtimes; lease holders track sandbox ids in the ledger |
| session-monitor-auto | a sandbox exceeding its beat (alive with no output growth) is a stall; pause -> inspect -> kill |
| mcp-auditor | unverified MCP servers are exactly the "hostile class" - their execution is microVM-tier |
| fleet-monitor | multi-seat runs: per-seat caps + the monitor's stall beats = the containment pair |

---

## 10) OPEN ITEMS (honest limits)

- Windows process-tier is weaker isolation (job objects, not kernels);
  fine for scratch work, flagged whenever used - do not run the hostile
  class there.
- No GPU sandboxing story (CUDA passthrough breaks cap semantics); GPU
  work stays on dedicated hosts with its own contracts.
- Image supply chain is as strong as the pinning discipline - digest pins
  + advisory scans are enforced by habit, not by tooling here; a CI gate
  for image builds is the natural next step (D04's eval-gate pattern).

---

## APPENDIX A - THE ONE-CARD OPERATING SUMMARY

```
CLASSIFY  input/task class -> tier (none / docker / microvm / process)
LAUNCH    one sandbox per run: caps (cpu/mem/pid/disk), network profile,
          RO in-mount, out-mount for artifacts, --rm lifetime
RUN       task inside; monitor beats: alive + output growth required
EXPORT    artifacts -> run dir (scan: secrets, oversize, file list)
BURN      teardown in the finally path; audit line (id, caps, exit)
FALLBACK  no container runtime -> process-tier, flagged in report
ESCAPE    hostile class NEVER in the cheap tier; tier table is mechanical
PROVE     V2 caps | V3 network | V4 teardown - the three skipped checks
NEVER     host network | rw $HOME mounts | env-inherited secrets |
          reuse across runs | teardown-optional
```

---

## APPENDIX B - THE TIER TABLE (mechanical, no judgment calls at run time)

| Class | Examples | Tier | Network | Caps (default) |
|-------|----------|------|---------|----------------|
| trusted-local | own repo edits, internal docs | none | - | - |
| external-parse | scraped PDFs, client uploads, email attachments | docker | none | 1 CPU / 512M / 1G |
| dep-install | new pip/npm/uv packages, postinstall scripts | docker | registry-only | 2 CPU / 2G |
| cloned-code | running a cloned repo's examples/scripts | docker | allowlist | 2 CPU / 2G |
| adapter-run | fleet adapters invoking external tools | docker | per-adapter profile | NumCPU-1 / 4G |
| hostile | payloads from a pentest target, unverified MCP servers, model-generated code about to execute | microvm | deny + explicit | 2 CPU / 1G |
| crew-worker | fan-out seats on shared box | docker | shared profile | 2 CPU / 4G per seat |
| no-runtime | Windows box without containers | process-tier | firewall profile | job-object caps |

Judgment happened at TABLE-WRITING time (per project, reviewed); the run
time decision is a lookup. That is what makes the tier cheap - and what
prevents the tier-skipping anti-pattern (risk decided in advance, not
under deadline pressure).

---

## APPENDIX C - THE RUNNER WRAPPER (sandbox_run.sh contract)

```
sandbox_run.sh <tier> <image-ref> -- <cmd...>
  1. resolve image by DIGEST (tag->digest pin check; untagged = refuse)
  2. caps + network per tier table (override file per run optional)
  3. mounts: <run>/in -> /in:ro ; <run>/out -> /out (rw, the ONLY rw)
  4. no secrets in env; scoped creds as files in /in if task needs them
  5. trap teardown: container rm + network disconnect (even on signal)
  6. audit line on start + exit (ts, sandbox id, image digest, caps,
     exit code, artifact list with sizes)
  7. return the task's exit code (the wrapper is transparent to callers)
```

Everything that wants to "just run docker" goes through the wrapper - the
same single-writer discipline as the worktree rule: one writer per
mechanism, no ad-hoc invocations that skip the caps.

---

## APPENDIX D - NETWORK PROFILES (the four that cover everything)

```
none          --network none                          (external-parse)
registry-only egress to registry-1.docker.io, pypi, npm  (dep-install)
allowlist     egress to named hosts:ports              (cloned-code, adapters)
shared-crew   allowlist: git remote + package registry + nothing else
                                                          (crew seats)
```

Profiles are docker networks + egress rules defined once per box (in the
fleet's infra scripts), referenced by name. An ad-hoc `--network bridge`
in a task command is a finding (the runner refuses it in strict mode).

Signals: escapes must be 0 (any nonzero = incident report + tier
escalation for that class); teardown=failed gets an auto-prune entry and a
runlog note; duration outliers (10x the class norm) feed the monitor's
stall beats. Same calibration line format as the other gates.

---

## APPENDIX F - FAQ (short, factual)

**Q: Docker adds seconds - is that worth it for a small parse?**
Yes: the class decides, not the mood. The external-parse class has a CVE
history (PDF parsers especially); the 2-second cap wall is the cheapest
insurance in the fleet. The overhead budget exists so the answer is
always yes.

**Q: The task needs a secret (e.g. an API key) inside the sandbox.**
Scoped single-use credential written to in/ as a file (RO), not env.
Sandbox is burned after the run. The fleet keychain never enters a
sandbox env.

**Q: My box has no container runtime and I need isolation NOW.**
Process-tier (S6) + flag it. It is honest containment, not pretend
isolation - the flag is what keeps the report honest.

**Q: Do MCP tool calls get sandboxed?**
The MCP server process runs in its sandbox if its mcp-auditor verdict says
unverified; verified servers with narrow tool surfaces can run host-tier
with the scope gate on. The mcp-auditor verdict is the classifier.

**Q: What about interactive debugging inside a sandbox?**
Allowed: attach a shell to the same sandbox, caps unchanged. The sandbox
is a boundary, not a hardship - debugging inside it is normal work.

**Q: Sandbox ran fine but artifacts contain secrets?**
The export scan caught it (V-class artifact gate): redact or re-scope the
credential, then export. An artifact leaving a sandbox is a boundary
crossing - treat it like one.

---

## APPENDIX E - METRICS (one line per sandboxed run)

```
sandbox | <date> | tier <docker|microvm|process> | image <digest12>
| caps <c/m/p/d> | net <profile> | dur <s> | exit <code>
| artifacts <n> | escapes <0> | teardown <ok|failed>
```

Signals: escapes must be 0 (any nonzero = incident report + tier
escalation for that class); teardown=failed gets an auto-prune entry and a
runlog note; duration outliers (10x the class norm) feed the monitor's
stall beats. Same calibration line format as the other gates.

---

## APPENDIX G - PROVISIONING A NEW BOX (the sandbox enablement runbook)

One-time per box, in order:

1. Install the container runtime (docker or podman; rootless preferred).
2. Build/pull the fleet base images by DIGEST; record digests in the
   infra runbook; run dependency-advisory + supply-chain-sentinel on
   the image set.
3. Create the four network profiles (Appendix D) and verify each with
   one allow + one deny probe.
4. Install sandbox_run.sh into the fleet bin path; chmod; smoke test
   with the T-suite equivalent (caps, network, teardown).
5. Wire the monitor: sandbox start/exit events land in beats.md.
6. Record the enablement in the runlog (sha of the runner + profiles).

Without provisioning, sessions fall back to process-tier silently - the
runbook makes the fallback visible instead of silent.

---

## APPENDIX Q - WORKED EXAMPLE: THE MCP SERVER UNDER MICROVM

An unverified MCP server enters the fleet (mcp-auditor flags it:
unknown publisher, tool metadata untrusted). Execution policy:

1. verdict=unverified -> hostile class -> microvm tier.
2. The server process runs with: no shared kernel, 1 CPU / 1G, network
   deny except its transport endpoint.
3. Tool calls logged at the scope gate AND the sandbox audit.
4. After 20 clean sessions: promote to docker-tier (shared kernel) -
   promotion is earned, the same arithmetic as review-gate rollout.

The failure it prevents is the mcp-auditor finding class: a malicious
tool definition that, once invoked, reaches the host. In the microvm,
the worst case is a dead microVM and a finding. In the host tier, it is
the incident.

---

## APPENDIX R - THE T-SUITE FOR SANDBOXES (prove the box is a box)

Run at provisioning and after any runner/profile change:

```
T1 caps:     container stress -> OOM-killed at the mem cap, host untouched
T2 pid cap:  fork bomb contained (pids-limit) -> container dies, host ok
T3 disk:     fill /tmp beyond tmpfs size -> capped error, host disk free
T4 network:  none-profile container curl -> timeout (V3 check)
T5 registry: dep-install container can reach pypi, NOT generic egress
T6 rw-check: container writes to /in -> read-only fs error (V2 analog)
T7 teardown: normal exit, ctrl-c, kill-9 -> container gone in all three
T8 artifact: out/ file appears on host with expected size; in/ unchanged
T9 audit:    start + exit lines present with correct digest + caps
T10 hostile: model-generated code touching /etc -> microvm tier, host
             untouched
```

T-suite record: one runlog line, all outcomes. A sandbox install that has
never seen T10 (kill-9 teardown) does not know if its cleanup path works -
the trap-based teardown is exactly the code that never runs until it must.

---

## APPENDIX R - WHEN NOT TO SANDBOX (the honest skip table)

| Situation | Skip? | Why |
|-----------|-------|-----|
| editing the fleet own repos | yes | trusted local, git-recoverable |
| reading brain notes / memory | yes | read-only, local |
| running the test suite on own code | usually | trusted code; repo is the boundary |
| curl an API with the fleet token | no | token boundary -> scoped cred + sandbox or scope-gate-only decision, recorded |
| executing user-uploaded data pipelines | docker | the class exists for this |
| pentest payload execution | microvm | hostile class, always |

The skip decision is recorded in the gates line (`sandbox: skipped:`,
with why) - an unexplained skip on an external-parse class task is an
audit finding.

---

## APPENDIX S - CREW FAN-OUT BLUEPRINT (per-seat caps, end to end)

An 8-seat build wave under this skill:

```
for seat in S1..S8:
  sandbox_run.sh docker <crew-image> --network shared-crew \\
    --cpus 2 --memory 4g --name run17-<seat> -- <worker cmd>

shared-crew profile: egress = git remote + pypi only
seats cannot see each other (separate containers, separate out/)
merge happens on the HOST from out/ exports (subagents S7 rule)
```

What changed vs the shared-box fleet: a seat cannot read another seat
out/ (cross-contamination class closed), a runaway seat dies at its cap
(host untouched), and the wave audit shows 8 start/exit pairs - the
footprint check (subagents rule) now has a mechanical source.

Failure story: seat 5 enters an exploit loop; its container hits the CPU
cap; exit code recorded; monitor flags the seat; recovery ladder absorbs
the scope. The host and the other 7 seats never noticed. That is the
D11 payoff in one paragraph.

---

## APPENDIX S - RELATION TO THE WORKTREE SINGLE-WRITER RULE

Sandboxes isolate EXECUTION; worktrees isolate WRITES. They compose:

- git write isolation: one worktree per agent (worktrees-single-writer)
- execution isolation: one sandbox per run (this skill)
- merge isolation: the orchestrator owns merge (subagents S7)

A seat that writes to its worktree from inside its sandbox produces the
full isolation stack: no shared FS state, no shared process space, no
shared git index. The three layers (worktree / sandbox / scope gate)
cover writes, execution, and targets respectively - together they close
the shared-state failure class that the fleet measured repeatedly.

## APPENDIX T - FAQ ADDENDUM (operational)

**Q: Pre-pull timing (cold start)?**
Session start pulls the tier images it needs (digest-pinned). The pull
is logged; a failed pull fails the session pre-flight, not mid-task.

**Q: Do tests run inside sandboxes in CI?**
Yes when they execute external input; pure unit tests of trusted code
run host-tier (the honest skip table). CI is the same rules, automated.

**Q: Sandbox images bloat the disk?**
Prune nightly (untouched 24h), images digest-pinned and shared across
seats - the 8-seat fan-out shares ONE image, not 8. Disk budget lives in
the resource-governor contract.

---

## APPENDIX U - THE ESCALATION LADDER FOR ESCAPES

An "escape" is any observable effect outside the sandbox boundary
(host file changed, network call seen on the host side, process on the
host). The ladder:

1. FREEZE the class: all runs of that tier/class stop (fail-closed).
2. Capture evidence: sandbox audit line, host-side artifacts, the task
   input (the payload goes to a quarantine dir, read-only).
3. Root-cause with ideas (profile the escape path - mount, kernel,
   network, credential).
4. Tier escalation for the class: docker -> microvm; microvm -> manual
   review gate for that input class.
5. Report honestly in the runlog: what escaped, how, the fix, the test
   that now proves the fix (T-suite addition).

An escape is an incident, never a footnote - the entire value of the
gate is the 0 on that metric.

---

## APPENDIX U - CONFIG SNAPSHOT (what gets recorded per box)

The sandbox enablement state lives in the box config snapshot so any
fresh session can verify the layer exists:

```
sandbox-provision:
  runner: /usr/local/bin/sandbox_run.sh (sha <...>)
  profiles: none | registry | allowlist | shared-crew (defined)
  images: <digest list>
  prune: nightly (cron line)
  last-t-suite: <date> 12/12
```

The dual-root-config-law applies: the config lands on ALL roots on ALL
boxes, verified per root. A box without the record is process-tier by
default - and sessions must surface that, not assume.

---

## APPENDIX V - THE DECISION TABLE IN PRACTICE (three quick rulings)

Ruling 1: "parse this CSV the client emailed."
Class: external-parse. Tier: docker, network none. Verdict takes 5
seconds and the parse runs in the 1CPU/512M box. If the CSV is 40GB,
the caps make the failure cheap (OOM inside the sandbox) - the cap is
also a sizing signal (stream it instead).

Ruling 2: "run this SQL against the staging DB."
Class: cloned-code-ish with a credential. The credential is scoped,
read-only; the sandbox gets registry+db-host egress. The scope gate
records the DB host as in-scope (staging is named in the engagement
doc).

Ruling 3: "install this cool new MCP server and wire it."
mcp-auditor first; unverified -> microvm tier until 20 clean sessions.
The enthusiasm tax is one paragraph of patience; the incident it
prevents is the mcp-auditor class.

These rulings are the skill in miniature: the table decides, the gate
records, the report cites.

## APPENDIX W - FINAL SELF-CHECK

1. Is the tier lookup mechanical (table) or ad-hoc per task?
2. Did every run leave a start+exit audit pair?
3. Did the T-suite run at provision and after any profile change?
4. Are escapes 0 (or, if not, is the incident handled per Appendix U)?
5. Is the config snapshot current on all roots (dual-root law)?

A sandbox layer that cannot answer these five from files is decoration -
the honest answer is process-tier + flag, which the gates line supports.

---

## APPENDIX X - COMPOSITION EXAMPLE: THE FULL STACK ON ONE TASK

Task: "clone this public repo and run its demo, then commit the output."

```
1. scope gate:      git clone to workspace = in-scope (allow, logged)
2. sandbox:         cloned-code tier -> docker, allowlist egress,
                    2CPU/2G, in/ RO (repo mounted), out/ for results
3. demo runs:       hits its cap once (stricter than needed) - exit
                    code recorded, rerun with tuned caps from the tier
                    override file (not ad-hoc flags)
4. artifact gate:   out/ scanned (secrets, size, file list) before copy
5. worktree:        commit happens in the seat worktree (single-writer)
6. audit:           sandbox pair + scope allow + worktree commit hash -
                    three records, one task, all generated
```

The composition shows why the layers are separate skills: each answers
one question (may I? / how contained? / who writes?) and none of them
trusts the others. The report cites all three evidence trails - that is
what a 9.x security dimension looks like in practice.

---

## APPENDIX Y - THE INSTALL ORDER FOR THE FLEET (dependency-aware)

1. scope-enforcer hooks (context-free, protects immediately)
2. sandbox provisioning on the two VPS boxes (Appendix P)
3. workspace mode on the PC (the file-placement rules made mechanical)
4. microvm runtime on the web-dev VPS only (hostile-class workload
   lives there); other boxes stay docker-tier
5. T-suite across all boxes (recorded per box)

Order rationale: the scope gate has zero prerequisites and stops the
widest class; sandboxes need provisioning time; the microvm tier is
optional and only where the hostile class actually runs.

---

## APPENDIX Z - GLOSSARY (exact meanings in this skill)

- sandbox: an ephemeral execution boundary with caps, network profile,
  and teardown - container, microVM, or process-tier.
- hostile class: input designed or likely designed to attack the
  executor (pentest payloads, unverified MCP servers, model-generated
  code about to execute).
- escape: any observable effect outside the sandbox boundary. Zero is
  the only healthy number.
- artifact gate: the export scan (secrets, oversize, file list) that
  every artifact passes when leaving a sandbox.
- process-tier: OS job-object/process caps without a container runtime -
  honest containment, always flagged in reports.
- burned: a sandbox destroyed after one use; never reused.

---

## APPENDIX Z2 - WHAT THIS SKILL DOES NOT COVER (scope boundary)

- Secrets storage/rotation: secret-rotation owns the vault; this skill
  only consumes scoped, single-use credentials.
- Network-wide isolation (fleet egress policy, firewall baselines):
  infra, not per-run; the network profiles consume the baseline.
- Code-signing / artifact attestation: supply-chain-sentinel owns it.
- VM image building pipelines: dependency-advisory + supply-chain-
  sentinel run on the images; building them is infra work.

The boundary is deliberate: the sandbox is the EXECUTION layer. It does
not try to own the credential store, the network fabric, or the artifact
pipeline - it consumes their outputs and enforces its own boundary.

---

## APPENDIX Z3 - MICROVM TIER DETAIL (when and how)

When the hostile class is real (payloads from live targets, unverified
MCP servers), the container tier shares a kernel with the host - good
enough for accidents, not for adversaries. The microVM tier:

- Microsandbox or E2B self-hosted: VM-per-run, kernel isolated,
  sub-second cold start on modern hosts, same caps semantics.
- Same runner contract (sandbox_run.sh abstracts the tier: the task
  never knows).
- Audit lines carry tier=microvm; the T-suite runs with T10 as the
  headline check.

Costs: memory overhead per VM, one more runtime to maintain. That is why
it is reserved for the hostile class and promoted from ONLY after clean
sessions accumulate - the same earned-promotion arithmetic as the
review gate rollout.

---

## APPENDIX Z4 - EXTENDED TROUBLESHOOTING

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| sandbox cannot resolve DNS | network profile too tight | add resolver to the profile (documented), never disable the profile
| image pull fails at session start | digest unpinned or registry down | pre-flight fails loudly; pin fix + retry - do not fall back to host-tier silently
| caps cause legit OOM | tier defaults too small for the class | tune the class override file (versioned), re-run T-suite, record
| sandbox start adds seconds every run | pull-per-run regression | digest cache missing; restore pre-pull at session start
| two seats share one sandbox | runner misuse | one run one sandbox; the audit pairs are the proof
| artifact gate false-positives on big outputs | oversize threshold stale | adjust threshold with rationale comment; oversize is a finding only if unexplained
| monitor sees alive-but-flat sandbox | task hung inside | beat rule: pause, inspect from outside (docker exec logs), kill if 2 beats

---

## APPENDIX Z5 - WHY A SKILL AND NOT JUST INFRA (the doctrine note)

The caps and profiles are infra; what makes this skill is the DECISION
layer: which class a task belongs to, what the tier table says, what the
gates line records. Infra without the decision layer produces sandboxes
for theater and skips for convenience; the skill binds the decision to
the audit so the skip is visible and the tier is mechanical. Every hook
in this fleet obeys the same shape: enforcement is cheap, the record is
the product, and the T-suite is what makes it real.

---

## APPENDIX Z6 - EXAMPLE MANIFEST (adapter declaring sandbox needs)

```yaml
# adapter manifest fragment (adapter-spec composition)
sandbox:
  tier: docker            # adapter never gets to choose its own tier
  network: allowlist
  egress:
    - "api.target-tools.example:443"
  caps:
    cpu: 4
    mem: 8g
    pids: 256
    disk: 5g
  inputs:
    - mount: in/readonly
  outputs:
    - mount: out
  secrets:
    - { name: tool_api_key, delivery: scoped-file, single_use: true }
```

The adapter-spec interface gains this block; the runner validates it
against the tier table (an adapter requesting tier=microvm without a
hostile-class justification is refused). The declaration is reviewed at
adapter authoring time - at run time, the table decides and the gate
records.

---

## APPENDIX Z7 - THE 10-SECOND CHECK BEFORE ANY UNTRUSTED RUN

1. Which class is this input (tier table)?
2. Is the runner script the pinned one (not ad-hoc docker)?
3. Are caps + network profile from the table (not inherited)?
4. Is the teardown in the wrapper trap (not "later")?
5. Will the start/exit audit pair exist when this run ends?

Any "no" means the run is either mis-classified or unprotected - fix
the classification first; the sandbox exists to make class mismatches
cheap to catch, not invisible.
