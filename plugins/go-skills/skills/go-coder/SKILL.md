---
name: go-coder
description: "Use when writing, reviewing, or refactoring Go code in production services — error handling, concurrency, interfaces, testing, and resilience patterns."
user_invocable: true
---

# Go Coder

A practical reference for production-grade Go, organized around the real-world development workflow. Every pattern cites one of five canonical sources:

- **[GoBook]** — *The Go Programming Language*, Donovan & Kernighan
- **[LearningGo]** — *Learning Go*, 2nd Ed., Jon Bodner
- **[ConcurrencyInGo]** — *Concurrency in Go*, Katherine Cox-Buday
- **[Mistakes]** — *100 Go Mistakes and How to Avoid Them*, Teiva Harsanyi
- **[CloudNative]** — *Cloud Native Go*, Matthew Titmus

---

## 1. Types & Interfaces — [GoBook] Ch.6-7, [LearningGo] Ch.6-8, [Mistakes] #5-#8

### Accept Interfaces, Return Structs

Define interfaces on the **consumer** side, not the producer. Return concrete types so callers get the full API.

```go
// DO: function accepts interface, returns concrete struct
type Storer interface {
    Save(ctx context.Context, id string, v any) error
}

func NewCache(s Storer) *Cache { return &Cache{store: s} }  // accepts interface
func NewServer(cfg Config) *Server { return &Server{cfg: cfg} }  // returns struct

// Compile-time interface check
var _ Storer = (*redisStore)(nil)
```

### Keep Interfaces Small

1-3 methods. If larger, split by consumer need. `[Mistakes] #5`

```go
type Reader interface { Read(p []byte) (n int, err error) }  // 1 method — perfect
type ReadWriter interface { Reader; Writer }                  // compose, don't bloat
```

### Implicit Implementation — Design at the Consumer

Don't define an interface in the same package as its implementation. Define it where it's used.

```go
// internal/store/store.go — defines concrete type
type RedisStore struct { ... }
func (r *RedisStore) Save(...) error { ... }

// internal/cache/cache.go — defines interface it needs
type Storer interface { Save(ctx context.Context, id string, v any) error }
type Cache struct { store Storer }
```

### Embedding for Composition

```go
type LoggedStore struct {
    Storer                // embed interface, not struct
    log *slog.Logger
}

func (l *LoggedStore) Save(ctx context.Context, id string, v any) error {
    l.log.Info("save", "id", id)
    return l.Storer.Save(ctx, id, v)
}
```

### Generics — [LearningGo] Ch.8

Use for type-safe containers and algorithms (same logic, different types). Avoid for behavior differences — use interfaces there.

```go
// Named constraint — prefer over inline unions
type Number interface { ~int | ~int64 | ~float64 }

func Sum[T Number](vals []T) T {
    var total T
    for _, v := range vals { total += v }
    return total
}

// Methods cannot have type parameters — use functions
// WRONG: func (s *Store) Get[T any](id string) T
// RIGHT: func Get[T any](s *Store, id string) T
```

---

## 2. Error Handling — [GoBook] Ch.5, [LearningGo] Ch.9, [Mistakes] #48-#53

### %w vs %v

```go
// DO: %w preserves chain — errors.Is/errors.As work on callers
return fmt.Errorf("query user %s: %w", id, err)

// DON'T: %v breaks chain — callers cannot inspect cause
return fmt.Errorf("query user: %v", err)  // errors.Is fails upstream
```

### errors.Is / errors.As

```go
// errors.Is: value/identity check
if errors.Is(err, ErrNotFound) {
    return http.StatusNotFound, nil
}

// errors.As: type extraction — gets structured data from error
var ve *ValidationError
if errors.As(err, &ve) {
    return http.StatusBadRequest, ve.Fields
}
```

### Sentinel vs Custom Error Type

```go
// Sentinel: when callers only need to identify the error
var ErrNotFound = errors.New("not found")
var ErrConflict = errors.New("conflict")

// Custom type: when callers need structured data from the error
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Message)
}
```

