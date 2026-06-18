---
name: riskzero-si-browse
version: 1.0.0
description: Use when running final browser verification for a riskzero-si feature, QA checklist, screenshots, or final report; triggers include "브라우저 테스트", "최종 검증", "browse", "화면 테스트". 일반 브라우저 테스트/QA에는 사용하지 않는다 (gstack /browse, /qa-only 사용).
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

# 브라우저 최종 검증 (Step 8)

## 역할

7단계(`/riskzero-si-qa`)에서 버그를 수정한 코드를, 실제 브라우저에서 최종 검증한다.
gstack의 `/browse`(헤드리스 브라우저)와 `/qa-only`(리포트 전용 테스트)를 활용하여 QA 체크리스트 전 항목을 검증하고, 스크린샷 증적과 함께 최종 리포트를 생성한다.

---

## 실행 절차

### 1. 설정 파일 로드

아래 순서로 설정 파일을 탐색하여 로드한다:
1. `.codex/si-config.yml`
2. `.claude/si-config.yml`
3. `si-config.yml`
4. `.codex/qa-config.yml`
5. `.claude/qa-config.yml`

### 2. 서버 상태 확인

프론트엔드/백엔드 서버가 구동 중인지 확인한다.
서버가 응답하지 않으면 사용자에게 서버 구동을 요청한다.

### 3. 테스트 모드 선택

사용자에게 테스트 모드를 확인한다:

1. **리포트 전용** (`/qa-only` 사용): 버그를 발견하고 리포트만 생성. 코드 수정 없음.
2. **브라우저 탐색** (`/browse` 사용): 특정 URL을 브라우저에서 열고 직접 탐색/검증.

### 4-A. 리포트 전용 모드

Skill 도구를 사용하여 `/qa-only`를 호출한다.

gstack qa-only가 수행하는 작업:
- QA 체크리스트 항목을 순서대로 브라우저에서 검증
- 각 항목별 PASS/FAIL 판정
- 스크린샷 캡처 (`.si-planning/{기능명}/evidence/screenshots/`)
- 재현 절차(repro steps) 기록
- health score 산출
- 최종 리포트 생성

### 4-B. 브라우저 탐색 모드

Skill 도구를 사용하여 `/browse {URL}`을 호출한다.

gstack browse가 제공하는 기능:
- 페이지 네비게이션 및 상호작용
- 폼 입력, 버튼 클릭, 파일 업로드
- 스크린샷 캡처
- before/after 비교
- 반응형 레이아웃 테스트
- 요소 상태 검증 (visible, editable, checked 등)

### 5. 최종 리포트 저장

테스트 결과를 `.si-planning/{기능명}/final-report.md`에 저장한다.
스크린샷과 로그는 `.si-planning/{기능명}/evidence/` 하위 디렉토리에 저장한다.

```markdown
# 최종 검증 리포트: {기능명}

## 실행 요약
- 실행일시: {YYYY-MM-DD HH:mm:ss}
- 대상 URL: {config.server.frontend.baseUrl}/{path}
- 총 테스트: N개
- PASS: N개 (N%)
- FAIL: N개 (N%)

## FAIL 항목 (즉시 확인 필요)
| ID | 테스트 케이스 | 기대 결과 | 실제 결과 | 스크린샷 |
|----|-------------|----------|----------|----------|

## 카테고리별 결과
(A~F 카테고리 상세)

## 스크린샷 목록
| 파일명 | 테스트 ID | 설명 |
|--------|----------|------|
```

### 6. 후속 조치

- **전 항목 PASS**: 파이프라인 완료. 축하 메시지 출력.
- **FAIL 항목 존재**: CRITICAL 이슈는 Step 7(`/riskzero-si-qa`)로 복귀 권고.
- 사소한 이슈는 사용자에게 판단을 위임.

---

## 단계 종료 시 필수 출력 (단계 종료 푸터)

이 스킬은 riskzero-si 8단계 파이프라인의 **Step 8 (마지막)**이다. 최종 보고서 저장 후 답변 맨 끝에 아래 푸터를 반드시 출력한다. (`{기능명}`은 실제 값으로 채운다.)

> ---
> ### 📍 파이프라인 진행 현황 — `{기능명}`
>
> | # | 단계 | 명령 | 상태 |
> |---|------|------|------|
> | 1~7 | 계획~버그수정 | … | ✅ |
> | 8 | 최종 검증 | `riskzero-si-browse` | 👉 방금 완료 |
>
> **방금 완료:** Step 8 — 산출물: `.si-planning/{기능명}/final-report.md` + `evidence/screenshots/`
>
> **✅ 전 항목 PASS 시:** 🎉 **파이프라인 완료** — 더 진행할 단계가 없습니다. 최종 보고서를 확인하세요.
>
> **❗ CRITICAL 이슈 발견 시 — 아래 명령어로 되돌아가세요:**
> ```
> /riskzero-si-qa {기능명}
> ```
>
> 💡 `/clear` 후 재개하더라도 `.si-planning/{기능명}/qa-checklist.md`와 `final-report.md`를 읽어 이어갈 수 있습니다.
> ---
