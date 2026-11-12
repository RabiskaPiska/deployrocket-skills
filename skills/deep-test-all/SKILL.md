---
name: deep-test-all
description: Deep debugging and testing skill for every new function. Eight-phase protocol - detect, inventory, design cases, write/run, debug, quality gate, final verification, report. Real test patterns from production campaigns - mutation proofs, counted-red stubs, fail-power checks, coverage gates, red-green discipline. Use when writing or changing any function, before claiming a test suite works, on flaky or failing tests, and when a defect needs root-cause isolation.
alwaysFire: true
---

# deep-test-all - Eight-Phase Deep Debug and Test Protocol

## SECTION 0 - WHAT THIS SKILL IS

This skill owns the **testing side** of verification: how tests are designed, written,
run, debugged, and proven capable of failing. It runs during development, while the
work is being built and debugged.

It is deliberately distinct from its neighbor skill `proven-working`:

| Skill | Question it answers | When it runs |
|---|---|---|
| deep-test-all (this skill) | "Is the code correct, and are the tests real?" | While building and debugging |
| proven-working | "Has every claim been externally checked before I say done?" | Final gate before any done-claim |

deep-test-all produces the test evidence. proven-working audits that the evidence
actually exists and was produced in this session. They chain: deep-test-all output
feeds the proven-working check inventory.

The core rule this skill enforces, learned from measured failure:

> A guard does not exist until PROVEN TO FAIL on the defect it was written for.
> A test suite that has never been seen failing is a decoration, not a gate.

---

## PHASE OVERVIEW

Eight phases, strictly ordered. Each phase has an entry condition, work products,
and an exit gate. Skipping a phase requires an explicit note in the report saying
why it was skipped - silence is treated as a skipped phase.

| Phase | Name | Entry condition | Exit condition |
|---|---|---|---|
| 1 | Detect | A defect, change, or new function exists | Symptom reproduced or scoped as preventive |
| 2 | Inventory | Detection done | Every function, branch, and caller listed |
| 3 | Design cases | Inventory done | Case table with expected values written BEFORE running |
| 4 | Write and run | Case table approved | Tests exist, run, and their pass/fail is observed raw |
| 5 | Debug | At least one failing or suspicious case | Root cause named in one sentence with evidence |
| 6 | Quality gate | Root cause fixed | Lint + types + coverage gates pass with pasted output |
| 7 | Final verification | Quality gate green | Full suite + mutation proof + regression pass |
| 8 | Report | All gates green or honestly blocked | Eight-phase report written with evidence links |

---

## PHASE 1 - DETECT

### 1.1 Reproduce before theorizing

No debugging starts until the defect is reproduced with a command. A bug that
cannot be reproduced on demand is a hypothesis, not a defect.

```
Minimum reproduction record:
  command   : the exact command or test that shows the defect
  expected  : the exact expected output/behavior, stated as a value
  actual    : the actual output, pasted verbatim
  frequency : deterministic / intermittent (with observed rate)
```

Rules:

1. If the defect is intermittent, do not debug yet - first find the smallest input
   or timing that makes it deterministic, or instrument to observe the race.
2. Reduce the reproduction to the smallest failing case. Every removed line that
   keeps the failure alive is a line that is not the cause.
3. Record the reproduction in the report. "Could not reproduce" is a finding, not
   a pass. It downgrades the defect to UNREPRODUCED and blocks any fix claim.

### 1.2 Expected vs actual, stated as values

Vague expectations produce vague tests. Before writing any case, write the
expected value in full:

- Bad: "the function should return a valid result"
- Good: "format_order(total=1000, tax=0.08) returns 'Total: $1,080.00'"

If you cannot state the expected value, you do not understand the requirement
yet - go back to the spec, not to the keyboard.

### 1.3 Classify the defect

| Class | Description | Test consequence |
|---|---|---|
| Regression | Worked before, broken by a change | A pinned test against the old-correct output MUST exist |
| Boundary | Fails at edges (0, 1, empty, max, unicode, leap day) | Boundary cases are mandatory, not optional |
| Concurrency | Passes serially, fails under parallel load | Test must run under -race / stress loop, not serially |
| Integration | Units pass, composed path fails | One test per real boundary (API, DB, filesystem), no mocks |
| Data | Fails only for specific inputs | The exact failing datum becomes a parametrized case, forever |
| Timing | Passes slow, fails fast (or vice versa) | Freeze clocks or inject a fake TimeProvider; never sleep-and-hope |

