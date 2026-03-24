# go-coder

A production-grade Go patterns reference skill for Claude Code.

Based on three canonical books:

- *100 Go Mistakes and How to Avoid Them* (Teiva Harsanyi)
- *Learning Go, 2nd Edition* (Jon Bodner)
- *Cloud Native Go* (Matthew Titmus)

## Contents

1. **Types & Interfaces** — accept interfaces / return structs, constructor return type rules
2. **Error Handling** — `%w` vs `%v`, `errors.Is/As`, sentinel vs custom types
3. **Context** — propagation, timeout, value usage rules
4. **Concurrency** — goroutine lifecycle, errgroup vs WaitGroup, mutex vs channel, leak prevention
5. **Service Design** — structured logging (slog), dependency injection, env-based config
6. **Resilience** — retry backoff, graceful shutdown, circuit breaker
7. **Performance & Pitfalls** — slice/map, string ops, defer, escape analysis

## Installation

```bash
/plugin marketplace add ppzxc/go-plugins
/plugin install go-coder
```

## Usage

Invoke as a slash command in Claude Code:

```
/go-coder
```

Also activates automatically when writing, reviewing, or refactoring Go code.
