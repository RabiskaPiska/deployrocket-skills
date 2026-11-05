---
name: joomla-playbook
description: "Defensive-security playbook for Joomla CMS (3.x-6.x) core and extension ecosystem. Covers the single front-controller routing model (index.php?option=com_...), the group/ACL ladder and Super User (core.admin) escalation target, #__ tables with md5:salt vs bcrypt vs argon2 hash formats, injection classes with 2024-2026 CVEs (CVE-2023-23752 webservice config disclosure - CISA KEV, CVE-2024-21726 InputFilter XSS-to-RCE, CVE-2025-22207 com_scheduler SQLi, 2026 MFA-bypass pair CVE-2026-92227/73337, Helix3 + JoomCCK unauth RCE), exfil via configuration.php disclosure, and RCE chains. USE WHEN: assessing or hardening a Joomla site, auditing extensions, or triaging Joomla CVEs. Every technique is paired with Detection + Hardening; verified live 2026-10-29 against the Joomla Security Centre and NVD."
license: MIT
tags: [joomla, cms, defensive-security, hardening, detection, xss, sqli, mfa-bypass, cve-currency]
updated: 2026-10-29
---

# Joomla — Attack & Defense Playbook (Condensed)

Joomla routes almost everything through one front controller (`index.php?option=com_<name>`), so the
surface is component-dispatch plus template/media/installer primitives reserved for Super Users.
This condensed skill keeps the pairing rule: **every offensive technique is paired with Detection
and Hardening**. All advisory IDs verified live 2026-10-29 from the Joomla Security Centre / NVD;
unconfirmed items flagged, never invented.

## 1. Attack Surface Map

- Frontend components: `option=com_content|com_users|com_fields|com_tags|com_finder` — reachable pre-auth (historical com_fields SQLi = CVE-2017-8917).
- Backend `/administrator/`; webservices `/api/index.php/v1/<endpoint>` (per-plugin disableable, plg_webservices_*).
- Templates `/templates/<name>/` + `/administrator/templates/` — Super-User PHP editing = de-facto RCE surface.
- Media store `/images/` + com_media; update component com_joomlaupdate; `com_ajax` plugin dispatch (Helix3 2026 mass-defacement vector); `/installation/` (delete post-install).
- Fingerprinting: generator meta, `language/en-GB/en-GB.xml`, `/administrator/manifests/files/joomla.xml`, `/media/jui/` layout, `/api/index.php/v1/config/application` 200 = J4+ webservices on.
- **Detection:** high-frequency `option=com_` probes with 404-heavy distributions (JoomScan pattern); HEAD/GET sweeps of `/api/index.php/v1/`; rate-based alerting beats UA matching. **Hardening:** suppress generator meta; deny web access to manifests/language XML; disable unused webservices plugins.

## 2. Auth & Privilege Classes

- Group ladder: Public < Registered < Author < Editor < Publisher < Manager < Administrator (7) < Super Users (8, `core.admin`). ACL = `#__usergroups` nested-set + `#__assets` rules tree walked by `JUser::authorise()`.
- Escalation ladder: Author (stored-XSS foothold) → Editor → Publisher (content goes live) → Manager (site-wide content poisoning, no installer) → Administrator (no `core.admin`) → **reaching `core.admin` unlocks installer/template/config = RCE**.
- 2026 ACL-gap primitives (fixed 5.4.8/6.1.3 or 5.4.9/6.1.4): CVE-2026-92226, CVE-2026-90913, CVE-2026-71574 (webservice edit/mutation ACL gaps), CVE-2026-92223, CVE-2026-90907 (guest account creation via `profile.save`).
- Sessions: DB handler default (`#__session`); MFA built into J4+ (TOTP/WebAuthn).
- **Detection:** Action Logs on group changes; `#__user_usergroups_map` inserts touching group 8; super-user logins from new IP/geo; canary super-user row. **Hardening:** mandatory MFA for Super Users **and patch to 5.4.9/6.1.4** — 2026 shipped two MFA bypasses (CVE-2026-92227 remember-me-before-MFA; CVE-2026-73337 state checks); restrict Author+ account creation.

## 3. Database & Hash Formats (condensed)

| Table | Attacker value |
|---|---|
| `#__users` | password hashes; MFA secret material (otpKey/otep, encrypted) |
| `#__user_usergroups_map` | fastest super-user (group 8) enumeration after a DB read |
| `#__extensions` | component config + exact versions of disabled-but-present vulnerable extensions |
| `#__assets` | ACL rules tree — tamper target after write access |
| `#__session` | active sessions when DB handler used |
| `#__action_logs` | audit records — blue-team gold (assume readable post-compromise; ship off-box) |

