---
name: riskzero-si-pipeline
version: 1.0.0
description: Use when developing an SI feature end-to-end with riskzero-si, initializing project config, or running selected pipeline stages; triggers include "SI 파이프라인", "기능 개발", "riskzero pipeline". 일반적인 기능 개발 요청에는 사용하지 않는다.
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

# SI 개발 파이프라인 오케스트레이터

## 역할

이 스킬은 SI(System Integration) 프로젝트의 기능 개발을 8단계 파이프라인으로 자동화하는 **오케스트레이터**이다.

개발자가 기능명 하나만 입력하면, 기획서 분석부터 코드 구현, 코드 리뷰, QA, 버그 수정, 최종 검증까지 전체 개발 라이프사이클을 순차적으로 실행한다. 각 단계는 독립적인 서브 스킬로 분리되어 있으며, 오케스트레이터는 단계 간 데이터 흐름과 게이트(진행/중단 판단)를 관리한다.

**핵심 원칙:**
- 각 단계의 산출물은 다음 단계의 입력이 된다
- 리뷰 단계에서 문제가 발견되면 이전 단계로 되돌아간다
- 모든 영구 산출물은 `config.outputs.root/config.outputs.featureDirPattern` 규칙으로 계산한 기능별 산출물 디렉토리에 저장한다
- `/tmp`는 임시 파일에만 사용하며, 최종 체크리스트/리포트/증거는 기능별 산출물 디렉토리로 복사하거나 직접 저장한다
- 사용자 확인 없이 다음 단계로 넘어가지 않는다
- 멀티 에이전트나 Dynamic Workflow를 쓰더라도 단계 게이트와 산출물 계약은 유지한다
- **매 단계 종료 시 [단계 종료 푸터](#단계-종료-푸터-필수-출력-규약)(전체 진행 현황 + 다음 단계 명령어 + `/clear` 안내)를 답변 맨 끝에 반드시 출력한다.** 이 푸터가 없으면 그 단계는 완료된 것이 아니다.

---

## 산출물 저장 계약

기본 산출물 루트는 `.si-planning/{기능명}/`이다. `si-config.yml`의 `outputs.root`와
`outputs.featureDirPattern`이 설정되어 있으면 해당 값을 우선한다.

| 구분 | 기본 경로 | 설명 |
|------|-----------|------|
| 설계 전 논의 | `.si-planning/{기능명}/discussion.md` | 사용자가 결정한 gray area와 미결정/위임 항목 |
| 계획 전 리서치 | `.si-planning/{기능명}/research.md` | 계획 전 기술 접근, 대안, 위험 조사 결과(선택) |
| 구현 계획 | `.si-planning/{기능명}/implementation-plan.md` | 최종 구현 설계 |
| TDD 계획 | `.si-planning/{기능명}/tdd-plan.md` | RED 테스트 케이스와 실행 명령 |
| TDD 증거 | `.si-planning/{기능명}/tdd-report.md` | RED/GREEN/REFACTOR 실행 기록 |
| 계획 리뷰 | `.si-planning/{기능명}/plan-review.md` | 아키텍처/보안 계획 리뷰 |
| 코드 리뷰 | `.si-planning/{기능명}/code-review.md` | 프로젝트 표준 + 리서치/TDD 증거 리뷰 |
| PR 리뷰 | `.si-planning/{기능명}/pr-review.md` | diff 기반 안전성 리뷰 |
| QA 체크리스트 | `.si-planning/{기능명}/qa-checklist.md` | 기능 QA 항목 |
| QA 리포트 | `.si-planning/{기능명}/qa-report.md` | QA 실행 결과 |
| 최종 리포트 | `.si-planning/{기능명}/final-report.md` | 최종 검증 요약 |
| 실행 증거 | `.si-planning/{기능명}/evidence/` | 스크린샷, 로그, 테스트 결과 |

기능별 산출물 디렉토리에는 아래 하위 디렉토리를 만든다:

```
evidence/
  screenshots/
  logs/
  test-results/
```

이 계약은 모든 서브 스킬의 기본 경로이다. 과거 호환을 위해 `/tmp/qa-*` 파일을
읽을 수는 있지만, 새로 생성하는 영구 산출물은 위 경로에 저장한다.

## 모델/오케스트레이션 정책

`si-config.yml`의 `orchestration.*` 설정이 있으면 단계별 실행 전략을 선택할 때 참고한다.
이 설정은 권장 라우팅이며, 런타임이 지원하지 않으면 현재 에이전트가 inline으로 실행한다.
subagent와 Dynamic Workflow는 단계 내부 보조자일 뿐이며, 다음 단계로 진행하거나 PASS/FAIL 게이트를 대신 확정할 수 없다.

| 단계 | 기본 추천 | 멀티 에이전트/Dynamic Workflow 사용 조건 |
|------|-----------|------------------------------------------|
| 1 계획/외부 리서치 | Claude Opus급 또는 Codex high reasoning | 외부 공식 문서 확인, 대규모 소스 분석처럼 read-only 병렬 작업이 있을 때 |
| 2 계획 리뷰 | Claude Opus급 또는 Codex review | 보안/아키텍처/TDD 반례를 독립 리뷰로 확인할 때 |
| 3 구현 | Codex | BE/FE/DB write set이 분리되어 충돌 위험이 낮을 때만 subagent 병렬화 |
| 4 코드 리뷰 | Codex review + Claude 교차 검토 | 구현 산출물은 고정하고 read-only 리뷰로만 병렬화 |
| 5 PR 리뷰 | Codex review | diff 기반 안전성 리뷰. 보조 리뷰는 read-only로 제한 |
| 6 QA 체크리스트 | Claude/Codex + gstack browse | 시나리오 작성과 브라우저 실행을 분리할 수 있을 때 |
| 7 QA 수정 | Codex | 재현/수정/검증 루프는 inline 우선. 독립 버그만 분리 |
| 8 최종 검증 | Codex + gstack browser | 스크린샷/로그 수집은 병렬 가능하나 최종 판정은 단일 리포트로 통합 |

Claude Code Dynamic Workflows는 research preview 성격의 기능이므로
`orchestration.dynamicWorkflowDefault`가 `allowed`가 아닌 한 사용자에게 먼저 묻는다.
작은 CRUD 화면, 사용자 의사결정이 잦은 설계 논의, 같은 파일을 여러 작업자가 수정하는 구현에는 사용하지 않는다.

### 병렬화 안전 제약

- subagent 기본 권한은 read-only로 본다.
- 병렬 쓰기는 `orchestration.allowParallelWrites: true`이고 BE/FE/DB처럼 write set이 명확히 분리될 때만 고려한다.
- 라우트 파일, API contract, `tdd-report.md`, `qa-report.md`, `final-report.md`처럼 공유 산출물은 inline 단일 오너가 작성한다.
- Step 6~8에서 브라우저/DB/인증 세션을 병렬로 조작하려면 테스트 데이터 prefix, 계정 격리, cleanup 정책이 있어야 한다.
- Dynamic Workflow 결과도 반드시 기능별 산출물 디렉토리로 합성하고, 최종 판정은 현재 파이프라인 오너가 기록한다.

## 사전 조건

### 설정 파일 필수

파이프라인 실행 전, **si-config.yml** 설정 파일이 반드시 존재해야 한다.

설정 파일이 없으면 다음 안내를 출력한다:

```
si-config.yml 설정 파일이 없습니다.
아래 명령으로 자동 생성할 수 있습니다:

  /riskzero-si-pipeline --init

또는 템플릿을 직접 복사하여 작성하세요:
  mkdir -p .codex
  cp ~/.codex/skills/riskzero-si/si-config.template.yml .codex/si-config.yml

Claude Code를 쓰는 경우:
  mkdir -p .claude
  cp ~/.claude/skills/riskzero-si/si-config.template.yml .claude/si-config.yml
```

### 설정 파일 탐색 순서

설정 파일은 다음 순서로 탐색한다. 먼저 발견된 파일을 사용한다:

1. `.codex/si-config.yml` (Codex 권장 위치)
2. `.claude/si-config.yml` (Claude Code 권장 위치)
3. `si-config.yml` (프로젝트 루트)
4. `.codex/qa-config.yml` (QA 전용 설정 호환)
5. `.claude/qa-config.yml` (QA 전용 설정 호환)

---

## --init 모드

`/riskzero-si-pipeline --init` 실행 시, 프로젝트 구조를 자동 감지하여 si-config.yml을 생성한다.

### 자동 감지 절차

1. **프로젝트 루트 탐색**
   - 현재 디렉토리 및 상위 디렉토리에서 프로젝트 경계를 찾는다
   - 모노레포 구조(frontend/backend 분리)인지 단일 프로젝트인지 판별한다

2. **프론트엔드 감지**
   - `package.json` 파일을 읽어 프레임워크를 판별한다
     - `react`, `react-dom` 의존성 → `react`
     - `vue` 의존성 → `vue`
     - `@angular/core` 의존성 → `angular`
     - `next` 의존성 → `next`
   - `scripts` 섹션에서 `build`, `lint`, `dev` 명령어를 추출한다
   - `vite.config.*`, `next.config.*`, `angular.json` 등으로 포트를 추출한다
   - `src/pages`, `src/views`, `app/` 등 페이지 디렉토리를 탐색한다
   - `src/router`, `src/routes`, `app/layout` 등 라우터 파일을 탐색한다

3. **백엔드 감지**
   - `build.gradle`, `build.gradle.kts` → Spring Boot (Gradle)
     - `sourceCompatibility`, `java.toolchain` 에서 Java 버전 추출
     - `group` 에서 basePackage 추출
   - `pom.xml` → Spring Boot (Maven)
     - `<groupId>`, `<java.version>` 추출
   - `requirements.txt`, `pyproject.toml` → Django/FastAPI
   - `package.json`의 `nestjs`, `express` 의존성 → NestJS/Express
   - `src/main/java` 하위 패키지 구조를 분석하여 structure 패턴을 추출한다
   - `src/main/resources/application.yml` 또는 `application.properties`에서 포트를 추출한다

4. **데이터베이스 감지**
   - 백엔드 설정 파일에서 DB 드라이버/URL을 분석하여 DB 타입을 판별한다
   - `*.sql`, `ddl`, `migration` 키워드가 포함된 디렉토리에서 DDL 파일을 탐색한다

5. **샘플 코드 감지**
   - `sample`, `example`, `demo` 키워드가 포함된 디렉토리를 탐색한다
   - README에서 샘플 경로 언급을 찾는다

6. **설정 파일 생성**
   - 감지 결과를 si-config.template.yml 템플릿에 매핑하여 Codex에서는 `.codex/si-config.yml`, Claude Code에서는 `.claude/si-config.yml`을 생성한다
   - 자동 감지가 불확실한 항목은 빈 값으로 두고 주석으로 표시한다

7. **사용자 확인**
   - 생성된 설정 파일 내용을 출력하고 사용자에게 검토를 요청한다
   - 수정이 필요한 항목이 있으면 안내한다

---

## 8단계 파이프라인

| 단계 | 스킬 명령 | 설명 | 산출물 |
|------|-----------|------|--------|
| 1 | `/riskzero-si-plan {기능명}` | 설계 전 논의 + 선택적 리서치 + 구현/TDD 계획 | `discussion.md`, `research.md`(선택), `implementation-plan.md`, `tdd-plan.md` |
| 2 | `/riskzero-si-plan-review` | 논의/리서치/TDD 포함 계획 리뷰 | `.si-planning/{기능명}/plan-review.md` |
| 3 | `/riskzero-si-impl {기능명}` | TDD 기반 FE/BE 코드 구현 | 실제 소스 파일들, `tdd-report.md` |
| 4 | `/riskzero-si-review` | 프로젝트 표준 + 리서치/TDD 증거 리뷰 | `.si-planning/{기능명}/code-review.md` |
| 5 | `/riskzero-si-pr-review` | PR diff 안전성 리뷰 | `.si-planning/{기능명}/pr-review.md` |
| 6 | `/riskzero-si-qa-checklist {기능명}` | QA 체크리스트 생성 | `.si-planning/{기능명}/qa-checklist.md`, `qa-report.md`(테스트 실행 시) |
| 7 | `/riskzero-si-qa` | 버그 조사 및 수정 | 수정된 소스 파일들 |
| 8 | `/riskzero-si-browse` | 브라우저 최종 검증 | `.si-planning/{기능명}/final-report.md`, `evidence/screenshots/` |

---

## 단계 종료 푸터 (필수 출력 규약)

오케스트레이터와 **모든 서브 스킬**은 한 단계를 끝낼 때마다 아래 "단계 종료 푸터"를 답변 맨 끝에 반드시 출력한다. 이 푸터가 없으면 그 단계는 완료된 것이 아니다.

**목적:**
- 사용자가 전체 8단계 중 지금 어디에 있는지 한눈에 파악한다
- 다음에 입력할 명령어를 **복사만 하면 되도록** 완성된 형태로 제공한다
- `/clear`로 컨텍스트를 비운 뒤에도 다음 단계를 바로 이어갈 수 있게 한다

### 푸터 출력 형식

답변 맨 끝에 아래 형식을 그대로 출력한다. `{기능명}`, 진행 표시(✅ 완료 / 👉 현재·다음 / ⬜ 대기), 산출물 경로, 다음 명령어는 실제 값으로 채운다.

> ---
> ### 📍 파이프라인 진행 현황 — `{기능명}`
>
> | # | 단계 | 명령 | 상태 |
> |---|------|------|------|
> | 1 | 논의+계획 | `riskzero-si-plan` | ✅ |
> | 2 | 계획 리뷰 | `riskzero-si-plan-review` | ✅ |
> | 3 | 구현(TDD) | `riskzero-si-impl` | 👉 방금 완료 |
> | 4 | 표준 리뷰 | `riskzero-si-review` | 👉 다음 |
> | 5 | PR 리뷰 | `riskzero-si-pr-review` | ⬜ |
> | 6 | QA 체크리스트 | `riskzero-si-qa-checklist` | ⬜ |
> | 7 | 버그 수정 | `riskzero-si-qa` | ⬜ |
> | 8 | 최종 검증 | `riskzero-si-browse` | ⬜ |
>
> **방금 완료한 단계:** Step 3 구현 — 산출물: `.si-planning/{기능명}/tdd-report.md` + 생성된 소스 파일
>
> **▶️ 다음 단계 — 아래 명령어를 복사해 실행하세요:**
> ```
> /riskzero-si-review {기능명}
> ```
>
> 💡 컨텍스트가 길어졌다면 `/clear` 후 위 명령어를 그대로 붙여넣으세요. 다음 단계는 `.si-planning/{기능명}/` 산출물을 읽어 단독 실행됩니다.
> ---

게이트가 FAIL/BLOCKER/CRITICAL이라 이전 단계로 되돌아가야 하면, "다음 단계" 자리에 **복귀 명령어**를 넣고 사유를 한 줄로 적는다. (예: `이슈로 인해 되돌아가기: /riskzero-si-plan {기능명}` — CRITICAL 계획 결함)

### 단계별 "다음 명령어" 맵

| 방금 끝낸 단계 | 게이트 PASS 시 다음 명령어 | 게이트 실패 시 복귀 명령어 |
|----------------|----------------------------|----------------------------|
| 1 plan | `/riskzero-si-plan-review {기능명}` | — |
| 2 plan-review | `/riskzero-si-impl {기능명}` | (CRITICAL) `/riskzero-si-plan {기능명}` |
| 3 impl | `/riskzero-si-review {기능명}` | (빌드/린트 실패) 수정 후 `/riskzero-si-impl {기능명}` |
| 4 review | `/riskzero-si-pr-review {기능명}` | (ERROR) 코드 수정 후 `/riskzero-si-review {기능명}` |
| 5 pr-review | `/riskzero-si-qa-checklist {메뉴명 또는 URL}` | (BLOCKER) 코드 수정 후 `/riskzero-si-pr-review {기능명}` |
| 6 qa-checklist | `/riskzero-si-qa {기능명}` | — |
| 7 qa | `/riskzero-si-browse {기능명 또는 URL}` | (5회+ 미해결) 사용자 보고 |
| 8 browse | 🎉 파이프라인 완료 | (CRITICAL) `/riskzero-si-qa {기능명}` |

### `/clear` 운용 정책 (단독 실행 안전성)

- 각 단계 산출물은 `.si-planning/{기능명}/`에 저장되어 **자기완결적**이다. 따라서 단계 사이에 `/clear`로 컨텍스트를 비워도, 다음 서브 스킬이 직전 산출물 파일을 읽어 단독으로 실행된다.
- 서브 스킬 대부분이 기능명을 인자로 받으므로, **다음 명령어에는 항상 `{기능명}`을 채워서** 출력한다. `/clear` 후 컨텍스트에서 기능명을 추론할 수 없기 때문이다.
- 예외: `/riskzero-si-qa-checklist`는 기능명이 아니라 **메뉴명 / URL / 화면ID**를 받는다. Step 5→6 전환 푸터에서는 이 점을 명시한다.
- 6→7, 7→8 전환은 `.si-planning/{기능명}/qa-checklist.md`를 기준으로 기능명을 자동 추론할 수 있으나, 혼선을 막기 위해 푸터에는 항상 기능명을 명시한다.

---

## --from, --to 옵션

파이프라인의 특정 구간만 실행할 수 있다.

```bash
# 3단계(구현)부터 실행
/riskzero-si-pipeline {기능명} --from=3

# 1~4단계만 실행 (논의/리서치/계획 → 리뷰 → 구현 → 코드 리뷰)
/riskzero-si-pipeline {기능명} --from=1 --to=4

# 6단계(QA)부터 끝까지
/riskzero-si-pipeline {기능명} --from=6

# 7단계만 실행 (버그 수정)
/riskzero-si-pipeline {기능명} --from=7 --to=7
```

**주의사항:**
- `--from`으로 중간 단계부터 시작할 때, 이전 단계의 산출물(`.si-planning/{기능명}/`)이 존재하는지 확인한다
- 산출물이 없으면 이전 단계부터 실행할 것을 권고한다
- `--to`를 지정하면 해당 단계 완료 후 파이프라인을 중단한다

---

## 단계별 실행 절차

### 1단계: 설계 전 논의 + 선택적 리서치 + 구현/TDD 계획 (`/riskzero-si-plan`)

**수행 내용:**
- si-config.yml에서 wireframe, publishing, ddl 경로를 읽는다
- 기획서/퍼블리싱/DDL/README를 얕게 분석하여 설계 전 논의가 필요한 gray area를 식별한다
- 사용자가 선택한 gray area를 논의하고 결정사항을 `discussion.md`에 저장한다
- 리서치 필요성을 판단하고 사용자에게 진행/스킵을 질문한다
- 리서치를 진행하면 기술 접근, 대안, 위험, 테스트 관점을 `research.md`에 저장한다
- 리서치를 스킵하거나 기존 `research.md`를 무시하면 `implementation-plan.md`에 상태와 사유를 기록한다
- 기획서(와이어프레임)를 분석하여 화면 구성, 필드, 동작을 파악한다
- 퍼블리싱 HTML/CSS를 분석하여 UI 컴포넌트 구조를 파악한다
- DDL을 분석하여 테이블 구조, 컬럼, 관계를 파악한다
- 샘플 코드를 참고하여 프로젝트 패턴에 맞는 구현 계획을 수립한다
- API 설계 (엔드포인트, 요청/응답 스펙)를 작성한다
- 프론트엔드 페이지 구조 및 컴포넌트 설계를 작성한다
- 백엔드 클래스 설계 (Controller, Service, Mapper, DTO, VO)를 작성한다
- 구현 전에 실패해야 하는 테스트 케이스와 실행 명령을 `tdd-plan.md`에 작성한다

**산출물:** `.si-planning/{기능명}/discussion.md`, `.si-planning/{기능명}/research.md`(선택), `.si-planning/{기능명}/implementation-plan.md`, `.si-planning/{기능명}/tdd-plan.md`

**진행 조건:** 사용자가 계획을 확인하고 승인한다.
**중단 조건:** 기획서, DDL 등 필수 입력 자료가 누락된 경우 사용자에게 경로를 확인 요청한다.

---

### 2단계: 논의/리서치/TDD 포함 계획 리뷰 (`/riskzero-si-plan-review`)

**수행 내용:**
- 1단계에서 생성된 `discussion.md`, `research.md`(있으면), `implementation-plan.md`, `tdd-plan.md`를 함께 리뷰한다
- `discussion.md`의 결정사항 또는 `research.md`의 권장사항/스킵 사유가 구현 계획에 반영되었는지 확인한다
- 프로젝트 README.md의 규칙이 계획에 반영되었는지 확인한다
- API 네이밍 규칙 준수 여부를 확인한다
- DTO/VO 분리 규칙 준수 여부를 확인한다
- 보안(권한 체크) 설계 포함 여부를 확인한다
- 유효성 검증 레이어 설계 포함 여부를 확인한다
- 구현 전에 실패해야 하는 RED 테스트가 `tdd-plan.md`에 구체적으로 정의되었는지 확인한다

**산출물:** `.si-planning/{기능명}/plan-review.md`

**진행 조건:** 리뷰 결과가 PASS이거나, 지적 사항을 1단계 계획에 반영 완료 후 재리뷰 통과.
**중단 조건:** CRITICAL 이슈가 있으면 1단계로 되돌아가 계획을 수정한다.

---

### 3단계: 코드 구현 (`/riskzero-si-impl`)

**수행 내용:**
- 승인된 `implementation-plan.md`를 기반으로 실제 코드를 생성한다
- `tdd-plan.md`의 테스트를 먼저 작성하고 RED(예상 실패)를 확인한다
- RED가 확인된 뒤 최소 구현으로 GREEN을 만들고, 필요 시 REFACTOR를 수행한다
- si-config.yml의 structure 패턴에 따라 파일을 생성한다
- 백엔드: Controller, Service, Mapper(interface), Mapper(XML), DTO, VO 파일 생성
- 프론트엔드: 목록 페이지, 상세 페이지, API 훅 파일 생성
- 라우터에 새 페이지 경로를 등록한다
- 샘플 코드의 패턴(임포트, 코드 스타일, 구조)을 정확히 따른다
- 템플릿 마커 주석(`// ---`)을 최종 코드에서 제거한다
- RED/GREEN/REFACTOR 실행 명령, 결과, 테스트 파일 목록을 `tdd-report.md`에 저장한다

**산출물:** 실제 소스 파일들 (프로젝트 디렉토리에 직접 생성), `.si-planning/{기능명}/tdd-report.md`

**진행 조건:** 빌드(buildCmd) 성공, 린트(lintCmd) 통과.
**중단 조건:** 빌드 또는 린트 실패 시 오류를 수정한 후 재실행. 3회 이상 실패 시 사용자에게 보고한다.

---

### 4단계: 프로젝트 표준 + 리서치/TDD 증거 리뷰 (`/riskzero-si-review`)

**수행 내용:**
- 3단계에서 생성/수정된 모든 파일을 대상으로 프로젝트 표준 준수 여부를 검사한다
- README.md 및 si-config.yml에 정의된 규칙 기반으로 리뷰한다
- `implementation-plan.md`가 리서치 수행/사용 상태이면 `research.md` 권장 접근, 위험 경고, 테스트 관점이 코드와 `tdd-report.md`에 반영되었는지 확인한다
- 검사 항목:
  - 네이밍 컨벤션 (변수, 메서드, 클래스, 파일명)
  - 패키지/디렉토리 구조
  - 임포트 순서 및 불필요한 임포트
  - 코드 스타일 (들여쓰기, 따옴표, 세미콜론)
  - DTO/VO 분리 규칙
  - 트랜잭션 어노테이션 위치
  - 권한 체크 어노테이션
  - API 응답 형식
  - 유효성 검증 누락
  - `tdd-plan.md`와 `tdd-report.md`가 존재하고 RED/GREEN 증거가 일치하는지 확인
  - 테스트 없이 구현된 주요 동작이 없는지 확인

**산출물:** `.si-planning/{기능명}/code-review.md`

**진행 조건:** 모든 항목 PASS 또는 WARNING 이하.
**중단 조건:** ERROR 항목이 있으면 3단계 코드를 수정한 후 재리뷰한다.

---

### 5단계: PR diff 리뷰 (`/riskzero-si-pr-review`)

**수행 내용:**
- git diff를 기반으로 변경된 코드 전체를 리뷰한다
- 코드 안전성 관점에서 품질을 평가한다
- 리뷰 관점:
  - 로직 정확성 및 엣지 케이스 처리
  - 보안 취약점 (SQL Injection, XSS 등)
  - 성능 이슈 (N+1 쿼리, 불필요한 재렌더링 등)
  - 에러 핸들링 누락
  - 테스트 필요 여부
  - TDD 증거 없이 추가된 위험 동작 또는 테스트 누락

**산출물:** `.si-planning/{기능명}/pr-review.md`

**진행 조건:** 리뷰 결과에 BLOCKER가 없다.
**중단 조건:** BLOCKER 이슈가 있으면 코드를 수정한 후 재리뷰한다.

---

### 6단계: QA 체크리스트 생성 (`/riskzero-si-qa-checklist`)

**수행 내용:**
- 구현된 기능의 QA 체크리스트를 생성한다
- 기획서 기반 기능 테스트 항목을 작성한다
- 항목 구성:
  - 화면 표시 확인 (목록, 상세, 등록, 수정, 삭제)
  - 필드별 입력 검증 (필수값, 형식, 길이)
  - 권한별 접근 제어 확인
  - 페이지네이션 동작
  - 파일 업로드/다운로드 (해당 시)
  - 에러 상황 처리 (네트워크 오류, 서버 오류)
  - 브라우저 호환성

**산출물:** `.si-planning/{기능명}/qa-checklist.md`

**진행 조건:** 체크리스트 생성 완료. 사용자 확인.
**중단 조건:** 없음 (항상 진행).

---

### 7단계: 버그 조사 및 수정 (`/riskzero-si-qa`)

**수행 내용:**
- 개발 서버를 실행하고 6단계 QA 체크리스트를 기반으로 테스트한다
- 발견된 버그를 조사(`/investigate`)하고 수정한다
- 수정 후 관련 항목을 재테스트(`/qa`)한다
- 반복: 모든 체크리스트 항목이 PASS될 때까지 수행한다

**수행 절차:**
1. 서버 상태 확인 (프론트엔드/백엔드 구동 여부)
2. QA 체크리스트의 각 항목을 순서대로 검증
3. FAIL 항목 발견 시:
   - 원인을 조사한다 (`/investigate`)
   - 코드를 수정한다
   - 해당 항목을 재검증한다
4. 모든 항목 PASS 후 결과를 기록한다

**산출물:** 수정된 소스 파일들, QA 체크리스트에 결과 업데이트

**진행 조건:** QA 체크리스트 전 항목 PASS.
**중단 조건:** 5회 이상 수정해도 해결되지 않는 버그는 사용자에게 보고하고 판단을 요청한다.

---

### 8단계: 브라우저 최종 검증 (`/riskzero-si-browse`)

**수행 내용:**
- 브라우저에서 실제 화면을 열고 최종 검증을 수행한다
- QA 체크리스트의 모든 항목을 브라우저에서 직접 확인한다
- 스크린샷을 캡처하여 증적을 남긴다
- 최종 보고서를 작성한다

**수행 절차:**
1. 프론트엔드 개발 서버 URL로 브라우저를 연다
2. 로그인 수행 (si-config.yml의 auth 설정 사용)
3. 대상 기능 페이지로 이동
4. 각 테스트 시나리오를 실행하며 스크린샷 캡처
5. 결과를 최종 보고서에 정리

**산출물:** `.si-planning/{기능명}/final-report.md`

**진행 조건:** 전 항목 검증 완료.
**중단 조건:** CRITICAL 이슈 발견 시 7단계로 되돌아간다.

---

## 실행 흐름 요약

```
[시작]
  │
  ▼
1. 논의 + 선택적 리서치 + 구현/TDD 계획 ──→ discussion.md + research.md(선택) + implementation-plan.md + tdd-plan.md
  │
  ▼
2. 계획 리뷰 ──────→ .si-planning/{기능명}/plan-review.md
  │                    ↑ CRITICAL 시 1단계로 복귀
  ▼
3. TDD 코드 구현 ───→ 실제 소스 파일 + tdd-report.md
  │                    ↑ 빌드 실패 시 재시도
  ▼
4. 표준+TDD 리뷰 ───→ .si-planning/{기능명}/code-review.md
  │                    ↑ ERROR 시 3단계 코드 수정
  ▼
5. PR 리뷰 ────────→ .si-planning/{기능명}/pr-review.md
  │                    ↑ BLOCKER 시 코드 수정
  ▼
6. QA 체크리스트 ──→ .si-planning/{기능명}/qa-checklist.md
  │
  ▼
7. 버그 수정 ──────→ 수정된 소스 파일
  │                    ↑ FAIL 항목 수정 반복
  ▼
8. 최종 검증 ──────→ .si-planning/{기능명}/final-report.md
  │                    ↑ CRITICAL 시 7단계로 복귀
  ▼
[완료]
```

---

## 출력 디렉토리 구조

모든 산출물은 기본적으로 프로젝트 루트의 `.si-planning/{기능명}/` 디렉토리에 저장된다.
`si-config.yml`의 `outputs` 설정이 있으면 해당 경로를 우선한다.

```
.si-planning/
  └── {기능명}/
      ├── discussion.md            # 1단계: 설계 전 논의 결정사항
      ├── research.md              # 1단계: 계획 전 기술 리서치(선택)
      ├── implementation-plan.md   # 1단계: 구현 계획서
      ├── tdd-plan.md              # 1단계: TDD 테스트 설계
      ├── plan-review.md           # 2단계: 계획 리뷰 결과
      ├── tdd-report.md            # 3단계: RED/GREEN/REFACTOR 증거
      ├── code-review.md           # 4단계: 코드 리뷰 결과
      ├── pr-review.md             # 5단계: PR 리뷰 결과
      ├── qa-checklist.md          # 6단계: QA 체크리스트
      ├── qa-report.md             # 6~7단계: QA 실행 리포트
      ├── final-report.md          # 8단계: 최종 검증 보고서
      └── evidence/
          ├── screenshots/
          ├── logs/
          └── test-results/
```

---

## 오케스트레이터 실행 로직

파이프라인 실행 시 오케스트레이터는 다음 순서로 동작한다:

1. **인자 파싱**: 기능명, --from, --to, --init 옵션을 파싱한다
2. **--init 처리**: --init 플래그가 있으면 초기화 모드로 진입한다
3. **설정 파일 로드**: si-config.yml을 탐색하고 로드한다
4. **산출물 디렉토리 생성**: `outputs` 설정을 해석하여 기능별 산출물 디렉토리와 `evidence/` 하위 디렉토리를 생성한다
5. **TaskCreate로 단계 등록**: 실행할 단계들을 Task로 등록하여 진행 상황을 추적한다
6. **순차 실행**: 각 단계를 Skill 호출로 실행한다
7. **게이트 체크**: 각 단계 완료 후 진행/중단/복귀를 판단한다
8. **완료 보고**: 전체 파이프라인 완료 시 요약을 출력한다

### 단계 간 데이터 전달

- 각 단계의 산출물 파일 경로를 다음 단계에 전달한다
- si-config.yml의 설정값은 모든 단계에서 공유한다
- 기능명은 모든 단계에서 일관되게 사용한다

### 오류 복구

- 서브 스킬 실행 실패 시 오류 내용을 사용자에게 보고한다
- 사용자가 수동으로 수정한 후 해당 단계부터 재실행할 수 있다 (`--from=N`)
- 이전 단계로 복귀가 필요한 경우 사용자에게 안내한다
