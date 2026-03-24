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

Invoke as a slash command in Claude Code:

```
/go-reviewer
```

Also activates automatically when reviewing Go code or pull requests.
