---
name: prestashop-playbook
description: "Defensive-security playbook for PrestaShop core and its module ecosystem. Covers the attack-surface ranking (module front controllers are the weakest surface, back-office /adminXXX, webservice API, Smarty templates), employee-vs-customer privilege model and _COOKIE_KEY_-MAC'd sessions, ps_ database layout with legacy MD5(salt+pw) vs bcrypt hashes, injection classes with 2024-2026 CVEs (GHSA-whxq-pxj5-qq7v core filter SQLi, CVE-2025-69633, CVE-2026-54159 faceted-search object injection CVSS 10.0, CVE-2025-61922 ps_checkout takeover), exfil via webservice keys, and RCE chains. USE WHEN: assessing or hardening a PrestaShop store, auditing modules, monitoring Friends-Of-Presta advisories, or writing detection rules. Every technique is paired with Detection + Hardening; verified live 2026-10-29."
license: MIT
tags: [prestashop, cms, ecommerce, defensive-security, hardening, detection, sqli, deserialization, cve-currency]
updated: 2026-10-29
---

# PrestaShop — Attack & Defense Playbook (Condensed)

PrestaShop's real-world breach surface is dominated by **third-party module front controllers**, not
the core. This condensed skill keeps the pairing rule: every offensive technique below carries a
**Detection** signal and a **Hardening** fix. Sources: PrestaShop core security advisories,
Friends-Of-Presta, NVD/GHSA, Patchstack/prestashield — all verified live 2026-10-29.

## 1. Attack Surface Map (ranked by historical compromise rate)

1. **Module AJAX/front controllers** — `/module/<m>/<c>`, `?fc=module&module=X&controller=Y`; most FOP advisories land here (unauth SQLi, PHP injection).
2. **Back office** `/adminXXX/` — core list-filter SQLi (GHSA-whxq-pxj5-qq7v) + module-upload RCE; "obscurity" of the dir name is the only gate.
3. **Webservice API** `/api/...` — XML CRUD; keys often full-CRUD and never rotated.
4. **Smarty templates** — SSTI if an attacker can write a `.tpl`.
5. **Image/attachment manager** — `/img/` PHP-handler misconfig → upload RCE.
6. **Import/SQL Manager** — `AdminRequestSql` = post-auth arbitrary SELECT by design.
7. **Faceted search** `?q=` params — cached/serialized values (CVE-2026-54159).
- **Fingerprinting:** `composer.json`, `CHANGELOG.md`, BO footer, `app/config/parameters.php`, `cache/class_index.php`; module listing via `/modules/` autoindex.
- **Detection:** access-log lines for `composer.json`, `/config/settings.inc.php`, `/modules/` listing, logo.png sweeps.
- **Hardening:** block web access to `composer.*`, `app/config`, `var/`, `cache/`, `tests/`; `Options -Indexes`; delete unused modules from disk.

## 2. Auth & Privilege

- Two parallel identities: `ps_employee` (BO, `ps_access` matrix: profile × tab × CRUD) vs `ps_customer` (front, own orders only).
- BO cookie = serialized object MAC'd with `_COOKIE_KEY_` (config file) — a leak forges **both** employee and customer cookies ⇒ full auth bypass. Legacy 1.6 derived password hashes from the same key.
- CSRF: per-page `token` on admin URLs; module BO pages skipping `Tools::getToken()` are the classic gap.
- Design rule: no customer row may yield BO rights — every real escalation is a bug (broken `isLoggedBack()`, admin-ish front controllers, or low-priv employees reading past `ps_access`, as in GHSA-whxq-pxj5-qq7v which exposed employee hashes to the most restricted profile).
- **Detection:** `ps_log` failed logins; surge of valid logins from a new ASN after a settings-file leak; `ps_employee` rows created outside change windows.
- **Hardening:** protect/rotate `_COOKIE_KEY_` (rotation = planned re-login-all); quarterly `ps_access` review; Logistics/Sales profiles get **no** `AdminModules`/`AdminRequestSql`; HTTPS + HSTS.

## 3. Database & Hash Formats (condensed)

| Table | Attacker value |
|---|---|
| `ps_employee` / `ps_access` | BO hashes + the whole permission matrix |
| `ps_customer` / `ps_address` / `ps_orders` | full PII join chain (identity + address + spend) |
| `ps_configuration` | cookie keys, SMTP creds, module secrets — prime SQLi exfil target |
| `ps_webservice_account` | API keys → immediate `/api/` dump |
| `ps_customer_message` | unauth-input-at-rest (stored-XSS/PII sink) |

