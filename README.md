# riskzero-si

SI 개발 파이프라인 스킬 모음. 기획서 + 퍼블리싱 + DDL을 입력으로 설계 전 논의, 선택적 리서치, 구현/TDD 계획 수립, 코드 구현, 리뷰, QA, 최종 검증까지 8단계를 자동화한다.

## 사전 요구사항

1. **Codex 또는 Claude Code** — 사용할 호스트 세션
2. **Bun** v1.0+ — `curl -fsSL https://bun.sh/install | bash`
3. **gstack** — 아래 설치 참조

## 설치

이 repo는 `~/riskzero-si`처럼 **임의 경로에 clone**해도 된다. 다만 설치할 대상 host에 맞게 `./setup --host codex` 또는 `./setup --host claude`를 명시해서 실행해야 한다.

### Codex 설치

#### Step 1: gstack 설치 (최초 1회)

```bash
git clone https://github.com/garrytan/gstack.git ~/gstack
cd ~/gstack && ./setup --host codex
```

#### Step 2: riskzero-si 설치

```bash
git clone https://github.com/wody-hub/riskzero-riskzero-si.git ~/riskzero-si
cd ~/riskzero-si && ./setup --host codex
```

설치 후 새 Codex 세션을 열면 `riskzero-si-*` 스킬이 로드된다.

### Claude Code 설치

#### Step 1: gstack 설치 (최초 1회)

```bash
git clone https://github.com/garrytan/gstack.git ~/gstack
cd ~/gstack && ./setup --host claude
```

#### Step 2: riskzero-si 설치

```bash
git clone https://github.com/wody-hub/riskzero-riskzero-si.git ~/riskzero-si
cd ~/riskzero-si && ./setup --host claude
```

### 설치 확인

Codex 또는 Claude Code를 **새 세션으로 다시 시작**한 뒤 아래 명령이 인식되는지 확인한다:

```
/riskzero-si-pipeline
```

> 스킬 목록에 `riskzero-si-pipeline`, `riskzero-si-plan` 등이 보이면 설치 완료.

### 업데이트

```bash
cd ~/riskzero-si && git pull
./setup --host codex
```

Claude Code를 쓰면 마지막 명령만 `./setup --host claude`로 바꾸면 된다.

문서만 바뀐 경우에도 새 스킬 추가나 링크 변경이 있을 수 있으므로 `./setup --host ...`를 다시 실행하는 편이 안전하다.

## 사용법

### 프로젝트 초기 설정 (프로젝트당 1회)

```
/riskzero-si-pipeline --init
```

프로젝트 구조를 자동 감지하여 `.codex/si-config.yml` 또는 `.claude/si-config.yml`을 생성한다.

### 전체 파이프라인 실행

```
/riskzero-si-pipeline {기능명}
```

8단계를 순차 실행한다.

### 구간 실행

```
/riskzero-si-pipeline {기능명} --from=3 --to=5
```

Step 3(구현) ~ Step 5(PR 리뷰)만 실행한다.

### 개별 스킬 실행

| 명령 | 단계 | 설명 |
|------|------|------|
| `/riskzero-si-plan {기능명}` | Step 1 | 기획서+퍼블+DDL → 논의 + 선택적 리서치 + 구현/TDD 계획 |
| `/riskzero-si-plan-review` | Step 2 | 논의/리서치/TDD 포함 계획 리뷰 |
| `/riskzero-si-impl {기능명}` | Step 3 | TDD 기반 FE/BE 코드 구현 |
| `/riskzero-si-review [경로]` | Step 4 | README.md 기반 표준 + 리서치/TDD 증거 리뷰 |
| `/riskzero-si-pr-review` | Step 5 | PR diff 안전성 리뷰 |
| `/riskzero-si-qa-checklist {기능명}` | Step 6 | QA 테스트 체크리스트 생성 |
| `/riskzero-si-qa [URL]` | Step 7 | 버그 조사 + 수정 |
| `/riskzero-si-browse [URL]` | Step 8 | 브라우저 최종 검증 |
| `/riskzero-si-help [질문]` | — | 사용 가이드 안내 (GUIDE.md 기반 Q&A) |

