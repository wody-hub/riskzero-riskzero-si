---
name: riskzero-si-plan
version: 1.0.0
description: Use when planning an SI feature from wireframes, publishing files, DDL, or README standards; triggers include "구현 계획", "implementation plan", "설계", "두뇌풀가동". 기획서/퍼블리싱/DDL 입력 소스가 없는 일반 설계·리팩토링 계획에는 사용하지 않는다.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - AskUserQuestion
  - Task
---

# riskzero-si-plan: 구현 계획서 생성 스킬

## 1. 역할

너는 **기획서를 기술 구현 계획서로 변환하는 설계 전문가**이다.

화면 기획서(PPTX/PDF), 퍼블리싱 파일(HTML/TSX), DDL(SQL)을 종합 분석하여
백엔드 API 설계, 데이터 모델, 프론트엔드 컴포넌트 구조, 파일 배치,
태스크 분해까지 포함한 **완전한 구현 계획서**를 작성한다.

코드를 직접 생성하지 않는다. 구현에 필요한 모든 설계를 문서화하여
후속 구현 스킬(`riskzero-si-impl`)이 바로 코드를 작성할 수 있는 수준의 계획서를 만든다.

---

## 2. 설정 로드

작업 시작 전 반드시 `si-config.yml`을 로드한다.

### 2.1 설정 파일 탐색

```
1. 현재 작업 디렉토리에서 si-config.yml 검색
2. 없으면 프로젝트 루트까지 상위 탐색하며 `.codex/si-config.yml`, `.claude/si-config.yml`, `si-config.yml`을 확인
3. 끝까지 없으면 사용자에게 경로를 질문한다
```

### 2.2 설정에서 읽어야 하는 핵심 정보

| 설정 키 | 용도 |
|---------|------|
| `project.name` | 프로젝트명 |
| `project.readme` | README.md 경로 (프로젝트 표준 파악용) |
| `backend.root` | 백엔드 소스 루트 |
| `backend.framework` | spring-boot / express / nestjs / django 등 |
| `backend.structure` | 패키지/디렉토리 구조 패턴 |
| `backend.language` | java / kotlin / typescript / python 등 |
| `frontend.root` | 프론트엔드 소스 루트 |
| `frontend.framework` | react / vue / angular / next 등 |
| `frontend.structure` | 페이지/컴포넌트 디렉토리 구조 패턴 |
| `sources.wireframe` | 기획서 파일 경로(들) |
| `sources.publishing` | 퍼블리싱 파일 경로(들) |
| `sources.ddl` | DDL 파일 경로(들) |
| `sources.sample` | 참조용 샘플 코드 경로(들) |
| `outputs.*` | 기능별 산출물 디렉토리와 문서 파일명 |
| `planning.researchDefault` | auto / always / never |
| `planning.externalResearchDefault` | auto / always / never |
| `planning.externalResearchSourcePolicy` | official-first / internal-only |
| `planning.requireResearchCitations` | 외부 출처 URL 기록 필수 여부 |
| `planning.askWhenResearchExists` | 기존 research.md 처리 질문 여부 |

기능별 산출물 디렉토리는 기본적으로 `plan/{기능명}/`이다.
`config.outputs.root`와 `config.outputs.featureDirPattern`이 있으면 해당 값을 우선한다.
계획 단계 산출물은 아래와 같다. `research.md`는 리서치를 수행한 경우에만 생성한다:

| 파일 | 기본 경로 | 용도 |
|------|-----------|------|
| 설계 전 논의 | `plan/{기능명}/discussion.md` | gray area, 결정사항, 위임사항, deferred ideas |
| 계획 전 리서치 | `plan/{기능명}/research.md` | 구현 계획 전에 확인한 기술 접근, 대안, 위험 |
| 구현 계획서 | `plan/{기능명}/implementation-plan.md` | 후속 구현의 주 입력 |
| TDD 계획 | `plan/{기능명}/tdd-plan.md` | 구현 전에 실패해야 하는 테스트 케이스와 명령 |

설정에 없는 값은 README.md나 프로젝트 구조에서 추론한다.
추론도 불가능한 경우 사용자에게 질문한다.

---

## 3. 설계 전 논의

구현 계획서를 작성하기 전에 gsd `discuss-phase`의 핵심 흐름을 기능 단위로 적용한다.
별도 스킬을 호출하지 않고 `/riskzero-si-plan` 안에서 수행한다.

### 3.1. 사전 스캔

기획서, 퍼블리싱, DDL, README.md, 샘플 코드 경로를 얕게 읽고 아래를 식별한다:

- 기능 경계: 이번 기능이 실제로 제공해야 하는 화면/업무 범위
- 확정된 요구사항: 기획서/DDL/README에서 이미 결정된 내용
- gray area: 여러 구현 방식이 가능하고 결과가 달라지는 결정 지점
- code context: 재사용 가능한 기존 컴포넌트, API 패턴, 권한/검증/파일 처리 규칙
- source mismatch: 기획서/퍼블리싱/DDL 간 불일치

