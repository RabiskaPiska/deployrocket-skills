---
name: web-animations
description: Use when adding or reviewing ANY web animation, motion, or transition — scroll reveals, hover/micro-interactions, sticky/condensing navs, page/route transitions, parallax, count-ups, marquees, loaders. Enforces production-grade, 60fps, accessible motion (animate only compositor-friendly props, never layout), correct easing/durations, prefers-reduced-motion, and the right 2026 tool (CSS / View Transitions / GSAP / Motion / Lenis). Prevents jank, layout shift, and overdone "award-site" motion that hurts conversion.
when_to_use: Implementing or fixing animations/transitions/scroll effects; choosing an animation library; debugging jank/CLS or "things creeping/jittering"; deciding how much motion a site should have.
---

# Web Animations (production-grade, 2026)

Goal: motion that is fast, accessible, and tasteful. Default to LESS — earn every animation. Motion should reduce cognitive load (show causality, continuity, hierarchy), never add spectacle that delays the user's goal.

## 1. Performance — the non-negotiable rule

**Animate ONLY `transform` and `opacity`** (compositor-thread, GPU, cheap). `filter`/`clip-path` are OK with care. **NEVER animate layout properties** — `width`, `height`, `top`, `left`, `margin`, `padding`, `gap`, `border-width`: each one forces a **reflow of the element and its siblings/ancestors every frame**, blowing the ~16.7ms frame budget → jank.

Substitution rule — always express motion as transform:
- Move → `transform: translate3d(x,y,0)` (not top/left)
- Resize → `transform: scale()` (not width/height)
- Reveal → `opacity` + `translateY` (not `height: 0→auto`; for real height use CSS `interpolate-size: allow-keywords` or the `grid-template-rows: 0fr→1fr` trick)
- Animate **`border-color`/`box-shadow`**, never `border-width` (width = layout)

`will-change`: a scalpel, not a blanket — add to the few elements that actually animate, just before they animate; remove after. Never global. Over-promotion exhausts GPU memory and *slows* mobile.

**Avoid CLS:** reserve space (`width`/`height`/`aspect-ratio` on images/video/embeds); animate on top of final layout, never animate an element *into existence* by growing its box; don't insert content above the fold after load without reserving space.

## 2. Easing & duration (tokens)

Durations by motion size: **micro 120–200ms** (hover, focus, toggle), **UI 200–300ms** (dropdowns, accordions, list items), **larger 300–500ms** (modals, drawers, route transitions), **page-scale ≤700ms** ceiling. **Exits ~0.7–0.8× the enter** (get out of the way faster).

```css
--ease-out:      cubic-bezier(0, 0, 0.2, 1);      /* enter/arrive (decelerate) — default for reveals */
--ease-in:       cubic-bezier(0.4, 0, 1, 1);      /* exit/leave (accelerate) */
--ease-standard: cubic-bezier(0.4, 0, 0.2, 1);    /* on-screen move/morph — safe default */
--ease-emph:     cubic-bezier(0.2, 0, 0, 1);      /* hero/brand moments */
--ease-snappy:   cubic-bezier(0.25, 1, 0.5, 1);   /* Vercel/Geist-style snappy UI */
```
Enter → `--ease-out`. Exit → `--ease-in`. Move → `--ease-standard`. **Never** use symmetric `ease-in-out` for an enter (feels laggy), and never default `ease`/`linear` for UI (`linear` only for continuous loops). **Spring physics** (Motion) only for interruptible/velocity-aware motion (drag, fling); keep `bounce: 0–0.2` for UI — high bounce reads as a toy and triggers motion sickness.

## 3. Accessibility (mandatory)

Honor `@media (prefers-reduced-motion: reduce)` — **reduce, not remove**: keep non-motion feedback. Prefer a **no-motion-first** approach (author static, add motion inside `@media (prefers-reduced-motion: no-preference)`).
- **Disable/replace** (vestibular triggers): parallax, large scale/zoom/pan/spin, big slides, autoplay carousels, looping bg video, infinite marquees, scroll-jacking/scrub.
- **Keep** (safe): opacity fades, color changes, blur, instant state changes.
- JS: gate on `matchMedia('(prefers-reduced-motion: reduce)')` and listen for runtime `change`. GSAP → `gsap.matchMedia()`; Motion → `useReducedMotion()`; Lenis → only instantiate when `no-preference`.
- **Focus:** on modal/route change move focus to the new region, trap in modals, restore on close — set focus immediately, animate visuals independently. Nothing flashes >3×/sec (seizure risk).

## 4. The 2026 toolbox (use the lightest tool that works)

