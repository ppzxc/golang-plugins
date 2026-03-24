# Go Skills Split Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Split the monolithic go-coder skill (960 lines) into three focused skills: go-coder (writing), go-reviewer (reviewing), go-tester (testing).

**Architecture:** Create two new plugin directories (`go-tester`, `go-reviewer`) alongside the existing `go-coder`. Each plugin has its own `plugin.json`, `SKILL.md`, and READMEs. Update `marketplace.json` to register all three. go-coder is trimmed (Section 6 removed, new slog/DI/config content added).

**Tech Stack:** Markdown SKILL.md files, Claude Code plugin system, JSON plugin manifests.

---

## File Map

### Created
- `plugins/go-tester/.claude-plugin/plugin.json`
- `plugins/go-tester/skills/go-tester/SKILL.md`
- `plugins/go-tester/README.md`
- `plugins/go-tester/README.ko.md`
- `plugins/go-reviewer/.claude-plugin/plugin.json`
- `plugins/go-reviewer/skills/go-reviewer/SKILL.md`
- `plugins/go-reviewer/README.md`
- `plugins/go-reviewer/README.ko.md`

### Modified
- `plugins/go-coder/skills/go-coder/SKILL.md` — remove Section 6, trim race detection, add slog/DI/config
- `plugins/go-coder/.claude-plugin/plugin.json` — update description
- `plugins/go-coder/README.md` — update skill list
- `plugins/go-coder/README.ko.md` — update skill list
- `.claude-plugin/marketplace.json` — add go-reviewer, go-tester plugins
- `README.md` — add two plugins to table and project structure
- `README.ko.md` — same

---

## Task 1: go-tester plugin — manifests and READMEs

**Files:**
- Create: `plugins/go-tester/.claude-plugin/plugin.json`
- Create: `plugins/go-tester/README.md`
- Create: `plugins/go-tester/README.ko.md`

- [ ] **Step 1: Create plugin directory structure**

```bash
mkdir -p plugins/go-tester/.claude-plugin
mkdir -p plugins/go-tester/skills/go-tester
```

- [ ] **Step 2: Write plugin.json**

Write `plugins/go-tester/.claude-plugin/plugin.json`:
```json
{
  "name": "go-tester",
  "version": "0.0.1",
  "description": "Go testing patterns — table-driven tests, benchmarks, fuzzing, mocks, integration tests, race detection, and coverage",
  "author": {
    "name": "ppzxc",
    "email": "cjh8487@naver.com",
    "url": "https://ppzxc.github.io"
  }
}
```

- [ ] **Step 3: Write README.md**

Write `plugins/go-tester/README.md`:
```markdown
# go-tester

A Go testing patterns reference skill for Claude Code.

## Contents

1. **Table-Driven Tests** — t.Run, struct slices, parallel subtests
2. **Test Helpers** — t.Helper, t.Cleanup, t.Parallel
3. **Mocks** — manual mock via interface, function-field pattern
4. **Data Race Detection** — go test -race
5. **Benchmarks** — b.ResetTimer, b.ReportAllocs, sub-benchmarks, benchstat
6. **Fuzzing** — testing.F, seed corpus (Go 1.18+)
7. **Integration Testing** — testcontainers-go, build tags
8. **httptest** — handler and middleware testing
9. **Golden File Testing** — testdata/ convention, -update flag
10. **Test Lifecycle** — TestMain, testing.TB helpers
11. **Coverage & Profiling** — -cover, -cpuprofile, -memprofile

## Installation

```bash
/plugin marketplace add ppzxc/go-plugins
/plugin install go-tester
```

## Usage

```
/go-tester
```

Also activates automatically when writing, debugging, or reviewing Go tests.
```

- [ ] **Step 4: Write README.ko.md**

Write `plugins/go-tester/README.ko.md`:
```markdown
# go-tester

Go 테스트 패턴 레퍼런스 스킬입니다.

## 포함 내용

1. **Table-Driven Tests** — t.Run, 병렬 서브테스트
2. **Test Helpers** — t.Helper, t.Cleanup, t.Parallel
3. **Mocks** — 인터페이스 기반 수동 mock
4. **Data Race Detection** — go test -race
5. **Benchmarks** — sub-benchmark, benchstat
6. **Fuzzing** — testing.F, seed corpus (Go 1.18+)
7. **Integration Testing** — testcontainers-go, build 태그
8. **httptest** — 핸들러/미들웨어 테스트
9. **Golden File Testing** — testdata/ 컨벤션
10. **Test Lifecycle** — TestMain, testing.TB 헬퍼
11. **Coverage & Profiling** — -cover, -cpuprofile

## 설치

```bash
/plugin marketplace add ppzxc/go-plugins
/plugin install go-tester
```

## 사용 방법

```
/go-tester
```

Go 테스트 작성, 디버깅, 리뷰 시 자동으로 활성화됩니다.
```