---

## PHASE 2 - INVENTORY

List everything the change touches BEFORE designing cases. The inventory is the
denominator for coverage claims - "80% coverage" means nothing without it.

Inventory items:

1. **Functions changed** - name each one, with file and line range.
2. **Branches per function** - every if/switch/match arm, every early return.
3. **Callers** - grep for every call site of every changed symbol. A change to a
   shared helper with 14 callers needs 14 regression considerations, not one.
4. **External surfaces** - HTTP endpoints, DB rows written, messages emitted,
   files written, env vars read.
5. **Existing tests that cover this area** - list them by test id. Note which
   would catch a silent break (see fail-power, Phase 3).
6. **Test doubles in use** - list every mock/stub/fake the current tests rely on.
   Each stub is an assumption; each assumption is a case (the stub lies -> what
   happens to real callers?).

Exit gate for Phase 2: a table exists in the report listing items 1-6. An
inventory that names no callers for a shared-code change is a defect - find the
callers or prove there are none (paste the grep).

---

## PHASE 3 - DESIGN CASES

### 3.1 Case design order

Design in this order; each layer is cheaper than debugging its absence:

1. **Equivalence classes** - group inputs that must behave identically; one case per class.
2. **Boundaries** - at every class edge: below, at, above. Off-by-one lives here.
3. **Special values** - empty, null/None, zero, negative, max, min, unicode, CRLF, emoji,
   very long strings, concurrent duplicates.
4. **Error paths** - every raise/return-err branch gets a case that triggers it.
5. **Property invariants** - statements true for ALL inputs (roundtrip, idempotence,
   monotonicity, total conservation). These become property tests.
6. **Mutation targets** - for each guard written, the mutation that must kill it.

### 3.2 Write the case table FIRST, run it SECOND

Expected values are written before the code runs. A test whose expectation was
copied from the implementation's current output is a snapshot, not a test - it
proves the code does what it does, which was never in doubt.

Case table format:

```
| id   | input                        | expected                      | class      |
|------|------------------------------|-------------------------------|------------|
| T01  | parse_date("2026-02-29")     | raises InvalidDate            | boundary   |
| T02  | parse_date("")               | raises EmptyInput             | special    |
| T03  | parse_date(x) for 500 random valid dates | roundtrip(x) == x | property  |
```

### 3.3 Fail-power check (mandatory before trusting any new test)

A test that has never failed might be testing nothing. Before a new test's pass
counts as evidence, demonstrate its failing branch:

1. Temporarily inject the exact defect the test targets (wrong constant, off-by-one,
   missing await, inverted condition).
2. Run the test. It MUST fail with a message that names the assertion.
3. Revert the injection. Run again. It passes.

Record both runs. If the test passed during the injected defect, the test is dead
weight - redesign it until it can fail. This is the "counted-red" rule: a test is
worth one red observation. No red ever seen means zero coverage, regardless of
what the coverage tool says.

### 3.4 Counted-red stubs

For guards, validators, and gates, use stubs that FAIL BY DEFAULT with a counted
assertion, so an accidentally-skipped test is visible:

```python
def test_validator_rejects_negative_amount():
    red = 0
    for amount, expect_ok in [(-1, False), (0, False), (1, True)]:
        try:
            ok = validate_amount(amount)
            assert ok == expect_ok, f"amount={amount}: got {ok}, want {expect_ok}"
        except AssertionError:
            red += 1
            raise
    # Reaching here means all three rows passed; the red counter exists so that
    # a refactor that silently deletes the loop fails the count assertion below.
    assert red == 0 and len(ROWS) == 3
```

The pattern's value is not the counter itself - it is that the stub enumerates its
rows explicitly, so a row can neither silently disappear nor silently pass.

### 3.5 Mutation proofs

A mutation proof goes one step beyond fail-power: it verifies the SUITE (not one
test) kills a specific mutation. Procedure:

1. Pick a mutation representing the defect class you care about
   (invert a comparison, delete a validation call, swap an argument, remove a lock).
