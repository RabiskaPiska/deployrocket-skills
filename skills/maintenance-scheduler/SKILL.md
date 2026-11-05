# maintenance-scheduler

## Purpose
Weekly/monthly/quarterly maintenance calendar for the fleet. Provides a structured checklist, per-cadence runbook, and pass/fail gate for each item. Run this before declaring the fleet "healthy" or during a scheduled review window.

## Trigger Conditions
- User says: "run maintenance", "check fleet health", "weekly review", "monthly audit", "is the fleet healthy", "maintenance run"
- Monday morning session startup (auto-suggest if last maintenance stamp is >6 days old)
- After any large deployment (post-deploy health sweep)
- Before a client delivery or public launch

---

## Maintenance Stamp File
```
~/.config/opencode/memory/__maintenance-stamp.json
```
Format:
```json
{
  "last_weekly":    "2026-10-27T09:00:00Z",
  "last_monthly":   "2026-10-01T09:00:00Z",
  "last_quarterly": "2026-10-01T09:00:00Z",
  "run_by":         "web-dev-vps",
  "status":         "PASS"
}
```
Update this file at the end of every maintenance run regardless of pass/fail.

---

## Weekly Checklist (every 7 days)

Run all items. Each must reach PASS before the weekly stamp is written.

### W1 — Pull-sync catalog and vault
```
Skill("pull-sync")
```
PASS: local stamps match remote (see update-checker). FAIL: pull-sync errors — investigate, re-run.

### W2 — CVE-currency check
Search for new advisories affecting the fleet's active stacks (WordPress, PrestaShop, PHP, Node, Cloudflare Workers, Supabase, Better Auth):
```
execute: tools.exa.web_search_exa({
  urls: ["https://nvd.nist.gov/vuln/search?query=wordpress&results_type=recent"]
})
+ tools["parallel-search"].web_search({ search_queries: [
    "wordpress CVE 2026 site:nvd.nist.gov",
    "prestashop CVE 2026 site:nvd.nist.gov",
    "PHP CVE 2026 critical"
  ]})
```
PASS: no new CVSS≥7.0 unpatched advisory for an in-use component.
FAIL / ACTION NEEDED: log the CVE to `cybersec/knowledge/CVE-CURRENCY.md`, update the relevant playbook skill, flag to user.

### W3 — Dead-link sweep (brain + skills)
```bash
# On VPS
grep -rh "https://" /srv/dr/brains/ ~/.config/opencode/skills/ \
  | grep -o 'https://[^)"` >]*' | sort -u > /tmp/links.txt
# Sample 20 random links (full sweep is monthly)
shuf -n 20 /tmp/links.txt | xargs -P5 -I{} curl -sI --max-time 6 {} \
  | grep -E "^(HTTP|curl)" | grep -v "200\|301\|302"
```
PASS: 0 dead links in the sample. FAIL: log dead links to `_ops/DEAD-LINKS.md`, update the referencing doc.

### W4 — MCP server health
```
Skill("mcp-health-checker")
```
PASS: all 4 servers UP or known-degraded with active fallback. FAIL: server has been down >48h — investigate root cause, update fallback.

### W5 — VPS process check (both VPSes)
```bash
ssh rabi@179.61.138.8 "ps aux | grep -E 'opencode|claude|supervisor' | grep -v grep | wc -l"
ssh rabi@179.61.251.98 "ps aux | grep -E 'opencode|claude|supervisor' | grep -v grep | wc -l"
```
PASS: at least 1 process per VPS. FAIL: restart via `bash /srv/dr/launch-ultra.sh` on the affected VPS.

### W6 — Brain sync verification
```bash
ssh rabi@179.61.138.8 "cd /srv/dr && git log --oneline -3"
ssh rabi@179.61.251.98 "cd /srv/dr && git log --oneline -3"
```
Both VPSes should show the same top commit as the GitHub central brain. FAIL: run `git pull origin main` on the lagging VPS.

---

## Monthly Checklist (1st Monday of each month)

Run ALL weekly items first, then:

### M1 — Full dead-link sweep
```bash
grep -rh "https://" /srv/dr/brains/ ~/.config/opencode/skills/ \
  | grep -o 'https://[^)"` >]*' | sort -u | xargs -P10 -I{} \
    curl -sI --max-time 8 {} | grep -E "^(HTTP|curl)" \
    | grep -v "200\|301\|302" > /tmp/dead-links-full.txt
