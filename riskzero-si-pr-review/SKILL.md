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

gstack 리뷰 결과를 `.si-planning/{기능명}/pr-review.md`에 저장한다.

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

### 4. 진행 게이트 (Progression Gate)

다음 단계 진행 여부는 **gstack 리뷰 지적과 TDD 체크 결과의 심각도로 결정**한다(판정 라벨이 아니라 발견사항 기준).

- **차단 = 수정이 필요한 지적(Medium 이상 / 안전성·보안·성능·에러핸들링 결함, BLOCKER) 또는 TDD 체크의 FAIL.**
  - `testing.evidenceRequired: true`인데 RED/GREEN 증거가 없으면 차단(Step 3 복귀)으로 취급한다.
- **차단 0건(잔여는 Low/권고만)일 때만** 다음 단계(Step 6: QA 체크리스트)로 진행한다.
- **1건이라도 있으면**: QA로 넘어가지 말고 → 코드를 수정 → `/riskzero-si-pr-review {기능명}`로 **재리뷰** → 차단 0건이 될 때까지 반복한 뒤 진행한다.
- **Low/권고**는 비차단(pr-review.md에 기록).
- **예외(명시적 수용)**: 사용자가 특정 차단 항목을 명시적으로 수용/보류한 경우, pr-review.md에 사유·후속 추적 위치를 기록한 항목에 한해 차단에서 제외할 수 있다.

---

## 단계 종료 시 필수 출력 (단계 종료 푸터)

이 스킬은 riskzero-si 8단계 파이프라인의 **Step 5**이다. 리뷰 산출물 저장 후 **§4 진행 게이트로 차단 발견사항(Medium 이상/BLOCKER/TDD FAIL) 건수를 집계**하고, 그 결과에 따라 아래 두 푸터 중 **하나만** 답변 맨 끝에 출력한다. (`{기능명}`은 실제 값으로 채운다.)

**(A) 차단 발견사항 0건 — 통과 시:**

> ---
> ### 📍 파이프라인 진행 현황 — `{기능명}`
>
> | # | 단계 | 명령 | 상태 |
> |---|------|------|------|
> | 1~4 | 계획~표준리뷰 | … | ✅ |
> | 5 | PR 리뷰 | `riskzero-si-pr-review` | 👉 방금 완료 (통과) |
> | 6 | QA 체크리스트 | `riskzero-si-qa-checklist` | 👉 다음 |
> | 7 | 버그 수정 | `riskzero-si-qa` | ⬜ |
> | 8 | 최종 검증 | `riskzero-si-browse` | ⬜ |
>
> **방금 완료:** Step 5 — 산출물: `.si-planning/{기능명}/pr-review.md` (차단 0건, 잔여 Low/권고만)
>
> **▶️ 다음 단계 — 아래 명령어를 복사해 실행하세요:**
> ```
> /riskzero-si-qa-checklist {메뉴명 또는 URL}
> ```
> *(⚠️ Step 6는 기능명이 아니라 검증 대상 **메뉴명 / URL / 화면ID**를 인자로 받습니다.)*
>
> 💡 컨텍스트가 길어졌다면 `/clear` 후 위 명령어를 그대로 붙여넣으세요. 다음 단계는 구현된 화면/라우트를 분석해 단독 실행됩니다.
> ---

**(B) 차단 발견사항 ≥1건 — 조치 필요 시 (QA로 진행하지 않는다):**

> ---
> ### 📍 파이프라인 진행 현황 — `{기능명}`
>
> | # | 단계 | 명령 | 상태 |
> |---|------|------|------|
> | 1~4 | 계획~표준리뷰 | … | ✅ |
> | 5 | PR 리뷰 | `riskzero-si-pr-review` | 🔴 조치 필요 (차단 {N}건) |
> | 6 | QA 체크리스트 | `riskzero-si-qa-checklist` | ⛔ 대기 |
>
> **방금 완료:** Step 5 — 산출물: `.si-planning/{기능명}/pr-review.md` (차단 {N}건)
>
> **⚠️ QA 단계로 진행하기 전에 차단 항목을 먼저 조치해야 합니다.** 코드를 수정한 뒤 아래 명령으로 **재리뷰**하세요. 차단 0건이 될 때까지 반복합니다. (TDD 증거 부재가 원인이면 `/riskzero-si-impl {기능명}`으로 Step 3 복귀.)
> ```
> /riskzero-si-pr-review {기능명}
> ```
> *(특정 차단 항목을 의도적으로 수용/보류한다면, pr-review.md에 사유·후속 추적 위치를 기록한 항목에 한해 차단에서 제외할 수 있습니다.)*
>
> 💡 컨텍스트가 길어졌다면 `/clear` 후 위 명령어를 그대로 붙여넣으세요.
> ---
> ---