- [ ] **Step 5: Verify files exist**

```bash
ls plugins/go-tester/.claude-plugin/plugin.json
ls plugins/go-tester/README.md
ls plugins/go-tester/README.ko.md
```
Expected: all three files listed.

---

## Task 2: go-tester SKILL.md

**Files:**
- Create: `plugins/go-tester/skills/go-tester/SKILL.md`

The SKILL.md assembles content from two sources:
1. **Moved from go-coder SKILL.md** — Section 6 (lines 594-695) and Data Race Detection subsection (lines 365-372)
2. **New content** — Fuzzing, Integration Testing, httptest, Golden File, TestMain, Coverage

- [ ] **Step 1: Write SKILL.md frontmatter and intro**

Write `plugins/go-tester/skills/go-tester/SKILL.md` starting with:

```markdown
---
name: go-tester
description: "Use when writing, debugging, or improving Go tests — table-driven tests, benchmarks, fuzzing, mocks, integration tests, race detection, and coverage."
user_invocable: true
---

# Go Tester

Testing patterns for production Go services.

Sources: [LearningGo] Ch.13, [Mistakes] #82-#84, Go 1.18+ fuzzing docs, testcontainers-go.

> See also: `/go-coder` for interface design patterns (used in manual mocks). `/go-reviewer` for code review checklists.

---
```

- [ ] **Step 2: Add Section 1 — Table-Driven Tests (moved from go-coder Section 6, lines 597-621)**

```markdown
## 1. Table-Driven Tests — [LearningGo] Ch.13

### t.Run Pattern

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
```

- [ ] **Step 3: Add Section 2 — Test Helpers (moved from go-coder Section 6, lines 624-657)**

```markdown
## 2. Test Helpers — [Mistakes] #82-#84

### t.Helper()

Call `t.Helper()` in every test helper. Without it, failure line points to the helper, not the test case.

```go
func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil { t.Fatalf("unexpected error: %v", err) }
}
```

### t.Cleanup()

Prefer `t.Cleanup()` over `defer` in tests. `defer` is skipped if `t.FailNow()` is called; `t.Cleanup()` always runs.

```go
func TestWithDB(t *testing.T) {
    db := setupDB(t)
    t.Cleanup(func() { db.Close() })  // always runs, even after t.Fatal
}
```

### t.Parallel()

```go
for _, tt := range tests {
    tt := tt  // capture range variable (required in Go < 1.22)
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()  // at the top, after t.Helper() if needed
        // never share mutable state between parallel subtests
    })
}
```
```

- [ ] **Step 4: Add Section 3 — Manual Mocks (moved from go-coder Section 6, lines 659-681)**

```markdown
## 3. Manual Mocks via Interface

Define interface in the package that needs it. Use function-field mock struct — no code generator needed for simple cases.

```go
type UserStore interface {
    FindByID(ctx context.Context, id string) (*User, error)
}

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
```

- [ ] **Step 5: Add Section 4 — Data Race Detection (moved from go-coder Section 4)**

```markdown
## 4. Data Race Detection — [Mistakes] #58

Always run tests with `-race`. Races are undefined behavior — they cannot be safely ignored.

```bash
go test -race ./...
go run -race main.go
```

Enable in CI. The race detector has ~5-10x slowdown — acceptable for test suites.

> See also: `/go-reviewer` Concurrency Safety checklist for reviewing concurrent code.
```

- [ ] **Step 6: Add Section 5 — Benchmarks (moved + expanded from go-coder lines 683-695)**

```markdown
## 5. Benchmarks

### Basic Benchmark

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

### Sub-Benchmarks

```go
func BenchmarkEncode(b *testing.B) {
    sizes := []int{10, 100, 1000}
    for _, n := range sizes {
        b.Run(fmt.Sprintf("size=%d", n), func(b *testing.B) {
            data := make([]byte, n)
            b.SetBytes(int64(n))  // report MB/s
            b.ResetTimer()
            for b.Loop() { encode(data) }
        })
    }
}
```

### Avoid Compiler Optimization

```go
var sink any  // package-level variable

func BenchmarkCompute(b *testing.B) {
    var result int
    for b.Loop() {
        result = compute(42)  // compiler may optimize away if unused
    }
    sink = result  // assign to package-level: prevents dead code elimination
}
```

### Compare Benchmarks with benchstat

```bash
go test -bench=. -count=5 ./... > old.txt
# make changes
go test -bench=. -count=5 ./... > new.txt
benchstat old.txt new.txt
```