| Version | Hash format | Cracking |
|---|---|---|
| ≤1.6 legacy | `MD5(_COOKIE_KEY_ . pw)` — salt is the global `_COOKIE_KEY_` | plain md5 speed once key known (`-m 0` with prepended salt) |
| 1.7/8/9 | bcrypt `$2y$10$…` via `password_hash()` | `hashcat -m 3200` |
| 1.7 migration edge | legacy md5 kept until next login, then auto-upgraded | detect by length (32 hex vs 60 chars) |

- **Post-incident SQLi exfil mapping:** `ps_employee.passwd`; customer⨝address⨝orders; `ps_configuration WHERE name LIKE '%KEY%'`; `ps_webservice_account`.
- **Detection:** slow-log / anomalous SELECT volume on those tables; canary rows. **Hardening:** least-privilege DB user; non-default `_DB_PREFIX_`.

## 4. Injection Classes — Top CVEs

- **GHSA-whxq-pxj5-qq7v (core, CVSS 6.5, no CVE):** back-office list-filter SQLi; logged-in low-priv employee reads whole DB incl. employee hashes. Fixed 9.1.5 / 8.2.8 (2026-08-18); 1.7.x EOL unpatched. — https://github.com/PrestaShop/PrestaShop/security/advisories/GHSA-whxq-pxj5-qq7v
- **CVE-2025-69633 / GHSA-7vwv-5gmf-fwq5 (advancedpopupcreator <1.2.7, CVSS 9.8):** unauth SQLi via `fromController` — the dominant module class. — https://security.friendsofpresta.org/modules/2026/02/16/advancedpopupcreator-cve-2025-69633.html
- **CVE-2026-54159 / GHSA-m5f5-28qr-9g9r (ps_facetedsearch 3.0.0–4.0.3, CVSS 10.0):** **PHP object injection, NOT SQLi** — slider value cached, raw `unserialize()` in `Block.php::getFromCache()` → Monolog gadget → webshell in module dir. Fixed 4.0.4 (commit 9ca839f). — https://github.com/PrestaShop/ps_facetedsearch/security/advisories/GHSA-m5f5-28qr-9g9r
- **CVE-2025-25692 / GHSA-wf46-fx4c-gx3r (core 8.2.0 only, CVSS 6.5):** PHAR deserialization in `_getHeaders` → code exec; no isolated vendor patch.
- **CVE-2025-61922 / GHSA-54hq-mf6h-48xh (ps_checkout ≥1.3.0, CVSS 9.1):** missing validation on Express Checkout → silent login → zero-click customer account takeover. Fixed 4.4.1 / 5.0.5 (+ siblings CVE-2025-61923/61924).
- **CVE-2024-36679 (livechatpro ≤8.4.0, CVSS 10.0):** guest-reachable PHP code injection — the arbitrary-PHP-in-module class.
- **Detection:** `filter_`/`orderby`/`sortorder` metachars in BO logs; `O:\d+:`/`Monolog\` in search URLs; `phar://` in requests; **unexpected `.php` files under `modules/ps_facetedsearch/`** (official IR wording); DB-error bursts in `ps_log`.
- **Hardening:** core ≥9.1.5/8.2.8; `ps_facetedsearch ≥4.0.4`, `ps_checkout ≥5.0.5`, `advancedpopupcreator ≥1.2.7`, `livechatpro >8.4.0`; PHP `disable_functions` (official list) on staging-tested config.

## 5. Exfil Paths

| Path | Mechanism | Detection | Hardening |
|---|---|---|---|
| SQLi extraction | §4 injections → hashes, PII, `ps_configuration` secrets | slow queries, `UNION SELECT` in logs | least-priv DB user, WAF, patch |
| Webservice API dump | valid key → `GET /api/customers?display=full` | per-`key=` access-log volume (usage not logged in-app) | least-priv keys, quarterly rotation, IP-restrict `/api/` |
| BO CSV/XML exports | employee grid exports (widened by §4.1 filter bugs) | CSV content-type responses per session | least-priv profiles; session logs |
| PII joins | direct SQL or API — monetizable payload | canary customer rows surfacing in spam | canary rows with unique emails/phones |
| `ps_configuration` secrets | SQLi or config read → SMTP/cookie keys → pivot | quarterly config review; daily diff | keep secrets out of config table; rotate after suspected read |

## 6. RCE Chains (ordered by attacker cost)

