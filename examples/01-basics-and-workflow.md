# 예제: 에이전트 동작 원리, 기본 사용, 실무 워크플로우

> 관련 모듈: 1, 3, 4, 5, 6, 7
> 예제 프로젝트: [EduHub](./00-sample-project.md)

---

## 모듈 1. 에이전트 루프 관찰하기

**과제:** "수강 신청 시 정원이 찼으면 예외를 던지도록 `EnrollmentService`를 수정해줘"

Claude Code가 실제로 거치는 단계(도구 호출 기록)를 따라가 봅니다.

| 단계 | 에이전트의 행동 | 사용 도구 |
|---|---|---|
| 1. 탐색 | `EnrollmentService`, `CourseSession` 파일을 찾는다 | 파일 검색 (Glob, Grep) |
| 2. 이해 | 서비스와 엔티티 코드를 읽고 `isFull()`이 이미 있는지 확인한다 | 파일 읽기 (Read) |
| 3. 계획 | "예외 클래스 추가 → 서비스 수정 → 테스트 추가" 순서로 정한다 | (모델의 추론) |
| 4. 실행 | `CapacityExceededException` 생성, 서비스에 체크 로직 추가 | 파일 수정 (Edit, Write) |
| 5. 검증 | `./gradlew test --tests "*EnrollmentServiceTest"` 실행 | 명령 실행 (Bash) |
| 6. 반복 | 테스트가 실패하면 원인을 읽고 3~5단계를 다시 수행 | |

**토론 포인트**
- Claude는 처음부터 코드를 다 알고 있었나? → 아니다. **도구로 읽어서** 알게 된다
- 테스트 명령을 어떻게 알았을까? → `build.gradle.kts`를 읽었거나 CLAUDE.md에 적혀 있었다 (모듈 8로 연결)
- 6단계가 없다면? → 검증 없는 코드가 된다. **검증 수단을 주는 것**이 에이전트 활용의 핵심이다

---

## 모듈 3. 환경별 실습 과제

| 환경 | 과제 |
|---|---|
| 터미널 | `cd backend && claude` 실행 후 "Gradle 의존성 중 사용하지 않는 것을 찾아줘" |
| IDE (IntelliJ / VS Code) | `CourseController.java`를 열고 메서드를 선택한 뒤 "이 API의 OpenAPI 어노테이션을 보완해줘" → diff 확인 |
| 웹 (클라우드) | "enrollment 도메인 전체에 단위 테스트를 추가하고 PR을 만들어줘"를 맡겨 두고 다른 실습 진행 |
| 모바일 | 웹에서 맡긴 작업의 진행 상황 확인, 질문에 답하기 |

> ⚠️ 클라우드 환경에서는 로컬 Docker의 PostgreSQL에 접근할 수 없습니다. Testcontainers나 H2로 테스트하도록 설정 스크립트를 준비하거나, DB가 필요한 작업은 로컬 환경에서 진행합니다.

---

## 모듈 4. 코드베이스 탐색 질문 예시

```text
이 프로젝트의 백엔드 구조를 설명해줘. 도메인별 패키지와 계층(Controller, Service, Repository)의 역할을 정리해줘.
```

```text
Course, CourseSession, Enrollment 엔티티의 연관관계를 ERD 형태로 그려줘. fetch 전략도 함께 표시해줘.
```

```text
수강 신청 버튼을 누르면 어떤 일이 일어나는지 프론트엔드(enrollment-mfe)부터 DB까지 호출 흐름을 따라가 설명해줘.
```

```text
과정 목록 API를 호출하면 쿼리가 몇 번 실행되는지 분석해줘. N+1 문제가 있는지 확인해줘.  (→ 문제 P1)
```

```text
shell과 각 remote의 Module Federation shared 설정을 비교해서 버전 충돌 가능성이 있는지 알려줘.  (→ 문제 P5)
```

---

## 모듈 5. 프롬프트 비교: 막연한 지시 vs 구조화된 지시

**❌ 막연한 지시**

```text
수강 취소 기능 만들어줘
```

**✅ 구조화된 지시**

