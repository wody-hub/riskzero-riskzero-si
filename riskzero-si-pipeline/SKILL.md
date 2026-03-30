---
name: riskzero-si-pipeline
version: 1.0.0
description: |
  SI 개발 파이프라인 오케스트레이터. 8단계 순차 실행.
  기획서+퍼블리싱+DDL → 구현계획 → 리뷰 → 구현 → 코드리뷰 → PR리뷰 → QA → 버그수정 → 최종검증.
  /riskzero-si-pipeline {기능명} [--from=N] [--to=N] [--init]
  Use when asked to "SI 파이프라인", "기능 개발", "riskzero pipeline".
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
- 모든 산출물은 `plan/{기능명}/` 디렉토리에 저장한다
- 사용자 확인 없이 다음 단계로 넘어가지 않는다

---

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
| 1 | `/riskzero-si-plan {기능명}` | 구현 계획 수립 | `plan/{기능명}/implementation-plan.md` |
| 2 | `/riskzero-si-plan-review` | 계획 리뷰 (아키텍처/보안) | `plan/{기능명}/plan-review.md` |
| 3 | `/riskzero-si-impl {기능명}` | FE/BE 코드 구현 | 실제 소스 파일들 |
| 4 | `/riskzero-si-review` | 프로젝트 표준 준수 리뷰 | `plan/{기능명}/code-review.md` |
| 5 | `/riskzero-si-pr-review` | PR diff 안전성 리뷰 | `plan/{기능명}/pr-review.md` |
| 6 | `/riskzero-si-qa-checklist {기능명}` | QA 체크리스트 생성 | `plan/{기능명}/qa-checklist.md` |
| 7 | `/riskzero-si-qa` | 버그 조사 및 수정 | 수정된 소스 파일들 |
| 8 | `/riskzero-si-browse` | 브라우저 최종 검증 | `plan/{기능명}/final-report.md` |

---

## --from, --to 옵션

파이프라인의 특정 구간만 실행할 수 있다.

```bash
# 3단계(구현)부터 실행
/riskzero-si-pipeline {기능명} --from=3

# 1~4단계만 실행 (계획 수립 → 리뷰 → 구현 → 코드 리뷰)
/riskzero-si-pipeline {기능명} --from=1 --to=4

# 6단계(QA)부터 끝까지
/riskzero-si-pipeline {기능명} --from=6

# 7단계만 실행 (버그 수정)
/riskzero-si-pipeline {기능명} --from=7 --to=7
```

**주의사항:**
- `--from`으로 중간 단계부터 시작할 때, 이전 단계의 산출물(`plan/{기능명}/`)이 존재하는지 확인한다
- 산출물이 없으면 이전 단계부터 실행할 것을 권고한다
- `--to`를 지정하면 해당 단계 완료 후 파이프라인을 중단한다

---

## 단계별 실행 절차

### 1단계: 구현 계획 수립 (`/riskzero-si-plan`)

**수행 내용:**
- si-config.yml에서 wireframe, publishing, ddl 경로를 읽는다
- 기획서(와이어프레임)를 분석하여 화면 구성, 필드, 동작을 파악한다
- 퍼블리싱 HTML/CSS를 분석하여 UI 컴포넌트 구조를 파악한다
- DDL을 분석하여 테이블 구조, 컬럼, 관계를 파악한다
- 샘플 코드를 참고하여 프로젝트 패턴에 맞는 구현 계획을 수립한다
- API 설계 (엔드포인트, 요청/응답 스펙)를 작성한다
- 프론트엔드 페이지 구조 및 컴포넌트 설계를 작성한다
- 백엔드 클래스 설계 (Controller, Service, Mapper, DTO, VO)를 작성한다

**산출물:** `plan/{기능명}/implementation-plan.md`

**진행 조건:** 사용자가 계획을 확인하고 승인한다.
**중단 조건:** 기획서, DDL 등 필수 입력 자료가 누락된 경우 사용자에게 경로를 확인 요청한다.

---

### 2단계: 계획 리뷰 (`/riskzero-si-plan-review`)

**수행 내용:**
- 1단계에서 생성된 `implementation-plan.md`를 아키텍처/보안 관점에서 리뷰한다
- 프로젝트 README.md의 규칙이 계획에 반영되었는지 확인한다
- API 네이밍 규칙 준수 여부를 확인한다
- DTO/VO 분리 규칙 준수 여부를 확인한다
- 보안(권한 체크) 설계 포함 여부를 확인한다
- 유효성 검증 레이어 설계 포함 여부를 확인한다

