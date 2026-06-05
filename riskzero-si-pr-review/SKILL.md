---
name: riskzero-si-pr-review
version: 1.0.0
description: Use when reviewing git diff or PR changes for safety after riskzero-si implementation; triggers include "PR 리뷰", "pr review", "diff 리뷰", "코드 안전성 리뷰". riskzero-si 구현과 무관한 일반 PR 리뷰에는 사용하지 않는다 (gstack /review 사용).
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

4단계(`/riskzero-si-review`)에서 프로젝트 표준 + 리서치/TDD 증거 리뷰를 통과한 코드를, git diff 기반으로 **안전성** 관점에서 리뷰한다.
gstack의 `/review` 스킬을 활용하여 SQL 안전성, 신뢰 경계 위반, 조건부 사이드이펙트 등 구조적 이슈를 점검한다.

**`/riskzero-si-review`(Step 4)와의 역할 분담:**
- Step 4 (`/riskzero-si-review`): "우리 규칙을 따르고 리서치/TDD 증거가 맞는가?" — README.md 기반, 프로젝트 컨벤션과 RED/GREEN 증거 점검
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
- 테스트 누락 및 TDD 증거 부재
- `implementation-plan.md`가 리서치 수행/사용 상태인 경우 `research.md` 권장 접근/위험 경고와 diff의 충돌 여부
- 외부 리서치를 수행한 경우 출처 URL/확인일/적용 판단이 있고, diff가 외부 권고와 충돌하지 않는지 여부

### 3. 리뷰 결과 저장

gstack 리뷰 결과를 `plan/{기능명}/pr-review.md`에 저장한다.

리뷰 결과에는 아래 TDD 체크를 별도 섹션으로 포함한다:

```markdown
## TDD / 자동화 테스트 증거
- tdd-plan.md 존재: Y/N
- tdd-report.md 존재: Y/N
- RED 실패 확인: PASS/FAIL/N/A
- GREEN 성공 확인: PASS/FAIL/N/A
- diff의 주요 동작과 테스트 연결: PASS/FAIL
```

`implementation-plan.md`의 리서치 상태가 `skipped` 또는 `ignored-existing`이면 아래 섹션을 N/A로 기록하고 스킵/무시 사유를 적는다.
그 외에 `research.md`가 있으면 아래 항목도 함께 기록한다:

```markdown
## 계획 전 리서치 반영
- research.md 존재: Y/N
- external citations: PASS/FAIL/N/A
- 권장 접근과 diff 일치: PASS/FAIL/N/A
- 리서치에서 경고한 위험 대응: PASS/FAIL/N/A
```

### 4. 후속 조치

- **PASS**: 다음 단계(Step 6: QA 체크리스트)로 진행
- **BLOCKER 발견**: 코드를 수정한 후 재리뷰
- **TDD 증거 부재**: `testing.evidenceRequired: true`이면 BLOCKER로 취급하고 Step 3으로 복귀
- 수정 후 `/riskzero-si-pr-review`를 다시 실행하여 BLOCKER가 해소되었는지 확인
