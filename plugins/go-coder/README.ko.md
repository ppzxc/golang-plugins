# go-coder

Production-grade Go 패턴 레퍼런스 스킬입니다.

다음 세 권의 책을 기반으로 작성되었습니다:

- *100 Go Mistakes and How to Avoid Them* (Teiva Harsanyi)
- *Learning Go, 2nd Edition* (Jon Bodner)
- *Cloud Native Go* (Matthew Titmus)

## 포함 내용

1. **Error Handling** — `%w` vs `%v`, `errors.Is/As`, sentinel/custom types
2. **Context** — 전달 방법, timeout, value 사용 규칙
3. **Generics** — 타입 제약, 함수 vs 메서드
4. **Concurrency** — goroutine lifecycle, errgroup vs WaitGroup, mutex vs channel, leak 방지
5. **Interface Design** — accept interfaces / return structs, 생성자 반환 타입 규칙
6. **Slice & Map** — nil vs empty, 사전 할당, backing array aliasing
7. **Constructor & Config** — functional options, config struct
8. **Resilience** — retry backoff, graceful shutdown, circuit breaker
9. **Project Structure** — cmd/internal/pkg, 패키지 네이밍
10. **구조화된 로깅** — slog 설정, 핸들러, 그룹 (Go 1.21+)
11. **의존성 주입** — cmd/에서 constructor wiring
12. **설정 로딩** — 환경변수 기반 설정, 시작 시 검증

## 설치

```bash
/plugin marketplace add ppzxc/go-plugins
/plugin install go-coder
```

## 사용 방법

설치 후 Claude Code에서 슬래시 커맨드로 스킬을 호출합니다:

```
/go-coder
```

Go 코드 작성, 리뷰, 리팩토링 시 자동으로 활성화되기도 합니다.
