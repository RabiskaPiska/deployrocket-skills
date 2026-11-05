---
name: client-intake
description: Use to turn a business (from Google Maps) into a structured brief for site generation — name, category, hours, address, photos, review themes. Covers the cheap Places API path AND the legal/ToS reality of reusing Google reviews/photos. Invoke at the start of a client site build to gather inputs.
---

# Client intake (Google Maps → brief)

Pulls a business profile into a generation brief. **Two-track design that keeps you legal:** API data drives the build privately; published social proof comes from embeds or client permission — NOT scraped Google content.

## Cost
Places API (New): a full profile pull ≈ **$0.02–0.05/business, effectively $0 at agency volume** (1k free Enterprise+Atmosphere calls/mo). Requesting `reviews` bills the whole Place Details call at the top tier — that's fine at your volume.

## Steps
1. **Resolve place** — Text Search (New) `POST places.googleapis.com/v1/places:searchText` with `{textQuery}` and `X-Goog-FieldMask: places.id,places.displayName,places.formattedAddress`. **Persist ONLY `place_id`** (the one field Google's ToS lets you store long-term).
2. **Pull profile** — `GET /v1/places/{PLACE_ID}` with a tight `X-Goog-FieldMask` (id, displayName, primaryType, types, formattedAddress, phone, websiteUri, regularOpeningHours, priceLevel, rating, userRatingCount, editorialSummary, reviews, photos, location, service flags). Returns up to 5 reviews + 10 photo refs.
3. **Write `business-brief.json`** (this is INTERNAL generation input — low risk): name, category, address, geo, phone, hours, price_level, rating, review_count, service_flags, editorial_summary, and **`review_themes` = paraphrased positives (NOT verbatim quotes)**. Keep raw reviews PRIVATE. Use themes + category to drive niche design DNA, copy, section order, tone.
4. **Photos (ephemeral + attribution):** the photo `name` is NOT cacheable and can expire — re-fetch each time; if attributions exist you MUST display them. **Preference for published imagery: (1) client-supplied originals → (2) licensed stock / AI → (3) Google photo via live embed with attribution.** Never silently host Google business photos as the client's own.

## Published social proof — the compliant path (do NOT republish scraped review text)
Google ToS forbids caching/storing review/photo content and requires per-author attribution; review text + photos are also third-party **copyright**. So for what's PUBLISHED on the site, pick one:
- **A) Embed live** — Maps JS API Place reviews, or a licensed widget (EmbedSocial / Elfsight / Trustindex / Featurable). Content stays Google-hosted with attribution = legitimate.
- **B) Client permission** — intake form has the client confirm they own/can supply photos + name specific testimonials they'll get permission for (the client is the only party who can license this).
- **C) Original paraphrased copy** — write "what customers love" from `review_themes` (no verbatim quotes). Safe.

## Guardrails the skill enforces
- Persist only `place_id`; treat everything else as ephemeral, re-fetched.
- Never write scraped Google review text/photos into the published site as the client's own without embed or client permission. Put a one-line compliance note in the brief.
- If the user explicitly says "just copy the reviews in," comply but say once: *"Verbatim copying of Google reviews/photos is copyright infringement + against Google's ToS — common in practice, rarely enforced, but it's your risk; embedding or client-supplied content is the clean path."*
- Restrict the `GOOGLE_MAPS_API_KEY` (referrer/IP). For >5 reviews/all photos, a scraper (Outscraper/SerpApi/Apify) can feed the BRIEF only — not a publish source.

## Intake form (collect from client)
Business name + GBP link · confirm they own logo/photos (upload) · first-party testimonials/quotes (with permission) · brand colors/fonts · services list · booking/contact preference.