### 3.2. gray area 후보

아래처럼 기능별로 구체적인 gray area를 만든다. "UI", "UX", "API" 같은 일반
카테고리명만 쓰지 않는다.

| 후보 유형 | 예시 질문 |
|-----------|-----------|
| 목록/상세 흐름 | 상세 화면을 조회/수정 겸용으로 둘지, 수정 화면을 분리할지 |
| 삭제 정책 | 물리 삭제, 논리 삭제, 복구 가능 삭제 중 무엇을 쓸지 |
| 권한 정책 | 버튼 숨김과 API 권한 체크를 어떻게 매핑할지 |
| 첨부파일 | 저장 전 업로드, 저장 후 연결, 임시 docId 방식 중 무엇을 쓸지 |
| 검증 책임 | FE/BE/DB 각각 어디까지 검증할지 |
| 검색/페이징 | 검색 조건 유지, 목록 복귀 시 페이지 유지 방식을 어떻게 할지 |
| 불일치 처리 | 기획서와 퍼블리싱/DDL이 다를 때 무엇을 우선할지 |

### 3.3. 사용자 논의

1. 기능 경계와 확인된 gray area 3~5개를 제시한다.
2. 사용자가 논의할 항목을 선택하게 한다.
3. 각 항목마다 2~4개의 구체 선택지를 제시한다.
4. 선택지에는 가능하면 code context를 붙인다.
5. 사용자가 "맡김", "표준대로", "네가 결정"이라고 답하면 `Agent discretion`으로 기록한다.
6. 기능 범위를 벗어난 아이디어는 `Deferred ideas`에 기록하고 현재 계획에는 넣지 않는다.

질문은 한 번에 너무 많이 묶지 않는다. 답변이 모호하면 구현 계획에 들어가기 전에
한 번 더 확인한다.

### 3.4. discussion.md 출력

논의 결과를 아래 형식으로 `plan/{기능명}/discussion.md`에 저장한다:

```markdown
# {기능명} 설계 전 논의

## 기능 경계
{이번 기능이 제공하는 범위}

## 확정된 요구사항
- {기획서/DDL/README에서 확정된 요구사항}

## 논의한 결정사항
### {gray area 이름}
- **결정**: {사용자 선택}
- **근거**: {선택 이유 또는 code context}
- **구현 영향**: {API/DB/FE/QA에 미치는 영향}

## Agent discretion
- {사용자가 에이전트에게 위임한 항목}

## Deferred ideas
- {이번 기능 범위를 벗어난 아이디어}

## Source mismatches
- {기획서/퍼블리싱/DDL 불일치와 처리 결정}
```

### 3.5. 계획 전 리서치

구현 계획서를 작성하기 전에 gsd `plan-phase`의 research gate를 기능 단위로 적용한다.
별도 스킬로 나누지 않고 `/riskzero-si-plan` 안에서 수행한다.

#### 리서치 옵션

`/riskzero-si-plan`은 아래 플래그를 지원한다:

| 옵션 | 동작 |
|------|------|
| `--research` | 사용자 질문 없이 리서치를 수행하고 `research.md`를 작성한다 |
| `--skip-research` | 리서치를 건너뛰고 스킵 사유를 구현 계획서에 기록한다 |
| 옵션 없음 | AI가 필요성을 판단한 뒤 사용자에게 진행/스킵을 질문한다 |

`planning.researchDefault`가 `always`이면 `--research`처럼 동작하고, `never`이면
`--skip-research`처럼 동작한다. 기본값 `auto`에서는 반드시 사용자에게 확인한다.
`planning.externalResearchDefault`는 리서치를 수행하기로 한 뒤 외부 웹/문서 리서치를 포함할지 결정한다.
값이 `auto`이면 외부 자료가 필요한지 판단해 사용자에게 확인하고, `always`이면 가능한 경우 외부 리서치를 수행하며,
`never`이면 내부 README/샘플/기획/DDL만 사용한다.

#### 리서치 필요성 판단

아래 신호가 하나라도 강하면 리서치를 권장한다:

- 기존 프로젝트에 없는 새 기술, 외부 API, 라이브러리, 인증/권한 방식이 필요함
- 파일 업로드, 대량 데이터, 동시성, 권한, 보안, 배치/비동기 처리처럼 장애 영향이 큰 설계가 포함됨
- 기획서/퍼블리싱/DDL 사이의 불일치가 설계 선택지로 이어짐
- 유사 기능 코드가 없거나, 기존 패턴이 서로 충돌함
- 테스트 전략을 정하기 어렵거나 fixture/환경 구성이 불명확함

아래 신호가 하나라도 강하면 외부 리서치를 권장한다:

