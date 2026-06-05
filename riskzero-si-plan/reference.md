# 범용화 원칙 상세 규칙

SKILL.md §9에서 참조하는 문서. 핵심 원칙 3가지는 SKILL.md에 요약되어 있으며,
이 문서는 설정 누락 시 동작과 다중 프로젝트 대응 등 상세 규칙을 담는다.

## 9.1 하드코딩 금지

- 모든 경로, 패키지명, 클래스명 패턴은 `si-config.yml`에서 읽는다
- config에 없으면 `README.md`에서 추론한다
- 추론도 불가능하면 사용자에게 질문한다
- 절대로 특정 프로젝트의 경로나 패키지를 코드에 직접 쓰지 않는다

## 9.2 README.md 우선

```
우선순위:
1. README.md에 명시된 코드 생성 가이드 / 코딩 컨벤션
2. si-config.yml에 정의된 설정
3. 기존 코드에서 학습한 패턴
4. 프레임워크의 일반적인 Best Practice
```

README.md에 **코드 생성 가이드** 섹션이 있다면 해당 내용을 최우선으로 따른다.
기존 코드의 패턴과 README.md 규칙이 충돌하면 README.md를 따른다.

## 9.3 기존 코드 패턴 학습

- 새 코드를 설계하기 전에 반드시 기존 코드를 탐색한다
- 유사한 CRUD 기능이 있다면 그 패턴을 참고한다
- 단, README.md 표준과 충돌하면 README.md를 따른다

## 9.4 설정 누락 시 동작

필수 설정이 누락된 경우의 동작:

| 누락 항목 | 동작 |
|-----------|------|
| `sources.wireframe` | 사용자에게 기획서 경로 질문 |
| `sources.publishing` | 퍼블리싱 없이 기획서 + DDL만으로 설계 |
| `sources.ddl` | DDL 없이 기획서 기반으로 테이블 구조 추론 |
| `backend.framework` | README.md / package.json / build.gradle 등에서 추론 |
| `frontend.framework` | README.md / package.json 등에서 추론 |
| `project.readme` | 프로젝트 루트에서 README.md 자동 탐색 |

## 9.5 다중 프로젝트 대응

모노레포 또는 별도 저장소 구조 모두 지원한다.

- `backend.root`와 `frontend.root`가 다른 경로일 수 있다
- 각각의 README.md가 별도로 존재할 수 있다
- 설정에서 명시된 경로를 기준으로 탐색한다