1. **Module upload (BO `AdminModules`) → PHP in `modules/<name>/`** — nulled-module ZIPs are a documented backdoor vector. *Detection:* new module dirs outside change windows. *Hardening:* `AdminModules` for SuperAdmin only; ZIP review.
2. **Smarty SSTI via template editing** — theme-edit rights = code-edit rights; modern Smarty 4/5 removed `{php}` but constant access/modifier abuse remains. *Hardening:* treat theme edit like SuperAdmin; keep Smarty current.
3. **PHAR / object-injection chains** — CVE-2025-25692 + CVE-2026-54159 (both unauth, live-verified). *Highest-yield check:* unexpected `.php` files under `modules/ps_facetedsearch/`. *Hardening:* patched versions; `Tools::unSerialize()` pattern; `disable_functions`.
4. **Theme ZIP import** — crafted theme carries `.tpl`/`.php`. *Hardening:* restrict import rights; validate on staging.
5. **Configuration write → persistent PHP injection** — SQLi-with-write or BO config pages poisoning hooks/caches. *Hardening:* read-only-by-design DB user turns this chain into read-only exfil.

## 7. Detection Essentials

- **`ps_log` table** (Advanced Parameters → Logs): ship daily to SIEM; alert on severity ≥3 bursts, `SQLSTATE` strings, repeated "Invalid token" from one IP.
- **Access-log signatures:** `/module/.*` with `(union|select|sleep|benchmark|orderby|sortorder)`; `/module/advancedpopupcreator/` + long `fromController=`; `O:\d+:`/`;i:`/`Monolog\` in search URLs; `phar://`; fingerprinting GETs (`composer.json`, `config/settings.inc.php`, `cache/class_index.php`); `/modules/` listing.
- **Canaries:** honeypot employee account; canary customer rows (`canary-<token>@yourdomain`); canary `ps_configuration` row; file-integrity baseline of `modules/`, `themes/`, `var/cache/`.
- **FOP feed watch (best ecosystem signal):** https://security.friendsofpresta.org/ + repo https://github.com/Friends-Of-Presta/security-advisories; core advisories at https://github.com/PrestaShop/PrestaShop/security/advisories. Weekly cron grep for your module names.

## 8. Hardening Checklist

| # | Control | Action |
|---|---|---|
| 1 | Core version | ≥9.1.5 / 8.2.8 minimum (current stable 9.2.x); 1.6/1.7 = EOL, migrate via official 1-Click Upgrade on staging |
| 2 | Modules | pin versions per §4; uninstall **and delete** unused; monthly version-vs-FOP-catalog diff |
| 3 | Webservice keys | one per consumer, view-only where possible, named owner, quarterly rotation, IP-restrict `/api/` |
| 4 | Back office | `/adminXXX` renamed + IP allowlist + fail2ban on misses; HTTPS + HSTS |
| 5 | DB user | SELECT/INSERT/UPDATE/DELETE on shop DB only — no FILE, no LOAD DATA, no cross-DB, no SUPER |
| 6 | Smarty/debug | `PS_SMARTY_DEBUG` off; clean `var/cache/`; never dev mode on prod |
| 7 | File perms | `app/config/`, `config/`, `var/` not web-readable; 600/640 on `settings.inc.php`/`parameters.php`; `/img/` served without PHP handler |
| 8 | `disable_functions` | `exec,passthru,shell_exec,system,proc_open,popen,proc_close,proc_nice,pcntl_exec,proc_get_status` (test on staging) |
| 9 | Monitoring | `ps_log` shipping + access-log signatures + FOP watch + canaries; keep the PHP runtime patched too (php2shell-class CVEs) |

## References

- `references/full-playbook.md` — complete source playbook (all sections + verified source index).
- `references/cve-currency.md` — PrestaShop rows from the cross-CMS CVE master table (verified 2026-10-29).
- `references/tooling.md` — ps-scan / PrestaScan Security / sqlmap tooling with licenses and log footprints.
- Core advisory (BO filter SQLi): https://github.com/PrestaShop/PrestaShop/security/advisories/GHSA-whxq-pxj5-qq7v
- Friends-Of-Presta: https://security.friendsofpresta.org/
- 9.1.5 security release: https://build.prestashop-project.org/news/2026/prestashop-9-1-5-security-release/
- ps_facetedsearch advisory: https://github.com/PrestaShop/ps_facetedsearch/security/advisories/GHSA-m5f5-28qr-9g9r

## Appendix — Quick Signatures

```regex
/module/.*(union|select|sleep|benchmark|orderby|sortorder)   # module SQLi (URL-decode first)
/module/advancedpopupcreator/.*fromController=.{40,}          # CVE-2025-69633 intake
[?&]q=.*(O:\d+:|;i:|Monolog\\)                                 # CVE-2026-54159 object injection
phar://                                                        # CVE-2025-25692-class PHAR
GET /(composer\.json|CHANGELOG\.md|config/settings\.inc\.php|app/config/parameters\.php|cache/class_index\.php)
GET /modules/($|\?)                                            # module directory listing recon
/modules/[a-z0-9_]+/logo\.png                                  # high-rate module enumeration
filter_[a-z_]+=.*('|%27|union|sleep)                           # BO list-filter SQLi (GHSA-whxq-pxj5-qq7v)
```
