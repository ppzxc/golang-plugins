# Go Skill Naming Design

**Date:** 2026-03-26
**Status:** Superseded (v2 decision below)
**Scope:** SKILL.md frontmatter `name` field in coder, reviewer, tester

---

## v2 Decision (Current)

### Problem (Updated)

[anthropics/claude-code#17271](https://github.com/anthropics/claude-code/issues/17271) 분석 결과, `name` 필드 유무에 따라 슬래시 커맨드 등록 방식이 달라짐:
- **`name` 필드 있음** → plugin prefix가 제거되고 `name` 값만 슬래시 커맨드로 등록 → `/go-coder`
- **`name` 필드 없음** → 디렉토리명 + plugin prefix 유지 → `/golang:coder`

`name: go-coder`로 설정하면 의도와 달리 `/go-coder`로 등록되고 plugin prefix가 제거된다. 원하는 형태는 `/golang:coder`.

### Decision

각 SKILL.md의 frontmatter에서 `name` 필드를 **제거**한다. 디렉토리명(`coder`, `reviewer`, `tester`)이 plugin prefix와 결합되어 `/golang:coder` 형태로 슬래시 커맨드에 등록된다.

| 파일 | 변경 전 | 변경 후 |
|------|--------|--------|
| `plugins/golang/skills/coder/SKILL.md` | `name: go-coder` | (name 필드 없음) |
| `plugins/golang/skills/reviewer/SKILL.md` | `name: go-reviewer` | (name 필드 없음) |
| `plugins/golang/skills/tester/SKILL.md` | `name: go-tester` | (name 필드 없음) |

### Result

- `/golang:coder`, `/golang:reviewer`, `/golang:tester`로 호출 가능
- Fully-qualified name: `golang:coder`, `golang:reviewer`, `golang:tester`
- Plugin namespace(`golang:`)가 명시되어 다른 플러그인 스킬과 충돌 없음

---

## v1 Decision (Superseded)

### Problem

슬래시 명령어 검색은 SKILL.md의 `name` 필드를 기준으로 동작한다. 현재 `name: coder`, `name: reviewer`, `name: tester`로 설정되어 있어 `/coder`처럼 범용적인 이름으로만 검색된다. Go 전용 스킬임을 식별할 수 없다.

### Decision

각 SKILL.md의 frontmatter `name` 필드에 `go-` 접두사를 추가한다.

**폐기 이유:** `name` 필드가 있으면 plugin prefix가 제거되어 `/go-coder`로 등록되며, 원하는 `/golang:coder` 형태가 되지 않음. Issue #17271에서 확인된 동작.
