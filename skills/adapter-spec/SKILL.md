---
id: adapter-spec
name: Adapter Interface Specification
description: |
  Interface contract for wrapping external security/pentest tools in the fleet-forge
  mega-app. Every adapter follows the same Invoke→Capture→Parse→NormalizedResult pipeline.
  Use when writing, reviewing, or debugging any tool adapter.
version: "1.0.0"
category: build-crew/shared
---

# Adapter Specification — fleet-forge mega-app

## Purpose

The mega-app wraps 100+ external tools (nmap, sqlmap, semgrep, ghidra, etc.) behind a
single uniform interface so the MCP layer can invoke any tool with the same call signature.
This skill defines that interface. Every adapter must satisfy it exactly.

---

## The Four-Stage Pipeline

```
Invoke(tool, args)
      │
      ▼
Capture(stdout, stderr, exit_code, duration_ms)
      │
      ▼
Parse(raw_stdout) → structured data
      │
      ▼
NormalizedResult{tool, exit_code, data, stderr, duration_ms}
```

---

## Go Interface (canonical)

```go
package adapter

import (
    "context"
    "encoding/json"
)

// Adapter is the interface every tool wrapper must satisfy.
type Adapter interface {
    // Invoke runs the external tool with the given args.
    // ctx carries the deadline and cancellation signal.
    // Must respect ctx.Done() — kill the subprocess if context is cancelled.
    Invoke(ctx context.Context, args map[string]string) (RawResult, error)

    // Parse converts raw stdout/stderr into structured JSON.
    // Must not fail on partial output (tool may have been killed mid-run).
    Parse(raw RawResult) (NormalizedResult, error)

    // Schema returns the adapter's metadata and argument schema.
    Schema() AdapterSchema
}

// RawResult is the direct output of a subprocess invocation.
type RawResult struct {
    Stdout     []byte
    Stderr     []byte
    ExitCode   int
    DurationMs int64
}

// NormalizedResult is the uniform output sent to the MCP layer.
type NormalizedResult struct {
    Tool       string          `json:"tool"`
    Version    string          `json:"version,omitempty"`
    ExitCode   int             `json:"exit_code"`
    Data       json.RawMessage `json:"data"`       // tool-specific structured output
    Stderr     string          `json:"stderr,omitempty"`
    DurationMs int64           `json:"duration_ms"`
    Truncated  bool            `json:"truncated,omitempty"` // true if output was capped
    Error      string          `json:"error,omitempty"`     // populated on Invoke error
}

// AdapterSchema describes the adapter for the MCP tool registry.
type AdapterSchema struct {
    ID          string            `json:"id"`          // e.g. "nmap"
    Binary      string            `json:"binary"`      // e.g. "nmap"
    ArgsTemplate map[string]Arg   `json:"args"`        // argument definitions
    OutputFmt   string            `json:"output_fmt"`  // "line-json" | "xml" | "tsv" | "custom"
    Category    string            `json:"category"`    // "recon" | "sqli" | "sast" | etc.
    License     string            `json:"license"`
    InstallCmd  string            `json:"install_cmd"`
}

type Arg struct {
    Flag     string `json:"flag"`      // CLI flag, e.g. "--target"
    Required bool   `json:"required"`
    Default  string `json:"default,omitempty"`
    Help     string `json:"help"`
}
```

---

## Tool Registry Format

The mega-app loads adapters from a registry at startup:

```json
{
  "id": "nmap",
  "binary": "nmap",
  "args_template": {
    "target": { "flag": "-oX - --target", "required": true, "help": "IP or CIDR" },
    "ports":  { "flag": "-p", "required": false, "default": "1-1024", "help": "Port range" },
    "timing": { "flag": "-T", "required": false, "default": "3", "help": "Timing (0-5)" }
  },
  "output_fmt": "xml",
  "category": "recon",
  "license": "NPSL/GPLv2",
  "install_cmd": "apt-get install -y nmap"
}
```

---

## Implementation Requirements

### 1. Context Cancellation (mandatory)

