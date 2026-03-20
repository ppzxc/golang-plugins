---
name: modern-patterns
description: "Use when writing, reviewing, or refactoring Go code in production services — error handling, concurrency, interfaces, testing, and resilience patterns."
user_invocable: true
---

# Modern Go Patterns

A practical reference for writing production-grade Go. Every section cites one of three canonical sources:

- **[Mistakes]** — *100 Go Mistakes and How to Avoid Them* (Teiva Harsanyi)
- **[LearningGo]** — *Learning Go, 2nd Edition* (Jon Bodner)
- **[CloudNative]** — *Cloud Native Go* (Matthew Titmus)

---

## 1. Error Handling — [Mistakes] #48-#53, [LearningGo] Ch 9

### Wrapping: %w vs %v

- `%w`: callers can inspect cause with `errors.Is`/`errors.As`
- `%v`: intentionally hides the implementation detail

```go
// DO: wrap to preserve chain
return fmt.Errorf("store message: %w", err)

// DON'T: break the chain
return fmt.Errorf("store message: %v", err)  // errors.Is fails
```

### errors.Is / errors.As

```go
if errors.Is(err, dmsc.ErrNotFound) { return http.StatusNotFound }

var motErr *util.MotError
if errors.As(err, &motErr) && motErr.Temporary { retry(msg) }
```

### Sentinel vs Custom Types

- Sentinel: identity checks only (`var ErrNotFound = errors.New("not found")`)
- Custom type: when caller needs structured data (fields like `Temporary bool`)

### Panic vs Return

Panic only for unrecoverable programmer bugs (nil callback). Everything else returns error.

### Errors in Goroutines

Never let goroutines silently swallow errors. Use channels or `errgroup`.

---

## 2. Context — [Mistakes] #60-#62, [CloudNative] Ch 4

- Always pass `context.Context` as first param named `ctx`. Never store in struct.
- `defer cancel()` immediately after `WithCancel`/`WithTimeout`/`WithDeadline`.
- `WithValue` keys must be unexported types. Values: request-scoped only (trace ID, auth).
- Check `ctx.Err()` in loops and before expensive operations.

```go
// DO
func (r *Client) Query(ctx context.Context, number string) (string, error)

// DON'T — context in struct
type Client struct { ctx context.Context }

// Timeout pattern
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

---

## 3. Generics (Go 1.18+) — [LearningGo] Ch 8

- Use for type-safe containers and algorithms (same logic, different types)
- Use interfaces when different types have different behavior
- Prefer named constraints over inline unions
- Generic functions preferred (methods cannot have type parameters)

```go
type Number interface { ~int | ~int64 | ~float64 }

func Sum[T Number](vals []T) T {
    var total T
    for _, v := range vals { total += v }
    return total
}

// WON'T COMPILE: func (s *Store) Get[T any](id string) T
// DO: func Get[T any](s *Store, id string) T
```

---

## 4. Concurrency — [Mistakes] #61-#82, [CloudNative] Ch 4

### Goroutine Lifecycle

The function that starts a goroutine owns its lifecycle. Always ensure it can be stopped.

```go
go func() {
    for {
        select {
        case <-ctx.Done(): return
        case msg := <-ch:  process(msg)
        }
    }
}()
```

### errgroup vs Manual WaitGroup — Choose carefully

| Use `errgroup` | Use manual `WaitGroup + chan error` |
|---|---|
| Independent tasks where first error should cancel all | Worker pool that must **drain jobs even after error** |
| Parallel fetch where any failure aborts the whole operation | Pipeline stages where stopping early blocks senders |

```go
// errgroup: parallel API calls — first error cancels all (correct)
g, ctx := errgroup.WithContext(ctx)
for _, url := range urls {
    g.Go(func() error { return fetch(ctx, url) })
}
if err := g.Wait(); err != nil { /* first error */ }

// Manual WaitGroup: worker pool — must drain jobs after error
errCh := make(chan error, n)
var wg sync.WaitGroup
for range n {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for job := range jobs {
            if ctx.Err() != nil { return }
            if err := process(job); err != nil {
                select { case errCh <- err: default: }
            }
        }
    }()
}
go func() { wg.Wait(); close(errCh) }()
```

### Mutex vs Channel

| **Mutex** | **Channel** |
|---|---|
| Protecting shared state (map, counter) | Passing data ownership between goroutines |
| Short critical sections | Signaling events, coordinating pipelines |

### Goroutine Leak Prevention — [Mistakes] #63

```go
// DON'T: unbuffered channel, nobody reads → leaked goroutine
ch := make(chan int)
go func() { ch <- compute() }()

// DO: buffered channel or context
ch := make(chan int, 1)
```

### sync.Once / sync.Pool

```go
var once sync.Once
var db *sql.DB
func getDB() *sql.DB {
    once.Do(func() { db = mustConnect() })
    return db
}