### Custom Metrics

```go
b.ReportMetric(float64(allocBytes)/float64(b.N), "bytes/op")
```
```

- [ ] **Step 7: Add Section 6 — Fuzzing (new content)**

```markdown
## 6. Fuzzing — Go 1.18+

Use for functions that accept arbitrary user input: parsers, deserializers, input validators.

```go
// Signature: FuzzXxx(f *testing.F)
func FuzzParseUser(f *testing.F) {
    // Seed corpus — initial interesting inputs
    f.Add(`{"name":"alice"}`)
    f.Add(`{"name":""}`)
    f.Add(`{}`)

    f.Fuzz(func(t *testing.T, data string) {
        // Must not panic — any panic = bug
        user, err := ParseUser([]byte(data))
        if err != nil { return }  // error is OK, panic is not
        // Invariant: if parsed successfully, name must not be empty
        if user.Name == "" {
            t.Errorf("parsed user with empty name from: %q", data)
        }
    })
}
```

```bash
# Run fuzzer (infinite loop until failure or ctrl-c)
go test -fuzz=FuzzParseUser ./...

# Run seed corpus only (fast, for CI)
go test -run=FuzzParseUser ./...
```

Corpus files are stored in `testdata/fuzz/FuzzParseUser/`. Commit found failures as regression tests.

**When to fuzz:** input validation, JSON/XML/binary parsers, serialization round-trips, URL parsing, cryptographic operations.
```

- [ ] **Step 8: Add Section 7 — Integration Testing (new content)**

```markdown
## 7. Integration Testing

### Build Tag Separation

```go
//go:build integration

package myservice_test
```

```bash
# Unit tests only (default)
go test ./...

# Include integration tests
go test -tags integration ./...
```

### testcontainers-go

```go
//go:build integration

func TestUserStore_Integration(t *testing.T) {
    ctx := context.Background()

    // Start a real Postgres container
    pgContainer, err := postgres.Run(ctx,
        "postgres:16-alpine",
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("user"),
        postgres.WithPassword("password"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2).WithStartupTimeout(30*time.Second),
        ),
    )
    require.NoError(t, err)
    t.Cleanup(func() {
        if err := pgContainer.Terminate(ctx); err != nil {
            t.Logf("failed to terminate container: %v", err)
        }
    })

    connStr, err := pgContainer.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    store, err := NewUserStore(connStr)
    require.NoError(t, err)

    // Now test against a real database
    user, err := store.Create(ctx, "alice@example.com")
    require.NoError(t, err)
    assert.NotEmpty(t, user.ID)
}
```

`go get github.com/testcontainers/testcontainers-go`
```

- [ ] **Step 9: Add Section 8 — httptest (new content)**

```markdown
## 8. httptest — HTTP Handler Testing

### Unit Style: httptest.NewRecorder

```go
func TestGetUser(t *testing.T) {
    store := &MockUserStore{
        FindByIDFn: func(_ context.Context, id string) (*User, error) {
            return &User{ID: id, Name: "alice"}, nil
        },
    }
    h := &UserHandler{store: store}

    req := httptest.NewRequest(http.MethodGet, "/users/123", nil)
    req.SetPathValue("id", "123")
    w := httptest.NewRecorder()

    h.GetUser(w, req)

    resp := w.Result()
    assert.Equal(t, http.StatusOK, resp.StatusCode)
    assert.Equal(t, "application/json", resp.Header.Get("Content-Type"))
}
```

### Integration Style: httptest.NewServer

```go
func TestServer_Integration(t *testing.T) {
    srv := httptest.NewServer(setupRouter())
    t.Cleanup(srv.Close)

    resp, err := http.Get(srv.URL + "/users/123")
    require.NoError(t, err)
    defer resp.Body.Close()
    assert.Equal(t, http.StatusOK, resp.StatusCode)
}
```

### Middleware Testing

```go
func TestLoggingMiddleware(t *testing.T) {
    var logBuf bytes.Buffer
    log := slog.New(slog.NewTextHandler(&logBuf, nil))
    middleware := Logging(log)

    handler := middleware(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    }))

    req := httptest.NewRequest(http.MethodGet, "/test", nil)
    w := httptest.NewRecorder()
    handler.ServeHTTP(w, req)

    assert.Contains(t, logBuf.String(), "GET")
    assert.Contains(t, logBuf.String(), "/test")
}
```
```

- [ ] **Step 10: Add Section 9 — Golden File Testing (new content)**

```markdown
## 9. Golden File Testing

Use when expected output is large (JSON responses, rendered HTML, generated code) and checking it inline would be unwieldy.

