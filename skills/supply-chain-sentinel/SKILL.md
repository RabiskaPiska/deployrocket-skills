---
name: supply-chain-sentinel
description: Supply-chain security skill covering SBOM generation (Syft/CycloneDX/SPDX), vulnerability scanning (Grype/OSV-Scanner), artifact signing (Sigstore/Cosign), GitHub Actions hardening (zizmor/actionlint/harden-runner), dependency auto-update (Renovate), malicious-package detection (depshield/OSSGadget), container admission (Trivy/Kyverno), SLSA provenance, OpenSSF Scorecard, and AI-skill auditing. Use when auditing dependencies, hardening CI/CD pipelines, generating SBOMs, signing artifacts, pinning Actions, detecting typosquatting or dependency confusion, or achieving SLSA/SSDF compliance.
---

## Trigger

Use this skill when the user asks to:
- Generate or diff SBOMs (CycloneDX, SPDX, VEX)
- Scan dependencies for known CVEs
- Sign or verify container images and build artifacts
- Harden GitHub Actions workflows (SHA pinning, egress control, PPE prevention)
- Detect malicious packages, typosquatting, or dependency confusion
- Auto-update dependencies (Renovate/Dependabot)
- Achieve SLSA Level 2/3 provenance
- Score a repo with OpenSSF Scorecard
- Audit Claude/OpenCode skills for prompt injection or over-permissive tool grants

---

## Core Stack (all INSTALL-verified, HTTP 200 confirmed 2026-10-28)

### SBOM Generation
| Tool | Install | License | Stars |
|---|---|---|---|
| **anchore/syft** | `curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh \| sh -s -- -b /usr/local/bin` | Apache-2.0 | 9,700 |
| **CycloneDX/cdxgen** | `npm install -g @cyclonedx/cdxgen` | Apache-2.0 | ~1,000 |
| **CycloneDX/cyclonedx-cli** | GitHub releases | Apache-2.0 | ~400 |
| **openvex/vexctl** | GitHub releases | Apache-2.0 | ~200 |
| **interlynk-io/sbomqs** | GitHub releases | Apache-2.0 | ~350 |

### Vulnerability Scanning
| Tool | Install | License | Stars |
|---|---|---|---|
| **anchore/grype** | `curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh \| sh -s -- -b /usr/local/bin` | Apache-2.0 | 13,128 |
| **google/osv-scanner** | `go install github.com/google/osv-scanner/v2/cmd/osv-scanner@latest` | Apache-2.0 | 11,300 |
| **pypa/pip-audit** | `pip install pip-audit` | Apache-2.0 | ~1,400 |

### Artifact Signing
| Tool | Install | License | Stars |
|---|---|---|---|
| **sigstore/cosign** | `curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64" && sudo install cosign-linux-amd64 /usr/local/bin/cosign` | Apache-2.0 | 6,400 |
| **sigstore/gitsign** | GitHub releases | Apache-2.0 | ~1,100 |
| **actions/attest-build-provenance** | `uses: actions/attest-build-provenance@v2` | MIT | official |

### GitHub Actions Hardening
| Tool | Install | License | Stars |
|---|---|---|---|
| **zizmorcore/zizmor** | `pip install zizmor` or `cargo install zizmor` | MIT | 6,808 |
| **rhysd/actionlint** | `winget install rhysd.actionlint` / `scoop install actionlint` | MIT | 4,344 |
| **step-security/harden-runner** | `uses: step-security/harden-runner@v2` | Apache-2.0 | ~1,300 |
| **mheap/pin-github-action** | `npm install -g pin-github-action` | MIT | 184 |

⚠️ **D-22 rename:** `woodruffw/zizmor` was transferred to `zizmorcore/zizmor` — the woodruffw URL redirects. Always use `zizmorcore/zizmor` as canonical. Version v1.7.0, MIT.

### Dependency Auto-Update
| Tool | Install | License | Stars |
|---|---|---|---|
| **renovatebot/renovate** | GitHub App at https://github.com/apps/renovate | AGPL-3.0 | 22,938 |
| **google/osv-scanner --fix** | (same binary, add `--fix` flag) | Apache-2.0 | — |

### Malicious Package Detection
| Tool | Install | License |
|---|---|---|
| **Gahrcoder/depshield** | GitHub releases / `go install` | MIT |
| **microsoft/OSSGadget** | GitHub releases | MIT |
| **SigmaHQ/sigma Axios rules** | `git clone SigmaHQ/sigma` → `rules-emerging-threats/2026/Malware/Axios-NPM-Compromise/` | DRL 1.1 |

### Container Admission & Scanning
| Tool | Install | License | Stars |
|---|---|---|---|
| **aquasecurity/trivy** | `curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \| sh -s -- -b /usr/local/bin` | Apache-2.0 | 38,800 |
| **kyverno/kyverno** | `helm install kyverno kyverno/kyverno -n kyverno --create-namespace` | Apache-2.0 | 8,300 |

### SLSA / SSDF
| Tool | Install | License | Stars |
|---|---|---|---|
| **slsa-framework/slsa-github-generator** | `uses: slsa-framework/slsa-github-generator/.github/workflows/...` | Apache-2.0 | ~1,400 |
| **ossf/scorecard** | `go install sigs.k8s.io/scorecard/v5@latest` | Apache-2.0 | 5,786 |

