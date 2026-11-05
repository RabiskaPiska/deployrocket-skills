---
name: domain-registration
description: Use to register a domain and wire DNS programmatically (no manual dashboard clicking, no approval waits) and tie it to a deployed site. Covers the best automatable registrars June 2026 and the full register -> DNS -> SSL -> verify pipeline. Invoke when a client project needs a domain.
---

# Domain registration (automated)

Fixes the manual pain (buying + DNS + waiting for approval). **Verdict (updated June 2026): use the Cloudflare Registrar API as the default — it's now a single-vendor buy→DNS→CDN/WAF→SSL→host pipeline with no dashboard step. Use Porkbun as the fallback when you need a TLD Cloudflare's beta doesn't support or DNS off Cloudflare. Do NOT use Namecheap in an automated workflow.**

## What changed
Cloudflare shipped a **Registrar API (beta, announced 2026-04-15)** that finally makes domain *purchase* programmatic at registry/at-cost pricing. Before this, no mainstream registrar offered clean programmatic purchase **plus** a unified edge stack — which is exactly why agencies were stuck on manual Namecheap+Cloudflare. The all-Cloudflare path is now the differentiated, fully-automated choice.

## Why each
- **Cloudflare Registrar (PRIMARY)** — buy + DNS + CDN + WAF + SSL + host from ONE vendor, every step an API call, **at registry cost** (~$8.57 .com, no markup/renewal hikes). Endpoints: `GET /accounts/{acct}/registrar/domain-search?q=`, `POST .../registrar/domain-check {"domains":[...]}`, `POST .../registrar/registrations {"domain_name":"x.com"}`, `GET .../registrations/{domain}/registration-status`. Returns 201 (done in seconds) or 202 + poll URL.
  - **Beta limits to design around:** only a SUBSET of TLDs is registrable via API (.com/.dev/.app confirmed); **renewals, transfers, and contact edits are dashboard-only for now**; premium domains need explicit fee acknowledgement. A Cloudflare-registered domain **MUST stay on Cloudflare nameservers** (full/primary DNS only — no external NS, no partial/CNAME zone). That's a feature for an all-Cloudflare agency, a blocker only if you need DNS elsewhere → then use Porkbun.
- **Porkbun (FALLBACK)** — complete programmatic lifecycle + an **official MCP** (Claude calls it natively). Use when a needed TLD is outside Cloudflare's beta or DNS must live off Cloudflare. `POST /domain/create/{domain}` **spends ACCOUNT CREDIT** (pre-fund a float or it returns `INSUFFICIENT_FUNDS`). Other: `/domain/checkDomain/{domain}`, `/domain/updateNs/{domain}`, DNS CRUD, `/ssl/retrieve/{domain}`. No IP allowlist, no eligibility gate. (~$11 .com, free WHOIS.)
- **Namecheap — AVOID for automation:** API needs per-IP IPv4 **allowlisting** (no CIDR/IPv6 → breaks on cloud/CI/serverless rotating IPs), gates prod access behind $50 balance / 20+ domains, retail pricing, dated API. This is the source of the manual friction — switching away fixes it.

## One-time setup (Cloudflare, then zero manual steps per site)
Account ID + API token with **Registrar write** + a default payment method + a default registrant contact + a signed Domain Registration Agreement. Once these 5 are set, registration is fully unattended. (Porkbun fallback: API key at porkbun.com/account/api, set registration defaults, **pre-fund balance**, wire the official Porkbun MCP with `PORKBUN_API_KEY`/`PORKBUN_SECRET_API_KEY`.)

## Primary pipeline (all-Cloudflare, each step one API call)
1. **Check:** `domain-search` + `domain-check` → availability + at-cost price.
2. **Confirm:** internally confirm name + price — **the ONE human checkpoint, because registration is non-refundable.**
3. **Register:** `POST registrar/registrations`, poll `registration-status` until `succeeded`. Zone auto-creates on CF nameservers and activates in seconds-to-minutes (CF controls the registry-side NS).
4. **Deploy:** push the build to Pages via REST (use wrangler for the upload step only).
5. **Attach domain:** `POST /accounts/{acct}/pages/projects/{project}/domains {"name":"example.com"}` — CF auto-creates the flattened apex record. ⚠️ **wrangler CANNOT add a Pages custom domain** (open bug workers-sdk#11772) — many guides wrongly say to use the dashboard; call the REST endpoint.
6. **Instant SSL:** immediately write the returned **DCV records** into the CF zone so Universal SSL issues right away instead of waiting up to 24h. Guard two failure modes: a CAA record that doesn't authorize the CA blocks issuance; a Worker/Access rule blocking `/.well-known/acme-challenge` blocks validation.
7. **Harden:** `PATCH` zone settings `always_use_https=on`, `ssl=full` (strict once origin verified); apply the edge security baseline via the Rulesets API (see `cloudflare-automation`). Add exponential backoff (CF token limit ~1,200 req / 5 min).

## Fallback pipeline (Porkbun)
Pre-fund balance → `POST /domain/create/{domain}` (handle `INSUFFICIENT_FUNDS`) → `/domain/updateNs` to point at Cloudflare (or manage Porkbun DNS directly) → continue the Cloudflare edge + deploy steps above. Send an Idempotency-Key; `/domain/create` is the single confirm-before-spend step.
