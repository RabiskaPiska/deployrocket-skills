---
name: magento-playbook
description: "Defensive-security playbook for Adobe Commerce / Magento Open Source 2.x. Covers the attack-surface map (REST /rest/V1 guest-carts, GraphQL /graphql, searchCriteria filters, admin frontName, var/export CSVs), ACL/token model incl. integration tokens surviving password rotation, core tables with the hash:salt:version chained format (MD5/SHA-256/Argon2id13), injection classes with verified CVEs (CosmicSting CVE-2024-34102 XXE-to-RCE - CISA KEV, CVE-2025-54236 SessionReaper, CVE-2026-75650 CVSS 10.0 exploited in wild, CVE-2016-4010, Mirasvit CVE-2026-45247 KEV), Magecart skimmer-planting via cmsBlock, exfil paths, and RCE chains. USE WHEN: assessing or hardening a Magento store, responding to CosmicSting/SessionReaper-class incidents, or writing detection rules. Every technique is paired with Detection + Hardening; verified live 2026-10-29."
license: MIT
tags: [magento, adobe-commerce, ecommerce, defensive-security, hardening, detection, xxe, deserialization, magecart, cve-currency]
updated: 2026-10-29
---

# Magento — Attack & Defense Playbook (Condensed)

Adobe Commerce / Magento Open Source 2.x pairs a rich anonymous REST/GraphQL surface with a
crypt-key-centered auth model — which is why one XXE read of `env.php` (CosmicSting) cascaded into
4,275+ compromised stores. Every technique below is paired with **Detection** and **Hardening**;
claims are marked verified [V] or UNVERIFIED [U] in the full playbook. Status date 2026-10-29.

## 1. Attack Surface Map

- **Admin panel** at a random `backend.frontName` (from `app/etc/env.php`); `/admin` still resolves on legacy/misconfigured stores.
- **REST API** `/rest/V1/...` — anonymous tier: guest-carts endpoints are the richest unauth surface (CVE-2016-4010, CosmicSting).
- **GraphQL** `/graphql` — introspection on by default; anonymous query abuse on headless stores.
- **searchCriteria filters** (REST+GraphQL) — historical unauth SQLi (CVE-2019-7139 in `prepareSqlCondition`); modern abuse = `pageSize` bulk-walk enumeration.
- Web Setup Wizard `/setup` — **removed in 2.4.0**; still live/exposed on ≤2.3.
- Admin grids/CMS blocks/email templates — stored XSS → session theft; **CMS-block editing is the observed Magecart planting vector**.
- `pub/media/` uploads (PHP-handler misconfig → webshell); cron/queue consumers (poisoned payloads run in elevated CLI context); `var/export/` CSVs (unauth PII if web-readable).
- **Detection:** bursts on `/magento_version`, `CHANGELOG.md`, `composer.json`, `/setup`, `/static/version*`; the ITW pair `GET /V1/cmsBlock/search` → `PUT /V1/cmsBlock/{id}`; any 200 on a second guessed admin path. **Hardening:** serve only `pub/` as docroot; strip `/magento_version` at CDN; production mode (no debug headers); IP-allowlist admin.

## 2. Auth & Privilege Classes

- Admin ACL: role → `authorization_role` → resource IDs gate UI **and** Web API; integration tokens inherit creator-role resources.
- Separate realms: admin vs customer sessions/cookies/password columns; cross-realm confusion is its own bug class.
- **Token types:** admin API token (2FA-gated since 2.4.0); **integration/OAuth 1.0a tokens survive admin password rotation** — the reason Adobe + Sansec ordered token cleanup during crypt-key rotation; customer tokens (storefront scope); sessions.
- **CosmicSting weaponization [V]:** XXE reads `env.php` → `crypt/key` → forge integration token offline → full admin API without touching login or 2FA.
- 2FA [V]: `Magento_TwoFactorAuth` mandatory since 2.4.0 for admin UI **and** token issuance (TOTP/Authy/Duo/U2F; `security:tfa:*` CLI for headless enrollment).
- **Detection:** `integration/admin/token` issuance spikes; issuance followed by cmsBlock writes; dormant-token first-use after >90 days; `oauth_token` table growth. **Hardening:** force 2FA; least-privilege roles; revoke integration tokens on offboarding; quarterly `oauth_token`/`oauth_consumer` audit; lockout + reCAPTCHA; `bin/magento config:set web/secure/use_in_adminhtml 1`.

## 3. Database & Hash Formats (condensed)

