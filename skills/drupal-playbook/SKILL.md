---
name: drupal-playbook
description: "Defensive-security playbook for Drupal core (D7 through D11) built around the Drupalgeddon lineage. Covers the route map (/user/login, /user/register, /jsonapi, /rest/*, /update.php, /core/install.php), role/permission escalation and uid 1, core tables with \$S\$ stretched-SHA512 vs D10.1+ bcrypt hash formats, injection classes with verified CVEs (Drupalgeddon1 CVE-2014-3704, Drupalgeddon2 CVE-2018-7600, Drupalgeddon3 CVE-2018-7602 - all CISA KEV, CVE-2026-9082 PostgreSQL SQLi KEV, CVE-2026-108097 Field UI RCE), exfil paths, and RCE chains including render-array #post_render injection and Twig SSTI. USE WHEN: assessing or hardening a Drupal site, writing WAF/Sigma-style detection rules, or responding to a suspected Drupalgeddon-pattern compromise. Every technique is paired with Detection + Hardening; verified live 2026-10-29 on drupal.org/NVD."
license: MIT
tags: [drupal, cms, defensive-security, hardening, detection, drupalgeddon, rce, sqli, jsonapi, cve-currency]
updated: 2026-10-29
---

# Drupal — Attack & Defense Playbook (Condensed)

Drupal's security story is defined by the Drupalgeddon lineage: request-array keys reaching query
builders (SQLi) and render arrays (`#post_render`) reaching `call_user_func` (RCE) — plus modern
JSON:API over-exposure and PHP object injection. Every technique below is paired with **Detection**
and **Hardening**; all advisory IDs verified live 2026-10-29 from drupal.org / NVD / CISA-KEV sources.

## 1. Attack Surface Map

- `/user/login` — brute force + Drupalgeddon1 POST target (flood table throttles); `/user/register` — Drupalgeddon2 classic PoC endpoint; `/user/pass` — reset-link probing.
- `/jsonapi` — default-on since D8.7.7; bulk content/user enumeration when anonymous collections are permissive.
- `/rest/*` + `_format=hal_json` — over-permissive REST resources (Drupalgeddon2-era vectors; hal_json deprecated in D10).
- `/update.php` (unauth on misconfig runs schema updates), `/core/install.php` (re-install hijack on partially built sites), `/cron/<key>` (forced-cron DoS).
- File uploads → `sites/default/files` (SVG/HTML XSS, PHP if handler misconfigured); `sites/default/settings.php` = DB creds + `hash_salt` (post-RCE prize).
- Fingerprinting: `/core/CHANGELOG.txt` (verified live in D11 tree — a recon gift; consider blocking), generator meta, `/core/misc/*` vs `/misc/*.js` (D8+ vs D7), `X-Drupal-Cache` headers; droopescan automates all of it.
- **Detection:** 404-heavy bursts on `/modules/*`, `/themes/*`, `*.info.yml`; `CHANGELOG.txt` fetch storms. **Hardening:** no directory indexes; trim CHANGELOG exposure; keep an asset inventory.

## 2. Auth & Privilege Classes

- Ladder: anonymous → authenticated → editor/publisher roles → administrator; **uid 1 bypasses all permission checks**.
- Privilege currency = permission machine names in `role_permission`; any role with `administer permissions` can grant itself `administer users` → reset uid 1's password via one-time link.
- One-time login links: HMAC of user data with `settings.php` `hash_salt` — DB read + hash_salt = link minting (post-exfil path).
- Sessions: Symfony-based (D8+), `sessions` table; cookies HTTP-only by default; `SESS<hash>=` name leaks D7 vs D8+.
- Flood control (`flood` table: 50 failed/hour per IP, 5 per account D7 defaults) — free rate limiting; do not disable.
- **Detection:** watchdog `type=user` login failures; one-time-link usage from new IPs; sudden role edits. **Hardening:** uid 1 long random password, rarely used; `tfa` module; monitor `role_permission` drift; keep flood limits.

## 3. Database & Hash Formats (condensed)

| Table | Security relevance |
|---|---|
| `users_field_data` (D8+) / `users` (D7) | password-hash harvesting; PII |
| `sessions` | sid row theft = impersonation; admin session = full site |
| `flood` | rate-limit state (attacker can flood a victim IP = lockout DoS) |
| `watchdog` (dblog) | IR goldmine — attacker can clear it post-auth; ship off-box |
| `key_value` (+`_expire`) | state/KV store — module secrets and update/cron state |
| `config` (D8+) | exported config; may embed mis-stored secrets |
| `role_permission`, `users_roles` | privilege-escalation audit trail |

| Era | Hash format | Cracking |
|---|---|---|
| D7–D10.0 | `$S$` stretched SHA-512 (phppass): `SHA512(hash . password)` × 2^count, 8-char salt (chars 4–12), default 15 rounds = 32,768 iterations, 55 chars total | bcrypt-like cost; long passphrases win |
| D10.1+ | PHP native `password_hash()` bcrypt cost 10 (change record drupal.org/node/3322420); `$S$` retained via compatibility layer | standard bcrypt tooling (`$2y$`) |
| Legacy | `U$` prefix = D6 md5 awaiting rehash; `$P$`/`$H$` = imported phpass | cheap targets — rehash on next login |

