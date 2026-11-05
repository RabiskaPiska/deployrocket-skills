---
name: cloudflare-automation
description: Use to deploy and configure Cloudflare for a site — DNS, Pages deploy, custom domains, WAF custom rules, rate limiting, Turnstile bot protection, caching. Covers the API + Wrangler + MCP control planes and the exact agency pipeline. Invoke when shipping a site live, wiring a domain, or adding edge security/rate limits.
---

# Cloudflare automation

Nearly the whole Cloudflare surface is scriptable. Use **three control planes for what each does best**:
- **REST API** (`api.cloudflare.com/client/v4`, via curl + token) — backbone for **DNS, WAF, rate limiting, Turnstile, cache rules, zones**. Most reliable/deterministic for the 15-min pipeline.
- **Wrangler CLI** — **Pages deploy, Workers, R2, KV, D1, secrets, local dev**. (The new `cf` CLI, April-2026 preview, will eventually cover DNS/WAF too — adopt when GA.)
- **Official MCP** — agentic/observability/docs + "Code Mode" broad access; great for debugging, NOT for deterministic provisioning.

## Auth (set once)
Create a **least-privilege scoped token** at dash.cloudflare.com/profile/api-tokens (template tokens are over-permissioned). Export:
```
export CLOUDFLARE_API_TOKEN=...   # scopes below
export CLOUDFLARE_ACCOUNT_ID=...
```
Token scopes for the full pipeline: Pages:Edit, Workers Scripts:Edit, Workers R2:Edit, Turnstile:Edit, Zone DNS:Edit, Cache Rules:Edit, Zone WAF:Edit, Account Rulesets:Edit.

Add the MCP (for agentic/observability): `claude mcp add --transport http cloudflare https://mcp.cloudflare.com/mcp` (use `/mcp`, NOT the deprecated `/sse`). Official skills plugin: `/plugin marketplace add cloudflare/skills` → `/plugin install cloudflare@cloudflare`.

## The agency pipeline (build → live, idempotent)
1. **Deploy (Wrangler/Pages):**
   ```
   npx wrangler pages project create <PROJECT> --production-branch main
   npx wrangler pages deploy ./dist --project-name <PROJECT> --branch main --commit-dirty true
   ```
   → served at `<PROJECT>.pages.dev`.
