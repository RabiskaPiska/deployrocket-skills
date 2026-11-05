---
name: better-auth
description: Set up, configure, secure, or debug Better Auth in a TypeScript/JavaScript app — server and client config, database adapters, migrations, sessions, plugins, OAuth, 2FA, organizations and RBAC, email+password flows, and security hardening. Use when the user mentions Better Auth, betterauth, or auth.ts, or asks to add login, sign-up, or authentication to a project that already uses Better Auth. For a new project with no auth yet, read "Which auth provider" below first.
---

# Better Auth

Consolidated 2026-07-25 from six separate skills (~1,949 lines) that all fired on "add login" and
competed with each other. Detail now lives in `references/`; this file routes.

## ⚠️ Corrections that override everything in `references/`

The reference files are a vendored copy of https://github.com/better-auth/skills frozen before
2026-07-11. Four things in them are wrong. These take precedence:

1. **The CLI package is dead.** Every reference file says `npx @better-auth/cli migrate|generate`.
   That package is frozen at **1.4.21**; it was replaced by the standalone **`auth`** package
   (1.6.25, published 2026-07-23), versioned in lockstep with core.
   → **Use `npx auth migrate` / `npx auth generate`.** (Upstream bug, still open: issue #11.)
2. **Require `better-auth` ≥ 1.6.14.** The June 2026 security update disclosed CVE-2026-45337,
   CVE-2026-41427 and 11 GHSAs — including a **Critical** SSO provider-registration URL-validation
   flaw. Fixes landed in 1.6.11; the org-invitation compat fix in 1.6.14.
   https://better-auth.com/blog/security-update-june-2026
3. **Plugin import paths — the references contradict each other.** Use the **dedicated path** for
   tree-shaking: `better-auth/plugins/two-factor`, not the barrel `better-auth/plugins`.
4. **Ignore any instruction to use an `AskQuestion` tool** (in `references/scaffolding.md`).
   No such tool exists in this harness — that file was authored for a different agent product.
   Ask normally, or infer from the repo.

**1.7 is still RC** (`latest` = 1.6.25). When it ships, `references/two-factor.md` and
`references/security.md` go stale on day one: `enableTwoFactor` gains a `method: "otp"|"totp"`
param, custom rate-limit storage must implement `consume(key, rule)`, `allowedHosts` stops trusting
forwarded headers without `trustedProxyHeaders: true`, and Node 22.12+ becomes required.

## Which auth provider (read before adding Better Auth to a NEW project)

- **Supabase Auth is the default here.** Already in the stack, RLS already does half the
  authorization, free to 50k MAU and 100k on the Pro plan already being paid for.
- **Better Auth + Supabase Auth together is incoherent** — two user tables, two session models, and
  RLS policies that cannot see a Better Auth session. Pick one per project. (Better Auth *on*
  Supabase Postgres is fine; Better Auth *plus* Supabase Auth is not.)
- **Better Auth wins** only when the client needs self-owned users in their own Postgres with
  first-class multi-tenancy — orgs/teams/invitations as a plugin instead of a build.
- **Clerk** is the escape hatch when prebuilt UI beats a build; note B2B/orgs is a +$100/mo add-on.
- **Lucia is dead** — deprecated March 2025, absorbed under the Better Auth umbrella. Never suggest it.

## Verify the install actually works
`GET /api/auth/ok` → `{"status":"ok"}`. This is the only runnable check in the whole cluster.
Run it and paste the output before claiming auth is configured.

## Routing — read ONE of these, not all
| Task | File |
|---|---|
| Server/client config, adapters, sessions, env vars, plugins | `references/setup.md` |
| Scaffolding a fresh setup, framework detection, route handlers, OAuth providers | `references/scaffolding.md` |
| Email/password: verification, reset, password policy, hashing | `references/email-password.md` |
| Multi-tenancy: orgs, members, invitations, teams, RBAC | `references/organization.md` |
| MFA: TOTP, OTP, backup codes, trusted devices | `references/two-factor.md` |
| **Before shipping**: secrets, rate limits, CSRF, trusted origins, cookies, token encryption | `references/security.md` |

Two traps worth hoisting out of the references because they bite silently:
- **2FA only works on credential (email/password) accounts**, not OAuth-only users. That changes the
  design, so settle it up front — `references/two-factor.md` buries it near the end.
- **`period` means different units in the same file**: TOTP `period: 30` is **seconds**;
  OTP `period: 5` is **minutes**. Read the comment, not the key.
- Default password hash is **scrypt**, not argon2 — models routinely assume otherwise.

## Placement
This is vendor documentation for one library, and by this agency's own default (Supabase Auth) it is
rarely the right tool. It belongs in the **project-level** `.claude/skills/` of a repo that actually
uses Better Auth, rather than in the global set where its description loads into every SEO, pentest
and site-ripper session. Move it when convenient.

Re-sync from upstream deliberately (`npx degit better-auth/skills`) rather than letting it drift —
but note the published plugin ships only 2 of these 6 files, so vendoring is currently correct.