- **Detection:** query logs with `;UPDATE users` patterns; sudden uid-1 `pass` change. **Hardening:** DB user without FILE privilege; alert on users-table writes from the web user.

## 4. Injection Classes — Top CVEs

- **Drupalgeddon1 — SA-CORE-2014-005 / CVE-2014-3704 (CVSS2 7.5, EPSS 94%):** unauth SQLi via `expandArguments()` array **keys** — POST `name[0;UPDATE users SET pass=... WHERE uid=1;#]=x` to `/user/login`. Mass-exploited within hours. — https://www.drupal.org/sa-core-2014-005
- **Drupalgeddon2 — SA-CORE-2018-002 / CVE-2018-7600 (CVSS 9.8, KEV):** Forms API kept `#`-prefixed render-array keys (`#post_render`, `#markup`) → `call_user_func` on render. PoC: `POST /user/register?element_parents=account/mail/%23value` with `mail[#post_render][]=exec&mail[#children]=id`. Metasploit `drupal_drupalgeddon2`; F5 ASM signature 200004440. — https://www.drupal.org/sa-core-2018-002 · https://github.com/g0tmi1k/drupalgeddon2
- **Drupalgeddon3 — SA-CORE-2018-004 / CVE-2018-7602 (CVSS 9.8, KEV):** same subsystem via URL-path parameters; in-the-wild within hours. (Naming note: Drupalgeddon3 = SA-CORE-2018-004, **not** -006/-009.) — https://www.drupal.org/sa-core-2018-004
- **CVE-2019-6340 (KEV):** RCE via REST/JSON:API hal_json deserialization when `rest`+`hal` enabled with GET/PATCH.
- **SA-CORE-2026-004 / CVE-2026-9082 (KEV 2026-05-22):** highly critical unauth SQLi, **PostgreSQL-only**; exploit attempts in the wild; fixed in 2026-05-20 releases (11.3/11.2/10.6/10.5). — https://www.drupal.org/security/core/advisory
- **SA-CORE-2026-014 / CVE-2026-108097:** critical RCE in Field UI; at least one unauth exploit path; mitigation: uninstall `field_ui` until patched.
- **CVE-2025-31674:** PHP object injection → file inclusion → possible RCE via chaining — the modern POI example in core.
- 2026 XSS set: CVE-2026-6365 (jQuery AJAX modal, critical), CVE-2026-15917 (HTMX attributes), CVE-2026-108514 (Navigation module).
- **Detection:** requests containing `element_parents=` AND `[#post_render]`/`[#children]`/`#markup`; `name[0;` array-key SQLi on `/user/login`; anonymous `/admin/structure/types/manage/*/fields` probing (Field UI). **Hardening:** current core; never re-trust request arrays (`RequestSanitizer` strips `#` keys); disable `rest`/`hal` unless needed; `jsonapi_extras` field filtering.

## 5. Exfil Paths

| Path | Trigger | Detection | Hardening |
|---|---|---|---|
| SQLi extraction | Drupalgeddon1-style array-key injection | `;UPDATE users` in query logs; watchdog user events | core ≥7.32; no FILE privilege |
| REST/JSON:API bulk GET | `/jsonapi/node/<type>?fields[...]&filter[...]` walking | high-volume `/jsonapi` 200s from one IP | anonymous collection perms off; field filtering; rate limit |
| Views/Webform exports | post-auth CSV exports | watchdog export events; CSV volume | restrict export perms; DLP on admin paths |
| `drush sql-dump` | post-auth shell primitive | shell-history audit; stray `.sql.gz` in files dir | least-priv DB user; dumps outside webroot |
| `key_value`/state secrets | SQLi or post-auth Devel/PHP read | slow-query log referencing `key_value` | keep secrets in a vault, not state/config |
| Private files | guessed `system/files/{path}` URLs | access-log bursts on `/system/files/` | private dir outside webroot; per-file checks |

## 6. RCE Chains

