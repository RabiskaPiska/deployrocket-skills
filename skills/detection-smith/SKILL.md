---
name: detection-smith
description: Detection engineering skill for Sigma rule authoring, YARA threat hunting, ATT&CK coverage mapping, AD/endpoint detection, FP tuning, purple-team validation with Atomic Red Team and MITRE Caldera, honeytoken/deception deployment, and ATR agent-layer detection. Use when writing or reviewing detection rules, building a detection-as-code pipeline, tuning alert fatigue, mapping ATT&CK coverage, authoring YARA signatures, or deploying honeytokens and canaries.
---

## Trigger

Use this skill when the user asks to:
- Write, review, convert, or validate Sigma rules
- Build or tune YARA signatures
- Map ATT&CK coverage and find detection gaps
- Reduce alert fatigue / tune false positives
- Run purple-team validation (Atomic Red Team, Caldera)
- Deploy honeytokens, canary accounts, or OpenCanary honeynet
- Detect AI-agent threats (ATR rules)
- Build a detection-as-code CI/CD pipeline

---

## Core Stack (all INSTALL-verified)

### Sigma Rule Corpus & Tooling
- **SigmaHQ/sigma** (`git clone https://github.com/SigmaHQ/sigma`) — 11,300+ rules, DRL 1.1
- **SigmaHQ/pySigma** (`pip install pySigma`) — Python conversion library, LGPL-2.1
- **sigma-cli** (`pip install sigma-cli`) — CLI conversion frontend
- **sigma-rule-validator** (`pip install sigma-rule-validator`) — schema + policy validation, Apache-2.0
- **pySigma-backend-splunk** (`pip install pySigma-backend-splunk`) — Splunk SPL, LGPL-2.1
- **pySigma-backend-elasticsearch** (`pip install pySigma-backend-elasticsearch`) — ES-QL/EQL, MIT
- **pySigma-pipeline-ocsf** (`pip install pySigma-pipeline-ocsf`) — OCSF normalization, MIT

### YARA
- **YARAHQ/yara-forge** — 12,455 production YARA rules (core/extended/full packages); Oct 2026 release
  - `curl -L https://github.com/YARAHQ/yara-forge/releases/download/20261026/yara-forge-rules-core.zip -o /tmp/yara-core.zip`
- **Neo23x0/yarGen-Go** (`git clone https://github.com/Neo23x0/yarGen-Go && go build -o yargen .`) — GPL-3.0

### Purple Validation
- **redcanaryco/atomic-red-team** (`git clone https://github.com/redcanaryco/atomic-red-team`) — 12,735 atomic tests, MIT
- **mitre/caldera** (`git clone --recursive https://github.com/mitre/caldera`) — adversary emulation, Apache-2.0

### ATR Agent-Layer Detection
- **Agent-Threat-Rule/agent-threat-rules** (`npm install agent-threat-rules`) — 784 Sigma-compatible AI rules, MIT
  - ⚠️ Repo was formerly listed without org; canonical is `Agent-Threat-Rule/agent-threat-rules`
- **ATR Scan GitHub Action**: `uses: Agent-Threat-Rule/atr-scan@v4.2.1`

### Deception & Honeytokens
- **thinkst/opencanary** (`pip install opencanary`) — multi-protocol honeynet, Apache-2.0
- **forgehk/honeytoken-sentinel** (`git clone https://github.com/forgehk/honeytoken-sentinel`) — Python honeytokens, MIT

### Supporting Data
- **elastic/detection-rules** — 2,941 detection rules, Elastic License 2.0
- **capture0x/ActiveDirectory-SAST** — 94 AD-specific Sigma rules (no license — use with caution)
- **dhurghamCreation/detection-engineering-arsenal** — CI-validated detection toolkit, MIT

---

## Sigma Rule Authoring Procedure

### Canonical Rule Template
```yaml
title: <Descriptive Title>
id: <UUID>
status: experimental  # experimental → test → stable
description: |
  One paragraph. What technique, what data source, why it matters.
author: <name>
date: YYYY-MM-DD
references:
  - https://attack.mitre.org/techniques/TXXXX.XXX/
logsource:
  category: process_creation   # or: security, powershell_script, dns_query, network_connection
  product: windows
detection:
  selection:
    CommandLine|contains|all:
      - '-EncodedCommand'
      - '-NoProfile'
  filter_legitimate:
    ParentImage|endswith: '\CcmExec.exe'
  condition: selection and not 1 of filter_*
falsepositives:
  - SCCM software deployment (filtered above)
  - <any other benign patterns>
level: high    # informational → low → medium → high → critical
tags:
  - attack.execution
  - attack.t1059.001
```

### Key Sigma Modifiers
| Modifier | Effect |
|---|---|
| `contains` | Case-insensitive substring |
| `startswith` / `endswith` | Prefix / suffix |
| `re` | Regex |
| `all` | All items must match |
| `windash` | Match `/` and `-` interchangeably |
| `base64` | Decode base64 then match |
| `cidr` | CIDR network match |