### errors.Join — [Mistakes] #50 (Go 1.20+)

```go
// Combine multiple independent errors
var errs []error
if err := validateName(name); err != nil { errs = append(errs, err) }
if err := validateEmail(email); err != nil { errs = append(errs, err) }
if err := errors.Join(errs...); err != nil { return err }
```

### Goroutine Error Collection

Never let goroutines silently swallow errors. Use `errgroup` for parallel tasks that cancel on first error (see Section 4 for full `errgroup` vs `WaitGroup` comparison).

### Panic — Programmer Bugs Only

```go
// DO: panic for invariant violations and nil callbacks
func NewWorker(fn func()) *Worker {
    if fn == nil { panic("fn must not be nil") }
    return &Worker{fn: fn}
}

// DON'T: panic for recoverable errors (I/O, validation, network)
```

---

## 3. Context — [GoBook] Ch.8, [LearningGo] Ch.12, [Mistakes] #60-#63

### Rules

- Always first parameter, named `ctx`. Never store in a struct. `[Mistakes] #60`
- Always `defer cancel()` immediately after `WithCancel`/`WithTimeout`/`WithDeadline`. `[Mistakes] #62`

```go
// DO
func (r *Repo) FindUser(ctx context.Context, id string) (*User, error)

// DON'T — context in struct prevents per-call cancellation
type Client struct { ctx context.Context }
```

### WithTimeout vs WithDeadline vs WithCancel

```go
// WithTimeout: I/O, external calls — relative duration
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

// WithDeadline: absolute time (e.g., SLA boundary)
ctx, cancel := context.WithDeadline(parent, deadline)
defer cancel()

// WithCancel: lifecycle control — cancel when work is done
ctx, cancel := context.WithCancel(parent)
defer cancel()
go worker(ctx)
```

### Context Values — Unexported Key Types

```go
// DO: unexported key type prevents collision across packages
type ctxKey string
const requestIDKey ctxKey = "requestID"

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}
func RequestIDFrom(ctx context.Context) (string, bool) {
    v, ok := ctx.Value(requestIDKey).(string)
    return v, ok
}

// DON'T: string key collides with other packages
ctx = context.WithValue(ctx, "requestID", id)  // BAD
```

Context values are for request-scoped data only: trace IDs, auth tokens, deadlines. Not for optional function parameters.

### Checking Cancellation

```go
// In loops and long operations
for {
    select {
    case <-ctx.Done(): return ctx.Err()
    case item := <-queue: process(item)
    }
}

// Before expensive operations
if ctx.Err() != nil { return ctx.Err() }
result, err := expensiveOp(ctx)
```

---

## 4. Concurrency — [GoBook] Ch.8-9, [ConcurrencyInGo] Ch.2-5, [Mistakes] #57-#73

### Goroutine Lifecycle — [ConcurrencyInGo] Ch.4

Never start a goroutine without knowing when it exits. The function that starts a goroutine owns its lifecycle.

```go
func startWorker(ctx context.Context, jobs <-chan Job) {
    go func() {
        for {
            select {
            case <-ctx.Done(): return       // exits cleanly on cancellation
            case j, ok := <-jobs:
                if !ok { return }           // exits when channel closed
                process(j)
            }
        }
    }()
}
```

### Pipeline Pattern — [ConcurrencyInGo] Ch.4

Chain stages with channels. Each stage reads from upstream and writes to downstream. Always propagate cancellation.

```go
func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case <-ctx.Done(): return
            case out <- n:
            }
        }
    }()
    return out
}

func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case <-ctx.Done(): return
            case out <- n * n:
            }
        }
    }()
    return out
}
```

### Fan-out / Fan-in — [ConcurrencyInGo] Ch.4

