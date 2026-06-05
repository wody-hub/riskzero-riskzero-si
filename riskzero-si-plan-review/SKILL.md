---
name: riskzero-si-plan-review
version: 1.0.0
description: Use when reviewing a riskzero-si implementation plan before coding; triggers include "계획 리뷰", "plan review", "아키텍처 리뷰", "설계 리뷰". riskzero-si 계획서가 없는 일반 설계/아키텍처 리뷰에는 사용하지 않는다.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - AskUserQuestion
  - Task
  - Skill
---

# 구현 계획서 리뷰 (Step 2)

## 역할

1단계(`/riskzero-si-plan`)에서 생성된 구현 계획서를 리뷰한다.
gstack의 `/plan-eng-review` 스킬을 활용하여 아키텍처, 데이터 흐름, 엣지 케이스, 보안 설계를 점검하며, 추가로 프로젝트 README.md의 표준 준수 여부도 확인한다.

---

## 실행 절차

### 1. 설정 파일 로드

아래 순서로 설정 파일을 탐색하여 로드한다:
1. `.codex/si-config.yml`
2. `.claude/si-config.yml`
3. `si-config.yml`
4. `.codex/qa-config.yml`
5. `.claude/qa-config.yml`

### 2. 계획서 확인

`.si-planning/` 디렉토리에서 가장 최근 구현 계획서를 찾는다.
- `.si-planning/{기능명}/implementation-plan.md`
- `.si-planning/{기능명}/discussion.md`
- `.si-planning/{기능명}/research.md` (`implementation-plan.md`가 performed/use-existing 상태일 때 읽음. skipped/ignored-existing이면 스킵/무시 사유 확인)
- `.si-planning/{기능명}/tdd-plan.md`

계획서가 없으면:
> 구현 계획서가 없습니다. `/riskzero-si-plan {기능명}`으로 먼저 계획을 수립하세요.

### 3. README.md 표준 사전 체크

gstack 리뷰 실행 **전에**, 프로젝트 README.md(`config.project.readme`)를 읽고 아래 항목이 계획서에 반영되었는지 확인한다:

- **패키지 구조**: `config.backend.structure.*` 패턴과 계획서의 파일 배치가 일치하는가
- **네이밍 규칙**: DTO/VO/Controller/Service 명명법이 README 규칙을 따르는가
- **API 설계**: REST API URI 패턴, HTTP 메서드, 응답 형식이 프로젝트 표준인가
- **보안**: 권한 체크 어노테이션이 포함되었는가
- **검증**: 3-layer 검증 (DTO → Validator → Service) 설계가 포함되었는가
- **트랜잭션**: Service 레이어에만 @Transactional이 설계되었는가
- **설계 전 논의**: `discussion.md`의 결정사항이 구현 계획에 반영되었는가
- **계획 전 리서치**: `implementation-plan.md`의 리서치 상태가 명확하고, `research.md` 권장사항 또는 스킵/무시 사유가 구현 계획에 반영되었는가
- **TDD 계획**: `tdd-plan.md`의 RED 테스트 케이스가 주요 BE/FE 동작과 연결되는가

불일치 항목이 있으면 목록으로 정리한다.

### 4. gstack plan-eng-review 실행

Skill 도구를 사용하여 `/plan-eng-review`를 호출한다.

gstack이 아키텍처, 데이터 흐름, 엣지 케이스, 테스트 커버리지, 성능 관점에서 계획서를 리뷰한다.

추가로 아래 항목을 반드시 확인한다:
- `discussion.md`가 존재하고 gray area 결정사항이 `implementation-plan.md`에 반영되었는가?
- 리서치를 수행했다면 `research.md`가 존재하고 권장 접근/위험/테스트 관점이 계획에 반영되었는가?
- 외부 리서치를 수행했다면 URL, 확인일, 출처 유형, 적용 판단이 `research.md`에 기록되었는가?
- 리서치를 스킵했다면 `implementation-plan.md`에 스킵 사유가 명확히 기록되었는가?
- 기존 `research.md`를 무시했다면 `implementation-plan.md`에 `Status: ignored-existing`과 무시 사유가 기록되었는가?
- `tdd-plan.md`가 존재하고 구현 전에 실패해야 할 테스트 케이스가 구체적인가?
- `backend.testCmd` / `frontend.testCmd`가 없을 때 대체 테스트 명령 탐색 전략이 있는가?
- 테스트가 구현 후 회귀 검증이 아니라 구현 전 RED 확인에 사용할 수 있게 설계되었는가?

### 5. 통합 리뷰 결과 정리

README.md 표준 체크 결과와 gstack 리뷰 결과를 통합하여 `.si-planning/{기능명}/plan-review.md`에 저장한다.

```markdown
# 계획 리뷰 결과: {기능명}

## README.md 표준 준수 체크
| # | 항목 | 상태 | 비고 |
|---|------|------|------|
| 1 | 패키지 구조 | PASS/FAIL | ... |
| 2 | 네이밍 규칙 | PASS/FAIL | ... |
| ... |

## gstack 리뷰 결과
(gstack /plan-eng-review 결과 요약)

## 설계 전 논의 반영
- discussion.md 존재: Y/N
- 구현 계획 반영: PASS/FAIL

## 계획 전 리서치 반영
- research status: performed / use-existing / skipped / ignored-existing
- external research status: performed / skipped / unavailable / not-needed
- research.md 존재: Y/N
- external citations: PASS/FAIL/N/A
- 리서치 수행/스킵 판단 근거: PASS/FAIL
- 권장 접근/위험/테스트 관점 반영: PASS/FAIL

## TDD 계획 검토
- tdd-plan.md 존재: Y/N
- RED 테스트 구체성: PASS/FAIL
- 테스트 명령 정의: PASS/FAIL
- 주요 동작 커버: PASS/FAIL

## 종합 판정
- PASS: 진행 가능
- WARN: 권고사항 있으나 진행 가능
- FAIL: 계획 수정 필요 → 1단계로 복귀
```

### 6. 후속 조치

- **PASS/WARN**: 다음 단계(Step 3: 구현)로 진행
- **FAIL**: CRITICAL 이슈 목록을 제시하고, `/riskzero-si-plan`으로 계획을 수정하도록 안내
