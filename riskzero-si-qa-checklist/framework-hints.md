# 프레임워크별 파일 탐색 힌트

SKILL.md Step 1(대상 화면 식별)에서 관련 파일을 찾을 때, 프로젝트의 프레임워크에 해당하는 섹션만 참고한다.

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
