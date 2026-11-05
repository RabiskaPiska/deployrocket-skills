---
name: sast-code-quality
description: Static Application Security Testing and code quality gates — Semgrep for taint-tracking and custom rule authoring, CodeQL for interprocedural data-flow analysis, SARIF output processing and deduplication, secure coding guidelines for Python/Go/JavaScript/TypeScript, and CI/CD quality gate integration. Use when adding automated security scanning to a codebase, writing custom detection rules, processing scan results, or enforcing secure coding patterns before merge.
---

## Trigger

Fire when: user says "SAST", "static analysis", "semgrep", "codeql", "code quality gate", "taint tracking", "write a semgrep rule", "security scan", "SARIF", "secure coding", "detect vulnerability in code", "prevent SQL injection", or any request to statically audit source code for security or quality issues.

---

## Tool Decision Tree

```
Need fast, regex + AST pattern matching with custom rules?
  YES → semgrep (open-source rules or custom)
Need interprocedural data-flow / taint tracking across files?
  YES → codeql (database build + query)
Have SARIF output and need to process/deduplicate it?
  YES → sarif-parsing workflow below
Adding a pre-merge gate to a repo?
  YES → GitHub Actions patterns below
Want language-specific secure coding enforcement?
  YES → language rules section below
```

---

## 1. Semgrep — Fast AST-Pattern SAST

**Install:**
```bash
pip install semgrep
# or
brew install semgrep
```

### Scan with community rulesets
```bash
# Security-focused scan (highest signal:noise)
semgrep scan --config "p/security-audit" ./

# OWASP Top 10 rules
semgrep scan --config "p/owasp-top-ten" ./

# Language-specific rules
semgrep scan --config "p/python" ./
semgrep scan --config "p/javascript" ./
semgrep scan --config "p/typescript" ./
semgrep scan --config "p/go" ./

# Combined high-confidence security rules
semgrep scan \
  --config "p/security-audit" \
  --config "p/secrets" \
  --config "p/owasp-top-ten" \
  --error \          # exit non-zero on findings
  --json > results.json \
  ./src/
```

### Semgrep Pro — cross-file taint analysis
```bash
# Requires SEMGREP_APP_TOKEN
semgrep scan --pro --config "p/default" ./

# Cross-file taint (sources → sinks across modules)
semgrep scan --pro-languages --config "p/taint-analysis" ./
```

### Write a custom rule
```yaml
# rules/custom-sql-injection.yml
rules:
  - id: raw-sql-f-string
    patterns:
      - pattern: |
          cursor.execute(f"... {$VAR} ...")
      - pattern-not: |
          cursor.execute(f"... {'%s'} ...")
    message: >
      Potential SQL injection: f-string directly in cursor.execute().
      Use parameterized queries: cursor.execute("... %s ...", (var,))
    severity: ERROR
    languages: [python]
    metadata:
      cwe: CWE-89
      owasp: A03:2021 - Injection

  - id: hardcoded-secret-assignment
    pattern: |
      $KEY = "..."
    metavariable-regex:
      metavariable: $KEY
      regex: (?i)(password|secret|api_key|token|private_key)
    message: Possible hardcoded secret in variable $KEY
    severity: WARNING
    languages: [python, javascript, typescript, go]
```

```bash
# Test the rule against a file
semgrep --config rules/custom-sql-injection.yml test_file.py

# Run only custom rules
semgrep scan --config ./rules/ ./src/
```

### Common high-value Semgrep patterns

```yaml
# Flask debug mode in production
- id: flask-debug-true
  pattern: app.run(..., debug=True, ...)
  message: Flask debug mode must be disabled in production
  languages: [python]
  severity: ERROR

# eval() on user input (JS/TS)
- id: eval-user-input
  patterns:
    - pattern: eval($X)
    - pattern-not: eval("...")
  message: eval() on dynamic input is a code injection risk
  languages: [javascript, typescript]
  severity: ERROR

# Go: SQL string concatenation
- id: go-sql-concat
  pattern: db.Query("..." + $VAR + "...")
  message: String concatenation in SQL query. Use db.Query("... ?", var)
  languages: [go]
  severity: ERROR

# Missing CSRF protection (Express)
- id: express-no-csrf
  patterns:
    - pattern: app.$METHOD(...)
    - pattern-not: app.use(csrf())
  message: Ensure CSRF middleware is applied to state-changing routes
  languages: [javascript, typescript]
  severity: WARNING
```

---

## 2. CodeQL — Interprocedural Data-Flow Analysis

**Install:**
```bash
# GitHub CLI extension
gh extension install github/gh-codeql

# Or download CodeQL CLI bundle
# https://github.com/github/codeql-action/releases
```

### Build database and run queries
```bash
# Build database for JavaScript/TypeScript
codeql database create js-db \
  --language=javascript-typescript \
  --source-root ./src/

# Build for Python
codeql database create py-db \
  --language=python \
  --source-root ./

# Build for Go
codeql database create go-db \
  --language=go \
  --source-root ./

# Run the security suite
codeql database analyze js-db \
  --format=sarif-latest \
  --output=results.sarif \
  javascript-security-extended.qls

# Run security-experimental (wider, more FPs)
codeql database analyze js-db \
  --format=sarif-latest \
  --output=results-experimental.sarif \
  javascript-security-experimental.qls
```

### CodeQL in GitHub Actions (free for public repos)
```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'   # weekly on Monday

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write
    strategy:
      matrix:
        language: [javascript-typescript, python]
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: security-extended
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
        with:
          output: results/
          upload: true
```