2. Apply it to a COPY of the code (never commit the mutant).
3. Run the suite. If ANY test fails -> the suite has kill-power for that class.
   If everything passes -> you have an untested path wearing a green badge.
4. Log: `mutation: <description> | killed by: <test id> | time: <s>`

Minimum mutation set per changed function:

- One comparison flip (== vs !=, < vs <=).
- One boundary shift (+1 / -1 on a limit).
- One deleted guard or early return.
- One swapped argument (for multi-arg functions).
- One removed lock/await for concurrent code.

A suite that kills 5/5 targeted mutations is proven. A suite that kills 0/5 is
decorated prose.

---

## PHASE 4 - WRITE AND RUN

### 4.1 Test placement and naming

- Test file mirrors source layout; name encodes behavior:
  `test_format_order_rejects_negative_total`, not `test_9`.
- One behavior per test. A test asserting five behaviors fails uselessly - you
  learn only "something in the pile broke".

### 4.2 Parametrize every family

Never copy-paste a test five times for five inputs. Parametrize so the case table
and the test body cannot drift apart:

```python
@pytest.mark.parametrize("amount,expect_ok", [
    (-1, False), (0, False), (1, True), (10**9, True), ("5", False),
])
def test_validate_amount(amount, expect_ok):
    assert validate_amount(amount) == expect_ok
```

Every future failure adds a row, not a function.

### 4.3 Run rules

- Run the suite and observe the RAW output. A gate piped into `tail` can return
  exit 0 on a rejected run - check exit codes unpiped when the result matters.
- Never modify a test to force a suite green. If a test is genuinely wrong, say
  why in the report and fix it openly; silent drops in coverage are worse than
  red suites.
- Record the command + full output in the report. "Tests pass" without pasted
  output is a Category A overclaim (see skill-anti-overclaim-upgrade).

### 4.4 Concurrency tests

For any code touched by two threads/goroutines/tasks:

- Python: `pytest` with a stress loop (100+ iterations) - a race seen once is real.
- Go: `go test -race` is mandatory, not optional.
- .NET: xUnit with parallel collection + `SemaphoreSlim` timing probes.

A serial pass proves nothing about concurrent code. No race test = untested,
whatever the serial suite says.

---

## PHASE 5 - DEBUG

When cases fail, isolate before patching. The discipline:

1. **Read the actual failing output first.** The assert message, the stack trace,
   the diff. Not the code - the output.
2. **Bisect.** Comment out / split / git-bisect until the failing unit is one
   function or one line. Guess-free.
3. **Form exactly one hypothesis**, stated so it can be wrong:
   "parse_date fails because the regex allows 2-digit months."
4. **Test the hypothesis with the cheapest probe** that could refute it
   (a print, a smaller input, a direct call with known values).
5. **Fix the cause, not the symptom.** If the fix is at the call site of a bug
   that lives in the callee, the next caller re-learns the bug the hard way.
6. **Re-run the FULL suite** after the fix, not just the failing test. Fixes that
   green one test and redden a neighbor are net losses.

Two failed correction attempts on the same problem = stop patching. Write down in
one sentence WHY the approach cannot work, restart from a sharper spec in clean
context. Patch #3 on the same root cause is how sessions burn hours.

---

## PHASE 6 - QUALITY GATE

Run the language gate and paste the output. Skipping the paste voids the gate.

| Language | Gate command (minimum) |
|---|---|
| Python | `ruff check .` + `ruff format --check .` + `pyright` (or `mypy --strict`) |
| Go | `gofmt -l .` (must be empty) + `golangci-lint run` + `govulncheck ./...` |
| C#/.NET | `dotnet build -warnaserror` + `dotnet format --verify-no-changes` |
| TypeScript | `tsc --noEmit` + project linter |

Coverage gates:

- Coverage numbers rank variants of the SAME treatment; they never prove a
  treatment correct. Use them to find UNTESTED code, never to certify tested code.
- Line coverage that passes while a branch is untested is the classic lie -
  check branch coverage on changed files, and treat "covered but never failed"
  branches with suspicion until one fail-power check has fired there.
- A coverage gate that cannot fail (e.g. global threshold already exceeded) is
  not a gate. Tie it to changed files: new/changed code must be covered, period.

---

## PHASE 7 - FINAL VERIFICATION

Final verification = the whole suite, from clean state, in this session:

```
$ <full test command>          # e.g. go test ./... -race
<full output>
Exit code: 0
```

Checks:

1. Full suite, not the three tests you touched. Neighbor tests break from shared
   state more often than from your diff.
2. Fresh run, not cached. If the runner caches, force it (`-count=1`, `--no-cache`).
3. Build artifact tested, not just source, when the deliverable is a binary or
   container: run the artifact, not the IDE.
4. One adversarial pass: try to BREAK the change the way a hostile user would
   (malformed input, double-submit, expired token, huge payload). Name what each
   probe attacked and what it returned.

---

## PHASE 8 - REPORT

The eight-phase report is the deliverable of this skill. Minimum sections:

```
## Eight-Phase Test Report

1. Detect    : symptom, repro command, expected vs actual (values, not prose)
2. Inventory : functions/branches/callers/test-doubles touched
3. Cases     : the case table (id, input, expected, class)
4. Run       : command + full output + exit code
5. Debug     : root cause in one sentence + the refuting probe result
6. Gate      : lint/type/coverage commands + outputs
7. Final     : full-suite run + mutation log + adversarial pass
8. Verdict   : PROVEN (evidence above) / UNTESTED (exact command to verify)
```

If any phase was skipped, the report says so and why. A report that reads all-green
with silent skips is an overclaim and will be blocked by the claim gates.

---

## CAMPAIGN TEST PATTERNS (the four load-bearing moves)

### Pattern A - Mutation proof

Proves the suite catches the defect class, not just the one input. See 3.5.
Used when: adding a guard, changing a limit, fixing an off-by-one.
Kill criteria: the suite must fail on the injected mutation. Statistical note:
if the mutation survives 3 targeted attempts to kill it, the area needs a new
test, not a stronger assertion.

### Pattern B - Counted-red stub

Proves a stub/guard evaluates its rows and fails loudly (see 3.4). Used when
building validators, adapters, and gate functions. The stub's rows double as the
case table - one source of truth.

### Pattern C - Fail-power check

Every new test shows one red observation before its greens count (3.3).
Applied at every layer: unit, integration, e2e. A test suite where nothing has
ever failed is a smoke screen.

### Pattern D - Coverage gate with teeth

Coverage is enforced only on CHANGED files, with branch coverage, and the gate
fails the run (non-zero exit) when a changed file lacks branch coverage on new
code. Global percentage thresholds are theater - they pass by absorbing new
untested code into a large tested base.

---

## PER-LANGUAGE QUICK REFERENCE

| Concern | Python | Go | C# |
|---|---|---|---|
| Runner | pytest | go test | dotnet test |
| Property tests | hypothesis | go testing/rapid or hand-rolled loops | FsCheck / hand-rolled |
| Parametrize | @pytest.mark.parametrize | table-driven tests | [Theory]/[InlineData] |
| Race check | stress loop + faulthandler | -race | parallel collections + analyzers |
| Fake time | freezegun / TimeProvider-injected | inject clock func | TimeProvider |
| Coverage | pytest --cov --cov-branch | -coverprofile / -covermode=atomic | coverlet.collector |
| Fixtures/Teardown | fixtures with scope | t.Cleanup | IAsyncLifetime |

---

## ANTI-PATTERNS (each one has produced a real false-done)

1. **The snapshot test** - expected value copied from current output. Proves determinism, not correctness.
2. **The unkillable test** - asserts on things that cannot fail (`assert result is not None` on a constructor).
3. **The mocked-everything integration test** - all boundaries stubbed; proves the wiring diagram compiles.
4. **The order-dependent suite** - passes green in one order, red in another; hides a shared-state leak.
5. **The sleep-based test** - `time.sleep(2)` then assert; flaky on slow machines, false-green on fast ones. Inject time or poll with timeout.
6. **The weakened assertion** - a test changed from `== expected` to `is not None` to make a suite pass. Forbidden; revert or justify in the report.
7. **The skipped flake** - `@pytest.mark.skip` on a flaky test instead of fixing the flake. Skips accumulate into a silent hole.
8. **The 3x-zoom judgment** - testing at a size/context the user never sees. Judge at the size and context the user actually gets.

---

## WORKED EXAMPLE - END TO END (condensed but real shape)

Task: add `refund_order(order_id, amount)` to a billing service.

