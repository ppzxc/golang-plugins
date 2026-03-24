---
name: go-reviewer
description: "Use when reviewing Go code in pull requests or auditing Go codebases — checklists for error handling, concurrency safety, naming conventions, API design, and performance pitfalls."
user_invocable: true
---

# Go Reviewer

Code review checklists for production Go. Each item is a yes/no question — flag it if the answer is "no" and there's no clear reason.

Sources: [Go CodeReviewComments](https://go.dev/wiki/CodeReviewComments), [go-concurrency checklist](https://github.com/code-review-checklists/go-concurrency), [Mistakes] #48-#73, [CloudNative].

> See also: `/go-coder` for full pattern explanations. `/go-tester` for test review patterns.

---

## 1. Go Official Style — [go.dev/wiki/CodeReviewComments](https://go.dev/wiki/CodeReviewComments)

### Naming
- [ ] Receiver names: 1-2 letter abbreviation of type name? (`r *Request`, not `this`/`self`)
- [ ] No underscores in Go names? (`MixedCaps` only, except test functions and generated code)
- [ ] Package name: lowercase, singular, no stutter? (`user.Record` not `user.UserRecord`)
- [ ] Acronyms consistent? (`userID`/`UserID` not `userId`/`UserId`)

### Error Strings
- [ ] Error strings lowercase and no trailing punctuation? (`"invalid input"` not `"Invalid input."`)
  - Reason: error strings are composed — `fmt.Errorf("parse %s: %w", ...)` starts a sentence mid-string.

### Code Style
- [ ] Error flow first, happy path un-indented?
  ```go
  // DO
  if err != nil { return err }
  doWork()
  // DON'T
  if err == nil { doWork() }
  ```
- [ ] No `_ = err` (discarding errors without handling)?
- [ ] All exported symbols have doc comments? (`// FunctionName does X`)
- [ ] Import groups: stdlib / external / internal, separated by blank lines?
- [ ] No blank import (`_ "pkg"`) without an explanatory comment?

---

## 2. Error Handling Review — [Mistakes] #48-#53

- [ ] Every `fmt.Errorf` uses `%w` OR there's a deliberate reason to use `%v`?
  - `%w` preserves chain; `%v` breaks it — callers cannot `errors.Is`/`errors.As` upstream.
- [ ] `errors.Is`/`errors.As` can reach the target through the chain?
  - If wrapped with `%v`, `errors.Is(err, ErrNotFound)` silently returns false.
- [ ] No errors silently discarded? (`if err != nil { _ = err }` or empty catch block)
- [ ] `panic` only for programmer bugs (nil callback, broken invariant), not I/O or user errors?
- [ ] `errors.Join` used for combining multiple independent validation errors (Go 1.20+)?
- [ ] Goroutines collecting errors via `errgroup` or buffered `chan error`? (not fire-and-forget)

---

## 3. Context Review — [Mistakes] #60-#63

- [ ] `ctx context.Context` is the first parameter in every function that does I/O?
- [ ] `defer cancel()` immediately follows every `WithCancel`/`WithTimeout`/`WithDeadline`?
  - Missing cancel leaks the context until parent is cancelled.
- [ ] No `context.Context` stored in a struct field?
  - Struct context prevents per-call cancellation and makes lifetime ambiguous.
- [ ] Context values use unexported key types (not `string`, `int`)?
  - String keys from different packages can collide silently.
- [ ] Context values are request-scoped only (trace IDs, auth tokens)?
  - Not optional function parameters — use explicit function args for those.
- [ ] Long loops check `ctx.Done()` or `ctx.Err()` before each iteration?

---

## 4. Concurrency Safety Review — [Mistakes] #57-#73, [go-concurrency checklist](https://github.com/code-review-checklists/go-concurrency)

### Goroutine Lifecycle
- [ ] Every goroutine has a known exit condition? (context, channel close, or explicit signal)
- [ ] The function that starts a goroutine owns its termination?

### Shared State
- [ ] All accesses to shared mutable state protected by `sync.Mutex` or `sync.RWMutex`?
- [ ] HTTP handlers are goroutine-safe? (no unprotected package-level state)
- [ ] `sync.Map.Store`/`Load` pairs replaced with `LoadOrStore` where atomicity is needed?
- [ ] Methods returning pointers to mutex-protected struct fields avoided?
  ```go
  // WRONG: caller can mutate without holding lock
  func (s *Store) GetUser() *User { return s.user }
  // RIGHT: return a copy
  func (s *Store) GetUser() User { s.mu.Lock(); defer s.mu.Unlock(); return *s.user }
  ```

### Channels
- [ ] Channel capacity is justified by comment? (unbuffered, 1, or N — each has different semantics)
- [ ] No goroutines blocked on unbuffered channel send with no guaranteed reader?
- [ ] Loop variables captured by value before spawning goroutines? (required in Go < 1.22)
  ```go
  for _, v := range items {
      v := v // capture — remove in Go 1.22+
      go func() { use(v) }()
  }
  ```

### Sync Primitives
- [ ] `sync.RWMutex` used for read-heavy, write-rarely workloads? (not always faster — benchmark)
- [ ] `sync.Once` used for one-time initialization instead of `init()`?

> See also: `/go-tester` Section 4 for running `go test -race ./...` to detect races automatically.

---

## 5. Interface & API Design Review — [Mistakes] #5-#8, [LearningGo] Ch.7

- [ ] Interfaces defined at the consumer side, not the producer side?
  - If `FooService` defines its own `FooServiceInterface` in the same package, it's likely producer-side.
- [ ] Interface has ≤3 methods? (larger interfaces reduce substitutability)
- [ ] No stutter in exported names? (`user.UserStore` → `user.Store`)
- [ ] Exported surface area minimal? (unexported what doesn't need to be public)
- [ ] Constructors with many optional parameters use functional options? (forward-compatible)
- [ ] Constructor returns concrete type (not interface) unless multiple implementations exist?
- [ ] Compile-time interface check present for critical implementations?
  ```go
  var _ io.Writer = (*MyWriter)(nil)
  ```
- [ ] Functions accept interfaces, return concrete structs?

---

## 6. Service Structure Review — [CloudNative] Ch.2, [GoBook] Ch.10

- [ ] `internal/` used for packages not intended for external use?
- [ ] `cmd/` contains only wiring and startup — business logic in `internal/`?
- [ ] No circular dependencies between packages?
- [ ] No global mutable state (package-level `var` that handlers read/write)?
  - Exception: `sync.Once`-initialized read-only singletons are fine.
- [ ] Package names: lowercase, singular, no underscores, no stutter?
- [ ] Middleware uses standard `func(http.Handler) http.Handler` signature?
- [ ] Dependencies injected via constructor, not fetched globally in methods?
- [ ] No `src/` directory? (Java convention — confusing in Go)

---

## 7. Resilience Review — [CloudNative] Ch.5-8

- [ ] Retry logic includes **both** exponential backoff **and** jitter?
  - Backoff without jitter causes thundering herd.
- [ ] Circuit breaker applied to all external service calls?
- [ ] Graceful shutdown handles `SIGINT`/`SIGTERM` with a timeout?
- [ ] In-flight requests given time to complete before shutdown? (`srv.Shutdown` not `srv.Close`)
- [ ] Rate limiting applied to public endpoints?
- [ ] `/healthz` (liveness) and `/readyz` (readiness) endpoints implemented?
  - Readiness checks downstream dependencies (DB, cache); liveness does not.

---

## 8. Performance Review — [Mistakes] #20-#29, #39, #47, #95-#97

### Slice & Map
- [ ] `make([]T, 0, cap)` used when final size is known?
- [ ] `append` result always assigned back? (`s = append(s, item)` not `append(s, item)`)
- [ ] Sub-slice (`orig[a:b]`) does not outlive the backing array unexpectedly?
- [ ] No write to nil map? (`var m map[string]int; m["k"] = 1` panics)
- [ ] JSON API responses use `make([]T, 0)` not `var s []T` for empty lists?
  - `nil` slice marshals to `null`; empty slice marshals to `[]`.

### Strings
- [ ] No string concatenation with `+` in a loop? (`strings.Builder` is O(n), `+` is O(n²))

### defer
- [ ] No `defer` inside a loop? (defers stack until function returns, not loop iteration)
  - Wrap in an inner function: `for _, f := range files { func() { defer f.Close(); ... }() }`

### Memory
- [ ] Sub-slice returned from function copies data if the original is large?
- [ ] No goroutines blocked forever (potential goroutine leak)?

> See also: `/go-tester` Section 5 for benchmarking to measure actual impact.

---

## 9. Security Review

- [ ] No SQL queries built with string concatenation or `fmt.Sprintf`? (use parameterized queries)
- [ ] No user input rendered directly into HTML templates without escaping? (use `html/template`, not `text/template`)
- [ ] Secret comparison uses `subtle.ConstantTimeCompare`, not `==`? (prevents timing attacks)
- [ ] TLS configured with minimum version `tls.VersionTLS12`?
- [ ] No sensitive data (passwords, tokens) logged?
- [ ] File paths from user input validated/sanitized? (path traversal risk)
- [ ] External URLs fetched only from an allowlist? (SSRF risk if user controls the URL)

---

## Quick Reference

| Area | Key questions |
|---|---|
| Naming | MixedCaps? No stutter? Receiver abbreviation? |
| Errors | %w intentional? errors.Is reachable? No silent discard? |
| Context | First param? defer cancel()? No struct field? Unexported key? |
| Goroutines | Exit condition? Shared state locked? Loop var captured? |
| Interface | Consumer-side? ≤3 methods? Concrete return? |
| Structure | internal/ correct? No globals? No circular deps? |
| Resilience | Retry has jitter? Circuit breaker? Graceful shutdown? |
| Performance | Pre-alloc? No + in loop? No defer in loop? |
| Security | Parameterized SQL? html/template? ConstantTimeCompare? |