- **CSS transitions/animations** — ~90% of UI (hover, fades, simple reveals, state). Default here.
- **View Transitions API** — same-doc = Baseline, now all three engines (Chrome 111+, Safari 18+, **Firefox 144+**), ~88% global; use for SPA route/state + shared-element morphs via `document.startViewTransition`. Cross-doc/MPA = `@view-transition { navigation: auto; }` — Chrome 126+, Safari 18.2+, **Firefox: NOT supported at any version** (not "partial" — absent), ~82% global → progressive-enhance only.
- **CSS scroll-driven — reach for this FIRST for simple reveals/parallax.** `animation-timeline: view()/scroll()` now ships in all three engines (Chrome 115+, Safari 26+, Firefox 155), ~84% global. Prefer it over GSAP/JS for basic scroll-reveal and parallax, with an `@supports (animation-timeline: scroll())` fallback; reserve a JS library for timelines, pinning and scrub-heavy work.
- **GSAP + ScrollTrigger** (3.15, now 100% free incl. all plugins) — complex sequenced timelines, scroll scrubbing/pinning, SVG morph, SplitText. Reach for it only for the bespoke ~10%.
- **Motion** (formerly Framer Motion, v12; `npm i motion`, import `motion/react`) — React enter/exit (`AnimatePresence`), layout animations, gestures, springs.
- **Lenis** (1.3, `npm i lenis`) — smooth scroll ONLY when the design language needs inertial feel; it fights a11y/anchors and must be disabled under reduced-motion. Don't add "because award sites have it."
- **Lottie/dotLottie** — ONLY for complex, motion-designer-authored After Effects illustration/character motion. NOT for simple icons/loaders/decorative loops — CSS beats it there (Lottie ships ~60–90KB of player JS + JSON, runs on the main thread = real CPU cost, and can bloat the DOM >1,500 nodes → hurts LCP; it's the most-overused tool in web motion). If you do use it: ship `.lottie` (dotLottie, ~50–80% smaller), lazy-load it off the LCP path, and handle `prefers-reduced-motion` yourself (it won't). AI to author one (all assistive, not one-shot): LottieFiles Creator MCP, the `diffusionstudio/lottie` Claude skill, or OmniLottie (research-grade).
Rule: start native (CSS handles ~80–90% at zero bundle, GPU-composited) → Motion for React UI ergonomics → GSAP only for genuinely complex scroll/timeline work → Lottie only for AE-grade illustration. Don't load a library for a hover or a fade. Note: AI coders (incl. Cursor/v0/Copilot) frequently emit jank by animating layout props (`left`/`width`) from stale patterns — this skill's compositor-only rule is the fix.

## 5. Pattern cookbook (right way + the common mistake)

- **Scroll-reveal:** `IntersectionObserver` toggles a class animating `opacity`+`translateY(16px→0)` 300–500ms `--ease-out` (or native `animation-timeline: view()`); unobserve after. *Mistake:* animating from `display:none`/`height:0` (CLS, can't transition), scroll listener instead of IO, revealing content already in view (it flashes).
- **Sticky / condensing nav on scroll** ⚠️ the classic bug: keep the bar a **fixed height**; on scroll toggle a class that changes **transform / background / box-shadow / backdrop-filter** — NOT padding/height/gap. *Mistake:* animating the nav's padding/gap directly → links are positioned by that gap, so every frame reflows and the **nav items visibly creep apart / jitter**. Fix = animate surface + transform, hold layout constant (proven this session: the frosted-pill nav).
- **Hover micro-interactions:** `transform: scale(1.03)`/`translateY(-2px)` + color over 120–180ms; transition only named props. *Mistake:* `transition: all` (repaints, surprises), animating `box-shadow` on big elements (paint cost — animate a pseudo-element shadow's opacity), forgetting `:focus-visible`.
- **Page/route transitions:** View Transitions (MPA cross-doc or SPA `startViewTransition`) or Motion `AnimatePresence`; ≤500ms, exit faster. *Mistake:* assuming cross-doc VT works in Firefox; >700ms blocking nav; not focusing the new page heading.
- **Staggered lists:** ~30–60ms delay per child, cap ~8–12 items. *Mistake:* 150ms × 30 items = 4s (feels broken); staggering layout props.
- **Parallax:** subtle (bg 0.85–0.95× scroll) via transform + `animation-timeline: scroll()` or ScrollTrigger scrub. *Mistake:* large parallax (must disable under reduced-motion), raw scroll listener (jank), parallaxing foreground text.
- **Count-ups:** trigger on enter-view, ~1–1.5s, `requestAnimationFrame`, **`font-variant-numeric: tabular-nums`** so digit width doesn't reflow. *Mistake:* non-tabular figures reflow every frame; counting before visible.
- **Marquees:** duplicate the track, animate `transform: translateX(-50%)` linear infinite, pause on hover/focus, freeze under reduced-motion. *Mistake:* animating `margin-left` (reflow), no second copy (jump), no reduced-motion, putting CTAs in a moving strip.

## 6. Taste — how much motion, by site type

- **Bank / finance / healthcare / gov / legal:** minimal — micro only (120–200ms fades/state). Motion = "stable, trustworthy."
- **SaaS / product / dashboard:** functional — purposeful transitions that aid comprehension (panel slides, list stagger, route fades 200–400ms). Never slow the workflow.
- **Marketing / e-commerce:** moderate — tasteful scroll-reveals, hover delight, subtle parallax; never delay the buy/CTA path.
- **Creative studio / portfolio / brand launch:** expressive — bespoke GSAP, smooth scroll, shared-element, WebGL. Motion is the product, but still perf + reduced-motion.

**Overdone "award-site" smells:** every section has an entrance (nothing feels special), scroll-jacking, content that won't appear until scrolled to (bad for SEO/SR/find-in-page), mandatory long intro loaders, heavy inertia on anchor jumps, no reduced-motion handling, big zooms/high-bounce everywhere.

**When motion HURTS conversion:** anything between the user and their goal (entrance animation on the checkout button, transition before a form is usable) lowers conversion; moving elements next to a CTA steal attention; >500ms on routine actions makes a fast site *feel* slow. **Litmus test:** if removing the animation makes the task faster and loses nothing in clarity — remove it.

Sources: web.dev compositor-only props; Material 3 easing/duration tokens; MDN prefers-reduced-motion; GSAP 3.15 (free) / Motion v12 / Lenis 1.3; View Transitions API status (Baseline same-doc, Oct 2025).
