---
id: bug-bounty-agent
name: Bug Bounty Agent — Triage, Scoring & Submission Orchestrator
description: >
  End-to-end bug bounty workflow: scope validation, CVSS scoring, PoC
  documentation, duplicate detection, platform-specific submission formatting
  (HackerOne, Bugcrowd, Intigriti), severity calibration, and disclosure
  timeline management. Triggers on "bug bounty", "HackerOne", "Bugcrowd",
  "Intigriti", "submit finding", "triage report", "write PoC", or after any
  pentest agent returns a confirmed vulnerability that needs submission.
triggers:
  - bug bounty
  - HackerOne
  - Bugcrowd
  - Intigriti
  - submit finding
  - triage report
  - write PoC
  - vulnerability report
  - disclose finding
  - CVSS score
  - severity calibration
version: "1.0"
author: fleet-seat-14
date: 2026-10-30
---

# Bug Bounty Agent — Triage, Scoring & Submission Orchestrator

## Purpose

This skill converts a raw technical finding into a correctly scored, well-documented,
platform-formatted bug bounty report that maximizes triage speed and acceptance rate.
It covers the full lifecycle: scope check → dedup → CVSS → PoC doc → submission →
disclosure timeline.

---

## Phase 1 — Scope Validation (run BEFORE any other phase)

### 1.1 In-Scope Check

```
Given: target asset (domain / endpoint / binary / API)

1. Fetch the current program policy page (H1 / Bugcrowd / Intigriti — always
   re-fetch; policies change without notice).
2. Match the asset against the SCOPE table:
     IN SCOPE   → proceed to Phase 2
     OUT OF SCOPE → STOP; do not document, do not submit; note why
     AMBIGUOUS  → contact program via "Ask a question" before testing further
3. Check for explicit EXCLUSIONS:
     - Third-party services (CDN, payment processor, OAuth provider)
     - Self-XSS, clickjacking without demonstrated impact
     - Rate-limiting without demonstrated impact (many programs)
     - SPF/DMARC issues without demonstrated spoofing path
     - Missing security headers (Content-Type-Options etc.) — often informational only
     - Scanner output without manual PoC
```

### 1.2 Eligibility Gates

All of the following must be TRUE before writing the report:

- [ ] Tested only in scope assets
- [ ] No automated scanning against production without explicit permission
- [ ] No data exfiltration beyond the minimum required to demonstrate impact
- [ ] No social engineering of program staff
- [ ] No public disclosure before the program's coordinated disclosure window closes
- [ ] Finding is reproducible (you can re-run it right now)

---

## Phase 2 — Duplicate Check Procedure

Submitting a known duplicate wastes everyone's time and hurts your reputation score.

### 2.1 Self-check (always)

1. Search your own submitted reports on the platform for the same asset + vuln class.
2. Check the program's "Resolved" public reports (H1 Hacktivity, Bugcrowd's public
   disclosure) for the same endpoint and technique.
3. If a duplicate is suspected: check if the root cause differs (same symptom,
   different parameter / different sink = potentially new report — document the
   distinction explicitly in the submission).

### 2.2 Dedup Signal Table

| Signal | Likely Duplicate | Action |
|--------|-----------------|--------|
| Identical endpoint + identical payload type | High | Check disclosed reports; if exact match, skip |
| Same endpoint, different parameter | Medium | Submit with explicit "different parameter" note |
| Same vuln class, different sub-domain | Low | Submit; note the scope expansion |
| Same endpoint, but new bypass of a deployed fix | None | Submit as new; reference the original report # |

---

## Phase 3 — CVSS 3.1 Scoring Guide

Use CVSS 3.1 (not 4.0 unless the program explicitly requests it — most platforms still
display 3.1 scores in triage dashboards as of 2026).

### 3.1 Metric Quick-Reference

```
Attack Vector (AV):    N=Network  A=Adjacent  L=Local  P=Physical
Attack Complexity (AC): L=Low  H=High
Privileges Required (PR): N=None  L=Low  H=High
User Interaction (UI):  N=None  R=Required
Scope (S):              U=Unchanged  C=Changed
Confidentiality (C):    N=None  L=Low  H=High
Integrity (I):          N=None  L=Low  H=High
Availability (A):       N=None  L=Low  H=High
```

### 3.2 Common Finding → CVSS Starter Scores