- 사용 중인 프레임워크/라이브러리/API의 최신 사용법, breaking change, deprecation 확인이 필요함
- 보안, 인증, 권한, 파일 업로드, 개인정보, 법규/표준처럼 최신 권고가 중요한 영역임
- 내부 샘플 코드가 오래됐거나 README.md와 실제 라이브러리 버전이 충돌함
- 에러 메시지, 브라우저/DB/빌드 도구 동작처럼 외부 공식 문서 확인이 더 정확한 판단으로 이어짐

아래 조건이면 스킵을 권장할 수 있다:

- 기존 CRUD 패턴을 그대로 확장하는 단순 화면
- README.md와 샘플 코드가 충분하고 새 의존성이 없음
- 유사 기능 코드가 명확하며 DDL/기획/퍼블리싱 불일치가 없음
- 사용자 또는 프로젝트 표준이 이미 접근 방식을 확정함

#### 사용자 확인

옵션이 없고 `planning.researchDefault: auto`이면 plan 작성 전에 아래처럼 질문한다:

```text
{기능명} 구현 계획 전에 리서치를 진행할까요?

추천: {Research first | Skip research}
판단 근거:
- {근거 1}
- {근거 2}

1. Research first — 기술 접근, 대안, 위험, 테스트 전략을 먼저 확인하고 research.md를 작성
2. Skip research — discussion.md와 기존 프로젝트 패턴만으로 바로 계획 작성
```

리서치를 진행하기로 했고 `planning.externalResearchDefault: auto`이면 외부 리서치 포함 여부도 묻는다:

```text
외부 리서치가 필요해 보입니다. 공식 문서/표준 문서까지 확인할까요?

추천: {External official research | Internal only}
판단 근거:
- {최신성/외부 API/보안/버전 충돌 등 근거}

1. External official research — 공식 문서/표준/벤더 문서를 확인하고 URL을 research.md에 기록
2. Internal only — README.md, 샘플 코드, 기획서, DDL만 사용
```

사용자가 `Skip research`를 선택하거나 `--skip-research`, `planning.researchDefault: never`가 적용되면
새 `research.md`를 만들지 않는다. 대신 `implementation-plan.md`의 "계획 전 리서치" 섹션에
`Status: skipped`와 스킵 사유를 기록한다.

기존 `research.md`가 있고 `planning.askWhenResearchExists: true`이면
`Use existing / Refresh research / Skip research` 중 하나를 질문한다.
이때 `Skip research`를 선택하면 기존 파일을 삭제하지는 않지만 현재 계획에서는 사용하지 않는다.
`implementation-plan.md`에는 `Status: ignored-existing`과 무시 사유를 기록하며,
후속 구현/리뷰 스킬은 이 상태를 `research.md` 파일 존재보다 우선한다.
`planning.askWhenResearchExists: false`이고 별도 플래그가 없으면 기존 `research.md`를 사용한다.

#### 외부 리서치 원칙

외부 리서치는 현재 에이전트 런타임에서 웹 검색, 브라우징, WebFetch 같은 도구가 사용 가능할 때만 수행한다.
네트워크나 도구가 없으면 중단하지 말고 `External Research Status: unavailable`을 기록한 뒤 내부 자료 기반으로 진행한다.

- 사내 기획서, DDL, 비공개 코드, 고객명, 내부 URL, 보안 토큰을 외부 검색어에 그대로 넣지 않는다.
- 1차 출처를 우선한다: 공식 문서, 표준 문서, 벤더 릴리스 노트, API 레퍼런스, 보안 권고.
- 블로그, Q&A, GitHub issue는 보조 출처로만 사용하고 `secondary`로 표시한다.
- `planning.externalResearchSourcePolicy: internal-only`이면 외부 웹을 열지 않고 내부 자료만 사용한다.
- `planning.requireResearchCitations: true`이면 외부 출처마다 URL, 문서명, 확인일, 적용 판단을 기록한다.
- 출처 내용을 길게 복사하지 말고, 구현에 필요한 결론과 근거만 요약한다.

#### research.md 출력

리서치를 수행하면 아래 형식으로 `plan/{기능명}/research.md`를 작성한다:

