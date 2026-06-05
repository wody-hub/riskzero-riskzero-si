---
name: riskzero-si-impl
version: 1.0.0
description: Use when implementing an approved riskzero-si plan or generating FE/BE code from plan artifacts; triggers include "구현", "코드 작성", "implement", "코드노예". riskzero-si 구현 계획서가 없는 일반 구현/코드 작성 요청에는 사용하지 않는다.
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
3. **plan/{기능명}/discussion.md** - 설계 전 논의 결정사항
4. **plan/{기능명}/research.md** - 계획 전 기술 리서치 결과(있으면 읽음)
5. **plan/{기능명}/tdd-plan.md** - 구현 전에 작성해야 할 RED 테스트 케이스

```
si-config.yml 주요 설정:
  backend.root        → 백엔드 소스 루트
  backend.buildCmd    → 백엔드 빌드 명령어
  backend.testCmd     → 백엔드 테스트 명령어
  backend.framework   → spring-boot | express | nestjs | django
  frontend.root       → 프론트엔드 소스 루트
  frontend.buildCmd   → 프론트엔드 빌드 명령어
  frontend.lintCmd    → 프론트엔드 린트 명령어
  frontend.testCmd    → 프론트엔드 테스트 명령어
  frontend.framework  → react | vue | angular | next
  sources.sample      → 샘플 코드 경로
  testing.tddRequired → TDD 필수 여부
```

## 3. 구현 절차

### 3.1. 태스크 추출

`plan/{기능명}/implementation-plan.md` 파일에서 구현 태스크 목록을 추출한다.
각 태스크는 BE/FE 구분이 명시되어 있어야 한다.

먼저 `implementation-plan.md`의 "계획 전 리서치" 상태를 확인한다.
`Status: skipped` 또는 `Status: ignored-existing`이면 `research.md`가 남아 있어도 현재 구현에는 적용하지 않고 스킵/무시 사유만 확인한다.
그 외에는 `plan/{기능명}/research.md`가 있으면 권장 접근, 위험, 테스트/QA 관점을 먼저 반영한다.

`plan/{기능명}/tdd-plan.md` 파일에서 RED 테스트 케이스와 실행 명령을 추출한다.
`testing.tddRequired: true`인데 `tdd-plan.md`가 없거나 비어 있으면 구현을 시작하지 말고
`/riskzero-si-plan {기능명}`을 다시 실행해 TDD 계획을 보강하도록 안내한다.

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

## 4. TDD + Ralph-loop 패턴 (최대 3회 반복)

각 태스크 그룹(BE 또는 FE)에 대해 아래 6단계를 반복한다.
최대 3회 반복하며, 품질 게이트를 통과하면 즉시 종료한다.

**핵심 원칙:** production code를 작성하기 전에 해당 동작을 검증하는 테스트를 먼저 작성하고,
그 테스트가 기대한 이유로 실패하는 RED 상태를 확인한다. 구현 후 추가하는 회귀 테스트는 허용되지만
RED 증거를 대체하지 않는다.

### Iteration N (N = 1, 2, 3)

#### 4.1. Plan (계획)
- 이번 반복에서 달성할 목표 1~3개를 선정한다.
- 첫 반복: 계획서의 태스크를 순서대로 선택
- 재반복: 이전 반복에서 실패한 항목 + 미완료 항목
- 각 목표와 연결된 `tdd-plan.md`의 테스트 케이스를 선정한다.

#### 4.2. RED (테스트 먼저 작성)
- 구현 대상 동작을 검증하는 최소 테스트를 먼저 작성한다.
- 테스트 파일은 프로젝트의 기존 테스트 위치와 네이밍 규칙을 따른다.
- 테스트 명령은 `config.backend.testCmd`, `config.frontend.testCmd`, 또는 프로젝트에서 탐색한 테스트 명령을 사용한다.
- 테스트를 실행하여 실패를 확인한다.
- 실패는 컴파일 오류나 테스트 코드 오타가 아니라, 아직 기능이 구현되지 않았기 때문에 발생해야 한다.
- 테스트가 바로 통과하면 기존 동작을 테스트한 것이므로 테스트를 다시 설계한다.
- RED 결과를 `plan/{기능명}/tdd-report.md`에 기록한다.

#### 4.3. GREEN Implement (구현)
- 최소한의 코드 변경으로 목표를 달성한다.
- README.md의 코딩 규칙을 반드시 준수한다.
- BE 구현 시 → `be-developer.md` 참조
- FE 구현 시 → `fe-developer.md` 참조

