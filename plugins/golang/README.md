# golang

A production-grade Go patterns plugin for Claude Code, providing three skills for writing, reviewing, and testing Go code.

Based on canonical Go sources:

- *100 Go Mistakes and How to Avoid Them* (Teiva Harsanyi)
- *The Go Programming Language* (Donovan & Kernighan)
- *Concurrency in Go* (Cox-Buday)
- *Learning Go, 2nd Edition* (Jon Bodner)
- *Cloud Native Go* (Matthew Titmus)
- *Let's Go / Let's Go Further* (Alex Edwards)
- *Effective Go* and *Go Code Review Comments* (Go Team)

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| go-coder | `/golang:go-coder` | Types, interfaces, error handling, concurrency, service design, resilience |
| go-reviewer | `/golang:go-reviewer` | Error handling, concurrency safety, naming, API design, performance, security |
| go-tester | `/golang:go-tester` | Table-driven tests, benchmarks, fuzzing, mocks, integration tests, coverage |

## Installation

```bash
/plugin marketplace add ppzxc/golang-plugins
/plugin install golang
```

## Usage

Invoke a skill by its fully-qualified slash command:

```
/golang:go-coder    # writing or refactoring Go production code
/golang:go-reviewer # reviewing Go code in pull requests
/golang:go-tester   # writing or improving Go tests
```

Skills also activate automatically based on context when working with Go code.

## Author

**ppzxc** — [ppzxc.github.io](https://ppzxc.github.io)
