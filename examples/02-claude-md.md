# 예제: EduHub의 CLAUDE.md

> 관련 모듈: 8
> 예제 프로젝트: [EduHub](./00-sample-project.md)

모노레포에서는 **루트에는 공통 규칙**, **하위 디렉터리에는 영역별 규칙**을 두는 구성이 효과적입니다.
Claude가 `backend/` 파일을 다룰 때는 `backend/CLAUDE.md`가 추가로 읽히므로, 프론트 작업에 백엔드 규칙이 섞이지 않습니다.

```
eduhub/
├── CLAUDE.md            ← 항상 읽힘: 프로젝트 개요, 공통 규칙
├── backend/CLAUDE.md    ← 백엔드 파일을 다룰 때 읽힘
└── frontend/CLAUDE.md   ← 프론트엔드 파일을 다룰 때 읽힘
```

---

## 1. 루트 `CLAUDE.md`

```markdown
# EduHub

사내 교육 신청 시스템. 백엔드(Spring Boot)와 프론트엔드(React 마이크로프론트엔드)로 구성된 모노레포.

## 구조
- `backend/`: Spring Boot 3, Java 21, JPA/Hibernate, PostgreSQL. 자세한 규칙은 backend/CLAUDE.md
- `frontend/`: React 18, Module Federation(shell + course-mfe + enrollment-mfe), MUI, Axios. 자세한 규칙은 frontend/CLAUDE.md
- API 계약: 백엔드 OpenAPI 어노테이션 → `/v3/api-docs` → 프론트 `packages/api-client`에 클라이언트 자동 생성

## 로컬 실행
- DB: `docker compose up -d postgres`
- 백엔드: `cd backend && ./gradlew bootRun` (Swagger: http://localhost:8080/swagger-ui.html)
- 프론트: `cd frontend && npm run dev` (shell: http://localhost:3000)

## 작업 규칙
- API를 추가·변경하면 반드시 프론트 API 클라이언트를 다시 생성한다: `cd frontend && npm run generate:api`
- 커밋 메시지는 Conventional Commits: `feat(enrollment): 수강 취소 API 추가`
- 백엔드와 프론트 변경은 커밋을 나눈다
- 작업을 마치기 전에 변경한 영역의 테스트를 실행해 통과를 확인한다

## 하지 말 것
- `frontend/packages/api-client/src/generated/`는 자동 생성 코드이므로 직접 수정하지 않는다
- `backend/src/main/resources/db/migration/`의 기존 Flyway 스크립트는 수정하지 않는다. 변경은 새 버전 파일로 추가한다
```

---

## 2. `backend/CLAUDE.md`

```markdown
# Backend 규칙

## 명령어
- 전체 테스트: `./gradlew test`
- 단일 테스트: `./gradlew test --tests "*EnrollmentServiceTest"`
- 포맷 적용: `./gradlew spotlessApply` (커밋 전 필수)
- 빌드: `./gradlew build`

## 패키지 구조
- 도메인별 패키지(`course`, `enrollment`) 안에 Entity, Repository, Service, Controller, dto를 둔다
- 공통 예외와 설정은 `common` 패키지

## JPA / Hibernate
- `@ManyToOne`, `@OneToOne`은 항상 `fetch = FetchType.LAZY`
- 컨트롤러에서 엔티티를 직접 반환하지 않는다. 반드시 DTO(record)로 변환한다
- `spring.jpa.open-in-view=false` 유지. 지연 로딩은 서비스 트랜잭션 안에서 해결한다
- 목록 조회는 N+1을 확인하고 `@EntityGraph` 또는 fetch join을 사용한다
- `ddl-auto`는 `validate`. 스키마 변경은 Flyway 마이그레이션(`V{번호}__{설명}.sql`)으로만 한다
- 엔티티에 `@Setter`를 쓰지 않는다. 상태 변경은 의미 있는 메서드(`cancel()`, `approve()`)로 한다

## API / OpenAPI
- 모든 컨트롤러에 `@Tag`, 모든 메서드에 `@Operation(summary=...)`과 실패 응답 `@ApiResponse`를 단다
- URL은 `/api/v1/{복수형 리소스}`
- 에러 응답은 `common.error.ErrorResponse` 형식, 예외는 `GlobalExceptionHandler`에서 처리

## 테스트
- 서비스: JUnit 5 + Mockito, 테스트 이름은 `메서드_조건_결과`, `@DisplayName`은 한국어
- 리포지토리: `@DataJpaTest` + Testcontainers(PostgreSQL)
- 테스트 데이터는 `fixture` 패키지의 Fixture 클래스를 사용
```