### 단계 사이 `/clear` 운용 (단계 종료 푸터)

모든 스킬은 한 단계를 끝낼 때마다 답변 맨 끝에 **단계 종료 푸터**를 출력한다:

- 전체 8단계 중 현재 위치(✅완료 / 👉현재·다음 / ⬜대기)
- **복사해서 바로 실행할 수 있는 다음 단계 명령어** (기능명이 채워진 완전한 형태)
- `/clear` 안내

각 단계 산출물은 `.si-planning/{기능명}/`에 자기완결적으로 저장되므로, 단계 사이에 `/clear`로 컨텍스트를 비운 뒤 푸터의 다음 명령어를 그대로 붙여넣으면 다음 단계가 직전 산출물을 읽어 단독 실행된다.

> ⚠️ Step 6 `/riskzero-si-qa-checklist`만 기능명이 아니라 **메뉴명 / URL / 화면ID**를 인자로 받는다. (Step 5 푸터에서 안내한다.)

## 8단계 파이프라인

```
Step 1: /riskzero-si-plan           ─ 논의 + 선택적 리서치 + 구현/TDD 계획
  │
Step 2: /riskzero-si-plan-review    ─ 논의/리서치/TDD 포함 계획 리뷰
  │
Step 3: /riskzero-si-impl           ─ TDD 기반 FE/BE 코드 구현
  │
Step 4: /riskzero-si-review         ─ 프로젝트 표준 + 리서치/TDD 증거 리뷰
  │
Step 5: /riskzero-si-pr-review      ─ PR diff 리뷰
  │
Step 6: /riskzero-si-qa-checklist   ─ QA 체크리스트 생성
  │
Step 7: /riskzero-si-qa             ─ 버그 조사 + 수정
  │
Step 8: /riskzero-si-browse         ─ 최종 브라우저 검증
```

## 팀원별 사용 패턴

| 역할 | 주로 사용하는 명령 |
|------|------------------|
| 신입 개발자 | `/riskzero-si-help` 로 학습 → `/riskzero-si-pipeline {기능명}` — 전체 자동 |
| 경험 개발자 | `/riskzero-si-plan` → 직접 구현 → `/riskzero-si-review` |
| 리뷰어 | `/riskzero-si-review` + `/riskzero-si-pr-review` |
| QA 담당 | `/riskzero-si-qa-checklist` → `/riskzero-si-browse` |
| 팀리드 | `/riskzero-si-pipeline --from=4` (리뷰~검증만) |

## si-config.yml

프로젝트별 설정 파일. `/riskzero-si-pipeline --init`으로 자동 생성하거나 `si-config.template.yml`을 복사하여 수동 작성.

권장 위치:
- Codex: `.codex/si-config.yml`
- Claude Code: `.claude/si-config.yml`
- 공용 fallback: `si-config.yml`

주요 설정:
- `project.readme` — 프로젝트 표준 문서 경로
- `sources.*` — 기획서/퍼블리싱/DDL 경로
- `server.*` — 프론트엔드/백엔드 포트
- `frontend/backend.*` — 프레임워크, 빌드 명령, 소스 구조
- `outputs.*` — 기능별 산출물 디렉토리와 문서 파일명
- `planning.*` — 계획 전 리서치 기본 동작
- `testing.*` — TDD 필수 여부와 RED/GREEN 증거 검증 정책
- `orchestration.*` — 단계별 권장 실행 주체와 멀티 에이전트/Dynamic Workflow 정책

## 산출물 구조

기본 산출물 위치는 `.si-planning/{기능명}/`이다. `si-config.yml`의 `outputs.root`와
`outputs.featureDirPattern`으로 변경할 수 있다.

```
.si-planning/
  └── {기능명}/
      ├── discussion.md
      ├── research.md              # 리서치 수행 시 생성
      ├── implementation-plan.md
      ├── tdd-plan.md
      ├── plan-review.md
      ├── tdd-report.md
      ├── code-review.md
      ├── pr-review.md
      ├── qa-checklist.md
      ├── qa-report.md
      ├── final-report.md
      └── evidence/
          ├── screenshots/
          ├── logs/
          └── test-results/
```