```go
// Fan-out: distribute work to N workers
func fanOut(ctx context.Context, in <-chan Work, n int) []<-chan Result {
    channels := make([]<-chan Result, n)
    for i := range n { channels[i] = process(ctx, in) }  // Go 1.22+: range over integer
    return channels
}

// Fan-in: merge N result channels into one
func merge(ctx context.Context, channels ...<-chan Result) <-chan Result {
    var wg sync.WaitGroup
    out := make(chan Result)
    forward := func(c <-chan Result) {
        defer wg.Done()
        for r := range c {
            select {
            case <-ctx.Done(): return
            case out <- r:
            }
        }
    }
    wg.Add(len(channels))
    for _, c := range channels { go forward(c) }
    go func() { wg.Wait(); close(out) }()
    return out
}
```

### errgroup vs sync.WaitGroup — [Mistakes] #71

| Use `errgroup` | Use `WaitGroup + chan error` |
|---|---|
| Independent tasks — first error cancels all | Worker pool that must drain jobs even after error |
| Parallel fetch where any failure aborts | Pipeline stages where stopping early blocks senders |

```go
// errgroup: parallel API calls — first error cancels all
g, ctx := errgroup.WithContext(ctx)
for _, url := range urls {
    url := url  // capture range variable (required in Go < 1.22)
    g.Go(func() error { return fetch(ctx, url) })
}
if err := g.Wait(); err != nil { return err }

// WaitGroup: worker pool — must drain all jobs
errCh := make(chan error, len(jobs))
var wg sync.WaitGroup
for range workers {  // Go 1.22+: range over integer
    wg.Add(1)
    go func() {
        defer wg.Done()
        for job := range jobs {
            if err := process(job); err != nil {
                select { case errCh <- err: default: }
            }
        }
    }()
}
go func() { wg.Wait(); close(errCh) }()
// caller: for err := range errCh { ... }  // drain errors after goroutines finish
```

### Mutex vs Channel — [ConcurrencyInGo] Ch.2

| **Mutex** | **Channel** |
|---|---|
| Protecting shared state (map, counter, cache) | Transferring data ownership between goroutines |
| Short critical sections | Signaling events, coordinating pipeline stages |

### Data Race Detection — [Mistakes] #58

Always run tests with `-race`. Races are undefined behavior — they cannot be safely ignored.

```bash
go test -race ./...
go run -race main.go
```

### Goroutine Leak Prevention — [Mistakes] #63

```go
// LEAK: unbuffered channel, nobody reads → goroutine blocked forever
ch := make(chan int)
go func() { ch <- compute() }()

// OK: buffered — goroutine can complete even if nobody reads immediately
ch := make(chan int, 1)
go func() { ch <- compute() }()

// OK: context cancellation
go func() {
    select {
    case ch <- compute():
    case <-ctx.Done():
    }
}()
```

### sync.Once / sync.Pool

```go
// sync.Once: one-time initialization (thread-safe)
var (
    once     sync.Once
    instance *DB
)
func GetDB() *DB {
    once.Do(func() { instance = mustConnect() })
    return instance
}

// sync.Pool: reduce allocations for frequently created/destroyed objects
var pool = sync.Pool{New: func() any { return new(bytes.Buffer) }}

func process(data []byte) string {
    buf := pool.Get().(*bytes.Buffer)
    defer func() { buf.Reset(); pool.Put(buf) }()
    buf.Write(data)
    return buf.String()
}
```

---

## 5. Service Design — [CloudNative] Ch.2-5, [LearningGo] Ch.14, [GoBook] Ch.10

### Project Structure

```
cmd/        — binary entry points (one directory per binary)
internal/   — private packages, not importable outside this module
pkg/        — public libraries safe for external import
```

```go
// cmd/server/main.go — wire everything together, start server
// internal/user/     — domain logic: User, UserStore, UserService
// internal/http/     — HTTP handlers, middleware, routing
// pkg/middleware/    — reusable middleware (could be used by other projects)
```

### Package Naming — [GoBook] Ch.10

- Lowercase, singular, no underscores
- No stutter: package name should not repeat in exported names

```go
package user   // NOT users, NOT userPkg

// DO: user.Record, user.Store, user.New
// DON'T: user.UserRecord, user.UserStore, user.NewUser
```

### HTTP Handlers — Dependencies via Closure or Receiver

