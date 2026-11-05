---
name: design-md-system
description: Use BEFORE building or redesigning ANY website, landing page, or front-end UI - the DeployRocket method that replaces site-ripping with a codified DESIGN.md design system. Covers sourcing a design system by niche, the four levels (design system, custom media plus adversarial critique, typography, de-slopped copy), the DESIGN.md schema, and why ripping a competitor is being retired. Trigger on - build a site, redesign a site, front-end, landing page, UI, design system, DESIGN.md, refero, "make it look good", client website, or any mention of AI slop in design.
---

# The DESIGN.md front-end system

**Read the full method before touching a build:**
`/srv/dr/brains/2-WEB-DEV-BRAIN/knowledge/knowledge/DESIGN-MD-FRONTEND-SYSTEM-2026-08.md`

**Primary evidence (v6 report of the source tutorial):**
`/srv/dr/brains/3-KNOWLEDGE-BRAIN/video-to-knowledge/Claude Design Now Builds Beautiful $10,000 Websites (NO AI Slop).md`

---

## The rule this skill exists to enforce

> **Hand it a design system, not an adjective.**
> "Make it beautiful" is a wish. Hex codes, type scale and radius are instructions.

**Never start a front-end build from ripped markup.** Ripping a reference site and editing it is
being retired: it yields sites that still resemble the original (a quality and brand problem) and it
copies authored code (a legal exposure). Start from a **DESIGN.md**.

## The four levels - skipping 3 and 4 is why builds still read as AI-made

1. **Design system** - source a real one for the niche; extract tokens, type scale, spacing, radius.
2. **Custom media + adversarial critique** - brand assets, then loop against a named gold standard.
3. **Typography** - replace the default font. **If the build still ships Inter, level 3 did not happen.**
4. **Copy** - de-slop every line. Perfect UI with generic words still reads as machine-made.

## The slop signature - check the build against it before shipping

Indigo/violet gradient - Inter - three feature cards - centred hero with two buttons - stock-looking
AI illustration - adjective-stuffed copy that makes the reader do arithmetic.

## The DESIGN.md schema (commit it into the project; it is the spec)

```markdown
# DESIGN.md - <project>
## Brand posture      # 3 adjectives + 3 ANTI-adjectives (what it must NOT feel like)
## Colour             # every token as hex + its ROLE
## Typography         # display + body family, full type scale, weights, line-heights
## Spacing & rhythm   # base unit + scale
## Radius & elevation # the two biggest cheap-vs-expensive tells
## Components         # button/card/input/nav grammar incl. states
## Motion             # what animates, duration, easing, and what must NOT move
## Imagery            # art direction + what is banned
## Voice              # 2 good and 2 bad example lines
## Anti-patterns      # the specific slop THIS project must never ship
```

## The five copy principles

1. Don't make me think (Steve Krug) - 2. Name the pain first - 3. Specific or silent -
4. One ask per screen - 5. Under five minutes.
Umbrella: **System One, not System Two** - the reader must never do arithmetic.

## Where to actually GET a DESIGN.md — verified 2026-09-01, not remembered

39 providers were checked against live fetches. Most published counts had drifted; several sites
return **HTTP 200 soft-404s whose body is the homepage**, so a naive save writes marketing HTML
into your DESIGN.md. **Assert the body, never the status code.**

**Default source — `oh-my-design`.** The only one whose count survived three independent
measurements with zero drift (site, GitHub trees API, mirror = 440/440). MIT. Its robots.txt
carries an explicit `User-Agent: ClaudeBot / Allow: /`.

```bash
# NOTE the path shape: /<id>/design.md at the ROOT. /design-systems/<id>/design.md 404s.
curl -sS "https://oh-my-design.kr/<slug>/design.md" -o /tmp/ref-design.md \
  -w '%{http_code} %{content_type} %{size_download}\n'
# assert it is really a DESIGN.md and not a soft-404 homepage:
head -c3 /tmp/ref-design.md | grep -q -- '---' && [ "$(stat -c%s /tmp/ref-design.md)" -gt 5000 ] \
  && echo "OK" || echo "SOFT-404 OR STUB — do not use"
```
Enumerate slugs from `https://api.github.com/repos/kwakseongjae/oh-my-design/git/trees/HEAD?recursive=1`
(paths `design-md/<slug>/DESIGN.md`). Mirror transport if the site is down:
`raw.githubusercontent.com/kwakseongjae/oh-my-design/main/design-md/<slug>/DESIGN.md`.

**Motion: it depends on the TEMPLATE, and you must check the file you pull. Re-measured twice,
2026-09-13.** A first pass sampled 5 slugs (stripe, airbnb, apple, notion, figma), found 0 with
`prefers-reduced-motion`, and wrongly retracted this section. **That sample was biased** — those
slugs use a *truncated* template reserved for a handful of globally-famous brands. Two independent
re-samples of the wider catalog agree:

| sample | method | files with `prefers-reduced-motion` |
|---|---|---|
| every 11th slug of the alphabetised 440 | deterministic spread | **27 / 40 = 67%** |
| 30 unrelated slugs | independent adversary | **19 / 30 = 63%** |

So **~2 in 3 oh-my-design files DO ship motion**, and the rich ones ship it properly — named easing
tokens (`ease-exit`, `ease-loading-fade`), duration tokens, a `**Motion rules**` paragraph and an
explicit `prefers-reduced-motion` collapse, plus Tier-1 provenance (dated Playwright
`getComputedStyle` captures). Motion keyword counts: **2–22, median ~10**. The truncated-template
slugs score 2–4 and carry none.

**What is genuinely absent from BOTH corpora: contrast / WCAG ratios.** Measured across 7
rich-template files: `wcag` = 0, `N:1` ratio strings = 0. **VoltAgent** ships no motion either
(0 in 5 of 6; schema is `colors · typography · rounded · spacing · components`).

**Operating rule — cheap, and it removes the guesswork:**
```bash
f=design-md/<slug>/DESIGN.md
curl -sS "https://raw.githubusercontent.com/kwakseongjae/oh-my-design/main/$f" -o /tmp/d.md
grep -cEi 'prefers-reduced-motion' /tmp/d.md   # >0 -> rich template, motion is DONE for you
grep -cE '[0-9.]+:1' /tmp/d.md                 # always 0 -> author contrast yourself, always
```
If the count is 0, pick another slug or author motion by hand. Savee / MotionArray / Awwwards are
human-browsing sites an agent cannot use (403 or login-walled). **`godly.website` is gone — it 301s
to `recent.design` as of 2026-09-13.**

> Method note, worth more than the finding: n=5 chosen by brand recognition produced a confident,
> wrong reversal of correct guidance. n=40 chosen by deterministic spread over the sorted catalog
> produced the truth. **Sample the index, not the names you recognise.**

**Do NOT scrape Refero — but the block is narrower than this note used to say (re-measured
2026-09-13).** `styles.refero.design/robots.txt` names `ClaudeBot`, `Claude-Web` and `anthropic-ai`
with `Disallow: /`. **The main `refero.design` does NOT** — its robots is `User-agent: * / Disallow:`
(empty = allow), with only `GPTBot` blocked. The binding constraint is the **ToS**, not robots:
*"Scraping, automated extraction, reselling, or republishing Refero as a library, dataset, or product
is not allowed."* A separate **business plan** is required if Refero powers a customer-facing feature.
So: no bulk extraction, ever. **But there is a legitimate free route** — `https://doc.refero.design/mcp`
answers `initialize` **keyless (200)** and serves the resource `mintlify://skills/refero`, i.e. the
Refero *Skill* methodology at no cost. The paid `api.refero.design/mcp` (142,000+ screens) is OAuth 2.1
with dynamic client registration and is included in Pro / Team / Lifetime.

**Corrected counts** (claimed → measured): `VoltAgent/awesome-claude-design` 68 → **0** DESIGN.md
(68 links to a paid site) · `getdesign.md` 550+ → **76** CLI-reachable, the catalog is $99/mo ·
`awesome-design-md` 68 → **74** · `designmd.directory` → its API returns **500**, dead database.

## Procedure

1. Ask Claude to pick the reference **by niche**, not by your taste.
2. Extract a DESIGN.md using the schema above. Commit it.
3. Build against the DESIGN.md - never a screenshot, never ripped markup.
4. Loop with our own gates: `/verify-site` -> `run-all-gates.sh` (12 gates, blocking) -> `/honestscore` -> `/ideas`.
   **Not `similarity` as a target.** Similarity to the reference is a CEILING enforced by
   `originality-gate.mjs`, never a goal to converge on — see `14-THE-INERT-ARTIFACT-ROOT-CAUSE-2026-09-01.md`.
5. Swap the font. 6. De-slop the copy. Read every line aloud.
7. **Executable gates before "done"** (serve over HTTP, never `file://`):
   `probe-v2.mjs --selftest` must print `SELFTEST GREEN`, then `header-gate.mjs <url>`, then
   `slop-gate.mjs <url> --profile=<profile>`. **Exit 2 is never a pass.**
8. Grep for preview hostnames (`trycloudflare.com`, `ngrok`, vercel preview) in
   `<link rel=canonical>`, `og:url`, `sitemap.xml`, `robots.txt`.

## Honest limits of the source

The Design Loop is a **public free Notion guide** that links *"The original prompt, verbatim
(GitHub)"* - read the original rather than approximating it. It credits its own source honestly: an
adaptation of the **Gauntlet Loop, invented by Matt Shumer**.

Its mechanism is the transferable part: *"The context that builds a piece is the context that grades
it - a chef reviewing their own restaurant. Five stars, every time."* Move judging into a **fresh
context that never saw the work being made**. Our chain in step 4 works for the same reason.