### Custom CodeQL query (taint tracking example)
```ql
/**
 * @name SQL injection via request parameter
 * @kind path-problem
 * @severity error
 * @id custom/sql-injection
 */
import python
import semmle.python.dataflow.new.DataFlow
import semmle.python.dataflow.new.TaintTracking
import semmle.python.ApiGraphs

class SqlInjectionConfig extends TaintTracking::Configuration {
  SqlInjectionConfig() { this = "SqlInjectionConfig" }

  override predicate isSource(DataFlow::Node source) {
    // Flask request.args, request.form, request.json
    exists(API::Node req |
      req = API::moduleImport("flask").getMember("request") and
      source = req.getMember(["args", "form", "json", "values"]).getAUse()
    )
  }

  override predicate isSink(DataFlow::Node sink) {
    // cursor.execute() calls
    exists(DataFlow::CallCfgNode call |
      call.getFunction().(DataFlow::AttrRead).getAttributeName() = "execute" and
      sink = call.getArg(0)
    )
  }
}

from SqlInjectionConfig cfg, DataFlow::PathNode source, DataFlow::PathNode sink
where cfg.hasFlowPath(source, sink)
select sink.getNode(), source, sink,
  "SQL injection from $@", source.getNode(), "user-controlled input"
```

---

## 3. SARIF Processing

### Deduplicate and filter SARIF results
```bash
# Install sarif-tools
pip install sarif-tools

# Summary of findings
sarif summary results.sarif

# Filter to errors only
sarif filter --level error results.sarif > errors.sarif

# Diff two SARIF files (new findings only)
sarif diff baseline.sarif current.sarif > new-findings.sarif

# Convert SARIF to CSV
sarif ls results.sarif --output csv > findings.csv
```

### Upload SARIF to GitHub Code Scanning
```bash
# Via gh CLI
gh api \
  --method POST \
  repos/{owner}/{repo}/code-scanning/sarifs \
  --field commit_sha=$(git rev-parse HEAD) \
  --field ref=refs/heads/main \
  --field sarif=$(gzip -c results.sarif | base64 -w0) \
  --field tool_name="semgrep"
```

---

## 4. Secure Coding Patterns by Language

### Python
```python
# SQL: ALWAYS parameterized
conn.execute("SELECT * FROM users WHERE id = %s", (user_id,))  # ✅
conn.execute(f"SELECT * FROM users WHERE id = {user_id}")       # ❌

# Subprocess: avoid shell=True with user input
subprocess.run(["ls", user_path], shell=False)    # ✅
subprocess.run(f"ls {user_path}", shell=True)     # ❌ command injection

# Secrets: never hardcode
import os
API_KEY = os.environ["API_KEY"]    # ✅
API_KEY = "sk-abc123..."           # ❌

# File paths: validate before use
import pathlib
safe = pathlib.Path("/safe/base") / user_filename
safe.resolve().relative_to("/safe/base")          # raises ValueError on traversal ✅
```

### JavaScript/TypeScript
```typescript
// XSS: use textContent not innerHTML
element.textContent = userInput;          // ✅
element.innerHTML = userInput;             // ❌

// SQL (Node.js + pg): parameterized
await pool.query("SELECT * FROM users WHERE id = $1", [userId]);  // ✅
await pool.query(`SELECT * FROM users WHERE id = ${userId}`);     // ❌

// SSRF: validate URLs before fetch
import { URL } from 'url';
const url = new URL(userInput);
if (!['https:'].includes(url.protocol)) throw new Error('Invalid');
if (url.hostname === 'localhost') throw new Error('SSRF blocked');

// Prototype pollution: sanitize keys
if (key === '__proto__' || key === 'constructor') throw new Error();
```

### Go
```go
// SQL: parameterized only
db.QueryRow("SELECT name FROM users WHERE id = ?", id)    // ✅
db.QueryRow("SELECT name FROM users WHERE id = " + id)    // ❌

// Command injection: exec.Command with separate args
exec.Command("ls", "-la", userPath)                       // ✅
exec.Command("sh", "-c", "ls "+userPath)                  // ❌

// Path traversal: use filepath.Clean + prefix check
clean := filepath.Clean("/safe/base/" + userFile)
if !strings.HasPrefix(clean, "/safe/base/") {
    return errors.New("path traversal blocked")
}
```

---

## 5. CI Quality Gate (Combined)

```yaml
# .github/workflows/security-quality.yml
name: Security & Quality Gate
on: [pull_request]

jobs:
  semgrep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v1
        with:
          config: >
            p/security-audit
            p/owasp-top-ten
            p/secrets
          auditOn: push
          generateSarif: "1"
      - uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif

  codeql:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript-typescript
          queries: security-extended
      - uses: github/codeql-action/analyze@v3
```

---

## References

- `returntocorp/semgrep` — https://github.com/returntocorp/semgrep (10k+ stars, LGPL-2.1 for OSS rules)
- `github/codeql` — https://github.com/github/codeql (7k+ stars, MIT for queries)
- `microsoft/sarif-tools` — https://github.com/microsoft/sarif-tools (MIT)
- Semgrep Registry — https://semgrep.dev/r (community rules)
- CodeQL query packs — https://github.com/github/codeql/tree/main/javascript/ql/src/Security
- OWASP Secure Coding Cheat Sheet — https://cheatsheetseries.owasp.org
- Source reports: `PENTEST-AI-AGENT-SKILL-MINE-2026-10-26.md`, `Fullstack-agent-skills-catalog-V3.md`