```go
// DO: dependencies injected via receiver
type UserHandler struct {
    store UserStore
    log   *slog.Logger
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    u, err := h.store.FindByID(r.Context(), id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            http.Error(w, "not found", http.StatusNotFound)
            return
        }
        http.Error(w, "internal error", http.StatusInternalServerError)
        return
    }
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(u)
}

// DON'T: global state in handlers
var globalDB *sql.DB  // BAD: untestable, not swappable
```

### Middleware — [CloudNative]

```go
// Standard signature: func(http.Handler) http.Handler
func Logging(log *slog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            next.ServeHTTP(w, r)
            log.Info("request",
                "method", r.Method,
                "path", r.URL.Path,
                "duration", time.Since(start),
            )
        })
    }
}

// Chain middleware
mux := http.NewServeMux()
handler := Logging(log)(Recovery()(mux))
```

### Functional Options vs Config Struct — [LearningGo]

```go
// Functional Options: when most params are optional
type Option func(*Server)
func WithPort(p uint16) Option      { return func(s *Server) { s.port = p } }
func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080, timeout: 30 * time.Second}
    for _, o := range opts { o(s) }
    return s
}
// Usage: NewServer(WithPort(9090), WithTimeout(60*time.Second))

// Config Struct: when settings come from file/env or >3 required fields
type Config struct {
    Host    string        `yaml:"host"`
    Port    uint16        `yaml:"port"`
    Timeout time.Duration `yaml:"timeout"`
}
func New(cfg Config) *Server { return &Server{cfg: cfg} }
```

### Constructor Naming

```go
func NewServer(opts ...Option) *Server       // primary constructor
func NewServerFromConfig(cfg Config) *Server // alternative source
// Return (T, error) when construction can fail; T when it cannot
func NewPool(cfg Config) (*Pool, error)
```

---

## 6. Testing — [LearningGo] Ch.13, [Mistakes] #82-#84

### Table-Driven Tests with t.Run

```go
func TestParseUser(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    *User
        wantErr bool
    }{
        {"valid", `{"name":"alice"}`, &User{Name: "alice"}, false},
        {"empty name", `{"name":""}`, nil, true},
        {"invalid json", `{bad}`, nil, true},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseUser([]byte(tt.input))
            if (err != nil) != tt.wantErr {
                t.Fatalf("ParseUser() error = %v, wantErr %v", err, tt.wantErr)
            }
            if !reflect.DeepEqual(got, tt.want) {
                t.Errorf("ParseUser() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### t.Helper() — [Mistakes] #82

Call `t.Helper()` in every test helper. Without it, failure line points to the helper, not the test case.

```go
func assertNoError(t *testing.T, err error) {
    t.Helper()  // failure now points to the caller, not this line
    if err != nil { t.Fatalf("unexpected error: %v", err) }
}
```

### t.Cleanup() — [Mistakes] #83

Prefer `t.Cleanup()` over `defer` in tests. `defer` is skipped if `t.FailNow()` is called; `t.Cleanup()` always runs.

```go
func TestWithDB(t *testing.T) {
    db := setupDB(t)
    t.Cleanup(func() { db.Close() })  // always runs, even after t.Fatal
    // ...
}
```

### t.Parallel() — [Mistakes] #84

```go
for _, tt := range tests {
    tt := tt  // capture range variable (required in Go < 1.22)
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()  // at the top, after t.Helper() if needed
        // never share mutable state between parallel subtests
    })
}
```

### Manual Mock via Interface

```go
// Define interface in the package that needs it
type UserStore interface {
    FindByID(ctx context.Context, id string) (*User, error)
}

// Manual mock — no generator needed for simple cases
type MockUserStore struct {
    FindByIDFn func(ctx context.Context, id string) (*User, error)
}
func (m *MockUserStore) FindByID(ctx context.Context, id string) (*User, error) {
    return m.FindByIDFn(ctx, id)
}

