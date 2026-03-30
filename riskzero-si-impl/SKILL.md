---
name: riskzero-si-impl
version: 1.0.0
description: |
  구현 계획서 기반 FE/BE 코드 구현. Ralph-loop 패턴 적용.
  계획서를 읽고 백엔드/프론트엔드 코드를 생성하며, 빌드 검증을 반복한다.
  /riskzero-si-impl {기능명} [--be-only] [--fe-only]
  Use when asked to "구현", "코드 작성", "implement", "코드노예".
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

# 구현 스킬 (Ralph-loop 패턴)

## 1. 역할

`plan/{기능명}/implementation-plan.md` 구현 계획서를 기반으로 백엔드/프론트엔드 코드를 생성한다.
계획서에 명시된 태스크를 순서대로 구현하며, Ralph-loop 패턴으로 품질을 보장한다.

## 2. 설정 로드

구현 시작 전 반드시 아래 파일을 읽는다:

1. **si-config.yml** - 프로젝트 경로, 빌드 명령어, 프레임워크 정보
2. **README.md** (백엔드/프론트엔드 각각) - 코딩 컨벤션, 패키지 구조, 네이밍 규칙

```
si-config.yml 주요 설정:
  backend.root        → 백엔드 소스 루트
  backend.buildCmd    → 백엔드 빌드 명령어
  backend.framework   → spring-boot | express | nestjs | django
  frontend.root       → 프론트엔드 소스 루트
  frontend.buildCmd   → 프론트엔드 빌드 명령어
  frontend.lintCmd    → 프론트엔드 린트 명령어
  frontend.framework  → react | vue | angular | next
  sources.sample      → 샘플 코드 경로
```

## 3. 구현 절차

### 3.1. 태스크 추출

`plan/{기능명}/implementation-plan.md` 파일에서 구현 태스크 목록을 추출한다.
각 태스크는 BE/FE 구분이 명시되어 있어야 한다.

### 3.2. 실행 분기

| 옵션 | 동작 |
|------|------|
| (없음) | BE 태스크 → FE 태스크 순서로 전체 구현 |
| `--be-only` | BE 태스크만 구현 |
| `--fe-only` | FE 태스크만 구현 |

### 3.3. 구현 순서

**BE 먼저 → FE 나중** 원칙을 따른다.
API가 먼저 존재해야 프론트엔드에서 연동할 수 있기 때문이다.

1. BE: Controller → Service → Mapper/Repository → DTO/VO → SQL/Schema
2. FE: API 훅 → 페이지 컴포넌트 → 공통 컴포넌트 → 라우트 등록

## 4. Ralph-loop 패턴 (최대 3회 반복)

각 태스크 그룹(BE 또는 FE)에 대해 아래 5단계를 반복한다.
최대 3회 반복하며, 품질 게이트를 통과하면 즉시 종료한다.

### Iteration N (N = 1, 2, 3)

#### 4.1. Plan (계획)
- 이번 반복에서 달성할 목표 1~3개를 선정한다.
- 첫 반복: 계획서의 태스크를 순서대로 선택
- 재반복: 이전 반복에서 실패한 항목 + 미완료 항목

#### 4.2. Implement (구현)
- 최소한의 코드 변경으로 목표를 달성한다.
- README.md의 코딩 규칙을 반드시 준수한다.
- BE 구현 시 → `be-developer.md` 참조
- FE 구현 시 → `fe-developer.md` 참조

#### 4.3. Verify (검증)
- BE: `config.backend.buildCmd` 실행 → 컴파일 성공 확인
- FE: `config.frontend.buildCmd` 실행 → 빌드 성공 확인
- 빌드 실패 시 에러 메시지 분석

#### 4.4. Review (자체 점검)
- 패키지/디렉토리 구조가 README.md 패턴과 일치하는가?
- 네이밍 규칙 (메서드명, 클래스명, 파일명)이 올바른가?
- 컨벤션 위반 사항이 없는가?
- 불필요한 코드가 포함되지 않았는가?

#### 4.5. Refine (수정)
- Verify 또는 Review에서 발견된 문제를 수정한다.
- 수정 후 다음 반복의 Plan 단계로 돌아간다.
- 3회 반복 후에도 실패하면 실패 사유를 상세히 보고한다.

## 5. 에이전트 위임

구현 단계에서 아래 문서를 참조하여 각 영역의 전문 규칙을 따른다:

| 영역 | 참조 문서 | 설명 |
|------|-----------|------|
| BE | `be-developer.md` | 백엔드 프레임워크별 구현 패턴, 레이어 규칙 |
| FE | `fe-developer.md` | 프론트엔드 프레임워크별 구현 패턴, 컴포넌트 규칙 |

## 6. 품질 게이트

아래 조건을 **모두** 만족해야 구현 완료로 판정한다:

- [ ] 빌드 성공 (BE buildCmd + FE buildCmd 모두 통과)
- [ ] 패키지/디렉토리 구조가 README.md 패턴과 일치
- [ ] 네이밍 규칙 준수 (README.md 기준)
- [ ] 컴파일 에러 및 런타임 에러 없음
- [ ] 계획서의 모든 태스크 구현 완료

## 7. 출력

구현 완료 시 아래 정보를 출력한다:

```
## 구현 결과

### 생성/수정된 파일
- [BE] src/main/java/.../controller/XxxController.java (신규)
- [BE] src/main/java/.../service/XxxService.java (신규)
- [FE] src/pages/xxx/XxxList.tsx (신규)
- ...

### 빌드 결과
- BE: ✅ 성공 (0 errors, 0 warnings)
- FE: ✅ 성공 (0 errors, 0 warnings)

### Ralph-loop 반복 횟수
- BE: 1회 (1회차에서 통과)
- FE: 2회 (1회차 빌드 실패 → 2회차 수정 후 통과)

### 참고 사항
- (특이사항이 있으면 기술)
```
