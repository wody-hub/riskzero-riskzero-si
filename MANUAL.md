# riskzero-si 사용 매뉴얼

> SI 개발 파이프라인 스킬 모음 — 기획서 + 퍼블리싱 + DDL로 8단계 자동 개발

---

## 목차

1. [개요](#1-개요)
2. [설치](#2-설치)
3. [사전 준비물](#3-사전-준비물)
4. [프로젝트 초기 설정](#4-프로젝트-초기-설정)
5. [8단계 파이프라인](#5-8단계-파이프라인)
6. [개별 스킬 사용법](#6-개별-스킬-사용법)
7. [단계별 실전 팁](#7-단계별-실전-팁)
8. [팀원별 활용 가이드](#8-팀원별-활용-가이드)
9. [si-config.yml 설정 가이드](#9-si-configyml-설정-가이드)
10. [트러블슈팅](#10-트러블슈팅)
11. [FAQ](#11-faq)

---

## 1. 개요

### riskzero-si란?

Codex 또는 Claude Code 위에서 동작하는 **SI 프로젝트 전용 개발 파이프라인**입니다.

화면 기획서(PPTX), 퍼블리싱 파일(HTML/TSX), DDL(SQL)을 입력하면 설계 전 논의, 선택적 리서치, 구현/TDD 계획 수립, 코드 생성, 코드 리뷰, QA 테스트, 버그 수정, 최종 검증까지 **8단계를 자동화**합니다.

### 전체 흐름

```
기획서 + 퍼블리싱 + DDL
        │
        ▼
Step 1: /riskzero-si-plan           ─ 논의 + 선택적 리서치 + 구현/TDD 계획
        │
Step 2: /riskzero-si-plan-review    ─ 논의/리서치/TDD 포함 계획 리뷰
        │
Step 3: /riskzero-si-impl           ─ TDD 기반 FE/BE 코드 구현
        │
Step 4: /riskzero-si-review         ─ 프로젝트 표준 + 리서치/TDD 증거 리뷰
        │
Step 5: /riskzero-si-pr-review      ─ PR diff 안전성 리뷰
        │
Step 6: /riskzero-si-qa-checklist   ─ QA 체크리스트 생성
        │
Step 7: /riskzero-si-qa             ─ 버그 조사 + 수정
        │
Step 8: /riskzero-si-browse         ─ 최종 브라우저 검증
        │
        ▼
      완료!
```

### 사전 요구사항

| 항목 | 버전 | 용도 |
|------|------|------|
| Codex 또는 Claude Code | 최신 | AI 코딩 어시스턴트 |
| Bun | v1.0+ | gstack 빌드/실행 |
| gstack | 최신 | 브라우저 자동화, PR 리뷰 등 |

---

## 2. 설치

설치/업데이트 절차는 **[README.md의 설치 섹션](README.md#설치)** 을 따릅니다.
(Bun → gstack → riskzero-si 순서, host별 `./setup --host codex|claude`, 설치 확인, 업데이트 방법 포함)

---

## 3. 사전 준비물

파이프라인을 실행하기 전에 아래 자료를 준비해야 합니다.

### 3-1. 기획서 (wireframe)

| 항목 | 설명 |
|------|------|
| **형식** | PPTX, PDF, PNG/JPG 이미지 |
| **내용** | 화면 레이아웃, 필드 정의, 동작 흐름(등록/수정/삭제/검색) |
| **위치** | si-config.yml의 `sources.wireframe` 경로에 저장 |
| **없는 경우** | DDL만으로도 계획 수립 가능 (품질은 다소 낮아짐) |

> 기획자/PM에게 해당 기능의 화면 설계서를 요청하세요.

### 3-2. 퍼블리싱 (publishing)

| 항목 | 설명 |
|------|------|
| **형식** | HTML, TSX, CSS 파일 (마크업 개발자의 산출물) |
| **내용** | UI 레이아웃, 폼 구조, 테이블 구조가 잡힌 정적 화면 |
| **위치** | si-config.yml의 `sources.publishing` 경로에 저장 |
| **없는 경우** | 기획서 + DDL만으로 진행 가능 |

### 3-3. DDL (테이블 정의)

| 항목 | 설명 |
|------|------|
| **형식** | SQL 파일 (CREATE TABLE 문) |
| **내용** | 해당 기능에서 사용하는 테이블 DDL. 전체 DDL도 가능 |
| **위치** | si-config.yml의 `sources.ddl` 또는 `database.ddlPaths` 경로 |
| **필수 여부** | **필수** — 데이터 모델 없이는 API 설계 불가 |

### 3-4. 프로젝트 README.md

| 항목 | 설명 |
|------|------|
| **역할** | 프로젝트의 코딩 규칙, 네이밍 컨벤션, 패키지 구조 정의 |
| **경로** | si-config.yml의 `project.readme` |
| **없는 경우** | 구현/리뷰 시 프로젝트 표준이 적용되지 않음 |

> README.md에 **코딩 스타일 가이드**, **패키지 구조**, **API 설계 규칙** 섹션이 있으면 AI가 이를 학습하여 프로젝트에 맞는 코드를 생성합니다. README가 상세할수록 결과 품질이 높아집니다.

### 3-5. 개발 서버

Step 6(QA 체크리스트)부터 Step 8(브라우저 검증)까지 **프론트엔드/백엔드 서버가 실행 중이어야** 합니다.

```bash
# 서버 기동 후 파이프라인 실행
# 프론트엔드 서버 (터미널 1)
cd {frontend.root} && npm run dev

# 백엔드 서버 (터미널 2)
cd {backend.root} && ./gradlew bootRun

# 파이프라인 실행 (터미널 3 — Codex 또는 Claude Code)
/riskzero-si-pipeline 안전보건자료실
```

> Step 1~5는 서버 없이 실행 가능합니다. 코드 분석만 수행하기 때문입니다.

### 3-6. Git 워크플로우

```
main ─── feature/{기능명} ─── 파이프라인 실행 ─── PR 생성
```

- **feature 브랜치**에서 작업을 권장합니다
- 파이프라인이 생성한 코드는 기존 파일을 **덮어쓰지 않고 새로 생성**합니다
- 기존 파일과 충돌 가능성이 있으면 AI가 확인을 요청합니다

---

## 4. 프로젝트 초기 설정

새 프로젝트에서 처음 사용할 때 **1회만** 실행합니다.

### 자동 설정

```
/riskzero-si-pipeline --init
```

이 명령은 프로젝트 구조를 자동 감지합니다:
- `package.json` → 프론트엔드 프레임워크(React, Vue, Angular, Next.js)
- `build.gradle` / `pom.xml` → 백엔드 프레임워크(Spring Boot)
- `application.yml` → 서버 포트, DB 설정
- DDL 파일 위치

감지 결과를 기반으로 Codex에서는 `.codex/si-config.yml`, Claude Code에서는 `.claude/si-config.yml`을 생성하고, 사용자에게 검토를 요청합니다.

### 수동 설정

자동 감지 대신 템플릿을 직접 복사하여 작성할 수도 있습니다:

```bash
mkdir -p .codex
cp ~/.codex/skills/riskzero-si/si-config.template.yml .codex/si-config.yml
```

Claude Code를 쓰는 경우에는 아래처럼 `.claude/si-config.yml`로 복사합니다:

```bash
mkdir -p .claude
cp ~/.claude/skills/riskzero-si/si-config.template.yml .claude/si-config.yml
```

생성한 설정 파일을 열어 프로젝트에 맞게 수정합니다. 상세 설정은 [9. si-config.yml 설정 가이드](#9-si-configyml-설정-가이드)를 참조하세요.

---

## 5. 8단계 파이프라인

### 전체 실행

```
/riskzero-si-pipeline {기능명}
```

예시:
```
/riskzero-si-pipeline 안전보건자료실
/riskzero-si-pipeline 사용자관리
/riskzero-si-pipeline 공지사항
```

8단계를 **순차 실행**합니다. 각 단계 완료 후 결과를 확인하고, 문제가 없으면 다음 단계로 진행합니다.

### 단계 종료 푸터 — 다음 명령어와 `/clear`

모든 스킬은 한 단계를 끝낼 때마다 답변 맨 끝에 **단계 종료 푸터**를 출력합니다:

- 전체 8단계 중 현재 위치 (✅완료 / 👉현재·다음 / ⬜대기)
- **복사해서 바로 실행할 다음 단계 명령어** (기능명이 채워진 완전한 형태)
- `/clear` 안내

각 단계 산출물은 `.si-planning/{기능명}/`에 자기완결적으로 저장되므로, 단계 사이에 `/clear`로 컨텍스트를 비운 뒤 푸터의 다음 명령어를 붙여넣으면 다음 단계가 직전 산출물을 읽어 단독 실행됩니다. 긴 기능은 단계마다 `/clear`하면 토큰 누적을 막아 후반 단계 품질을 유지할 수 있습니다.

> ⚠️ Step 6 `/riskzero-si-qa-checklist`만 기능명이 아니라 **메뉴명/URL/화면ID**를 인자로 받습니다(Step 5 푸터에서 안내).

### 구간 실행

특정 구간만 실행할 수 있습니다:

```bash
# 3단계(구현)부터 시작
/riskzero-si-pipeline 안전보건자료실 --from=3

# 1~4단계만 (계획 → 리뷰 → 구현 → 코드 리뷰)
/riskzero-si-pipeline 안전보건자료실 --from=1 --to=4

# 6단계(QA)부터 끝까지
/riskzero-si-pipeline 안전보건자료실 --from=6

# 7단계만 (버그 수정)
/riskzero-si-pipeline 안전보건자료실 --from=7 --to=7
```

> **주의**: `--from`으로 중간부터 시작할 때, 이전 단계의 산출물(`.si-planning/{기능명}/`)이 있어야 합니다.

### 산출물 구조

모든 산출물은 `.si-planning/{기능명}/` 디렉토리에 저장됩니다:

```
.si-planning/
  └── 안전보건자료실/
      ├── discussion.md            # Step 1: 설계 전 논의 결정사항
      ├── research.md              # Step 1: 계획 전 기술 리서치(선택)
      ├── implementation-plan.md   # Step 1: 구현 계획서
      ├── tdd-plan.md              # Step 1: TDD 테스트 설계
      ├── plan-review.md           # Step 2: 계획 리뷰 결과
      ├── tdd-report.md            # Step 3: RED/GREEN/REFACTOR 증거
      ├── code-review.md           # Step 4: 코드 리뷰 결과
      ├── pr-review.md             # Step 5: PR 리뷰 결과
      ├── qa-checklist.md          # Step 6: QA 체크리스트
      ├── qa-report.md             # Step 6~7: QA 실행 리포트
      ├── final-report.md          # Step 8: 최종 검증 보고서
      └── evidence/
          ├── screenshots/
          ├── logs/
          └── test-results/
```

---

## 6. 개별 스킬 사용법

각 단계를 독립적으로 실행할 수 있습니다.

---

### Step 1: `/riskzero-si-plan {기능명}` — 논의 + 선택적 리서치 + 구현/TDD 계획

**입력**: 기획서(PPTX) + 퍼블리싱(HTML/TSX) + DDL(SQL)

**하는 일**:
- 기획서에서 화면 구성, 필드 정의, 동작 흐름을 분석
- 퍼블리싱에서 UI 컴포넌트 구조를 분석
- DDL에서 테이블 구조, 컬럼, 관계를 분석
- 기존 코드 패턴(README.md, 샘플 코드)을 학습
- 설계 전 gray area를 논의하고 `discussion.md`에 결정사항을 저장
- 리서치 필요성을 판단해 사용자에게 진행/스킵을 묻고, 진행 시 `research.md`에 결과를 저장
- API 설계, 데이터 모델, 컴포넌트 구조, 파일 배치, 태스크 분해를 포함한 구현 계획서 생성
- 구현 전 RED 테스트 케이스와 실행 명령을 `tdd-plan.md`에 저장

**사용 예시**:
```
/riskzero-si-plan 안전보건자료실
/riskzero-si-plan 안전보건자료실 --research       # 리서치 강제 진행
/riskzero-si-plan 안전보건자료실 --skip-research  # 리서치 강제 스킵
```

**산출물**: `.si-planning/안전보건자료실/discussion.md`, `research.md`(선택), `implementation-plan.md`, `tdd-plan.md`

> `discussion.md`에는 설계 전 gray area 논의 결정사항이 저장됩니다.
> `research.md`는 기술 접근, 대안, 위험, 테스트 관점을 계획 전에 조사한 경우에만 생성됩니다.
> 외부 리서치를 수행하면 공식 문서/표준/벤더 문서를 우선 확인하고, URL/확인일/적용 판단을 `research.md`에 남깁니다.
> 네트워크나 웹 도구가 없으면 `External Research Status: unavailable`로 기록하고 내부 자료 기반으로 진행합니다.
> 기존 `research.md`를 사용하지 않기로 하면 계획서에 `ignored-existing` 상태와 사유가 남고, 후속 단계는 그 상태를 우선합니다.
> `tdd-plan.md`에는 구현 전에 실패를 확인해야 하는 RED 테스트 케이스가 저장됩니다.

---

### Step 2: `/riskzero-si-plan-review` — 논의/리서치/TDD 포함 계획 리뷰

**입력**: Step 1에서 생성된 `discussion.md`, `research.md`(있으면), `implementation-plan.md`, `tdd-plan.md`

**하는 일**:
- 프로젝트 README.md 표준 준수 여부 사전 체크 (패키지 구조, 네이밍, API 설계, 보안, 검증)
- 아키텍처, 데이터 흐름, 엣지 케이스, 보안 설계를 점검
- `discussion.md` 결정사항, `research.md` 권장사항/스킵 사유, `tdd-plan.md` RED 테스트 구체성을 확인
- PASS / WARN / FAIL 판정

**사용 예시**:
```
/riskzero-si-plan-review
```

**진행 게이트**: 판정 라벨이 아니라 **차단 발견사항(체크 항목 FAIL·수정필요 지적) 건수**로 진행을 판단한다.
- **차단 0건**(전 항목 PASS, 잔여 Low/권고만) → Step 3으로 진행
- **차단 ≥1건** → Step 1(`/riskzero-si-plan`)로 돌아가 수정 후 재리뷰 (차단 0건까지 반복). WARN(Low/권고만)은 비차단

---

### Step 3: `/riskzero-si-impl {기능명}` — TDD 기반 FE/BE 코드 구현

**입력**: 승인된 구현 계획서 + `research.md`(있으면) + `tdd-plan.md`

**하는 일**:
- 계획서를 기반으로 백엔드(Controller, Service, Mapper, DTO, VO) 생성
- 프론트엔드(목록 페이지, 상세 페이지, API 훅) 생성
- 테스트 먼저 작성 → RED 확인 → 최소 구현 → GREEN 확인 → 리팩터링
- Ralph-loop 패턴으로 빌드/린트/테스트 검증과 자체 점검 반복 (최대 3회)

**옵션**:
```bash
/riskzero-si-impl 안전보건자료실              # FE + BE 모두
/riskzero-si-impl 안전보건자료실 --be-only    # 백엔드만
/riskzero-si-impl 안전보건자료실 --fe-only    # 프론트엔드만
```

**산출물**: 실제 소스 코드 파일들 + `.si-planning/{기능명}/tdd-report.md`

---

### Step 4: `/riskzero-si-review [경로]` — 프로젝트 표준 + 리서치/TDD 증거 리뷰

**입력**: Step 3에서 생성된 코드 (또는 지정된 파일/디렉토리)

**하는 일**:
- **아키텍처 점검**: 레이어 분리, 패키지 구조, 트랜잭션 설계, 보안 취약점
- **코딩 컨벤션 점검**: 네이밍 규칙, 코드 스타일, 미사용 변수, DTO/VO 분리
- **API 설계 점검**: REST URI, HTTP 메서드, 응답 형식, 데이터 모델 일관성
- **리서치 반영 점검**: 계획서가 리서치 수행/사용 상태이면 `research.md` 권장 접근과 위험 경고가 코드에 반영됐는지 확인
- **TDD 증거 점검**: `tdd-plan.md`와 `tdd-report.md`의 RED/GREEN 기록 확인
- README.md에서 프로젝트별 규칙을 동적으로 읽어 적용

**사용 예시**:
```bash
/riskzero-si-review                           # 최근 변경 파일 대상
/riskzero-si-review src/pages/example/        # 특정 디렉토리 지정
```

**핵심**: "우리 프로젝트 규칙을 따르는가?"를 점검합니다.

**진행 게이트**: 판정 라벨(PASS/CONDITIONAL PASS/FAIL)이 아니라 **발견사항 심각도**로 진행을 판단합니다(PASS 라벨도 Medium을 포함할 수 있음). 차단 = **Critical/High/Medium·영역 FAIL**.
- **차단 0건**(잔여 Low/권고만) → Step 5로 진행
- **차단 ≥1건** → 코드 수정 후 `/riskzero-si-review` 재리뷰 (차단 0건까지 반복). 명시적 수용 항목은 `code-review.md`에 사유·추적 위치 기록 시 차단 제외

---

### Step 5: `/riskzero-si-pr-review` — PR diff 안전성 리뷰

**입력**: git diff (현재 브랜치의 변경 사항)

**하는 일**:
- SQL Injection, XSS 등 보안 취약점 점검
- N+1 쿼리, 불필요한 재렌더링 등 성능 이슈 점검
- 에러 핸들링 누락 점검
- 사이드이펙트 분석

**사용 예시**:
```
/riskzero-si-pr-review
```

**핵심**: "코드가 안전한가?"를 점검합니다. Step 4와 상호보완 관계입니다.

**진행 게이트**: 차단 = **Medium 이상 / BLOCKER / TDD 증거 FAIL**.
- **차단 0건**(잔여 Low/권고만) → Step 6으로 진행
- **차단 ≥1건** → 코드 수정 후 `/riskzero-si-pr-review` 재리뷰 (차단 0건까지 반복). TDD 증거 부재면 Step 3 복귀

---

### Step 6: `/riskzero-si-qa-checklist {기능명}` — QA 체크리스트 생성

**입력**: 기능명 또는 URL 경로

**하는 일**:
- 대상 화면의 프론트엔드/백엔드/DDL 코드를 분석
- 6개 카테고리 체크리스트 자동 생성:
  - **A. 등록**: 필드별 검증, 필수값, 파일 첨부, 정상 저장
  - **B. 수정**: 수정 모드, 기존 데이터, 변경 저장
  - **C. 검색**: 개별 검색, 복합 검색, 초기화
  - **D. 페이징**: 페이지 이동, 크기 변경, 총 건수
  - **E. 네비게이션**: 목록↔상세 이동, 검색조건 유지
  - **F. 삭제**: 확인 다이얼로그, 삭제 처리
- 선택적으로 더미데이터 생성 및 브라우저 자동 테스트 실행

**사용 예시**:
```
/riskzero-si-qa-checklist 안전보건자료실
```

**산출물**: QA 체크리스트 마크다운 파일

---

### Step 7: `/riskzero-si-qa [URL]` — 버그 조사 + 수정

**입력**: QA 체크리스트의 FAIL 항목

**하는 일**:
- 4단계 원인 분석: 현상 파악 → 코드 분석 → 가설 수립 → 수정 방안 도출
- 코드 수정 후 해당 항목 재검증
- 모든 FAIL 항목이 PASS될 때까지 반복

**사용 예시**:
```
/riskzero-si-qa
/riskzero-si-qa {si-config의 frontend.baseUrl}/목록경로
```

**핵심 원칙**: 원인 파악 없이 코드를 수정하지 않습니다.

---

### Step 8: `/riskzero-si-browse [URL]` — 최종 브라우저 검증

**입력**: 수정 완료된 코드 + QA 체크리스트

**하는 일**:
- 실제 브라우저에서 화면을 열고 최종 검증
- 각 테스트 항목별 스크린샷 캡처
- PASS/FAIL 판정 및 최종 리포트 생성

**두 가지 모드**:
- **리포트 전용**: 버그 리포트만 생성, 코드 수정 없음
- **브라우저 탐색**: 특정 URL을 직접 열어서 검증

**사용 예시**:
```
/riskzero-si-browse
/riskzero-si-browse {si-config의 frontend.baseUrl}/목록경로
```

**산출물**: `.si-planning/{기능명}/final-report.md` + `.si-planning/{기능명}/evidence/screenshots/` 스크린샷 파일들

---

## 7. 단계별 실전 팁

각 단계에서 AI가 어떤 질문을 하고, 무엇을 확인해야 하는지 안내합니다.

### Step 1: 논의/리서치/구현 계획 — "계획서를 꼼꼼히 읽어라"

**AI가 하는 일**: 기획서/퍼블리싱/DDL/README를 사전 스캔해 설계 전 논의가 필요한 gray area를 정리하고, 리서치 필요성을 판단해 사용자에게 묻습니다. 결정사항과 research 결과 또는 스킵 사유를 반영해 API 설계, 데이터 모델, 컴포넌트 구조, 파일 배치, 태스크 분해, TDD 계획을 작성합니다.

**AI가 물어보는 것**:
- "기획서에 {기능}의 상세 동작이 명시되어 있지 않습니다. 어떻게 처리할까요?"
- "DDL에 {테이블}이 없습니다. 새로 생성해야 하나요?"
- "첨부파일/권한/삭제 정책은 프로젝트 표준대로 위임할까요, 아니면 이번 기능에서 별도 결정할까요?"
- "이 기능은 외부 API/새 패턴이 있어 리서치를 권장합니다. 진행할까요, 스킵할까요?"
- "외부 공식 문서까지 확인할까요, 아니면 내부 README/샘플 코드만 기준으로 계획할까요?"

**확인 포인트**:
- `discussion.md`의 결정사항이 계획서에 반영됐는지 확인
- 계획서가 리서치 수행/사용 상태이면 `research.md` 권장 접근과 위험이 계획서에 반영됐는지 확인
- 외부 리서치를 했다면 출처 URL, 확인일, 적용 판단이 `research.md`에 있는지 확인
- 리서치를 스킵했다면 계획서에 스킵 사유가 적혀 있는지 확인
- `.si-planning/{기능명}/implementation-plan.md` 파일을 열어서 **API 엔드포인트 목록**이 맞는지 확인
- `tdd-plan.md`의 RED 테스트 케이스가 핵심 동작을 포함하는지 확인
- **테이블 매핑**이 정확한지 확인 (잘못된 컬럼 참조가 있을 수 있음)
- **파일 배치 경로**가 프로젝트 구조와 맞는지 확인

### Step 2: 계획+TDD 리뷰 — "FAIL이면 수정 요청"

**AI가 하는 일**: 논의 결정사항, 리서치 반영/스킵 사유, TDD 계획, 아키텍처/보안/표준 관점을 함께 검토하고 PASS/WARN/FAIL 판정합니다.

**결과별 대응** (게이트는 발견사항 기준 — 차단 = FAIL 항목·수정필요 지적):
- **차단 0건 (전 항목 PASS, 잔여 Low/권고만)**: 바로 Step 3으로 진행
- **WARN (Low/권고만)**: 비차단 — 권고사항을 읽고 진행(필요 시 반영)
- **차단 ≥1건 (FAIL 등)**: `/riskzero-si-plan`을 재실행해 계획을 수정한 뒤 재리뷰. 차단 0건이 될 때까지 반복

### Step 3: 코드 구현 — "빌드 성공이 기본"

**AI가 하는 일**: `tdd-plan.md` 기준으로 테스트를 먼저 작성해 RED 실패를 확인하고,
최소 구현으로 GREEN을 만든 뒤 빌드/린트/테스트 검증을 반복합니다 (TDD + Ralph-loop, 최대 3회).

**AI가 물어보는 것**:
- "기존 파일 {파일명}이 이미 존재합니다. 덮어쓸까요?"
- "빌드 에러가 발생했습니다. 수정 후 재시도할까요?"

**확인 포인트**:
- 빌드가 성공하는지 확인 (빌드 실패 시 AI가 자동 수정 시도)
- `tdd-report.md`에 RED/GREEN 실행 결과가 남았는지 확인
- 생성된 파일 목록이 계획서의 파일 배치와 일치하는지 확인
- 코드 스타일이 프로젝트 규칙과 맞는지 눈으로 한번 확인

### Step 4~5: 리뷰 — "두 가지 관점"

**Step 4 (표준+TDD 리뷰)**: "우리 프로젝트 규칙을 따르고, 리서치/TDD 증거가 맞는가?" — 네이밍, 패키지 구조, API 설계, RED/GREEN 기록 등
**Step 5 (PR 리뷰)**: "코드가 안전한가?" — SQL injection, XSS, N+1 쿼리 등

**확인 포인트**:
- 지적된 이슈 중 **CRITICAL/HIGH**는 반드시 수정
- TDD 증거 누락 이슈는 테스트 작성 또는 예외 사유 기록으로 해소
- MEDIUM 이하는 판단하여 수정 여부 결정
- 수정 후 해당 리뷰를 다시 실행하여 이슈 해소 확인

### 모델/멀티 에이전트 운영 — "병렬화는 독립 작업에만"

**가능한 방식**:
- Codex: 현재 세션의 subagent 도구가 있으면 독립 리서치, BE/FE 분리 구현, 리뷰, QA 증거 수집을 병렬 위임할 수 있습니다. 단, 인라인 모델 지정은 런타임이 지원할 때만 가능합니다.
- Claude Code: custom subagent의 `model` frontmatter로 `opus`, `sonnet`, `haiku` 또는 모델 ID를 지정할 수 있습니다. background subagent와 `--agent` 실행도 사용할 수 있습니다.
- Claude Code Dynamic Workflows: Max/Team/Enterprise 등 지원 플랜과 설정이 있으면 대규모 병렬 작업에 사용할 수 있습니다. research preview 기능이므로 비용과 토큰 사용량을 확인하고 시작해야 합니다.

**권장 라우팅**:
| 단계 | 추천 |
|------|------|
| Step 1 계획/외부 리서치 | Claude Opus급 또는 Codex high reasoning. 외부 리서치는 read-only subagent에 위임 가능 |
| Step 2 계획 리뷰 | Claude Opus급 또는 Codex review. 설계 반례/보안/TDD 누락 점검 |
| Step 3 구현 | Codex 중심. BE/FE 파일 범위가 분리될 때만 subagent 병렬화 |
| Step 4 코드 리뷰 | Codex review + Claude 교차 검토 |
| Step 5 PR 리뷰 | Codex review |
| Step 6 QA 체크리스트 | Claude/Codex 모두 가능. 브라우저 실행은 gstack browse |
| Step 7 QA 수정 | Codex 중심. 재현→수정→검증 루프 |
| Step 8 최종 검증 | Codex + gstack browser |

**Dynamic Workflow를 쓰기 좋은 경우**:
- 여러 도메인/화면/모듈을 동시에 조사해야 하는 대규모 기능
- 보안/성능/죽은 코드/마이그레이션처럼 독립 검증이 많은 작업
- BE/FE/DB/QA가 서로 분리된 파일 집합으로 나뉘고, 마지막 통합자가 명확한 경우

**쓰면 위험한 경우**:
- 설계 전 논의처럼 사용자의 의사결정이 자주 필요한 단계
- 같은 파일을 여러 에이전트가 동시에 수정할 가능성이 큰 단계
- 권한/결제/운영 데이터처럼 실수 비용이 큰 변경
- 작은 CRUD 한두 화면처럼 병렬화 비용이 이득보다 큰 작업

### Step 6: QA 체크리스트 — "서버가 떠 있어야 한다"

**사전 조건**: 프론트엔드 + 백엔드 서버가 **반드시 실행 중**이어야 합니다.

**AI가 하는 일**: 코드를 분석하여 등록/수정/검색/페이징/네비게이션/삭제 체크리스트를 자동 생성합니다.

**AI가 물어보는 것**:
- "더미데이터를 생성할까요?" → 테스트 데이터가 없으면 "예"
- "브라우저 자동 테스트를 실행할까요?" → 체크리스트 확인 후 "예"
- "로그인 계정 정보를 알려주세요" → 테스트용 ID/PW 입력

### Step 7: 버그 수정 — "원인 파악이 먼저"

**AI가 하는 일**: FAIL 항목의 원인을 4단계로 분석(현상→코드→가설→수정)하고, 수정 후 재검증합니다.

**핵심 원칙**: 원인 파악 없이 코드를 수정하지 않습니다. AI가 원인을 찾지 못하면 사용자에게 물어봅니다.

**확인 포인트**:
- 수정된 코드가 다른 기능에 영향을 주지 않는지 확인
- FAIL → PASS로 변경된 항목의 스크린샷 확인

### Step 8: 최종 검증 — "스크린샷이 증거"

**AI가 하는 일**: 모든 체크리스트 항목을 실제 브라우저에서 검증하고, 스크린샷 + 최종 리포트를 생성합니다.

**산출물**: `.si-planning/{기능명}/final-report.md` + `.si-planning/{기능명}/evidence/screenshots/` 스크린샷 파일들

**완료 후**: 모든 PASS이면 feature 브랜치에서 PR을 생성하여 코드 리뷰를 요청합니다.

---

## 8. 팀원별 활용 가이드

### 신입 개발자 — 전체 자동

```
/riskzero-si-pipeline 안전보건자료실
```

8단계 전체를 순차적으로 실행합니다. 각 단계에서 확인을 요청하므로 결과를 검토하며 진행하면 됩니다.

### 경험 개발자 — 계획 + 직접 구현 + 리뷰

```
/riskzero-si-plan 안전보건자료실       # 계획만 수립
# → 직접 코딩
/riskzero-si-review                    # 내가 짠 코드 리뷰
```

구현은 직접 하되, 계획 수립과 코드 리뷰는 AI를 활용합니다.

### 리뷰어 — 코드 점검

```
/riskzero-si-review src/pages/example/ # 표준+TDD 증거 리뷰
/riskzero-si-pr-review                 # PR 안전성 리뷰
```

두 가지 관점(표준 준수 + 안전성)으로 코드를 리뷰합니다.

### QA 담당 — 테스트 문서 + 검증

```
/riskzero-si-qa-checklist 안전보건자료실   # 체크리스트 생성
/riskzero-si-browse                            # 브라우저 테스트 (URL은 si-config에서 자동 참조)
```

체크리스트를 자동 생성하고, 브라우저에서 검증합니다.

### 팀리드 — 리뷰 이후만 실행

```
/riskzero-si-pipeline 안전보건자료실 --from=4
```

개발자가 구현한 코드를 리뷰(Step 4)부터 최종 검증(Step 8)까지 실행합니다.

---

## 9. si-config.yml 설정 가이드

### 파일 위치

권장 위치:
- Codex: `.codex/si-config.yml`
- Claude Code: `.claude/si-config.yml`
- 공용 fallback: `si-config.yml`

### 주요 설정 항목

#### 프로젝트 기본 정보

```yaml
project:
  name: "MyProject"                 # 프로젝트명
  readme: "README.md"              # 코딩 규칙 문서 경로
```

#### 입력 소스 경로

```yaml
sources:
  wireframe: "docs/wireframe/"     # 기획서 PPTX 디렉토리
  publishing: "publishing/"        # 퍼블리싱 HTML/TSX 디렉토리
  ddl: "db/ddl/"                   # DDL SQL 파일 디렉토리
  sample: "src/pages/sample/"      # 샘플 코드 경로 (선택)
```

#### 서버 설정

```yaml
server:
  frontend:
    port: 3000                              # 프로젝트에 맞게 변경
    baseUrl: "http://localhost:3000"         # 프로젝트에 맞게 변경
  backend:
    port: 8080                              # 프로젝트에 맞게 변경
    baseUrl: "http://localhost:8080"         # 프로젝트에 맞게 변경
```

> 포트와 URL은 프로젝트마다 다릅니다. 반드시 실제 개발 서버 설정에 맞춰 변경하세요.

#### 인증 설정

```yaml
auth:
  type: "jwt"
  loginApi: "/api/auth/login"
  loginFields:
    username: "loginId"
    password: "password"
  tokenField: "accessToken"
  tokenHeader: "Authorization"
  tokenPrefix: "Bearer "
```

#### 프론트엔드 설정

```yaml
frontend:
  root: "my-frontend"              # 프론트엔드 루트 디렉토리
  framework: "react"               # react | vue | angular | next
  buildCmd: "npm run build"
  lintCmd: "npm run lint"
  routes:
    main: "src/_routes.tsx"         # 메인 라우트 파일
  pages:
    root: "src/pages"
    listFile: "index.tsx"
    detailPattern: "components/*Detail.tsx"
    apiPattern: "*Api.ts"
```

#### 백엔드 설정

```yaml
backend:
  root: "my-backend"               # 백엔드 루트 디렉토리
  framework: "spring-boot"         # spring-boot | express | nestjs | django
  language: "java"
  buildCmd: "./gradlew compileJava"
  testCmd: "./gradlew test"
  sourceRoot: "src/main/java"
  basePackage: "com.example.project"
  structure:                        # 파일 패턴 (변수: {domain})
    controller: "api/**/{domain}/controller/*Controller.java"
    service: "api/**/{domain}/service/*ServiceImpl.java"
    mapper: "api/**/{domain}/mapper/*Mapper.java"
    dto: "api/**/{domain}/model/dto/*Dto.java"
    vo: "api/**/{domain}/model/vo/*VO.java"
    mapperXml: "src/main/resources/mapper/**/{domain}/*Mapper.xml"
```

#### 데이터베이스 설정

```yaml
database:
  type: "postgresql"               # postgresql | mysql | oracle | mssql
  ddlPaths:
    - "db/ddl/postgres-ddl.sql"
```

#### 리서치/TDD/오케스트레이션 설정

```yaml
planning:
  researchDefault: "auto"                 # auto | always | never
  externalResearchDefault: "auto"         # auto | always | never
  externalResearchSourcePolicy: "official-first" # official-first | internal-only
  requireResearchCitations: true
  askWhenResearchExists: true

testing:
  tddRequired: true
  regressionAfterAllowed: true
  evidenceRequired: true

orchestration:
  mode: "inline"                          # inline | subagents | dynamic-workflow
  requireUserGate: true
  allowParallelReadOnly: true
  allowParallelWrites: false
  modelRouting: "auto"
  dynamicWorkflowDefault: "ask"           # ask | never | allowed
  writePolicy:
    subagentDefault: "read-only"
    requireIsolatedRootsForParallelWrites: true
    sharedFilesRequireInlineOwner: true
```

> `orchestration.*`은 권장 실행 전략입니다. Codex/Claude Code 런타임이 해당 기능이나 모델 지정을 지원하지 않으면 현재 에이전트가 inline으로 진행합니다.
> subagent나 Dynamic Workflow는 단계 내부 보조자이며, 다음 단계 진행 여부와 최종 PASS/FAIL 판정은 현재 파이프라인 오너가 결정합니다.

---

## 10. 트러블슈팅

### 빌드 실패가 반복될 때

**증상**: Step 3(구현)에서 빌드 에러가 3회 반복되어 중단됨

**대처**:
1. 에러 메시지를 확인 — 대부분 import 누락이나 타입 불일치
2. `--be-only` 또는 `--fe-only`로 한쪽만 먼저 완성
3. 그래도 실패하면 계획서(`.si-planning/{기능명}/implementation-plan.md`)의 데이터 모델을 수정 후 재구현

```
/riskzero-si-impl 안전보건자료실 --be-only    # 백엔드만 먼저
/riskzero-si-impl 안전보건자료실 --fe-only    # 이후 프론트엔드
```

### 생성된 코드가 프로젝트 스타일과 다를 때

**원인**: README.md에 코딩 규칙이 충분히 기술되지 않음

**대처**:
1. 프로젝트 README.md에 **코딩 스타일 가이드** 섹션을 보강
2. `si-config.yml`의 `sources.sample` 경로에 **잘 작성된 기존 코드**를 지정
3. 재실행하면 AI가 README + 샘플 코드를 학습하여 스타일이 개선됨

### Step 6~8에서 브라우저 테스트가 실행되지 않을 때

**증상**: "페이지에 접근할 수 없습니다" 또는 타임아웃 발생

**체크리스트**:
- [ ] 프론트엔드 개발 서버가 실행 중인가?
- [ ] 백엔드 개발 서버가 실행 중인가?
- [ ] `si-config.yml`의 `server.frontend.baseUrl`과 `server.backend.baseUrl`이 실제 서버 주소와 일치하는가?
- [ ] 방화벽이나 프록시가 localhost 접근을 차단하고 있지 않은가?

### 로그인이 실패할 때

**증상**: QA 테스트 시 로그인 단계에서 멈춤

**체크리스트**:
- [ ] `si-config.yml`의 `auth.loginApi` 경로가 정확한가?
- [ ] `auth.loginFields.username`과 `auth.loginFields.password` 필드명이 실제 API와 일치하는가?
- [ ] `auth.tokenField` 경로가 로그인 응답 JSON 구조와 맞는가?
- [ ] 테스트 계정이 유효하고 비밀번호가 만료되지 않았는가?

### 계획 리뷰에서 FAIL 판정이 반복될 때

**원인**: 프로젝트 README.md의 규칙과 계획서 사이의 불일치

**대처**:
1. `.si-planning/{기능명}/plan-review.md`에서 CRITICAL 이슈 목록 확인
2. 대부분은 패키지 구조, 네이밍 규칙, 검증 레이어 누락
3. README.md의 해당 규칙을 계획서에 반영하고 `/riskzero-si-plan`을 재실행

### si-config.yml 설정을 잘 모르겠을 때

**대처**: `--init`으로 자동 감지를 다시 실행하세요.

```
/riskzero-si-pipeline --init
```

AI가 프로젝트 구조를 분석하여 초안을 생성합니다. 이후 필요한 부분만 수정하면 됩니다.

### Codex 또는 Claude Code 세션 컨텍스트가 길어져서 응답이 느려질 때

**원인**: 파이프라인 전체(8단계)를 한 세션에서 실행하면 컨텍스트가 매우 길어짐

**대처**:
- `--from=N --to=M`으로 구간을 나눠서 실행 (권장: 1~4, 5~6, 7~8)
- 각 구간 시작 전 새 세션을 열면 응답 속도가 개선됨

---

## 11. FAQ

### Q: 설치 시 "gstack이 설치되어 있지 않습니다" 오류

gstack을 먼저 설치해야 합니다:
```bash
git clone https://github.com/garrytan/gstack.git ~/gstack
cd ~/gstack && ./setup --host codex
```

Claude Code를 쓰면 마지막 명령만 `./setup --host claude`로 바꾸면 됩니다.

### Q: Codex 또는 Claude Code에서 `/riskzero-si-pipeline`이 인식되지 않음

새 세션을 여세요. 스킬은 세션 시작 시 로드됩니다.

### Q: si-config.yml이 없다는 오류

프로젝트 디렉토리에서 `--init`으로 자동 생성하세요:
```
/riskzero-si-pipeline --init
```

### Q: 특정 단계만 다시 실행하고 싶음

`--from`과 `--to` 옵션을 사용하세요:
```
/riskzero-si-pipeline 안전보건자료실 --from=4 --to=4
```

### Q: 파이프라인 없이 개별 스킬만 사용 가능한가?

네. 각 스킬은 독립적으로 실행할 수 있습니다:
```
/riskzero-si-plan 안전보건자료실
/riskzero-si-review
/riskzero-si-qa-checklist 안전보건자료실
```

### Q: 다른 프레임워크(Vue, NestJS 등)에서도 사용 가능한가?

네. si-config.yml의 `frontend.framework`과 `backend.framework`를 변경하면 해당 프레임워크에 맞는 코드가 생성됩니다.

지원 프레임워크:
- **프론트엔드**: React, Vue, Angular, Next.js
- **백엔드**: Spring Boot, Express, NestJS, Django

### Q: 업데이트는 어떻게 하나?

```bash
cd ~/riskzero-si && git pull
./setup --host codex
```

Claude Code를 쓰면 마지막 명령만 `./setup --host claude`로 바꾸면 됩니다.

---

## 명령어 요약 카드

```
┌─────────────────────────────────────────────────────────────┐
│                    riskzero-si 명령어 요약                     │
├──────────────────────────────────┬──────────────────────────┤
│ /riskzero-si-pipeline {기능명}    │ 8단계 전체 실행            │
│ /riskzero-si-pipeline --init     │ 프로젝트 초기 설정          │
│ /riskzero-si-pipeline --from=N   │ N단계부터 실행             │
├──────────────────────────────────┼──────────────────────────┤
│ /riskzero-si-plan {기능명}        │ Step 1: 논의+리서치+계획   │
│ /riskzero-si-plan-review         │ Step 2: 계획+TDD 리뷰      │
│ /riskzero-si-impl {기능명}        │ Step 3: TDD 구현           │
│ /riskzero-si-review [경로]        │ Step 4: 표준+TDD 리뷰      │
│ /riskzero-si-pr-review           │ Step 5: PR 리뷰           │
│ /riskzero-si-qa-checklist {기능명} │ Step 6: QA 체크리스트      │
│ /riskzero-si-qa [URL]            │ Step 7: 버그 수정          │
│ /riskzero-si-browse [URL]        │ Step 8: 최종 검증          │
└──────────────────────────────────┴──────────────────────────┘
```
