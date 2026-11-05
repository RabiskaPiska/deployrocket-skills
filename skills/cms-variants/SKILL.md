---
name: cms-variants
description: >-
  Advanced/variant exploit-flow playbook for WordPress, PrestaShop, Joomla, Drupal
  and Magento — the chains to reach for when the basic paths are closed: wp-cli as
  attack primitive, REST/XML-RPC abuse (multicall brute, pingback SSRF), object-cache
  poisoning, second-order backup-restore injection (CVE-2026-19949 class), PrestaShop
  module upload + Smarty SSTI + PHAR depth, Joomla extension-install and com_* chains,
  the Drupalgeddon lineage + PHP filter + hal+json, Magento admin deserialization +
  SessionReaper (CVE-2025-54236) + layout-XML injection, and the cross-CMS php2shell /
  PHPGGC shared-runtime chains. EVERY flow pairs its detection signal and hardening
  fix. Use for authorized CMS engagements (flow selection by access level + exposure),
  purple-team planning, and building detection/WAF coverage. All targets are
  placeholders; substitute only in-scope systems.
license: internal-agency-use
compatibility: OpenCode skills (SKILL.md frontmatter spec, verified 2026-10-29)
metadata:
  source-report: 00-MASTER-STATUS/CMS-VARIANT-FLOWS-2026-10-29.md
  run: FORGE-DEPLOY-V2 seat-32cms-retry
  verified: 2026-10-29
---

# cms-variants — Advanced CMS Exploit-Flow Variants (detection-paired)

Variant flows for when the basic CMS paths are closed. **Every flow carries PRECONDITIONS → CHAIN → VARIANT NOTES → DETECTION → FIX.** Full content lives in `references/` — this file is the router.

## START HERE — Scenario Index

| I have… | …and this is exposed | Flow → reference |
|---------|----------------------|------------------|
| Unauthenticated | WP XML-RPC on | WP-A3 multicall brute + pingback SSRF → `references/wordpress-advanced.md` |
| Unauthenticated | WP REST open | WP-A2 user enum → app-password chains → `references/wordpress-advanced.md` |
| Unauthenticated | PS ps_facetedsearch ≤4.0.3 | PS-P2 unauth webshell write (CVE-2026-54159) → `references/prestashop.md` |
| Unauthenticated | PS module endpoints | PS-P4 SQLi seeds · PS-P5 PHAR → `references/prestashop.md` |
| Unauthenticated | Joomla 4.0–4.2.7 API | JA-03c CVE-2023-23752 → DB creds → `references/joomla.md` |
| Unauthenticated | Drupal pre-SA-CORE-2018-002 | D1 Drupalgeddon2 (CVE-2018-7600, KEV) → `references/drupal.md` |
| Unauthenticated | Any of the five, PHP ≤8.2 early | CC-1 php2shell UAF (KEV 2026-10-26) → `references/cross-cms-runtime.md` |
| Low-priv account | Magento REST/GraphQL | MG-F1 SessionReaper (CVE-2025-54236) → `references/magento.md` |
| WP admin | Plugin/theme upload | WP-A4 zip-slip · A2-F1 archive restore (CVE-2026-19949) → `references/wordpress-second-order.md` |
| WP admin | Backup/import plugin | A2-F2/F3 backup-restore carriers → `references/wordpress-second-order.md` |
| WP shell/cron | wp-cli binary present | WP-A1 wp-cli primitive → `references/wordpress-advanced.md` |
| PS employee | Filter/export · translation edit | PS-P3 BO chain · PS-P2b Smarty SSTI → `references/prestashop.md` |
| Joomla admin | Extension install · template mgr | JA-01/JA-02 → `references/joomla.md` |
| Drupal admin | PHP filter format · AJAX callbacks | D3/D2 → `references/drupal.md` |
| Magento admin | Layout updates · ≤2.3.x wizard | MG-F5/MG-F4 → `references/magento.md` |
| Deserialization sink | monolog/guzzle/symfony in tree | CC-3 PHPGGC · MG-F3 Sylius-class → `references/cross-cms-runtime.md` |
| Defender | Building coverage for any flow | `references/detection-matrix.md` (master table, WAF, canaries, KEV pack) |

## Reference Map

| File | Contents |
|------|----------|
| `references/wordpress-advanced.md` | WP-A1 wp-cli · WP-A2 REST/app-passwords · WP-A3 XML-RPC (CVE-2022-3590) · WP-A4 zip-slip · WP-A5 object-cache |
| `references/wordpress-second-order.md` | CVE-2026-19949 canonical case · backup lifecycle · import carriers · install/update pipeline |
| `references/prestashop.md` | PS-P1 module upload · PS-P2 Smarty SSTI · PS-P3 BO chain · PS-P4/P5 SQLi+PHAR (5 seed advisories verified) |
| `references/joomla.md` | JA-01 install chains · JA-02 template injection · JA-03 com_* classes · JA-04 supply-chain note |
| `references/drupal.md` | D1 Drupalgeddon lineage (SA-CORE-2014-005 → 2018-002 → 2019-003) · D2 AJAX renderer · D3 PHP filter · D4 hal+json |
| `references/magento.md` | MG-F1 SessionReaper · MG-F2 unserialize lineage · MG-F3 Sylius-class · MG-F4 setup wizard · MG-F5 layout XML · MG-F6 API |
| `references/cross-cms-runtime.md` | php2shell (CVE-2026-109618/109619) · phar:///.htaccess/session primitives · PHPGGC · N-day families · php.ini baseline |
| `references/detection-matrix.md` | ~60-row master table · 8 WAF sketches · canary placements · KEV fingerprint pack · per-CMS SIEM checklists |
| `references/tooling.md` | wp-cli/wpscan/droopescan/joomscan/PHPGGC/nuclei — install + defender's mirror per tool |
| `references/packaging.md` | Catalog packaging decisions + sanitizer checklist |

## Usage Rules

1. **Authorized engagements only.** All content is published tradecraft (public CVEs, vendor advisories, Patchstack/Wordfence/Friends-of-Presta, MITRE ATT&CK T1190/T1210/T1505.003/T1059/T1078/T1005).
2. **Carry the defensive half back.** A flow executed without shipping its DETECTION + FIX is an incomplete engagement deliverable.
3. **Check the flagged gaps** (aggregate table in the source report's Evidence Statement) before trusting: AI1WM restore-time internals, Drupalgeddon1 payload body, Magento APSB26-146 bands, SessionReaper KEV status, WAF sketches (not lab-fired — deploy in Detection mode first).
4. **Companion skills:** base attack-surface playbooks = `cms-playbooks` (seat-34, landed 2026-10-29 post-merge; disjointness adversarially re-checked — variant layer holds) · runtime chains detail = `php2shell` (seat-32) · CVE currency = `sqli-cve-currency`, `php-cve-currency`.

## Verification

Install check: `opencode debug skill` should list `cms-variants`. Sanitizer: only placeholder hosts (`target.tld`, loopback) — no credentials anywhere in this skill.