| Version | Format | Notes |
|---|---|---|
| 1.5–3.1 | `md5(password+salt):<32-hex-salt>` | cheap offline crack |
| 3.2–3.10 | bcrypt `$2y$10$…` (legacy `$P$`/`{SHA256}` still verified + rehashed at login) | commit 01f2d71 |
| 4.0–5.x | argon2id/argon2i/bcrypt map; md5+phpass deprecated since 5.0; transparent rehash on login | `UserHelper::HASH_ALGORITHMS` |

- **Detection:** `#__action_logs` user-record changes; canary super-user. **Hardening:** least-privilege DB user; current patch level so J4+ auto-rehashes legacy hashes.

## 4. Injection Classes — Top CVEs

- **CVE-2023-23752 (CISA KEV, added 2024-01-08):** improper access check — `?public=true` on webservice endpoints leaks the full user list AND `config/application` with **DB credentials in plaintext**. 4.0.0–4.2.7 → fix 4.2.8. NVD 5.3; PoC https://github.com/Acceis/exploit-CVE-2023-23752; Metasploit `joomla_api_improper_access_checks`; nuclei template. — https://developer.joomla.org/security-centre/894-20230201-core-improper-access-check-in-webservice-endpoints.html
- **CVE-2024-21726 (CVSS 6.5):** InputFilter XSS — mb_strpos/substr UTF-8 mismatch lets `<` survive `cleanTags()`; documented chain to admin template PHP write = RCE. Fix 3.10.15-elts/4.4.3/5.0.3. — https://www.sonarsource.com/blog/joomla-multiple-xss-vulnerabilities/
- **CVE-2025-22207:** com_scheduler backend SQLi (improperly built order clauses) → 4.4.11/5.2.4. **CVE-2025-25226:** framework `quoteNameStr()` SQLi → 2.2.0/3.4.0.
- **2026 batch (all → 5.4.8/6.1.3 or 5.4.9/6.1.4):** CVE-2026-73373 (SHTML upload → code exec on SSI servers), CVE-2026-73337 + CVE-2026-92227 (MFA bypasses), CVE-2026-73336 (schema.org XSS), CVE-2026-71573 (CORS), CVE-2026-90915 (cache-purge path traversal), CVE-2026-92222 (SSRF), CVE-2026-92231/92232 (InputFilter bypasses), plus the ACL gaps in §2.
- Extension class: CVE-2025-26855 (Articles Calendar unauth SQLi, 9.8), CVE-2025-22211 (JoomShopping), CVE-2025-54474 (DJ-Classifieds); Helix3 unauth file-write/delete campaign (~50 sites/48h, July 2026, fix 3.1.2); JoomCCK unauth `template.save` PHP-write RCE (GHSA-whf2-8p4w-v6xv).
- Honest note: no core deserialization advisories found 2024–2026 — treat "core deserialization RCE" claims as unproven until cited.
- **Detection:** `?public=true` log signatures; `list[`/`filter[`/`order` payloads on list views; `on*=`/`javascript:` scans of content tables; unauthenticated `com_ajax` POSTs; `../` in task/cache params. **Hardening:** current patch level; PHP 8.3+ (mbstring fix not backported); CSP headers; never grant raw/PHP filters below Super User.

## 5. Exfil Paths

- **SQLi → `#__users`/`#__session`/`#__extensions`** (extension params often hold API keys). *Detection:* WAF + anomalous query patterns; canary row. *Hardening:* least-priv DB user.
- **CVE-2023-23752 config disclosure** — two unauth GETs return DB creds + mail/captcha secrets; the cheapest exfil in the J4 landscape. *Detection:* the `?public=true` signature. *Hardening:* patch; **rotate DB creds if ever exposed unpatched** (KEV = assume prior exploitation).
- **configuration.php read** (`$host/$user/$password/$db/$secret`) — treat any config-disclosure incident as full DB compromise. *Detection:* FIM on configuration.php; alerts on `configuration.php*`, `*.bak`, `.swp` requests. *Hardening:* perms 600/640; backups out of webroot.
- **Admin exports & Akeeba Backup** (`.jpa` archives; default output inside site tree). *Detection:* backup events + large archive downloads. *Hardening:* off-webroot/remote storage; restrict downloads; encrypt.
- **Audit-log inverse-read:** assume `#__action_logs` is readable post-compromise — only off-box shipping keeps it useful.

## 6. RCE Chains

