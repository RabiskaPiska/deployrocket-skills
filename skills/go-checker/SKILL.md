# go-checker — Go Static Analysis and Quality Gate

## Frontmatter

```yaml
id: go-checker
name: go-checker
description: >
  Go static analysis and quality gate skill. Covers the full golangci-lint
  config for this fleet, staticcheck rules, govulncheck workflow, race detector
  usage, goroutine leak detection with goleak, context propagation patterns,
  error wrapping discipline (fmt.Errorf with %w), and interface design rules.
  Use when .go files exist, "go build fails", running golangci-lint, go test,
  doing a Go code review, or before merging a Go PR.
triggers:
  - ".go files"
  - "go build fails"
  - golangci-lint
  - go test
  - Go code review
  - goroutine leak
  - race detector
  - govulncheck
  - staticcheck
  - context propagation
  - error wrapping
  - interface design Go
```

---

## 1. Pre-flight — Always Run These First

```sh
go build ./...                   # must exit 0
go vet ./...                     # must exit 0
gofmt -l .                       # must print nothing (no unformatted files)
goimports -l .                   # must print nothing
```

Never call a Go change done until all four pass cleanly. Paste the real output.

---

## 2. Fleet golangci-lint Config

Save as `.golangci.yml` at the repo root. This is the canonical fleet config — do not strip linters without a documented reason.

```yaml
run:
  timeout: 5m
  go: "1.26"

linters:
  enable:
    # correctness
    - errcheck        # unhandled errors
    - govet           # go vet rules
    - staticcheck     # SA + ST + QF rules
    - gosec           # security issues (G* rules)
    - bodyclose       # http.Response.Body not closed
    - noctx           # http.Request without context
    - rowserrcheck    # sql.Rows.Err() not checked
    - sqlclosecheck   # sql.Rows/Stmt not closed
    # code style / quality
    - gofmt
    - goimports
    - revive          # replacement for golint
    - misspell
    - unconvert       # unnecessary type conversions
    - unparam         # params always receiving the same value
    - unused          # unused code
    - nolintlint      # bad //nolint directives
    # performance
    - prealloc        # slice preallocation opportunities
    - copylock        # passing locks by value
    - exportloopref   # loop variable capture (pre-Go 1.22)
    # goroutine safety
    - gocritic
    - godot           # comment punctuation

linters-settings:
  errcheck:
    check-type-assertions: true
    check-blank: true
  gosec:
    excludes:
      - G104   # allow errors in defer (handled case by case)
  revive:
    rules:
      - name: exported
      - name: var-naming
      - name: error-return
      - name: error-strings
      - name: increment-decrement
  staticcheck:
    checks: ["all"]

issues:
  exclude-rules:
    # test files may use t.Fatal without check
    - path: "_test\\.go"
      linters: [errcheck, gosec]
  max-same-issues: 0
  max-issues-per-linter: 0
```

Run:

```sh
golangci-lint run ./...
# or with a specific linter:
golangci-lint run --disable-all --enable errcheck ./...
```

---

## 3. staticcheck Rules Reference

Key SA (static analysis) rules to know:

| Rule | What it catches |
|---|---|
| SA1006 | Printf with dynamic format string |
| SA1019 | Use of deprecated API |
| SA4006 | Unused assigned variable |
| SA4009 | Function argument never used |
| SA5001 | Deferring Close before checking error |
| SA9003 | Empty body in if/else |
| ST1000 | Package doc comment style |
| QF1001 | Apply De Morgan's law simplification |

Run standalone:

```sh
staticcheck ./...
staticcheck -checks="all" ./...
```

---

## 4. govulncheck Workflow

```sh
# Install once
go install golang.org/x/vuln/cmd/govulncheck@latest

# Run on the module
govulncheck ./...

# In CI — fail on any vulnerability in called code
govulncheck -json ./... | jq '.findings[] | select(.osv != null)'
```

`govulncheck` only reports vulnerabilities in functions **actually called** by your code — not every transitive dep. A clean run means no reachable CVEs, not no CVEs in go.sum.

Run this before every release and after any `go get` that bumps a dependency.

---

## 5. Race Detector

```sh
# All tests with race detector
go test -race ./...

# A specific package
go test -race -count=1 ./internal/cache/...

# A specific test
go test -race -run TestConcurrentWrite ./...
```

**When to use:** any code that uses goroutines, channels, sync primitives, or shared state. The race detector has ~5–10% CPU overhead — acceptable in CI, not for production binaries.

**In CI:** always run `go test -race ./...`. A data race that passes without `-race` is a latent bug.

---

## 6. Goroutine Leak Detection with goleak

```sh
go get go.uber.org/goleak
```

```go
// In TestMain (catches leaks across the whole package):
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}

// Per test:
func TestMyFunc(t *testing.T) {
    defer goleak.VerifyNone(t)
    // ... test body
}
```

