---
name: riskzero-si-qa-checklist
version: 1.0.0
description: |
  화면 QA 체크리스트 생성 및 브라우저 테스트. 게시판형 CRUD 화면의
  등록/수정/검색/페이징/네비게이션/삭제를 자동 분석하여 체크리스트를 생성한다.
  더미데이터 생성과 브라우저 자동 테스트도 지원.
  Use when asked to "화면검증", "qa checklist", "QA 체크리스트".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
  - Task
---

# 화면 QA 체크리스트 생성기

`$ARGUMENTS`로 전달된 메뉴명(한글), URL 경로, 또는 화면ID를 기반으로 대상 화면을 식별하고 QA 체크리스트를 생성한다. 선택적으로 더미데이터 생성 및 브라우저 테스트까지 수행한다.

---

## Step 0: 설정 파일 로드

### 0-1. 설정 파일 탐색

아래 순서로 설정 파일을 찾는다:

1. `.claude/si-config.yml` (프로젝트 루트 기준)
2. `.claude/qa-config.yml` (프로젝트 루트 기준)
3. `qa-config.yml` (프로젝트 루트)

찾으면 YAML 파싱하여 `config` 변수로 사용한다.

### 0-2. 설정 파일 없으면 → 초기화 플로우

설정 파일이 없으면 프로젝트 구조를 자동 감지하여 `qa-config.yml` 초안을 생성한다.

#### 자동 감지 항목

**프론트엔드 감지:**
- `package.json` → `react`, `vue`, `angular`, `next` 판별
- 라우트 파일 패턴 탐색: `_routes.tsx`, `router/index.ts`, `app/routes/`, `pages/` 등
- 페이지 디렉토리 구조 파악
- 유효성 검증 방식 탐색: `checkValidate`, `yup`, `zod`, `formik` 등
- 파일 업로드 컴포넌트 탐색
- 페이지네이션 컴포넌트 탐색
- 권한 체크 패턴 탐색

**백엔드 감지:**
- `build.gradle` / `pom.xml` → Spring Boot 판별
- `package.json` (server) → Express / NestJS 판별
- `requirements.txt` / `manage.py` → Django 판별
- 소스 루트 및 패키지 구조 파악
- Controller/Service/Mapper 패턴 탐색

**데이터베이스 감지:**
- DDL 파일 탐색: `*.sql` 파일들
- DB 타입 판별: `application.yml` / `.env` 등에서 JDBC URL 확인

**서버 포트 감지:**
- 프론트엔드: `vite.config.*`, `next.config.*`, `package.json` scripts
- 백엔드: `application.yml`, `application.properties`, `.env`

#### 초안 생성

감지 결과를 바탕으로 `.claude/qa-config.yml` 파일을 생성한다. 템플릿:

```yaml
# ============================================================
# QA Checklist Configuration
# 프로젝트별 화면 QA 체크리스트 설정
# ============================================================

project:
  name: "{감지된 프로젝트명}"

# --- 서버 ---
server:
  frontend:
    port: {감지된 포트}
    baseUrl: "http://localhost:{감지된 포트}"
  backend:
    port: {감지된 포트}
    baseUrl: "http://localhost:{감지된 포트}"

# --- 인증 ---
auth:
  type: "jwt"                            # jwt | session | cookie
  loginApi: "/api/auth/login"
  loginFields:
    username: "loginId"
    password: "password"
  tokenField: "accessToken"
  tokenHeader: "Authorization"
  tokenPrefix: "Bearer "

# --- 프론트엔드 ---
frontend:
  root: "{감지된 디렉토리}"
  framework: "{감지된 프레임워크}"
  routes:
    main: "{감지된 메인 라우트}"
    sub: "{감지된 하위 라우트 패턴}"
  pages:
    root: "src/pages"
    listFile: "index.tsx"
    detailPattern: "components/*Detail.tsx"
    apiPattern: "*Api.ts"
  validation:
    method: "{감지된 검증 메서드}"
  fileUpload:
    component: "{감지된 컴포넌트명}"
    props:
      accept: "accept"
      maxCount: "maxCnt"
      maxSize: "maxSize"
  pagination:
    component: "{감지된 컴포넌트명}"
  auth:
    pattern: "{감지된 패턴}"

# --- 백엔드 ---
backend:
  root: "{감지된 디렉토리}"
  framework: "{감지된 프레임워크}"
  language: "{감지된 언어}"
  sourceRoot: "{감지된 소스 루트}"
  basePackage: "{감지된 패키지}"
  structure:
    controller: "{감지된 패턴}"
    service: "{감지된 패턴}"
    mapper: "{감지된 패턴}"
    dto: "{감지된 패턴}"
    vo: "{감지된 패턴}"
    mapperXml: "{감지된 패턴}"
  validation:
    annotations:
      - "@NotBlank"
      - "@NotNull"
      - "@Size"
      - "@Pattern"
      - "@Min"
      - "@Max"
      - "@Email"
  auth:
    pattern: "{감지된 패턴}"

# --- 데이터베이스 ---
database:
  type: "{감지된 DB}"
  ddlPaths:
    - "{감지된 경로}"

# --- 더미데이터 생성 ---
dummyData:
  count: 25
  fileUploadApi: "/api/file/upload"
  prefix: "[QA-{NNN}]"
```

