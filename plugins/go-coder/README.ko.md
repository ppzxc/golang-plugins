# go-coder

Production-grade Go 패턴 레퍼런스 스킬입니다.

다음 세 권의 책을 기반으로 작성되었습니다:

- *100 Go Mistakes and How to Avoid Them* (Teiva Harsanyi)
- *Learning Go, 2nd Edition* (Jon Bodner)
- *Cloud Native Go* (Matthew Titmus)

## 포함 내용

1. **Types & Interfaces** — accept interfaces / return structs, 생성자 반환 타입 규칙
2. **Error Handling** — `%w` vs `%v`, `errors.Is/As`, sentinel/custom types
3. **Context** — 전달 방법, timeout, value 사용 규칙
4. **Concurrency** — goroutine lifecycle, errgroup vs WaitGroup, mutex vs channel, leak 방지
5. **Service Design** — 구조화된 로깅(slog), 의존성 주입, 환경변수 기반 설정
6. **Resilience** — retry backoff, graceful shutdown, circuit breaker
7. **Performance & Pitfalls** — slice/map, 문자열 연산, defer, escape analysis

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