// In test
store := &MockUserStore{
    FindByIDFn: func(ctx context.Context, id string) (*User, error) {
        return &User{ID: id, Name: "alice"}, nil
    },
}
```

### Benchmarks

```go
func BenchmarkParseUser(b *testing.B) {
    data := []byte(`{"name":"alice","email":"alice@example.com"}`)
    b.ResetTimer()      // exclude setup time
    b.ReportAllocs()    // show allocations/op
    for b.Loop() {      // Go 1.24+; use b.N in earlier versions
        ParseUser(data)
    }
}
// Run: go test -bench=. -benchmem ./...
```

---

## 7. Resilience — [CloudNative] Ch.5-8, [ConcurrencyInGo] Ch.5

### Retry with Exponential Backoff + Jitter — [CloudNative] Ch.7

Jitter prevents thundering herd: all retrying clients desynchronize.

```go
func Retry(ctx context.Context, maxAttempts int, fn func() error) error {
    var err error
    for i := range maxAttempts {
        if ctx.Err() != nil { return ctx.Err() }
        if err = fn(); err == nil { return nil }
        if i == maxAttempts-1 { break }
        backoff := time.Duration(1<<uint(i)) * 100 * time.Millisecond
        jitter := time.Duration(rand.Int63n(int64(backoff / 2)))
        select {
        case <-ctx.Done(): return ctx.Err()
        case <-time.After(backoff + jitter):
        }
    }
    return fmt.Errorf("after %d attempts: %w", maxAttempts, err)
}
```

### Circuit Breaker — [CloudNative] Ch.8

State machine: `Closed` (normal) → `Open` (failing, reject fast) → `Half-Open` (testing recovery).

```go
type State int
const (
    StateClosed   State = iota  // requests pass through
    StateOpen                    // requests rejected immediately
    StateHalfOpen               // probing — note: this simplified example does not enforce single-probe semantics
)

type CircuitBreaker struct {
    mu          sync.Mutex
    state       State
    failures    int
    threshold   int
    lastFailure time.Time
    cooldown    time.Duration
}

func (cb *CircuitBreaker) Do(fn func() error) error {
    cb.mu.Lock()
    if cb.state == StateOpen {
        if time.Since(cb.lastFailure) > cb.cooldown {
            cb.state = StateHalfOpen
        } else {
            cb.mu.Unlock()
            return errors.New("circuit open")
        }
    }
    cb.mu.Unlock()

    err := fn()

    cb.mu.Lock()
    defer cb.mu.Unlock()
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.threshold { cb.state = StateOpen }
        return err
    }
    cb.failures = 0
    cb.state = StateClosed
    return nil
}
```

### Graceful Shutdown — [CloudNative] Ch.5

```go
func run(ctx context.Context) error {
    srv := &http.Server{Addr: ":8080", Handler: mux}

    ctx, stop := signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    go func() {
        <-ctx.Done()
        shutCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        if err := srv.Shutdown(shutCtx); err != nil {
            log.Printf("shutdown error: %v", err)
        }
    }()

    if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
        return err
    }
    return nil
}
```

### Rate Limiting — [ConcurrencyInGo] Ch.5

```go
import "golang.org/x/time/rate"

// 10 requests/second, burst of 10
limiter := rate.NewLimiter(rate.Every(time.Second/10), 10)

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if !limiter.Allow() {
        http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
        return
    }
    // handle request
}

// For async: limiter.Wait(ctx) blocks until token is available
```

### Health Checks — [CloudNative]

```go
// /healthz — liveness: is the process alive?
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
})

