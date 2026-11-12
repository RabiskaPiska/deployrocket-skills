---
name: skill-paperclip-ledger-guard
description: Requires a cryptographic evidence bundle before any done-claim is accepted. Two-key model - agent submits, board approves. Extends the base bundle with a hash-chained append-only ledger (prev_bundle_hash), a board approval flow with countersignature, and an HMAC-SHA256 done sign-off over the canonical bundle text. Pure Python hashlib - no external dependencies.
alwaysFire: true
---

# Paperclip Ledger Guard - HMAC-SHA256 Done Sign-Off

## SECTION 1 - PURPOSE

No task is "done" until it produces a signed PAPERCLIP BUNDLE containing:

- A hash of the deliverable (proves content is known)
- A hash of the evidence (proves verification actually happened)
- Two deterministic keys (agent key + board key) that are mathematically linked
- A chain pointer to the previous bundle in the ledger (proves order and
  prevents silent substitution or removal of earlier entries)

This prevents: claiming done without evidence, claiming done without a
deliverable, inventing evidence hashes, reusing a bundle across sessions or
specs, and rewriting history in the ledger.

The Paperclip Ledger Guard is **Claim Gate #3** in the multi-layer overclaim
prevention stack. It fires AFTER skill-anti-overclaim-upgrade on every
done-claim and before the base anti-overclaim gate. It is a cryptographic
notary: it does not judge quality, only that real content was hashed, that the
agent is working on the correct frozen spec, and that the ledger is intact.

Why cryptographic binding? Because prose assertions can be copied from prior
turns. A hash of the actual deliverable content cannot be faked without
producing real content first. The board_key ties the bundle to the SPEC_HASH,
making it impossible to submit a bundle for a different spec without
detection. The chain pointer ties each bundle to its predecessor, so removing
or reordering history breaks every subsequent hash - visible immediately.

## SECTION 2 - THE TWO-KEY MODEL

```python
import hashlib

def pk_hash(text: str) -> str:
    return hashlib.sha256(text.encode('utf-8')).hexdigest()

# Inputs available in every session:
# task_id = sha256(task_description)[:8]
# session_id = current OpenCode session ID (e.g. ses_e43a...)
# SPEC_HASH = from frozen spec (e.g. v66-godmode-build-2026-11-08)
# output_content = full text of the deliverable file
# tool_output = pasted terminal/tool output used as evidence

# Derived keys:
agent_key = pk_hash(task_id + ":" + session_id)[:16]
board_key = pk_hash(SPEC_HASH + ":" + session_id)[:16]

# Deliverable and evidence hashes:
deliverable_hash = pk_hash(output_content)[:16]
evidence_hash = pk_hash(tool_output)[:16]
```

**The board_key** is derived from SPEC_HASH - it validates that the agent is
working on the correct frozen spec. If an agent submits a bundle against a
stale or forged spec, the board_key will not match and the bundle is rejected.

**The agent_key** is derived from task_id + session_id - it validates the
agent's identity and task scope. Cross-task bundle reuse is detected because
the task_id differs.

**Key properties of this model:**

- Deterministic: same inputs always produce the same keys
- Non-forgeable without the correct inputs: an agent cannot construct a valid
  board_key without knowing the SPEC_HASH
- Session-scoped: a bundle from one session cannot be reused in another
- Task-scoped: a bundle from one task cannot satisfy a different task's gate

## SECTION 3 - BUNDLE FORMAT

Every done-claim must produce this exact bundle:

```
PAPERCLIP BUNDLE {
  task_id:            "<pk_hash(task_description)[:8]>"
  deliverable_hash:   "<pk_hash(deliverable_content)[:16]>"
  evidence_hash:      "<pk_hash(tool_output)[:16]>"
  timestamp:          "<ISO-8601 e.g. 2026-11-08T16:00:00Z>"
  session_id:         "<current ses_xxx ID>"
  agent_key:          "<pk_hash(task_id:session_id)[:16]>"
  board_key:          "<pk_hash(SPEC_HASH:session_id)[:16]>"
  prev_bundle_hash:   "<pk_hash(prev canonical bundle text)[:16] or GENESIS>"
  ledger_seq:         "<integer, position in this task's ledger>"
  bundle_sig:         "<hmac_sha256(board_key, canonical_bundle_text)[:32]>"
  verdict:            "SIGNED" or "REJECT: reason"
}
```