| Finding Type | Typical Vector | Base Score |
|---|---|---|
| Unauthenticated SQLi → full DB read | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H | 9.8 Critical |
| Auth SQLi → own data only | AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:L/A:N | 5.4 Medium |
| Stored XSS → admin panel | AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N | 5.4 Medium |
| Reflected XSS → no CSP | AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N | 6.1 Medium |
| IDOR → read other user data | AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N | 6.5 Medium |
| IDOR → modify other user data | AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N | 8.1 High |
| SSRF → internal metadata | AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:N/A:N | 8.6 High |
| RCE via file upload | AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H | 8.8 High |
| Auth bypass → admin access | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H | 9.8 Critical |
| Open redirect | AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N | 6.1 Medium |
| Subdomain takeover | AV:N/AC:L/PR:N/UI:N/S:C/C:L/I:L/A:N | 6.5 Medium |

### 3.3 Severity Calibration Rules

Programs routinely downgrade submissions. Pre-empt downgrades with these rules:

- **Never claim Critical without demonstrated data exfil or RCE.** "Could lead to" is
  not evidence — show the actual output.
- **CVSS base score ≠ program severity.** Many programs cap payouts at "High" even for
  9.8 base if the business impact is limited. Acknowledge this in the report.
- **AC:L requires an unconditional exploit path.** If you needed a race condition,
  specific browser version, or non-default config, use AC:H.
- **UI:N for XSS is rare.** A victim must click / visit something — almost always UI:R
  unless you have a persistent XSS in an auto-rendered context (email client, dashboard
  widget that auto-loads).
- **S:C (Scope Changed) only when you break out of the vulnerable component's security
  domain** — a reflected XSS that steals cookies from a different sub-domain qualifies;
  one that only runs in the victim's own session does not.

---

## Phase 4 — PoC Documentation Template

Fill every section. Incomplete PoCs are the #1 cause of triager delays.

```markdown
## Title
[Vuln Class] in [Component/Endpoint] allows [Impact]

Example: "Stored XSS in /profile/bio allows attacker to steal admin session tokens"

## Severity
CVSS 3.1: [vector string]  Base Score: [X.X] [Critical/High/Medium/Low]

## Asset
URL / endpoint / binary / API route:

## Description
One paragraph: what the vulnerability is, why it exists, what an attacker can do.
No jargon the triage team won't know. No speculation — only what you demonstrated.

## Steps to Reproduce
1. Log in as a low-privilege user (email: test@example.com, password: [provided])
2. Navigate to [exact URL]
3. In the [field name] field, enter the following payload:
   [PAYLOAD — exact, copy-pasteable]
4. Click [button / trigger action]
5. Observe: [exact observable result — cookie exfil, error message, query output]

## Impact
What an attacker can do with this vulnerability, in business terms:
- Data accessed / modified
- Accounts compromised
- Regulatory exposure (GDPR, PCI-DSS if relevant)
- Escalation path (does this chain to something worse?)

## Evidence
- Screenshot 1: [description] — attached as poc-1.png
- HTTP request/response: [Burp Suite export or raw request block]
- Video (optional but strongly recommended for complex chains)

## Remediation Suggestion
Brief, actionable. Example: "Parameterize the query at line 42 of UserController.php"
Not required by all programs, but improves acceptance rate significantly.

## Additional Notes
- Tested on: [date]
- Affected versions: [if known]
- Not tested on: [out-of-scope related assets you did NOT touch]
```

---

## Phase 5 — Platform-Specific Submission Formats

### 5A. HackerOne (H1)

- **Weakness field**: map to the CWE list H1 provides (CWE-89 for SQLi, CWE-79 for XSS,
  CWE-918 for SSRF, etc.). Correct CWE = faster triage routing.
- **Severity**: use the CVSS calculator built into the submission form — do not type the
  score manually; let H1 compute it from the vector.
- **Attachments**: upload raw HTTP requests as `.txt`, screenshots as `.png`. Never embed
  credentials in attachments even if they are test credentials.
- **Program median response time**: check before submitting. If median first response > 7
  days, set your calendar reminder for day 8 to follow up politely once.
- **Collaboration**: H1 allows inviting collaborators — use this for chain bugs where a
  co-researcher found part of the path.
- **Disclosure request**: do NOT request public disclosure until the program marks the
  report as Resolved or Informative. Premature requests annoy programs and reduce future
  bounty willingness.

### 5B. Bugcrowd