사용자에게 리뷰를 요청한다:
> `qa-config.yml`을 생성했습니다. 경로와 설정을 확인해 주세요. 수정이 필요하면 파일을 수정 후 다시 실행하거나, 그대로 진행할 수 있습니다.

사용자가 진행을 선택하면 Step 1로 넘어간다.

---

## Step 1: 대상 화면 식별

`$ARGUMENTS`를 분석하여 대상 화면을 특정한다.

### 1-1. 입력 유형별 탐색

- **메뉴명(한글)**: `{config.frontend.root}/{config.frontend.pages.root}/` 하위 라우트 파일들에서 매칭
- **URL 경로** (예: `/gh_safety/reg_law`): `{config.frontend.routes.main}` → 하위 라우트에서 해당 path의 컴포넌트 추적
- **화면ID** (예: `reg_law`): 라우트 파일들에서 path 직접 매칭

### 1-2. 관련 파일 자동 탐색

대상 페이지 디렉토리를 기준으로 아래 파일들을 수집한다:

**프론트엔드** (`{config.frontend.root}/{페이지디렉토리}/`):
- `{config.frontend.pages.listFile}` — 목록 페이지
- `{config.frontend.pages.detailPattern}` — 상세/등록/수정 페이지
- `{config.frontend.pages.apiPattern}` — API 훅 정의

**백엔드** (`{config.backend.root}/{config.backend.sourceRoot}/{basePackagePath}/`):
- API 파일에서 URL 경로 추출 → 해당 Controller 역추적
- `{config.backend.structure.controller}` 패턴으로 Controller 탐색
- `{config.backend.structure.service}` 패턴으로 Service 탐색
- `{config.backend.structure.mapper}` 패턴으로 Mapper 탐색
- `{config.backend.structure.dto}` 패턴으로 DTO 탐색
- `{config.backend.structure.vo}` 패턴으로 VO 탐색

**Mapper XML** (`{config.backend.root}/{config.backend.structure.mapperXml}`):
- 해당 Mapper의 XML 파일

**DDL**:
- `{config.database.ddlPaths}` 에서 관련 테이블 검색
- VO/Mapper에서 테이블명 추출 → DDL에서 해당 CREATE TABLE 구문 탐색

### 1-3. 탐색 결과 보고

식별된 파일 목록을 테이블로 출력한다:

```
| 구분 | 파일 | 설명 |
|------|------|------|
| FE 목록 | .../index.tsx | 목록 페이지 |
| FE 상세 | .../*Detail.tsx | 상세/등록/수정 |
| FE API | ...*Api.ts | API 훅 |
| BE Controller | ...*Controller.java | REST API |
| BE Service | ...*ServiceImpl.java | 비즈니스 로직 |
| BE Mapper XML | ...*Mapper.xml | SQL 쿼리 |
| BE DTO | ...*RegDto.java | 등록 DTO |
| DDL | ...ddl.sql (table_name) | 테이블 정의 |
```

---

## Step 2: 화면기획서 분석 (선택)

사용자에게 화면기획서 PPTX 파일 경로를 확인한다. 아래 선택지를 제시한다:

1. PPTX 파일 경로 지정 → python-pptx로 분석
2. 기획서 없이 코드 기반으로만 진행

### PPTX 분석 시 추출 항목