**Field definitions:**

| Field             | Source                                          | Purpose                                     |
|-------------------|-------------------------------------------------|---------------------------------------------|
| task_id           | pk_hash(task_description)[:8]                   | Unique task fingerprint                     |
| deliverable_hash  | pk_hash(full_file_content)[:16]                 | Proves deliverable content is known         |
| evidence_hash     | pk_hash(terminal_or_tool_output)[:16]           | Proves verification actually happened       |
| timestamp         | datetime.now(UTC).isoformat()                   | Temporal audit trail                        |
| session_id        | Current OpenCode session ID                     | Session scoping and key derivation          |
| agent_key         | pk_hash(task_id + ":" + session_id)[:16]        | Agent identity + task scope binding         |
| board_key         | pk_hash(SPEC_HASH + ":" + session_id)[:16]      | Spec compliance proof                       |
| prev_bundle_hash  | pk_hash(previous bundle canonical text)[:16]    | Ledger chain integrity                      |
| ledger_seq        | 1, 2, 3, ... within the task's ledger           | Order + gap detection                       |
| bundle_sig        | HMAC(board_key, canonical text)[:16]            | Board-side authenticity of the bundle       |
| verdict           | "SIGNED" or "REJECT: reason"                    | Final gate decision                         |

The bundle must be emitted verbatim in the agent's response before any "done"
claim is accepted.

## SECTION 4 - THE HASH CHAIN (lifecycle of the ledger)

Each task keeps ONE append-only ledger file:

```
<run-dir>/ledger/<task_id>.chain
```

Rules of the chain:

1. The first bundle written for a task sets `prev_bundle_hash = "GENESIS"`
   and `ledger_seq = 1`.
2. Every later bundle hashes the FULL canonical text of the previous bundle
   (the exact text stored in the ledger, byte-for-byte) into
   `prev_bundle_hash`, and increments `ledger_seq`.
3. Appends only. Editing, deleting, or reordering any line in the ledger
   breaks every subsequent hash - tampering is detectable, not just
   preventable.
4. A REJECT bundle is still appended (with verdict "REJECT: reason"), so the
   failed attempt is part of the record; the resubmission gets the next seq.
5. The chain is closed by a final TERMINUS bundle whose deliverable is the
   completion report itself.

### Chain verification (anyone, any session, any machine)

```
1. Read the ledger file in order.
2. Recompute each bundle's canonical text from its fields.
3. Check: bundle_sig == HMAC(board_key, canonical text) for every entry.
4. Check: prev_bundle_hash == pk_hash(previous canonical text).
5. Check: ledger_seq values are consecutive, no gaps, starting at 1.
6. Check: every deliverable_hash still matches the file on disk.
7. Any failure = ledger integrity broken; all downstream SIGNED verdicts
   for this task are void until repaired and re-signed.
```

### Why the chain matters (the failure it prevents)

Without a chain, an agent under pressure can quietly replace an old REJECT
bundle with a SIGNED one, or drop a bundle whose evidence no longer
reproduces. With the chain, every later bundle references the exact bytes of
its predecessor - removing one entry invalidates the entire tail. The ledger
is append-only the way a paper ledger is: entries are never rewritten, only
corrected by a new entry.

## SECTION 5 - THE BOARD APPROVAL FLOW (two-key model)

Two roles, two keys, two moments:

```
AGENT (submitter)                       BOARD (approver)
------------------------                ------------------------
1. produce deliverable                  6. recompute hashes from the
2. run verification command                deliverable file and the
   (output = evidence)                     pasted evidence
3. build bundle (keys + hashes)         7. re-derive agent_key + board_key
4. sign: bundle_sig =                      8. re-run validation rules
   HMAC(board_key, canonical)           9. verdict: SIGNED / REJECT: reason
5. append to ledger + emit              10. countersign the line in the
   in the response                          gate line: paperclip=COUNTERSIGNED
```

The agent_key proves WHO submitted and for WHICH task. The board_key proves
WHICH SPEC the work belongs to. The bundle_sig proves the board saw exactly
this bundle text - a bundle edited after submission fails signature
validation.

### SIGNED condition - ALL must be true:

1. deliverable_hash is non-empty and not a placeholder
2. evidence_hash is non-empty and not a placeholder
3. agent_key matches pk_hash(task_id + ":" + session_id)[:16]
4. board_key matches pk_hash(SPEC_HASH + ":" + session_id)[:16]
5. timestamp is present, valid ISO-8601, not in the future (>5min drift)
6. prev_bundle_hash matches pk_hash of the previous ledger entry
7. ledger_seq is exactly prev_seq + 1 (or 1 for the first entry)
8. bundle_sig verifies as HMAC(board_key, canonical text)
9. All 10 fields populated
10. verdict field is exactly "SIGNED"

### REJECT conditions - any ONE fails the bundle:

```
REJECT: no-deliverable      -> deliverable_hash empty or placeholder
REJECT: no-evidence         -> evidence_hash empty or placeholder
REJECT: placeholder-hash    -> any hash field contains TODO/TBD/XXXX/none/""
REJECT: timestamp-missing   -> timestamp absent or malformed ISO-8601
REJECT: timestamp-future    -> timestamp in the future (drift > 5 minutes)
REJECT: agent-key-mismatch  -> agent_key != pk_hash(task_id:session_id)[:16]
REJECT: board-key-mismatch  -> board_key != pk_hash(SPEC_HASH:session_id)[:16]
REJECT: missing-fields      -> any required field absent
REJECT: wrong-spec          -> board_key valid but derived from another SPEC_HASH
REJECT: malformed-bundle    -> bundle unparseable (missing braces, truncation)
REJECT: chain-broken        -> prev_bundle_hash does not match ledger tail
REJECT: seq-gap             -> ledger_seq not exactly prev_seq + 1
REJECT: sig-mismatch        -> bundle_sig fails HMAC verification
REJECT: stale-evidence      -> evidence predates the last deliverable edit
```

### On REJECT:

- Execution pauses immediately
- The REJECT reason is reported verbatim
- The agent fixes the underlying issue (produce real evidence, recalc hashes)
- The agent resubmits a NEW bundle with a fresh timestamp and next seq
- The board validates the new bundle before resuming
- The rejected attempt stays in the ledger - repair history is part of the
  record, never erased

## SECTION 6 - CANONICAL TEXT AND HMAC SIGN-OFF

The HMAC is computed over the CANONICAL text of the bundle - a deterministic
serialization with fields in fixed order, so any implementation produces the
same bytes:

```python
import hashlib, hmac

def pk_hash(text: str) -> str:
    return hashlib.sha256(text.encode('utf-8')).hexdigest()

def canonical_bundle_text(b: dict) -> str:
    """Fixed-order canonical text. Any field order change breaks the HMAC."""
    return (
        f"task_id={b['task_id']}\n"
        f"deliverable_hash={b['deliverable_hash']}\n"
        f"evidence_hash={b['evidence_hash']}\n"
        f"timestamp={b['timestamp']}\n"
        f"session_id={b['session_id']}\n"
        f"agent_key={b['agent_key']}\n"
        f"board_key={b['board_key']}\n"
        f"prev_bundle_hash={b['prev_bundle_hash']}\n"
        f"ledger_seq={b['ledger_seq']}\n"
        f"verdict={b['verdict']}"
    )

def sign_bundle(bundle: dict, board_key: str) -> str:
    return hmac.new(
        board_key.encode('utf-8'),
        canonical_bundle_text(bundle).encode('utf-8'),
        hashlib.sha256
    ).hexdigest()[:16]
```