wc -l /tmp/dead-links-full.txt
```
PASS: <5 dead links. >5: triage and update all referencing docs.

### M2 — License and ToS recheck
Verify licenses for the top 5 tools used in client deliverables (Playwright, Tailwind, shadcn/ui, GSAP free tier, Cloudflare free tier).
```
execute: tools.exa.web_search_exa({
  urls: ["https://greensock.com/licensing/", "https://tailwindcss.com/docs/license"]
})
```
PASS: no license changes that affect current usage. FAIL: update `_ops/LICENSE-REGISTRY.md` and notify user.

### M3 — Dependency advisory scan (active projects)
```bash
ssh rabi@179.61.138.8 "find /srv/dr/projects -name 'package.json' -not -path '*/node_modules/*' \
  | head -10 | xargs -I{} dirname {} | xargs -I{} sh -c 'cd {} && npm audit --json 2>/dev/null | python3 -c \"import sys,json; d=json.load(sys.stdin); print(d.get(\\\"metadata\\\",{}).get(\\\"vulnerabilities\\\",{}))\"'"
```
PASS: 0 critical/high unpatched. FAIL: log to `_ops/DEPENDENCY-ADVISORIES.md`, patch or document accepted risk.

### M4 — Storage audit
```bash
ssh rabi@179.61.138.8 "df -h / && du -sh /srv/dr/projects/* 2>/dev/null | sort -rh | head -10"
ssh rabi@179.61.251.98 "df -h / && du -sh /srv/dr/brains/* 2>/dev/null | sort -rh | head -10"
```
PASS: <80% disk usage. FAIL: archive old projects, clean node_modules, compress logs.

### M5 — Backup integrity check
```bash
# Verify OneDrive sync has recent backup
ls -lt "C:/Users/rabim/Backups/AGENCY-BACKUP/" | head -5      # PC
# Verify GitHub brain has recent commit
curl -s "https://api.github.com/repos/RabiskaPiska/deployrocket-central-brain/commits/main" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['commit']['author']['date'])"
```
PASS: last backup <48h old. FAIL: trigger manual backup immediately.

---

## Quarterly Checklist (1st Monday of Jan/Apr/Jul/Oct)

Run ALL weekly + monthly items first, then:

### Q1 — Full roster review
Audit all declared skills in `~/.config/opencode/skills/` — remove skills not used in 90 days, consolidate near-duplicates, update descriptions in AGENTS.md.

### Q2 — Vault freshness audit
Read every note in `agency-brain/_INDEX.md`. For each note, check if the core claims are still accurate (versions, pricing, tool availability). Date-stamp stale notes with `[NEEDS-REFRESH]`.

### Q3 — Fleet credential rotation reminder
Check expiry dates for: GitHub PAT, Supabase service keys, Cloudflare API tokens, any active MCP OAuth tokens. Flag anything expiring within 60 days to the user.

### Q4 — Security posture review
Run the full pentest-edu skill stack against one representative client site. Log findings to `cybersec/knowledge/QUARTERLY-POSTURE.md`.

### Q5 — Brain consolidation
Merge or archive redundant notes. Promote any 2+ pattern occurrences to `patterns/`. Remove TODOs older than 90 days that are no longer actionable.

---

## Maintenance Report Format

```
MAINTENANCE RUN — 2026-10-31T10:00:00Z
  Cadence     : WEEKLY
  Machine     : web-dev-vps (179.61.138.8)

  W1 pull-sync      : ✅ PASS
  W2 CVE-currency   : ✅ PASS  (no new CVSS≥7 advisories)
  W3 dead-link sample: ✅ PASS  (0/20 dead)
  W4 MCP health     : ⚠️  DEGRADED  playwright DOWN (fallback active)
  W5 VPS processes  : ✅ PASS  (3 procs on .8, 2 on .98)
  W6 brain sync     : ✅ PASS  (both VPSes on commit a3f9b12)

  OVERALL: PASS with 1 advisory (playwright)
  Stamp written: ~/.config/opencode/memory/__maintenance-stamp.json
```

---

## Notes
- Never write PASS to the stamp if any FAIL item is unresolved.
- ADVISORY items (degraded but fallback active) do not block the PASS stamp, but must be logged.
- The maintenance-scheduler does not fix issues — it diagnoses them and invokes the right skill/runbook. Log all findings; execute fixes autonomously when they are simple (pull-sync, process restart); escalate to user only for credentials, payments, or destructive ops.