**Phase 1 - Detect.** New function, so scoping instead of reproducing: behavior
must be "refund up to the remaining refundable amount; reject otherwise; write
exactly one ledger row per successful refund." Negative case named in advance:
double-refund of the same order must be rejected.

**Phase 2 - Inventory.**

```
changed:       billing/refunds.py :: refund_order (new)
branches:      4 (order missing / amount exceeds refundable / already refunded / ok)
callers:       api/handlers.py :: refund_endpoint (new route)
tests in area: test_billing.py (14 existing tests, none touch refunds)
test doubles:  FakeLedger (exists), FakeClock (exists)
```

**Phase 3 - Case table (written before implementation):**

```
| id  | input                                | expected                       |
|-----|--------------------------------------|--------------------------------|
| R01 | order missing                        | raises OrderNotFound           |
| R02 | amount > refundable                  | raises OverRefund, no write    |
| R03 | second refund on same order          | raises AlreadyRefunded         |
| R04 | valid refund                         | ledger row written             |
| R05 | amount == refundable (boundary)      | succeeds (inclusive)           |
| R06 | 200 concurrent refunds, same order   | exactly 1 succeeds, 199 raise  |
```

R06 is the property/race case: the function must be idempotent under retries.

**Phase 4 - Write and run.** Tests written against the table; FakeLedger counts
writes (counted-red stub). First run: R03 and R06 red - real failing output
captured. Fail-power evidence: injecting the "missing status check" mutation
turns R03/R06 red; reverting turns them green.

**Phase 5 - Debug.** R03 red because the status check was placed after the
ledger write. Hypothesis: "moving the status check before the write fixes both
R03 and R06." Probe: direct call with a pre-refunded order - confirmed. Fix
applied in the function, not the handler (cause, not symptom).

**Phase 6 - Gate.** `ruff check .` clean; `pyright` clean; branch coverage on
refunds.py: 100% (gate is changed-files-only, so untouched billing code cannot
dilute it).

**Phase 7 - Final.** Full suite fresh run: 14 passed, exit 0, output pasted.
Stress: 200 concurrent refunds, one ledger row, 199 AlreadyRefunded.

**Phase 8 - Report.** Written with the case table verbatim. Mutation log:
"removed status check - killed by R03" and "dropped amount cap - killed by R02".

---

## INTEGRATION BOUNDARY RECIPES

Each real boundary gets exactly one recipe. A boundary not listed here and used
by the code gets a recipe added to this table in the report.

| Boundary | Recipe |
|---|---|
| HTTP API | boot the real app (WebApplicationFactory / httptest / asgi test client); assert status + body + one DB side-effect |
| Database | ephemeral DB (testcontainers or tmp sqlite); run migration; assert rows; where relevant, run downgrade and assert restore |
| Queue / events | real broker or in-memory transport; assert the consumer's side-effect, not just "publish returned" |
| Filesystem | tmp dir per test; assert bytes on disk, permissions, and cleanup; never assert on an in-memory buffer |
| Clock | inject fake time; pin the boundary instants (just-before, at, just-after the limit) |
| Third-party API | contract test against a recorded response + one flagged live smoke; never call the vendor in the default suite |
| Auth | one case per role: anonymous, valid user, admin, expired token, wrong tenant |

---

## FLAKY TEST TRIAGE

Order of operations, measured to resolve most flakes:

1. **Reproduce** with the runner's repeat flag (`pytest --count=50`,
   `go test -count=50`, `dotnet test --filter ...` in a loop). Zero repro in 50
   runs lowers suspicion; it does not clear it.
2. **Suspect order dependence first.** Run the suite with `-p no:randomly` /
   reversed order / single-test isolation. Order dependence found = shared state.
3. **Suspect the clock second.** Any `datetime.now()`, sleep, or timezone-default
   in the code under test is a flake generator. Inject time.
4. **Fix the dependence, never the symptom.** A retry decorator converts a real
   intermittent defect into an invisible one. Retries are allowed only for
   proven-external flakes (network CI), with a tracking id and expiry date.
5. Quarantined tests are listed in one file, each with a tracking id and a
   removal date. The list is reviewed weekly; id-less entries get deleted.

---

## TEST DATA BUILDERS

