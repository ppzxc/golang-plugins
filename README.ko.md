# golang-plugins

Claude Code 플러그인 마켓플레이스입니다. Production-grade Go 개발에 필요한 패턴과 베스트 프랙티스를 스킬로 제공합니다.

## 포함된 플러그인

| 플러그인 | 버전 | 설명 |
|---------|------|------|
| [golang](./plugins/golang) | 0.0.1 | Production-grade Go 패턴 — 코딩, 리뷰, 테스트 스킬 |

## 설치

### 1. 마켓플레이스 등록

```bash
/plugin marketplace add ppzxc/golang-plugins
```

### 2. 플러그인 설치

```bash
/plugin install golang
```

## 사용 방법

```
/golang:coder    # Go 프로덕션 코드 작성 또는 리팩토링 시
/golang:reviewer # Go 코드 PR 리뷰 시
/golang:tester   # Go 테스트 작성 또는 개선 시
```

## 프로젝트 구조

```
golang-plugins/
├── .claude-plugin/
│   └── marketplace.json        # 마켓플레이스 메타데이터
└── plugins/
    └── golang/                 # Go 스킬 플러그인
        ├── .claude-plugin/plugin.json
        ├── README.md
        └── skills/
            ├── coder/SKILL.md
            ├── reviewer/SKILL.md
            └── tester/SKILL.md
```

## 저자

**ppzxc** — [ppzxc.github.io](https://ppzxc.github.io)