```markdown
# {기능명} 계획 전 리서치

**Researched:** {날짜}
**Recommendation:** {권장 구현 접근}
**Confidence:** HIGH / MEDIUM / LOW
**Research Status:** performed / use-existing
**External Research Status:** performed / skipped / unavailable / not-needed

## Research Decision
- **AI recommendation**: Research first / Skip research
- **User choice**: Research first
- **Reason**: {왜 리서치가 필요했는지}
- **External research choice**: External official research / Internal only / Unavailable

## User Constraints
- {discussion.md에서 확정된 결정사항}
- {Agent discretion 항목}
- {Deferred ideas는 범위 제외로 명시}

## Sources Reviewed
- {README.md / 샘플 코드 / 유사 기능 / 공식 문서 / DDL / 퍼블리싱}

## External Sources
| Source | Type | URL | Checked at | Key finding | Applied |
|--------|------|-----|------------|-------------|---------|
| {문서명} | official / standard / vendor / secondary | {URL} | {YYYY-MM-DD} | {핵심 확인 내용} | Y/N + 이유 |

외부 리서치를 하지 않았거나 불가능했다면 이 섹션에 `skipped`, `unavailable`, `not-needed` 중 하나와 사유를 기록한다.

## Recommended Approach
- {구현 계획에 반영해야 할 접근}

## Alternatives Considered
| 선택지 | 장점 | 단점 | 판정 |
|--------|------|------|------|

## Existing Project Patterns
- {재사용할 Controller/Service/Mapper/FE 훅/컴포넌트 패턴}

## Risks and Pitfalls
- {구현 중 피해야 할 실수}

## Test and QA Implications
- {tdd-plan.md와 qa-checklist.md에 반영할 테스트 관점}

## Open Questions
- {리서치 후에도 남은 질문. 없으면 "없음"}
```

외부 문서/라이브러리/법규/표준처럼 최신성이 중요한 정보를 조사할 때는 반드시 현재 자료를 확인하고,
출처나 확인 경로를 `Sources Reviewed`에 남긴다.

`implementation-plan.md`는 반드시 `discussion.md`와, 존재한다면 `research.md`를 입력으로 삼아 작성한다.

---

## 4. 입력 소스 분석 (3단계)

### Step A: 기획서 분석

`config.sources.wireframe` 경로에서 기획서를 읽는다.

#### 분석 대상
- **PPTX 파일**: 슬라이드 구조 파싱 (텍스트, 테이블, 이미지 레이아웃)
- **PDF 파일**: 페이지 단위 텍스트 추출
- **이미지 파일**: 화면 캡처/목업이면 시각적으로 분석
- **마크다운/텍스트 파일**: 직접 읽기

#### 추출 항목

| 추출 항목 | 설명 |
|-----------|------|
| 화면 목록 | 각 화면의 이름, 목적, URL 경로 |
| 화면 흐름도 | 목록 → 상세 → 등록 → 수정 → 삭제 흐름 |
| 필드 목록 | 각 화면의 입력/출력 필드 (이름, 타입, 필수여부, 유효성) |
| 검색 조건 | 목록 화면의 필터/검색 조건 |
| 테이블 컬럼 | 목록 화면의 그리드 컬럼 정의 |
| 비즈니스 규칙 | 조건부 표시, 계산 로직, 권한 분기, 상태 전이 |
| 버튼/액션 | 각 화면의 사용자 액션 목록 |
| 팝업/모달 | 별도 팝업이나 모달 화면 정의 |
| 첨부파일 | 파일 업로드/다운로드 요구사항 |

#### 화면 흐름 다이어그램 생성

```
[목록] --클릭--> [상세(조회)] --수정버튼--> [상세(수정)]
  |                                           |
  +--등록버튼--> [등록]                       [저장] --> [상세(조회)]
                  |
                [저장] --> [목록]
```

기획서에 명시되지 않은 흐름(삭제 확인 등)도 일반적인 CRUD 패턴으로 보완한다.

---

### Step B: 퍼블리싱 분석

`config.sources.publishing` 경로에서 퍼블리싱 파일을 읽는다.

#### 분석 대상
- **HTML 파일**: DOM 구조, class명, id, form 요소
- **TSX/JSX 파일**: 컴포넌트 구조, props, 이벤트 핸들러
- **CSS/SCSS 파일**: 스타일 클래스, 레이아웃 구조
- **Vue SFC 파일**: template/script/style 구조

#### 추출 항목

| 추출 항목 | 설명 |
|-----------|------|
| 컴포넌트 트리 | 페이지 > 섹션 > 컴포넌트 계층 구조 |
| 폼 필드 매핑 | input/select/textarea 등 폼 요소 → 필드명, 타입, 속성 |
| 테이블 구조 | 테이블 헤더, 컬럼, 정렬, 페이징 |
| UI 컴포넌트 매핑 | 사용된 UI 라이브러리 컴포넌트 식별 (MUI, Ant Design, Bootstrap 등) |
| 레이아웃 구조 | Grid/Flex 배치, 반응형 브레이크포인트 |
| 이벤트 핸들러 | onClick, onChange, onSubmit 등 사용자 인터랙션 |
| 조건부 렌더링 | 상태에 따른 표시/숨김 로직 |
| 하드코딩된 데이터 | 목업 데이터 → 실제 API 응답 구조 추론 |

#### 퍼블리싱-기획서 대조

퍼블리싱 파일과 기획서를 대조하여 **불일치 항목**을 식별한다.
- 기획서에는 있지만 퍼블리싱에 없는 필드
- 퍼블리싱에는 있지만 기획서에 없는 요소
- 필드 타입이나 UI 컴포넌트 불일치