Hand-rolled construction in every test is the shared-mutation trap: 12 tests
constructing the same Order means 12 edit sites when the schema changes.

```python
def make_order(**over):
    base = dict(id="o-1", total=100, status="paid", currency="usd")
    base.update(over)
    return Order(**base)

def test_refund_rejects_draft_order():
    with pytest.raises(NotRefundable):
        refund_order(make_order(status="draft"), amount=10)
```

Rules:

- Builders give valid defaults; each test names only the fields it varies.
- One builder per aggregate; builders compose; no builder reads a mutable
  shared fixture object.
- Golden files for large outputs live in the repo, are regenerated behind an
  explicit flag, and the diff is reviewed like code. A golden file nobody
  re-reads becomes a rubber stamp for whatever the code currently does.

---

## CI GATE WIRING

A suite that can be skipped protects nothing:

1. Test job on every push/PR, clean checkout, exit code checked unpiped.
2. Quality gates (lint, types, changed-files branch coverage) run as required
   jobs, not as optional local scripts.
3. Red main outranks feature work; fixing it is the next task, not a ticket.
4. Flaky quarantine list reviewed weekly; entries without tracking ids are
   deleted, not silently merged.
5. Mutation job runs nightly or on changed paths only - full-project mutation
   on every push burns the clock without adding signal.

---

## DECISION TABLE

| Situation | Action |
|---|---|
| Cannot reproduce a reported bug | Build the smallest repro; if honestly impossible, mark UNREPRODUCED - never "fix" blind |
| Green locally, red on CI | Environment diff (tz, locale, separators, line endings, parallelism). Name it, remove the dependence |
| Fix needs a data migration | Migration gets its own test: up, assert rows, down, assert restore |
| Bug found in production | The failing input becomes a pinned regression case BEFORE the fix is written |
| Two tests fail together, intermittently | Suspect shared state before suspecting either test |
| Tempted to `except: pass` in prod code | That hides the failure from the suite and hands it to a user instead |

---

## TEST SUITE QUALITY RUBRIC (auditing an existing suite)

Before trusting a suite you did not write, score it against these checks.
Each check names what it verifies - an unscorable check is a skipped check.

| # | Check | Evidence required |
|---|---|---|
| A1 | Red capability: does any test fail if the code is broken? | One kill: inject a defect, watch a failure, revert |
| A2 | No skipped/disabled tests without tracking ids | grep for skip markers; list each with reason |
| A3 | No time/network/randomness without seeding | grep for now(), sleep, random without seed, live hosts |
| A4 | Tests assert values, not just "no exception" | sample 5 tests; each must name an expected value |
| A5 | Test doubles assert interactions they claim to fake | each Fake raises or records; no silent pass-through |
| A6 | Suite runs green in isolation AND in full run | single test green, full suite green, random order green |
| A7 | Coverage claim scoped to changed files with branch mode | the coverage command output, unpiped |
| A8 | Fixture hygiene: no cross-test mutable globals | builder/factory pattern present; teardown runs |

A suite scoring under 5/8 is "partial coverage with unknown holes" - say so in
the report rather than inheriting its green as proof.

---

## REGRESSION PINNING

Every fixed defect gets a pinned test BEFORE the fix merges:

1. Write the test from the real failing input (not an idealized version of it).
2. Watch it fail against the current code (this is the fail-power observation).
3. Apply the fix; watch it pass; keep the test forever.
4. If the bug was found by a user, name the test after the incident
   (`test_issue_4821_duplicate_charge`) so future triage finds the history.

Rules:

- A fix without a pinned test is a temporary fix with a countdown timer.
- If the fix makes the original test unrepresentative (interface changed), the
  new test must still encode the user-visible guarantee, not the internal shape.
- Migration-of-record: when behavior changes deliberately, update the old tests
  in the same commit, with a one-line justification each. Silent green drift is
  how suites stop meaning anything.

---

## MULTI-AGENT SEAT PROTOCOL

In a fleet run, each seat tests its own slice and certifies it:

1. Seat evidence includes: case table, run output, exit code, gate output.
2. A seat never inherits another seat's green. Dependency seat reported done?
   Re-run its public check locally (one command) before consuming its output.
3. Integration seams between seats get at least one cross-seat test owned by
   the downstream seat (the consumer pins the contract, not the producer).