1. **Drupalgeddon2 end-to-end (verified):** unauth POST to any Form API AJAX endpoint → `#post_render` render array → `Renderer::render()` → `call_user_func` → PHP webshell write (Metasploit-automated). *Detection:* the `element_parents`+`#post_render` signature. *Hardening:* patch chain 7.58→7.59; RequestSanitizer lesson.
2. **PHP object injection (CVE-2019-6340, CVE-2025-31674 class):** serialized objects via REST/hal_json; gadget-dependent (contrib libs). *Detection:* `O:\d+:` markers in request bodies. *Hardening:* disable rest/hal when unused; `composer audit`.
3. **Twig SSTI:** contrib modules treating user content as Twig (Webform, Inline Formatter Field, etc.); `{{ ['id']|filter('system') }}`-class payloads when sandbox bypassed. *Detection:* `{{`/`{%` in field values. *Hardening:* Twig sandbox for user input; no Twig processing for editor roles.
4. **Module/theme upload as admin:** Update Manager tar.gz upload → webshell module. *Detection:* watchdog install events; FIM on `modules/contrib`. *Hardening:* disable Update Manager upload in prod; Composer/CI only.
5. **PHP filter module (documented feature, not CVE):** `use PHP for settings` role → `<?php` in block/node = instant persistence RCE. *Hardening:* disable PHP filter entirely — standing Drupal security-team advice.
6. **Drush/Composer post-access:** `drush upwd`, `drush user:login`, settings.php retarget — protect CLI access (SSH keys, no shared web-user shell).

## 7. Detection Essentials

- **watchdog/dblog** → ship to syslog/SIEM (DB-only retention is a mistake — attacker with SQLi clears it).
- **Web-log signatures (verified):** `element_parents=` + `[#post_render]` (F5 ASM 200004440) · `name[0;` on `/user/login` · `/CHANGELOG.txt` + `/core/CHANGELOG.txt` recon bursts · `_format=hal_json` probes.
- **Sigma honesty note:** the current SigmaHQ master tree (checked 2026-10-29, full recursive scan) contains **no** Drupal/Drupalgeddon rule — port the F5/Unit42 signatures yourself.
- **Canary admin row** (decoy, high uid, never used) — any login or one-time-link generation for it = attacker action; pair with watchdog alert on that uid.
- **FIM + JSON:API watch:** AIDE/osquery hashes on `modules/`, `themes/`, `settings.php`; alert on >N distinct `/jsonapi/*` collections per IP per 5 min and on anonymous `/jsonapi/user/user`.

## 8. Hardening Checklist

| # | Control | Action |
|---|---|---|
| 1 | Patch discipline | highly-critical SA = patch same day (public exploits within hours); subscribe to drupal.org security newsletter + PSA pre-announcements |
| 2 | Version floor | D10.1+ (bcrypt) or current 10.6/11.x with 2026-05-20 + 2026-10-14 releases; D7 is EOL — migrate or HeroDevs-style NES |
| 3 | PHP filter | disabled in production; audit `filter` table |
| 4 | REST/JSON:API | `rest`/`hal` off unless required; per-resource perms; `jsonapi_extras`; anonymous-surface role-playthrough each release |
| 5 | update.php / install | IP-restrict at webserver + permission gate; block `/core/install.php` post-deploy |
| 6 | trusted hosts | `$settings['trusted_host_patterns']` set; `update_free_access` FALSE |
| 7 | DB user | Drupal-schema-only, no FILE/cross-DB; separate backups user; weigh MySQL vs PG per CVE-2026-9082 |
| 8 | Drush/CLI | no drush as web user; uploads dir not writable-and-executable; serve files from bucket/CDN |
| 9 | 2FA | `tfa` module (TOTP) for admin/editor; verify your 2FA module's own advisory history |
| 10 | Flood | keep limits; monitor attacker-induced lockout DoS |
| 11 | Self-audit | quarterly droopescan against your own site — see what an attacker's first five minutes see |
| 12 | Incident-ready | off-box syslog, canary row, FIM baseline, documented "assume-RCE" runbook (patch → sweep `#post_render` IoCs → rotate hash_salt + DB creds) |

## References

- `references/full-playbook.md` — complete source playbook (hash-format detail, PoC syntax, cross-references).
- `references/cve-currency.md` — Drupal rows from the cross-CMS CVE master table (verified 2026-10-29).
- `references/tooling.md` — droopescan / Drush / Drupwn tooling with licenses, staleness notes and log footprints.
- Drupalgeddon2 advisory: https://www.drupal.org/sa-core-2018-002 · PoC: https://github.com/g0tmi1k/drupalgeddon2
- Security index: https://www.drupal.org/security
- SA-CORE-2026-004 (PostgreSQL SQLi): https://www.drupal.org/security/core/advisory
- D10.1 password hashing change: https://www.drupal.org/node/3322420
- Scanner: https://github.com/SamJoan/droopescan

## Appendix — Quick Signatures

```regex
element_parents=.*( %23 |#)(post_render|children|markup)     # Drupalgeddon2/3 (CVE-2018-7600/7602)
name\[0[;#]                                                   # Drupalgeddon1 array-key SQLi
/(core/)?CHANGELOG\.(txt|md)                                  # version recon (droopescan)
/jsonapi/(node|user)/                                         # anonymous collection enumeration
_format=hal_json                                              # REST/hal probing (CVE-2019-6340 class)
/admin/structure/types/manage/.*/fields                       # Field UI probing (CVE-2026-108097)
O:\d+:                                                        # PHP object injection (CVE-2025-31674 class)
\{\{.*\|\s*filter\(                                           # Twig SSTI payloads in content
```
