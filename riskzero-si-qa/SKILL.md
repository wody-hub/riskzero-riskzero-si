---
name: riskzero-si-qa
version: 1.0.0
description: |
  버그 조사 및 수정. gstack investigate + qa를 래핑한다.
  QA 체크리스트 기반으로 버그를 조사하고, 원인을 분석하여 수정한 뒤 재검증한다.
  /riskzero-si-qa [URL]
  Use when asked to "버그 수정", "qa", "QA", "버그 조사", "테스트 수정".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
  - Task
  - Skill
---

# 버그 조사 및 수정 (Step 7)

## 역할

6단계(`/riskzero-si-qa-checklist`)에서 생성된 QA 체크리스트를 기반으로 버그를 조사하고 수정한다.
gstack의 `/investigate`(원인 분석)와 `/qa`(수정 + 재검증)를 순차 활용한다.

**핵심 원칙:** 원인 파악 없이 코드를 수정하지 않는다 (Iron Law: no fixes without root cause).

---

## 실행 절차

### 1. 설정 파일 로드

아래 순서로 설정 파일을 탐색하여 로드한다:
1. `.claude/si-config.yml`
2. `si-config.yml`
3. `.claude/qa-config.yml`

### 2. QA 체크리스트 확인

아래 위치에서 QA 체크리스트를 찾는다:
- `plan/{기능명}/qa-checklist.md`
- `/tmp/qa-checklist-*.md`
- `/tmp/qa-report-*.md` (이전 테스트 리포트)

체크리스트가 없으면:
> QA 체크리스트가 없습니다. `/riskzero-si-qa-checklist {기능명}`으로 먼저 생성하세요.

### 3. 서버 상태 확인

프론트엔드/백엔드 서버가 구동 중인지 확인한다:
```bash
curl -s -o /dev/null -w "%{http_code}" {config.server.frontend.baseUrl}
curl -s -o /dev/null -w "%{http_code}" {config.server.backend.baseUrl}
```

서버가 응답하지 않으면 사용자에게 서버 구동을 요청한다.

### 4. 버그 조사 (/investigate)

FAIL 항목이 있으면 Skill 도구를 사용하여 `/investigate`를 호출한다.

gstack investigate가 4단계로 원인을 분석한다:
1. **Investigate**: 현상 파악 및 재현
2. **Analyze**: 관련 코드/로그 분석
3. **Hypothesize**: 근본 원인 가설 수립
4. **Implement**: 수정 방안 도출

### 5. 버그 수정 (/qa)

원인이 파악되면 Skill 도구를 사용하여 `/qa`를 호출한다.

gstack qa가 수행하는 작업:
- 소스 코드에서 버그 수정
- 수정 사항을 atomic commit으로 커밋
- 수정 후 해당 항목 재검증
- before/after 스크린샷으로 증적

### 6. 반복

모든 FAIL 항목이 PASS될 때까지 4~5 단계를 반복한다.

**중단 조건:**
- 5회 이상 수정해도 해결되지 않는 버그는 사용자에게 보고
- 근본적인 설계 변경이 필요한 경우 Step 3(구현)으로 복귀 권고

### 7. 결과 기록

수정 완료 후 QA 체크리스트에 결과를 업데이트한다:
- FAIL → PASS로 변경된 항목 표시
- 수정된 파일 목록
- 남은 이슈 (있는 경우)