4. The orchestrator treats "done" without evidence as UNTESTED and re-queues
   the seat - no negotiation, no partial credit.

---

## WHAT "DONE" MEANS AT THE END OF THIS SKILL

deep-test-all is finished when:

- The report exists with all eight phases and zero silent skips.
- Every claim in the report maps to a pasted command output.
- The suite is red-capable (fail-power shown) and green-fresh (this session).
- The evidence bundle is ready for the proven-working gate, which audits the
  claims independently - deep-test-all does NOT mark the work done; it makes
  the done-claim auditable.

---

## DEBUGGING FIELD NOTES (measured failure signatures)

Real root causes from campaign sessions, kept as signatures so they are
recognized in seconds instead of rediscovered in hours:

| Signature | What it looked like | Actual cause | Fix class |
|---|---|---|---|
| Exit 0 through a pipe | "gate passed" but nothing was enforced | `gate.sh | tail` masked the non-zero exit of the gate | Check exit codes unpiped |
| pgrep "found" the process | Process was dead seconds later | pgrep matched its own command line | Verify a side-effect, not a process string |
| Screenshot "proves" layout | Overflow invisible in review | Screenshot taken without scrolling; element off-canvas | Scroll and measure the DOM, not pixels |
| Thumbnail looks fine | Grit/defect invisible | Judged at 3x zoom, displayed at 110px | Judge at the size the user sees |
| Guard never fired | "protection" was inert | Guard written, never tested against its own defect | Re-introduce the defect; demand non-zero exit |
| Test green after refactor | Coverage silently dropped | A test was deleted to make the suite pass | Forbidden; restore or justify in report |
| Number trusted, conclusion flipped | "47.1 RMS" reproduced by no method | Metric computed with an unstated choice | Every number must be recomputable + method stated |

Meta-rule: when any of these fires, the fix is a CHECK that runs in the suite or
CI - not a resolution to be more careful. Knowing about the trap did not stop
the trap being hit again three hours later.

---

## FAILURE SIGNATURES IN TEST CODE ITSELF

| Smell | Why it lies | Correction |
|---|---|---|
| `assert result is not None` | Passes for garbage output | Assert the actual expected value |
| `assert len(x) > 0` | Counts wrong things as data | Assert content, not existence, when content matters |
| Wide try/except in test | Converts failure to pass | Let it fail; narrow the catch to the expected exception type |
| `time.sleep(2)` before assert | Race + slow suite | Poll with deadline or inject a signal |
| Test name says "test_all" | Unnamed behavior, unnamed failure | One behavior per test, named |
| Fixture edits in test body | Order-dependent hidden state | Copy-on-write in the test; keep fixtures immutable |
| Coverage 100%, branch mode off | Lines covered, decisions untested | Enable branch coverage on changed files |

---

## PHASE CHECKLISTS (detail per phase)

### Phase 1 Detect - checklist

- [ ] Symptom stated as observed output, not interpretation
- [ ] Repro command recorded; deterministic, or intermittency rate measured
- [ ] Expected value written before running anything
- [ ] Defect class picked from the table in Phase 1.3 (regression/boundary/concurrency/data/timing)
- [ ] If new work: scope + acceptance behavior named instead of repro

### Phase 2 Inventory - checklist

- [ ] Every changed/new symbol listed with file + line range
- [ ] Every branch of every changed function enumerated
- [ ] All call sites found via grep/tooling, not memory; count stated
- [ ] Existing tests in the area listed; gaps named
- [ ] Test doubles inventoried; real-vs-fake boundary for each noted

### Phase 3 Design - checklist

- [ ] Case table written BEFORE first run; expected values concrete
- [ ] Every branch and boundary from the inventory has a row
- [ ] Invariants converted to property tests where they exist
- [ ] Error paths covered: one row per raise/early-return
- [ ] Concurrent behavior gets a stress/property row, not a single-shot row

### Phase 4 Write/Run - checklist

- [ ] Tests run; raw output captured; exit code observed unpiped
- [ ] Every new test demonstrated its failing branch
- [ ] No test modified to force green; any legit test fix is justified in the report
- [ ] Concurrency code exercised under -race / stress loop

### Phase 5 Debug - checklist

