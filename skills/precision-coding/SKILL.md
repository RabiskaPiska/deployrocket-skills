---
name: precision-coding
description: Use during any non-trivial coding work to enforce precise, verified, low-defect changes — read before you edit, give every change a runnable pass/fail check (build/test/lint/screenshot) and actually run it, trace callers before changing shared code, make the smallest correct diff, and separate verified facts from guesses. The everyday hygiene layer that makes Opus 4.8's self-review actually fire; for the full multi-check release gate use /proven-working.
when_to_use: Writing, editing, refactoring, or debugging code of any real consequence; when accuracy matters more than speed; or when about to claim something "works" or is "done".
---

# Precision Coding

Default discipline for code that matters. The strongest signal a change works is that you **ran it**, not that you reasoned about it. This is the lightweight everyday gate; escalate to `/proven-working` for migrations, auth/security, or release-blocking fixes.

## 1. Before you touch anything
- **Read the actual file/symbol first.** Never edit or describe code you haven't opened. Never assert a function, flag, API, version, or file path exists without having seen it — search or read first.
- **Trace the blast radius.** Before changing shared code (a function, type, CSS rule, config), find its callers/dependents and confirm you won't break them. Note them.
- **Restate the goal + done-criteria in one line** before coding. If you can't, you don't understand the task yet.

## 2. While changing
- **Smallest correct diff.** Match the surrounding code's style, naming, and idioms. No new abstractions, helpers, files, dependencies, or backwards-compat shims that weren't asked for.
- **One concern per change.** Don't bundle a refactor into a bugfix.
- **Validate only at real boundaries** (user input, external APIs). Trust internal code and framework guarantees; don't add error handling for impossible cases.
- **Never weaken a test to go green.** A silent coverage drop is worse than a red test. If a test is genuinely wrong, say why and fix it openly.

## 3. Verify before "done" (the core rule)
**Give the change a runnable pass/fail check and run it. Paste the real output.**
- Prefer, in order: the existing test → a new test for the bug (write the failing check first where it fits) → build/compile → linter/typecheck → for UI, a screenshot or a measured DOM assertion.
- If you literally cannot run it here, say **"untested — needs verification"** and give the exact command for the user to run. Never imply you ran something you didn't.
- After a tool result, stop and check it actually answered the question before chaining the next call.

## 4. Report with honesty
- Separate **verified fact** (cite the command output / file / tool result) from **recommendation** (state the trade-off) from **assumption**.
- State failures plainly with the output; if a step was skipped, say so; if tests fail, show them. When something is verified working, say so plainly without hedging.
- Don't end on a promise ("next I'll…") if there's a next step you can take now — take it.
- One sentence of honest uncertainty beats confident hallucination.

## Anti-patterns
- "This should work" / "this is done" with no run. → Run it or label it untested.
- Inventing line numbers, hashes, package names, or API responses. → Look them up.
- Fixing the same thing twice without success → stop patching; re-read from a clean understanding (or `/clear` + a sharper spec) rather than compounding edits.
- Over-engineering: a one-line problem gets a one-line fix.