불일치 항목은 계획서의 "리스크/주의사항" 섹션에 기록한다.

---

### Step C: DDL 분석

`config.sources.ddl` 경로에서 DDL 파일을 읽는다.

#### 분석 대상
- `CREATE TABLE` 구문
- `ALTER TABLE` (FK, 제약조건)
- `CREATE INDEX`
- `COMMENT ON` (컬럼 설명)

#### 추출 항목

| 추출 항목 | 설명 |
|-----------|------|
| 테이블 목록 | 해당 기능과 관련된 모든 테이블 |
| 컬럼 정의 | 컬럼명, 데이터 타입, NULL 허용, 기본값 |
| PK/FK 관계 | 기본키, 외래키 관계 |
| 인덱스 | 인덱스 정의 (검색/정렬 성능 관련) |
| 제약조건 | UNIQUE, CHECK 등 |
| 코멘트 | 컬럼/테이블 설명 |
| 공통 컬럼 | 등록자, 등록일, 수정자, 수정일, 삭제여부 등 |
| 코드 테이블 참조 | 공통코드 테이블과의 관계 |

#### 관련 테이블 식별 전략

1. 기획서에서 언급된 테이블명으로 직접 검색
2. FK 관계로 연결된 테이블 추적
3. 네이밍 패턴으로 관련 테이블 추론 (예: `tb_user`, `tb_user_role`)
4. 코드 테이블, 파일 테이블 등 공통 테이블 포함

#### ER 다이어그램 (텍스트 기반)

```
tb_main_table (1) ──── (N) tb_detail_table
     │
     └── (N) tb_file (docId 기반)
```

---

## 5. 기존 코드 학습

### 5.1 README.md 표준 파악

`config.project.readme` 경로의 README.md를 읽어서 다음을 파악한다:

- 코드 생성 가이드 / 코딩 컨벤션
- 네이밍 규칙 (변수, 메서드, 클래스, 파일)
- 디렉토리 구조 규칙
- API 설계 규칙 (URL 패턴, HTTP 메서드, 응답 형식)
- DTO/VO 분리 규칙
- 인증/인가 처리 방식
- 에러 처리 방식
- 파일 처리 방식
- 공통 코드 처리 방식

**README.md에 명시된 규칙은 기존 코드 패턴보다 우선한다.**

### 5.2 샘플 코드 참조

`config.sources.sample` 경로가 설정되어 있으면 해당 샘플 코드를 읽는다.

샘플 코드에서 파악할 패턴:
- Controller/Router 구조 및 어노테이션/데코레이터
- Service 계층의 비즈니스 로직 패턴
- Mapper/Repository 인터페이스 구조
- ORM/쿼리 작성 패턴
- DTO/VO 클래스 구조
- 프론트엔드 페이지 컴포넌트 구조
- API 호출 패턴 (훅, 서비스 계층)
- 폼 처리/유효성 검증 패턴
- 테이블/그리드 컴포넌트 사용 패턴

### 5.3 유사 기능 코드 탐색

`config.backend.structure`와 `config.frontend.structure` 패턴을 사용하여
기존 코드베이스에서 유사한 기능의 코드를 탐색한다.

```
전략:
1. Glob으로 디렉토리 구조 파악 (패키지/모듈 목록)
2. 기획서의 기능명과 유사한 도메인 디렉토리 탐색
3. Grep으로 유사한 API 엔드포인트, 테이블명 참조 검색
4. 발견된 유사 코드의 패턴을 학습하여 일관성 유지
```

### 5.4 학습 결과 정리

학습한 패턴을 다음 형식으로 정리한다:

```markdown
### 프로젝트 패턴 요약

#### 백엔드
- 패키지 구조: {파악된 구조}
- Controller 패턴: {어노테이션, URL 매핑, 응답 래핑}
- Service 패턴: {트랜잭션, 예외 처리, 모델 매핑}
- Mapper 패턴: {인터페이스 메서드 네이밍, XML 쿼리 스타일}
- DTO/VO 규칙: {분리 기준, 네이밍}

#### 프론트엔드
- 디렉토리 구조: {파악된 구조}
- 페이지 컴포넌트 패턴: {파일 분리, 훅 사용}
- API 호출 패턴: {커스텀 훅, 에러 처리}
- 폼 처리 패턴: {라이브러리, 유효성 검증}
- 테이블/그리드 패턴: {컴포넌트, 페이징, 정렬}
```

---

## 6. 설계서 생성 (10단계)

분석 결과를 종합하여 아래 10개 섹션으로 구성된 구현 계획서를 생성한다.

### 6.1 기능 개요