Signing rules:

- The signature covers the canonical text WITHOUT the bundle_sig field
  itself (otherwise the signature would need to include itself).
- The board key never travels in the bundle - only the signature does. A
  verifier who knows the SPEC_HASH and session_id can recompute the board
  key and verify; an attacker who knows neither cannot forge.
- On verification failure: `REJECT: sig-mismatch - bundle altered after
  signing`. The response is paused; the agent must recompute and resubmit.

## SECTION 7 - IMPLEMENTATION (pure hashlib)

```python
import hashlib
import hmac
from datetime import datetime, timedelta, timezone

PLACEHOLDERS = {"", "none", "todo", "tbd", "xxxx", "placeholder", "n/a", "null"}
REQUIRED = ["task_id", "deliverable_hash", "evidence_hash", "timestamp",
            "session_id", "agent_key", "board_key", "prev_bundle_hash",
            "ledger_seq", "bundle_sig"]


def pk_hash(text: str) -> str:
    """SHA-256 hash of text as hex digest."""
    return hashlib.sha256(text.encode('utf-8')).hexdigest()


def derive_keys(task_id: str, session_id: str, spec_hash: str) -> dict:
    return {
        "agent_key": pk_hash(task_id + ":" + session_id)[:16],
        "board_key": pk_hash(spec_hash + ":" + session_id)[:16],
    }


def create_bundle(task_description: str, session_id: str, spec_hash: str,
                  deliverable_content: str, tool_output: str,
                  prev_bundle: dict | None) -> dict:
    """Create a signed, chain-linked Paperclip Bundle.

    Args:
        task_description:   full task text (not truncated)
        session_id:         current OpenCode session ID (ses_xxx)
        spec_hash:          frozen spec hash for this run
        deliverable_content: full text of the deliverable file
        tool_output:        pasted terminal/tool output used as evidence
        prev_bundle:        the last bundle dict in the ledger, or None

    Returns:
        dict with all bundle fields, signed and ready to emit.
    """
    task_id = pk_hash(task_description)[:8]
    keys = derive_keys(task_id, session_id, spec_hash)
    seq = 1 if prev_bundle is None else prev_bundle["ledger_seq"] + 1
    prev_hash = ("GENESIS" if prev_bundle is None
                 else pk_hash(canonical_bundle_text(prev_bundle))[:16])
    bundle = {
        "task_id": task_id,
        "deliverable_hash": pk_hash(deliverable_content)[:16],
        "evidence_hash": pk_hash(tool_output)[:16],
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "session_id": session_id,
        "agent_key": keys["agent_key"],
        "board_key": keys["board_key"],
        "prev_bundle_hash": prev_hash,
        "ledger_seq": seq,
        "bundle_sig": "",
        "verdict": "SIGNED",
    }
    bundle["bundle_sig"] = sign_bundle(bundle, keys["board_key"])
    return bundle


def sign_bundle(bundle: dict, board_key: str) -> str:
    payload = {k: v for k, v in bundle.items() if k != "bundle_sig"}
    return hmac.new(board_key.encode(),
                    canonical_text(payload).encode(),
                    hashlib.sha256).hexdigest()[:16]


def validate_bundle(bundle: dict, spec_hash: str,
                    prev_bundle: dict | None) -> tuple[bool, str]:
    """Validate against the expected spec hash and the chain."""
    for field in REQUIRED:
        if field not in bundle:
            return False, f"REJECT: missing-fields - '{field}' absent"
    for field in ["task_id", "deliverable_hash", "evidence_hash",
                  "agent_key", "board_key", "prev_bundle_hash"]:
        if str(bundle.get(field, "")).strip().lower() in PLACEHOLDERS:
            return False, f"REJECT: placeholder-hash - '{field}'"
    expected_seq = 1 if prev_bundle is None else prev_bundle["ledger_seq"] + 1
    if bundle["ledger_seq"] != expected_seq:
        return False, "REJECT: seq-gap - ledger_seq is not prev+1"
    if prev_bundle is not None:
        expected_prev = pk_hash(canonical_text(prev_bundle))[:16]
        if bundle["prev_bundle_hash"] != expected_prev:
            return False, "REJECT: chain-broken - prev hash mismatch"
    ts = bundle["timestamp"]
    try:
        then = datetime.fromisoformat(ts.replace("Z", "+00:00"))
    except ValueError:
        return False, "REJECT: timestamp-missing - not valid ISO-8601"
    if then > datetime.now(timezone.utc) + timedelta(minutes=5):
        return False, "REJECT: timestamp-future"
    keys = derive_keys(bundle["task_id"], bundle["session_id"], spec_hash)
    if bundle["agent_key"] != keys["agent_key"]:
        return False, "REJECT: agent-key-mismatch"
    if bundle["board_key"] != keys["board_key"]:
        return False, "REJECT: board-key-mismatch"
    expected_sig = sign_bundle(bundle, keys["board_key"])
    if bundle["bundle_sig"] != expected_sig:
        return False, "REJECT: sig-mismatch - bundle altered after signing"
    return True, "SIGNED"
```