python-pptx를 사용하여 슬라이드에서 추출:
- 화면 ID, 화면명
- 필드 정의 테이블 (필드명, 필수/선택, 타입, 유효성 규칙, 에러 메시지)
- 파일 첨부 제약 (허용 형식, 최대 크기, 최대 개수)
- 목록 컬럼 정의
- 검색 조건 목록
- 버튼 및 네비게이션 플로우

```python
# python-pptx 분석 스크립트 (Bash로 실행)
python3 -c "
from pptx import Presentation
from pptx.util import Inches
import json, sys

prs = Presentation(sys.argv[1])
result = {'slides': []}
for i, slide in enumerate(prs.slides):
    slide_data = {'number': i+1, 'shapes': []}
    for shape in slide.shapes:
        if shape.has_table:
            table = shape.table
            rows = []
            for row in table.rows:
                rows.append([cell.text.strip() for cell in row.cells])
            slide_data['shapes'].append({'type': 'table', 'rows': rows})
        elif shape.has_text_frame:
            text = shape.text_frame.text.strip()
            if text:
                slide_data['shapes'].append({'type': 'text', 'content': text})
    result['slides'].append(slide_data)
print(json.dumps(result, ensure_ascii=False, indent=2))
" "$PPTX_PATH"
```

---

## Step 3: 코드 분석

수집된 파일들을 읽고 아래 항목을 추출한다.

### 3-1. 프론트엔드 분석

**유효성 검증 규칙** — `{config.frontend.validation.method}()` 호출부에서:
- 필드명, 에러 메시지 매핑 추출

**파일 업로드 설정** — `{config.frontend.fileUpload.component}` 컴포넌트에서:
- `{config.frontend.fileUpload.props.accept}` — 허용 파일 형식
- `{config.frontend.fileUpload.props.maxCount}` — 최대 파일 수
- `{config.frontend.fileUpload.props.maxSize}` — 최대 파일 크기
- 도움말 텍스트

**목록 컬럼** — `columns` 배열에서:
- `headerName`, `id`, `width`, `align`

**검색 필터** — `SearchBox` 또는 검색 영역 내부 구성:
- 필터 라벨, 입력 타입 (Select/Input/DatePicker), 옵션값

**페이지네이션** — `{config.frontend.pagination.component}` 관련:
- 페이지 크기 옵션

**권한 코드** — `{config.frontend.auth.pattern}` 패턴 추출

**네비게이션 패턴** — `navigate()` 호출에서:
- 목록→상세, 목록→등록, 상세→수정 경로 패턴

### 3-2. 백엔드 분석

**Bean Validation 어노테이션** — DTO 파일에서 `{config.backend.validation.annotations}` 추출:
- `@NotBlank`, `@NotNull` → 필수 필드
- `@Size(min=, max=)` → 문자열 길이 제한
- `@Pattern(regexp=)` → 정규식 패턴
- `@Min`, `@Max` → 숫자 범위
- `@Email` → 이메일 형식

**Service 검증 로직** — ServiceImpl에서:
- 조건부 필수 필드 (if문으로 검증하는 항목)
- 중복 체크 로직
- 비즈니스 규칙 검증

**Mapper XML 검색 조건** — `whereClause` 또는 `<where>` 절에서:
- `<if test="...">` 조건별 검색 가능 필드
- 검색 타입별 분기

**API 권한** — Controller에서:
- `{config.backend.auth.pattern}` 권한 코드

### 3-3. DDL 분석

테이블 정의에서:
- `NOT NULL` 컬럼 (필수 필드)
- 데이터 타입 및 길이 (`VARCHAR(100)`, `INTEGER` 등)
- `DEFAULT` 값
- 외래키 관계 (`REFERENCES`)
- UNIQUE 제약

---

## Step 4: 불일치 분석

기획서(있는 경우), 프론트엔드, 백엔드, DDL 간 불일치를 감지한다.

### 감지 항목

| 불일치 유형 | 심각도 | 설명 |
|------------|--------|------|
| 필수 필드 누락 | HIGH | DDL NOT NULL인데 FE/BE 검증 없음 |
| 검증 불일치 | HIGH | FE에 검증 있는데 BE에 없음 (또는 반대) |
| 검색 조건 미구현 | MEDIUM | 기획서에 검색조건 있는데 Mapper에 구현 안 됨 |
| 타입 불일치 | MEDIUM | DDL 타입과 DTO 타입 불일치 |
| 길이 제한 불일치 | LOW | DDL VARCHAR 길이와 @Size 불일치 |
| 에러 메시지 미정의 | LOW | 검증은 있는데 사용자 에러 메시지 없음 |