```text
[목표] 수강 취소 API와 화면을 추가해줘.

[맥락]
- 백엔드: enrollment 도메인. 기존 신청 API(POST /api/v1/enrollments) 구조를 따라줘.
- 프론트: enrollment-mfe의 "내 신청 내역" 화면에 취소 버튼 추가.

[규칙]
- 교육 시작 3일 전까지만 취소 가능. 그 이후에는 409 응답.
- 취소하면 대기자 1순위를 자동으로 신청 상태로 바꿔줘.
- API는 DELETE /api/v1/enrollments/{id}, OpenAPI 어노테이션 필수.
- 프론트는 생성된 API 클라이언트를 다시 생성해서 사용하고, MUI Dialog로 확인 창을 띄워줘.

[완료 기준]
- 서비스 단위 테스트(취소 가능, 기한 초과, 대기자 승격) 통과
- ./gradlew test, npm test 모두 통과
- Swagger UI에서 새 API가 보일 것

먼저 계획을 보여주고, 내가 승인하면 구현해줘.
```

**Plan 모드 실습:** 위 지시를 Plan 모드로 실행해 계획을 받아 보고, "대기자 승격은 별도 이벤트로 분리해줘"처럼 계획 단계에서 수정한 뒤 구현을 승인합니다.

---

## 모듈 6. 기능 개발 & 버그 수정

### 6-1. TDD: 실패하는 테스트부터

```java
@ExtendWith(MockitoExtension.class)
class EnrollmentServiceTest {

    @Test
    @DisplayName("정원이 찬 회차에 신청하면 CapacityExceededException이 발생한다")
    void enroll_whenSessionIsFull_throwsException() {
        // given
        CourseSession fullSession = CourseSessionFixture.withCapacity(1).withActiveEnrollments(1);
        given(sessionRepository.findById(1L)).willReturn(Optional.of(fullSession));

        // when & then
        assertThatThrownBy(() -> enrollmentService.enroll(1L, MEMBER_ID))
                .isInstanceOf(CapacityExceededException.class);
    }
}
```

```text
EnrollmentServiceTest에 테스트를 추가해 두었어. 이 테스트가 통과하도록 구현해줘.
테스트 코드는 수정하지 말고, 통과할 때까지 ./gradlew test --tests "*EnrollmentServiceTest"를 반복 실행해줘.
```

### 6-2. 버그 수정 시나리오

| 문제 | Claude에게 줄 지시 예시 | 기대하는 해결 방향 |
|---|---|---|
| P1 N+1 | "과정 목록 조회의 N+1 문제를 고쳐줘. 수정 전후 실행 쿼리 수를 테스트로 검증해줘" | `@EntityGraph` 또는 fetch join, `@DataJpaTest` + Hibernate Statistics로 쿼리 수 검증 |
| P2 LazyInitializationException | "과정 상세 API에서 나는 에러 로그야: (로그 붙여넣기). 원인과 해결책을 설명하고 고쳐줘" | 엔티티 대신 DTO 반환, `spring.jpa.open-in-view=false` 유지 |
| P3 동시성 | "동시에 100명이 신청해도 정원을 넘지 않도록 고쳐줘. 동시성 테스트도 작성해줘" | 비관적 락(`@Lock(PESSIMISTIC_WRITE)`) 또는 낙관적 락(`@Version`), `ExecutorService` 기반 테스트 |
| P6 Axios 에러 처리 | "MFE마다 다른 Axios 에러 처리를 api-client의 인터셉터로 통합하고, MUI Snackbar로 공통 표시해줘" | 공통 인터셉터 + 에러 이벤트 + shell의 전역 Snackbar |

> 💡 에러 로그, 스택 트레이스, 실패한 테스트 출력은 **그대로 붙여넣는 것**이 가장 좋은 컨텍스트입니다.

---

## 모듈 7. Git & 협업 워크플로우

```text
이슈 #42 "수강 취소 기능"을 처리해줘.
feature/42-cancel-enrollment 브랜치를 만들고, 백엔드와 프론트를 각각 별도 커밋으로 나눠줘.
커밋 메시지는 Conventional Commits 형식(feat(enrollment): ...)으로 작성하고, 마지막에 PR을 만들어줘.
PR 본문에는 변경 요약, API 변경 사항(Swagger 경로), 테스트 방법을 넣어줘.
```

**git worktree로 병렬 작업**

```bash
git worktree add ../eduhub-backend-fix  -b fix/n-plus-one
git worktree add ../eduhub-mui-theme    -b fix/mfe-theme

# 터미널 1
cd ../eduhub-backend-fix && claude   # N+1 문제 수정 (P1)
# 터미널 2
cd ../eduhub-mui-theme && claude     # MFE 테마 문제 수정 (P5)
```
