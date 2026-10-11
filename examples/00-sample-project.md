# 실습 예제 프로젝트: 사내 교육 신청 시스템 (EduHub)

> 교육과정 전체 실습에서 공통으로 사용하는 예제 프로젝트입니다.
> 모든 모듈의 예시는 이 프로젝트를 기준으로 작성되어 있습니다.

---

## 1. 서비스 개요

임직원이 사내 교육과정을 조회하고 수강 신청하는 시스템입니다.

| 기능 | 설명 |
|---|---|
| 교육과정 관리 | 과정 등록·수정·조회, 회차(일정)와 정원 관리 |
| 수강 신청 | 신청, 취소, 대기자 등록, 정원 초과 체크 |
| 마이페이지 | 내 신청 내역, 수료 현황 |

예제 도메인으로 고른 이유:
- 엔티티 관계(1:N, N:1)가 적당히 있어 **JPA 연관관계, N+1 문제**를 실습하기 좋다
- 정원 체크, 중복 신청 방지 같은 **비즈니스 규칙**이 있어 TDD 실습에 적합하다
- 화면이 "과정"과 "신청"으로 자연스럽게 나뉘어 **마이크로프론트엔드** 구조를 설명하기 좋다

---

## 2. 기술 스택

| 영역 | 기술 | 비고 |
|---|---|---|
| 언어 / 런타임 | Java 21 | |
| 백엔드 프레임워크 | Spring Boot 3.x | Spring Web, Validation |
| 영속성 | Spring Data JPA, Hibernate 6 | DB 스키마는 Flyway로 관리 |
| DB | PostgreSQL (로컬은 Docker), 테스트는 Testcontainers | |
| API 문서 | OpenAPI 3 + Swagger UI (springdoc-openapi) | `/swagger-ui.html`, `/v3/api-docs` |
| 빌드 | Gradle (Kotlin DSL) | Spotless로 코드 포맷 |
| 프론트엔드 | React 18 + TypeScript | |
| 마이크로프론트엔드 | Webpack 5 Module Federation | shell(host) + 2개 remote |
| UI 라이브러리 | MUI (Material UI) | 공통 테마는 shell에서 제공 |
| HTTP 클라이언트 | Axios | OpenAPI 스펙으로 클라이언트 코드 자동 생성 |
| 테스트 | JUnit 5, Mockito, `@DataJpaTest` / Jest, React Testing Library | |

---

## 3. 아키텍처

```
┌──────────────────── 브라우저 ────────────────────┐
│  shell (host, :3000)                             │
│   ├─ 공통 레이아웃, MUI 테마, 라우팅, 인증          │
│   ├─ course-mfe (remote, :3001)  ← 교육과정 화면   │
│   └─ enrollment-mfe (remote, :3002) ← 수강신청 화면│
└───────────────┬──────────────────────────────────┘
                │ Axios (OpenAPI로 생성한 클라이언트)
                ▼
┌──────────── backend (Spring Boot, :8080) ────────┐
│  Controller (OpenAPI 어노테이션)                   │
│     → Service (트랜잭션, 비즈니스 규칙)             │
│        → Repository (Spring Data JPA)             │
│           → Hibernate → PostgreSQL                │
└──────────────────────────────────────────────────┘
```

**API 계약 흐름 (Contract-First가 아닌 Code-First 방식)**

1. 백엔드 컨트롤러에 OpenAPI 어노테이션 작성
2. springdoc이 `/v3/api-docs`로 스펙 생성, Swagger UI로 확인
3. 프론트엔드에서 `openapi-generator`로 TypeScript + Axios 클라이언트 생성
4. 각 MFE는 생성된 클라이언트를 공통 패키지에서 가져다 사용

> 💡 이 흐름 덕분에 "API를 바꾸면 프론트 타입까지 자동으로 따라오는" 구조가 됩니다. Claude에게 기능 추가를 맡길 때 **백엔드 → 스펙 → 프론트**를 한 번에 처리하게 만들 수 있어, 스킬(모듈 10) 예제로 쓰기 좋습니다.

---

## 4. 디렉터리 구조

```
eduhub/
├── CLAUDE.md                     # 공통 지침 (모듈 8)
├── .claude/
│   ├── settings.json             # 권한, 훅 (모듈 9, 12)
│   ├── skills/                   # 스킬 (모듈 10)
│   └── agents/                   # 서브에이전트 (모듈 11)
├── .mcp.json                     # MCP 서버 (모듈 13)
├── backend/
│   ├── CLAUDE.md                 # 백엔드 전용 지침
│   ├── build.gradle.kts
│   └── src/
│       ├── main/java/com/example/eduhub/
│       │   ├── course/           # 교육과정 도메인
│       │   │   ├── Course.java
│       │   │   ├── CourseSession.java
│       │   │   ├── CourseRepository.java
│       │   │   ├── CourseService.java
│       │   │   ├── CourseController.java
│       │   │   └── dto/
│       │   ├── enrollment/       # 수강신청 도메인
│       │   └── common/           # 예외 처리, 공통 설정
│       ├── main/resources/
│       │   ├── application.yml
│       │   └── db/migration/     # Flyway 스크립트
│       └── test/java/...
└── frontend/
    ├── CLAUDE.md                 # 프론트엔드 전용 지침
    ├── packages/
    │   └── api-client/           # OpenAPI로 생성한 Axios 클라이언트
    └── apps/
        ├── shell/                # host
        ├── course-mfe/           # remote
        └── enrollment-mfe/       # remote
```

---

## 5. 핵심 코드 예시

