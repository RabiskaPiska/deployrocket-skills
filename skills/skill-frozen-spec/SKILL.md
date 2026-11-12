---
name: skill-frozen-spec
description: Hash-locked frozen spec with drift guard -- prevents scope creep by locking the spec hash and alerting on any drift
---
# SKILL: Hash-Locked Frozen Spec with Drift Guard

## Purpose

Before any planner, AI jury, or agent crew runs, the acceptance spec is frozen via
SHA-256 of its canonical serialization. If the spec file changes on disk without a
formal version bump, the drift guard exits 2 and blocks ALL downstream work.

This skill ALWAYS fires before any done claim and before any jury or automated
acceptance check invocation.

---

## Why This Exists

Specs drift silently. An agent edits a criterion mid-run to make a check pass. A
human tweaks wording without bumping the version. Two agents hold divergent copies
of the same criterion id. All of these corrupt the evidence chain without producing
any loud error. The frozen spec pattern converts silent drift into a hard exit-2
gate that is impossible to miss.

Rule: if you cannot prove the spec hash matches the frozen hash, the run is invalid.
Not "probably fine." Invalid. Full stop.

---

## Core Data Model

Every acceptance criterion is a YAML atom. No prose paragraphs. No ambiguous
sentences. Each atom is exactly one falsifiable statement with a machine-runnable
verification method.

Canonical criterion atom schema (all fields required unless marked optional):

```yaml
- id: "CRIT-001"
  statement: "The login endpoint returns HTTP 200 with a valid JWT when given correct credentials."
  kind: integration_test
  verification:
    method: "pytest tests/test_auth.py::test_login_success -v 2>&1 | tail -5"
    expected: "1 passed"
  evidence:
    artifact: "evidence/CRIT-001.txt"
    captured: null
  result: UNKNOWN
  blocking: true
```

Field definitions:

| Field                   | Type    | Allowed values                                                                 |
|-------------------------|---------|--------------------------------------------------------------------------------|
| id                      | string  | Unique slug, kebab-case or CRIT-NNN format                                     |
| statement               | string  | Exactly one falsifiable sentence. No "and". No "or".                           |
| kind                    | enum    | manual_observation, external_tool, unit_test, integration_test, performance,   |
|                         |         | security, visual                                                                |
| verification.method     | string  | Shell command. Must be runnable. Must produce deterministic output.             |
| verification.expected   | string  | Exact substring or full output to match against command stdout+stderr.          |
| evidence.artifact       | string  | Relative path where proof text is written (stdout capture).                     |
| evidence.captured       | string  | null until proven. Set by probe runner, never by agent self-report.             |
| result                  | enum    | UNKNOWN (default), PASS, FAIL. UNKNOWN = not yet run. Never self-report PASS.  |
| blocking                | bool    | true = must be PASS for done gate. false = informational only.                 |

NEVER set result: PASS by editing the YAML manually. The probe runner sets it.
NEVER set evidence.captured to anything other than actual stdout from the command.

---

## Five Criterion Atom Examples

### Example 1 -- Unit Test (kind: unit_test)

```yaml
- id: "CRIT-001"
  statement: "The spec hash function produces identical output for the same input regardless of key insertion order."
  kind: unit_test
  verification:
    method: "python -m pytest tests/test_spec_hash.py::test_hash_stability -v 2>&1 | tail -3"
    expected: "1 passed"
  evidence:
    artifact: "evidence/CRIT-001.txt"
    captured: null
  result: UNKNOWN
  blocking: true
```

### Example 2 -- Integration Test (kind: integration_test)

```yaml
- id: "CRIT-002"
  statement: "The freeze-check script exits 2 when the on-disk spec file is modified without bumping the version field."
  kind: integration_test
  verification:
    method: "bash tests/integration/test_drift_detection.sh 2>&1 | tail -4"
    expected: "DRIFT DETECTED: exit 2"
  evidence:
    artifact: "evidence/CRIT-002.txt"
    captured: null
  result: UNKNOWN
  blocking: true
```

### Example 3 -- External Tool (kind: external_tool)

```yaml
- id: "CRIT-003"
  statement: "The spec file passes yamllint strict validation with zero errors."
  kind: external_tool
  verification:
    method: "yamllint -d strict spec/acceptance.yaml 2>&1 | tail -2"
    expected: "0 errors"
  evidence:
    artifact: "evidence/CRIT-003.txt"
    captured: null
  result: UNKNOWN
  blocking: true
```

### Example 4 -- Performance (kind: performance)

```yaml
- id: "CRIT-004"
  statement: "The spec hash computation completes in under 500ms for a spec with 200 criteria."
  kind: performance
  verification:
    method: "python scripts/bench_hash.py --criteria-count 200 2>&1 | grep elapsed_ms"
    expected: "elapsed_ms: "
  evidence:
    artifact: "evidence/CRIT-004.txt"
    captured: null
  result: UNKNOWN
  blocking: false
```

### Example 5 -- Security (kind: security)

```yaml
- id: "CRIT-005"
  statement: "The spec envelope does not contain any plaintext secrets, tokens, or credentials."
  kind: security
  verification:
    method: "trufflehog filesystem spec/ --no-verification 2>&1 | grep -c 'Found verified' || echo 'found:0'"
    expected: "found:0"
  evidence:
    artifact: "evidence/CRIT-005.txt"
    captured: null
  result: UNKNOWN
  blocking: true
```

---

## Spec Envelope Format

The full spec file is a YAML document with two top-level keys:

```yaml
envelope:
  spec_id: "my-project-acceptance-v1.0.0"
  version: "1.0.0"
  frozen_at: "2026-11-05T12:00:00Z"
  frozen_by: "agent-orchestrator"
  hash_algorithm: "sha256"
  canonical_hash: "a3f8d2..."
  git_tag: "spec/my-project-acceptance-v1.0.0"

criteria:
  - id: "CRIT-001"
    ...
  - id: "CRIT-002"
    ...
```

The canonical_hash covers only the criteria list, not the envelope itself. This
allows the envelope to record metadata (frozen_at, frozen_by) without invalidating
the hash.

---

## Canonical Serialization Rules

The hash must be deterministic across machines and Python versions. Apply these rules
before hashing:

1. Sort all criterion dicts by their id field (lexicographic, ascending).
2. Within each criterion dict, sort keys lexicographically (recursively).
3. Set evidence.captured to null in every criterion before hashing (captured is
   runtime state, not spec state).
4. Set result to UNKNOWN in every criterion before hashing (same reason).
5. Serialize to JSON (not YAML) using json.dumps with sort_keys=True, indent=None,
   separators=(',', ':'), ensure_ascii=True.
6. Encode to UTF-8. Normalize line endings to LF.
7. Compute sha256 of the encoded bytes.

Python reference implementation:

```python
import hashlib
import json
import copy

def canonical_hash(criteria: list[dict]) -> str:
    """
    Compute the canonical SHA-256 hash of a criteria list.
    criteria: list of dicts loaded from YAML
    returns: hex string
    """
    # Deep copy so we do not mutate the caller's data
    c = copy.deepcopy(criteria)

    # Normalize runtime fields before hashing
    for atom in c:
        atom.setdefault("evidence", {})["captured"] = None
        atom["result"] = "UNKNOWN"

    # Sort atoms by id
    c.sort(key=lambda x: x["id"])

    # Recursively sort keys within each atom dict
    def sort_keys(obj):
        if isinstance(obj, dict):
            return {k: sort_keys(v) for k, v in sorted(obj.items())}
        if isinstance(obj, list):
            return [sort_keys(i) for i in obj]
        return obj

    normalized = sort_keys(c)

    blob = json.dumps(normalized, sort_keys=True, indent=None,
                      separators=(',', ':'), ensure_ascii=True)
    return hashlib.sha256(blob.encode("utf-8")).hexdigest()
```

---

## freeze-check.sh

Save as `scripts/freeze-check.sh`. Run before any downstream agent work.

