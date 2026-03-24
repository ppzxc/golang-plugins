# go-coder

A production-grade Go patterns reference skill for Claude Code.

Based on three canonical books:

- *100 Go Mistakes and How to Avoid Them* (Teiva Harsanyi)
- *Learning Go, 2nd Edition* (Jon Bodner)
- *Cloud Native Go* (Matthew Titmus)

## Contents

1. **Error Handling** — `%w` vs `%v`, `errors.Is/As`, sentinel vs custom types
2. **Context** — propagation, timeout, value usage rules
3. **Generics** — type constraints, functions vs methods
4. **Concurrency** — goroutine lifecycle, errgroup vs WaitGroup, mutex vs channel, leak prevention
5. **Interface Design** — accept interfaces / return structs, constructor return type rules
6. **Slice & Map** — nil vs empty, pre-allocation, backing array aliasing
7. **Constructor & Config** — functional options, config struct
8. **Testing** — table-driven, t.Helper/Cleanup/Parallel, manual mock
9. **Resilience** — retry backoff, graceful shutdown, circuit breaker
10. **Project Structure** — cmd/internal/pkg, package naming

## Installation

```bash
/plugin marketplace add ppzxc/go-skills
/plugin install go-coder
```

## Usage

Invoke as a slash command in Claude Code:

```
/go-coder
```

Also activates automatically when writing, reviewing, or refactoring Go code.
