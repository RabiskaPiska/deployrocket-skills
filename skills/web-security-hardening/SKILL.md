---
name: web-security-hardening
description: Use on EVERY client website/web-app build and before any commit or deploy — prevents SQL injection, XSS, CSRF, secret leaks, and abuse via rate limiting; runs the Semgrep + /security-review gate. Invoke for any backend/auth/forms/payment code or when hardening a site before shipping.
---

# Web security hardening (client code = liability)

One XSS/SQLi/secret-leak on a client site is a liability event. Apply this to every build; run the gate before every commit/PR/deploy. **"0 vulnerabilities" cannot be guaranteed by anyone — this is defense-in-depth that gets as close as honestly possible.**

## 1. SQL injection — eliminate, don't sanitize
- **Never** build SQL by string concatenation/interpolation. Use the Supabase client / an ORM / parameterized queries ($1,$2) ONLY.
- Defense-in-depth: **RLS enabled on every table**, policies scoped via `auth.uid()`. RLS is your backstop if app-layer authz is ever bypassed.
- Validate + type every input at the boundary (zod/valibot). Reject, don't coerce, malformed input.

## 2. XSS
- Escape all output by default (React/Vue/Svelte do this). **Never** `dangerouslySetInnerHTML` / `v-html` / `{@html}` with user or fetched data without sanitizing (DOMPurify).
- Set a strict **Content-Security-Policy** header (no `unsafe-inline` for scripts; nonce/hash).

## 3. CSRF & sessions
- Use the framework's CSRF protection for state-changing routes. Cookies: `HttpOnly`, `Secure`, `SameSite=Lax/Strict`.
- On any auth change, audit the FULL surface: token expiry, refresh-token rotation, privilege escalation, CSRF, XSS, session fixation.

## 4. Secrets
- Secrets live in env vars / secret managers ONLY — never in code, never echoed, never committed. `.env` reads are deny-listed.
- Use restricted/scoped keys (e.g. Stripe `rk_*`, Supabase publishable key client-side, never `service_role` client-side or in an agent).

## 5. Untrusted input = EVERYTHING external
Treat as hostile: user input, URL params, webhooks, **scraped/fetched content** (Firecrawl, Figma community, 3rd-party components), AND **database rows an AI agent reads** (prompt-injection vector — see `supabase-backend`). Validate/parameterize/review before use or accept.

## 6. Rate limiting (anti-abuse / anti-DoS) — layered
- **Front (Cloudflare/Vercel WAF):** coarse per-IP limits — the only layer that blocks before backend cost. Always enable on public sites + forms + auth endpoints.
- **App/edge (Upstash Redis sliding window):** per-user/IP fine limits on mutating/expensive endpoints → return 429.
- **Postgres counters** for hard per-user quotas.
- Turnstile/captcha on forms + signup to stop bots.

## 7. The scanning GATE (run, don't skip)
- **Semgrep Guardian** auto-scans every edit (SAST + SCA + secrets) → regenerate until clean.
- **`/security-review`** before every commit/PR — catches authz/business-logic flaws scanners miss. Pairs with Guardian by design.
- **Snyk MCP** for deeper dependency/license/container SCA when in scope.
- **CI gate:** the `claude-code-security-review` GitHub Action + Semgrep as **required PR checks** (see `ultra_plan/ci_security.yml`). Inline + pre-commit + CI = defense in depth.

## 8. Pre-ship checklist (must pass before deploy)
- [ ] No string-built SQL; RLS on every table; inputs validated at boundary
- [ ] Output escaped; CSP set; no unsanitized HTML injection
- [ ] CSRF protection on state-changing routes; secure cookies
- [ ] Secrets only in env/secret manager; scoped keys; nothing committed
- [ ] Rate limits at the edge + on auth/forms/expensive endpoints; bot protection on forms
- [ ] Semgrep clean + `/security-review` passed + CI security checks green
- [ ] Auth surface audited (expiry, refresh rotation, escalation, fixation)

## Honesty
Never disable a security control to make something work. If a control must be relaxed, say so and document the risk. "Zero vulnerabilities" is aspirational — ship defense-in-depth, not false guarantees.