```bash
#!/usr/bin/env bash
# freeze-check.sh
# Usage: bash scripts/freeze-check.sh <spec-file>
# Exit 0 = hash matches (spec is clean)
# Exit 1 = usage error or spec parse error
# Exit 2 = DRIFT DETECTED (hash mismatch -- halt all downstream work)

set -euo pipefail

SPEC_FILE="${1:-spec/acceptance.yaml}"

if [[ ! -f "$SPEC_FILE" ]]; then
  echo "ERROR: spec file not found: $SPEC_FILE" >&2
  exit 1
fi

# Check dependencies
for dep in python3 sha256sum; do
  if ! command -v "$dep" &>/dev/null; then
    echo "ERROR: required tool not found: $dep" >&2
    exit 1
  fi
done

# Extract stored hash from envelope
STORED_HASH=$(python3 - "$SPEC_FILE" <<'PYEOF'
import sys, yaml
with open(sys.argv[1]) as f:
    doc = yaml.safe_load(f)
envelope = doc.get("envelope", {})
h = envelope.get("canonical_hash", "")
if not h:
    print("ERROR: envelope.canonical_hash is missing or empty", file=sys.stderr)
    sys.exit(1)
print(h)
PYEOF
)

if [[ -z "$STORED_HASH" ]]; then
  echo "ERROR: could not extract stored hash from envelope." >&2
  exit 1
fi

# Recompute hash from current criteria list
COMPUTED_HASH=$(python3 - "$SPEC_FILE" <<'PYEOF'
import sys, yaml, hashlib, json, copy

def canonical_hash(criteria):
    c = copy.deepcopy(criteria)
    for atom in c:
        atom.setdefault("evidence", {})["captured"] = None
        atom["result"] = "UNKNOWN"
    c.sort(key=lambda x: x["id"])

    def sort_keys(obj):
        if isinstance(obj, dict):
            return {k: sort_keys(v) for k, v in sorted(obj.items())}
        if isinstance(obj, list):
            return [sort_keys(i) for i in obj]
        return obj

    normalized = sort_keys(c)
    blob = json.dumps(normalized, sort_keys=True, indent=None,
                      separators=(',', ':'), ensure_ascii=True)
    return hashlib.sha256(blob.encode("utf-8")).hexdigest()

with open(sys.argv[1]) as f:
    doc = yaml.safe_load(f)
criteria = doc.get("criteria", [])
print(canonical_hash(criteria))
PYEOF
)

if [[ -z "$COMPUTED_HASH" ]]; then
  echo "ERROR: could not compute hash from current criteria." >&2
  exit 1
fi

if [[ "$STORED_HASH" == "$COMPUTED_HASH" ]]; then
  echo "SPEC CLEAN: hash matches ($STORED_HASH)"
  echo "spec_file=$SPEC_FILE" >> "${SPEC_VERIFY_LOG:-/dev/null}"
  exit 0
else
  echo "DRIFT DETECTED: exit 2" >&2
  echo "  stored:   $STORED_HASH" >&2
  echo "  computed: $COMPUTED_HASH" >&2
  echo "" >&2
  echo "The spec has been modified without a formal version bump." >&2
  echo "Options:" >&2
  echo "  1. Revert spec to the frozen version:  git checkout spec/acceptance.yaml" >&2
  echo "  2. Bump the version and re-freeze:     python scripts/freeze_spec.py --bump" >&2
  echo "  3. Investigate the diff:               git diff spec/acceptance.yaml" >&2
  exit 2
fi
```

---

## freeze_spec.py -- Freeze Helper

Save as `scripts/freeze_spec.py`. Run after every intentional spec edit.

```python
#!/usr/bin/env python3
"""
freeze_spec.py
Recomputes the canonical hash and writes it into the spec envelope.
Optionally bumps the version (--bump) and creates a signed git tag.

Usage:
  python scripts/freeze_spec.py                     # re-freeze with same version
  python scripts/freeze_spec.py --bump              # bump patch version and re-freeze
  python scripts/freeze_spec.py --bump --minor      # bump minor version
  python scripts/freeze_spec.py --file path/to/spec.yaml
"""

import argparse
import copy
import hashlib
import json
import subprocess
import sys
from datetime import datetime, timezone
from pathlib import Path

import yaml


def canonical_hash(criteria: list) -> str:
    c = copy.deepcopy(criteria)
    for atom in c:
        atom.setdefault("evidence", {})["captured"] = None
        atom["result"] = "UNKNOWN"
    c.sort(key=lambda x: x["id"])

    def sort_keys(obj):
        if isinstance(obj, dict):
            return {k: sort_keys(v) for k, v in sorted(obj.items())}
        if isinstance(obj, list):
            return [sort_keys(i) for i in obj]
        return obj

    normalized = sort_keys(c)
    blob = json.dumps(normalized, sort_keys=True, indent=None,
                      separators=(",", ":"), ensure_ascii=True)
    return hashlib.sha256(blob.encode("utf-8")).hexdigest()


def bump_version(version: str, bump_minor: bool = False) -> str:
    parts = version.split(".")
    if len(parts) < 3:
        parts += ["0"] * (3 - len(parts))
    major, minor, patch = int(parts[0]), int(parts[1]), int(parts[2])
    if bump_minor:
        minor += 1
        patch = 0
    else:
        patch += 1
    return f"{major}.{minor}.{patch}"


def git_current_user() -> str:
    try:
        result = subprocess.run(
            ["git", "config", "user.name"],
            capture_output=True, text=True, check=True
        )
        return result.stdout.strip()
    except Exception:
        return "unknown"


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--file", default="spec/acceptance.yaml")
    parser.add_argument("--bump", action="store_true",
                        help="Bump patch version before freezing")
    parser.add_argument("--minor", action="store_true",
                        help="Bump minor version (requires --bump)")
    parser.add_argument("--tag", action="store_true",
                        help="Create a signed git tag after freezing")
    args = parser.parse_args()

    spec_path = Path(args.file)
    if not spec_path.exists():
        print(f"ERROR: spec file not found: {spec_path}", file=sys.stderr)
        sys.exit(1)

    with open(spec_path) as f:
        doc = yaml.safe_load(f)

    envelope = doc.get("envelope", {})
    criteria = doc.get("criteria", [])

    if not criteria:
        print("ERROR: spec has no criteria list.", file=sys.stderr)
        sys.exit(1)

    version = envelope.get("version", "1.0.0")
    if args.bump:
        version = bump_version(version, bump_minor=args.minor)
        print(f"Version bumped to: {version}")

    new_hash = canonical_hash(criteria)
    spec_id_base = envelope.get("spec_id", spec_path.stem)
    # rebuild spec_id with new version
    spec_id_parts = spec_id_base.rsplit("-v", 1)
    spec_id = f"{spec_id_parts[0]}-v{version}"

    envelope["spec_id"] = spec_id
    envelope["version"] = version
    envelope["frozen_at"] = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
    envelope["frozen_by"] = git_current_user()
    envelope["hash_algorithm"] = "sha256"
    envelope["canonical_hash"] = new_hash
    envelope["git_tag"] = f"spec/{spec_id}"

    doc["envelope"] = envelope

    with open(spec_path, "w") as f:
        yaml.dump(doc, f, default_flow_style=False, sort_keys=False, allow_unicode=False)

    print(f"Spec frozen: {spec_path}")
    print(f"  spec_id:        {spec_id}")
    print(f"  version:        {version}")
    print(f"  canonical_hash: {new_hash}")
    print(f"  git_tag:        spec/{spec_id}")

    if args.tag:
        tag = f"spec/{spec_id}"
        try:
            subprocess.run(["git", "add", str(spec_path)], check=True)
            subprocess.run(
                ["git", "commit", "-m", f"chore: freeze spec {spec_id}"],
                check=True
            )
            subprocess.run(
                ["git", "tag", "-s", tag, "-m", f"Frozen spec {spec_id}"],
                check=True
            )
            print(f"Git tag created: {tag}")
        except subprocess.CalledProcessError as e:
            print(f"WARNING: git operation failed: {e}", file=sys.stderr)
            print("Spec file was updated on disk but git tag was not created.")


if __name__ == "__main__":
    main()
```

---

## Done Gate Logic

The done gate is a strict boolean conjunction. ALL of the following must be true:

1. `freeze-check.sh` exits 0 (hash is clean, no drift).
2. For every criterion where blocking == true: result == PASS.
3. For every criterion where blocking == true: evidence.captured is not null.
4. The git tag for the frozen spec exists and points to the correct commit.
5. No criterion has result == UNKNOWN (every criterion was actually run).

```python
def done_gate(spec_path: str) -> tuple[bool, list[str]]:
    """
    Returns (is_done, list_of_failures).
    is_done is True only when ALL conditions pass.
    """
    import yaml, subprocess
    failures = []

    # 1. Drift check
    result = subprocess.run(
        ["bash", "scripts/freeze-check.sh", spec_path],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        failures.append(f"DRIFT: {result.stderr.strip()}")

    with open(spec_path) as f:
        doc = yaml.safe_load(f)
    criteria = doc.get("criteria", [])

    for atom in criteria:
        cid = atom.get("id", "?")

        # 5. No UNKNOWN results
        if atom.get("result") == "UNKNOWN":
            failures.append(f"{cid}: result is UNKNOWN (not yet run)")
            continue

        if not atom.get("blocking", False):
            continue

        # 2. All blocking must be PASS
        if atom.get("result") != "PASS":
            failures.append(f"{cid}: blocking criterion is {atom.get('result')}")

        # 3. evidence.captured must not be null
        captured = (atom.get("evidence") or {}).get("captured")
        if not captured:
            failures.append(f"{cid}: evidence.captured is null for blocking criterion")

    return (len(failures) == 0), failures
```

---

## Probe Runner

A probe runner executes each criterion's verification.method, captures stdout, and
writes the result. It never self-reports PASS -- the exit code and output match
determine the result.