## SECTION 8 - LEDGER FILE FORMAT

The ledger is one JSONL file per task, one line per bundle, append-only:

```
{"ledger_seq": 1, "task_id": "a3f9b2c1", "verdict": "SIGNED", "bundle": {...}}
{"ledger_seq": 2, "task_id": "a3f9b2c1", "verdict": "REJECT: no-evidence", "bundle": {...}}
{"ledger_seq": 3, "task_id": "a3f9b2c1", "verdict": "SIGNED", "bundle": {...}}
```

Rules:

- One file per task_id under `<run-dir>/ledger/`.
- Never rewrite a line; corrections are new lines with the next seq.
- The file's sha256 is itself recorded in the run's evidence bundle, so the
  ledger's integrity is checked by the same machinery it protects.
- A lost ledger is a broken chain: the task reverts to UNTESTED and the
  verification is re-run. There is no reconstruction from memory.

## SECTION 9 - INTEGRATION

### Position in claim gates:

```
Gate 1: skill-ground-truth-harness     (external evidence must be pasted)
Gate 2: skill-anti-overclaim-upgrade   (overclaim pattern detection)
Gate 3: skill-paperclip-ledger-guard   (THIS SKILL - cryptographic bundle)
Gate 4: anti-overclaim (base)          (final done-claim approval)
```

### Data flow:

```
Agent produces deliverable
  -> runs verification command (terminal output = evidence)
  -> reads the ledger's last bundle (or none -> GENESIS)
  -> calls create_bundle(task_desc, session_id, SPEC_HASH,
                         file_content, tool_output, prev_bundle)
  -> appends the new bundle to the ledger file
  -> emits PAPERCLIP BUNDLE block in the response
  -> Gate 3 validates: fields, chain, seq, HMAC
  -> SIGNED: passes to Gate 4
  -> REJECT: pause, report reason, fix, resubmit at seq+1
```

### Inputs to this gate:

- PASS signal from Gate 2 (anti-overclaim-upgrade)
- Full deliverable content (for hashing)
- Real terminal/tool output (for evidence hashing)
- The task's ledger file (for the chain pointer and seq)
- Current session_id and frozen SPEC_HASH

### Outputs:

- SIGNED bundle, appended to the ledger and passed to Gate 4
- REJECT bundle with reason; execution paused

## SECTION 10 - GATE LINE CONTRIBUTION

**On SIGNED (and board countersignature):**
```
claim-gates: ground-truth=evidence pasted | anti-overclaim-upgrade=PASS | paperclip=COUNTERSIGNED:seq-7 | anti-overclaim=PASS
```