```go
func (a *NmapAdapter) Invoke(ctx context.Context, args map[string]string) (RawResult, error) {
    cmd := exec.CommandContext(ctx, "nmap", buildArgs(args)...)
    // exec.CommandContext automatically kills the process when ctx is cancelled.
    stdout, err := cmd.Output()
    // ...
}
```

`exec.CommandContext` is required — plain `exec.Command` is a contract violation.

### 2. Output Capping

No adapter may return more than **10MB of raw output** to the MCP layer. Cap and set
`Truncated: true`:

```go
const maxOutputBytes = 10 * 1024 * 1024

func capOutput(raw []byte) ([]byte, bool) {
    if len(raw) > maxOutputBytes {
        return raw[:maxOutputBytes], true
    }
    return raw, false
}
```

### 3. Partial-Output Tolerance

`Parse()` must not panic or return an error on partial output. Tools killed mid-run
produce truncated stdout. Parse what exists; mark the result `Truncated: true`.

### 4. Argument Sanitization

All args passed to `exec.Command` must be individual string arguments (not shell-interpolated).
Never construct a shell string and pass it to `sh -c`. This prevents command injection.

```go
// CORRECT
cmd := exec.CommandContext(ctx, "nmap", "-p", args["ports"], args["target"])

// WRONG — command injection surface
cmd := exec.CommandContext(ctx, "sh", "-c", "nmap -p "+args["ports"]+" "+args["target"])
```

### 5. Error vs. No-Results

Distinguish between:
- `exit_code: 0, data: []` → tool ran, found nothing (not an error)
- `exit_code: 1` → tool error (populate `error` field)
- `exit_code: 2` → usage/config error (populate `error` + `stderr`)

---

## Required Tests

Every adapter ships with exactly these three tests:

```go
// TestInvoke — happy path against a known-good target
func TestInvoke(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    a := NewNmapAdapter()
    raw, err := a.Invoke(ctx, map[string]string{"target": "127.0.0.1", "ports": "22,80"})
    require.NoError(t, err)
    assert.Equal(t, 0, raw.ExitCode)
}

// TestParse — fixture-based, no binary required
func TestParse(t *testing.T) {
    a := NewNmapAdapter()
    fixture, _ := os.ReadFile("testdata/nmap-output.xml")
    result, err := a.Parse(RawResult{Stdout: fixture, ExitCode: 0})
    require.NoError(t, err)
    assert.NotEmpty(t, result.Data)
    // Verify required JSON fields exist
    var data map[string]any
    require.NoError(t, json.Unmarshal(result.Data, &data))
    assert.Contains(t, data, "hosts")
}

// TestGovernorKill — context cancellation within 5s
func TestGovernorKill(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()
    a := NewNmapAdapter()
    _, err := a.Invoke(ctx, map[string]string{"target": "192.0.2.1", "ports": "1-65535"})
    // Should error due to context deadline, not hang
    assert.Error(t, err)
}
```

---

## MCP Exposure

The mega-app exposes each adapter as a single MCP tool:

```json
{
  "name": "invoke_nmap",
  "description": "Run nmap port scan. Returns NormalizedResult JSON.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "target": { "type": "string" },
      "ports":  { "type": "string", "default": "1-1024" },
      "timing": { "type": "string", "default": "3" }
    },
    "required": ["target"]
  }
}
```

The orchestrator (forge-architect) selects and chains tools by calling `invoke_<tool_id>`
MCP tools in sequence.

---

## Checklist — New Adapter

- [ ] Implements `Adapter` interface (Invoke + Parse + Schema)
- [ ] Uses `exec.CommandContext` (never bare `exec.Command`)
- [ ] Output capped at 10MB with `Truncated` flag
- [ ] Args passed as individual strings (no shell interpolation)
- [ ] `TestInvoke` + `TestParse` + `TestGovernorKill` pass
- [ ] `AdapterSchema` JSON complete (id, binary, args, output_fmt, category, license)
- [ ] Registered in `registry/registry.json`