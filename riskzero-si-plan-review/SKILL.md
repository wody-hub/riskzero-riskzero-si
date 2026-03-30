---
name: riskzero-si-plan-review
version: 1.0.0
description: |
  구현 계획서 아키텍처/보안 리뷰. gstack plan-eng-review를 래핑한다.
  구현 계획서의 아키텍처, 데이터 흐름, 엣지 케이스, 보안 설계를 점검하고
  프로젝트 README.md 표준 준수 여부도 함께 확인한다.
  /riskzero-si-plan-review
  Use when asked to "계획 리뷰", "plan review", "아키텍처 리뷰", "설계 리뷰".
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
1. `.claude/si-config.yml`
2. `si-config.yml`
3. `.claude/qa-config.yml`

### 2. 계획서 확인

`plan/` 디렉토리에서 가장 최근 구현 계획서를 찾는다.
- `plan/{기능명}/implementation-plan.md`

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

불일치 항목이 있으면 목록으로 정리한다.

### 4. gstack plan-eng-review 실행

Skill 도구를 사용하여 `/plan-eng-review`를 호출한다.

gstack이 아키텍처, 데이터 흐름, 엣지 케이스, 테스트 커버리지, 성능 관점에서 계획서를 리뷰한다.

### 5. 통합 리뷰 결과 정리

README.md 표준 체크 결과와 gstack 리뷰 결과를 통합하여 `plan/{기능명}/plan-review.md`에 저장한다.

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

## 종합 판정
- PASS: 진행 가능
- WARN: 권고사항 있으나 진행 가능
- FAIL: 계획 수정 필요 → 1단계로 복귀
```

### 6. 후속 조치

- **PASS/WARN**: 다음 단계(Step 3: 구현)로 진행
- **FAIL**: CRITICAL 이슈 목록을 제시하고, `/riskzero-si-plan`으로 계획을 수정하도록 안내