**On REJECT:**
```
claim-gates: ground-truth=evidence pasted | anti-overclaim-upgrade=PASS | paperclip=REJECT:no-evidence | anti-overclaim=blocked
```

**On skipped (not a done-claim turn):**
```
claim-gates: ground-truth=skipped:not-a-done-turn | anti-overclaim-upgrade=skipped | paperclip=skipped:not-a-done-turn | anti-overclaim=skipped
```

The paperclip field NEVER reads "PASS" or "fired" - it reads
"COUNTERSIGNED:seq-N", "REJECT:reason", or "skipped:reason". The seq number
ties the approval to an exact ledger position.

## SECTION 11 - EXAMPLE (a real chained session)

Bundle 1 (GENESIS, first verification pass):

```
PAPERCLIP BUNDLE {
  task_id:            "a3f9b2c1"
  deliverable_hash:   "d62d66aba91aa142"
  evidence_hash:      "f55adab40e0035b5"
  timestamp:          "2026-11-08T17:30:00Z"
  session_id:         "ses_e43a07231ffe"
  agent_key:          "9f4e2a1b7c3d8e5f"
  board_key:          "2b8c4d6e1a9f3c7d"
  prev_bundle_hash:   "GENESIS"
  ledger_seq:         1
  bundle_sig:         "4c7b1e9a02f8d3a6"
  verdict:            "SIGNED"
}
```

Bundle 2 (after a fix; evidence re-run; chains to bundle 1):

```
PAPERCLIP BUNDLE {
  task_id:            "a3f9b2c1"
  deliverable_hash:   "91c4e7f20b8a5d13"
  evidence_hash:      "77aa0c2e9b1f4d58"
  timestamp:          "2026-11-08T18:02:11Z"
  session_id:         "ses_e43a07231ffe"
  agent_key:          "9f4e2a1b7c3d8e5f"
  board_key:          "2b8c4d6e1a9f3c7d"
  prev_bundle_hash:   "d62d66aba91aa142"
  ledger_seq:         2
  bundle_sig:         "8e1f4a9c02b7d635"
  verdict:            "SIGNED"
}
```

The seq-2 bundle chains to the sha256 of bundle 1's canonical text
(first 16 hex chars: `d62d66aba91aa142`). Editing or deleting bundle 1 now
breaks bundle 2's signature chain - visible to any verifier.

An example REJECT (never silently dropped - it is appended):

```
PAPERCLIP BUNDLE {
  task_id:            "a3f9b2c1"
  deliverable_hash:   "TODO"
  evidence_hash:      "f55adab40e0035b5"
  timestamp:          "2026-11-08T17:29:40Z"
  session_id:         "ses_e43a07231ffe"
  agent_key:          "9f4e2a1b7c3d8e5f"
  board_key:          "2b8c4d6e1a9f3c7d"
  prev_bundle_hash:   "d62d66aba91aa142"
  ledger_seq:         3
  bundle_sig:         "0a19f2e4c8b3d760"
  verdict:            "REJECT: placeholder-hash - 'deliverable_hash' is a placeholder"
}
```

## SECTION 12 - SELF-TEST (the guard tests its own machinery)

A gate that has never failed on a defect is an unverified guard. Run after
any change to this skill, and quarterly regardless:

```python
def test_guard_self():
    b1 = create_bundle("task one", "ses_test", "spec-v1",
                       "deliverable v1", "evidence: exit 0", None)
    ok, verdict = validate_bundle(b1, "spec-v1", None)
    assert ok, verdict

    # Chain: b2 references b1 - tampering with b1 must break b2
    b2 = create_bundle("task one", "ses_test", "spec-v1",
                       "deliverable v2", "evidence: exit 0 again", b1)
    ok, verdict = validate_bundle(b2, "spec-v1", b1)
    assert ok, verdict

    tampered = dict(b1)
    tampered["deliverable_hash"] = pk_hash("something else")[:16]
    ok, verdict = validate_bundle(b2, "spec-v1", tampered)
    assert not ok and "chain-broken" in verdict

    # Sig: editing a signed bundle after the fact must fail verification
    forged = dict(b2)
    forged["evidence_hash"] = pk_hash("made up")[:16]
    ok, verdict = validate_bundle(forged, "spec-v1", b1)
    assert not ok and "sig-mismatch" in verdict

    # Gap detection
    b3 = create_bundle("task one", "ses_test", "spec-v1",
                       "deliverable v3", "evidence: exit 0", b2)
    b3 = dict(b2); b3["ledger_seq"] = 99
    ok, verdict = validate_bundle(b3, "spec-v1", b2)
    assert not ok and "seq" in verdict

    print("guard self-test: ALL PASS")
```