불일치 항목이 있으면 체크리스트에 **[MISMATCH]** 태그로 표시한다.

---

## Step 5: 체크리스트 생성

아래 6개 카테고리로 구조화된 마크다운 파일을 생성한다.

**출력 파일**: `/tmp/qa-checklist-{도메인}-{YYYYMMDD-HHmmss}.md`

### 체크리스트 구조

```markdown
# QA 체크리스트: {메뉴명}

## 대상 화면 정보
- 프로젝트: {config.project.name}
- 메뉴명: {메뉴명}
- URL: {config.server.frontend.baseUrl}/{URL 경로}
- 프론트엔드: {페이지 디렉토리}
- 백엔드: {Controller 클래스}
- 테이블: {테이블명}

## 불일치 사항
| # | 유형 | 심각도 | FE | BE | DDL | 기획서 | 설명 |
|---|------|--------|----|----|-----|--------|------|
(불일치 항목 나열, 없으면 "불일치 없음")

---

## A. 등록 테스트 (Registration)

### A-1. 필드별 검증 규칙
| # | 필드명 | 입력 타입 | 필수 | FE 검증 | BE 검증 | DDL 제약 | 에러 메시지 |
|---|--------|----------|------|---------|---------|----------|------------|
| 1 | {필드명} | text/select/date/file | Y/N | @NotBlank | NOT NULL | {메시지} |

### A-2. 등록 시나리오
- [ ] A-2-1: 빈 폼에서 저장 버튼 클릭 → 필수 필드 에러 메시지 모두 표시
- [ ] A-2-2: 필수 필드 하나씩 채우며 저장 → 나머지 필드 에러 유지 확인
- [ ] A-2-3: (조건부 필수 있는 경우) 조건 충족 시 필수 필드 활성화 확인
- [ ] A-2-4: (파일 첨부 있는 경우) 허용되지 않는 파일 형식 업로드 시도 → 에러
- [ ] A-2-5: (파일 첨부 있는 경우) 최대 개수/크기 초과 업로드 시도 → 에러
- [ ] A-2-6: 모든 필드 정상 입력 후 저장 → 성공, 목록으로 이동
- [ ] A-2-7: 저장 후 목록에서 새 데이터 확인
- [ ] A-2-8: 등록 화면에서 취소 버튼 → 목록으로 이동, 데이터 미저장

## B. 수정 테스트 (Edit)

- [ ] B-1: 상세 화면에서 수정 버튼 클릭 → 수정 모드 전환, 기존 데이터 표시
- [ ] B-2: 기존 데이터 필드별 정상 표시 확인 (텍스트, 날짜, 셀렉트, 파일 등)
- [ ] B-3: 필수 필드 값 삭제 후 저장 → 에러 메시지 표시
- [ ] B-4: (파일 첨부 있는 경우) 기존 파일 삭제 후 저장 → 에러 또는 성공 확인
- [ ] B-5: 정상적으로 값 변경 후 저장 → 성공, 상세 화면에 변경값 반영
- [ ] B-6: 수정 취소 → 상세 화면으로 복귀, 데이터 미변경

## C. 검색 테스트 (Search)

### C-1. 개별 검색 조건
(Mapper XML 또는 API의 검색 조건별로 자동 생성)
- [ ] C-1-1: {검색조건1} 단독 검색 → 결과 정확성 확인
- [ ] C-1-2: {검색조건2} 단독 검색 → 결과 정확성 확인
- [ ] ...

### C-2. 복합 검색
- [ ] C-2-1: 2개 이상 조건 조합 검색 → AND 조건 결과 확인
- [ ] C-2-2: 검색 결과 없는 조건 → "데이터가 없습니다" 메시지 표시

### C-3. 초기화
- [ ] C-3-1: 검색 조건 입력 후 초기화 버튼 → 모든 필터 초기값으로 복원
- [ ] C-3-2: 초기화 후 전체 목록 표시

## D. 페이징 테스트 (Pagination)

> 전제: 20건 이상 데이터 필요

- [ ] D-1: 첫 페이지에서 다음 페이지 이동 → 데이터 변경 확인
- [ ] D-2: 마지막 페이지 이동 → 정상 표시
- [ ] D-3: 페이지 크기 변경 (10→25→50) → 표시 건수 변경 확인
- [ ] D-4: 페이지 크기 변경 시 1페이지로 리셋 확인
- [ ] D-5: 총 건수 표시 정확성 확인
- [ ] D-6: 페이지 이동 후 상세→목록 돌아오기 → 페이지 유지

## E. 네비게이션 테스트 (Navigation)

- [ ] E-1: 목록 → 등록 → 저장 → 목록 이동 → 새 데이터 목록에 표시
- [ ] E-2: 목록 (검색/페이지 설정) → 상세 → 목록 돌아오기 → 검색조건/페이지 유지
- [ ] E-3: 목록 → 상세 → 수정 → 저장 → 상세 화면에 변경 반영
- [ ] E-4: 브라우저 뒤로가기 → 정상 동작 확인

## F. 삭제 테스트 (Delete)

- [ ] F-1: 삭제 버튼 클릭 → 확인 다이얼로그 표시
- [ ] F-2: 확인 다이얼로그에서 취소 → 데이터 유지
- [ ] F-3: 확인 다이얼로그에서 확인 → 삭제 성공, 목록 갱신
- [ ] F-4: 삭제 후 목록에서 해당 데이터 제거 확인
```