| Table | Security relevance |
|---|---|
| `admin_user` / `tfa_user_config` | hash-crack target; 2FA enrollment forensics |
| `authorization_role`/`authorization_rule` | spot/plant over-privileged roles |
| `customer_entity` + address tables | primary PII exfil set via `searchCriteria` |
| `sales_order*` | buyer PII; core stores only gateway tokens/last-4 |
| `core_config_data` | crypt-adjacent config, **admin path**, payment endpoints — the secrets map |
| `oauth_token`/`integration` | persistence surviving password rotation [V] |
| `cms_block`/`cms_page` | Magecart skimmer landing zone [V] |
| `quote`/`quote_*` | guest-cart REST payloads — CosmicSting intake |

- **Hash format [V: Encryptor.php + Adobe docs]:** stored as `hash:salt:version[:version...]` — salt **prepended** (`salt . password`), chained versions. `0` = MD5 (2.0–2.1 legacy), `1` = SHA-256, `2`/`3` = Argon2id13 (libsodium). `:0`/`:1` rows crack fast on GPU; audit with `SELECT password FROM admin_user WHERE password LIKE '%:0' OR password LIKE '%:1';` and run `bin/magento customer:hash:upgrade`.
- **`app/etc/env.php` = the prize file [V]:** `backend.frontName` (secret admin path), `crypt.key` (token forgery), DB credentials. This is why CosmicSting exfiltrates exactly this path.
- **Detection:** slow-log probes shaped like `prepareSqlCondition` fragments; canary admin + customer rows. **Hardening:** table-scoped app-scoped MySQL GRANTs (no FILE/PROCESS); `secure_file_priv` set; no root creds in `env.php`.

## 4. Injection Classes — Top CVEs

- **CosmicSting — CVE-2024-34102 / GHSA-m8cj-3v68-3cxj (CVSS 9.8, KEV 2024-07-17) [V]:** unauth XXE (CWE-611) via nested deserialization in `/rest/V1/guest-carts/<id>/shipping-information` (`totalsReader.collectorList.totalCollector.sourceData`) → read `env.php` → forge tokens → cmsBlock JS skimmers (Sansec: 4,275 stores, 7 groups). Optional RCE via CNEXT glibc iconv (CVE-2024-2961). Fixed 2.4.7-p1 / 2.4.6-p6 / 2.4.5-p8 / 2.4.4-p9 + isolated ACSD-60241. — https://sansec.io/research/cosmicsting · https://helpx.adobe.com/security/products/magento/apsb24-40.html
- **CVE-2016-4010 (CVSS 9.8):** unauth PHP object injection via guest-cart REST `payment` object → file write → RCE; Metasploit `magento_unserialize`, EDB-39838. The historical archetype.
- **CVE-2025-54236 "SessionReaper" [V via secondary]:** unauth RCE via nested deserialization + `/customer/address_file/upload` on file-based session stores (Monolog/Guzzle/Laminas POP); ~81% of stores "visited" within 10 days per Sansec. ⚠️ confirm APSB bulletin ID on helpx.adobe.com before citing.
- **CVE-2026-45247 (Mirasvit Full Page Cache Warmer <1.11.12, KEV 2026-06-03) [V]:** unauth `unserialize()` of the `CacheWarmer` cookie → gadget RCE — proof that third-party modules now carry the class.
- **CVE-2026-103831 (TrueLayer plugin 2.4.0–2.4.2) [V]:** `Psr16CacheAdapter` unrestricted `unserialize()` on cache backends — cache-poisoning precondition class.
- **APSB26-146 / CVE-2026-75650 (CVSS 10.0, exploited in wild per Adobe) [V]:** template-engine injection (CWE-1336), unauth → arbitrary code execution; hotfix 2026-09-07, **separate from Sep/Oct isolated patches**.
- Honest notes [U]: "Cesium143" gadget name unverified (iconv half = CNEXT, verified); "Sylius-style injection" is not a Magento class; no 2024–2026 classic core-`unserialize()` APSB entry found.
- **Detection:** cookie/param payloads starting `O:`/`a:` or base64 `Tz|Qz|YT`; Sansec indicator `CacheWarmer:(Tz|Qz|YT)`; `convert.iconv.` in bodies; OOB DNS from PHP processes. **Hardening:** patch modules per KEV feed; `unserialize(..., ['allowed_classes' => false])` or JSON in custom code; authenticated, network-isolated Redis/Memcached; `composer audit` in CI.

## 5. Exfil Paths