- **VRT (Vulnerability Rating Taxonomy)**: Bugcrowd uses its own VRT, not raw CVSS.
  Map your finding to the VRT tree (e.g., `Server-Side Injection > SQL Injection >
  Blind`). Incorrect VRT mapping is the #1 cause of Bugcrowd N/A decisions.
- **Target**: always specify the exact target from the program's target list, not a free-
  text URL. Mismatch → automatic out-of-scope.
- **Known issues list**: Bugcrowd programs often maintain a "known issues" list in the
  brief. Check it before submitting.
- **P1–P5 severity**: Bugcrowd maps VRT to P1 (critical) through P5 (informational).
  P1 = RCE, SQLi with data access, auth bypass to privileged function. P4/P5 = missing
  headers, self-XSS, open redirect on non-sensitive redirect targets.

### 5C. Intigriti

- **CVSS + business impact**: Intigriti triagers weigh business impact heavily alongside
  CVSS. Add a "Business Impact" section explicitly.
- **Scope matching**: Intigriti uses domain wildcards precisely. `*.example.com` does NOT
  include `example.com` itself — confirm the exact asset.
- **Response SLA**: Intigriti commits to a first response within 16 business hours for
  most programs. If no response after 24 hours, use the platform's nudge feature — do
  not email the company directly.
- **Hall of Fame only programs**: some Intigriti programs are non-monetary. Confirm bounty
  eligibility before investing deep research time.
- **CVE coordination**: Intigriti will coordinate CVE assignment for qualifying findings
  on request — mention this in the report if you want a CVE.

---

## Phase 6 — Disclosure Timeline Management

```
Day 0:   Submit report to platform
Day 1–3: Await triage acknowledgement
Day 7:   If no acknowledgement → send ONE polite follow-up via platform messaging
Day 14:  If no response → escalate within platform (H1: "Request mediation";
          Bugcrowd: contact support; Intigriti: use the nudge system)
Day 45:  If report is unacknowledged or "needs more info" stalled → send formal notice
          that you will disclose in 45 days unless resolved
Day 90:  Standard responsible disclosure window closes. Request public disclosure
          (or publish with your own write-up if the program is unresponsive, after
          notifying CERT/CC if the severity warrants it)

EXCEPTIONS:
  - Active exploitation in the wild → 7-day expedited timeline, notify CISA/CERT
  - Critical infrastructure (hospitals, water, power) → coordinate with ICS-CERT
    before any disclosure regardless of program response
  - Program asks for extension → grant ONE 30-day extension only; document it
```

### Disclosure Checklist Before Publishing

- [ ] Bounty paid or formal "won't fix / informational" decision received
- [ ] Program has marked the issue resolved (or 90 days elapsed)
- [ ] All real credentials / PII scrubbed from write-up
- [ ] No active exploitation path remains (verify fix deployed)
- [ ] CVE assigned and published if applicable
- [ ] Write-up reviewed: does it enable a copycat attack on an unpatched system?
      If yes, delay details until patch is widely deployed.

---

## Phase 7 — Post-Submission Triage Response

### Common Triage Outcomes and Counter-Responses

| Outcome | Likely Cause | Your Response |
|---------|-------------|---------------|
| "Duplicate" | Finding already known | Ask for original report # to confirm; if root cause differs, explain the distinction |
| "Out of scope" | Asset not in scope table | Provide the scope policy text that shows it IS in scope; or accept and move on |
| "Informational" | No demonstrated impact | Add a concrete impact PoC — show actual data exfil or account compromise |
| "Not reproducible" | Steps incomplete | Re-test yourself, attach a screen recording, simplify the steps |
| "Accepted / Need more info" | PoC works but impact unclear | Add the business impact section; quantify data exposed |
| Severity downgrade | CVSS disagreement | Provide the CVSS vector justification citing NVD precedent for same class |

---

## Phase 8 — Reputation & Program Relationship Rules

- One polite follow-up per milestone (triage, fix, payout). Never spam.
- Thank the triage team when a report is resolved, even for a low bounty — programs
  remember researchers who are professional.
- Never threaten disclosure as leverage for a higher bounty — this is extortion in most
  jurisdictions and will result in program banning.
- Build a track record on the platform before attempting P1 submissions — some programs
  require a signal score above a threshold to receive high-severity bounties.
- Chain bugs carefully: a P3 + P3 = P2 chain is legitimate; claiming P1 for a chain
  where each step requires unrealistic conditions will get you a downgrade and a note.
