---
name: cloud-config-audit
description: Cloud and container infrastructure misconfiguration auditing — trivy for container/IaC scanning, checkov for Terraform/Helm/Dockerfile policy checks, nuclei cloud-misconfiguration templates, kubectl-score for k8s manifest grading, kube-bench CIS benchmark, AWS Config/GCP Security Command Center integration patterns. Use when scanning container images, IaC files, Kubernetes manifests, Helm charts, or cloud account configs for security misconfigurations before deploy or during a security review.
---

## Trigger

Fire when: user says "scan image", "audit IaC", "check Terraform", "k8s security", "Docker misconfiguration", "CIS benchmark", "cloud security posture", "misconfiguration scan", or any request to check infrastructure-as-code for security issues.

---

## Tool Decision Tree

```
Is the target a container image or filesystem?
  YES → trivy image / trivy fs
Is the target Terraform, CloudFormation, Helm, Dockerfiles, k8s YAML?
  YES → checkov
Is the target a live Kubernetes cluster?
  YES → kube-bench (CIS) + kubectl-score
Is the target a live cloud account (AWS/GCP/Azure)?
  YES → trivy aws / nuclei cloud templates
Are you doing a web endpoint audit for cloud exposures?
  YES → nuclei -t cloud/
```

---

## 1. trivy — Container & IaC Scanner

**Install:** `brew install trivy` or `apt install trivy` or Docker:
```bash
docker pull aquasec/trivy:latest
```

### Scan a container image
```bash
# Full scan: CVEs + secrets + misconfigurations
trivy image --scanners vuln,secret,misconfig nginx:latest

# Exit non-zero if HIGH or CRITICAL findings
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:prod

# Output SARIF for CI upload
trivy image --format sarif --output trivy-results.sarif myapp:prod
```

### Scan IaC files (Terraform, Helm, Dockerfile, k8s YAML)
```bash
# Recursive scan of a directory
trivy config ./infra/

# Scan specific file types
trivy config --include-non-failures --format table ./terraform/

# Helm chart
trivy config ./charts/myapp/
```

### Scan live AWS account
```bash
# Requires AWS credentials in environment
trivy aws --region us-east-1 --service s3 --service iam --service ec2

# Check specific account for public buckets
trivy aws --region us-east-1 --service s3 --format table
```

### Key misconfig checks trivy covers
- S3 public ACLs and bucket policies
- Security groups with 0.0.0.0/0 ingress
- Unencrypted EBS/RDS volumes
- IAM roles with wildcard permissions
- Dockerfile: running as root, no HEALTHCHECK, ADD instead of COPY
- k8s: privileged containers, hostPID, no resource limits, no securityContext

---

## 2. checkov — Policy-as-Code for IaC

**Install:** `pip install checkov`

### Scan Terraform
```bash
# Scan entire Terraform directory
checkov -d ./terraform/ --framework terraform

# Check only HIGH severity
checkov -d ./terraform/ --check HIGH --compact

# Output JUnit XML for CI
checkov -d ./terraform/ -o junitxml > checkov-results.xml

# Skip specific checks
checkov -d ./terraform/ --skip-check CKV_AWS_18,CKV_AWS_20
```

### Scan Kubernetes manifests
```bash
checkov -d ./k8s/ --framework kubernetes --compact

# Common k8s findings:
# CKV_K8S_11 - CPU limits not set
# CKV_K8S_12 - Memory limits not set
# CKV_K8S_13 - Memory requests not set
# CKV_K8S_28 - Privileged container
# CKV_K8S_30 - Run as non-root
# CKV_K8S_37 - Minimize admission of containers with added capabilities
```

### Scan Dockerfile
```bash
checkov -f Dockerfile --framework dockerfile
# CKV_DOCKER_1  - Ensure port 22 is not exposed
# CKV_DOCKER_2  - Ensure that HEALTHCHECK instructions are used
# CKV_DOCKER_3  - Ensure that /bin/sh -c is not used
# CKV_DOCKER_4  - Ensure that COPY is used instead of ADD
# CKV_DOCKER_5  - Ensure update instructions are not used alone
# CKV_DOCKER_7  - Ensure the base image uses a non latest version tag
# CKV_DOCKER_8  - Ensure the last USER is not root
```

### CI gate pattern
```yaml
# .github/workflows/iac-security.yml
- name: Run Checkov
  uses: bridgecrewio/checkov-action@master
  with:
    directory: terraform/
    framework: terraform
    soft_fail: false
    output_format: sarif
    output_file_path: results/checkov.sarif
```

---

## 3. kube-bench — CIS Kubernetes Benchmark

**Install:**
```bash
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro \
  -v $(which kubectl):/usr/local/mount-from-host/bin/kubectl \
  -e KUBECONFIG=$KUBECONFIG \
  --rm aquasec/kube-bench:latest \
  --version 1.29
```

**Key CIS controls checked:**
- 1.x: Master node config (API server flags, etcd, controller manager)
- 2.x: etcd node config
- 3.x: Control plane config
- 4.x: Worker node config (kubelet, proxy)
- 5.x: Kubernetes policies (RBAC, network policies, pod security standards)

---

## 4. nuclei — Cloud Template Scanning

```bash
# Install
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest

# Run cloud misconfiguration templates
nuclei -t cloud/ -target https://target.com

# Specific cloud checks
nuclei -t cloud/aws/ -target https://s3.amazonaws.com/bucket-name/
nuclei -t cloud/gcp/ -target https://storage.googleapis.com/bucket/

# Config/exposure checks
nuclei -t exposures/ -target https://target.com
nuclei -t misconfiguration/ -target https://target.com
```

---

## Severity Priority Matrix

| Finding | Tool | Severity | Action |
|---------|------|----------|--------|
| Public S3 bucket / GCS bucket | trivy aws / nuclei | CRITICAL | Block deploy |
| Container running as root | trivy / checkov | HIGH | Fix before prod |
| No resource limits in k8s | checkov | MEDIUM | Fix in sprint |
| EXPOSE 22 in Dockerfile | checkov | HIGH | Block deploy |
| Wildcard IAM permissions | trivy / checkov | HIGH | Block deploy |
| etcd not encrypted | kube-bench | CRITICAL | Immediate fix |
| No network policy in k8s | checkov | MEDIUM | Fix in sprint |
| Image uses latest tag | trivy / checkov | LOW | Warn only |

---

## CI Gate Pattern (full)

```bash
#!/bin/bash
set -e

echo "=== trivy image scan ==="
trivy image --exit-code 1 --severity CRITICAL,HIGH --no-progress "$IMAGE_TAG"

echo "=== checkov IaC scan ==="
checkov -d ./terraform/ --framework terraform --compact --quiet
checkov -d ./k8s/ --framework kubernetes --compact --quiet

echo "=== trivy config scan ==="
trivy config --exit-code 1 --severity HIGH,CRITICAL ./

echo "All gates passed."
```

---

## References

- `aquasecurity/trivy` — https://github.com/aquasecurity/trivy (24k+ stars, Apache-2.0)
- `bridgecrewio/checkov` — https://github.com/bridgecrewio/checkov (7k+ stars, Apache-2.0)
- `aquasecurity/kube-bench` — https://github.com/aquasecurity/kube-bench (7k+ stars, Apache-2.0)
- `projectdiscovery/nuclei-templates` — https://github.com/projectdiscovery/nuclei-templates (13,224 stars, MIT, 2026-10-26)
- Source reports: `PENTEST-AI-AGENT-SKILL-MINE-2026-10-26.md`, `config-smith-deep-research-2026-10-27.md`