```python
#!/usr/bin/env python3
"""
probe_runner.py
Runs all criteria in a spec file and writes evidence.

Usage:
  python scripts/probe_runner.py spec/acceptance.yaml
  python scripts/probe_runner.py spec/acceptance.yaml --id CRIT-001  # single criterion
"""

import argparse
import os
import subprocess
import sys
from pathlib import Path

import yaml


def run_criterion(atom: dict, evidence_dir: Path) -> dict:
    cid = atom["id"]
    method = atom["verification"]["method"]
    expected = atom["verification"]["expected"]
    artifact = evidence_dir / atom["evidence"]["artifact"].lstrip("/")

    artifact.parent.mkdir(parents=True, exist_ok=True)

    try:
        proc = subprocess.run(
            method,
            shell=True,
            capture_output=True,
            text=True,
            timeout=120
        )
        output = proc.stdout + proc.stderr
    except subprocess.TimeoutExpired:
        output = "TIMEOUT: verification method exceeded 120s"

    # Write evidence artifact
    with open(artifact, "w") as f:
        f.write(f"CRITERION: {cid}\n")
        f.write(f"METHOD: {method}\n")
        f.write(f"EXPECTED: {expected}\n")
        f.write(f"OUTPUT:\n{output}\n")

    # Determine PASS or FAIL
    passed = expected.strip() in output
    atom["evidence"]["captured"] = output.strip()[:500]  # cap at 500 chars
    atom["result"] = "PASS" if passed else "FAIL"

    status = "PASS" if passed else "FAIL"
    print(f"  [{status}] {cid}: {atom['statement'][:70]}")
    if not passed:
        print(f"         expected: {expected!r}")
        print(f"         got:      {output.strip()[:120]!r}")

    return atom


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("spec_file")
    parser.add_argument("--id", help="Run only this criterion id")
    parser.add_argument("--evidence-dir", default=".")
    args = parser.parse_args()

    spec_path = Path(args.spec_file)
    evidence_dir = Path(args.evidence_dir)

    with open(spec_path) as f:
        doc = yaml.safe_load(f)

    # Drift check first
    result = subprocess.run(
        ["bash", "scripts/freeze-check.sh", str(spec_path)],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        print("ABORT: spec drift detected. Fix spec before running probes.", file=sys.stderr)
        sys.exit(2)

    criteria = doc.get("criteria", [])
    print(f"Running {len(criteria)} criteria...")
    updated = []
    for atom in criteria:
        if args.id and atom["id"] != args.id:
            updated.append(atom)
            continue
        updated.append(run_criterion(atom, evidence_dir))

    doc["criteria"] = updated
    with open(spec_path, "w") as f:
        yaml.dump(doc, f, default_flow_style=False, sort_keys=False, allow_unicode=False)

    failures = [a for a in updated if a.get("blocking") and a.get("result") != "PASS"]
    print(f"\nProbe run complete. Blocking failures: {len(failures)}")
    if failures:
        for a in failures:
            print(f"  FAIL: {a['id']}: {a['statement']}")
        sys.exit(1)
    sys.exit(0)


if __name__ == "__main__":
    main()
```

---

## Adoption Guide (15 Steps)

### Step 1 -- Install dependencies

```bash
pip install pyyaml yamllint
# For trufflehog (security criterion):
# brew install trufflehog  or  pip install trufflehog
```

### Step 2 -- Create spec directory

```bash
mkdir -p spec evidence scripts
```

### Step 3 -- Create initial spec file

```bash
cat > spec/acceptance.yaml << 'EOF'
envelope:
  spec_id: "my-project-v1.0.0"
  version: "1.0.0"
  frozen_at: ""
  frozen_by: ""
  hash_algorithm: "sha256"
  canonical_hash: ""
  git_tag: ""

criteria:
  - id: "CRIT-001"
    statement: "Replace this with your first falsifiable acceptance statement."
    kind: unit_test
    verification:
      method: "echo 'placeholder' | grep placeholder"
      expected: "placeholder"
    evidence:
      artifact: "evidence/CRIT-001.txt"
      captured: null
    result: UNKNOWN
    blocking: true
EOF
```

### Step 4 -- Write your real criteria

Replace the placeholder CRIT-001 with real criteria. Keep each statement to exactly
one falsifiable sentence. No "and". No "or". Split compound statements.

### Step 5 -- Copy the scripts

Copy `freeze-check.sh`, `freeze_spec.py`, and `probe_runner.py` from this skill
into your `scripts/` directory. Make them executable:

```bash
chmod +x scripts/freeze-check.sh
```

### Step 6 -- Initial freeze

```bash
python scripts/freeze_spec.py
```

This writes the canonical_hash into the envelope. Inspect the file to confirm.

### Step 7 -- Verify the freeze

```bash
bash scripts/freeze-check.sh spec/acceptance.yaml
# Expected: SPEC CLEAN: hash matches (...)
```

### Step 8 -- Commit and tag

```bash
git add spec/acceptance.yaml scripts/
git commit -m "feat: initial frozen spec v1.0.0"
git tag -s "spec/my-project-v1.0.0" -m "Frozen spec v1.0.0"
git push origin main --tags
```

### Step 9 -- Wire into CI

Add this as the first step in your CI pipeline before any test run:

```yaml
# .github/workflows/ci.yml
- name: Check spec integrity
  run: |
    bash scripts/freeze-check.sh spec/acceptance.yaml
    if [ $? -eq 2 ]; then
      echo "SPEC DRIFT DETECTED -- aborting CI"
      exit 2
    fi
```

### Step 10 -- Wire into agent orchestrator

Before any agent crew runs, call:

```python
import subprocess, sys
result = subprocess.run(
    ["bash", "scripts/freeze-check.sh", "spec/acceptance.yaml"],
    capture_output=True, text=True
)
if result.returncode != 0:
    print("Spec drift -- agent run blocked.", file=sys.stderr)
    sys.exit(2)
```

### Step 11 -- Run probes after implementation

```bash
python scripts/probe_runner.py spec/acceptance.yaml
```

Review any FAIL outputs. Fix the implementation, not the criterion.

### Step 12 -- Amend a criterion (formal process)

If a criterion genuinely needs to change:
1. Edit the criterion in `spec/acceptance.yaml`.
2. Bump version: `python scripts/freeze_spec.py --bump`
3. Commit and re-tag.
4. Never edit a criterion while probes are in-flight.

### Step 13 -- Run done gate

```python
from scripts.done_gate import done_gate
ok, failures = done_gate("spec/acceptance.yaml")
if not ok:
    for f in failures:
        print(f"GATE FAIL: {f}")
    sys.exit(1)
print("DONE GATE PASSED")
```

### Step 14 -- Archive evidence

After a successful done gate, archive the evidence directory:

```bash
tar -czf "evidence-$(date +%Y%m%d-%H%M%S).tar.gz" evidence/
```

### Step 15 -- Post-mortem update

If a criterion was wrong (the statement or expected output did not match reality),
document why in a `spec/SPEC-HISTORY.md` entry before bumping and re-freezing.
Never silently delete a failing criterion to make the gate pass.

---

## Troubleshooting (10 Scenarios)

### Scenario 1 -- freeze-check exits 2 but I did not change the spec

Cause: A tool reformatted the YAML (auto-formatter, editor plugin, yaml.dump with
different settings). Even whitespace or key ordering changes the hash.
Fix: `git diff spec/acceptance.yaml` to see the diff. If it is only whitespace,
revert with `git checkout spec/acceptance.yaml` and configure your editor to not
auto-format YAML. Then re-freeze with `freeze_spec.py`.

### Scenario 2 -- canonical_hash is empty after running freeze_spec.py

Cause: The criteria list is empty or the yaml.safe_load returned None.
Fix: Confirm your spec has a `criteria:` key with at least one item. Run
`python -c "import yaml; print(yaml.safe_load(open('spec/acceptance.yaml')))"`.

### Scenario 3 -- Two agents produce different hashes for the same spec

Cause: Different Python versions serialize floats or None differently.
Fix: The canonical_hash function uses json.dumps with explicit separators and
ensure_ascii=True. Ensure all agents use the same Python version. Add a
`python --version` check to freeze-check.sh.

### Scenario 4 -- probe_runner sets evidence.captured but result stays UNKNOWN

Cause: The probe runner was interrupted mid-write and the YAML was not flushed.
Fix: Check if `spec/acceptance.yaml` is a partial file (open it and inspect the
tail). If truncated, `git checkout spec/acceptance.yaml` and re-run the probe.

### Scenario 5 -- freeze-check.sh hangs

Cause: The python3 heredoc inside the shell script is waiting for input (some
shell environments behave differently with here-documents).
Fix: Extract the Python snippets to standalone `.py` files and call them directly.
This also makes the script more debuggable.

### Scenario 6 -- evidence.captured contains only 500 chars but the output is longer

