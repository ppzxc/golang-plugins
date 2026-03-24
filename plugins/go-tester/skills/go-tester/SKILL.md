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

---

## 2. Test Helpers — [Mistakes] #82-#84

### t.Helper()

Call `t.Helper()` in every test helper. Without it, failure line points to the helper, not the test case.

```go
func assertNoError(t *testing.T, err error) {
	t.Helper()
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
}
```

### t.Cleanup()

Prefer `t.Cleanup()` over `defer` in tests. `defer` is skipped if `t.FailNow()` is called; `t.Cleanup()` always runs.

```go
func TestWithDB(t *testing.T) {
	db := setupDB(t)
	t.Cleanup(func() { db.Close() }) // always runs, even after t.Fatal
}
```

### t.Parallel()

```go
for _, tt := range tests {
	tt := tt // capture range variable (required in Go < 1.22)
	t.Run(tt.name, func(t *testing.T) {
		t.Parallel() // at the top, after t.Helper() if needed
		// never share mutable state between parallel subtests
	})
}
```

---

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

---

## 4. Data Race Detection — [Mistakes] #58

Always run tests with `-race`. Races are undefined behavior — they cannot be safely ignored.

```bash
go test -race ./...
go run -race main.go
```

Enable in CI. The race detector has ~5-10x slowdown — acceptable for test suites.

> See also: `/go-reviewer` Concurrency Safety checklist for reviewing concurrent code.

---

## 5. Benchmarks

### Basic Benchmark

```go
func BenchmarkParseUser(b *testing.B) {
	data := []byte(`{"name":"alice","email":"alice@example.com"}`)
	b.ResetTimer()   // exclude setup time
	b.ReportAllocs() // show allocations/op
	for b.Loop() {   // Go 1.24+ preferred
		ParseUser(data)
	}
}

// Go 1.23 and earlier — use b.N
func BenchmarkParseUserLegacy(b *testing.B) {
	data := []byte(`{"name":"alice","email":"alice@example.com"}`)
	b.ResetTimer()
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
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
			b.SetBytes(int64(n)) // report MB/s
			b.ResetTimer()
			for b.Loop() {
				encode(data)
			}
		})
	}
}
```

### Avoid Compiler Optimization

```go
var sink any // package-level variable

func BenchmarkCompute(b *testing.B) {
	var result int
	for b.Loop() {
		result = compute(42) // compiler may optimize away if unused
	}
	sink = result // assign to package-level: prevents dead code elimination
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

---

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
		if err != nil {
			return // error is OK, panic is not
		}
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

---

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

	user, err := store.Create(ctx, "alice@example.com")
	require.NoError(t, err)
	assert.NotEmpty(t, user.ID)
}
```

`go get github.com/testcontainers/testcontainers-go`

---

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

---

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

---

## 10. Test Lifecycle

### TestMain — Global Setup and Teardown

```go
func TestMain(m *testing.M) {
	// Setup: runs once before all tests in the package
	db, cleanup := setupTestDB()

	code := m.Run() // run all tests

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
func TestFoo(t *testing.T) { store := setupStore(t); _ = store }
func BenchmarkFoo(b *testing.B) { store := setupStore(b); _ = store }
```

---

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
import _ "net/http/pprof" // registers /debug/pprof/ handlers

// curl http://localhost:6060/debug/pprof/goroutine?debug=2
```

---

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