| # | Path | Detection | Hardening |
|---|---|---|---|
| 1 | REST bulk PII — `GET /V1/customers/search?searchCriteria[pageSize]=500&page=N` with a token | token-scoped volume anomalies; non-export-role token reading at rate | least-priv API roles; pageSize caps; per-token limits |
| 2 | GraphQL introspection walk | `__schema` from non-dev IPs; deep anonymous `orders(filter:)` | introspection off in prod; rate-limit `/graphql` |
| 3 | `core_config_data` secrets via SQLi or env.php read | canary config row; SELECTs outside deploy windows | secrets in env/vault; crypt-key rotation runbook |
| 4 | `var/export/*.csv` direct download on misconfigured docroots | any direct `/var/export/` GET (should be zero) | `var/` outside docroot; signed-URL storage |
| 5 | SQLi extraction of admin hashes/sessions (CVE-2019-7139 class) | 500-rate spikes; latency shifts; MySQL slow log | ≥2.3.1 everywhere; strict binding |
| 6 | Magecart skimming via cmsBlock PUTs | client-side script-hash diffing; CSP reports; content diffs | strict CSP; SRI; checkout DOM monitoring |

## 6. RCE Chains

1. **Deserialization gadget chains (POI → RCE):** PHPGGC arsenals `Magento2/FD1`, `Monolog/RCE1|RCE2`, `Guzzle/FW1` reachable via the global autoloader; live 2026 sinks = Mirasvit cookie POI (KEV) and TrueLayer cache adapter. *Detection:* serialized-object indicators; OOB DNS; FIM in `pub/`/`var/`. *Hardening:* module inventory + `composer audit`; `allowed_classes:false`; isolated caches.
2. **Admin template/CMS → code execution:** server-side `{{...}}` directives; CVE-2026-75650 proves the pre-auth version of the class is live. *Detection:* cms_block/email-template diffs; unexpected `{{config`/`{{block` tokens. *Hardening:* apply APSB26-146 hotfix; treat content-edit roles as code-execution roles in the ACL model.
3. **Upload/extension bypass:** surviving variant is misconfiguration — PHP handler on `pub/media`, `.htaccess` overrides. *Hardening:* media static-only; extension allowlist; `disable_functions` + `open_basedir`.
4. **CosmicSting full chain:** XXE → env.php → token forgery → skimmer planting → optional CNEXT RCE → persistent backdoors (fake kernel process names `[kswapd0]`-style observed). *Hardening:* APSB24-40 + **crypt-key rotation WITH old-token invalidation** (rotation alone insufficient per Adobe) + patched glibc.
5. **Supply-chain/module install:** Composer-only since 2.4 — typo-squatted/backdoored packages or flawed vendor updates are the residual path. *Detection:* `composer.lock` diffs in CI; `module:status` vs change tickets. *Hardening:* pin deps; private mirror; review vendors.
6. **Assessor code-review pass:** grep `unserialize(`, `__wakeup|__destruct|__toString`, check input origin (cookie/param/API/cache) and PHPGGC-chain availability in the installed tree.

## 7. Detection Essentials

- **Logging:** Commerce-only Admin Actions Log (`magento_logging_event*`) — Open Source must compensate with reverse-proxy logs + DB triggers on `admin_user`/`cms_block`/`core_config_data`. Ship `var/log/system.log|exception.log|debug.log` to SIEM (iconv/unserialize debris).
- **Web-log signatures:** guest-cart shipping-information with DOCTYPE/SYSTEM (CosmicSting) · `searchCriteria[pageSize]=1000` walking · `GET /V1/cmsBlock/search` → `PUT /V1/cmsBlock/{id}` · `convert.iconv` · `__schema` · recon bursts on `/magento_version`/`CHANGELOG.md`.
- **Canaries:** honeypot admin + honeypot integration token — any auth event = high-confidence breach signal.
- **WAF:** Fastly with Adobe-managed rules (Commerce Cloud) or ModSecurity CRS + Magento rules; precedent — Sansec Shield blocked CosmicSting on disclosure day.
- **KEV watch:** CVE-2024-34102 (2024-07-17) and CVE-2026-45247 (2026-06-03) currently listed — ingest the feed and diff against installed versions/modules.

## 8. Hardening Checklist