var bufPool = sync.Pool{ New: func() any { return new(bytes.Buffer) } }
```

---

## 5. Interface Design — [LearningGo] Ch 7, [Mistakes] #5

- **Accept interfaces, return structs.** Define interfaces on the consumer side.
- Keep to 1-3 methods. If larger, split by consumer need.
- **Constructor return type rule:**
  - Return **concrete struct** by default (`func New(c Config) *Server`)
  - Return **interface** only when the package provides multiple implementations that callers swap (`func New(c Config) gateway.Module` — smsc, mmsc are both Modules)
- Compile-time check: `var _ gateway.Module = (*module)(nil)`
- Use `any` only when truly necessary. Prefer concrete types or constrained generics.

```go
// Interface composition
type Module interface {
    Transmitter
    Serve(context.Context, Callback) error
}
```

---

## 6. Slice & Map Gotchas — [Mistakes] #21-#28

```go
var s []int          // nil → json: "null"
s := []int{}         // empty → json: "[]"
s := make([]int, 0)  // empty → json: "[]"
```

- Pre-allocate: `make([]T, 0, len(source))` when size is known
- Slicing shares backing array: use `slices.Clone()` to avoid aliasing
- nil map write panics: always `make(map[K]V)` before writing
- Map iteration order is random

---

## 7. Constructor & Config Patterns — [LearningGo] Ch 7, 13

### Functional Options (optional params > 3)

```go
type Option func(*Server)
func WithPort(p uint16) Option { return func(s *Server) { s.port = p } }
func WithTimeout(d time.Duration) Option { return func(s *Server) { s.timeout = d } }

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080, timeout: 30 * time.Second}
    for _, o := range opts { o(s) }
    return s
}
```

### Config Struct (settings from YAML/env)

```go
type Config struct {
    Host string `mapstructure:"host" default:"localhost"`
    Port uint16 `mapstructure:"port" default:"8080"`
}
func New(c Config) gateway.Module { return &module{config: c} }
```

- Return `(T, error)` when construction can fail. Return `T` when it cannot.
- Name: `NewXxx`, multiple constructors: `NewXxxFromYyy`

---

## 8. Testing — [LearningGo] Ch 15

### Table-driven with t.Run

```go
tests := []struct {
    name string; input string; want MessageType; wantErr bool
}{
    {"SMS", "SMS", TypeSMS, false},
    {"unknown", "FAX", 0, true},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := NewMessageType(tt.input)
        if (err != nil) != tt.wantErr { t.Fatalf("err = %v, wantErr = %v", err, tt.wantErr) }
        if got != tt.want { t.Errorf("got %v, want %v", got, tt.want) }
    })
}
```

- `t.Helper()` in test helpers for correct line reporting
- `t.Cleanup()` over `defer` — runs even after `t.Fatal`
- `t.Parallel()` for independent tests only (never with shared mutable state)
- Manual mocks with function fields, no frameworks

---

## 9. Resilience Patterns — [CloudNative] Ch 4, 5

### Retry with Exponential Backoff

```go
func Retry(ctx context.Context, max int, fn func() error) error {
    var err error
    for i := range max {
        if ctx.Err() != nil { return ctx.Err() }
        if err = fn(); err == nil { return nil }
        time.Sleep(time.Duration(1<<uint(i)) * 100 * time.Millisecond)
    }
    return fmt.Errorf("after %d attempts: %w", max, err)
}
```

### Graceful Shutdown

```go
sig := make(chan os.Signal, 1)
signal.Notify(sig, syscall.SIGINT, syscall.SIGTERM)
go func() { <-sig; cancel() }()
```

### Circuit Breaker

Track failures with mutex-protected counter. Open circuit after threshold. Half-open after cooldown. Reset on success.

---

## 10. Project Structure — [LearningGo] Ch 10

```
cmd/       — main packages (one per binary)
internal/  — private, not importable outside module
pkg/       — public libraries safe for external import
```

- Short, lowercase, single-word package names: `config`, `gateway`, `dmsc`
- No stutter: `smsc.New()`, not `smsc.NewSMSC()`
- Package name should not repeat parent: `gateway/smsc`, not `gateway/gatewaysmsc`

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| `%v` when caller needs to inspect cause | Use `%w` |
| `errgroup` for worker pool that must drain | Use manual `WaitGroup + chan error` |
| Returning concrete type when package has multiple impls | Return the interface |
| Returning interface from single-impl constructor | Return concrete struct |
| `var s []int` in JSON API response | `make([]T, 0)` |
| Writing to nil map | `make(map[K]V)` before first write |
| Goroutine without `ctx.Done()` exit | Always select on `ctx.Done()` |
| Context stored in struct | Pass as first function parameter |

---

## Quick Decision Reference

| Situation | Pattern | Source |
|---|---|---|
| Caller needs to inspect cause | `%w` | [Mistakes] #49 |
| Hide implementation detail | `%v` | [Mistakes] #48 |
| Many optional params | Functional options | [LearningGo] Ch 13 |
| Settings from YAML/env | Config struct | [LearningGo] Ch 13 |
| Protect shared state | `sync.Mutex` | [Mistakes] #72 |
| Pass data between goroutines | Channel | [Mistakes] #72 |
| N independent tasks, cancel-on-error | `errgroup` | [CloudNative] Ch 4 |
| Worker pool, drain-on-error | `WaitGroup + chan error` | [Mistakes] #63 |
| Unstable downstream | Circuit breaker | [CloudNative] Ch 5 |
| Transient failures | Retry + backoff | [CloudNative] Ch 5 |
| JSON API returns empty list | `make([]T, 0)` | [Mistakes] #22 |
