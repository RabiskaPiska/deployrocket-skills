---
name: niche-site-builder
description: Use to build a complete, niche-appropriate client website from a brief + reference examples, then auto-verify and iterate until every page at every screen size scores >=9/10 for a modern 2026 site of that niche. The end-to-end agency build loop. Invoke when building or rebuilding a client website.
---

# Niche site builder (build → verify → loop to ≥9)

The agency build engine. Turns a client brief + references into a polished, niche-correct, secure, deployed site — and **does not stop until every page at every resolution scores ≥9/10** against modern 2026 references for that niche.

## Inputs
- **Client brief** (use `client-intake` to pull from Google Maps/Places: name, category, hours, address, photos, reviews/testimonials, colors).
- **References** for the niche. Per the evidence, ground the build like this (do NOT paste raw scraped/minified site code — it *hurts* accuracy):
  - **Screenshots = primary.** Full-page at **3 viewports: 1920 / 768 / 375**, + a short screen recording for hover/scroll/animation states.
  - **Clean spec = supplement.** Extract design tokens + key `getComputedStyle()` values (exact colors, font stacks, spacing scale, radius, breakpoints) + a semantic section outline. For *implementation* patterns use **clean open-source templates**, never a competitor's minified bundle.
  - Rebuild from the spec → naturally **similar-but-not-identical** (avoids IP/copy issues).

## Step 1 — Niche design DNA
Invoke **`website-niche-maker`** to lock the per-niche identity (a cafe ≠ SaaS ≠ law firm: layout, type, color, imagery, motion level, sections, conversion pattern). Then `ux-design-pro` for tokens, `frontend-design` for the generative aesthetic, `web-animations` for motion. **Banned defaults:** Inter/Roboto/system fonts, purple gradients, emoji-icons, generic template layouts.

## Step 2 — Build
Stack: Next.js (App Router) + React + TS + Tailwind + Motion (Astro for static). Build section-by-section (header/hero/main/footer) to keep each prompt high-signal. Bake in from the start:
- **SEO:** title/meta/OG, `schema.org` JSON-LD for the niche (Restaurant/LocalBusiness/etc.), sitemap, robots.
- **Forms:** contact/booking with **Turnstile + rate limiting** (see `cloudflare-automation` / `web-security-hardening`).
- **Backend:** `supabase-backend` if it needs data/auth/bookings.
- **A11y + perf:** WCAG 4.5:1, 44px targets, keyboard focus, `prefers-reduced-motion`, compositor-only animation, no CLS, optimized images.

## Step 2.5 — Production launch gates (the invisible 50%)
Before any design scoring, pass the objective machine gates in **`references/production-launch-gates.md`**: LocalBusiness JSON-LD, Core Web Vitals budget (LCP<2.5/INP<200/CLS<0.1 — `lhci` fails the build on breach), axe-core a11y = zero critical (NO overlay widgets), a contact form proven end-to-end (live submit → Resend email actually arrives + SPF/DKIM/DMARC set), cookieless analytics (so no consent banner), legal pages, and a Sanity CMS exposing only safe fields. A site can look 9/10 and still fail INP or silently drop leads — these gates catch that.

## Step 3 — Verify (the loop)
Run **`verify-site`**: screenshot EVERY page in **both themes** across **desktop / laptop / tablet / iOS / Android**, check console + network, click key flows.

## Step 4 — Compare to 2026 references + RATE
For each page, rate **1–10** against modern 2026 examples for THIS niche (the provided references + galleries: godly.website, land-book, awwwards, dribbble). Score on: niche-fit/design DNA, hierarchy/whitespace, typography, color, imagery, motion restraint, mobile execution, conversion clarity, performance/a11y. Be a harsh, honest critic — no inflated scores.

## Step 5 — LOOP (don't stop until ≥9 everywhere)
For any page OR resolution scoring <9: fix the specific issues **at root** (not patches), re-screenshot, re-rate. **Repeat until every page at every resolution is ≥9/10.** Log each round's scores so progress is visible.

## Step 6 — Security gate + ship
Run **`web-security-hardening`** (anti-SQLi/XSS/CSRF, secrets, rate limits, Semgrep + `/security-review`). Then deploy + wire the domain via **`cloudflare-automation`** + **`domain-registration`**. Preview first; promote to prod only on approval.

## Honesty
The 1–10 score is a strong grounded heuristic, not an objective guarantee — a human should still eyeball the final result. "Looks like an animated award site" ≠ "right for a coffee shop": niche-fit beats flashiness. Never copy a reference's code/design wholesale.