`research.md`는 리서치를 수행한 경우에만 생성한다. 옵션 없이 `/riskzero-si-plan`을 실행하면
AI가 필요성을 판단해 사용자에게 진행/스킵을 묻는다. 강제 실행은 `--research`,
강제 스킵은 `--skip-research`를 사용한다.
기존 `research.md`가 있어도 사용자가 스킵하면 계획서에 `ignored-existing` 상태와 사유를 남기고
후속 단계는 계획서의 리서치 상태를 파일 존재보다 우선한다.

외부 리서치는 런타임에 웹 검색/브라우징 도구가 있을 때만 수행한다. 기본 정책은
`planning.externalResearchDefault: auto`와 `planning.externalResearchSourcePolicy: official-first`이며,
공식 문서/표준/벤더 문서를 우선하고 출처 URL, 확인일, 적용 판단을 `research.md`에 남긴다.
네트워크가 없거나 정책이 `internal-only`이면 내부 README/샘플/기획/DDL 기반으로 진행하고 사유를 기록한다.

## 모델/멀티 에이전트 적용

가능하다. 다만 host별 제약이 다르다.

| 단계 | 권장 방식 | 이유 |
|------|-----------|------|
| Step 1 계획 | Claude Opus급 또는 Codex high reasoning, 필요 시 외부 리서치 subagent | 요구사항/DDL/퍼블리싱을 종합하고 gray area를 논의해야 함 |
| Step 2 계획 리뷰 | Claude Opus급 또는 Codex review | 설계 반례, 보안, TDD 누락을 비판적으로 점검 |
| Step 3 구현 | Codex 중심, BE/FE가 분리되면 subagents 가능 | 실제 파일 편집과 테스트 반복에 적합 |
| Step 4 코드 리뷰 | Codex review + Claude 교차 검토 권장 | 프로젝트 표준과 설계 반영을 서로 다른 관점으로 확인 |
| Step 5 PR 리뷰 | Codex review | diff 기반 안전성 점검에 적합 |
| Step 6 QA 체크리스트 | Claude/Codex 모두 가능, 브라우저 테스트는 gstack/browse | 화면 시나리오 생성과 실제 조작이 분리됨 |
| Step 7 QA 수정 | Codex 중심 | 재현, 수정, 테스트 루프가 필요 |
| Step 8 최종 검증 | Codex + gstack browser | 실제 브라우저 증거 수집 |

Codex 환경에서는 현재 세션의 subagent 도구가 모델을 직접 지정하지 않을 수 있으므로
`orchestration.stageRecommendations`는 권장 라우팅 문서로 사용하고, 실제 모델 선택은 런타임 설정을 따른다.
Claude Code에서는 custom subagent의 `model` frontmatter 또는 실행 시 모델 설정을 사용할 수 있다.
Claude Code Dynamic Workflows는 큰 리서치/감사/대규모 구현처럼 병렬성이 높은 작업에만 `ask` 후 사용한다.
subagent와 Dynamic Workflow는 단계 내부 보조자이며, 사용자 확인 게이트와 최종 PASS/FAIL 판정은 파이프라인 오너가 유지한다.
병렬 쓰기는 기본 금지이고, BE/FE처럼 파일 범위가 명확히 분리될 때만 별도 허용한다.

`/tmp`는 임시 파일과 과거 호환 fallback으로만 사용한다. 최종 체크리스트,
리포트, 스크린샷, 테스트 결과는 기능별 산출물 디렉토리에 저장한다.

## 의존성

gstack을 사전 설치해야 한다. 아래 gstack 스킬을 내부적으로 활용한다:

| riskzero-si 스킬 | 내부 사용 gstack 스킬 |
|---|---|
| `/riskzero-si-plan-review` | `/plan-eng-review` |
| `/riskzero-si-pr-review` | `/review` |
| `/riskzero-si-qa` | `/investigate` + `/qa` |
| `/riskzero-si-browse` | `/browse` + `/qa-only` |

## 라이선스

Private — Riskzero internal use only.