This is by design (the probe runner caps at 500 chars to prevent huge YAML files).
The full output is in the evidence artifact file. Set the cap higher if needed:
edit the `[:500]` slice in `probe_runner.py`.

### Scenario 7 -- CI passes freeze-check but local runs fail

Cause: The spec file on the branch differs from what was tagged. CI likely checks
out the branch tip, which may have a newer (unfrozen) edit.
Fix: Always run `freeze_spec.py` and commit before pushing a spec change. Add a
pre-push git hook that runs freeze-check.sh.

### Scenario 8 -- done_gate returns False but all criteria show PASS in the YAML

Cause: The spec was modified after the last probe run, invalidating the hash.
freeze-check detects this and returns non-zero even though the criteria values look
correct.
Fix: Re-freeze with the current criteria (no --bump needed if no logical change),
then re-run probes to regenerate evidence.captured from the clean spec.

### Scenario 9 -- A non-blocking criterion fails and the done gate still passes

This is correct behavior. Non-blocking criteria are informational. The gate only
hard-requires blocking == true criteria. Document non-blocking failures in a
`spec/WARNINGS.md` for visibility.

### Scenario 10 -- The git tag command fails with "not a git repository"

Cause: The script is running from a directory that is not inside a git repo (e.g.,
a temp directory or a detached checkout).
Fix: Run from the repo root. Or skip the --tag flag and create the tag manually:
`git tag -s "spec/..." -m "..."` from the correct directory.

---

## FAQ (8 Questions)

### Q1: Why SHA-256 and not MD5 or CRC32?

SHA-256 is collision-resistant and is the standard for content addressing (git,
docker, sigstore all use it). MD5 and CRC32 have known collision attacks. For a
spec where integrity is security-relevant, SHA-256 is non-negotiable.

### Q2: Can I use the hash to prove the spec was not tampered with by a third party?

Only if you combine it with a signed git tag (using a GPG key or SSH signing key).
The hash alone proves the content is consistent with the envelope. The signature
proves a specific identity created that hash. Use `git tag -s` for GPG-signed tags
or configure `gpg.format=ssh` in git config for SSH-signed tags.

### Q3: What if I need to change a criterion during an active sprint?

Formally: bump version, re-freeze, re-run all probes from scratch. Any partial
evidence from before the version bump is invalidated. If that is too expensive,
structure your spec into phases: freeze only the phase you are actively working and
keep future phases as DRAFT (outside the frozen criteria list).

### Q4: Why is PASS never self-reported?

Because self-report is the main failure mode. An agent that evaluates its own output
against criteria it wrote is a closed loop with no external reality check. The probe
runner runs the actual shell command and compares real stdout to expected. That is an
external check. An agent writing `result: PASS` in YAML is not.

### Q5: Can evidence.captured be a file hash instead of stdout?

Yes. Change the verification.method to output a hash:
  `sha256sum output/report.pdf | cut -d' ' -f1`
And set verification.expected to the known good hash. This works well for binary
artifacts where you cannot embed stdout.

### Q6: How do I handle a criterion whose command is flaky?

Do not use a flaky command as a verification method. If the underlying system is
non-deterministic, make it deterministic: use a fixed seed, mock the random source,
or snapshot the external dependency. A flaky criterion is worse than no criterion
because it erodes trust in the gate. Mark it blocking: false until it is reliable.

### Q7: Does this skill work with Windows (PowerShell)?

The bash scripts need WSL or Git Bash on Windows. The Python scripts are
platform-neutral. For a Windows-first setup, port `freeze-check.sh` to a PowerShell
script using `python scripts/freeze_check.py --check-only` and read the exit code.

### Q8: What is the relationship between this skill and honestscore?

honestscore provides the rubric for scoring a deliverable. This skill provides the
machine-checkable ground truth that honestscore draws from. honestscore should import
the done_gate result as one of its inputs: a deliverable that fails the done gate
cannot score above a 5.0 on the honestscore rubric, regardless of qualitative
assessment. Wire them together: run probe_runner first, then done_gate, then
honestscore.

---

## Quick Reference

Freeze: `python scripts/freeze_spec.py`
Check: `bash scripts/freeze-check.sh spec/acceptance.yaml`
Run probes: `python scripts/probe_runner.py spec/acceptance.yaml`
Done gate: `python -c "from scripts.done_gate import done_gate; ok,f=done_gate('spec/acceptance.yaml'); print('PASS' if ok else f)"`
Bump version: `python scripts/freeze_spec.py --bump`

Always fire: YES. Check before any done claim. Check before any jury invocation.
Gate logic: ALL blocking criteria PASS + all evidence.captured non-null + hash clean.

# ===========================================================================
# ALWAYS-FIRE RULE
# ===========================================================================
# This skill fires BEFORE any task execution begins. Always. No exceptions.
# Hash-locks what we are building so scope never drifts mid-execution.
# If you are about to start work on any task, frozen-spec activates first.
# A task without a frozen spec is a task without a definition of done.
# ===========================================================================


# SECTION: SPEC VERSIONING GUIDE
# ===========================================================================
#
# A frozen spec is immutable once hashed. To change it, supersede it.
# Never edit a frozen spec file in place. Always create a new version.
#
# VERSION FORMAT: MAJOR.MINOR.PATCH
# -----------------------------------
# MAJOR version bump (e.g. 1.0.0 -> 2.0.0):
#   - A BLOCKING CRITERION is added (new thing that must be true to call done)
#   - A BLOCKING CRITERION is removed (something previously required is now N/A)
#   - A BLOCKING CRITERION is TIGHTENED (e.g. ">= 90% test coverage" -> ">= 95%")
#   - The scope of work fundamentally changes
#   - The definition of success changes
#   ALL MAJOR bumps require a new spec hash. Agents with the old hash must stop.
#
# MINOR version bump (e.g. 1.0.0 -> 1.1.0):
#   - A BLOCKING CRITERION is LOOSENED (e.g. ">= 95% coverage" -> ">= 90%")
#   - A previously required output file is made optional
#   - A new OPTIONAL (non-blocking) guideline is added
#   ALL MINOR bumps require a new spec hash. Agents see this as a relaxation;
#   they may continue BUT must re-read the spec to confirm compatibility.
#
# PATCH version bump (e.g. 1.0.0 -> 1.0.1):
#   - Rewording only: same semantics, clearer language
#   - Fixing a typo that did not change meaning
#   - Adding an example or clarifying note (not a new criterion)
#   - Reformatting (whitespace, section order)
#   PATCH bumps still require a new spec hash (the file changed).
#   Agents may continue; the patch must not change any blocking criterion.
#
# SUPERSESSION PROCESS
# ---------------------
# 1. Copy the existing spec to spec-v{MAJOR}.{MINOR}.{PATCH}.md (old version)
# 2. Create new spec file with updated content
# 3. Compute new hash: sha256sum SPEC.md
# 4. Record old hash in SPEC.md under "supersedes: {old_hash}"
# 5. Announce new hash to all agents (write to /srv/.spec-hash-current)
# 6. Agents reading old hash: halt and await new spec
# 7. Never delete old spec versions; they are the audit trail
#
# HASH COMPUTATION (canonical)
# -----------------------------
#   # On Linux/Mac
#   SPEC_HASH=$(sha256sum SPEC.md | awk '{print $1}')
#
#   # On Python
#   import hashlib
#   content = open("SPEC.md", "rb").read()
#   spec_hash = hashlib.sha256(content).hexdigest()
#
#   # Store in environment for agents
#   export FROZEN_SPEC_HASH=$SPEC_HASH
#
# WHAT TO DO WHEN YOU WANT TO CHANGE A FROZEN SPEC
# --------------------------------------------------
# 1. STOP. Do not edit the file. Do not git commit any edits.
# 2. Ask: is this MAJOR, MINOR, or PATCH?
# 3. Write a new file: SPEC-v{new_version}.md with the changes
# 4. Hash the new file; announce to fleet
# 5. Archive old spec; link to new one
# 6. Agents pick up new hash on their next spec-check cycle
#
# ===========================================================================


