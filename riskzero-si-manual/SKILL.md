---
name: riskzero-si-manual
version: 1.0.0
description: Use when creating or updating GHSSIS-style user manual PPT for SI menus; triggers include "매뉴얼 제작", "사용자 매뉴얼", "매뉴얼 PPT", "manual ppt". 실행 중인 프론트엔드에서 화면을 자동 캡처하고 마커·설명을 얹어 기존 작업자 매뉴얼(GHSSIS-MZ-01-01)과 동일한 판형의 PPTX를 생성한다. 일반 PPT 제작 요청에는 사용하지 않는다.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# SCSMS 사용자 매뉴얼 자동 제작

기존 작업자 제작 매뉴얼(`GHSSIS-MZ-01-01-사용자매뉴얼_Web_v0.1`)의 설계 철학을
재현하는 파이프라인. 메뉴 1개 = 섹션 스펙 1파일이며, 스펙만 작성하면
캡처 → 조판 → 검증이 자동화된다.

디자인 상세는 `references/design-spec.md` (원본 XML 실측 명세) 필독.

## 파일 맵 (scsms-frontend 기준)

| 파일 | 역할 |
|---|---|
| `e2e/manual/ghssis/template.mjs` | 디자인 시스템 (판형·색·컴포지터·페이지 자동분할) |
| `e2e/manual/ghssis/assets/gh-logo.png` | 원본 추출 GH 로고 (180×44) |
| `e2e/manual/sections/<id>.section.mjs` | **메뉴별 섹션 스펙 (작업 단위)** |
| `e2e/manual/capture-sections.mjs` | Playwright 캡처 러너 (마커 좌표 자동 산출) |
| `e2e/manual/build-ghssis.mjs` | PPTX 조립 (표지·목차·간지·본문) |
| `manual-out/ghssis/` | 산출물 (manifest, images, pptx) |

## 사전 조건

- 프론트(3100)·백엔드(8085) 기동, `.env.e2e`에 E2E 계정
- 세션 만료 시 러너가 자동 재로그인
- 검증 도구: `/Applications/LibreOffice.app/Contents/MacOS/soffice`, `pdftoppm`

## 메뉴 1개 추가 절차

### 1. DOM 프로브 (셀렉터 확보)

스크립트로 대상 화면의 구조를 먼저 확인한다:

```bash
node --input-type=module -e "
import { chromium } from '@playwright/test';
const b = await chromium.launch();
const ctx = await b.newContext({ storageState: './e2e/.auth/user.json', viewport: { width: 1920, height: 900 } });
const page = await ctx.newPage();
await page.goto('http://localhost:3100/<라우트>');
await page.waitForLoadState('networkidle');
console.log('버튼:', (await page.locator('main button:visible').allTextContents()).filter(Boolean));
console.log('main:', await page.locator('main').boundingBox());
await b.close();"
```

기존 `e2e/<번호>-<메뉴>.spec.js`에 라우트·셀렉터·플로우가 이미 있으니 참조.

### 2. 섹션 스펙 작성

`e2e/manual/sections/<id>.section.mjs` — `notice.section.mjs`가 표준 예시.

```js
export default {
  id: '<id>',
  chapter: { no: N, title: '장 제목', desc: '장 설명 한 줄' },
  section: { no: 'N-M', title: '절 제목' },
  breadcrumb: ['사용자 매뉴얼', '장 제목', 'N-M. 절 제목'],
  summary: "화면 목적 1~2문장. '핵심어'는 작은따옴표로 강조.",
  blocks: [
    {
      label: '접속하기',          // 기능 라벨: 접속하기/조회/등록하기/설정하기…
      body: "안내 문장. '버튼명'은 작은따옴표 → 파란 볼드 렌더.",
      capture: {
        url: '/라우트',
        prepare: async (page) => {},        // 행 클릭 등 상태 조작 (옵션)
        clip: 'main' | 'viewport' | locator, // 캡처 영역
        clipBottom: (page) => locator,       // 하단 빈 여백 제거 기준 (옵션)
        markers: [
          { target: (page) => locator, text: '요소 역할 설명' },
        ],
      },
      tip: 'Tip 문구 (옵션 — 권한·제한·예외 안내)',
    },
  ],
};
```

**문체 규칙 (원본 준수)**
- 경어체: "~해 주시기 바랍니다", "~하실 수 있습니다"
- 버튼·메뉴명: 작은따옴표 필수 → 파란 볼드 자동 변환
- 마커 텍스트: 요소가 "무엇이고 무엇을 할 수 있는지"
- 블록 구성 표준: 접속하기 → 조회 → 등록/수정 → (Tip's)

### 3. 실행

```bash
node e2e/manual/capture-sections.mjs <id...>   # 캡처 + 마커 좌표
node e2e/manual/build-ghssis.mjs <id...>       # PPTX 조립 (여러 id = 합본)
```

### 4. 검증 (필수 — 눈으로 확인 전에 완료 선언 금지)

```bash
cd manual-out/ghssis && cp GHSSIS-사용자매뉴얼.pptx v.pptx
/Applications/LibreOffice.app/Contents/MacOS/soffice --headless --convert-to pdf --outdir . v.pptx
pdftoppm -png -r 80 v.pdf gs
```

렌더 이미지를 Read로 열어 확인:
- [ ] 마커가 대상 요소 위에 정확히 있는가 (locator 첫 매칭 오류 주의)
- [ ] 스크린샷 하단 빈 여백 없는가 (`clipBottom` 조정)
- [ ] 텍스트 넘침/겹침 없는가 (본문 페이지 하한 y=10.35)
- [ ] 절 내 마커 연번이 이어지는가 (❶…❿, 10개 초과 시 블록 분할)
- [ ] 원본과 비교: `manual-out/.raw/web-ref.pdf` 해당 유형 페이지

### 5. 자율 모드 (여러 메뉴 일괄)

메뉴별로 1→4를 반복하되, **캡처 실패(마커 대상 못 찾음 경고 포함)는 절대
조용히 넘기지 말 것** — 스펙을 수정해 재시도하고, 해결 불가 시 보고서에 명시.
`test.skip()`류 데이터 부재 화면은 시드 데이터 준비 후 재캡처.

## 함정 (실측)

- pptxgenjs `LAYOUT_16x9`는 10×5.625in — 반드시 `defineLayout` 사용 (A4P 정의됨)
- `{width=…}{page=…}` 같은 지시자 체이닝 개념 없음 — 좌표는 전부 인치 수치
- LibreOffice 변환은 **산출물 디렉터리에서** 실행 (상대경로 이슈)
- soffice가 한글 파일명을 못 열면 ASCII 이름으로 복사 후 변환
- 원본 판형은 A4 세로 — 스크린샷은 전체 화면이 아닌 **기능 영역 크롭**이 원칙
- pptxgenjs는 npm 전역이 아닌 프로젝트 node_modules 필요 (`yarn add -D pptxgenjs` 미반영 상태면 재설치)