**산출물:** `plan/{기능명}/plan-review.md`

**진행 조건:** 리뷰 결과가 PASS이거나, 지적 사항을 1단계 계획에 반영 완료 후 재리뷰 통과.
**중단 조건:** CRITICAL 이슈가 있으면 1단계로 되돌아가 계획을 수정한다.

---

### 3단계: 코드 구현 (`/riskzero-si-impl`)

**수행 내용:**
- 승인된 `implementation-plan.md`를 기반으로 실제 코드를 생성한다
- si-config.yml의 structure 패턴에 따라 파일을 생성한다
- 백엔드: Controller, Service, Mapper(interface), Mapper(XML), DTO, VO 파일 생성
- 프론트엔드: 목록 페이지, 상세 페이지, API 훅 파일 생성
- 라우터에 새 페이지 경로를 등록한다
- 샘플 코드의 패턴(임포트, 코드 스타일, 구조)을 정확히 따른다
- 템플릿 마커 주석(`// ---`)을 최종 코드에서 제거한다

**산출물:** 실제 소스 파일들 (프로젝트 디렉토리에 직접 생성)

**진행 조건:** 빌드(buildCmd) 성공, 린트(lintCmd) 통과.
**중단 조건:** 빌드 또는 린트 실패 시 오류를 수정한 후 재실행. 3회 이상 실패 시 사용자에게 보고한다.

---

### 4단계: 프로젝트 표준 리뷰 (`/riskzero-si-review`)

**수행 내용:**
- 3단계에서 생성/수정된 모든 파일을 대상으로 프로젝트 표준 준수 여부를 검사한다
- README.md 및 si-config.yml에 정의된 규칙 기반으로 리뷰한다
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

**산출물:** `plan/{기능명}/code-review.md`

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

**산출물:** `plan/{기능명}/pr-review.md`

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

**산출물:** `plan/{기능명}/qa-checklist.md`

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

**산출물:** `plan/{기능명}/final-report.md`

**진행 조건:** 전 항목 검증 완료.
**중단 조건:** CRITICAL 이슈 발견 시 7단계로 되돌아간다.

---

## 실행 흐름 요약

```
[시작]
  │
  ▼
1. 구현 계획 수립 ──→ plan/{기능명}/implementation-plan.md
  │
  ▼
2. 계획 리뷰 ──────→ plan/{기능명}/plan-review.md
  │                    ↑ CRITICAL 시 1단계로 복귀
  ▼
3. 코드 구현 ──────→ 실제 소스 파일
  │                    ↑ 빌드 실패 시 재시도
  ▼
4. 표준 리뷰 ──────→ plan/{기능명}/code-review.md
  │                    ↑ ERROR 시 3단계 코드 수정
  ▼
5. PR 리뷰 ────────→ plan/{기능명}/pr-review.md
  │                    ↑ BLOCKER 시 코드 수정
  ▼
6. QA 체크리스트 ──→ plan/{기능명}/qa-checklist.md
  │
  ▼
7. 버그 수정 ──────→ 수정된 소스 파일
  │                    ↑ FAIL 항목 수정 반복
  ▼
8. 최종 검증 ──────→ plan/{기능명}/final-report.md
  │                    ↑ CRITICAL 시 7단계로 복귀
  ▼
[완료]
```

---

## 출력 디렉토리 구조

모든 산출물은 프로젝트 루트의 `plan/{기능명}/` 디렉토리에 저장된다.

```
plan/
  └── {기능명}/
      ├── implementation-plan.md   # 1단계: 구현 계획서
      ├── plan-review.md           # 2단계: 계획 리뷰 결과
      ├── code-review.md           # 4단계: 코드 리뷰 결과
      ├── pr-review.md             # 5단계: PR 리뷰 결과
      ├── qa-checklist.md          # 6단계: QA 체크리스트
      └── final-report.md          # 8단계: 최종 검증 보고서
```

---

## 오케스트레이터 실행 로직

파이프라인 실행 시 오케스트레이터는 다음 순서로 동작한다:

1. **인자 파싱**: 기능명, --from, --to, --init 옵션을 파싱한다
2. **--init 처리**: --init 플래그가 있으면 초기화 모드로 진입한다
3. **설정 파일 로드**: si-config.yml을 탐색하고 로드한다
4. **산출물 디렉토리 생성**: `plan/{기능명}/` 디렉토리를 생성한다
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
