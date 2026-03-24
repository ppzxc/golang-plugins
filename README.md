# go-skills

Claude Code 플러그인 마켓플레이스입니다. Production-grade Go 개발에 필요한 패턴과 베스트 프랙티스를 스킬로 제공합니다.

## 포함된 플러그인

| 플러그인 | 버전 | 설명 |
|---------|------|------|
| [go-coder](./plugins/go-coder) | 0.0.1 | Production-grade Go 패턴 — 에러 처리, 동시성, 인터페이스, 테스트, 복원력 |

## 설치

### 1. 마켓플레이스 등록

```bash
/plugin marketplace add ppzxc/go-skills
```

### 2. 플러그인 설치

```bash
/plugin install go-coder
```

## 프로젝트 구조

```
go-skills/
├── .claude-plugin/
│   └── marketplace.json        # 마켓플레이스 메타데이터
└── plugins/
    └── go-coder/        # Modern Go 패턴 플러그인
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            └── go-coder/
                └── SKILL.md
```

## 저자

**ppzxc** — [ppzxc.github.io](https://ppzxc.github.io)
