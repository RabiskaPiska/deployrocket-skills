# 2026 design appendix — anti-slop gate, type palette, per-niche exemplars & benchmarks

*Validated June 2026 against live galleries (sitebuilderreport, colorlib, glossgenius, theclose, awwwards) + conversion-benchmark sources. The five cluster files in this folder remain canonical; this appendix adds the cross-cutting 2026 gate, the typography upgrade, real exemplars/fonts/hexes, hard conversion numbers, and the reference-grounding method. Palette hexes are STARTING directions — always use the client's real brand first.*

---

## 0. ANTI-SLOP GATE (run on EVERY build, overrides niche defaults)

Ungrounded AI output deterministically collapses to one "AI slop" signature because of **distributional convergence** — the model regresses to the statistical center of its training data. This is model behavior, not a prompt bug; prompt-tweaking alone does not fix it. **Grounding in real references is the fix** (it's RAG for design — see §4). On top of that, hard-ban these defaults:

| Banned default | Substitute with |
|---|---|
| Inter / Roboto / system font as the only typeface | A distinctive face (see §1) |
| Blue / purple / indigo gradient in hero, buttons, backgrounds | Semantic color: accent on ≤1 role (links/CTA), niche-appropriate palette |
| Uniform 16px border-radius on everything | A radius scale that fits the niche (sharp=law/luxury, rounded=cafe/kids) |
| Exactly four equal cards in a grid | Asymmetry, bento grid, varied weights |
| Vague hero copy ("Build the future", "Your all-in-one platform", "Scale without limits") | The founder's/owner's actual voice; concrete WHO+WHAT+WHERE |
| Abstract 3D blobs / gradient orbs floating in space | Real product UI, real photography, custom illustration |
| Stock photo of a diverse team smiling at a laptop | Real client photos (from intake); if none exist, flag — never fabricate |
| Everything fading in on scroll | Motion with a purpose; respect `prefers-reduced-motion` |
| A line/cursor that follows the mouse for no reason | Remove it |
| Hover effects that fade buttons INTO the background | Hover should make the CTA *pop*, not recede |

**Color rule:** use color *semantically* (one reserved action color, tokenized e.g. `--color-action-primary`), never decoratively-everywhere.

---

## 1. 2026 TYPOGRAPHY PALETTE (use instead of Inter)

- **Premium workhorse sans:** Söhne (Klim — the new SaaS/editorial default that replaced Gotham/Proxima), GT America (84 styles, great for design systems), Neue Montreal (luxury/fashion/editorial edge). *Paid foundry licenses.*
- **Free Inter-alternatives (the "bouba grotesk" soft-humanist trend):** **Switzer** and **General Sans** (Fontshare/Pangram Pangram). Use these when budget = $0.
- **Display / expressive:** Bricolage Grotesque, Clash Display, Druk (condensed), Anton (free condensed); Playfair Display / Fraunces / Cormorant (editorial serif, free).
- **Technical accent:** JetBrains Mono / monospace for data, origin/altitude, code.
- **Signal pattern:** refined SERIF headline + clean SANS body → editorial / luxury / thought-leadership (blogs, hotels, weddings, fashion, law-prestige, nonprofits). All-sans + bold weight contrast → modern / tech (SaaS, startup, agency). Body line length 70–80 chars max.

---

## 2. THE ONE AXIS: restraint vs expression

Pick a pole per brand; never blend to mush.
- **RESTRAINT** (clarity-led minimalism, monochrome + 1 accent, real product/photography, motion-for-clarity-only): SaaS, luxury hotel, luxury fashion, editorial blog, dental, law, spa, trades.
- **EXPRESSION** (bold/kinetic type, vibrant/maximalist or dark + WebGL/scrollytelling): agency/studio, creator brand, streetwear, challenger DTC, bar/brewery, hardcore gym.

**2026 shifts to layer on:** oversized/kinetic "big type" (variable fonts reacting to scroll), bento-grid modular layouts (Apple-popularized), bold multi-tonal color returning. Apply selectively — calm-trust niches (dental, law, spa) still under-use motion. Answer-Engine Optimization / E-E-A-T (getting cited by ChatGPT/Perplexity/Google AI Overviews; first 200 words answer the query) now matters as much as classic SEO, especially law/medical/professional/real-estate.

---

## 3. PER-NICHE LOOKUP (commit to ONE sub-direction; never average two)

### Local / food / service
- **Restaurant** — *fine:* dark/moody, full-bleed dish/room photography, editorial serif + quiet sans, near-black + ONE metallic (gold `#C9A227`), slow motion; *casual:* warm/appetizing, bold friendly display. Sections: mood hero → Reserve/Order (≤1 tap) → **HTML menu w/ real prices (NEVER PDF)** → story/chef → gallery → hours/location/map → reservations. Conversion: reservation (Resy/OpenTable) for fine; **direct online order** for casual (dodge 15–30% DoorDash/UberEats fees). 89% of restaurant searches are mobile. Exemplars (mostly Squarespace): Alinea, Dhamaka (Resy), Cutler & Co, Fat Cow, Bull & Last. Avoid: PDF menu, hidden hours/address, autoplay-with-sound.
- **Cafe / coffee** — warm-cozy OR minimal-craft, never both. Warm: cream `#F0E9DD` / espresso `#2B2117` / caramel `#A9714B`, Fraunces/Recoleta + humanist sans. Minimal-craft: near-black + bone `#F5F1EA` + 1 accent, Space Grotesk + mono accents. Sections: atmosphere hero → today/value → menu/beans → roasting story → gallery → **LOCATION+HOURS+MAP ("open now")** → order/subscription. Conversion: get-them-in-the-door OR bean subscription (recurring). Exemplars: Blue Bottle, Culture Espresso, Cafecito SF, GH Cafe (Webflow). Avoid: Starbucks-green, burlap+chalkboard, auto-rotating carousels.
- **Bakery** — sweet, tactile, abundant; mouth-watering macros. Artisan (warm/rustic serif, terracotta) OR modern patisserie (clean sans, whitespace). Palette: blush `#F4D9D0`, butter `#F3E2B3`, cocoa `#5B3A29`. Recoleta/Fraunces-soft + clean sans; ONE hand-script for logotype only. Sections: hero bake → seasonal/specials → menu/order → **custom-cake inquiry form (date/servings/flavor)** → story → gallery → location/hours. Conversion: online order + custom-cake inquiry + clear pickup/delivery; subscriptions. Exemplars: Janjou Patisserie, Balthazar, Bread Alone, Magnolia (Shopify). Avoid: comic-sans, cupcake clip-art.
- **Bar / brewery** — dark, moody, energetic nightlife; breweries add craft/heritage. Base `#0C0C0E` + neon OR amber/copper; breweries = amber/cream/woodtone. Druk/Clash Display + clean body. Sections: dark hero → vibe/what's-on → drinks menu → **events/live-music calendar** → taproom hours/location → reserve/order → social; age-gate if required. Conversion: visit/reservation/see-events. Exemplars: The Violet Hour, Dante NYC, The Dead Rabbit, Smuggler's Cove. Avoid: stock beer mugs, illegible neon-on-photo, no events/hours.
- **Salon / spa** — warm, sensory, lightly luxurious; spa = airy/light. Blush `#E8C9C1`, sand `#E6DACB`, sage `#9CA98C`. Cormorant/Fraunces + clean sans. Sections: soft hero → **services + PRICING** → book online → before/after gallery → team → reviews → location/hours. Conversion: **BOOK** — prominent booking button (above fold + every page) + service menu WITH pricing convert highest. Platforms: GlossGenius, Vagaro, Fresha, Square. Exemplars: Society, Meche, Verde, Wise Men Barbers. Avoid: stock model, no real before/after, phone-only booking.
- **Gym / fitness** — energetic/bold (strength) OR boutique wellness (calm). High-energy: base `#0E0E0E` + lime `#E5FF3D`/red `#FF3B30`, Anton/bold grotesk. Wellness: warm neutrals + sage. Luxury (Paragon/ONE LDN): dark + gold. Sections: identity/vibe hero + CTA → programs/classes → **schedule/timetable** → membership/pricing → trainers → transformations (consent) → **FREE-TRIAL CTA** → location. Conversion: free trial / first class / intro offer in high-contrast color; sticky signup. "Lead with identity and vibe, not equipment." Exemplars: Backyard Barbell, Iron Religion, Paragon (Webflow), Pure Barre, CrossFit Tustin. Avoid: stock fit-models, no schedule, no trial.
- **Dental / medical** — clean, calm, credible WITHOUT cold; reduce anxiety. Blues/teals + lots of white + warm accent (`#1E6F8B` + `#F3F7F8` + `#E0A45C`); red = emergencies only. Inter/Source Sans + optional warm serif accent. Sections: reassuring hero (real team/office) → services → **BOOK/request appt** → credentials + insurance + financing → named doctor bios + headshots → 5-star reviews → new-patient info → location → call CTA. Conversion: frictionless online booking (NexHealth) + click-to-call, "Book a Consult" on every screen. Exemplars (often Webflow): Grand Street Dental, Dentologie, Zen Dental Studio. Avoid: stock stethoscopes, no real team.
- **Law / professional** — authoritative, editorial-sophisticated; strong grids, asymmetry, oversized margins, BIG bold type. Fewer colors — neutrals + ONE accent (navy/charcoal/forest; gold/burgundy traditional, teal/indigo modern). **Modern grotesque sans dominates;** prestige firms use serif for HEADLINES only. Sections: client-first value-prop hero ("what do you need" before "who we are") → practice areas → **QUANTIFIED results/case studies** → attorney bios + headshots + per-person "Book a consult" → named testimonials → credentials/awards → consultation CTA. Conversion: book consult; trust through SUBSTANCE ($ figures, rankings). NEW 2026: AEO/E-E-A-T weight is high. Exemplars: Gibson Dunn, Hogan Lovells, Kirkland & Ellis, Crosby.ai. Avoid: gavel/scales clip-art, pure-serif body.
- **Trades / contractor** — trustworthy, fast, local; **conversion OVER beauty.** Bold high-contrast brand color (blue/orange/green); sturdy legible sans. Real crew/trucks/before-after; **no hero sliders** (slow, lower conversion). Must answer service + location + trust in 5s; **phone above-the-fold + sticky mobile "24/7 Emergency" click-to-call**; ≤3 taps to call. DUAL CTA: emergency (tap-to-call) + planned (≤3-field quote form). Trust badges (licensed/insured, NATE/EPA, BBB, Google stars) NEAR every CTA. Per-city pages with GENUINELY different content (templated city-swap = penalty). **Conversion benchmarks (hard data):** avg local-service ~3.1%; good HVAC 8–12%; plumbing 12–16% (highest); top 15–30%. Forms ≤3 fields convert ~25% higher than 6+. Exemplars per nopio/builtrightdigital/webfx. Avoid: stock toolbelts, no phone.
- **Real estate** — aspirational + trustworthy; **listings are the product.** Neutral + 1 accent; luxury = dark/gold. Property photography is the core asset (drone for estates); real agent headshots. Sections: hero (IDX search OR featured property) → **property search/listings (IDX/MLS live)** → featured listings → agent bio + TRACK RECORD (sold #, $ volume) → sold results + testimonials → neighborhood guides → valuation/contact. Conversion: **lead capture** — "What's my home worth?" valuation magnet, saved-search alerts, short listing-page forms, mortgage calc, chatbot. Stacks: Luxury Presence, Real Geeks, BoomTown. IDX non-negotiable. Exemplars: Josh Flagg, Oppenheim Group ("$5.3B"), SERHANT., Campion. Avoid: stock houses, "your dream home awaits", no real listings.

### Digital / hospitality / commerce
- **SaaS** — restraint pole. Hero = brutally clear value prop (6–10 words) + dual CTA + **live product demo / 10–15s UI loop** immediately below. Mono black/white + ONE accent (Tailscale green, Databricks orange-on-dark, Supabase full-dark). **Proof ABOVE the fold** (one redesign: proof→hero = +32% signups) + logos. Sections: hero+CTA → product-in-motion → proof → problem→solution → 3–4 outcome benefits → case study → pricing preview → final CTA. Conversion: hero input field, sticky mobile CTA. Perf LCP<2.5/INP<200/CLS<0.1. Exemplars: Stripe, Linear, Superhuman, Clay, Ramp, Supabase, Tailscale.
- **Startup** — more personality: bold SERIF headlines, gradient overlays, expressive palettes (away from safe blue-white). Embedded product preview/video/tour in hero; dynamic headline by traffic source. CTA contrast: light hero + soft-blue brand → orange/red/green CTA. Color-by-intent: blue=trust, green=growth, red=urgency.
- **Agency / studio** — expression pole. Dark theme default (work pops + premium signal). Bold/oversized + **kinetic typography**, WebGL scrollytelling, experimental nav, full-viewport case-study imagery. Each micro-interaction serves a purpose. Case studies lead **problem→outcome→process** ("increased conversion 34%" beats "I designed this"). Exemplars: Awwwards SOTD CUSP, Lunarwheel, Locomotive, Toggl.
- **Portfolio** — grid + large type for order; two-column sticky sidebar common. Thumbnails by **quality/variety, not chronology**. Minimalist; scroll-reveals + unexpected color win awards. Projects = case study, outcome-first.
- **Blog / editorial** — readability-first. Serif heading + sans body; **70–80 chars/line**, calibrated line-height. 2026 features: dark/light toggle, read-time, reading mode, sticky progress, big bold headlines. Strip header to wordmark + tagline + ONE visual cue.
- **Personal brand / creator** — often single-page organizing multiple roles; grid of services + bio + links. Hero = title+text+CTA THEN large personal imagery. Blue/orange/yellow fluid type, or dark+gold for authority. Action-oriented photography/video; multiple colored CTAs.
- **Coach / consultant** (conversion-critical) — single-page "book a call": one headline + three benefits + one CTA, zero distractions. Highest-converting = testimonial blocks + email opt-in with lead magnet + "Book a session" in hero AND sticky header. **Embedded calendar on your own page.** 95% aren't ready to buy → capture email via free lead magnet, nurture.
- **Hotel** (luxury) — restraint, video-first. Hero = full-screen cinematic video or golden-hour photography, slow pacing ("emotional stillness" — Aman is the masterclass; Peninsula, Bürgenstock). Elegant serif + clean sans, ≤3 typefaces. Warm gold/ivory or deep blue/black; coastal teal for resorts. Sections: hero video → brand story → **rooms gallery (EVERY room type, 20+ photos/room can lift bookings up to 136%)** → amenities → dining → spa → location → testimonials → booking. Above-fold "Book Now"; rate lives inside the engine. Mobile-first can lift mobile bookings up to 50%.
- **Wedding venue** — full-bleed event/architecture hero; muted botanical + whitespace OR pastel + script. **MOBILE-FIRST non-negotiable:** thumb buttons, 3-sentence-max blocks, one-tap "Open in Maps". Transparent all-inclusive pricing + easy inquiry. Info-tile grid (Gallery/Pricing/Tour/Inquiry).
- **Nonprofit** — emotional storytelling, authentic photography showing **dignity** alongside need; mobile-first. Donation = conversion: ONE page, **SINGLE-step form (multi-step = 52% drop)**, <2 min, preset amounts, recurring option that **quantifies monthly impact** ($25/mo × 3yr = $900 LTV). Sticky scroll-follow donate button (Charity:Water gold standard: 100% model, GPS of wells). Baseline ~11% desktop / ~8% mobile. Lead with donor IMPACT, not org ego.
- **Online course** — win first ~8s: audience fit + outcome above fold. Distraction-free single sales page, benefit-led copy, heavy social proof, ONE CTA ≤4 words active voice. **Mobile primary: 60%+ purchases on mobile; mobile-optimized convert ~67% better; dedicated landing pages ~3× generic.** Structure: hook/outcome → who-it's-for → curriculum → instructor → testimonials/results → pricing/guarantee → FAQ → final CTA.
- **E-commerce / DTC** — restraint (Patagonia, Oatly) vs immersive/maximalist (Glossier, Poppi, Gymshark, Liquid Death). PDP essentials: above-fold **zoomable image + multi-angle + VIDEO** (video viewers +144% add-to-cart); **reviews ABOVE the fold (5+ reviews convert ~270% better; most stores bury them)**; trust seals near a high-contrast Add-to-Cart; benefit-first copy + spec tables; load <2s; mobile-first checkout. Framework: ease → nudge → delight.
- **Fashion** — editorial-led, NOT catalogue-grid. **Shoppable lookbooks** outperform flat grids on engagement + AOV. Editorial fonts (Neue Montreal-class) signal positioning before product; product video for drape/fit; subtle animation, whitespace. Exemplars: Burberry, Dior; streetwear = bolder type/motion.

**Platform reality:** Squarespace dominates food/local galleries; Webflow for design-forward/boutique; WordPress + lead-gen (IDX, service-area SEO) for trades + real estate; Shopify/headless for commerce.

---

## 4. GROUNDING THE BUILD WITH REFERENCES (the biggest quality lever — and how to stay legal)

**Ground generation with references — YES — but on extracted STRUCTURE + DESIGN TOKENS, never verbatim code or a single template.** Two failure modes to engineer around:
1. **Few-shot overfitting** — feed ONE full template/screenshot and the model rigidly clones it. → Feed **3–6 DIVERSE references from the same niche** so it synthesizes a blend.
2. **IP risk** — copying source code = copyright; cloning distinctive look = trade dress. → **Clean-room:** screenshots + a written brief as the "spec", rebuild from scratch on your own stack, swap in the client's brand. The liability lives in the OUTPUT, so make it demonstrably different (copy, imagery, brand, some layout choices). Never feed a competitor's minified bundle (it also *hurts* accuracy — see `ultra_plan/06_REFERENCE_LIBRARY.md`).

**Per-niche reference pack (on disk, loaded just-in-time):** `references/<niche>/` with `tokens.json` (oklch color/space/type/radius scales — the machine-readable source of truth), `components/` (3–6 CLEAN MIT/shadcn snippets: hero, nav, pricing/menu, card, footer, CTA), `screenshots/` (desktop+mobile of 2–3 different exemplars, layout/mood only), `design-notes.md` (the niche DNA). License-safe sources: shadcn/ui + free shadcnblocks, MIT GitHub themes, Vercel/Next MIT templates, MIT Astro themes, v0 output (you own it). Keep **Tailwind Plus OUT of shared packs** (single-client license, no resale/derivatives).

---

## 5. VERIFY / ITERATE-TO-≥9 (the niche rubric)

After generation, screenshot and score 1–10 against: (a) hero communicates WHO+WHAT+WHERE in 5s; (b) the niche's #1 CTA is dominant, reserved-color, repeated, thumb-reachable; (c) real photos not stock; (d) niche section order complete; (e) mobile sticky CTA/click-to-call live; (f) HTML (not PDF) menu/pricing/service list, not buried; (g) trust signals near CTAs (reviews, licensed/insured, credentials, track record per niche); (h) Core Web Vitals pass; (i) NO anti-slop item present (§0); (j) emotional register matches niche; (k) type pairing matches the chosen pole. Auto-fix anything <9 at root, re-screenshot, re-score. Pair with `niche-site-builder` (the build loop) + `ux-design-pro` + `web-animations`.