---

## 3. `frontend/CLAUDE.md`

```markdown
# Frontend 규칙

## 명령어
- 전체 개발 서버: `npm run dev` (shell + 모든 remote)
- 테스트: `npm test -w apps/course-mfe` (워크스페이스 단위)
- 린트: `npm run lint`
- API 클라이언트 생성: `npm run generate:api` (백엔드가 실행 중이어야 함)

## Module Federation
- `react`, `react-dom`, `@mui/material`, `@emotion/react`는 shared singleton. 버전을 바꿀 때는 shell과 모든 remote를 함께 바꾼다
- remote는 `./XxxApp` 하나만 expose하고, 내부 라우팅은 remote가 담당한다
- remote끼리 직접 import하지 않는다. 공유가 필요하면 `packages/`로 옮긴다

## API 호출
- Axios를 직접 생성하지 않는다. `@eduhub/api-client`의 `courseApi`, `enrollmentApi`를 사용한다
- 서버 상태는 커스텀 훅(`useCourses`, `useEnrollments`)으로 감싼다
- 에러 처리는 공통 인터셉터가 담당하므로 컴포넌트에서 중복 처리하지 않는다

## UI (MUI)
- 테마는 shell의 `theme.ts`만 사용한다. remote에서 `createTheme`을 새로 만들지 않는다
- 스타일은 `sx` prop 우선, 반복되면 `styled()`로 분리
- 로딩은 `CircularProgress`/`Skeleton`, 에러는 `Alert`, 알림은 전역 Snackbar 사용
```

---

## 4. 좋은 예와 나쁜 예

| ❌ 나쁜 예 | 문제 | ✅ 좋은 예 |
|---|---|---|
| "깔끔한 코드를 작성할 것" | 막연해서 행동이 바뀌지 않는다 | "엔티티에 `@Setter`를 쓰지 않는다. 상태 변경은 `cancel()` 같은 메서드로 한다" |
| "테스트를 잘 작성할 것" | 무엇을 어떻게 할지 모른다 | "단일 테스트: `./gradlew test --tests "*XxxTest"`" |
| DB 접속 정보 `password: eduhub1234` | 비밀정보가 Git에 공유된다 | "DB 접속 정보는 `.env`에 있다 (커밋 금지)" |
| JPA 공식 문서 내용을 길게 복사 | 컨텍스트 낭비, 중요한 규칙이 묻힌다 | 팀이 실제로 지키는 규칙 5~6줄만 |
| "들여쓰기는 4칸" | Spotless가 이미 강제한다 | 생략하고 "커밋 전 `spotlessApply`"만 적는다 |
| "API 추가 절차: 1단계 ... 15단계" | 매번 필요하지 않은 긴 절차 | 스킬로 분리 ([예제](./03-settings-skills-agents-hooks.md#모듈-10-스킬-add-api-endpoint)) |

---

## 5. 실습 체크리스트

- [ ] `/init`으로 초안을 만든 뒤 위 예시와 비교해 빠진 규칙, 불필요한 내용 정리
- [ ] CLAUDE.md 없이 "과정 목록 API 추가"를 시키고 결과 저장
- [ ] CLAUDE.md를 넣고 같은 지시를 다시 실행 → DTO 사용, LAZY, OpenAPI 어노테이션, 클라이언트 재생성 여부 비교
- [ ] 루트와 `backend/CLAUDE.md`에 일부러 충돌하는 규칙(예: 테스트 명명 규칙)을 넣고 결과 관찰