# SECTION: MULTI-AGENT SPEC SHARING
# ===========================================================================
#
# A frozen spec is a single file, read by multiple agents concurrently.
# The spec itself never changes during a fleet run. Agents share it safely
# because it is read-only after freezing. The hash is the authority.
#
# DRIFT GUARD: PER-AGENT, PER-INVOCATION
# ----------------------------------------
# Before EVERY agent invocation, the agent reads and verifies the spec hash.
# This is not optional. It is the first line of agent boot code.
#
#   import hashlib, os, sys
#
#   EXPECTED_HASH = os.environ["FROZEN_SPEC_HASH"]  # set by orchestrator
#
#   def verify_spec(spec_path: str) -> None:
#       content = open(spec_path, "rb").read()
#       actual_hash = hashlib.sha256(content).hexdigest()
#       if actual_hash != EXPECTED_HASH:
#           print(
#               f"SPEC DRIFT DETECTED\n"
#               f"  Expected: {EXPECTED_HASH}\n"
#               f"  Actual:   {actual_hash}\n"
#               f"  File:     {spec_path}\n"
#               f"Agent refuses to run. Escalating to orchestrator."
#           )
#           sys.exit(99)  # exit code 99 = spec mismatch
#       print(f"Spec verified OK: {actual_hash[:16]}...")
#
#   verify_spec("/srv/frozen-spec/SPEC.md")
#
# WHAT HAPPENS ON HASH MISMATCH
# --------------------------------
# 1. Agent exits with code 99 (never 0, never 1 -- 99 is unambiguous)
# 2. Agent logs: timestamp, its agent_id, expected hash, actual hash, spec path
# 3. Orchestrator catches exit code 99
# 4. Orchestrator checks if spec was deliberately updated (new hash announced):
#    - If YES: restart agent with new FROZEN_SPEC_HASH
#    - If NO: THIS IS A SECURITY/INTEGRITY EVENT. Halt the fleet. Alert human.
# 5. Never silently continue after a hash mismatch
#
# FLEET-WIDE HASH CONSISTENCY CHECK
# ------------------------------------
# Run this before starting any multi-agent fleet run:
#
#   #!/bin/bash
#   # check_spec_hash.sh
#   SPEC=/srv/frozen-spec/SPEC.md
#   EXPECTED=$(cat /srv/.spec-hash-current)
#   ACTUAL=$(sha256sum "$SPEC" | awk '{print $1}')
#
#   if [ "$ACTUAL" != "$EXPECTED" ]; then
#     echo "SPEC HASH MISMATCH:"
#     echo "  Expected: $EXPECTED"
#     echo "  Actual:   $ACTUAL"
#     echo "Fleet launch ABORTED."
#     exit 1
#   fi
#   echo "Spec hash OK: $ACTUAL"
#
#   # Also verify all agents have the same expected hash in their env
#   for env_file in /srv/worktrees/wt-*/.worktree-env; do
#     agent_hash=$(grep FROZEN_SPEC_HASH "$env_file" | cut -d= -f2)
#     if [ "$agent_hash" != "$EXPECTED" ]; then
#       echo "AGENT ENV MISMATCH: $env_file has $agent_hash"
#       exit 1
#     fi
#   done
#   echo "All agent envs consistent. Fleet OK to launch."
#
# DISTRIBUTING THE HASH TO AGENTS
# ---------------------------------
# Option A (env file per worktree):
#   echo "FROZEN_SPEC_HASH=$SPEC_HASH" >> /srv/worktrees/wt-{id}/.worktree-env
#
# Option B (shared file, read at boot):
#   SPEC_HASH=$(cat /srv/.spec-hash-current)
#
# Option C (orchestrator ENV injection):
#   subprocess.Popen(["python3", "agent.py"],
#     env={**os.environ, "FROZEN_SPEC_HASH": spec_hash})
#
# Prefer Option C for new code; Option A for compatibility with existing fleet.
#
# ===========================================================================


# SECTION: INTEGRATION WITH skill-ai-jury
# ===========================================================================
#
# skill-ai-jury convenes a panel of AI judges to evaluate architectural
# decisions. It requires a frozen spec to operate correctly.
#
# WHY THE JURY NEEDS A FROZEN SPEC
# ----------------------------------
# - The jury evaluates proposals against FIXED criteria
# - If criteria drift mid-deliberation, the verdict is invalid
# - A frozen spec is the jury's constitution: immutable, hashable, auditable
# - Proposer and all jurors must read the SAME spec version
#
# PRE-JURY CHECKLIST (enforced by frozen-spec)
# ---------------------------------------------
# Before convening a jury:
#   1. Frozen spec MUST exist at: /srv/frozen-spec/SPEC.md
#   2. Spec hash MUST be recorded: /srv/.spec-hash-current
#   3. All jury members receive spec hash as first argument
#   4. Proposer prompt includes spec hash on line 1 (see format below)
#
# PROPOSER PROMPT FORMAT (line 1 must be spec hash)
# ---------------------------------------------------
#   SPEC_HASH: {sha256_of_spec}
#   SPEC_VERSION: {MAJOR.MINOR.PATCH}
#   SPEC_PATH: /srv/frozen-spec/SPEC.md
#
#   PROPOSAL: {title}
#   ...rest of proposal...
#
# If line 1 of the proposer prompt is NOT "SPEC_HASH: {valid_hash}",
# the jury is immediately disbanded and the proposal is rejected with reason:
# "No frozen spec hash. Proposal cannot be evaluated without fixed criteria."
#
# SPEC DRIFT DURING JURY
# -----------------------
# If the spec changes AFTER a jury is convened but BEFORE it reaches verdict:
#   1. Any juror detects the hash mismatch (they re-verify on each vote)
#   2. That juror exits with code 99 and logs: "Spec changed mid-deliberation"
#   3. Verdict is INVALIDATED regardless of vote count so far
#   4. Jury must be RE-CONVENED with the new spec hash
#   5. All proposals must be re-evaluated against the new spec
#   6. The old verdict is archived as "INVALIDATED - spec drift at {timestamp}"
#
# JURY VERDICT FORMAT (must include spec hash)
# ---------------------------------------------
# A valid jury verdict is a JSON file containing:
#   {
#     "verdict": "APPROVED" | "REJECTED" | "DEFERRED",
#     "spec_hash": "{sha256_at_time_of_verdict}",
#     "spec_version": "{MAJOR.MINOR.PATCH}",
#     "timestamp": "{ISO-8601}",
#     "proposal_id": "{uuid}",
#     "votes": {"approve": N, "reject": M, "abstain": K},
#     "rationale": "{summary}",
#     "valid": true
#   }
# A verdict without spec_hash is not a valid verdict. Discard it.
#
# CHECKING VERDICT AGAINST CURRENT SPEC
# ----------------------------------------
#   def is_verdict_still_valid(verdict: dict, current_spec_hash: str) -> bool:
#       if verdict["spec_hash"] != current_spec_hash:
#           print(f"Verdict from spec {verdict['spec_hash'][:16]} "
#                 f"does not match current spec {current_spec_hash[:16]}. "
#                 f"Verdict INVALID.")
#           return False
#       return verdict.get("valid", False)
#
# ===========================================================================