```go
var update = flag.Bool("update", false, "update golden files")

func TestRenderReport(t *testing.T) {
    got := renderReport(testData)
    golden := filepath.Join("testdata", t.Name()+".golden")

    if *update {
        os.WriteFile(golden, got, 0644)
        return
    }

    want, err := os.ReadFile(golden)
    require.NoError(t, err, "run with -update to create golden file")
    assert.Equal(t, string(want), string(got))
}
```

```bash
# Create/update golden files
go test -run TestRenderReport -update ./...

# Normal test run (compare against golden files)
go test -run TestRenderReport ./...
```

Golden files live in `testdata/` — commit them to version control.
```

- [ ] **Step 11: Add Section 10 — Test Lifecycle (new content)**

```markdown
## 10. Test Lifecycle

### TestMain — Global Setup and Teardown

```go
func TestMain(m *testing.M) {
    // Setup: runs once before all tests in the package
    db, cleanup := setupTestDB()

    code := m.Run()  // run all tests

    // Teardown: runs once after all tests
    cleanup()
    db.Close()

    os.Exit(code)
}
```

### Custom testing.TB Helper

Works with both `*testing.T` (tests) and `*testing.B` (benchmarks):

```go
func setupStore(tb testing.TB) *Store {
    tb.Helper()
    store, err := NewStore(":memory:")
    if err != nil {
        tb.Fatalf("setup store: %v", err)
    }
    tb.Cleanup(func() { store.Close() })
    return store
}

// Used in both tests and benchmarks
func TestFoo(t *testing.T) { store := setupStore(t); ... }
func BenchmarkFoo(b *testing.B) { store := setupStore(b); ... }
```
```

- [ ] **Step 12: Add Section 11 — Coverage and Profiling (new content)**

```markdown
## 11. Coverage & Profiling

### Coverage

```bash
# Run with coverage
go test -cover ./...

# Generate HTML report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html

# Coverage threshold check (in CI)
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out | tail -1  # shows total %
```

### CPU and Memory Profiling in Benchmarks

```bash
go test -bench=BenchmarkParseUser -cpuprofile=cpu.out -memprofile=mem.out ./...

# Analyze CPU profile
go tool pprof -http=:8080 cpu.out

# Analyze memory profile
go tool pprof -http=:8080 mem.out
```

### Profiling in Production (pprof endpoint)

```go
import _ "net/http/pprof"  // registers /debug/pprof/ handlers

// curl http://localhost:6060/debug/pprof/goroutine?debug=2
```
```

- [ ] **Step 13: Add Quick Reference table (testing rows from original)**

```markdown
## Quick Reference

| Situation | Pattern |
|---|---|
| Multiple test cases, same function | Table-driven with `t.Run` |
| Failure line in test, not helper | `t.Helper()` in helper func |
| Cleanup guaranteed after `t.Fatal` | `t.Cleanup()` not `defer` |
| Parallel subtests | `t.Parallel()` + capture loop var |
| Simple mock | Function-field struct implementing interface |
| Detect data races | `go test -race ./...` |
| Arbitrary input testing | Fuzzing with `testing.F` |
| Real database/service in test | testcontainers-go + `//go:build integration` |
| Test HTTP handler | `httptest.NewRecorder` (unit) / `httptest.NewServer` (integration) |
| Large expected output | Golden file in `testdata/` |
| Global test setup | `TestMain(m *testing.M)` |
| Coverage report | `go test -coverprofile=c.out && go tool cover -html=c.out` |
| Compare benchmark results | `benchstat old.txt new.txt` |
```

- [ ] **Step 14: Verify SKILL.md**

```bash
wc -l plugins/go-tester/skills/go-tester/SKILL.md
```
Expected: ~500 lines. Check frontmatter, section headers, all code blocks are valid markdown.

- [ ] **Step 15: Commit**

```bash
git add plugins/go-tester/
git commit -m "feat(go-tester): add go-tester plugin with testing patterns"
```

---

## Task 3: go-reviewer plugin — manifests and READMEs

**Files:**
- Create: `plugins/go-reviewer/.claude-plugin/plugin.json`
- Create: `plugins/go-reviewer/README.md`
- Create: `plugins/go-reviewer/README.ko.md`

- [ ] **Step 1: Create plugin directory structure**

```bash
mkdir -p plugins/go-reviewer/.claude-plugin
mkdir -p plugins/go-reviewer/skills/go-reviewer
```

- [ ] **Step 2: Write plugin.json**

Write `plugins/go-reviewer/.claude-plugin/plugin.json`:
```json
{
  "name": "go-reviewer",
  "version": "0.0.1",
  "description": "Go code review checklists — error handling, concurrency safety, naming conventions, API design, performance pitfalls, and security",
  "author": {
    "name": "ppzxc",
    "email": "cjh8487@naver.com",
    "url": "https://ppzxc.github.io"
  }
}
```

- [ ] **Step 3: Write README.md**

Write `plugins/go-reviewer/README.md`:
```markdown
# go-reviewer