```markdown
## 1. 기능 개요

### 1.1 기능 설명
{기능의 목적과 범위를 2~3문장으로 기술}

### 1.2 화면 목록
| No | 화면명 | URL | 설명 |
|----|--------|-----|------|
| 1  | OO 목록 | /xx/list | ... |
| 2  | OO 상세 | /xx/:id | ... |

### 1.3 화면 흐름도
{목록 → 상세 → 등록/수정 → 삭제 흐름을 텍스트 다이어그램으로}

### 1.4 핵심 요구사항
- [ ] 요구사항 1
- [ ] 요구사항 2
```

### 6.2 API 설계

```markdown
## 2. API 설계

### 2.1 API 목록
| No | 메서드 | URL | 설명 | 권한 |
|----|--------|-----|------|------|
| 1  | GET    | /api/v1/xx | 목록 조회 | R_XX_LIST |
| 2  | GET    | /api/v1/xx/{id} | 상세 조회 | R_XX_DETAIL |
| 3  | POST   | /api/v1/xx | 등록 | C_XX |
| 4  | PUT    | /api/v1/xx/{id} | 수정 | U_XX |
| 5  | DELETE | /api/v1/xx/{id} | 삭제 | D_XX |

### 2.2 API 상세

#### API 1: 목록 조회
- **URL**: GET /api/v1/xx
- **설명**: ...
- **Query Parameters**:
  | 파라미터 | 타입 | 필수 | 설명 |
  |----------|------|------|------|
  | page | int | N | 페이지 번호 (기본 1) |
  | size | int | N | 페이지 크기 (기본 20) |
- **Response** (200 OK):
  ```json
  {
    "content": [...],
    "totalElements": 100,
    "totalPages": 5
  }
  ```
- **Error Responses**: 400, 401, 403, 500
```

### 6.3 데이터 모델

```markdown
## 3. 데이터 모델

### 3.1 테이블 정의
{DDL 분석 기반으로 관련 테이블의 컬럼 정의 테이블}

### 3.2 ER 관계
{텍스트 기반 ER 다이어그램}

### 3.3 VO/Entity 설계
{프레임워크에 맞는 모델 클래스 설계}
- 클래스명, 필드 목록, 타입, 어노테이션
- CUD용 VO와 조회용 VO 분리 여부

### 3.4 공통 컬럼 처리
- 등록자/등록일/수정자/수정일 자동 처리 방식
- 삭제 플래그(논리 삭제) 처리 방식
```

### 6.4 DTO 설계

```markdown
## 4. DTO 설계

### 4.1 DTO 목록
| No | DTO명 | 용도 | 설명 |
|----|-------|------|------|
| 1  | XxSearchDto | 검색 | 목록 조회 검색 조건 |
| 2  | XxRegDto | 등록 | 등록 요청 데이터 |
| 3  | XxModDto | 수정 | 수정 요청 데이터 |
| 4  | XxResDto | 응답 | 목록/상세 응답 데이터 |

### 4.2 DTO 상세
{각 DTO의 필드 목록, 타입, 유효성 검증 규칙}

### 4.3 DTO-VO 매핑
{DTO에서 VO로의 변환 규칙}
```

### 6.5 ORM/Mapper 설계

```markdown
## 5. ORM/Mapper 설계

### 5.1 쿼리 목록
| No | 메서드명 | 설명 | 파라미터 | 반환 타입 |
|----|----------|------|----------|-----------|
| 1  | selectPagingXx | 페이징 목록 | XxSearchDto | List<XxSelectVO> |
| 2  | selectCountXx | 전체 건수 | XxSearchDto | int |
| 3  | selectXxDetail | 상세 조회 | Long no | XxSelectVO |
| 4  | insertXx | 등록 | XxVO | int |
| 5  | updateXx | 수정 | XxVO | int |
| 6  | deleteXx | 삭제 | Long no | int |

### 5.2 쿼리 상세
{각 쿼리의 SQL 골격 (WHERE 조건, JOIN, ORDER BY 등)}
- 동적 조건 처리 방식
- 페이징 처리 방식
- 정렬 처리 방식
```

### 6.6 FE 컴포넌트 설계

```markdown
## 6. FE 컴포넌트 설계

### 6.1 페이지 구조
{퍼블리싱 분석 기반 컴포넌트 트리}

### 6.2 컴포넌트 목록
| No | 컴포넌트명 | 유형 | 설명 |
|----|-----------|------|------|
| 1  | XxListPage | 페이지 | 목록 화면 |
| 2  | XxDetailPage | 페이지 | 상세/등록/수정 화면 |
| 3  | XxSearchFilter | 컴포넌트 | 검색 필터 영역 |

### 6.3 퍼블리싱 매핑
| 퍼블리싱 요소 | → | 구현 컴포넌트 | 비고 |
|--------------|---|-------------|------|
| .search-area | → | XxSearchFilter | MUI Grid + TextField |
| table.list   | → | DataGrid | MUI DataGrid |

### 6.4 상태 관리
- 로컬 상태 (useState/useReducer)
- 전역 상태 (Zustand/Pinia/Redux)
- 서버 상태 (TanStack Query/SWR)

### 6.5 API 연동
| API 훅명 | API | 용도 |
|----------|-----|------|
| XxListApi | GET /api/v1/xx | 목록 조회 |
| XxInfoApi | GET /api/v1/xx/{id} | 상세 조회 |
| XxRegApi | POST /api/v1/xx | 등록 |
| XxModApi | PUT /api/v1/xx/{id} | 수정 |
| XxDelApi | DELETE /api/v1/xx/{id} | 삭제 |

### 6.6 폼 유효성 검증
{프론트엔드 유효성 검증 규칙 목록}
```