### AI Skill Auditing
| Tool | Install | License |
|---|---|---|
| **tarang-tj/claude-skill-audit** | `git clone https://github.com/tarang-tj/claude-skill-audit && npm install && npm run build` | MIT |
| **cisco-ai-defense/skill-scanner** | GitHub releases (Go binary) | Apache-2.0 |

---

## Key Procedures

### 1. Generate a CycloneDX SBOM
```bash
syft dir:. -o cyclonedx-json > sbom.cdx.json
syft dir:. -o spdx-json > sbom.spdx.json

# Score the SBOM quality
sbomqs score sbom.cdx.json
```

### 2. Scan for CVEs
```bash
# Scan SBOM with Grype
grype sbom:./sbom.cdx.json

# Scan lockfiles with OSV-Scanner
osv-scanner --lockfile package-lock.json
osv-scanner --lockfile requirements.txt

# Guided remediation (auto-fix)
osv-scanner --lockfile package-lock.json --fix
```

### 3. Sign a Release Artifact (keyless via Sigstore)
```bash
# Sign a blob
cosign sign-blob --yes --bundle artifact.bundle my-binary

# Verify
cosign verify-blob \
  --bundle artifact.bundle \
  --certificate-identity-regexp "https://github.com/myorg/myrepo/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  my-binary
```

### 4. Harden GitHub Actions Workflows

**Step 1: Audit with zizmor**
```bash
zizmor .github/workflows/          # finds unpinned actions, PPE risks, over-permissions
```

**Step 2: Pin all action SHAs**
```bash
GITHUB_TOKEN=<token> pin-github-action .github/workflows/
```

**Step 3: Add harden-runner as first step**
```yaml
steps:
  - uses: step-security/harden-runner@v2
    with:
      egress-policy: audit    # change to 'block' after baseline established
      allowed-endpoints: >
        api.github.com:443
        registry.npmjs.org:443
```

**Step 4: Validate with actionlint**
```bash
actionlint .github/workflows/*.yml
```

### 5. Detect Malicious Packages
```bash
# Pre-install typosquat/confusion check
depshield check package.json

# Analyze installed packages for suspicious behaviour
oss-find-squats --packages-file package.json   # OSSGadget

# Sigma rules for Axios-style compromise (add to SIEM):
# rules-emerging-threats/2026/Malware/Axios-NPM-Compromise/
```

### 6. Renovate Config (minimal safe starting point)
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "platformAutomerge": true,
  "packageRules": [
    {"matchUpdateTypes": ["patch","minor"], "automerge": true, "minimumReleaseAge": "3 days"},
    {"matchUpdateTypes": ["major"], "automerge": false}
  ]
}
```

### 7. Audit Claude/OpenCode Skills
```bash
# Scan for prompt injection, over-permissive tools, secret leaks in skill files
node dist/cli.js ~/.config/opencode
node dist/cli.js .opencode/skills/

# Cisco scanner (Go binary)
skill-scanner scan --path .opencode/skills/ --output sarif
```

---

## Priority Install Order

1. **harden-runner** — immediate CI runtime telemetry, zero config overhead
2. **syft** — SBOM foundation for everything downstream
3. **grype + osv-scanner** — CVE scanning against SBOM
4. **cosign** — artifact signing for releases
5. **renovate** — dependency auto-update (add `renovate.json` to each repo)
6. **zizmor + actionlint + pin-github-action** — CI/CD hardening (one-time pass)
7. **trivy** — container image scanning in CI
8. **scorecard** — repo health baseline
9. **tarang-tj/claude-skill-audit** — AI skill security gate

---

## Local Gap Checklist

| Gap | Risk | Fix |
|---|---|---|
| No SBOM in release pipeline | HIGH | Add `anchore/sbom-action@v0` to release workflow |
| Lockfiles not diffed on PRs | HIGH | Add `actions/dependency-review-action` |
| Actions not SHA-pinned | HIGH | `pin-github-action` + zizmor CI gate |
| No runner egress policy | MEDIUM | `harden-runner` egress-policy: audit → block |
| No malicious-package gate | MEDIUM | `depshield` pre-install hook on npm projects |
| No artifact signing | MEDIUM | `cosign sign-blob` in release workflow |
| AI skills not audited | MEDIUM | `claude-skill-audit` + `skill-scanner` quarterly |

---

## References
- supply-chain-sentinel deep research: crew-1-offensive/supply-chain-sentinel-deep-research-2026-10-27.md
- zizmorcore/zizmor (canonical): https://github.com/zizmorcore/zizmor (MIT, v1.7.0) — D-22 rename confirmed
- anchore/syft: https://github.com/anchore/syft (Apache-2.0, 9,700★)
- anchore/grype: https://github.com/anchore/grype (Apache-2.0, 13,128★ API-verified 2026-10-28)
- google/osv-scanner: https://github.com/google/osv-scanner (Apache-2.0, 11,300★)
- sigstore/cosign: https://github.com/sigstore/cosign (Apache-2.0, 6,400★)
- renovatebot/renovate: https://github.com/renovatebot/renovate (AGPL-3.0, 22,938★ API-verified 2026-10-28)
- ossf/scorecard: https://github.com/ossf/scorecard (Apache-2.0, 5,786★)
- Axios NPM compromise Sigma rules: https://github.com/SigmaHQ/sigma/tree/master/rules-emerging-threats/2026/Malware/Axios-NPM-Compromise/