A Go code review checklist skill for Claude Code.

## Contents

1. **Go Official Style** — naming, error strings, doc comments, import grouping
2. **Error Handling** — wrapping, sentinel errors, error discarding
3. **Context** — propagation, cancellation, value usage
4. **Concurrency Safety** — goroutine lifecycle, mutex usage, race conditions
5. **Interface & API Design** — surface area, forward-compatibility
6. **Service Structure** — project layout, package naming, dependency direction
7. **Resilience** — retry, circuit breaker, graceful shutdown
8. **Performance** — slice/map pitfalls, string building, memory leaks
9. **Security** — injection, input validation, TLS

## Installation

```bash
/plugin marketplace add ppzxc/go-plugins
/plugin install go-reviewer
```

## Usage

```
/go-reviewer
```

Also activates automatically when reviewing Go code or pull requests.
```

- [ ] **Step 4: Write README.ko.md**

Write `plugins/go-reviewer/README.ko.md`:
```markdown
# go-reviewer

Go 코드 리뷰 체크리스트 스킬입니다.

## 포함 내용

1. **Go 공식 스타일** — 네이밍, 에러 문자열, doc comments, import 그룹핑
2. **Error Handling** — wrapping, sentinel 에러, 에러 무시
3. **Context** — 전달, 취소, value 사용
4. **동시성 안전** — goroutine lifecycle, mutex 사용, race condition
5. **인터페이스 & API 설계** — 노출 범위, forward-compatibility
6. **서비스 구조** — 프로젝트 레이아웃, 패키지 네이밍, 의존성 방향
7. **복원력** — retry, circuit breaker, graceful shutdown
8. **성능** — slice/map 함정, string 빌딩, 메모리 릭
9. **보안** — injection, 입력 검증, TLS

## 설치

```bash
/plugin marketplace add ppzxc/go-plugins
/plugin install go-reviewer
```

## 사용 방법

```
/go-reviewer
```

Go 코드 리뷰 또는 PR 리뷰 시 자동으로 활성화됩니다.
```

- [ ] **Step 5: Verify**

```bash
ls plugins/go-reviewer/.claude-plugin/plugin.json plugins/go-reviewer/README.md plugins/go-reviewer/README.ko.md
```

---

## Task 4: go-reviewer SKILL.md

**Files:**
- Create: `plugins/go-reviewer/skills/go-reviewer/SKILL.md`

Format: All sections are **checklists** — quick scan during PR review. Each item is `- [ ] question?` with brief rationale inline.

- [ ] **Step 1: Write frontmatter and intro**

Write `plugins/go-reviewer/skills/go-reviewer/SKILL.md`:

```markdown
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
```

- [ ] **Step 2: Add Section 1 — Go Official Style (new, from CodeReviewComments)**

```markdown
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
```

- [ ] **Step 3: Add Section 2 — Error Handling Review**

```markdown
## 2. Error Handling Review — [Mistakes] #48-#53

- [ ] Every `fmt.Errorf` uses `%w` OR there's a deliberate reason to use `%v`?
  - `%w` preserves chain; `%v` breaks it — callers cannot `errors.Is`/`errors.As` upstream.
- [ ] `errors.Is`/`errors.As` can reach the target through the chain?
  - If wrapped with `%v`, `errors.Is(err, ErrNotFound)` silently returns false.
- [ ] No errors silently discarded? (`if err != nil { _ = err }` or empty catch block)
- [ ] `panic` only for programmer bugs (nil callback, broken invariant), not I/O or user errors?
- [ ] `errors.Join` used for combining multiple independent validation errors (Go 1.20+)?
- [ ] Goroutines collecting errors via `errgroup` or buffered `chan error`? (not fire-and-forget)
```

- [ ] **Step 4: Add Section 3 — Context Review**

```markdown
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
```

- [ ] **Step 5: Add Section 4 — Concurrency Safety Review**

```markdown
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
      v := v  // capture — remove in Go 1.22+
      go func() { use(v) }()
  }
  ```

### Sync Primitives
- [ ] `sync.RWMutex` used for read-heavy, write-rarely workloads? (not always faster — benchmark)
- [ ] `sync.Once` used for one-time initialization instead of `init()`?

> See also: `/go-tester` Section 4 for running `go test -race ./...` to detect races automatically.
```

