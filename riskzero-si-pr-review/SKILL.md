---
name: riskzero-si-pr-review
version: 1.0.0
description: |
  PR diff 기반 코드 안전성 리뷰. gstack review를 래핑한다.
  SQL 안전성, 신뢰 경계 위반, 조건부 사이드이펙트 등 구조적 이슈를 점검한다.
  /riskzero-si-pr-review
  Use when asked to "PR 리뷰", "pr review", "diff 리뷰", "코드 안전성 리뷰".
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

# PR Diff 리뷰 (Step 5)

## 역할

4단계(`/riskzero-si-review`)에서 프로젝트 표준 리뷰를 통과한 코드를, git diff 기반으로 **안전성** 관점에서 리뷰한다.
gstack의 `/review` 스킬을 활용하여 SQL 안전성, 신뢰 경계 위반, 조건부 사이드이펙트 등 구조적 이슈를 점검한다.

**`/riskzero-si-review`(Step 4)와의 역할 분담:**
- Step 4 (`/riskzero-si-review`): "우리 규칙을 따르는가?" — README.md 기반, 프로젝트 컨벤션 점검
- Step 5 (`/riskzero-si-pr-review`): "코드가 안전한가?" — PR diff 기반, 범용 안전성 점검

---

## 실행 절차

### 1. 변경 사항 확인

현재 브랜치의 변경된 파일을 확인한다:
```bash
git diff --name-only HEAD~1
# 또는 base 브랜치 대비
git diff --name-only main...HEAD
```

변경 사항이 없으면:
> 리뷰할 변경 사항이 없습니다. 코드 구현 후 다시 실행하세요.

### 2. gstack review 실행

Skill 도구를 사용하여 `/review`를 호출한다.

gstack이 아래 관점에서 diff를 리뷰한다:
- SQL 안전성 (SQL Injection 등)
- LLM 신뢰 경계 위반
- 조건부 사이드이펙트
- 보안 취약점 (OWASP Top 10)
- 성능 이슈 (N+1 쿼리, 불필요한 재렌더링 등)
- 에러 핸들링 누락

### 3. 리뷰 결과 저장

gstack 리뷰 결과를 `plan/{기능명}/pr-review.md`에 저장한다.

### 4. 후속 조치

- **PASS**: 다음 단계(Step 6: QA 체크리스트)로 진행
- **BLOCKER 발견**: 코드를 수정한 후 재리뷰
- 수정 후 `/riskzero-si-pr-review`를 다시 실행하여 BLOCKER가 해소되었는지 확인