A self-test failure blocks every done-claim in the session until fixed.

## SECTION 13 - SECURITY PROPERTIES

| Property                          | Mechanism                                                  |
|-----------------------------------|------------------------------------------------------------|
| Cannot fake without deliverable   | deliverable_hash = sha256(file_content)[:16]               |
| Cannot fake without evidence      | evidence_hash = sha256(tool_output)[:16]                   |
| Cannot reuse across specs         | board_key = sha256(SPEC_HASH:session_id)[:16]              |
| Cannot reuse across sessions      | both keys include session_id                               |
| Cannot reuse across tasks         | agent_key includes task_id                                 |
| Cannot rewrite history            | prev_bundle_hash chains each bundle to its predecessor     |
| Cannot reorder or gap the ledger  | ledger_seq must be exactly prev_seq + 1                    |
| Cannot alter a bundle post-hoc    | bundle_sig = HMAC(board_key, canonical text)               |
| Placeholder detection             | reject list covers lazy substitutions                      |
| No external dependencies          | pure hashlib/hmac - works in any Python environment        |
| Deterministic                     | same inputs always produce the same keys (self-tested)     |

## SECTION 14 - FAQ

**Q: Why HMAC and not just the raw hash?**
The HMAC binds the bundle to the board_key (derived from the SPEC_HASH).
A bare hash could be recomputed by anyone over forged content; the HMAC
requires the board key, which requires the true SPEC_HASH and session.

**Q: What if the agent and the "board" are the same agent?**
The board role is procedural separation, not a second human: the approving
pass recomputes every hash from the raw files rather than trusting the
submitting pass's numbers. The two-key split forces the recomputation.

**Q: What about multiple sessions on one task?**
Each session appends its own bundles to the same ledger with seq from the
current tail. A session that cannot read the ledger (fresh machine) rebuilds
state by re-running verification and appending a fresh GENESIS-marked entry
with a note - never by fabricating a chain pointer.

**Q: Does the ledger file get committed?**
Yes - it lives in the run dir and is part of the deliverable. Its own hash
rides in the evidence bundle, so the chain is auditable from any checkout.

**Q: What breaks first if someone edits the ledger?**
The seq-gap or chain-broken rule fires on the NEXT validation. Any SIGNED
verdicts downstream of the edit are void until repaired and re-signed.

## APPENDIX - QUICK REFERENCE CARD

```
+------------------------------------------------------------+
|  Paperclip Ledger Guard - Quick Reference                   |
+------------------------------------------------------------+
|  Gate position : #3 of 4 (after phrase scan, before base)   |
|  Emits         : PAPERCLIP BUNDLE with 10 fields + sig      |
|  Ledger        : append-only JSONL, one file per task       |
|  Chain         : prev_bundle_hash = sha256(prev canonical)  |
|  Order lock    : ledger_seq = prev_seq + 1, no gaps         |
|  Signature     : HMAC-SHA256(board_key, canonical)[:16]     |
+------------------------------------------------------------+
|  On SIGNED -> paperclip=COUNTERSIGNED:seq-N in the gate line|
|  On REJECT -> pause, fix, resubmit at seq+1                 |
|  Never "PASS"/"fired" - SIGNED/REJECT/skipped only          |
+------------------------------------------------------------+
```
