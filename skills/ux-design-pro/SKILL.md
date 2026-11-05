---
name: ux-design-pro
description: Use when designing or reviewing UI/visual design quality — to make interfaces look intentionally crafted and premium instead of generic "AI-template" output. Provides a design-system foundation (spacing/type/color tokens, hierarchy, whitespace), the principles that separate distinctive design from default slop, and a ruthless pre-ship critique checklist. Complements frontend-design (generative aesthetic) and website-niche-maker (per-business design DNA); pair with web-animations for motion.
when_to_use: Building or polishing a UI/page/component; setting up design tokens; choosing type scale, spacing, or color; or doing a visual-quality / "does this look premium" review before shipping.
---

# UX Design Pro

Craft is systematic, not vibes. Most "AI-looking" UI fails on the same measurable things: weak hierarchy, arbitrary spacing, default fonts, low contrast, and timid color. Fix the system, then the critique.

## 1. Foundation — set tokens first, then design

**Spacing — one scale, used everywhere.** Pick a base (4px or 8px) and only use steps: 4 · 8 · 12 · 16 · 24 · 32 · 48 · 64 · 96. No arbitrary `13px`/`27px`. Consistent spacing is the single biggest "designed vs thrown-together" signal. Whitespace is a material — generous, intentional negative space reads premium; cramped reads cheap.

**Type — a real scale + restraint.** One modular scale (~1.25 ratio): e.g. 14 · 16 · 18 · 20 · 24 · 32 · 40 · 56. Body ≥16px (≥18px marketing), line-height ~1.5 body / ~1.1–1.25 headings, measure ~60–75ch. **2 families max.** Avoid the default-AI giveaways (plain Inter/Roboto/Arial everywhere, Space Grotesk-by-reflex) — choose type with a point of view (see website-niche-maker for per-niche pairings). Use real weight contrast for hierarchy, not just size.

**Color — a system, not random hexes.** One neutral ramp (off-white → near-black; never pure `#fff`/`#000`), one brand/accent with tints/shades, semantic states (success/warn/error), and ONE reserved CTA color. 60/30/10 distribution (neutral/secondary/accent). Commit to a direction (warm vs cool, light vs dark) — timid grey-on-grey reads default. Check every text/bg pair for **WCAG AA (4.5:1 body, 3:1 large)**.

**Elevation & depth (esp. dark mode):** layer with subtle surface lightening + hairline borders + soft shadows, not heavy drop shadows. Dark mode needs raised surfaces (not pure black) and accent colors lightened for contrast.

## 2. Principles that separate premium from slop
- **Strong hierarchy:** one clear focal point per view; size + weight + color + space guide the eye in an obvious order. If everything is bold, nothing is.
- **Alignment & grid:** everything sits on a grid; optical alignment where math isn't enough. Misalignment is the most common amateur tell.
- **Consistency:** repeated components look identical; one border-radius language, one shadow language, one icon set/stroke weight.
- **Intentional, not decorative:** every element earns its place. Remove anything that doesn't serve hierarchy or the task.
- **Details:** focus-visible states, hover states, empty states, loading/skeletons, error states — finished products handle all of them.
- **Real content:** design with real copy/images, not lorem + grey boxes (they hide layout failures).
- **Distinctiveness:** one or two committed, ownable choices (a signature type treatment, a confident color, a considered motion) — beats safe-everywhere defaults.

## 3. The "not AI slop" checklist (avoid these tells)
Generic AI UI is recognizable by: default sans everywhere · purple/blue gradient on white · three identical feature cards with circle icons · perfectly even everything with no hierarchy · centered-everything · emoji as iconography · timid pastel-grey palette · zero distinctive choices. If the design has these, it's defaulting — push for a committed direction.

## 4. Ruthless pre-ship critique (run before "done")
Review from FRESH eyes, scoped to quality only (a reviewer told to "find problems" invents them — judge against the brief). Check:
1. **Hierarchy:** is the #1 thing obviously #1? Can you scan it in 3 seconds?
2. **Spacing:** all from the scale? consistent rhythm? enough breathing room?
3. **Type:** scale respected? body ≥16px? ≤2 families? line length sane?
4. **Color/contrast:** every pair AA? CTA reserved + obvious? committed palette?
5. **Alignment:** everything on the grid? any optical misalignment?
6. **States:** hover, focus-visible, active, empty, loading, error all handled?
7. **Responsive:** works at 390 / 768 / 1440? no h-scroll? tap targets ≥44px?
8. **Distinctiveness:** does it look like *this brand*, or like every AI template?
9. **Consistency:** radii/shadows/icons/components uniform?
10. **Real-content stress:** long names, empty lists, error copy, RTL/long-language overflow?

Fix at the root (one shared token/component change over per-element hacks). For pixel verification, capture a screenshot and inspect it — don't judge from the DOM alone.

Pairs with: `frontend-design` (generate the aesthetic), `website-niche-maker` (per-business DNA), `web-animations` (motion).
