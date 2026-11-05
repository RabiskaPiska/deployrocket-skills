---
id: resource-governor
name: Resource Governor — ≤90% CPU/RAM Contract
description: |
  The ≤90% CPU/RAM contract for all fleet-forge adapters and the orchestrator.
  Defines worker pool sizing (NumCPU-1), per-adapter memory caps, backpressure
  on output channels, and the TestGovernorKill enforcement protocol.
version: "1.0.0"
category: build-crew/shared
---

# Resource Governor — ≤90% CPU/RAM Contract

## The Contract (hard limits)

| Resource | Limit | Scope |
|----------|-------|-------|
| CPU sustained | ≤90% over any 30s window | Per host |
| RAM | ≤90% of adapter memory cap | Per adapter process |
| Worker goroutines | NumCPU - 1 (floor: 1) | Per pool |
| Output buffer | 10MB per invocation | Per adapter call |
| Context kill time | ≤5s after cancellation | Per invocation |

These are not guidelines. An adapter that violates any limit is a **build blocker**.

---

## Worker Pool Sizing

```go
import "runtime"

// NumCPU-1 is the universal rule.
// Rationale: leave 1 core free for the OS, scheduler, and MCP server overhead.
// This keeps sustained CPU below 90% under full load on all observed hardware.
func workerCount() int {
    n := runtime.NumCPU() - 1
    if n < 1 {
        n = 1 // single-core machine: still run, just serialized
    }
    return n
}
```

### Semaphore Pattern (use this, not goroutine-per-task unbounded)

```go
type WorkerPool struct {
    sem chan struct{}
    wg  sync.WaitGroup
    ctx context.Context
}

func NewWorkerPool(ctx context.Context) *WorkerPool {
    return &WorkerPool{
        sem: make(chan struct{}, workerCount()),
        ctx: ctx,
    }
}

func (p *WorkerPool) Submit(task func()) {
    p.wg.Add(1)
    go func() {
        defer p.wg.Done()
        select {
        case p.sem <- struct{}{}: // acquire slot
            defer func() { <-p.sem }() // release on exit
            task()
        case <-p.ctx.Done():
            // context cancelled before we got a slot — drop the task
        }
    }()
}

func (p *WorkerPool) Wait() { p.wg.Wait() }
```

---

## Per-Adapter Memory Caps

Memory caps are defined per adapter category. Adapters must not allocate beyond their cap.

| Category | RAM Cap | Rationale |
|----------|---------|-----------|
| recon (nmap, reconftw, katana) | 512 MB | Network I/O bound; minimal state |
| sqli (sqlmap, ghauri) | 256 MB | Payload lists held in memory |
| sast (semgrep, codeql) | 2048 MB | AST + data-flow graphs are large |
| re (ghidra headless) | 4096 MB | Ghidra JVM; large binary analysis |
| xss (dalfox, XSStrike) | 256 MB | Crawl state; bounded by URL queue |
| hash (hashcat) | 512 MB | GPU handles cracking; CPU wrapper light |
| mobile (MobSF) | 1024 MB | APK/IPA unpacking + static analysis |
| python-glue | 128 MB | Thin wrappers; no heavy state |

### Enforcing a Memory Cap in Go

```go
import "runtime/debug"

// SetAdapterMemCap sets a soft GC trigger at capMB.
// Not a hard OS limit — use for GC pressure tuning.
// For hard limits, run the adapter as a child process with cgroup or ulimit.
func SetAdapterMemCap(capMB int64) {
    debug.SetMemoryLimit(capMB * 1024 * 1024)
}
```

For adapters that spawn a subprocess (the common case), enforce via the OS:

```go
// Linux: set RLIMIT_AS on the child process
cmd := exec.CommandContext(ctx, "nmap", args...)
cmd.SysProcAttr = &syscall.SysProcAttr{
    // Pdeathsig ensures child dies if parent dies
    Pdeathsig: syscall.SIGKILL,
}
// Memory limit via cgroup v2 (if available) or ulimit is set in the deployment config
```

---

## Backpressure on Output Channels

When the orchestrator reads results from adapters via channels, the producer (adapter)
must block — never drop — when the consumer (MCP layer) is slow.

```go
// CORRECT — bounded channel, producer blocks on full channel
results := make(chan NormalizedResult, workerCount()*2) // small buffer, blocks when full

go func() {
    defer close(results)
    pool.Submit(func() {
        r, _ := adapter.Invoke(ctx, args)
        select {
        case results <- r:   // blocks if consumer is slow — backpressure applied
        case <-ctx.Done():   // cancelled — stop producing
        }
    })
    pool.Wait()
}()

// WRONG — unbounded channel, no backpressure, memory grows without limit
results := make(chan NormalizedResult, 100000) // ← DO NOT DO THIS
```

---

## TestGovernorKill — Mandatory for Every Adapter

Every adapter must pass this test. It is run by both app-engineer (during implementation)
and perf-tuner (during benchmark gate).

```go
func TestGovernorKill(t *testing.T) {
    // Give the adapter 100ms to start, then cancel
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    adapter := NewAdapterUnderTest()

    // Start a long-running invocation
    done := make(chan error, 1)
    go func() {
        _, err := adapter.Invoke(ctx, longRunningArgs())
        done <- err
    }()

    // Cancel after 100ms
    time.Sleep(100 * time.Millisecond)
    cancel()

    // Must complete within 5 seconds total (4.9s after cancel)
    select {
    case err := <-done:
        // Any error is fine — context.Canceled, DeadlineExceeded, or tool-specific
        // What matters is that it returned, not what it returned
        t.Logf("adapter stopped with: %v", err)
    case <-time.After(4900 * time.Millisecond):
        t.Fatal("GOVERNOR KILL FAILED: adapter did not stop within 5s of context cancellation")
    }
}
```

---

## CPU/RAM Measurement Under Load

perf-tuner runs this protocol to verify the contract under realistic conditions:

```bash
# 1. Start the adapter under sustained load (NumCPU-1 workers, 60 seconds)
go test -bench=BenchmarkAdapterSustained -benchtime=60s ./adapter/... &
BENCH_PID=$!

# 2. Sample CPU and RAM every 5 seconds for 30 seconds
for i in $(seq 1 6); do
    # Linux
    ps -p $BENCH_PID -o %cpu,rss --no-headers
    sleep 5
done

# 3. Kill bench, report max observed CPU% and RSS MB
kill $BENCH_PID

# PASS condition: max CPU% <= 90, max RSS <= cap_MB * 1024
```

On Windows, use:
```powershell
Get-Process -Id $pid | Select-Object CPU, WorkingSet64
```

---

## Deployment Config — Cgroup Limits (Linux VPS)

In the Docker Compose stack, each adapter container has resource limits:

```yaml
# docker-compose.yml snippet
services:
  megapp:
    deploy:
      resources:
        limits:
          cpus: '0.9'        # 90% of 1 core (scale per host)
          memory: 2G         # adjust per adapter category
        reservations:
          memory: 256M
```

---

## Checklist — New Adapter (resource contract)

- [ ] Worker pool uses `workerCount()` (NumCPU-1)
- [ ] Semaphore pattern used (not unbounded goroutines)
- [ ] `exec.CommandContext` used (subprocess killed on cancel)
- [ ] Output capped at 10MB
- [ ] `TestGovernorKill` passes (5s kill deadline)
- [ ] Memory cap documented in adapter's `Schema().Category`
- [ ] CPU/RAM measured under 30s sustained load by perf-tuner (≤90% both)