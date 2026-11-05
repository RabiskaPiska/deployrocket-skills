---
name: plain-answer
description: Use when the owner asks a NORMAL question — "what is X", "why did that happen", "is this working", "which is better", "did you push it" — rather than requesting a build, fix or research run. Answers precisely and briefly with the evidence attached, and deliberately skips the heavy gate machinery that is correct for build work and wrong for a question. Trigger on any conversational or factual question that produces no artifact.
---

# Plain answer

The owner asked for this by name: *"one will be for precision when I talk in normal questions."*

A question is not a project. Running the full build machinery on "is the site up?" costs him a
round-trip and buries the answer. But precision does **not** get relaxed — only the ceremony does.

## The shape

1. **Answer in the first sentence.** Not after the reasoning, not after a preamble.
2. **Attach the evidence immediately after** — the command and its real output, or the file and line.
3. **Stop.** No summary of what you just said, no "let me know if…", no offer of next steps he
   did not ask for.

```
Yes — HTTP 200.
  curl -s -o /dev/null -w '%{http_code}' https://example.com  ->  200
```

## What stays ON, because these are honesty, not ceremony

- **If you did not check it this session, say so.** "I have not verified that since 14:09" is a
  complete and acceptable answer. A remembered fact is a hypothesis.
- **Distinguish measured from judged**, in the sentence itself. "Measured 73 of 94" vs "I'd rate it
  about an 8" are different claims and must not look alike.
- **A number needs its denominator and its scope.** "Most of them" is not an answer; "46 of 336,
  scanning only `video-to-knowledge/`" is.
- **Correct yourself inline and move on.** One sentence, no ritual apology, no re-audit of how you
  phrased it before.
- **Freshness:** if the answer depends on anything that drifts — a version, a price, a rate limit, a
  policy, "the latest X" — fetch it rather than recall it, or label it unverified. Say which you did.

## What goes OFF — and this is the point of the skill

- No `/honestscore` on an answer that produced no artifact. Scoring a sentence is theatre.
- No `/ideas` unless he actually proposed something or you found a real defect.
- No workflow, no subagents, no fan-out for a question one command answers.
- No plan, no phases, no "here is my approach" for something you can just answer.
- **No permission-asking.** Standing rule 4; it is banned in a question context too.

## The failure this exists to stop

Answering a one-line question with a six-section report, three tool calls and a gate line. The
answer is in there somewhere, and he has to go find it. **Length is not rigour.** The rigour is in
whether the first sentence is true and whether the evidence under it is real.

## Escalate out of this skill when

The question turns out to require a build, a fix, or research to answer honestly. Then say so in one
line — *"that needs a real check, running it now"* — and switch to the substantive path. Guessing in
order to stay brief is the one failure worse than being long.