- [ ] **Step 6: Add Section 5 — Interface & API Design Review**

```markdown
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
```

- [ ] **Step 7: Add Section 6 — Service Structure Review**

```markdown
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
```

- [ ] **Step 8: Add Section 7 — Resilience Review**

```markdown
## 7. Resilience Review — [CloudNative] Ch.5-8

- [ ] Retry logic includes **both** exponential backoff **and** jitter?
  - Backoff without jitter causes thundering herd.
- [ ] Circuit breaker applied to all external service calls?
- [ ] Graceful shutdown handles `SIGINT`/`SIGTERM` with a timeout?
- [ ] In-flight requests given time to complete before shutdown? (`srv.Shutdown` not `srv.Close`)
- [ ] Rate limiting applied to public endpoints?
- [ ] `/healthz` (liveness) and `/readyz` (readiness) endpoints implemented?
  - Readiness checks downstream dependencies (DB, cache); liveness does not.
```

- [ ] **Step 9: Add Section 8 — Performance Review**

```markdown
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
```

- [ ] **Step 10: Add Section 9 — Security Review (new)**

```markdown
## 9. Security Review

- [ ] No SQL queries built with string concatenation or `fmt.Sprintf`? (use parameterized queries)
- [ ] No user input rendered directly into HTML templates without escaping? (use `html/template`, not `text/template`)
- [ ] Secret comparison uses `subtle.ConstantTimeCompare`, not `==`? (prevents timing attacks)
- [ ] TLS configured with minimum version `tls.VersionTLS12`?
- [ ] No sensitive data (passwords, tokens) logged?
- [ ] File paths from user input validated/sanitized? (path traversal risk)
- [ ] External URLs fetched only from an allowlist? (SSRF risk if user controls the URL)
```

- [ ] **Step 11: Add Quick Reference**

```markdown
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
```

- [ ] **Step 12: Verify SKILL.md**

```bash
wc -l plugins/go-reviewer/skills/go-reviewer/SKILL.md
```
Expected: ~380-420 lines.

- [ ] **Step 13: Commit**

```bash
git add plugins/go-reviewer/
git commit -m "feat(go-reviewer): add go-reviewer plugin with code review checklists"
```

---

## Task 5: Modify go-coder SKILL.md

**Files:**
- Modify: `plugins/go-coder/skills/go-coder/SKILL.md`

Changes:
1. Update frontmatter `description` (narrow scope — remove "reviewing")
2. Remove Section 6 (Testing, lines 594-697) — replace with cross-reference
3. In Section 4, replace "Data Race Detection" subsection with cross-reference
4. Add three new subsections to Section 5 (Service Design): slog, DI, config loading
5. Update Quick Reference — remove testing rows, add slog/DI rows

- [ ] **Step 1: Update frontmatter description**

In `plugins/go-coder/skills/go-coder/SKILL.md`, change line 3:

```
Old: description: "Use when writing, reviewing, or refactoring Go code in production services — error handling, concurrency, interfaces, testing, and resilience patterns."
New: description: "Use when writing or refactoring Go production code — types, interfaces, error handling, concurrency, context, service structure, and resilience patterns."
```

- [ ] **Step 2: Remove Section 6 — Testing (lines 594-697)**

Delete from `## 6. Testing` header through `---` separator at line 697.

Replace with:
```markdown
---

> **Testing patterns** → See `/go-tester` for table-driven tests, benchmarks, fuzzing, mocks, httptest, and coverage.

---
```

- [ ] **Step 3: Replace Data Race Detection subsection in Section 4**

In Section 4 (Concurrency), find the `### Data Race Detection` block (lines ~365-372):

```markdown
### Data Race Detection — [Mistakes] #58

Always run tests with `-race`. Races are undefined behavior — they cannot be safely ignored.

```bash
go test -race ./...
go run -race main.go
```
```

Replace with:
```markdown
> **Race detection** → See `/go-tester` Section 4. Always run `go test -race ./...`.
```

- [ ] **Step 4: Add slog section to Section 5 (Service Design)**

After the `### Constructor Naming` subsection (around line 590), add before the `---` separator:

```markdown
### Structured Logging with slog — Go 1.21+

```go
// Setup: create logger with handler
log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
}))

// Structured fields — key-value pairs
log.Info("request completed",
    "method", r.Method,
    "path", r.URL.Path,
    "status", 200,
    "duration", time.Since(start),
)

// Group related fields
log.With(slog.Group("user",
    "id", userID,
    "role", "admin",
)).Info("user action", "action", "login")