### 6.7 파일 배치 계획

```markdown
## 7. 파일 배치 계획

### 7.1 백엔드 파일
| No | 파일 경로 | 설명 |
|----|----------|------|
| 1  | {backend.root}/controller/XxController.java | 컨트롤러 |
| 2  | {backend.root}/service/XxService.java | 서비스 |
| 3  | {backend.root}/mapper/XxMapper.java | 매퍼 인터페이스 |
| 4  | {backend.root}/mapper/XxMapper.xml | 매퍼 XML |
| 5  | {backend.root}/model/dto/XxSearchDto.java | 검색 DTO |
| 6  | {backend.root}/model/dto/XxRegDto.java | 등록 DTO |
| 7  | {backend.root}/model/dto/XxResDto.java | 응답 DTO |
| 8  | {backend.root}/model/vo/XxVO.java | CUD VO |
| 9  | {backend.root}/model/vo/XxSelectVO.java | 조회 VO |

### 7.2 프론트엔드 파일
| No | 파일 경로 | 설명 |
|----|----------|------|
| 1  | {frontend.root}/pages/xx/XxListPage.tsx | 목록 페이지 |
| 2  | {frontend.root}/pages/xx/XxDetailPage.tsx | 상세 페이지 |
| 3  | {frontend.root}/pages/xx/components/... | 하위 컴포넌트 |
| 4  | {frontend.root}/pages/xx/hooks/... | 커스텀 훅 |

### 7.3 신규 vs 수정
{신규 생성 파일과 기존 파일 수정 목록 구분}
```

### 6.8 태스크 분해

```markdown
## 8. 태스크 분해

### 8.1 구현 순서
{의존 관계를 고려한 구현 순서}

| 순서 | 태스크 | 의존성 | 예상 범위 |
|------|--------|--------|----------|
| 1 | DB 테이블 생성/변경 | - | DDL |
| 2 | VO/Entity 생성 | 1 | BE 모델 |
| 3 | DTO 생성 | 2 | BE DTO |
| 4 | Mapper 인터페이스 + XML | 2 | BE ORM |
| 5 | Service 구현 | 3,4 | BE 로직 |
| 6 | Controller 구현 | 3,5 | BE API |
| 7 | FE API 훅 정의 | 6 | FE API |
| 8 | FE 목록 페이지 | 7 | FE 화면 |
| 9 | FE 상세/등록/수정 페이지 | 7 | FE 화면 |
| 10 | 통합 테스트 | 8,9 | 검증 |

### 8.2 의존성 다이어그램
{텍스트 기반 의존성 그래프}
```

### 6.9 TDD / 자동화 테스트 계획

```markdown
## 9. TDD / 자동화 테스트 계획

### 9.1 테스트 명령
| 영역 | 명령 | 비고 |
|------|------|------|
| BE | {config.backend.testCmd} | 비어 있으면 프로젝트에서 테스트 러너 탐색 |
| FE | {config.frontend.testCmd} | 비어 있으면 package.json scripts에서 test 계열 탐색 |

### 9.2 RED 테스트 케이스
구현 전에 먼저 작성하고 실패를 확인해야 하는 테스트를 나열한다.

| ID | 영역 | 테스트 파일 | 동작 | 실패해야 하는 이유 | 관련 구현 태스크 |
|----|------|------------|------|-------------------|----------------|
| TDD-BE-01 | BE | {path} | {행동} | 아직 API/Service가 없거나 규칙이 미구현 | Task # |
| TDD-FE-01 | FE | {path} | {행동} | 아직 컴포넌트/훅 동작이 미구현 | Task # |

### 9.3 테스트 데이터/fixture
- {테스트에 필요한 최소 데이터}

### 9.4 회귀 테스트 보강
- 구현 이후 추가해도 되는 보강 테스트. 단, RED 테스트를 대체하지 않는다.
```

이 섹션은 별도 파일 `plan/{기능명}/tdd-plan.md`에도 동일하게 저장한다.

### 6.10 리스크/주의사항

