# Go Skill Naming Design

**Date:** 2026-03-26
**Status:** Approved
**Scope:** SKILL.md frontmatter `name` field in coder, reviewer, tester

## Problem

슬래시 명령어 검색은 SKILL.md의 `name` 필드를 기준으로 동작한다. 현재 `name: coder`, `name: reviewer`, `name: tester`로 설정되어 있어 `/coder`처럼 범용적인 이름으로만 검색된다. Go 전용 스킬임을 식별할 수 없다.

## Decision

각 SKILL.md의 frontmatter `name` 필드에 `go-` 접두사를 추가한다.

| 파일 | 변경 전 | 변경 후 |
|------|--------|--------|
| `plugins/golang/skills/coder/SKILL.md` | `name: coder` | `name: go-coder` |
| `plugins/golang/skills/reviewer/SKILL.md` | `name: reviewer` | `name: go-reviewer` |
| `plugins/golang/skills/tester/SKILL.md` | `name: tester` | `name: go-tester` |

## Result

- `/go-coder`, `/go-reviewer`, `/go-tester`로 검색 가능
- Fully-qualified name: `golang:go-coder`, `golang:go-reviewer`, `golang:go-tester`

### FQN 중복 분석

`golang:go-coder`에서 "go"가 plugin name과 skill name 양쪽에 등장한다.

**중복을 피할 수 없는 구조적 이유:** 플러그인 시스템이 `plugin_name:skill_name`으로 자동 조합하므로, 검색용 name에 언어 접두사를 넣으면 FQN에서 반드시 중복된다.

| 방안 | 검색 | FQN | 문제 |
|------|------|-----|------|
| `name: go-coder` | `/go-coder` | `golang:go-coder` | FQN 중복 |
| `name: coder` (현행) | `/coder` | `golang:coder` | 검색 모호 |
| plugin name `go`로 변경 | `/go-coder` | `go:go-coder` | 여전히 중복 + 기존 설치 깨짐 |

**결론: 검색 경험 > FQN 미관.** FQN 사용 주체는 AI(Skill 도구 내부 호출)이고 사용자가 직접 타이핑하지 않으므로, 중복을 수용하고 `/go-coder` 검색 명확성을 취한다.

## Scope

- SKILL.md frontmatter `name` 필드 변경 (3개 파일)
- SKILL.md 본문 내 cross-reference를 새 FQN으로 업데이트 (15곳)
- README 4개 파일의 스킬 테이블 및 사용 예시 업데이트
- 디렉토리 구조, plugin.json은 변경 없음

## Verification

1. 각 SKILL.md의 `name` 필드가 `go-coder`, `go-reviewer`, `go-tester`로 변경됨
2. frontmatter 외 다른 내용은 변경되지 않음
3. 마크다운 파싱이 정상 동작함
