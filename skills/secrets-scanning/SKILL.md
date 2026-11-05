---
name: secrets-scanning
description: Secrets and credential leak detection in git history, source code, container images, and CI pipelines — TruffleHog for entropy + regex scanning of commits, Gitleaks for pre-commit and CI gates, detect-secrets baseline management, nuclei exposure templates for live endpoints. Use when auditing a repo for accidentally committed API keys, passwords, tokens, private keys, or connection strings; or when adding a pre-commit/CI gate to prevent future leaks.
---

## Trigger

Fire when: user says "secrets scan", "check for leaked keys", "api key in git", "trufflehog", "gitleaks", "credential leak", "sensitive data in code", "pre-commit secret check", or "secrets in history".

---

## Tool Decision Tree

```
Scanning git history (committed secrets)?
  YES → trufflehog git / gitleaks detect
Scanning a running container image?
  YES → trivy image --scanners secret
Scanning local filesystem (not git)?
  YES → gitleaks detect --no-git / trufflehog filesystem
Adding a pre-commit gate?
  YES → gitleaks + pre-commit hook
Scanning live endpoints for exposed .env / config files?
  YES → nuclei -t exposures/
Establishing a baseline to suppress known FPs?
  YES → detect-secrets baseline
```

---

## 1. TruffleHog — Deep Git History + Entropy

**Install:**
```bash
# Binary (recommended)
curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b /usr/local/bin

# Homebrew
brew install trufflesecurity/trufflehog/trufflehog

# Docker
docker pull trufflesecurity/trufflehog:latest
```

### Scan entire git history
```bash
# Full history scan (local repo)
trufflehog git file://. --only-verified

# Scan a remote repo (no clone needed)
trufflehog github --repo https://github.com/org/repo --only-verified

# Scan GitHub org
trufflehog github --org myorg --only-verified --concurrency 4

# Include unverified results too (more noise, more coverage)
trufflehog git file://. --include-detectors all
```

### Scan container image
```bash
# Scan all layers
trufflehog docker --image myapp:latest --only-verified

# Scan with output as JSON
trufflehog docker --image myapp:latest --json
```

### Scan filesystem (no git)
```bash
trufflehog filesystem /path/to/directory --only-verified --json
```

### CI integration (GitHub Actions)
```yaml
- name: TruffleHog Secrets Scan
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.repository.default_branch }}
    head: HEAD
    extra_args: --only-verified
```

---

## 2. Gitleaks — Fast Regex-Based Scanning

**Install:**
```bash
brew install gitleaks
# or
go install github.com/gitleaks/gitleaks/v8@latest
```

### Scan git history
```bash
# Detect secrets in full git history
gitleaks detect --source . --verbose

# Only scan commits since a ref
gitleaks detect --source . --log-opts="HEAD~50..HEAD"

# Output report
gitleaks detect --source . --report-path gitleaks-report.json --report-format json
```

### Scan staged files (pre-commit gate)
```bash
gitleaks protect --staged --verbose
```

### Pre-commit hook setup
```bash
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```
```bash
pre-commit install
```

### CI gate (GitHub Actions)
```yaml
- name: Run Gitleaks
  uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # required for org scans
```

### Custom rules (extend detection)
```toml
# .gitleaks.toml
[extend]
useDefault = true

[[rules]]
id = "custom-internal-api-key"
description = "Internal API key pattern"
regex = '''MYAPP_[A-Z0-9]{32}'''
tags = ["key", "internal"]
```

---

## 3. detect-secrets — Baseline + Low-FP Approach

**Install:** `pip install detect-secrets`

```bash
# Create baseline (scan current state, mark all as "known")
detect-secrets scan > .secrets.baseline

# Audit baseline to mark FPs as benign
detect-secrets audit .secrets.baseline

# Check for NEW secrets vs baseline (CI gate)
detect-secrets scan --baseline .secrets.baseline
# exits non-zero if new secrets found

# Commit baseline to repo
git add .secrets.baseline
```

---

## 4. trivy — Secret Scanning in Images + Repos

```bash
# Scan image for secrets (embedded in layers)
trivy image --scanners secret --no-progress myapp:latest

# Scan filesystem
trivy fs --scanners secret ./

# Scan IaC/config files for hardcoded secrets
trivy config --scanners secret ./infra/
```

---

## 5. nuclei — Live Endpoint Exposure Check

```bash
# Check for exposed .env, config files, backup files
nuclei -t exposures/ -target https://target.com

# Specific exposure templates
nuclei -t exposures/configs/ -target https://target.com     # .env, web.config, etc.
nuclei -t exposures/files/ -target https://target.com       # backup zips, sql dumps
nuclei -t exposures/tokens/ -target https://target.com      # exposed API tokens
```

Common live exposures found:
- `/.env` — Laravel/Node env files
- `/config.php`, `/config.bak` — PHP config backups
- `/.git/config` — exposed git config
- `/server-status` — Apache mod_status
- `/phpinfo.php` — PHP info pages

---

## Priority Secret Types to Always Check

| Type | Example patterns | Risk |
|------|-----------------|------|
| AWS credentials | `AKIA[A-Z0-9]{16}`, `aws_secret_access_key` | CRITICAL — full account takeover |
| GitHub tokens | `ghp_[A-Za-z0-9]{36}`, `ghs_`, `gho_` | CRITICAL — repo/org access |
| Stripe keys | `sk_live_[A-Za-z0-9]{24,}` | CRITICAL — payment fraud |
| Private keys | `-----BEGIN RSA PRIVATE KEY-----` | CRITICAL |
| Google API keys | `AIza[A-Za-z0-9\-_]{35}` | HIGH |
| Database URLs | `postgresql://user:pass@`, `mysql://` | HIGH |
| JWT secrets | `jwt_secret`, `JWT_SECRET=` in .env | HIGH |
| Slack tokens | `xoxb-`, `xoxp-`, `xoxa-` | MEDIUM |
| Generic passwords | `password=`, `passwd=`, `secret=` in config | MEDIUM |

---

## Recommended Multi-Layer Gate

```bash
#!/bin/bash
set -e
echo "=== gitleaks (git history) ==="
gitleaks detect --source . --exit-code 1

echo "=== trufflehog (verified only) ==="
trufflehog git file://. --only-verified --fail

echo "=== trivy secrets (filesystem) ==="
trivy fs --scanners secret --exit-code 1 ./

echo "All secrets gates passed."
```

---

## References

- `trufflesecurity/trufflehog` — https://github.com/trufflesecurity/trufflehog (17k+ stars, AGPL-3.0)
- `gitleaks/gitleaks` — https://github.com/gitleaks/gitleaks (18k+ stars, MIT)
- `Yelp/detect-secrets` — https://github.com/Yelp/detect-secrets (3k+ stars, Apache-2.0)
- `aquasecurity/trivy` — https://github.com/aquasecurity/trivy (Apache-2.0) — secrets scanner
- `projectdiscovery/nuclei-templates` — https://github.com/projectdiscovery/nuclei-templates (MIT) — exposures/
- Source reports: `PENTEST-AI-AGENT-SKILL-MINE-2026-10-26.md`, `config-smith-deep-research-2026-10-27.md`