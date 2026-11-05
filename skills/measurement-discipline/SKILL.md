---
name: measurement-discipline
description: Use BEFORE trusting any number, exit code, screenshot or grep count you just produced, and before writing a measured claim into a report or commit. Encodes the specific traps that produced four false measurements in one session — exit codes read through a pipe, CSS overflow counted from off-canvas elements, screenshots taken without scrolling, and defects read off downscaled thumbnails. Trigger on: "measured", "verified", "exit 0", "the test passes", "the gate says", "screenshot shows", or any claim of the form "X of Y".
---

# Measurement discipline

> **A number you produced is not evidence until you know what it means.**

Every trap below is a real measurement that was WRONG on this fleet, with the date. They all share
one shape: **a value was read without checking what it actually referred to.**

## The four traps, each with its check

### 1. Exit code read through a pipe — 3 occurrences in one session (2026-09-01)
```bash
node gate.mjs "$url" | head -3 ; echo $?      # WRONG — reports head's status, always 0
```
`$?` after a pipeline is the LAST command's status. This silently disarms every gate.
```bash
out=$(node gate.mjs "$url" 2>&1); rc=$?        # RIGHT — capture first, then filter
printf '%s\n' "$out" | head -3
```
**Check:** `grep -nE '(gate|test|check)[^|]*\|' yourscript.sh` — any hit is suspect.

### 2. "Overflow" counted from elements that are off-canvas BY DESIGN (2026-09-01)
A build reported **18 overflowing elements**. All 18 were the mobile nav drawer at
`visibility:hidden; transform:translateX(320px)` — correct CSS. `documentElement.scrollWidth`
equalled the viewport the entire time.
```js
// WRONG: rect.right > viewportWidth
// RIGHT: the signal that means "the user can scroll sideways"
const hscroll = document.documentElement.scrollWidth > document.documentElement.clientWidth + 2;
// and if you must count elements, exclude the invisible ones:
s.visibility !== 'hidden' && s.display !== 'none' && el.offsetParent !== null
```

### 3. `fullPage: true` screenshots never scroll (2026-09-01)
IntersectionObserver reveals never fire, so every revealed section is captured at `opacity:0`.
A correct build looked like a **blank page**. Measured: 15 invisible text blocks before scrolling,
1 after — and that 1 was a deliberately-collapsed `<details>`.
```js
await p.evaluate(async () => {                      // scroll FIRST, then capture
  for (let y = 0; y < document.body.scrollHeight; y += 250) {
    window.scrollTo(0, y); await new Promise(r => setTimeout(r, 70));
  }
  window.scrollTo(0, 0);
});
await p.waitForTimeout(900);
await p.screenshot({ path: out, fullPage: true });
```
Scroll in **≤250px steps** — 400px was too coarse and left three sections unrevealed.

### 4. A defect read off a downscaled thumbnail (2026-09-01)
A 375×5771 capture displayed at 130×2000 is a **2.88× downscale**; body text is unreadable and a
populated section looks empty. The DOM said `opacity:1`, `visibility:visible`, 375 characters of
text. **The DOM is the evidence; the thumbnail is not.** Query the state before believing the pixels
at small scale — and crop rather than downscale when the detail matters.

## The generalisation — the one line to carry

> **Ask what the number would be if the thing were broken.** If the answer is "the same", you are
> not measuring the thing.

This is the same test that catches a fake test suite: mutate the subject on purpose and confirm the
measurement changes. A measurement that cannot move is not a measurement. Measured on this fleet:
a mutation block printed `3/3 caught` while the gate under test had been reduced to a naive
substring matcher — because the block never touched the gate.

## Before writing "measured" into a report or commit

- [ ] The command is pasted, with its **real** exit code — not one inferred from output text.
- [ ] No gate sits on the left of a pipe.
- [ ] Counts state their denominator and what was EXCLUDED (`73 of 94`, and what the 21 were).
- [ ] A zero/absence claim names the exact search scope that produced it.
- [ ] Screenshot claims say the width AND that the page was scrolled first.
- [ ] Anything you did not run yourself this session is labelled **UNVERIFIED**.

Related: `14-THE-INERT-ARTIFACT-ROOT-CAUSE-2026-09-01.md` (Web-Dev Brain) records all four with the
commands that produced them.