### FP Tuning Lifecycle (10-step)
1. Author at `level: informational`
2. Deploy logging-only for 7–30 days
3. Collect all matches; group by ParentImage, User, CommandLine
4. Classify each group: TP / benign-TP / FP
5. Calculate precision: TP / (TP + FP)
6. Write `falsepositives:` section from confirmed FP groups
7. Add `filter_` blocks for confirmed FP patterns
8. Promote to `level: medium` when precision > 70%
9. Promote to `level: high` when precision > 95% sustained 30 days
10. **Never mute** — tune; muting makes the technique invisible

### Conversion Commands
```bash
# Splunk SPL
sigma convert -t splunk -p splunk_windows rules/windows/

# Elasticsearch EQL
sigma convert -t es-ql -p ecs_windows rules/windows/

# OCSF-normalized Elastic
sigma convert -t es-ql -p ecs_windows,ocsf rules/cloud/

# Validate schema
sigma check --validation-config validation.yml rules/
sigma-rule-validator rules/
```

---

## AD Detection Map (Critical Event IDs)

| Event ID | Channel | Detection Use |
|---|---|---|
| 4624 | Security | PtH (type 3+NTLM), Golden Ticket |
| 4662 | Security | DCSync (DS-Replication GUIDs), LDAP enumeration |
| 4741 | Security | RBCD computer account creation |
| 4768 | Security | AS-REP roasting |
| 4769 | Security | Kerberoasting (EncType 0x17) |
| 4771 | Security | Password spray |
| 4886 | Security | ADCS ESC1 cert issuance |
| 5136 | Security | RBCD write, ACL manipulation, dMSA BadSuccessor |
| 4104 | PowerShell | Encoded commands, obfuscation |
| 1 | Sysmon | Process creation (primary) |
| 10 | Sysmon | LSASS access / credential dump |
| 22 | Sysmon | DNS query / C2 |

### Kerberoasting Rule (production example)
```yaml
title: Kerberoasting — RC4 TGS Request Spike
logsource:
  product: windows
  service: security
detection:
  selection:
    EventID: 4769
    TicketOptions: '0x40810000'
    TicketEncryptionType: '0x17'
  filter_computer:
    ServiceName|endswith: '$'
  filter_system:
    ServiceName|in:
      - 'krbtgt'
      - 'kadmin'
  condition: selection and not 1 of filter_*
level: high
tags:
  - attack.credential_access
  - attack.t1558.003
```
**Atomic validation:** `Invoke-AtomicTest T1558.003 -TestNumbers 1`

---

## Purple Validation Loop

```
ghost-walker executes technique
        ↓
detection-smith writes Sigma rule
        ↓
Invoke-AtomicTest T<ID> -TestNumbers 1
        ↓
Does rule fire? ── NO ──→ analyze logsource/field name mismatch → fix → retry
        ↓ YES
FP rate OK? ── NO ──→ add filter_* blocks → retest
        ↓ YES
Promote to stable → commit → update ATT&CK Navigator layer
```

---

## ATR Agent-Layer Rules

ATR (Agent Threat Rules) is to AI-agent detection what Sigma is to SIEM detection.

**Four ATR threat categories:**
- `T-INJECT-001/002` — prompt injection in user input / tool output
- `T-EXFIL-001/002` — system prompt exfiltration / credential leakage
- `T-POISON-001` — MCP server response poisoning
- `T-PIVOT-001` — agent used as lateral movement pivot

**Deploy ATR scan in CI:**
```yaml
- uses: Agent-Threat-Rule/atr-scan@v4.2.1
  with:
    scan-paths: '.opencode/skills/**,**.skill.md,mcp.json'
    fail-on: high
```

---

## Detection-as-Code CI Pipeline
```yaml
name: Validate Detection Rules
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install pySigma sigma-cli sigma-rule-validator
      - run: sigma-rule-validator rules/
      - run: sigma convert -t splunk -p splunk_windows rules/
      - run: sigma convert -t es-ql -p ecs_windows rules/
      - uses: Agent-Threat-Rule/atr-scan@v4.2.1
        with: { scan-paths: '.opencode/skills/**', fail-on: 'high' }
```

---

## References
- SigmaHQ/sigma: https://github.com/SigmaHQ/sigma (DRL 1.1, 11,300+ rules)
- redcanaryco/atomic-red-team: https://github.com/redcanaryco/atomic-red-team (MIT)
- Agent-Threat-Rule/agent-threat-rules: https://github.com/Agent-Threat-Rule/agent-threat-rules (MIT)
- YARAHQ/yara-forge: https://github.com/YARAHQ/yara-forge (MIT, Oct 2026 release verified)
- thinkst/opencanary: https://github.com/thinkst/opencanary (Apache-2.0)
- mitre/caldera: https://github.com/mitre/caldera (Apache-2.0)
- Detection-smith deep research report: crew-1-offensive/detection-smith-deep-research-2026-10-27.md