// Error level with error field
log.Error("db query failed", "err", err, "query", "SELECT ...")
```

Inject logger via constructor — never use `slog.Default()` in library code.
```

- [ ] **Step 5: Add Dependency Injection section to Section 5**

```markdown
### Dependency Injection — Constructor Wiring

Wire dependencies manually in `cmd/server/main.go`. No framework needed for most services.

```go
// cmd/server/main.go — the composition root
func main() {
    cfg := config.Load()
    log := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    db, err := sql.Open("postgres", cfg.DSN)
    if err != nil { log.Error("db connect", "err", err); os.Exit(1) }

    // Wire dependencies bottom-up
    userStore := postgres.NewUserStore(db)
    userSvc := user.NewService(userStore, log)
    userHandler := http.NewUserHandler(userSvc, log)

    mux := http.NewServeMux()
    userHandler.Register(mux)

    srv := &http.Server{Addr: cfg.Addr, Handler: Logging(log)(mux)}
    run(srv)
}
```

**Rule:** Each layer only knows about the layer below it. `http.UserHandler` knows `user.Service` interface; it does not know `postgres.UserStore`.
```

- [ ] **Step 6: Add Config Loading section to Section 5**

```markdown
### Configuration Loading

```go
// internal/config/config.go
type Config struct {
    Addr    string        `env:"ADDR"       envDefault:":8080"`
    DSN     string        `env:"DATABASE_URL,required"`
    Timeout time.Duration `env:"TIMEOUT"    envDefault:"30s"`
    Debug   bool          `env:"DEBUG"      envDefault:"false"`
}

func Load() Config {
    var cfg Config
    if err := env.Parse(&cfg); err != nil {
        // Fatal at startup — config errors are programmer/ops errors
        slog.Error("config error", "err", err)
        os.Exit(1)
    }
    return cfg
}
```

`go get github.com/caarlos0/env/v11`

**Validate at startup, not at use.** If a required env var is missing, fail fast with a clear message.
```

- [ ] **Step 7: Update Quick Reference table**

Remove rows:
- `Detect data races` (moved to go-tester)

