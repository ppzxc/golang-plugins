---
name: coder
description: "Use when writing or refactoring Go production code — types, interfaces, error handling, concurrency, context, HTTP, service design, resilience, and performance patterns."
user_invocable: true
---

# Go Coder

Rule-based reference for production Go code. Every rule cites a canonical source.

**Sources:** [GoBook] The Go Programming Language, [Mistakes] 100 Go Mistakes and How to Avoid Them, [Concurrency] Concurrency in Go, [LearningGo] Learning Go 2nd Ed, [CloudNative] Cloud Native Go, [LetsGo] Let's Go / Let's Go Further, [EffectiveGo] Effective Go, [CodeReview] Go Code Review Comments

---

## Section 1: Types & Data Structures

### DO

- **[SHOULD] Order struct fields by alignment**: Place larger fields first to minimize padding [Mistakes#91]
  ```go
  type Good struct {
      Data  [256]byte // 256 bytes
      Count int64     // 8 bytes
      Flag  bool      // 1 byte + 7 padding
  }
  ```

- **[SHOULD] Pre-allocate slices with known capacity**: Avoid repeated reallocations with make [Mistakes#21]
  ```go
  users := make([]User, 0, len(rows))
  for _, r := range rows {
      users = append(users, toUser(r))
  }
  ```

- **[SHOULD] Provide map size hints**: Reduce rehashing when count is known or estimable [Mistakes#27]
  ```go
  m := make(map[string]*User, len(ids))
  for _, id := range ids {
      m[id] = fetchUser(id)
  }
  ```

- **[SHOULD] Use named types for domain concepts**: Prevent accidental mixing of same-underlying types [GoBook§2]
  ```go
  type UserID string
  type OrderID string
  func GetOrder(uid UserID, oid OrderID) *Order // compiler rejects swapped args
  ```

- **[SHOULD] Design for useful zero values**: Make the zero value of your types immediately usable [EffectiveGo]
  ```go
  type Counter struct{ n int64 }
  func (c *Counter) Inc() { c.n++ } // works without constructor
  var c Counter; c.Inc()             // zero value is valid
  ```

- **[SHOULD] Use iota for enums starting with unknown zero**: Prevent uninitialized values from being valid [Mistakes#5]
  ```go
  type Status int
  const (
      StatusUnknown Status = iota // zero value = unknown
      StatusActive
      StatusInactive
  )
  ```

- **[SHOULD] Use go:embed for static assets**: Embed files at compile time instead of runtime I/O [EffectiveGo]
  ```go
  import _ "embed"

  //go:embed templates/*.html
  var templates embed.FS

  //go:embed version.txt
  var version string
  ```

### DON'T

- **[SHOULD] Don't use generics prematurely**: Prefer concrete types until you have 3+ duplicated implementations [LearningGo§8]
  ```go
  // BAD: generic for a single use case
  func ProcessItems[T Item](items []T) error { ... }
  // GOOD: concrete until proven need
  func ProcessOrders(orders []Order) error { ... }
  ```

- **[SHOULD] Don't embed types just for convenience**: Embed only when the outer type IS-A inner type [Mistakes#9]
  ```go
  // BAD: leaks sync.Mutex methods to callers
  type Cache struct { sync.Mutex; data map[string]string }
  // GOOD: unexported field hides implementation
  type Cache struct { mu sync.Mutex; data map[string]string }
  ```

### WHEN

- **Generics for type-safe containers**: "When the same logic applies to multiple types with no behavior difference, use generics" [LearningGo§8]

- **Generics with constraints for type-safe utilities**: "When writing generic functions, use constraint interfaces to restrict type parameters" [LearningGo§8]
  ```go
  type Number interface { ~int | ~int64 | ~float64 }
  func Sum[T Number](vals []T) T {
      var total T
      for _, v := range vals { total += v }
      return total
  }
  ```

---

## Section 2: Interfaces

### DO

- **[SHOULD] Accept interfaces, return structs**: Functions take interface params and return concrete types [LearningGo§7]
  ```go
  type Storer interface { Save(ctx context.Context, v any) error }
  func NewCache(s Storer) *Cache { return &Cache{store: s} }
  ```

- **[SHOULD] Keep interfaces small (1-3 methods)**: Compose small interfaces rather than defining large ones [Mistakes#5]
  ```go
  type Reader interface { Read(p []byte) (int, error) }
  type Writer interface { Write(p []byte) (int, error) }
  type ReadWriter interface { Reader; Writer }
  ```

- **[SHOULD] Define interfaces at consumer side**: The package that uses the abstraction owns the interface [EffectiveGo]
  ```go
  // internal/cache/cache.go defines what it needs
  type Storer interface { Save(ctx context.Context, id string, v any) error }
  type Cache struct { store Storer }
  ```

- **[MUST] Use type assertions with ok pattern**: Avoid panics on failed assertions [GoBook§7]
  ```go
  if w, ok := val.(io.Writer); ok {
      w.Write(data)
  }
  ```

- **[SHOULD] Compile-time interface checks**: Verify implementations at compile time [CodeReview]
  ```go
  var _ http.Handler = (*UserHandler)(nil)
  ```

### DON'T

- **[SHOULD] Don't export interfaces for implementation**: Exported interfaces force all consumers to use the same abstraction [Mistakes#6]
  ```go
  // BAD: package store exports interface
  type Store interface { Save(ctx context.Context, v any) error }
  // GOOD: package store exports struct, consumer defines interface
  type RedisStore struct { ... }
  ```

- **[SHOULD] Don't create 5+ method interfaces**: Large interfaces reduce reusability and testability [Mistakes#5]
  ```go
  // BAD: too many methods
  type UserManager interface {
      Create(u User) error; Update(u User) error; Delete(id string) error
      Find(id string) (*User, error); List() ([]User, error); Count() (int, error)
  }
  ```

- **[MUST] Don't use interface{}/any as lazy typing**: Use specific types or small interfaces [Mistakes#7]
  ```go
  // BAD: loses type safety
  func Process(data any) any { ... }
  // GOOD: explicit types
  func Process(data Input) Output { ... }
  ```

### WHEN

- **Return interface for multiple implementations**: "When a factory must return different concrete types based on config, return an interface" [LearningGo§7]

---

## Section 3: Error Handling

### DO

- **[MUST] Always check errors**: Never discard an error return value [Mistakes#48]
  ```go
  f, err := os.Open(path)
  if err != nil { return fmt.Errorf("open config: %w", err) }
  defer f.Close()
  ```

- **[SHOULD] Wrap errors with context using %w**: Preserve the error chain for callers [Mistakes#49]
  ```go
  user, err := s.repo.FindByID(ctx, id)
  if err != nil { return nil, fmt.Errorf("find user %s: %w", id, err) }
  ```

- **[SHOULD] Use sentinel errors for expected conditions**: Define package-level error values for known states [GoBook§5]
  ```go
  var ErrNotFound = errors.New("not found")
  var ErrConflict = errors.New("conflict")
  if errors.Is(err, ErrNotFound) { return http.StatusNotFound }
  ```

- **[MAY] Use custom error types for rich context**: Carry structured data when callers need details [LearningGo§9]
  ```go
  type ValidationError struct { Field, Message string }
  func (e *ValidationError) Error() string {
      return fmt.Sprintf("validation: %s: %s", e.Field, e.Message)
  }
  ```

- **[MUST] Match errors with errors.Is/errors.As**: Never compare with == on wrapped errors [Mistakes#50]
  ```go
  var ve *ValidationError
  if errors.As(err, &ve) {
      return http.StatusBadRequest, ve.Field
  }
  ```

- **[MUST] Handle errors once**: Log or return, never both [Mistakes#48]
  ```go
  // BAD: handled twice
  if err != nil { log.Error("fail", "err", err); return err }
  // GOOD: return with context, let caller decide
  if err != nil { return fmt.Errorf("save order: %w", err) }
  ```

### DON'T

- **[MUST] Don't panic for expected errors**: Panic is for programmer bugs, not runtime conditions [GoBook§5]
  ```go
  // BAD: panic on I/O error
  data, err := os.ReadFile(path)
  if err != nil { panic(err) }
  // GOOD: return error
  if err != nil { return nil, fmt.Errorf("read config: %w", err) }
  ```

- **[SHOULD] Don't start error messages with capital or punctuation**: Errors are often wrapped in chains [CodeReview]
  ```go
  // BAD: "Query failed: connection refused"
  return fmt.Errorf("Query failed: %w", err)
  // GOOD: "query user: connection refused"
  return fmt.Errorf("query user: %w", err)
  ```

- **[SHOULD] Don't ignore deferred Close() errors**: Use errors.Join to combine with the main error [Mistakes#50] [Go 1.20+]
  ```go
  func writeFile(path string, data []byte) (err error) {
      f, err := os.Create(path)
      if err != nil { return err }
      defer func() { err = errors.Join(err, f.Close()) }()
      _, err = f.Write(data)
      return err
  }
  ```

### WHEN

- **Use %v to intentionally hide cause**: "When error wrapping would leak internal implementation, use %v instead of %w" [Mistakes#49]

---

## Section 4: Functions & Methods

### DO

- **[MAY] Functional options for complex config**: Use when most parameters are optional [LearningGo§14]
  ```go
  type Option func(*Server)
  func WithPort(p int) Option { return func(s *Server) { s.port = p } }
  func NewServer(opts ...Option) *Server {
      s := &Server{port: 8080}; for _, o := range opts { o(s) }; return s
  }
  ```

- **[SHOULD] Use defer for cleanup**: Guarantee resource release on all exit paths [GoBook§5]
  ```go
  mu.Lock()
  defer mu.Unlock()
  return doWork()
  ```

- **[SHOULD] Be consistent with receiver types**: If any method needs a pointer, all methods use pointer [CodeReview]
  ```go
  type User struct{ Name string }
  func (u *User) SetName(n string) { u.Name = n }
  func (u *User) String() string   { return u.Name } // pointer too
  ```

### DON'T

- **[SHOULD] Don't use named returns for short functions**: Named returns hurt readability in simple functions [CodeReview]
  ```go
  // BAD: unnecessary named return
  func add(a, b int) (result int) { result = a + b; return }
  // GOOD: direct return
  func add(a, b int) int { return a + b }
  ```

- **[SHOULD] Don't exceed 4-5 parameters**: Use a config struct or functional options instead [Mistakes#5]
  ```go
  // BAD: too many params
  func NewConn(host string, port int, user, pass string, timeout time.Duration, tls bool) {}
  // GOOD: config struct
  func NewConn(cfg ConnConfig) (*Conn, error) { ... }
  ```

- **[SHOULD] Don't use naked returns**: Naked returns obscure what is being returned [CodeReview]
  ```go
  // BAD: naked return
  func find(id string) (u *User, err error) { u, err = repo.Get(id); return }
  // GOOD: explicit return
  func find(id string) (*User, error) { return repo.Get(id) }
  ```

### WHEN

- **Use named returns for deferred error handling**: "In functions where defer modifies the return value, named returns are required" [Mistakes#47]

---

## Section 5: Concurrency

### DO

- **[MUST] Every goroutine has a clear exit path**: The function that starts a goroutine owns its lifecycle [Concurrency§4]
  ```go
  func startWorker(ctx context.Context, jobs <-chan Job) {
      go func() {
          for { select { case <-ctx.Done(): return; case j, ok := <-jobs:
              if !ok { return }; process(j) } }
      }()
  }
  ```

- **[SHOULD] Use errgroup for concurrent error handling**: Cancel remaining work on first failure [Mistakes#71]
  ```go
  g, ctx := errgroup.WithContext(ctx)
  for _, url := range urls {
      g.Go(func() error { return fetch(ctx, url) }) // Go 1.22+: url captured correctly per iteration
  }
  if err := g.Wait(); err != nil { return err }
  ```

- **[SHOULD] Mutex for state, channel for communication**: Choose the right synchronization primitive [Concurrency§2]
  ```go
  // Mutex: protect shared map
  type Cache struct { mu sync.RWMutex; data map[string]string }
  // Channel: transfer ownership
  results := make(chan Result, workers)
  ```

- **[SHOULD] Channel direction in function signatures**: Restrict channel access to prevent misuse [GoBook§8]
  ```go
  func produce(out chan<- int) { out <- 42 }
  func consume(in <-chan int)  { v := <-in }
  ```

- **[SHOULD] sync.Once for lazy initialization**: Thread-safe one-time setup [Concurrency§3]
  ```go
  var once sync.Once
  var db *DB
  func GetDB() *DB { once.Do(func() { db = mustConnect() }); return db }
  ```

- **[MUST] Select with context for blocking ops**: Always include context cancellation in select [Concurrency§4]
  ```go
  select {
  case <-ctx.Done(): return ctx.Err()
  case item := <-queue: process(item)
  case results <- output:
  }
  ```

### DON'T

- **[MUST] Don't start goroutines in init()**: Makes testing and lifecycle control impossible [Mistakes#3]
  ```go
  // BAD: goroutine in init
  func init() { go backgroundProcess() }
  // GOOD: start in main or constructor with context
  func NewService(ctx context.Context) *Service { go s.run(ctx); return s }
  ```

- **[MUST] Don't capture loop vars by reference in goroutines**: Pre-Go 1.22 closure captures the variable, not the value [Mistakes#63] [Go < 1.22 only]
  ```go
  // BAD (Go < 1.22): all goroutines share same v
  for _, v := range items { go func() { process(v) }() }
  // GOOD: pass as argument
  for _, v := range items { go func(v Item) { process(v) }(v) }
  ```

- **[MUST] Don't copy sync primitives**: Mutex, WaitGroup, Cond must not be copied after first use [Mistakes#72]
  ```go
  // BAD: copying mutex
  type Cache struct { mu sync.Mutex }
  c2 := c1 // copies mutex — undefined behavior
  // GOOD: use pointer or never copy
  func NewCache() *Cache { return &Cache{} }
  ```

- **[SHOULD] Don't mix mutex and channel for same data**: Pick one synchronization strategy per shared resource [Concurrency§2]
  ```go
  // BAD: mutex AND channel guarding same counter
  type Stats struct { mu sync.Mutex; count int; updates chan int }
  // GOOD: one mechanism
  type Stats struct { mu sync.Mutex; count int }
  ```

- **[MUST] Don't use unbuffered channels without guaranteed receiver**: Goroutine will leak if nobody reads [Mistakes#66]
  ```go
  // BAD: goroutine blocks forever if caller returns early
  ch := make(chan int)
  go func() { ch <- compute() }()
  // GOOD: buffer of 1 lets goroutine complete
  ch := make(chan int, 1)
  go func() { ch <- compute() }()
  ```

### WHEN

- **Use WaitGroup over errgroup**: "When workers must drain all jobs even after errors, use WaitGroup + error channel" [Concurrency§4]
- Cross-ref: See `/golang:reviewer` Section 2 for concurrency code review checks
- Cross-ref: See `/golang:tester` Section 8 for race detection and concurrent testing

---

## Section 6: Context

### DO

- **[MUST] Pass context as first param named ctx**: Standard convention across all Go libraries [Mistakes#60]
  ```go
  func (r *Repo) FindUser(ctx context.Context, id string) (*User, error) {
      return r.db.QueryRowContext(ctx, query, id).Scan(...)
  }
  ```

- **[SHOULD] WithTimeout for external calls**: Bound all I/O and network operations [Mistakes#62]
  ```go
  ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
  defer cancel()
  resp, err := client.Do(req.WithContext(ctx))
  ```

- **[SHOULD] Respect cancellation in loops**: Check ctx.Done() to exit long-running work early [Concurrency§4]
  ```go
  for _, item := range items {
      if ctx.Err() != nil { return ctx.Err() }
      if err := process(ctx, item); err != nil { return err }
  }
  ```

### DON'T

- **[MUST] Don't store context in structs**: Context is per-request; storing it prevents per-call cancellation [Mistakes#60]
  ```go
  // BAD: context in struct
  type Client struct { ctx context.Context; http *http.Client }
  // GOOD: context per method call
  func (c *Client) Get(ctx context.Context, url string) (*Response, error)
  ```

- **[SHOULD] Don't use context.Value for control flow**: Values are for request-scoped metadata only [Mistakes#61]
  ```go
  // BAD: using Value for business logic
  if ctx.Value("isAdmin").(bool) { allowDelete() }
  // GOOD: explicit parameter
  func Delete(ctx context.Context, isAdmin bool) error
  ```

### WHEN

- **Use context.WithoutCancel**: "When a deferred operation must outlive the request (e.g., async cleanup), use context.WithoutCancel" [Mistakes#62] [Go 1.21+]

---

## Section 7: HTTP & Web

### DO

- **[SHOULD] Use Go 1.22+ ServeMux patterns**: Method and path parameter routing built in [LetsGo] [Go 1.22+]
  ```go
  mux := http.NewServeMux()
  mux.HandleFunc("GET /users/{id}", h.GetUser)
  mux.HandleFunc("POST /users", h.CreateUser)
  mux.HandleFunc("DELETE /users/{id}", h.DeleteUser)
  ```

- **[SHOULD] Middleware for cross-cutting concerns**: Standard func(http.Handler) http.Handler signature [CloudNative§4]
  ```go
  func Logging(log *slog.Logger) func(http.Handler) http.Handler {
      return func(next http.Handler) http.Handler {
          return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
              start := time.Now(); next.ServeHTTP(w, r)
              log.Info("req", "method", r.Method, "dur", time.Since(start))
          })
      }
  }
  ```

- **[SHOULD] DisallowUnknownFields for JSON decoding**: Reject typos and unexpected input fields [LetsGo]
  ```go
  dec := json.NewDecoder(r.Body)
  dec.DisallowUnknownFields()
  if err := dec.Decode(&input); err != nil { /* 400 */ }
  ```

- **[MUST] Set server timeouts explicitly**: Prevent slow-client attacks [LetsGo]
  ```go
  srv := &http.Server{
      Addr: ":8080", Handler: mux,
      ReadTimeout: 5 * time.Second, WriteTimeout: 10 * time.Second,
      IdleTimeout: 120 * time.Second,
  }
  ```

### DON'T

- **[MUST] Don't use default http.Client**: No timeouts means requests can hang forever [Mistakes#81]
  ```go
  // BAD: no timeout
  resp, err := http.Get(url)
  // GOOD: explicit client with timeout
  client := &http.Client{Timeout: 10 * time.Second}
  resp, err := client.Get(url)
  ```

- **[MUST] Don't write body after error status**: WriteHeader can only be called once [LetsGo]
  ```go
  // BAD: writes body then tries error (status 200 already sent)
  json.NewEncoder(w).Encode(data)
  if err != nil { http.Error(w, "fail", 500) } // too late
  // GOOD: check error first, then write
  if err != nil { http.Error(w, "fail", 500); return }
  json.NewEncoder(w).Encode(data)
  ```

- **[MUST] Don't read request body without size limit**: Prevent memory exhaustion from large payloads [LetsGo]
  ```go
  // BAD: unbounded read
  body, _ := io.ReadAll(r.Body)
  // GOOD: limit to 1MB
  r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
  if err := json.NewDecoder(r.Body).Decode(&v); err != nil { /* 400 */ }
  ```

### WHEN

- **Use http.MaxBytesReader vs io.LimitReader**: "For HTTP handlers use MaxBytesReader (returns 413); for general I/O use LimitReader" [LetsGo]

---

## Section 8: Service Design

### DO

- **[SHOULD] Constructor injection for dependencies**: Wire in main.go, no frameworks needed [CloudNative§2]
  ```go
  func main() {
      db := mustOpenDB(cfg.DSN)
      store := postgres.NewUserStore(db)
      svc := user.NewService(store, log)
      handler := api.NewHandler(svc, log)
  }
  ```

- **[SHOULD] Use slog for structured logging**: Key-value pairs, not format strings [EffectiveGo]
  ```go
  log := slog.New(slog.NewJSONHandler(os.Stdout, nil))
  log.Info("request", "method", r.Method, "path", r.URL.Path,
      "status", status, "duration", time.Since(start))
  ```

- **[SHOULD] Config from environment with validation at startup**: Fail fast on missing config [CloudNative§3]
  ```go
  type Config struct {
      DSN  string `env:"DATABASE_URL,required"`
      Port int    `env:"PORT" envDefault:"8080"`
  }
  func Load() Config { var c Config; env.Parse(&c); return c }
  ```

- **[SHOULD] Separate transport/business/storage layers**: Each layer only knows the layer below via interface [CloudNative§2]
  ```go
  // transport: internal/http/handler.go — depends on Service interface
  // business:  internal/user/service.go — depends on Repository interface
  // storage:   internal/postgres/repo.go — implements Repository
  ```

### DON'T

- **[SHOULD] Don't use global state for dependencies**: Globals prevent testing and parallel execution [Mistakes#1]
  ```go
  // BAD: global DB
  var db *sql.DB
  func GetUser(id string) (*User, error) { return queryUser(db, id) }
  // GOOD: injected dependency
  type Repo struct{ db *sql.DB }
  func (r *Repo) GetUser(ctx context.Context, id string) (*User, error)
  ```

- **[MUST] Don't log sensitive data**: Mask tokens, passwords, PII in log output [CloudNative§3]
  ```go
  // BAD: logs full token
  log.Info("auth", "token", token)
  // GOOD: mask sensitive fields
  log.Info("auth", "token", token[:8]+"...")
  ```

### WHEN

- **Use config struct over env vars directly**: "When config is shared across packages, load once into a struct and inject" [CloudNative§3]

---

## Section 9: Resilience

### DO

- **[SHOULD] Exponential backoff with jitter for retries**: Prevent thundering herd on failures [CloudNative§7]
  ```go
  func Retry(ctx context.Context, max int, fn func() error) error {
      for i := range max {
          if err := fn(); err == nil { return nil }
          backoff := time.Duration(1<<uint(i)) * 100 * time.Millisecond
          jitter := time.Duration(rand.Int63n(int64(backoff / 2)))
          select { case <-ctx.Done(): return ctx.Err(); case <-time.After(backoff+jitter): }
      }
      return fmt.Errorf("after %d attempts: %w", max, fn())
  }
  ```

- **[MUST] OS signal handling for graceful shutdown**: Drain in-flight requests before exiting [CloudNative§5]
  ```go
  ctx, stop := signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)
  defer stop()
  go func() { <-ctx.Done()
      shutCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
      defer cancel(); srv.Shutdown(shutCtx)
  }()
  ```

- **[MAY] Circuit breaker for external dependencies**: Fail fast when downstream is unhealthy [CloudNative§8]
  ```go
  type State int
  const (
      Closed State = iota
      Open
      HalfOpen
  )

  type Breaker struct {
      mu        sync.Mutex
      state     State
      failures  int
      threshold int
      lastFail  time.Time
      cooldown  time.Duration
  }

  func (b *Breaker) Do(fn func() error) error {
      b.mu.Lock()
      if b.state == Open {
          if time.Since(b.lastFail) < b.cooldown {
              b.mu.Unlock()
              return ErrCircuitOpen
          }
          b.state = HalfOpen
      }
      b.mu.Unlock()

      err := fn()

      b.mu.Lock()
      defer b.mu.Unlock()
      if err != nil {
          b.failures++
          b.lastFail = time.Now()
          if b.failures >= b.threshold { b.state = Open }
          return err
      }
      b.failures = 0
      b.state = Closed
      return nil
  }
  ```

### DON'T

- **[SHOULD] Don't retry without backoff**: Immediate retries amplify failures under load [CloudNative§7]
  ```go
  // BAD: tight retry loop hammers failing service
  for i := 0; i < 3; i++ { err = callService(); if err == nil { break } }
  // GOOD: exponential backoff (see Retry above)
  err = Retry(ctx, 3, callService)
  ```

- **[MUST] Don't ignore shutdown context timeout**: Services that hang on shutdown delay deployments [CloudNative§5]
  ```go
  // BAD: blocks forever waiting for connections
  srv.Shutdown(context.Background())
  // GOOD: bounded shutdown
  ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
  defer cancel()
  srv.Shutdown(ctx)
  ```

### WHEN

- **Skip retries for non-transient errors**: "When the error is 4xx (client error), don't retry; only retry on 5xx or network errors" [CloudNative§7]

---

## Section 10: Performance

### DO

- **[SHOULD] strings.Builder for loop concatenation**: O(n) vs O(n^2) for repeated string concat [Mistakes#39]
  ```go
  var b strings.Builder
  b.Grow(totalLen)
  for _, s := range parts { b.WriteString(s) }
  result := b.String()
  ```

- **[SHOULD] Pre-allocate when size is known**: Apply to slices, maps, and builders [Mistakes#21]
  ```go
  result := make([]string, 0, len(input))
  for _, v := range input { result = append(result, transform(v)) }
  ```

- **[MUST] Profile before optimizing with pprof**: Measure first, optimize second [Mistakes#95]
  ```go
  import _ "net/http/pprof"
  // go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
  // go tool pprof http://localhost:6060/debug/pprof/heap
  ```

- **[MAY] sync.Pool for high-churn objects**: Reuse allocations for frequently created/destroyed objects [Concurrency§3]
  ```go
  var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}
  func process(data []byte) string {
      buf := bufPool.Get().(*bytes.Buffer); defer func() { buf.Reset(); bufPool.Put(buf) }()
      buf.Write(data); return buf.String()
  }
  ```

### DON'T

- **[MUST] Don't optimize without measurement**: Guessing at bottlenecks wastes time and adds complexity [Mistakes#95]
  ```go
  // BAD: premature optimization without profiling
  // "I think this map lookup is slow" -> rewrites with unsafe
  // GOOD: profile first
  // go test -bench=. -cpuprofile=cpu.prof ./...
  // go tool pprof cpu.prof
  ```

- **[SHOULD] Don't use pointer receivers for small read-only structs**: Value receivers avoid heap allocation [Mistakes#95]
  ```go
  // BAD: pointer for small immutable struct adds GC pressure
  func (p *Point) Distance(q *Point) float64 { ... }
  // GOOD: value receiver, stays on stack
  func (p Point) Distance(q Point) float64 { ... }
  ```

- **[SHOULD] Don't defer in tight loops**: Defers stack until function returns, not loop iteration [Mistakes#47]
  ```go
  // BAD: all defers accumulate until function returns
  for _, f := range files { defer f.Close() }
  // GOOD: wrap in closure to scope the defer
  for _, f := range files {
      func() { defer f.Close(); process(f) }()
  }
  ```

### WHEN

- **Use sync.Pool only for high-frequency allocations**: "When profiling shows GC pressure from a specific type allocated in hot paths, use sync.Pool" [Concurrency§3]