### 5-1. JPA 엔티티

```java
@Entity
@Table(name = "course_session")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CourseSession {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)          // N:1은 항상 LAZY
    @JoinColumn(name = "course_id", nullable = false)
    private Course course;

    @Column(nullable = false)
    private LocalDate startDate;

    @Column(nullable = false)
    private int capacity;

    @OneToMany(mappedBy = "session")
    private List<Enrollment> enrollments = new ArrayList<>();

    public boolean isFull() {
        return enrollments.stream().filter(Enrollment::isActive).count() >= capacity;
    }
}
```

### 5-2. 컨트롤러와 OpenAPI 어노테이션

```java
@RestController
@RequestMapping("/api/v1/courses")
@RequiredArgsConstructor
@Tag(name = "Course", description = "교육과정 API")
public class CourseController {

    private final CourseService courseService;

    @Operation(summary = "교육과정 목록 조회", description = "카테고리로 필터링하고 페이지 단위로 조회합니다.")
    @ApiResponse(responseCode = "200", description = "조회 성공")
    @GetMapping
    public Page<CourseSummaryResponse> getCourses(
            @Parameter(description = "카테고리 코드") @RequestParam(required = false) String category,
            @ParameterObject Pageable pageable) {
        return courseService.getCourses(category, pageable);
    }

    @Operation(summary = "교육과정 상세 조회")
    @ApiResponse(responseCode = "404", description = "과정을 찾을 수 없음",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    @GetMapping("/{courseId}")
    public CourseDetailResponse getCourse(@PathVariable Long courseId) {
        return courseService.getCourse(courseId);
    }
}
```

### 5-3. 프론트엔드: 생성된 클라이언트와 Axios 설정

```bash
# 백엔드 실행 후, 스펙으로 TypeScript + Axios 클라이언트 생성
npx @openapitools/openapi-generator-cli generate \
  -i http://localhost:8080/v3/api-docs \
  -g typescript-axios \
  -o packages/api-client/src/generated
```

```ts
// packages/api-client/src/http.ts
import axios from "axios";
import { Configuration, CourseApi, EnrollmentApi } from "./generated";

export const http = axios.create({
  baseURL: process.env.API_BASE_URL,
  timeout: 10_000,
});

http.interceptors.request.use((config) => {
  const token = sessionStorage.getItem("accessToken");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

http.interceptors.response.use(
  (res) => res,
  (error) => {
    if (error.response?.status === 401) window.dispatchEvent(new Event("auth:expired"));
    return Promise.reject(error);
  },
);

const config = new Configuration();
export const courseApi = new CourseApi(config, undefined, http);
export const enrollmentApi = new EnrollmentApi(config, undefined, http);
```

### 5-4. Module Federation 설정

```js
// apps/shell/webpack.config.js (host)
const { ModuleFederationPlugin } = require("webpack").container;

new ModuleFederationPlugin({
  name: "shell",
  remotes: {
    courseMfe: "courseMfe@http://localhost:3001/remoteEntry.js",
    enrollmentMfe: "enrollmentMfe@http://localhost:3002/remoteEntry.js",
  },
  shared: {
    react: { singleton: true, requiredVersion: "^18.0.0" },
    "react-dom": { singleton: true, requiredVersion: "^18.0.0" },
    "@mui/material": { singleton: true },
    "@emotion/react": { singleton: true },
  },
});
```

```js
// apps/course-mfe/webpack.config.js (remote)
new ModuleFederationPlugin({
  name: "courseMfe",
  filename: "remoteEntry.js",
  exposes: { "./CourseApp": "./src/CourseApp" },
  shared: { /* shell과 동일 */ },
});
```

### 5-5. MUI 화면

```tsx
// apps/course-mfe/src/pages/CourseListPage.tsx
export default function CourseListPage() {
  const { data, isLoading, error } = useCourses({ page: 0, size: 20 });

  if (isLoading) return <CircularProgress />;
  if (error) return <Alert severity="error">교육과정을 불러오지 못했습니다.</Alert>;

  return (
    <Grid container spacing={2}>
      {data?.content.map((course) => (
        <Grid item xs={12} md={4} key={course.id}>
          <Card>
            <CardContent>
              <Typography variant="h6">{course.title}</Typography>
              <Chip label={course.category} size="small" />
            </CardContent>
          </Card>
        </Grid>
      ))}
    </Grid>
  );
}
```

---

## 6. 의도적으로 심어 둔 "실습용 문제"

실습에서 Claude와 함께 찾아 고치도록 예제 코드에 일부러 남겨 둔 문제입니다.

| 번호 | 문제 | 관련 기술 | 실습 모듈 |
|---|---|---|---|
| P1 | 과정 목록 조회 시 **N+1 쿼리** 발생 | JPA, Hibernate | 모듈 4, 6 |
| P2 | 컨트롤러에서 엔티티를 직렬화해 **LazyInitializationException** 발생 | Hibernate, Jackson | 모듈 6 |
| P3 | 동시에 신청하면 **정원 초과** (동시성 문제) | JPA 락, 트랜잭션 | 모듈 6 |
| P4 | 일부 API에 OpenAPI 어노테이션 누락 → 생성된 클라이언트 타입이 `any` | springdoc, OpenAPI Generator | 모듈 10 |
| P5 | remote마다 MUI 버전이 달라 **테마가 깨짐** | Module Federation shared | 모듈 4, 11 |
| P6 | Axios 에러 처리가 MFE마다 제각각 | Axios 인터셉터 | 모듈 6 |