**Common leak causes:**
- Goroutine blocked on a channel send with no receiver
- `http.Server` started without a shutdown call
- Ticker not stopped (`defer tick.Stop()`)
- `context.WithCancel` cancel not called (`defer cancel()`)
- Database connection pool left open

**goleak options:**
```go
// Ignore known background goroutines from third-party libs
goleak.VerifyNone(t,
    goleak.IgnoreTopFunction("go.opencensus.io/stats/view.worker"),
)
```

---

## 7. Context Propagation Rules

**Rule 1:** `context.Context` is always the first parameter, never stored in a struct.

```go
// ✅ correct
func (s *Service) GetUser(ctx context.Context, id string) (*User, error)

// ❌ wrong — context stored in struct
type Service struct { ctx context.Context }
```

**Rule 2:** Propagate context to every blocking call — HTTP, DB, gRPC, file I/O.

```go
// ✅
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
rows, err := db.QueryContext(ctx, query, args...)

// ❌ — ignores cancellation / deadline
req, _ := http.NewRequest(...)
rows, err := db.Query(query, args...)
```

**Rule 3:** Add timeouts at the boundary, not deep in the stack.

```go
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()
```

**Rule 4:** Never pass `context.Background()` through a chain that already has a context — you lose the cancellation signal.

**Rule 5:** Use `context.WithValue` sparingly, only for request-scoped metadata (trace IDs, auth tokens), never for passing function arguments.

---

## 8. Error Wrapping Discipline

**Always wrap with `%w`** so callers can use `errors.Is` / `errors.As`:

```go
// ✅ correct
if err != nil {
    return fmt.Errorf("getting user %s: %w", id, err)
}

// ❌ loses the original error — callers cannot unwrap
return fmt.Errorf("getting user %s: %v", id, err)

// ❌ double-wrap (adds "error: " prefix)
return fmt.Errorf("error: %w", err)
```

**Sentinel errors** — define at package level, compare with `errors.Is`:

```go
var ErrNotFound = errors.New("not found")

// ✅ correct comparison
if errors.Is(err, ErrNotFound) { ... }

// ❌ string comparison breaks with wrapping
if err.Error() == "not found" { ... }
```

**Custom error types** — use a struct, implement `Error() string` and `Unwrap() error`:

```go
type ValidationError struct {
    Field   string
    Message string
    Err     error
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}

func (e *ValidationError) Unwrap() error { return e.Err }
```

---

## 9. Interface Design Rules

**Rule 1:** Interfaces belong in the package that **uses** them, not the package that implements them (accept interfaces, return structs).

```go
// ✅ — consumer defines the interface
package storage
type Reader interface { Read(ctx context.Context, id string) (*Record, error) }

// ❌ — implementer defines it, forcing the consumer to import
package db
type DBReader interface { ... }
```

**Rule 2:** Keep interfaces small — prefer single-method interfaces.

```go
// ✅
type Stringer interface { String() string }
type Reader   interface { Read(p []byte) (n int, err error) }

// ❌ — fat interface forces large mocks
type AllThingsDB interface {
    Query(...) ...
    Exec(...) ...
    Begin(...) ...
    Close() ...
}
```

**Rule 3:** Embed to compose, not to inherit.

```go
type ReadWriter interface {
    Reader
    Writer
}
```

**Rule 4:** Never use an interface with only one concrete implementation for DI unless you need testability. A real type is simpler and the compiler gives better errors.

---

## 10. Common go test Flags

```sh
go test ./...                    # all packages
go test -v ./...                 # verbose output
go test -count=1 ./...           # disable caching (fresh run)
go test -run TestName ./...      # specific test by regex
go test -bench=. ./...           # run benchmarks
go test -benchmem ./...          # include memory allocation stats
go test -coverprofile=cov.out ./... && go tool cover -html=cov.out
go test -timeout 60s ./...       # custom timeout (default 10m)
```

**In CI, always run:**
```sh
go test -race -count=1 -timeout 120s ./...
```

---

## 11. Pre-Merge Gate Checklist

Run in order; all must pass before merging:

- [ ] `go build ./...` — exit 0
- [ ] `go vet ./...` — exit 0
- [ ] `gofmt -l .` — empty output
- [ ] `goimports -l .` — empty output
- [ ] `golangci-lint run ./...` — zero issues
- [ ] `staticcheck -checks=all ./...` — zero issues
- [ ] `go test -race -count=1 ./...` — all pass
- [ ] `govulncheck ./...` — no reachable CVEs
- [ ] goleak: no leaked goroutines in tests
- [ ] All error returns wrapped with `%w`
- [ ] All blocking calls receive a `context.Context`
- [ ] `defer cancel()` present for every `WithCancel` / `WithTimeout`

---

gates — godmode:activation: fired | /honestscore: skipped: skill file, not code | /ideas: skipped: skill authoring, no defect | brain: skipped: subagent seat, no brain access