생성된 파일 경로를 사용자에게 알려준다.

---

## Step 6: 테스트 실행 (선택적)

체크리스트 생성 후 사용자에게 다음 실행 범위를 확인한다:

1. **체크리스트만 생성** (이미 완료) → 종료
2. **더미데이터 생성** → 6-1 실행
3. **더미데이터 + 브라우저 테스트** → 6-1 + 6-2 실행

### 6-1. 더미데이터 생성

백엔드 API를 통해 `{config.dummyData.count}`건의 테스트 데이터를 생성한다.

#### 사전 준비

```bash
# 1. 로그인하여 토큰 획득
# 사용자에게 테스트 계정 정보를 확인한다 (ID/PW)
TOKEN=$(curl -s -X POST {config.server.backend.baseUrl}{config.auth.loginApi} \
  -H 'Content-Type: application/json' \
  -d '{"{config.auth.loginFields.username}":"$ID","{config.auth.loginFields.password}":"$PW"}' \
  | jq -r '.{config.auth.tokenField}')
```

#### 데이터 생성 전략

- DTO의 필드별 다양한 값으로 `{config.dummyData.count}`건 생성
- 검색 테스트를 위해 의도적으로 다양한 값 분포:
  - 텍스트 필드: `{config.dummyData.prefix}` 형식으로 식별 가능하게 (예: `[QA-001] ~ [QA-025]`)
  - 날짜 필드: 최근 30일 범위 내 분산
  - 셀렉트/코드 필드: 가능한 모든 옵션 골고루 분배
  - 파일 첨부: 5건 정도에만 첨부

#### 파일 첨부 더미 생성

```bash
# 최소 크기 테스트 파일 생성
# 1x1 투명 PNG (68 bytes)
printf '\x89PNG\r\n\x1a\n\x00\x00\x00\rIHDR\x00\x00\x00\x01\x00\x00\x00\x01\x08\x06\x00\x00\x00\x1f\x15\xc4\x89\x00\x00\x00\nIDATx\x9cc\x00\x01\x00\x00\x05\x00\x01\r\n\xb4\x00\x00\x00\x00IEND\xaeB`\x82' > /tmp/qa-test.png

# 빈 PDF
echo '%PDF-1.0
1 0 obj<</Type/Catalog/Pages 2 0 R>>endobj
2 0 obj<</Type/Pages/Kids[3 0 R]/Count 1>>endobj
3 0 obj<</Type/Page/MediaBox[0 0 3 3]>>endobj
xref
0 4
0000000000 65535 f
0000000009 00000 n
0000000058 00000 n
0000000115 00000 n
trailer<</Size 4/Root 1 0 R>>
startxref
190
%%EOF' > /tmp/qa-test.pdf

# 파일 업로드 → docId 획득
DOC_ID=$(curl -s -X POST {config.server.backend.baseUrl}{config.dummyData.fileUploadApi} \
  -H "{config.auth.tokenHeader}: {config.auth.tokenPrefix}$TOKEN" \
  -F "files=@/tmp/qa-test.png" | jq -r '.docId')
