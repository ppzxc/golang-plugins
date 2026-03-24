# go-reviewer

Go 코드 리뷰 체크리스트 스킬입니다.

## 포함 내용

1. **Go 공식 스타일** — 네이밍, 에러 문자열, doc comments, import 그룹핑
2. **Error Handling** — wrapping, sentinel 에러, 에러 무시
3. **Context** — 전달, 취소, value 사용
4. **동시성 안전** — goroutine lifecycle, mutex 사용, race condition
5. **인터페이스 & API 설계** — 노출 범위, forward-compatibility
6. **서비스 구조** — 프로젝트 레이아웃, 패키지 네이밍, 의존성 방향
7. **복원력** — retry, circuit breaker, graceful shutdown
8. **성능** — slice/map 함정, string 빌딩, 메모리 릭
9. **보안** — injection, 입력 검증, TLS

## 설치

```bash
/plugin marketplace add ppzxc/go-plugins
/plugin install go-reviewer
```

## 사용 방법

설치 후 Claude Code에서 슬래시 커맨드로 스킬을 호출합니다:

```
/go-reviewer
```

Go 코드 리뷰 또는 PR 리뷰 시 자동으로 활성화됩니다.