Add rows after existing entries (existing table has 3 columns: Situation | Pattern | Source):
```markdown
| Structured logging | `slog.New(slog.NewJSONHandler(...))` + inject via constructor | Go 1.21+ |
| Wire dependencies | Constructor injection in `cmd/*/main.go` | [CloudNative] Ch.2 |
| Load config from env | `env.Parse(&cfg)` at startup, fatal on error | — |
```

- [ ] **Step 8: Verify go-coder SKILL.md**

```bash
wc -l plugins/go-coder/skills/go-coder/SKILL.md
grep -n "## 6\." plugins/go-coder/skills/go-coder/SKILL.md  # should find nothing
grep -n "go-tester" plugins/go-coder/skills/go-coder/SKILL.md  # should find cross-references
grep -n "slog" plugins/go-coder/skills/go-coder/SKILL.md  # should find new section
```

Expected: ~700-750 lines, no Section 6 header, cross-references present, slog section exists.

- [ ] **Step 9: Update go-coder plugin.json description**

In `plugins/go-coder/.claude-plugin/plugin.json`, update description:
```json
"description": "Production Go patterns — types, interfaces, error handling, concurrency, service design, resilience, and structured logging"
```

- [ ] **Step 10: Update go-coder README.md and README.ko.md**

In `plugins/go-coder/README.md`, update Contents section to reflect removal of Testing and addition of slog/DI/config:

Remove: `**Testing** — table-driven, t.Helper/Cleanup/Parallel, manual mock, benchmarks`
Add:
```
10. **Structured Logging** — slog setup, handlers, groups
11. **Dependency Injection** — constructor wiring in cmd/
12. **Configuration** — env-based config, startup validation
```

Apply equivalent update to `plugins/go-coder/README.ko.md`.

- [ ] **Step 11: Commit**

```bash
git add plugins/go-coder/
git commit -m "refactor(go-coder): remove testing section, add slog/DI/config patterns"
```

---

## Task 6: Update marketplace.json and root READMEs

**Files:**
- Modify: `.claude-plugin/marketplace.json`
- Modify: `README.md`
- Modify: `README.ko.md`

- [ ] **Step 1: Update marketplace.json — add two plugins**

In `.claude-plugin/marketplace.json`, change the `"plugins"` array to:
```json
"plugins": [
  {
    "name": "go-coder",
    "source": "./plugins/go-coder",
    "description": "Production Go patterns — types, interfaces, error handling, concurrency, service design, resilience, and structured logging",
    "version": "0.0.1"
  },
  {
    "name": "go-reviewer",
    "source": "./plugins/go-reviewer",
    "description": "Go code review checklists — error handling, concurrency safety, naming, API design, performance, security",
    "version": "0.0.1"
  },
  {
    "name": "go-tester",
    "source": "./plugins/go-tester",
    "description": "Go testing patterns — table-driven, benchmarks, fuzzing, mocks, integration tests, race detection, coverage",
    "version": "0.0.1"
  }
]
```

- [ ] **Step 2: Update root README.md — plugin table**

Replace the single-row plugin table with:
```markdown
| Plugin | Version | Description |
|--------|---------|-------------|
| [go-coder](./plugins/go-coder) | 0.0.1 | Production Go patterns — types, interfaces, error handling, concurrency, service design, resilience |
| [go-reviewer](./plugins/go-reviewer) | 0.0.1 | Code review checklists — error handling, concurrency safety, naming, API design, performance, security |
| [go-tester](./plugins/go-tester) | 0.0.1 | Testing patterns — table-driven, benchmarks, fuzzing, mocks, integration tests, coverage |
```

- [ ] **Step 3: Update root README.md — installation section**

Replace the current "Install plugin" step with:
```markdown
### 2. Install plugins

```bash
/plugin install go-coder    # writing production Go code
/plugin install go-reviewer # reviewing Go code
/plugin install go-tester   # writing Go tests
```
```

- [ ] **Step 4: Update root README.md — project structure**

Replace project structure block with:
```markdown
```
go-plugins/
├── .claude-plugin/
│   └── marketplace.json        # Marketplace metadata
└── plugins/
    ├── go-coder/               # Production Go patterns
    │   ├── .claude-plugin/plugin.json
    │   └── skills/go-coder/SKILL.md
    ├── go-reviewer/            # Code review checklists
    │   ├── .claude-plugin/plugin.json
    │   └── skills/go-reviewer/SKILL.md
    └── go-tester/              # Testing patterns
        ├── .claude-plugin/plugin.json
        └── skills/go-tester/SKILL.md
```
```

- [ ] **Step 5: Apply equivalent updates to README.ko.md**

Mirror all changes from Step 2-4 in Korean in `README.ko.md`:
- Plugin table in Korean
- Installation section in Korean
- Project structure (same ASCII, Korean comments)

- [ ] **Step 6: Verify**

```bash
cat .claude-plugin/marketplace.json | python3 -m json.tool  # valid JSON
grep -c "go-reviewer\|go-tester" README.md  # should be ≥ 4
```

- [ ] **Step 7: Commit**

```bash
git add .claude-plugin/marketplace.json README.md README.ko.md
git commit -m "feat: register go-reviewer and go-tester in marketplace"
```

---

## Task 7: Final Verification

- [ ] **Step 1: Verify all SKILL.md files exist**

```bash
ls plugins/go-{coder,reviewer,tester}/skills/*/SKILL.md
```
Expected: 3 files.

- [ ] **Step 2: Verify no go-skills references remain**

```bash
grep -r "go-skills" --include="*.json" --include="*.md" . --exclude-dir=docs --exclude-dir=.claude
```
Expected: no output.

- [ ] **Step 3: Verify frontmatter names**

```bash
grep "^name:" plugins/go-coder/skills/go-coder/SKILL.md
grep "^name:" plugins/go-reviewer/skills/go-reviewer/SKILL.md
grep "^name:" plugins/go-tester/skills/go-tester/SKILL.md
```
Expected: `name: go-coder`, `name: go-reviewer`, `name: go-tester`.

- [ ] **Step 4: Verify cross-references**

```bash
grep "go-tester\|go-reviewer" plugins/go-coder/skills/go-coder/SKILL.md
grep "go-coder\|go-tester" plugins/go-reviewer/skills/go-reviewer/SKILL.md
grep "go-coder\|go-reviewer" plugins/go-tester/skills/go-tester/SKILL.md
```
Expected: cross-references present in each.

- [ ] **Step 5: Verify content migration — no lost topics**

Check each original section exists in at least one skill:
```bash
# Section 6 (Testing) moved to go-tester
grep "Table-Driven" plugins/go-tester/skills/go-tester/SKILL.md

# Race detection moved to go-tester
grep "race" plugins/go-tester/skills/go-tester/SKILL.md

# New content in go-coder
grep "slog" plugins/go-coder/skills/go-coder/SKILL.md

# Checklists in go-reviewer
grep "\- \[" plugins/go-reviewer/skills/go-reviewer/SKILL.md | wc -l  # should be ≥ 40
```

- [ ] **Step 6: Final commit tag**

```bash
git log --oneline -5
```
Expected: 4 commits visible from this feature (go-tester, go-reviewer, go-coder refactor, marketplace update).