- **Chain A — template file editing:** stolen Super User session → `<?php` in template override (CVE-2024-21726 chain documented by Sonar). *Detection:* template FIM + Action Logs. *Hardening:* minimal Super Users; git/CI deploys; PR #48171 path checks.
- **Chain B — extension install upload:** com_installer runs package `script.php` (core intentionally allows unsafe uploads, PR #48038). *Hardening:* vetted packages only; disable URL-install.
- **Chain C — media upload bypass:** CVE-2026-73373 (.shtml) + June-2026 extension wave (unauth upload → PHP). **Single highest-value control: serve `/images/` static, no PHP handler.**
- **Chain D — XSS → admin click → template PHP write** (CVE-2024-21726 end-to-end). *Hardening:* patch + CSP + Chain-A controls.
- **Chain E — extension RCEs:** Helix3 (com_ajax, unauth, 4 primitives) and JoomCCK (`template.save` → `views/*/tmpl/*.php`). *Detection:* unauth `com_ajax` POSTs; FIM on tmpl dirs. *Hardening:* Helix3 ≥3.1.2, JoomCCK fixed release; WAF-block unauth com_ajax `save`/`import`/`remove`.

## 7. Detection Essentials

- **Action Logs** (System → User Actions Log): logins, group changes, installs, template/config/content changes; email alerts to a monitored mailbox; treat unfamiliar-IP super-user logins as incident input.
- **Web-log signatures:** `GET /api/index.php/v1/config/application?public=true` · `/v1/users?public=true` · distinct `option=com_` at high rate · `POST index.php?option=com_ajax` without session cookie · `option=com_templates&task=*.save` · `../` in task params · `configuration.php*` fetches.
- **WAF (official CVE-2023-23752 mitigations):** mod_rewrite `RewriteCond %{QUERY_STRING} public=` + `RewriteRule api/?. - [F]`; mod_security rule pair on `api/index.php/v1/` + `ARGS_GET_NAMES ^public$`.
- **Canary super-user row** (`svc-audit-admin`, group 8, unused) — any login/enumeration of it is a direct compromise signal.
- **FIM + DB watch:** hash-scan `/templates/`, `/images/`, `/components/`; log group-8 map inserts and new `#__extensions` rows.

## 8. Hardening Checklist

| # | Control | Action |
|---|---|---|
| 1 | Patch level | **5.4.9 / 6.1.4** (Sept 2026 batch incl. both MFA bypasses); 4.x legacy — migrate |
| 2 | Webservices | disable plg_webservices_* not in use; edge-block `/api/` if no integration |
| 3 | Template editing | git/CI deploys; minimal Super Users; FIM on template dirs |
| 4 | Upload allowlist | minimal extensions in Global Config → Media; **add SHTML awareness post-73373**; no PHP handler on `/images/` |
| 5 | DB user | one-site schema only; no FILE/PROCESS; rotate after any config disclosure |
| 6 | configuration.php | 600/640; no webroot backups/swap copies |
| 7 | MFA | built-in TOTP/WebAuthn mandatory for group 8 — patched version required (2 known bypasses in 2026) |
| 8 | Action Logs | enabled + email notifications + off-box shipping |
| 9 | Install hygiene | delete `/installation/`; vet vendors via the Vulnerable Extensions List (extensions.joomla.org/vulnerable-extensions/vulnerable) |
| 10 | CORS | configure origins correctly if webservices used (CVE-2026-71573) |

## References

- `references/full-playbook.md` — complete source playbook (full CVE table, chains, honesty ledger).
- `references/cve-currency.md` — Joomla rows from the cross-CMS CVE master table (verified 2026-10-29).
- `references/tooling.md` — JoomScan / droopescan tooling with licenses, staleness notes and log footprints.
- CVE-2023-23752 advisory: https://developer.joomla.org/security-centre/894-20230201-core-improper-access-check-in-webservice-endpoints.html
- PoC: https://github.com/Acceis/exploit-CVE-2023-23752
- Security Centre (primary feed): https://developer.joomla.org/security-centre.html
- JoomCCK advisory: https://github.com/JoomCoder-com/JoomCCK/security/advisories/GHSA-whf2-8p4w-v6xv
- Scanner: https://github.com/OWASP/joomscan

## Appendix — Quick Signatures

```regex
/api/index\.php/v1/(config/application|users)\?public=true   # CVE-2023-23752 (KEV) exploit/scan
option=com_[a-z0-9_]+                                        # high-rate + 404-heavy = component enum
POST index\.php\?option=com_ajax                             # unauth (no prior cookie) = Helix3-class abuse
option=com_installer.*install\.install                       # stolen-session installer use (off-window)
option=com_templates&task=.*\.save                           # template PHP-write chain (A/D)
[?&](option|task|cg).*(\.\./)                                # cache-purge traversal (CVE-2026-90915)
(configuration\.php|\.bak|\.save|\.swp)$                      # config-prize probing
list\[|filter\[|order=.*('|\%27)                             # extension ORDER BY SQLi probes
```