```

#### 데이터 등록

```bash
# DTO 구조에 맞게 N건 등록
for i in $(seq 1 {config.dummyData.count}); do
  curl -s -X POST {config.server.backend.baseUrl}/api/{endpoint} \
    -H "{config.auth.tokenHeader}: {config.auth.tokenPrefix}$TOKEN" \
    -H 'Content-Type: application/json' \
    -d "{
      \"field1\": \"{prefix}$(printf '%03d' $i) 테스트 데이터 $i\",
      ...
    }"
  sleep 0.2
done
```

> 실제 필드명과 값은 DTO 분석 결과를 기반으로 동적 생성한다.

### 6-2. 브라우저 테스트

qa-tester 에이전트 지침(`~/.claude/skills/riskzero-si-qa-checklist/qa-tester.md`)을 참조하여 브라우저 테스트를 실행한다.

- 체크리스트 파일 경로와 대상 URL을 전달
- 로그인 자격증명은 사용자에게 확인
- config에서 서버 URL, 인증 정보 등을 참조

### 6-3. 리포트 생성

브라우저 테스트 완료 후 결과를 리포트로 정리한다.

**출력 파일**: `/tmp/qa-report-{도메인}-{YYYYMMDD-HHmmss}.md`

```markdown
# QA Test Report: {메뉴명}

## 실행 요약
- 프로젝트: {config.project.name}
- 실행일시: {YYYY-MM-DD HH:mm:ss}
- 대상 URL: {config.server.frontend.baseUrl}/{path}
- 총 테스트: N개
- PASS: N개 (N%)
- FAIL: N개 (N%)
- SKIP: N개 (N%)

## 불일치 사항 (코드 분석 결과)
| # | 유형 | 심각도 | 설명 |
|---|------|--------|------|

## 실패 항목
| ID | 테스트 케이스 | 기대 결과 | 실제 결과 | 스크린샷 |
|----|-------------|----------|----------|----------|

## 카테고리별 결과

### A. 등록 테스트
| ID | 테스트 케이스 | 상태 | 비고 |
|----|-------------|------|------|

### B. 수정 테스트
| ID | 테스트 케이스 | 상태 | 비고 |

### C. 검색 테스트
| ID | 테스트 케이스 | 상태 | 비고 |

### D. 페이징 테스트
| ID | 테스트 케이스 | 상태 | 비고 |

### E. 네비게이션 테스트
| ID | 테스트 케이스 | 상태 | 비고 |

### F. 삭제 테스트
| ID | 테스트 케이스 | 상태 | 비고 |
```

---

## 참고: 프레임워크별 파일 탐색 힌트

### React (Vite)
- 라우트: `src/_routes.tsx` → `pages/{group}/_route.tsx`
- 페이지: `pages/{group}/{domain}/index.tsx`
- 상세: `pages/{group}/{domain}/components/*Detail.tsx`
- API: `pages/{group}/{domain}/*Api.ts`

### Vue
- 라우트: `src/router/index.ts` → `views/` 또는 `pages/`
- 페이지: `views/{domain}/index.vue` 또는 `List.vue`
- 상세: `views/{domain}/Detail.vue`
- API: `api/{domain}.ts` 또는 `services/{domain}.ts`

### Next.js
- 라우트: `app/` (App Router) 또는 `pages/` (Pages Router)
- 페이지: `app/{domain}/page.tsx`
- API: `app/api/{domain}/route.ts`

### Angular
- 라우트: `app-routing.module.ts` → `{module}-routing.module.ts`
- 페이지: `{domain}/{domain}.component.ts`
- API: `services/{domain}.service.ts`

### Spring Boot (Java/Kotlin)
- Controller: `api/**/{domain}/controller/*Controller.java`
- Service: `api/**/{domain}/service/*ServiceImpl.java`
- Mapper: `api/**/{domain}/mapper/*Mapper.java`
- DTO: `api/**/{domain}/model/dto/*Dto.java`
- Mapper XML: `resources/mapper/**/{domain}/*Mapper.xml`

### Express / NestJS
- Controller: `src/{domain}/{domain}.controller.ts`
- Service: `src/{domain}/{domain}.service.ts`
- DTO: `src/{domain}/dto/*.dto.ts`

### Django
- View: `{app}/views.py`
- Serializer: `{app}/serializers.py`
- URL: `{app}/urls.py`
- Model: `{app}/models.py`
