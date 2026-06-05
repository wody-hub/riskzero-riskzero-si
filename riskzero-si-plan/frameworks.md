# 프레임워크별 설계 패턴 가이드

SKILL.md §8에서 참조하는 문서. 설정 파일의 `backend.framework`와 `frontend.framework` 값을
확인한 뒤, **해당 프레임워크 섹션만** 읽고 설계에 적용한다. 다른 프레임워크 섹션은 읽지 않는다.

## 8.1 백엔드 프레임워크 분기

### spring-boot (Java/Kotlin)
- **계층 구조**: Controller → Service → Mapper
- **ORM**: MyBatis XML 매퍼 (또는 JPA Entity)
- **DTO 분리**: SearchDto, RegDto, ModDto, ResDto 각각 별도 파일
- **VO 분리**: CUD용 VO, 조회용 SelectVO
- **트랜잭션**: Service 계층에 `@Transactional`
- **인증/인가**: `@PreAuthorize` 어노테이션
- **유효성 검증**: Bean Validation + Spring Validator + Service 로직
- **응답 형식**: `ResponseEntity<T>`, RFC 7807 에러
- **공통 코드**: `CommonCodeService.selectCommonCodeName()` 사용, SQL JOIN 금지

### express (Node.js/TypeScript)
- **계층 구조**: Router → Controller → Service
- **ORM**: Sequelize / TypeORM / Prisma / Knex
- **미들웨어**: 인증, 유효성 검증, 에러 핸들링
- **응답 형식**: Express Response 래핑

### nestjs (TypeScript)
- **계층 구조**: Controller → Service → Repository
- **ORM**: TypeORM / Prisma / MikroORM
- **데코레이터**: `@Controller`, `@Injectable`, `@Get/@Post/@Put/@Delete`
- **Guard**: 인증/인가 Guard
- **Pipe**: 유효성 검증 Pipe (class-validator)
- **DTO**: class-validator + class-transformer

### django (Python)
- **계층 구조**: View → Serializer → Model
- **ORM**: Django ORM (QuerySet)
- **URL 라우팅**: `urls.py` + ViewSet
- **인증/인가**: Permission 클래스
- **직렬화**: DRF Serializer

## 8.2 프론트엔드 프레임워크 분기

### react (CRA/Vite)
- **컴포넌트**: 함수형 컴포넌트 + Hook
- **상태 관리**: Zustand / Redux Toolkit
- **서버 상태**: TanStack Query (useQuery, useMutation)
- **API 호출**: 커스텀 API 훅 (useApi 등 프로젝트 표준)
- **라우팅**: React Router v6+
- **폼 처리**: React Hook Form / 직접 관리
- **UI 라이브러리**: MUI v5/v6 / Ant Design

### vue (Vue 3)
- **컴포넌트**: SFC (Single File Component) + Composition API
- **상태 관리**: Pinia
- **서버 상태**: VueQuery / 직접 관리
- **라우팅**: Vue Router
- **폼 처리**: VeeValidate + Zod/Yup

### angular
- **컴포넌트**: Component + Template + Module
- **서비스**: Injectable Service (HttpClient)
- **상태 관리**: NgRx / Akita / Signal
- **폼 처리**: Reactive Forms + Validators
- **라우팅**: Angular Router

### next (Next.js)
- **라우팅**: App Router (app/) 또는 Pages Router (pages/)
- **컴포넌트**: Server Component (기본) + Client Component ('use client')
- **데이터**: Server Actions / Route Handlers / fetch
- **상태 관리**: Zustand / Jotai (Client Component 전용)