// /readyz — readiness: can this instance serve traffic?
mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
    if err := db.PingContext(r.Context()); err != nil {
        http.Error(w, "db unavailable", http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

---

## 8. Performance & Pitfalls — [Mistakes] #20-#29, #39, #47, #95-#97, [GoBook] Ch.3

### Slice — [Mistakes] #20-#24

```go
// Pre-allocate when size is known — avoids O(log n) reallocations
result := make([]User, 0, len(rows))

// Always reassign append result
s = append(s, item)  // NOT: append(s, item) — result silently discarded

// Backing array aliasing — sub-slice shares memory with original
orig := []int{1, 2, 3, 4, 5}
sub := orig[1:3]        // shares backing array
sub = append(sub, 99)   // may overwrite orig[3]!

safe := slices.Clone(orig[1:3])  // independent copy

// nil vs empty slice — JSON marshaling differs
var s []int          // nil  → json.Marshal → "null"
s := []int{}         // empty → json.Marshal → "[]"
s := make([]int, 0)  // empty → json.Marshal → "[]"
// Use make([]T, 0) for JSON API responses that must return "[]"
```

### Map — [Mistakes] #27-#29

```go
// PANIC: writing to nil map
var m map[string]int
m["key"] = 1  // panic: assignment to entry in nil map

// Always initialize before writing
m := make(map[string]int)
m["key"] = 1  // OK

// Pre-allocate when size is known
m := make(map[string]int, len(items))

// Iteration order is random — never rely on it
for k, v := range m { ... }  // order not guaranteed
```

### strings.Builder — [Mistakes] #39

```go
// DON'T: O(n²) — new string allocation on every +
result := ""
for _, s := range parts { result += s }

// DO: O(n) — single allocation at the end
var b strings.Builder
b.Grow(totalLen)  // optional: pre-allocate if total size known
for _, s := range parts { b.WriteString(s) }
result := b.String()
```

### defer in Loops — [Mistakes] #47

```go
// WRONG: all defers stack until function returns, not loop end
for _, f := range files {
    defer f.Close()  // files not closed until the outer function exits
}

// RIGHT: use an inner function to scope the defer
for _, f := range files {
    func() {
        defer f.Close()
        process(f)
    }()
}
```

### Memory Leaks — [Mistakes] #95-#97

```go
// Slice leak: sub-slice keeps entire backing array alive
func getFirst100(data []byte) []byte {
    return data[:100]  // BAD: entire data kept in memory
}
func getFirst100(data []byte) []byte {
    result := make([]byte, 100)
    copy(result, data[:100])
    return result  // GOOD: independent allocation
}

// Goroutine leak: always ensure goroutines can exit
// See Section 4 for patterns
```

### Escape Analysis

```go
// Check what escapes to heap
// go build -gcflags="-m" ./... 2>&1 | grep "escapes to heap"

// Minimize pointer returns to reduce GC pressure
func compute() Point { return Point{X: 1, Y: 2} }   // stack-allocated
func compute() *Point { return &Point{X: 1, Y: 2} }  // heap-allocated (escapes)
```

---

## Quick Decision Reference

| Situation | Pattern | Source |
|---|---|---|
| Caller needs to inspect error cause | `%w` | [Mistakes] #49 |
| Intentionally hide implementation detail | `%v` | [Mistakes] #48 |
| Combine multiple independent errors | `errors.Join` | [Mistakes] #50 |
| N independent tasks, cancel-on-first-error | `errgroup` | [Mistakes] #71 |
| Worker pool, must drain all jobs | `WaitGroup + chan error` | [ConcurrencyInGo] Ch.4 |
| Protect shared state | `sync.Mutex` | [ConcurrencyInGo] Ch.2 |
| Transfer data ownership between goroutines | Channel | [ConcurrencyInGo] Ch.2 |
| Many optional constructor params | Functional Options | [LearningGo] Ch.14 |
| Config from file/env, >3 required fields | Config Struct | [LearningGo] Ch.14 |
| Single implementation constructor | Return concrete struct | [LearningGo] Ch.7 |
| Multiple swappable implementations (exception) | Return interface | [LearningGo] Ch.7 |
| JSON API returns empty list | `make([]T, 0)` | [Mistakes] #22 |
| Unstable downstream service | Circuit breaker | [CloudNative] Ch.8 |
| Transient failures | Retry + exponential backoff + jitter | [CloudNative] Ch.7 |
| Detect data races | `go test -race ./...` | [Mistakes] #58 |
| Repeated string concatenation | `strings.Builder` | [Mistakes] #39 |