| # | Control | Action |
|---|---|---|
| 1 | Patch level [V] | 2026-oct builds (APSB26-153) on 2.4.4–2.4.9; **APSB26-146 hotfix applied separately**; OSS 2.4.6 has no fixed release → plan migration to 2.4.7+ |
| 2 | Admin surface | non-default frontName verified via `bin/magento info:adminuri`; edge IP allowlist/VPN; 2FA mandatory; lockout + reCAPTCHA; tight `admin/security/session_lifetime` |
| 3 | Setup wizard | `/setup` returns 404 on 2.4+; remove/restrict on ≤2.3 |
| 4 | Database | app-scoped MySQL user, no FILE/PROCESS; `secure_file_priv`; encrypted restore-tested backups |
| 5 | API | per-token/per-IP rate limits; pageSize caps; revoke orphaned integrations; introspection off in prod |
| 6 | Crypt key | rotate after any suspected file-read **and invalidate old integrations**; `env.php` 0600, never web-readable |
| 7 | Caching/GraphQL | full-page + edge caching; cache-poisoning monitoring |
| 8 | CSP & frontend | core CSP module report-only → enforce; SRI on payment scripts; client-side integrity monitoring (Magecart is a client-side failure) |
| 9 | Scans | Adobe Security Scan Service scheduled; `magescan` external posture; `composer audit` |
| 10 | Verification | 12-point assessment checklist in full playbook §8.11 (version, admin path, 2FA, /setup, docroot hygiene, crypt key age, CSP, logs) |

## References

- `references/full-playbook.md` — complete source playbook (worked CosmicSting chain, request shapes, SIEM mapping, honesty ledger).
- `references/cve-currency.md` — Magento rows from the cross-CMS CVE master table (verified 2026-10-29).
- `references/tooling.md` — MageScan / n98-magerun2 tooling with licenses, staleness notes and log footprints.
- CosmicSting research: https://sansec.io/research/cosmicsting · PoC: https://github.com/Chocapikk/CVE-2024-34102
- Adobe bulletin APSB24-40: https://helpx.adobe.com/security/products/magento/apsb24-40.html
- Gadget chains: https://github.com/ambionics/phpggc
- Core source: https://github.com/magento/magento2

## Appendix — Quick Signatures

```regex
"guest-carts" AND "shipping-information" AND ("<!DOCTYPE" OR "SYSTEM")   # CosmicSting XXE (CVE-2024-34102)
searchCriteria%5BpageSize%5D=(500|1000)|searchCriteria\[pageSize\]=(500|1000)  # bulk-walk probing
PUT /rest/.*/V1/cmsBlock/                                                 # skimmer-planting PUT
GET  /rest/.*/V1/cmsBlock/search                                          # skimmer-planting recon pair
convert\.iconv                                                            # CNEXT filter-chain (CVE-2024-2961)
__schema                                                                  # GraphQL introspection walk
CacheWarmer:(Tz|Qz|YT)                                                    # Mirasvit CVE-2026-45247 (KEV)
GET /(magento_version|CHANGELOG\.md|composer\.json|setup)                 # recon burst
```

## PolyShell Campaign — CVE-2026-48356 / APSB26-73

**CVE:** CVE-2026-48356 | **Adobe Bulletin:** APSB26-73
**Severity:** Critical (unauthenticated remote code execution)
**Status:** Actively exploited — 953+ stores injected as of Oct 2026
**First seen:** March 2026 (471 stores); ongoing campaign Oct 2026

### Technique
Unauthenticated file upload via `pub/media/custom_options/quote/` endpoint.
Attackers disguise PHP webshells as PNG files (magic bytes manipulated).
Shells dropped to `pub/media/custom_options/quote/<random>.png` but executed as PHP.

### IOC
- Loader domain: `deepveindrilling[.]com`
- File pattern: `pub/media/custom_options/quote/*.png` (check for PHP content)
- Detection: `grep -r "<?php" pub/media/custom_options/`

### Exploitation chain
1. POST to unauthenticated media upload endpoint
2. PNG-disguised PHP shell written to `pub/media/custom_options/quote/`
3. Attacker GETs shell URL to execute commands
4. Full RCE achieved without authentication

### Detection
```bash
find pub/media/custom_options/quote/ -name "*.png" -exec grep -l "<?php" {} \;
grep -r "deepveindrilling" /var/log/nginx/access.log
```

### Patch
Apply APSB26-73 patch (released March 2026). Verify Magento version ≥ patched release.
Restrict write permissions on `pub/media/custom_options/quote/` post-patch.

### References
- https://helpx.adobe.com/security/products/magento/apsb26-73.html
- https://sansec.io/research/cosmicsting-magento-webshell-campaign-2026