# SECTION: SPEC TEMPLATES FOR COMMON TASK TYPES
# ===========================================================================
#
# Copy-paste these templates when starting a new task.
# Fill in every {PLACEHOLDER} before hashing the spec.
# Never begin a task without a completed spec.
#
# TEMPLATE 1: SKILL UPGRADE TASK
# --------------------------------
# ---
# spec_type: skill-upgrade
# version: 1.0.0
# created: {ISO-8601-DATE}
# task_id: {UUID}
#
# GOAL
# ----
# Upgrade skill file at: {SKILL_PATH}
# Target line count: {MIN_LINES}+
# Content to add: {SECTION_NAMES}
#
# BLOCKING CRITERIA (all must be true to call DONE)
# --------------------------------------------------
# 1. File exists at exact path: {SKILL_PATH}
# 2. Line count >= {MIN_LINES} (verify with: wc -l {SKILL_PATH})
# 3. All target sections present (grep for section headers)
# 4. File is valid UTF-8 / ASCII (no binary garbage)
# 5. File contains no secrets, API keys, or passwords
# 6. Line count verified on the REMOTE host (not assumed from write)
#
# OUT OF SCOPE
# ------------
# - Do not modify other skill files not listed above
# - Do not change skill directory structure
# - Do not upgrade catalog; that runs separately
#
# END-TO-END CHECK
# -----------------
# ssh root@{HOST} "wc -l {SKILL_PATH} && head -5 {SKILL_PATH}"
# Expected: first number >= {MIN_LINES}
# ---
#
# TEMPLATE 2: VPS DEPLOY TASK
# -----------------------------
# ---
# spec_type: vps-deploy
# version: 1.0.0
# created: {ISO-8601-DATE}
# task_id: {UUID}
#
# GOAL
# ----
# Deploy {SERVICE_NAME} to VPS at {VPS_HOST}
# Service path: {SERVICE_PATH}
# Expected URL: {EXPECTED_URL}
#
# BLOCKING CRITERIA
# ------------------
# 1. Code is on the VPS at: {SERVICE_PATH}
# 2. Service is running: systemctl is-active {SERVICE_NAME}
# 3. HTTP check passes: curl -sf {EXPECTED_URL} -> HTTP 200
# 4. Logs show no ERROR in first 60s: journalctl -u {SERVICE_NAME} -n 50
# 5. Previous version is backed up at: {BACKUP_PATH}
#
# OUT OF SCOPE
# ------------
# - Do not modify firewall rules beyond what service requires
# - Do not change other running services
# - Do not update OS packages
#
# END-TO-END CHECK
# -----------------
# curl -sf {EXPECTED_URL} | head -20
# systemctl status {SERVICE_NAME}
# ---
#
# TEMPLATE 3: WEBSITE BUILD TASK
# --------------------------------
# ---
# spec_type: website-build
# version: 1.0.0
# created: {ISO-8601-DATE}
# task_id: {UUID}
#
# GOAL
# ----
# Build website for client: {CLIENT_NAME}
# Niche: {NICHE}
# Output path: {OUTPUT_PATH}
# Live URL: {LIVE_URL}
#
# BLOCKING CRITERIA
# ------------------
# 1. All pages exist: {LIST_OF_PAGES}
# 2. Mobile responsive (test at 375px, 768px, 1440px viewports)
# 3. No broken links (run: linkchecker {LIVE_URL})
# 4. Page load < 3s on 4G sim (Lighthouse performance >= 80)
# 5. Contact form submits successfully (manual test)
# 6. No console errors in production build
# 7. Deployed and reachable at: {LIVE_URL}
#
# OUT OF SCOPE
# ------------
# - SEO optimization beyond basic meta tags
# - Backend / CMS integration (static site only)
# - E-commerce functionality
#
# END-TO-END CHECK
# -----------------
# lighthouse {LIVE_URL} --output json | jq .categories.performance.score
# curl -sf {LIVE_URL} | grep -c "<title>"
# ---
#
# TEMPLATE 4: RESEARCH REPORT TASK
# ----------------------------------
# ---
# spec_type: research-report
# version: 1.0.0
# created: {ISO-8601-DATE}
# task_id: {UUID}
#
# GOAL
# ----
# Research topic: {TOPIC}
# Output file: {OUTPUT_PATH}
# Minimum length: {MIN_WORDS} words
#
# BLOCKING CRITERIA
# ------------------
# 1. Report exists at: {OUTPUT_PATH}
# 2. Word count >= {MIN_WORDS} (wc -w {OUTPUT_PATH})
# 3. At least {MIN_SOURCES} distinct sources cited with URLs
# 4. All URLs verified live (HTTP 200) at time of writing
# 5. Each major claim has an inline citation [SOURCE_N]
# 6. Report covers all required subsections: {SUBSECTION_LIST}
# 7. No fabricated statistics (all numbers from cited sources)
#
# OUT OF SCOPE
# ------------
# - Implementation; this is research only
# - Recommendations beyond what sources support
#
# END-TO-END CHECK
# -----------------
# wc -w {OUTPUT_PATH}
# grep -c "http" {OUTPUT_PATH}
# ---
#
# TEMPLATE 5: CODE DEBUGGING TASK
# ---------------------------------
# ---
# spec_type: code-debug
# version: 1.0.0
# created: {ISO-8601-DATE}
# task_id: {UUID}
#
# GOAL
# ----
# Fix bug in: {FILE_PATH}
# Bug description: {BUG_DESCRIPTION}
# Reproduction command: {REPRO_COMMAND}
#
# BLOCKING CRITERIA
# ------------------
# 1. Reproduction command no longer produces the bug
# 2. All existing tests pass: {TEST_COMMAND}
# 3. No new tests were deleted or weakened
# 4. The fix is minimal: only lines necessary to fix the bug are changed
# 5. Root cause is documented in: {RUNBOOK_PATH}
# 6. If a regression is possible, a new test covers the fixed case
#
# OUT OF SCOPE
# ------------
# - Refactoring unrelated code
# - Performance improvements not related to the bug
# - Adding new features
#
# END-TO-END CHECK
# -----------------
# {REPRO_COMMAND}  # must exit 0 and produce correct output
# {TEST_COMMAND}   # must all pass
# ---
#
# ===========================================================================
# HOW TO USE THESE TEMPLATES
# ===========================================================================
# 1. Pick the template matching your task type
# 2. Copy it to: /srv/frozen-spec/SPEC-{task_id}.md
# 3. Fill in every {PLACEHOLDER}
# 4. Hash it: sha256sum SPEC-{task_id}.md | awk '{print $1}'
# 5. Write hash to: /srv/.spec-hash-current
# 6. Write hash + spec path to agent env: FROZEN_SPEC_HASH=... SPEC_PATH=...
# 7. Begin work. Never modify the spec after hashing.
# ===========================================================================

---

## WORKED EXAMPLES -- 5 Full Spec + Drift Guard Runs

### WORKED EXAMPLE 1: Feature Spec -- Contact Form

TASK: Add contact form with email notification and Supabase storage.

FROZEN SPEC FILE: /srv/frozen-spec/SPEC-contact-form-v1.0.md

---
# task_id: contact-form-001
# version: 1.0
# frozen_at: 2026-11-06T14:23:00Z
# frozen_hash: (computed after writing, before any work starts)
#
# GOAL
# ----
# Add a contact form to the site that stores submissions in Supabase
# and sends an email notification via Resend API.
#
# ACCEPTANCE CRITERIA
# -------------------

- id: "CRIT-001"
  statement: "POST /api/contact with valid fields returns HTTP 200 and stores a row in the contacts table."
  kind: integration_test
  verification:
    method: "pytest tests/test_contact.py::test_valid_submission -v 2>&1 | tail -5"
    expected: "1 passed"
  evidence:
    artifact: "evidence/CRIT-001.txt"
  blocking: true

- id: "CRIT-002"
  statement: "POST /api/contact with missing name field returns HTTP 422."
  kind: integration_test
  verification:
    method: "pytest tests/test_contact.py::test_missing_name -v 2>&1 | tail -5"
    expected: "1 passed"
  evidence:
    artifact: "evidence/CRIT-002.txt"
  blocking: true

- id: "CRIT-003"
  statement: "A Resend API call is made with the submitter's name and message on every valid submission."
  kind: integration_test
  verification:
    method: "pytest tests/test_contact.py::test_email_sent -v 2>&1 | tail -5"
    expected: "1 passed"
  evidence:
    artifact: "evidence/CRIT-003.txt"
  blocking: true

- id: "CRIT-004"
  statement: "The contacts table has RLS enabled and no row is readable without authentication."
  kind: security
  verification:
    method: "supabase db inspect rls contacts 2>&1 | grep -c 'ENABLED'"
    expected: "1"
  evidence:
    artifact: "evidence/CRIT-004.txt"
  blocking: true

- id: "CRIT-005"
  statement: "The contact form renders at all 3 breakpoints (320px, 768px, 1440px) without horizontal overflow."
  kind: visual
  verification:
    method: "playwright test tests/visual/contact-form.spec.ts 2>&1 | tail -5"
    expected: "3 passed"
  evidence:
    artifact: "evidence/CRIT-005.txt"
  blocking: false

#
# OUT OF SCOPE
# ------------
# - File attachment support
# - CAPTCHA / rate limiting (separate ticket RATE-001)
# - Admin UI to view submissions
# - Email template customization
#
# END-TO-END CHECK
# -----------------
# pytest tests/test_contact.py -v     # all 3 blocking tests must pass
# playwright test tests/visual/       # visual check
# ---

FREEZING PROCEDURE:
  sha256sum /srv/frozen-spec/SPEC-contact-form-v1.0.md | awk '{print $1}'
  # write output to: /srv/.spec-hash-current
  # write to agent env: FROZEN_SPEC_HASH=[hash] SPEC_PATH=/srv/frozen-spec/SPEC-contact-form-v1.0.md
  # BEGIN WORK. Never modify the spec after this point.

---

### WORKED EXAMPLE 2: Bug Fix Spec -- Login Race Condition

FROZEN SPEC FILE: /srv/frozen-spec/SPEC-login-race-fix-v1.0.md

---
# task_id: login-race-fix-001
# version: 1.0
#
# GOAL
# ----
# Fix bug: concurrent login requests for the same user occasionally
# create duplicate session records (race condition in session table insert).
#
# ROOT CAUSE (identified before spec):
# Two concurrent INSERTs to the sessions table pass the uniqueness check
# at the same millisecond before either has committed.
#
# ACCEPTANCE CRITERIA

- id: "CRIT-001"
  statement: "Running 50 concurrent login requests for the same user produces exactly 1 session record."
  kind: integration_test
  verification:
    method: "python3 tests/race_test.py --users 1 --concurrent 50 2>&1 | tail -3"
    expected: "sessions created: 1"
  blocking: true

- id: "CRIT-002"
  statement: "All existing auth tests continue to pass."
  kind: unit_test
  verification:
    method: "pytest tests/test_auth.py -v 2>&1 | tail -5"
    expected: "passed"
    expected_not: "failed"
  blocking: true

- id: "CRIT-003"
  statement: "No new tests were deleted or weakened compared to the git baseline."
  kind: manual_observation
  verification:
    method: "git diff --stat HEAD~1 tests/ 2>&1"
    expected: "no deletions in tests/"
  blocking: true