#### 4.4. Verify (검증)
- 먼저 RED에서 작성한 테스트를 다시 실행하여 GREEN을 확인한다.
- BE: `config.backend.buildCmd` 실행 → 컴파일 성공 확인
- BE: `config.backend.testCmd`가 있으면 실행 → 관련 테스트 성공 확인
- FE: `config.frontend.buildCmd` 실행 → 빌드 성공 확인
- FE: `config.frontend.lintCmd` 실행 → 린트 성공 확인
- FE: `config.frontend.testCmd`가 있으면 실행 → 관련 테스트 성공 확인
- 빌드 실패 시 에러 메시지 분석
- GREEN 결과와 빌드/테스트 로그 위치를 `tdd-report.md`에 기록한다.

#### 4.5. Review (자체 점검)
- 패키지/디렉토리 구조가 README.md 패턴과 일치하는가?
- 네이밍 규칙 (메서드명, 클래스명, 파일명)이 올바른가?
- 컨벤션 위반 사항이 없는가?
- 불필요한 코드가 포함되지 않았는가?
- 테스트가 실제 동작을 검증하는가, mock만 검증하지 않는가?
- 테스트 이름이 기대 동작을 설명하는가?

#### 4.6. REFACTOR / Refine (수정)
- Verify 또는 Review에서 발견된 문제를 수정한다.
- 수정 후 다음 반복의 Plan 단계로 돌아간다.
- 3회 반복 후에도 실패하면 실패 사유를 상세히 보고한다.
- 리팩터링 후에는 관련 테스트와 빌드/린트를 다시 실행하고 결과를 기록한다.

## 5. 에이전트 위임

구현 단계에서 아래 문서를 참조하여 각 영역의 전문 규칙을 따른다:

| 영역 | 참조 문서 | 설명 |
|------|-----------|------|
| BE | `be-developer.md` | 백엔드 프레임워크별 구현 패턴, 레이어 규칙 |
| FE | `fe-developer.md` | 프론트엔드 프레임워크별 구현 패턴, 컴포넌트 규칙 |

## 6. 품질 게이트

아래 조건을 **모두** 만족해야 구현 완료로 판정한다:

- [ ] `tdd-plan.md`의 필수 RED 테스트가 작성됨
- [ ] RED 실패 로그가 `tdd-report.md`에 기록됨
- [ ] GREEN 성공 로그가 `tdd-report.md`에 기록됨
- [ ] 빌드 성공 (BE buildCmd + FE buildCmd 모두 통과)
- [ ] 설정된 테스트 명령어 통과 (backend.testCmd / frontend.testCmd)
- [ ] 패키지/디렉토리 구조가 README.md 패턴과 일치
- [ ] 네이밍 규칙 준수 (README.md 기준)
- [ ] 컴파일 에러 및 런타임 에러 없음
- [ ] 계획서의 모든 태스크 구현 완료

## 7. TDD 리포트 형식

구현 완료 시 `plan/{기능명}/tdd-report.md`를 작성한다.

```markdown
# TDD 실행 리포트: {기능명}

## 요약
- BE RED/GREEN: PASS / FAIL / N/A
- FE RED/GREEN: PASS / FAIL / N/A
- Build: PASS / FAIL
- Test: PASS / FAIL

## 테스트 케이스별 결과
| ID | 영역 | 테스트 파일 | RED 결과 | GREEN 결과 | 관련 태스크 |
|----|------|------------|----------|------------|------------|

## 실행 명령
| 단계 | 명령 | 결과 | 로그/증거 |
|------|------|------|-----------|
| RED | ... | expected failure | evidence/test-results/... |
| GREEN | ... | pass | evidence/test-results/... |

## 회귀 테스트 보강
- 구현 후 추가한 테스트가 있으면 기록한다. RED 테스트를 대체하지 않는다.

## 미해결 사항
- 테스트 명령 부재, 환경 문제, flaky 테스트 등
```

## 8. 출력

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

### TDD 결과
- RED: PASS - 기대한 실패 확인
- GREEN: PASS - 테스트 통과
- 리포트: plan/{기능명}/tdd-report.md

### Ralph-loop 반복 횟수
- BE: 1회 (1회차에서 통과)
- FE: 2회 (1회차 빌드 실패 → 2회차 수정 후 통과)

### 참고 사항
- (특이사항이 있으면 기술)
```
