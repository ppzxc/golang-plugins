# golang-plugins

A Claude Code plugin marketplace providing production-grade Go patterns and best practices as skills.

## Plugins

| Plugin | Version | Description |
|--------|---------|-------------|
| [golang](./plugins/golang) | 0.0.1 | Production-grade Go patterns — coding, reviewing, and testing skills |

## Installation

### 1. Add marketplace

```bash
/plugin marketplace add ppzxc/golang-plugins
```

### 2. Install plugin

```bash
/plugin install golang
```

## Usage

```
/golang:coder    # writing or refactoring Go production code
/golang:reviewer # reviewing Go code in pull requests
/golang:tester   # writing or improving Go tests
```

## Project Structure

```
golang-plugins/
├── .claude-plugin/
│   └── marketplace.json        # Marketplace metadata
└── plugins/
    └── golang/                 # Go skills plugin
        ├── .claude-plugin/plugin.json
        ├── README.md
        └── skills/
            ├── coder/SKILL.md
            ├── reviewer/SKILL.md
            └── tester/SKILL.md
```

## Author

**ppzxc** — [ppzxc.github.io](https://ppzxc.github.io)