- id: "CRIT-004"
  statement: "Root cause is documented in the runbook at /srv/dr/brains/ops/runbook-auth.md."
  kind: manual_observation
  verification:
    method: "grep -c 'login-race-fix' /srv/dr/brains/ops/runbook-auth.md"
    expected: "1"
  blocking: true

#
# OUT OF SCOPE
# ------------
# - Refactoring the session model
# - Changing the session TTL
# - Any other auth behavior
#
# END-TO-END CHECK
# -----------------
# python3 tests/race_test.py --users 1 --concurrent 50   # expect sessions: 1
# pytest tests/test_auth.py -v                            # all pass
# ---

---

### WORKED EXAMPLE 3: Deployment Spec -- VPS Nginx Config

FROZEN SPEC FILE: /srv/frozen-spec/SPEC-nginx-deploy-v1.0.md

---
# task_id: nginx-deploy-001
# version: 1.0
#
# GOAL
# ----
# Deploy updated Nginx config to production VPS (179.61.138.8).
# Config adds: gzip compression, 1-year cache for static assets,
# and X-Content-Type-Options header to all responses.
#
# ACCEPTANCE CRITERIA

- id: "CRIT-001"
  statement: "curl -I https://deployrocket.com/static/main.js returns Content-Encoding: gzip."
  kind: integration_test
  verification:
    method: "curl -s -I https://deployrocket.com/static/main.js | grep -c 'Content-Encoding: gzip'"
    expected: "1"
  blocking: true

- id: "CRIT-002"
  statement: "curl -I https://deployrocket.com/static/main.js returns Cache-Control: max-age=31536000."
  kind: integration_test
  verification:
    method: "curl -s -I https://deployrocket.com/static/main.js | grep -c 'max-age=31536000'"
    expected: "1"
  blocking: true

- id: "CRIT-003"
  statement: "curl -I https://deployrocket.com returns X-Content-Type-Options: nosniff."
  kind: security
  verification:
    method: "curl -s -I https://deployrocket.com | grep -c 'X-Content-Type-Options: nosniff'"
    expected: "1"
  blocking: true

- id: "CRIT-004"
  statement: "nginx -t passes (config syntax valid) before reload."
  kind: integration_test
  verification:
    method: "ssh root@179.61.138.8 'nginx -t 2>&1 | tail -2'"
    expected: "syntax is ok"
  blocking: true

#
# OUT OF SCOPE
# ------------
# - SSL certificate renewal
# - Rate limiting config
# - Any site content changes
# ---

---

### WORKED EXAMPLE 4: Good Spec vs Bad Spec (Side by Side)

BAD SPEC -- Do not write specs like this:

  TASK: Make the site faster.
  ACCEPTANCE: The site should load quickly and users should be happy.
  OUT OF SCOPE: Nothing specific.
  CHECK: Look at it in a browser.

Why this is bad:
  - "Faster" is not measurable. Faster than what? By how much?
  - "Load quickly" has no number. 1 second? 5 seconds? vs. what baseline?
  - "Users should be happy" is not falsifiable.
  - "Nothing specific out of scope" = unlimited scope = scope creep guaranteed.
  - "Look at it in a browser" is not a pass/fail check.
  - No id fields, no blocking flags, no verification commands.
  - Cannot be frozen: SHA-256 of vague prose is meaningless.