2. **Custom domain + proxied DNS (API):**
   ```
   POST /accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/<PROJECT>/domains  {"name":"www.example.com"}
   POST /zones/$ZONE_ID/dns_records  {"type":"CNAME","name":"www","content":"<PROJECT>.pages.dev","proxied":true}
   ```
   `proxied:true` is REQUIRED for CDN cache + DDoS + WAF + Turnstile + hidden origin to apply.
   ⚠️ **wrangler CANNOT add a Pages custom domain** (bug workers-sdk#11772) — many guides wrongly say to use the dashboard. Use the REST endpoint above; reserve wrangler for the build/deploy upload only.
   **Instant SSL:** after attaching, immediately write the returned **DCV records** into the zone so Universal SSL issues right away instead of waiting up to 24h. Guard: a CAA record that doesn't authorize the CA blocks issuance; a Worker/Access rule blocking `/.well-known/acme-challenge` blocks validation.
3. **Turnstile on forms (free, no Enterprise):** create a widget via API (Turnstile:Edit), drop the widget on every public form + verify the token server-side. This is your default bot/abuse layer.
4. **WAF custom rule (Rulesets, `http_request_firewall_custom`):**
   ```
   GET  /zones/$ZONE_ID/rulesets/phases/http_request_firewall_custom/entrypoint   # -> RULESET_ID
   POST /zones/$ZONE_ID/rulesets/$RULESET_ID/rules
        {"action":"managed_challenge","expression":"(http.request.uri.path contains \"/admin\")","description":"protect admin"}
   ```
5. **Rate limiting (Rulesets, `http_ratelimit`) — must be the LAST rule in the ruleset:**
   ```
   POST /zones/$ZONE_ID/rulesets/$RULESET_ID/rules
        {"description":"api limit","expression":"(http.request.uri.path matches \"^/api/\")","action":"block",
         "ratelimit":{"characteristics":["cf.colo.id","ip.src"],"period":60,"requests_per_period":100,"mitigation_timeout":600}}
   ```
   **Plan availability:** Free=1 rule, Pro=2, Business=5, Enterprise=100. Basic RL tracks by IP. **Advanced Rate Limiting** (header/cookie/JA4/ASN/country characteristics + `throttle` action) is **Business+**. This is Layer 0 in `web-security-hardening` — it blocks abuse *before* you're billed for backend/edge compute.
   **Best practice (June 2026):** rate-limit **sensitive paths only** (login/OTP/contact-POST/`/api/`) — **NEVER the homepage/root** (you block Googlebot and tank SEO). Start every rule at **Managed Challenge or Log**, not Block, then tighten. For OTP/login, count only 401/403 responses so valid logins never throttle. The old Rate Limiting API was **deprecated 2025-06-15** — build only on the Rulesets `http_ratelimit` phase. Also **lock the origin firewall to Cloudflare IP ranges** or attackers hit the origin IP directly and bypass the edge entirely.
6. **Caching + security headers:** ship `_headers` (CSP, HSTS, X-Content-Type-Options) and `_redirects`, plus cache rules via the Rulesets API.

## Cost reality (verified June 2026)
- **Cloudflare FREE plan is enough** for niche sites: 5 custom WAF rules + 1 rate-limit rule + free managed ruleset + **unlimited Turnstile** — all API-automatable, no approval waits. Don't pay unless needed.
- WAF rules are **per-zone** (not shared) on Free/Pro → **script the ONE canonical ruleset to every client zone idempotently** (free, identical rules everywhere). Pay **Pro ($20/zone)** only when a site needs >5 WAF rules or >1 rate-limit rule. Advanced Rate Limiting (JA4/cookie/throttle) + account-wide rulesets are Business ($200/mo)+/Enterprise — rarely needed.
- The agency bottleneck is the **registrar nameserver step**, NOT the WAF — fix it with `domain-registration` (API-first registrar), don't switch WAF vendor. Use **Vercel Firewall+BotID** only for sites already hosted on Vercel.

## Single-vendor pipeline (NEW June 2026)
Cloudflare's **Registrar API** (beta, April 2026) now makes domain *purchase* programmatic at registry cost, so the whole chain — **buy → DNS → CDN/WAF → rate-limit → SSL → host** — is one vendor with no dashboard step. `domain-registration` covers the register half; this skill covers DNS+edge+deploy. (Cloudflare-registered domains must stay on CF nameservers — use Porkbun via `domain-registration` only if you need DNS off Cloudflare or a TLD outside the beta.)

## Notes
- Get `ZONE_ID` via `GET /zones?name=example.com`.
- Prefer the API/CLI for build steps (deterministic); reserve the MCP for natural-language debugging/observability.
- Turnstile gotchas: **always** validate the token server-side (`POST .../siteverify` with secret+response) before processing — frontend token alone is worthless; tokens are single-use, expire in 300s; CSP must allow `https://challenges.cloudflare.com`; pair with a honeypot field. Invisible mode requires the Turnstile Privacy Addendum in your privacy policy.
- For AI-built sites the #1 real risk is NOT DDoS — it's exposed secrets (`NEXT_PUBLIC_`/`VITE_` credential-shaped vars in the bundle) and broken Supabase RLS (CVE-2025-48757: 170+ Lovable apps leaked ~13k users). The security gate must **test RLS effectiveness** (attempt a cross-tenant read), not just check policies exist. See `web-security-hardening` + `supabase-backend`.
- Pair with `domain-registration` (register the domain) and `web-security-hardening` (the layered rate-limit + anti-abuse model).
