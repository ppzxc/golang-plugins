---
name: go-reviewer
description: "Use when reviewing Go code in pull requests or auditing Go codebases — checklists for error handling, concurrency safety, naming conventions, API design, performance pitfalls, security, and package structure."
user_invocable: true
---

# Go Reviewer

CHECK/FLAG review checklist for production Go. Every rule cites a canonical source.

**Sources:** [GoBook] The Go Programming Language, [Mistakes] 100 Go Mistakes and How to Avoid Them, [Concurrency] Concurrency in Go, [LearningGo] Learning Go 2nd Ed, [CloudNative] Cloud Native Go, [LetsGo] Let's Go / Let's Go Further, [EffectiveGo] Effective Go, [CodeReview] Go Code Review Comments

---

## Section 1: Error Handling Review

### CHECK

> These checks duplicate coder patterns. See `/golang:coder` Section 3 for full error handling rules with code examples.
>
> Verify: errors checked `[MUST]` · wrapped with %w `[SHOULD]` · consistent %w/%v strategy `[SHOULD]` · sentinels are package-level `[SHOULD]` · lowercase messages `[SHOULD]`

### FLAG (code smells to raise)

- **[MAY] Flag errors.New inside loops**: Allocates a new error value every iteration; only relevant in hot paths [Mistakes#48]
  ```go
  // BAD — allocates on every iteration
  for _, v := range items {
      if v < 0 { return errors.New("negative value") }
  }
  ```

- **[MUST] Flag panic in library code**: Libraries must return errors, not crash the caller [EffectiveGo]
  ```go
  // BAD — library function should not panic
  func Parse(data []byte) Config {
      if len(data) == 0 { panic("empty data") }
  }
  ```

- **[MUST] Flag logging AND returning the same error**: Produces duplicate noise in logs [Mistakes#49]
  ```go
  // BAD — caller will also log the returned error
  if err != nil {
      log.Printf("failed: %v", err)
      return fmt.Errorf("operation failed: %w", err)
  }
  ```

- **[SHOULD] Flag ignoring deferred Close() errors**: Write-path Close can fail and lose data [Mistakes#50]
  ```go
  // BAD — file write may be incomplete
  defer f.Close()
  // GOOD — check error on write path
  defer func() { closeErr = f.Close() }()
  ```

> Cross-ref: See `/golang:coder` Section 3 for full error handling patterns.

---

## Section 2: Concurrency Safety

### CHECK

> These checks duplicate coder patterns. See `/golang:coder` Section 5 for full concurrency rules with code examples.
>
> Verify: goroutine exit path `[MUST]` · shared state synchronized `[MUST]` · WaitGroup.Add before goroutine `[MUST]` · defer Unlock after Lock `[MUST]` · channel direction in params `[SHOULD]`

### FLAG (code smells to raise)

- **[MUST] Flag unbuffered channel without guaranteed receiver**: Sender blocks forever if no goroutine reads [Concurrency#3]
  ```go
  // BAD — if nobody reads ch, this goroutine leaks
  ch := make(chan int)
  go func() { ch <- expensiveCalc() }()
  ```

- **[MUST] Flag sync primitives copied by value**: Copying a mutex or WaitGroup breaks internal state [Mistakes#73]
  ```go
  // BAD — mu is copied, lock is meaningless
  type Cache struct{ mu sync.Mutex; data map[string]string }
  func (c Cache) Get(k string) string { c.mu.Lock(); defer c.mu.Unlock(); return c.data[k] }
  // Use pointer receiver: func (c *Cache) Get(...)
  ```

- **[MUST] Flag closure capturing loop variable in goroutine**: Variable is shared across iterations in Go < 1.22 [Mistakes#63]
  ```go
  // BAD — all goroutines see last value of v (Go < 1.22)
  for _, v := range items {
      go func() { process(v) }()
  }
  ```

- **[SHOULD] Flag mixing mutex and channel for same data**: Pick one synchronization mechanism per data path [Concurrency#4]
  ```go
  // BAD — two mechanisms guarding the same counter
  mu.Lock()
  counter++
  mu.Unlock()
  ch <- counter // also sending through channel
  ```

- **[MUST] Flag blocking goroutine without select on context**: Goroutine cannot be cancelled [Concurrency#5]
  ```go
  // BAD — no way to cancel this goroutine
  go func() {
      result := <-longRunningCh
      process(result)
  }()
  ```

- **[MUST] Flag goroutine launched without clear ownership**: Caller must know who stops it [Concurrency#4]
  ```go
  // BAD — fire and forget, no shutdown mechanism
  func StartProcessor() {
      go func() { for { processNext() } }()
  }
  ```

> Cross-ref: See `/golang:tester` Section 8 for race detection testing patterns.

---

## Section 3: Naming & Style

### CHECK (things that must be present/correct)

- **[MUST] Verify MixedCaps with no underscores**: Go convention is MixedCaps or mixedCaps, never snake_case [CodeReview]
  ```go
  // GOOD
  type UserAccount struct{}
  var maxRetryCount int
  func parseRequestBody() {}
  ```

- **[SHOULD] Verify short names for narrow scope**: Single-letter or abbreviated names for local variables with small scope [EffectiveGo]
  ```go
  // GOOD — short scope, short name
  for i, v := range items { use(i, v) }
  func (s *Server) handleReq(w http.ResponseWriter, r *http.Request) {}
  ```

- **[MUST] Verify acronyms are all-caps**: HTTP, URL, ID, API, SQL — not Http, Url, Id [CodeReview]
  ```go
  // GOOD
  type HTTPClient struct{}
  var userID int
  func ServeHTTP(w http.ResponseWriter, r *http.Request) {}
  ```

- **[MUST] Verify package names are lowercase single words**: No underscores, no mixedCaps in package names [EffectiveGo]
  ```go
  // GOOD
  package user
  package httputil
  package testhelper
  ```

- **[SHOULD] Verify single-method interfaces use -er suffix**: Reader, Writer, Closer, Formatter [EffectiveGo]
  ```go
  // GOOD
  type Validator interface { Validate() error }
  type Renderer interface  { Render(w io.Writer) error }
  ```

- **[SHOULD] Verify exported names have doc comments**: Every exported type, function, const, var needs a comment [CodeReview]
  ```go
  // GOOD
  // UserStore persists user records to the database.
  type UserStore struct{}

  // ErrNotFound is returned when the requested resource does not exist.
  var ErrNotFound = errors.New("not found")
  ```

### FLAG (code smells to raise)

- **[SHOULD] Flag name stuttering**: Package name should not repeat in exported identifiers [CodeReview]
  ```go
  // BAD — callers write http.HTTPServer
  package http
  type HTTPServer struct{}
  // GOOD — callers write http.Server
  type Server struct{}
  ```

- **[SHOULD] Flag //nolint without justification**: Every lint suppression must explain why [CodeReview]
  ```go
  // BAD — no reason given
  //nolint:errcheck
  f.Close()
  // GOOD — explains the decision
  //nolint:errcheck // best-effort cleanup, error logged at caller
  f.Close()
  ```

---

## Section 4: API Design

### CHECK

> These checks duplicate coder patterns. See `/golang:coder` Section 2 (interfaces), Section 4 (functions), Section 6 (context) for full rules with code examples.
>
> Verify: interfaces at consumer side `[SHOULD]` · accept interfaces/return structs `[SHOULD]` · functional options for 3+ optional params `[MAY]` · context.Context first param `[MUST]` · error last return `[MUST]`

### FLAG (code smells to raise)

- **[SHOULD] Flag interface with 5+ methods**: Large interfaces reduce substitutability and testability [Mistakes#5]
  ```go
  // BAD — too many methods, hard to mock
  type UserService interface {
      Create(ctx context.Context, u User) error
      Update(ctx context.Context, u User) error
      Delete(ctx context.Context, id int) error
      Get(ctx context.Context, id int) (User, error)
      List(ctx context.Context) ([]User, error)
      Search(ctx context.Context, q string) ([]User, error)
  }
  ```

- **[SHOULD] Flag exported interface with single implementation**: Premature abstraction, test with concrete type instead [Mistakes#6]
  ```go
  // BAD — only one implementation exists, interface adds indirection
  type Notifier interface { Send(msg string) error }
  type EmailNotifier struct{}
  func (e *EmailNotifier) Send(msg string) error { /* ... */ }
  ```

> Cross-ref: See `/golang:coder` Section 2 for full interface and API patterns.

---

## Section 5: Performance Pitfalls

### FLAG

> Performance anti-patterns to flag during review. See `/golang:coder` Section 1 (pre-allocation) and Section 10 (performance) for DO rules with code examples.
>
> Flag: string concat in loops `[SHOULD]` · slice append without pre-alloc `[SHOULD]` · map without size hint `[SHOULD]` · unnecessary pointer for small structs `[SHOULD]` · string/[]byte conversion in hot path `[SHOULD]` · defer in tight loops `[SHOULD]` · range loop copying large values `[SHOULD]`

---

## Section 6: Security

### CHECK (things that must be present/correct)

- **[MUST] Verify user input is validated at boundary**: Validate and sanitize at the entry point, trust internally [LetsGo#11]
  ```go
  // GOOD — validate at handler, pass clean data inward
  func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
      var input CreateRequest
      if err := json.NewDecoder(r.Body).Decode(&input); err != nil { /* 400 */ }
      if err := input.Validate(); err != nil { /* 422 */ }
      h.service.Create(r.Context(), input.ToDomain())
  }
  ```

- **[MUST] Verify SQL uses parameterized queries**: Never interpolate user input into query strings [LetsGo#4]
  ```go
  // GOOD — parameterized, safe from injection
  row := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id = $1", userID)
  ```

- **[MUST] Verify file paths are sanitized with filepath.Clean**: Prevent path traversal attacks [GoBook#1]
  ```go
  // GOOD — normalize and restrict to base directory
  clean := filepath.Clean(userInput)
  full := filepath.Join(baseDir, clean)
  rel, err := filepath.Rel(baseDir, full)
  if err != nil || strings.HasPrefix(rel, "..") { return ErrInvalidPath }
  ```

- **[MUST] Verify TLS for external connections**: All outbound traffic to external services must use TLS [CloudNative#8]
  ```go
  // GOOD
  srv := &http.Server{
      TLSConfig: &tls.Config{MinVersion: tls.VersionTLS12},
  }
  ```

- **[MUST] Verify secrets are not in source or logs**: No hardcoded tokens, passwords, or API keys [CloudNative#8]
  ```go
  // GOOD — read from environment
  apiKey := os.Getenv("API_KEY")
  if apiKey == "" { return errors.New("API_KEY not set") }
  // Never: log.Printf("connecting with key: %s", apiKey)
  ```

---

## Section 7: Package & Project Structure

### CHECK (things that must be present/correct)

- **[MUST] Verify no circular dependencies**: Package A imports B and B imports A will not compile [GoBook#10]
  ```go
  // GOOD — one-way dependency, use interfaces to invert
  // package handler imports package service
  // package service does NOT import package handler
  // handler defines its own interface for what it needs from service
  ```

- **[SHOULD] Verify internal/ is used for private packages**: Packages under internal/ are inaccessible outside the module [GoBook#10]
  ```go
  // GOOD project layout
  // myapp/
  //   cmd/myapp/main.go
  //   internal/user/store.go    ← private to module
  //   internal/order/service.go ← private to module
  //   pkg/httputil/response.go  ← public API (if needed)
  ```

- **[MUST] Verify one package per directory**: Multiple packages in one directory will not compile [EffectiveGo]
  ```go
  // GOOD
  // internal/user/store.go     → package user
  // internal/user/handler.go   → package user
  // internal/order/service.go  → package order
  ```

- **[SHOULD] Verify cmd/ is used for entry points**: Each binary gets its own subdirectory under cmd/ [CloudNative#2]
  ```go
  // GOOD
  // cmd/api/main.go      ← HTTP server binary
  // cmd/worker/main.go   ← background worker binary
  // cmd/migrate/main.go  ← database migration binary
  ```

### FLAG (code smells to raise)

- **[SHOULD] Flag side-effect import without comment**: Blank imports must explain why the side effect is needed [CodeReview]
  ```go
  // BAD — no explanation for blank import
  import _ "net/http/pprof"

  // GOOD — comment explains the side effect
  import _ "net/http/pprof" // Register pprof HTTP handlers
  ```

---

## Quick Reference

| Area | CHECK | FLAG |
|---|---|---|
| Errors | → coder §3 | errors.New in loop, panic in lib, log+return |
| Concurrency | → coder §5 | Unbuffered no reader, copied sync, loop var |
| Naming | MixedCaps, acronyms, -er suffix | Stuttering, //nolint without reason |
| API Design | → coder §2/§4/§6 | 5+ method interface, single impl |
| Performance | → coder §1/§10 | (see cross-ref summary) |
| Security | Input validated, parameterized SQL, TLS | — |
| Structure | No cycles, internal/, cmd/ | Uncommented blank import |