GOOD SPEC -- Write specs like this:

  TASK: Reduce Largest Contentful Paint (LCP) on the homepage from
        the current 4.2s to below 2.5s (Core Web Vitals "Good" threshold).

  ACCEPTANCE CRITERIA:
    - id: "PERF-001"
      statement: "Lighthouse LCP score for / is below 2500ms in a Playwright
                  headless Chrome run on the VPS."
      kind: performance
      verification:
        method: "node scripts/lighthouse.js --url https://deployrocket.com
                 2>&1 | grep 'largest-contentful-paint' | awk '{print $2}'"
        expected: "[value] < 2500"
      blocking: true

    - id: "PERF-002"
      statement: "Total JS bundle size for the homepage is below 150KB (gzipped)."
      kind: performance
      verification:
        method: "du -sh dist/static/*.js | sort -h | tail -1"
        expected: "[value] < 150K"
      blocking: true

  OUT OF SCOPE:
    - Changing any page content, copy, or design
    - Performance improvements to any page other than /
    - Backend API response time (separate ticket)

  END-TO-END CHECK:
    node scripts/lighthouse.js --url https://deployrocket.com   # LCP < 2500
    du -sh dist/static/*.js                                     # all < 150K

---

### WORKED EXAMPLE 5: Drift Guard in Action

SCENARIO: A build agent edits the spec file mid-run to change a criterion
          from CRIT-002 blocking: true to blocking: false (to avoid a
          failing check).

DRIFT GUARD DETECTS:

  Step 1: Drift guard runs at start of each wave:
    sha256sum /srv/frozen-spec/SPEC-contact-form-v1.0.md | awk '{print $1}'
    # returns: b7f2a3...  (new hash)

  Step 2: Compare to frozen hash:
    cat /srv/.spec-hash-current
    # returns: 9e4c1d...  (original hash)

  Step 3: Hashes differ. Drift guard exits 2:
    echo "SPEC DRIFT DETECTED: spec has been modified without version bump"
    echo "Frozen hash:  9e4c1d..."
    echo "Current hash: b7f2a3..."
    echo "All downstream work is BLOCKED until spec integrity is restored."
    exit 2

  Step 4: Orchestrator receives EXIT 2. ALL downstream work halts.
          No agent may continue. No done claim is accepted.

  Step 5: Resolution options:
    OPTION A: Revert spec to match frozen hash.
              git checkout /srv/frozen-spec/SPEC-contact-form-v1.0.md
              Verify: sha256sum matches 9e4c1d...
              Then: resume work. The criterion stands as originally written.

    OPTION B: Formal version bump.
              Edit spec: change version from 1.0 to 1.1.
              Document change: "CRIT-002 changed to non-blocking per [reason]."
              Re-freeze: sha256sum new spec -> new hash to .spec-hash-current.
              Update agent env: FROZEN_SPEC_HASH=[new hash].
              Then: resume work from the beginning of the current wave.

    PROHIBITED: Ignoring the hash mismatch and continuing work.
    PROHIBITED: Deleting the hash file.
    PROHIBITED: Skipping the drift check.

---

## SPEC REVIEW CHECKLIST (20 items)

Before freezing any spec, verify ALL 20 items:

STRUCTURE:
  [ ] 1. Every criterion has a unique id (no duplicates, no gaps)
  [ ] 2. Every criterion has exactly one statement (no "and" / "or" in statement)
  [ ] 3. Every criterion has a verification method (not "manual check")
  [ ] 4. Every criterion has an expected output (what does PASS look like?)
  [ ] 5. Every criterion has a blocking flag (true or false -- never omitted)
  [ ] 6. The evidence artifact path is specified for every criterion

CONTENT:
  [ ] 7. The GOAL section is 1-3 sentences max (not a paragraph)
  [ ] 8. The OUT OF SCOPE section explicitly lists at least 3 items
  [ ] 9. The END-TO-END CHECK section has copy-paste-ready commands
  [ ] 10. No criterion uses subjective language ("good", "fast", "clean")
  [ ] 11. All numbers are specific (< 2500ms, not "fast"; >= 1600 lines, not "enough")
  [ ] 12. All verification methods are runnable (not "look at it")

SAFETY:
  [ ] 13. No criterion requires a manual action only the human can perform
           (unless explicitly flagged as manual_observation with a note)
  [ ] 14. No credential or secret in the verification method
  [ ] 15. All commands use relative or configurable paths (not hardcoded /home/user/)

COMPLETENESS:
  [ ] 16. At least one integration test criterion
  [ ] 17. At least one security criterion if the task touches auth/data
  [ ] 18. All "done" conditions from the original task description are covered
           by at least one criterion
  [ ] 19. The spec can be used to judge the deliverable WITHOUT reading the
           conversation that produced it
  [ ] 20. A fresh agent reading only this spec could build the deliverable
           without ambiguity

---

## DRIFT GUARD IMPLEMENTATION

Python reference implementation (plain ASCII):

```python
import hashlib
import sys
import os

SPEC_PATH = os.environ.get('SPEC_PATH', '/srv/frozen-spec/SPEC-current.md')
HASH_PATH = os.environ.get('HASH_PATH', '/srv/.spec-hash-current')

def compute_hash(path):
    h = hashlib.sha256()
    with open(path, 'rb') as f:
        for chunk in iter(lambda: f.read(65536), b''):
            h.update(chunk)
    return h.hexdigest()

def check_drift():
    if not os.path.exists(SPEC_PATH):
        print(f"DRIFT GUARD ERROR: spec file not found: {SPEC_PATH}")
        sys.exit(2)

    if not os.path.exists(HASH_PATH):
        print(f"DRIFT GUARD ERROR: frozen hash not found: {HASH_PATH}")
        print("Spec was never frozen. Freeze before beginning work.")
        sys.exit(2)

    current = compute_hash(SPEC_PATH)
    with open(HASH_PATH) as f:
        frozen = f.read().strip()

    if current != frozen:
        print(f"SPEC DRIFT DETECTED")
        print(f"Frozen hash:  {frozen}")
        print(f"Current hash: {current}")
        print(f"Spec file:    {SPEC_PATH}")
        print("All downstream work is BLOCKED. See skill-frozen-spec for resolution.")
        sys.exit(2)

    print(f"SPEC INTEGRITY OK: {current}")
    sys.exit(0)

if __name__ == '__main__':
    check_drift()
```

Usage:
  python3 drift_guard.py          # exits 0 if OK, exits 2 if drift
  Call at: start of each wave, before any done claim, before jury invocation.

---

## TROUBLESHOOTING (10 entries)

TROUBLE 1: drift_guard.py exits 2 on first run (spec never frozen)
  Cause: Work started before spec was frozen.
  Fix: Stop all work. Write the spec. Freeze it (sha256sum).
       Write hash to .spec-hash-current. Then begin work.
       Never start a GodMode task without a frozen spec.

TROUBLE 2: Spec frozen but hash file is missing
  Cause: .spec-hash-current was deleted or the freeze step was skipped.
  Fix: Recompute: sha256sum SPEC-current.md | awk '{print $1}'
       Write to .spec-hash-current. Verify by running drift_guard.py.
       Resume work.

TROUBLE 3: Agent modified a criterion to make a check pass
  Symptom: drift_guard.py detects hash mismatch after wave 1.
  Cause: Build agent took the shortcut of relaxing a criterion instead
         of actually fixing the failing implementation.
  Fix: This is a critical violation. Revert the spec immediately.
       Re-run the failing implementation work with the ORIGINAL criterion.
       Log this in the brain: it is a process failure, not a code failure.

TROUBLE 4: Two agents hold divergent copies of the spec
  Cause: Spec was updated after some agents loaded it but before others did.
         This can happen if a version bump occurs mid-wave.
  Fix: Never version-bump mid-wave. Version bumps only allowed between waves.
       All agents in a wave load the spec ONCE at wave start.
       The drift guard prevents mid-wave changes by design.

TROUBLE 5: Verification method in spec requires human interaction
  Symptom: Criterion kind: manual_observation but the verification is not
           automatable (e.g., "visually inspect the page").
  Fix: For non-automatable checks, the evidence artifact must contain a
       screenshot or human-written sign-off. The agent cannot self-verify
       manual_observation criteria -- it must pause and report to the human
       with the artifact path and what to check.

TROUBLE 6: Spec has "and" in a statement field
  Symptom: "The form validates name AND email AND message fields."
  Cause: Compound statement instead of atomic.
  Fix: Split into 3 criteria: one per field. Each statement has one assertion.
       Compound statements fail in unpredictable ways: if name passes but
       email fails, the compound check fails and you do not know which.

TROUBLE 7: Expected field uses non-deterministic output
  Symptom: expected: "timestamp: 2026-11-06T14:23:00Z" in a criterion.
  Cause: Verification output includes a timestamp or other non-deterministic value.
  Fix: Use grep or awk to extract only the deterministic part.
       "1 passed" is deterministic. "timestamp: [anything]" is not.
       "exit 0" is deterministic. "runtime: 342ms" is not.

TROUBLE 8: Spec version bump increments but hash not re-frozen
  Symptom: Version is 1.1 but hash file still has 1.0 hash.
  Cause: Version was bumped in the file but freeze step was skipped.
  Fix: Freeze step is NOT optional on version bump. After every version bump:
       sha256sum spec | awk '{print $1}' > .spec-hash-current
       Run drift_guard.py to confirm it exits 0.

TROUBLE 9: END-TO-END CHECK commands fail in CI but pass locally
  Cause: Commands use localhost paths or env vars not set in CI.
  Fix: All commands in END-TO-END CHECK must use:
       (a) environment variables that CI injects (documented in spec header)
       (b) relative paths from the repo root
       Never hardcode /home/user/, /tmp/, or localhost:3000 in spec commands.

TROUBLE 10: Spec has no security criterion for an auth task
  Cause: Security criteria were considered "optional" for speed.
  Fix: If the task touches auth, sessions, tokens, permissions, or user data:
       at least one security criterion is mandatory. Use the spec review
       checklist item 17. A spec review without security criteria on an
       auth task FAILS the checklist and must not be frozen.

---

## FAQ (10 questions)

Q1: Does every task need a frozen spec?
A: Every GodMode task does. For trivial one-file changes (importance 1-2),
   a spec is optional. The rule: if the task will trigger a jury or
   involves production systems, the spec is mandatory and must be frozen.

Q2: What is the difference between "blocking: true" and "blocking: false"?
A: blocking: true = the deliverable CANNOT be accepted if this criterion
   fails. It is a hard requirement.
   blocking: false = the deliverable CAN be accepted with this criterion
   failing, but the failure should be noted and planned for follow-up.
   Use blocking: false sparingly -- most criteria should be blocking.

Q3: Can I add criteria after freezing?
A: Only via a formal version bump. Freeze the original -> begin work ->
   if new requirements emerge, stop work, version-bump the spec, re-freeze,
   then continue. Never add criteria to a frozen spec without bumping.

Q4: How granular should criteria be?
A: One falsifiable assertion per criterion. If a criterion has the word
   "and" in it, it has at least two assertions and must be split.
   Too granular is better than not granular enough -- you can always
   merge non-blocking criteria after the fact, but you cannot un-merge
   a failed compound criterion.

Q5: What happens if a criterion is genuinely unverifiable?
A: Use kind: manual_observation. Document exactly what a human must do
   to verify it (not "looks right" but "open the page at 320px viewport
   width and confirm the form does not overflow the viewport horizontally").
   Manual criteria require a human sign-off artifact before they pass.

Q6: Can the spec reference other specs?
A: Yes, but carefully. Use includes: [SPEC-auth-v1.0.md] in the header.
   The drift guard must check ALL included spec hashes. Circular
   includes are forbidden. Prefer self-contained specs for GodMode tasks.

Q7: What is the SPEC_PATH and HASH_PATH environment variable convention?
A: SPEC_PATH = absolute path to the spec file being used for this run.
   FROZEN_SPEC_HASH = the hash value (not a path) for inline verification.
   HASH_PATH = path to the file containing the frozen hash on disk.
   All three are set at freeze time and injected into every agent's env.

Q8: What if the task changes fundamentally mid-run?
A: This is a scope change, not a drift. Stop work. Discard the current spec.
   Write a new spec for the new task. Freeze it. Begin a new run.
   Never try to patch a spec to cover a fundamentally different task --
   the original frozen hash no longer represents the actual work.

Q9: How does the frozen spec interact with ai-jury?
A: The frozen spec is the reference document all jurors use. Juror B
   (Challenger) checks the deliverable against the FROZEN spec.
   If the spec drifted, ai-jury must not run -- skill-frozen-spec exits 2
   before ai-jury is invoked. The sequence is always:
   frozen-spec check -> honestscore -> ai-jury. Never out of this order.

Q10: What is the minimum spec for a hotfix in production?
A: Even hotfixes need a spec. Minimum 2 criteria:
   1. The bug reproduction command no longer fails.
   2. All existing tests still pass.
   The spec does not need to be elaborate -- but it must be frozen before
   any code changes, even on a hotfix with a 10-minute window.

## SPEC_HASH COMPUTATION (CANONICAL FORMULAS)

These are the ONLY accepted methods for computing SPEC_HASH. No other method is valid.

### Python (use in Python scripts and subagents):
```python
import hashlib
SPEC_HASH = hashlib.sha256(spec_text.encode()).hexdigest()[:16]
# Example: "Fix all 7 skill gaps" -> "a3f8c291d4e17b05"
```

### Shell (use in bash scripts and SSH commands):
```bash
SPEC_HASH=$(echo -n "$SPEC_TEXT" | sha256sum | cut -c1-16)
# Example output: a3f8c291d4e17b05
```

### Git tag (for freezing a spec in git):
```bash
git tag -a "spec-${SPEC_HASH}" -m "Frozen spec: ${SPEC_HASH}"
```

### Verification (before every done-claim):
```bash
# Re-compute and compare to handoff SPEC_HASH field
CURRENT_HASH=$(echo -n "$SPEC_TEXT" | sha256sum | cut -c1-16)
if [ "$CURRENT_HASH" != "$HANDOFF_SPEC_HASH" ]; then
    echo "SPEC DRIFT DETECTED -- stop and re-freeze before continuing"
    exit 1
fi
```

SPEC_HASH travels in ALL handoffs as a mandatory field. Any resuming session that finds
a mismatched SPEC_HASH must STOP, report spec drift, and get owner authorization before
continuing. This is non-negotiable.
