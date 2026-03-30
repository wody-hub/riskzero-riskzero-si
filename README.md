# riskzero-si

SI 개발 파이프라인 스킬 모음. 기획서 + 퍼블리싱 + DDL을 입력으로 구현 계획 수립부터 코드 구현, 리뷰, QA, 최종 검증까지 8단계를 자동화한다.

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

### 업데이트

```bash
cd ~/riskzero-si && git pull
./setup --host codex
```

Claude Code를 쓰면 마지막 명령만 `./setup --host claude`로 바꾸면 된다.

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
| `/riskzero-si-plan {기능명}` | Step 1 | 기획서+퍼블+DDL → 구현 계획서 |
| `/riskzero-si-plan-review` | Step 2 | 계획서 아키텍처/보안 리뷰 |
| `/riskzero-si-impl {기능명}` | Step 3 | FE/BE 코드 구현 |
| `/riskzero-si-review [경로]` | Step 4 | README.md 기반 표준 리뷰 |
| `/riskzero-si-pr-review` | Step 5 | PR diff 안전성 리뷰 |
| `/riskzero-si-qa-checklist {기능명}` | Step 6 | QA 테스트 체크리스트 생성 |
| `/riskzero-si-qa [URL]` | Step 7 | 버그 조사 + 수정 |
| `/riskzero-si-browse [URL]` | Step 8 | 브라우저 최종 검증 |

## 8단계 파이프라인

```
Step 1: /riskzero-si-plan           ─ 구현 계획 수립
  │
Step 2: /riskzero-si-plan-review    ─ 계획 리뷰
  │
Step 3: /riskzero-si-impl           ─ FE/BE 코드 구현
  │
Step 4: /riskzero-si-review         ─ 프로젝트 표준 리뷰
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
| 신입 개발자 | `/riskzero-si-pipeline {기능명}` — 전체 자동 |
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