```markdown
## 10. 리스크/주의사항

### 10.1 기획-퍼블리싱 불일치
{Step A와 Step B 비교에서 발견된 불일치 목록}

### 10.2 DDL-기획 불일치
{DDL에 없는 필드, 타입 불일치 등}

### 10.3 기술적 주의사항
- 성능 고려 (대량 데이터, N+1 쿼리 등)
- 동시성 처리 (낙관적 락, 비관적 락)
- 파일 업로드 처리 (docId 연동 등)
- 권한 처리 (화면/API 레벨)
- 트랜잭션 범위
- 공통 코드 처리

### 10.4 확인 필요 사항
{기획서에 명확하지 않아 확인이 필요한 항목}
```

---

## 7. 출력 형식

### 7.1 출력 파일

계획 단계 산출물은 다음 경로에 마크다운 파일로 저장한다:

```
plan/{기능명}/discussion.md
plan/{기능명}/research.md          # 리서치 수행 시 생성
plan/{기능명}/implementation-plan.md
plan/{기능명}/tdd-plan.md
```

`plan/{기능명}/` 디렉토리가 없으면 생성한다.

### 7.2 파일 구조

```markdown
# {기능명} 구현 계획서

> 생성일: {날짜}
> 기획서: {소스 파일 경로}
> 퍼블리싱: {소스 파일 경로}
> DDL: {소스 파일 경로}
> 설계 전 논의: discussion.md
> 계획 전 리서치: research.md / skipped / ignored-existing ({사유})
> TDD 계획: tdd-plan.md

---

## 설계 전 논의 결정사항
{discussion.md의 핵심 결정사항 요약}

---

## 계획 전 리서치
{research.md의 핵심 권장사항, 스킵 사유, 또는 기존 research.md를 무시한 사유}

---

## 프로젝트 패턴 요약
{5.4에서 정리한 내용}

---

## 1. 기능 개요
...

## 2. API 설계
...

## 3. 데이터 모델
...

## 4. DTO 설계
...

## 5. ORM/Mapper 설계
...

## 6. FE 컴포넌트 설계
...

## 7. 파일 배치 계획
...

## 8. 태스크 분해
...

## 9. TDD / 자동화 테스트 계획
...

## 10. 리스크/주의사항
...
```

### 7.3 작성 원칙

- 각 섹션에 **코드 블록**, **테이블**, **텍스트 다이어그램**을 적극 사용한다
- API 설계에는 요청/응답 JSON 예시를 포함한다
- DTO/VO 설계에는 필드 목록과 타입을 테이블로 정리한다
- 쿼리 설계에는 SQL 골격을 코드 블록으로 포함한다
- 모든 설계 항목에는 **근거**를 명시한다. 리서치 상태가 수행/사용이면 어느 research.md 권장사항인지도 함께 적는다

---

## 8. 프레임워크별 분기

설정 파일의 `backend.framework`와 `frontend.framework` 값에 따라 설계 패턴을 분기한다.
이 스킬 디렉토리의 **`frameworks.md`** 에서 해당 프레임워크 섹션만 읽고 적용한다.
다른 프레임워크 섹션은 읽지 않는다.
(지원: spring-boot / express / nestjs / django × react / vue / angular / next)

---

## 9. 범용화 원칙

이 스킬은 특정 프로젝트에 종속되지 않는 **범용 설계 도구**이다. 핵심 원칙:

1. **하드코딩 금지** — 경로/패키지/클래스명 패턴은 `si-config.yml` → README.md 추론 → 사용자 질문 순서로 결정한다
2. **README.md 우선** — README 코드 생성 가이드 > si-config.yml > 기존 코드 패턴 > 프레임워크 Best Practice
3. **기존 코드 패턴 학습** — 설계 전 반드시 유사 기능 코드를 탐색하되, README 표준과 충돌하면 README를 따른다

설정 누락 시 동작 표, 다중 프로젝트 대응 등 상세 규칙은 이 스킬 디렉토리의
**`reference.md`** 를 읽고 따른다 (설정이 불완전하거나 모노레포 구조일 때 필수).

---

## 실행 절차 요약

```
1. si-config.yml 로드
2. 기획서/퍼블리싱/DDL/README.md/샘플 코드 사전 스캔
3. gray area 논의 → plan/{기능명}/discussion.md 저장
4. 리서치 필요성 판단 → 사용자 확인 → research.md 작성 또는 스킵 사유 기록
5. 기획서 분석 → 화면/필드/규칙 추출
6. 퍼블리싱 분석 → 컴포넌트/UI 매핑
7. DDL 분석 → 테이블/컬럼/관계 추출
8. 기존 코드 학습 → 프로젝트 패턴 정리
9. 기획-퍼블리싱-DDL 교차 검증 → 불일치 식별
10. 10개 섹션 구현 계획서 생성 → implementation-plan.md 저장
11. TDD 테스트 설계 → tdd-plan.md 저장
```

각 단계에서 판단이 어려운 사항은 사용자에게 질문하되,
가능한 한 분석 결과에 기반하여 자율적으로 설계한다.
