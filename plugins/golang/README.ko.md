# golang

Claude Code용 Production-grade Go 패턴 플러그인입니다. Go 코드 작성, 리뷰, 테스트를 위한 세 가지 스킬을 제공합니다.

다음 Go 정규 소스를 기반으로 작성되었습니다:

- *100 Go Mistakes and How to Avoid Them* (Teiva Harsanyi)
- *The Go Programming Language* (Donovan & Kernighan)
- *Concurrency in Go* (Cox-Buday)
- *Learning Go, 2nd Edition* (Jon Bodner)
- *Cloud Native Go* (Matthew Titmus)
- *Let's Go / Let's Go Further* (Alex Edwards)
- *Effective Go* 및 *Go Code Review Comments* (Go Team)

## 스킬

| 스킬 | 커맨드 | 설명 |
|------|--------|------|
| coder | `/golang:coder` | 타입, 인터페이스, 에러 처리, 동시성, 서비스 설계, 복원력 |
| reviewer | `/golang:reviewer` | 에러 처리, 동시성 안전, 네이밍, API 설계, 성능, 보안 |
| tester | `/golang:tester` | table-driven 테스트, 벤치마크, fuzzing, mock, 통합 테스트, 커버리지 |

## 설치

```bash
/plugin marketplace add ppzxc/golang-plugins
/plugin install golang
```

## 사용 방법

스킬을 완전한 슬래시 커맨드로 호출합니다:

```
/golang:coder    # Go 프로덕션 코드 작성 또는 리팩토링 시
/golang:reviewer # Go 코드 PR 리뷰 시
/golang:tester   # Go 테스트 작성 또는 개선 시
```

Go 코드 작업 시 컨텍스트에 따라 스킬이 자동으로 활성화되기도 합니다.

## 저자

**ppzxc** — [ppzxc.github.io](https://ppzxc.github.io)