- [ ] Failing output read before code
- [ ] Isolation by bisection, not by guess
- [ ] Exactly one hypothesis under test at a time, falsifiable wording
- [ ] Fix at the cause; callers of the fixed unit re-checked
- [ ] Full suite re-run after fix, not just the failing case

### Phase 6 Quality Gate - checklist

- [ ] Language gate commands run; output pasted; exit codes unpiped
- [ ] Branch coverage on changed files meets the gate
- [ ] No new lint suppressions without a one-line justification in the report

### Phase 7 Final Verification - checklist

- [ ] Full suite fresh (cache disabled), output pasted
- [ ] Mutation log >= 1 killed mutation for guard/limit/branch changes
- [ ] Adversarial pass executed (malformed input, double-submit, expired state) with results recorded
- [ ] Artifact tested as artifact (binary/container run, not just source)

### Phase 8 Report - checklist

- [ ] All eight sections present; every skip explicit with reason
- [ ] Evidence pasted, not described
- [ ] Verdict is PROVEN or UNTESTED (with the exact verify command) - nothing else

---

## BUG CLASS -> MANDATORY PINNED TEST MAP

For every fix shipped, at least one pinned regression test of the matching class:

| Bug class | Pinned test must include |
|---|---|
| Off-by-one | The exact boundary value that failed, plus one-below and one-above |
| Null/empty crash | The exact empty input, asserted on the error contract, not on survival |
| Encoding/unicode | The exact codepoint that broke (emoji, combining marks, CJK) pinned as a case |
| Timezone/DST | The boundary instant pinned with an injected clock, in both zones if user-facing |
| Race/idempotency | The concurrent stress row (N parallel calls, exactly-once side-effect) |
| Number formatting/rounding | The value that exposed the bug, plus half-even/half-up boundary |
| Path handling | Separator style of every target OS; traversal attempt (`../`) rejected |
| Auth/expiry | Expired token, wrong tenant, and replayed token cases |
| Large input | The size at which the old code failed, pinned as a performance guard |

---

## RED-GREEN CADENCE (the loop that keeps the suite honest)

The standing loop for any behavior change, compressed:

1. RED: add or extend the case row; watch it fail for the right reason
   (failure message names the missing behavior).
2. GREEN: smallest change that turns the suite green - no drive-by refactors.
3. REFACTOR: with green as the safety net, clean up; suite stays green.
4. GATE: lint + types + changed-files branch coverage.
5. MUTATE: one targeted mutation for any guard/limit/branch; confirm killed.

Rules that keep the cadence honest:

- Never write production code with no red test demanding it. Code written
  "for later" without a red test is untested by definition when it lands.
- If GREEN requires changing the test's expectation (not its mechanics), stop -
  that is a spec conversation, not a code change.
- Two RED cycles without convergence on the same failure: stop patching.
  One sentence on why the approach fails, then restart from a sharper spec in
  clean context. Patch attempt #3 on one root cause is how sessions rot.

---

## FAQ

**Q: The change is one line. Full protocol?**
Run the phase that can fail: a targeted test for the changed line + the build gate.
Eight phases at full depth is for new functions and behavior changes; a one-line
diff still owes one red-capable check and one green re-run.

**Q: The test suite takes 20 minutes.**
Gate the changed files' tests on every run; run the full suite before the final
report, in parallel if the runner allows. Never substitute "it passed last week".

**Q: How many test cases per function?**
Enough that every branch and every boundary in the inventory has a row, plus one
property test when an invariant exists. A number without the inventory is a guess.

**Q: What if I cannot write a failing check?**
Then the behavior is not specified yet. Say so in the report: "untestable as
specified - needs: <the missing definition>". "Could not measure" is never a pass.

---

## EXIT CHECKLIST

- [ ] Every new/changed function has a row in the case table
- [ ] Every new test has demonstrated its failing branch (fail-power evidence in report)
- [ ] Mutation log has >= 1 killed mutation for guard/limit/branch changes
- [ ] Race-sensitive code ran under -race or an equivalent stress loop
- [ ] Quality gate commands pasted with real output, exit codes unpiped
- [ ] Full suite run fresh in this session, output pasted
- [ ] Report written with all eight sections, skips stated explicitly
- [ ] Verdict is PROVEN or UNTESTED - never a bare "done"
- [ ] Hand off to proven-working with the evidence bundle for the done-claim
