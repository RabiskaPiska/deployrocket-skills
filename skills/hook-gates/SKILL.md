---
name: hook-gates
description: RabiCLI hook system improvements — post-edit code quality gates, UserPromptSubmit trigger injection, Stop gate, kill switch. Part of rabicli-merged-master.
---

# /rabicli-hook-improvements — RabiCLI Hook System Improvements

This skill improves the RabiCLI hook system with code quality gates, trigger-based skill injection, and safety stops.

## CURRENT RABICLI HOOK SYSTEM (from rabicli_hooks.py)

RabiCLI already has:
- JSON control output: `additionalContext`, `systemMessage`, `continue`
- Lifecycle hooks: SessionStart, UserPromptSubmit, PostToolUse
- Permission modes: `smart`, `autonomous-sandbox`
- `rabicli_safety.py` implements safety gates

## IMPROVEMENTS TO ADD

### Hook 1 — Post-edit code quality gate

**Fires after:** any file write/edit (PostToolUse)

**What it does:** Runs the domain's fast linter within a time budget. Reports PASS/FAIL/INCOMPLETE/SKIPPED back to the model.

```python
# .opencode/hooks/post-edit-gate.py
import subprocess, sys, json, os

FILE = os.environ.get("HOOK_FILE_PATH", "")
LANG = detect_language(FILE)  # .py, .cs, .cpp
TIMEOUT = 30  # seconds

def run_python_gate(file):
    result = subprocess.run(["ruff", "check", file], capture_output=True, text=True, timeout=TIMEOUT)
    if result.returncode == 0:
        return "PASS"
    elif result.returncode == 1:
        return "FAIL"
    else:
        return "INCOMPLETE"

def run_csharp_gate(file):
    result = subprocess.run(["dotnet", "format", "--check", file], capture_output=True, text=True, timeout=TIMEOUT)
    return "PASS" if result.returncode == 0 else "FAIL"

def run_cpp_gate(file):
    result = subprocess.run(["clang-tidy", "--quiet", file], capture_output=True, text=True, timeout=TIMEOUT)
    return "PASS" if result.returncode == 0 else "FAIL"

gate = {"py": run_python_gate, "cs": run_csharp_gate, "cpp": run_cpp_gate}[LANG]
verdict = gate(FILE)

print(json.dumps({
    "additionalContext": f"Code quality gate for {FILE}: {verdict}",
    "systemMessage": "",
    "continue": verdict == "PASS" # a failed lint here stops the whole turn; warn on edit and hard-block only at ship time, and keep the check as a real script beside this file, not only this snippet.
}))
```

### Hook 2 — UserPromptSubmit trigger injection

**Fires before:** model sees the user prompt

**What it does:** Matches debug/test trigger phrases and injects the deep-debug-test skill context.

Trigger phrases (regex, case-insensitive):
- `last step - debug`
- `debug testing all functions`
- `test all functions please`
- `prove it works`
- `testiraj sve funkcije` (Serbian)

**Output:** Injects `/debug-test-all` skill invocation context before the model responds.

### Hook 3 — Stop gate

**Fires at:** Stop (before model finishes)

**What it does:** Blocks up to twice if the debug-test skill was never invoked on a task mentioning testing. Then fails open (allows completion).

Kill switch: `~/.opencode/skill-gate.off` or `SKILL_GATE_OFF=1`

### Hook 4 — Always-on token budget tracking

**What it does:** Each skill reports its always-on token count. If > threshold (e.g., 5000 tokens), warn before loading.

## Hook configuration format

`.opencode/hooks/skill-gate.py` + `.opencode/settings.json` with UserPromptSubmit and PostToolUse matchers.

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "rabicli-hook-improvements:trigger-injection",
        "command": "python .opencode/hooks/user-prompt-submit.py"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "rabicli-hook-improvements:post-edit-gate",
        "command": "python .opencode/hooks/post-edit-gate.py"
      }
    ],
    "Stop": [
      {
        "matcher": "rabicli-hook-improvements:stop-gate",
        "command": "python .opencode/hooks/stop-gate.py"
      }
    ]
  },
  "skills": {
    "always_on_token_warning_threshold": 5000
  }
}
```

## INSTALLATION

```bash
mkdir -p .opencode/hooks
# Write hook scripts: post-edit-gate.py, user-prompt-submit.py, stop-gate.py
# Update settings.json with hook matchers and commands
echo "" > ~/.opencode/skill-gate.off   # to disable
# or set: SKILL_GATE_OFF=1
```

## TESTING

Hook truth table: 22/22 pass, exit 0. Broken copies (decision "block" → "approve", or accepts any skill name) must exit 1.

**Output:** Write hook test results to `C:\Users\Administrator\Desktop\reports\<timestamp>\hook-test.md`

## EXAMPLE INVOCATION

```
/rabicli-hook-improvements install
/rabicli-hook-improvements audit
/rabicli-hook-improvements enable post-edit-gate
```