# 모범 사례: Java(Spring Boot) + React 프로젝트에서 자주 만드는 서브에이전트

> 관련 모듈: 11
> 예제 프로젝트: [EduHub](./00-sample-project.md) / 에이전트 기본 개념은 [에이전트 가이드](./10-agents.md) 참고

> 💡 여기 소개하는 에이전트는 Java + React 팀에서 흔히 쓰는 **역할 패턴**입니다. 처음부터 전부 만들 필요는 없습니다. 아래 "시작 세트"부터 만들고, 반복해서 시키는 일이 생길 때 하나씩 추가하세요.

---

## 1. 한눈에 보는 에이전트 카탈로그

| 분류 | 에이전트 | 하는 일 | 수정 권한 | 추천 모델 |
|---|---|---|---|---|
| **탐색** | `code-explorer` | 코드 위치, 호출 흐름(React → API → Service → DB) 찾기 | ✕ 읽기 전용 | 빠른 모델 |
| **설계** | `architect` | 요구사항 → API 설계, 엔티티 변경안, 작업 분할 | ✕ | 고성능 모델 |
| **구현** | `backend-dev` | Spring Boot 계층 구현, Flyway | ○ `backend/` | 기본 |
| | `frontend-dev` | React 화면, 훅, MUI, API 클라이언트 연결 | ○ `frontend/` | 기본 |
| **테스트** | `backend-test-writer` | JUnit 5, Mockito, `@DataJpaTest`, Testcontainers | ○ 테스트 폴더 | 기본 |
| | `frontend-test-writer` | Jest/Vitest, React Testing Library, API 모킹 | ○ 테스트 파일 | 기본 |
| | `e2e-tester` | Playwright로 화면 시나리오 검증 | ○ e2e 폴더 | 기본 |
| **리뷰** | `jpa-reviewer` | N+1, fetch 전략, 트랜잭션, 락 | ✕ | 기본 |
| | `spring-api-reviewer` | REST 설계, 검증, 예외 처리, OpenAPI 문서 | ✕ | 기본 |
| | `react-reviewer` | 훅 규칙, 리렌더링, 상태 관리, 접근성 | ✕ | 기본 |
| | `mfe-integration-checker` | Module Federation shared 버전, 테마, remote 경계 | ✕ | 기본 |
| | `security-reviewer` | 인증·인가, 입력 검증, XSS, 비밀정보 노출, 의존성 취약점 | ✕ | 고성능 모델 |
| **변경 관리** | `api-contract-checker` | API 변경의 하위 호환성, 프론트 영향 범위 | ✕ | 기본 |
| | `migration-reviewer` | Flyway 스크립트의 잠금·대용량 테이블 영향, 롤백 | ✕ | 고성능 모델 |
| | `dependency-upgrader` | Spring Boot, React, MUI 버전 업그레이드 | ○ | 기본 |
| **분석** | `build-fixer` | Gradle·npm 빌드 실패, 타입 오류 원인 분석과 수정 | ○ | 기본 |
| | `perf-analyzer` | 느린 API(쿼리), 무거운 번들·렌더링 분석 | ✕ | 기본 |
| **문서** | `doc-writer` | PR 본문, 릴리스 노트, API 변경 안내 | ○ `docs/` | 빠른 모델 |

---

## 2. 시작 세트: 먼저 만들 3개

| 순서 | 에이전트 | 이유 |
|---|---|---|
| 1 | `code-explorer` | 가장 자주 쓰이고, 메인 컨텍스트를 아끼는 효과가 바로 보인다 |
| 2 | `jpa-reviewer` (또는 팀의 가장 큰 고민 분야 리뷰어) | 구현과 분리된 검토로 품질 효과가 크다. 읽기 전용이라 안전 |
| 3 | `backend-test-writer` | 테스트 작성은 반복적이고 검증 기준이 분명해 위임하기 좋다 |

그다음 팀 상황에 따라: 프론트 비중이 크면 `react-reviewer`, MFE 구조면 `mfe-integration-checker`, 보안 요구가 높으면 `security-reviewer`.

---

## 3. 주요 에이전트 정의 예시

### 3-1. `code-explorer` (탐색, 읽기 전용, 빠른 모델)

```markdown
---
name: code-explorer
description: Java Spring Boot + React 코드베이스에서 코드 위치, 호출 흐름, 사용처를 찾을 때 사용한다. 화면 → Axios 호출 → Controller → Service → Repository → 테이블까지 추적한다. 넓은 탐색이 필요하면 적극적으로 사용한다.
tools: Read, Grep, Glob
model: haiku
---

너는 코드 탐색 전문가다. 코드는 수정하지 않는다.

## 탐색 요령
- 백엔드: `@RestController`, `@RequestMapping`, `@Service`, `JpaRepository` 상속, `@Entity`로 검색
- 프론트: 생성된 API 클라이언트 메서드 이름, `use*` 훅, 라우트 설정으로 검색
- 생성 코드(`generated/`), `node_modules/`, `build/`는 읽지 않는다

## 보고 형식
- 흐름을 단계별로: `파일경로:줄번호 — 역할`
- 마지막에 핵심 파일 5개 이내 목록
- 추측한 부분은 "추정"이라고 표시
```

### 3-2. `spring-api-reviewer` (리뷰, 읽기 전용)

```markdown
---
name: spring-api-reviewer
description: Spring Boot REST API 변경을 검토한다. 컨트롤러·DTO·예외 처리·검증·OpenAPI 문서가 바뀐 뒤 사용한다.
tools: Read, Grep, Glob
---

너는 Spring Boot API 설계 리뷰어다. backend/CLAUDE.md 규칙을 먼저 읽는다.

## 검토 항목
1. URL·HTTP 메서드·상태 코드가 REST 규칙과 팀 규칙에 맞는가
2. 요청 DTO에 Bean Validation이 있고, 컨트롤러에 `@Valid`가 있는가
3. 엔티티를 직접 반환하지 않는가 (DTO 변환)
4. 예외가 `GlobalExceptionHandler`의 공통 에러 형식으로 처리되는가
5. `@Operation`, 실패 응답 `@ApiResponse`가 있는가
6. 기존 API 응답 필드를 삭제·변경해 프론트를 깨뜨리지 않는가

## 보고 형식
심각/경고/제안으로 나누고, `파일:줄`, 문제, 수정 예시 코드를 제시한다. 코드는 직접 수정하지 않는다.
```

### 3-3. `react-reviewer` (리뷰, 읽기 전용)

```markdown
---
name: react-reviewer
description: React(TypeScript) 컴포넌트와 훅 변경을 검토한다. 화면·훅·상태 관리 코드가 바뀐 뒤 사용한다.
tools: Read, Grep, Glob
---

너는 React 코드 리뷰어다. frontend/CLAUDE.md 규칙을 먼저 읽는다.

## 검토 항목
1. 훅 규칙 위반, `useEffect` 의존성 배열 누락·과다
2. 불필요한 리렌더링 (인라인 객체·함수, 큰 컴포넌트 분리 필요 여부)
3. 서버 상태와 UI 상태의 혼용, API 호출이 컴포넌트에 직접 있는지 (커스텀 훅 사용 규칙)
4. 로딩·에러·빈 상태 처리 누락
5. MUI 사용 규칙: 사내 테마 사용, `sx` 우선, 하드코딩된 색상·크기
6. 접근성: 버튼·입력의 레이블, 키보드 조작
7. `any` 타입, 생성된 API 타입 대신 직접 정의한 타입

## 보고 형식
심각/경고/제안, `파일:줄`, 수정 예시. 코드는 수정하지 않는다.
```

### 3-4. `backend-test-writer` (테스트, 수정 가능)

```markdown
---
name: backend-test-writer
description: Spring Boot 서비스·리포지토리·컨트롤러 테스트를 작성하고 실행한다. 백엔드 구현이 끝난 뒤 테스트 보강이 필요할 때 사용한다.
tools: Read, Grep, Glob, Write, Edit, Bash
---

너는 Java 테스트 전문가다. 수정은 `backend/src/test/` 안에서만 한다.

## 계층별 테스트 선택
- Service: JUnit 5 + Mockito (빠른 단위 테스트)
- Repository: `@DataJpaTest` + Testcontainers(PostgreSQL). 쿼리 수 검증이 필요하면 Hibernate Statistics 사용
- Controller: `@WebMvcTest` + MockMvc (요청 검증, 상태 코드, 에러 응답 형식)

## 규칙
- 성공, 실패·예외, 경계값을 모두 다룬다
- Fixture 클래스를 사용하고, 테스트 이름은 `메서드_조건_결과`, `@DisplayName`은 한국어
- `./gradlew test --tests "{클래스}" -q`로 실행해 통과 확인
- 구현 버그가 의심되면 테스트를 맞추지 말고 보고한다

## 보고 형식
추가한 테스트 파일과 개수, 실행 결과, 의심되는 구현 문제
```

### 3-5. `security-reviewer` (리뷰, 읽기 전용, 고성능 모델)

```markdown
---
name: security-reviewer
description: Spring Boot + React 변경의 보안 문제를 검토한다. 인증·인가, 입력 처리, 파일 업로드, 외부 연동, 의존성 변경이 있을 때 사용한다.
tools: Read, Grep, Glob, Bash
model: opus
---

너는 웹 애플리케이션 보안 리뷰어다. 코드는 수정하지 않는다.

## 백엔드
- 엔드포인트별 인가 누락 (`@PreAuthorize`, 보안 설정), 다른 사용자의 데이터 접근(IDOR)
- 네이티브 쿼리·JPQL 문자열 결합(SQL 인젝션), 입력 검증 누락
- 예외 메시지·로그에 개인정보·비밀정보 노출
- CORS, CSRF 설정 변경

## 프론트엔드
- `dangerouslySetInnerHTML`, URL 파라미터를 그대로 렌더링(XSS)
- 토큰 저장 방식, 콘솔 로그에 민감 정보

## 의존성
- 새로 추가·변경된 라이브러리의 알려진 취약점 여부를 빌드 도구로 확인

## 보고 형식
위험도(높음/중간/낮음), 공격 시나리오, `파일:줄`, 수정 방안
```

### 3-6. `build-fixer` (분석과 수정)

```markdown
---
name: build-fixer
description: Gradle 빌드, Java 컴파일, npm 빌드, TypeScript 타입 오류가 발생했을 때 원인을 분석하고 수정한다.
tools: Read, Grep, Glob, Edit, Bash
---

너는 빌드 문제 해결 전문가다.
1. 실패한 명령을 다시 실행해 오류를 재현한다 (출력은 마지막 100줄만 확인)
2. 첫 번째 오류부터 원인을 찾는다 (뒤따르는 오류는 연쇄 오류일 수 있음)
3. 최소한의 수정으로 고치고 같은 명령으로 확인한다
4. 의존성 버전 변경이 필요하면 수정하지 말고 제안만 한다
```

---

## 4. 이렇게 조합해서 쓴다

### 4-1. 기능 개발 흐름

```
code-explorer (관련 코드 파악)
   → architect (설계, 사람이 승인)
   → backend-dev → [frontend-dev ∥ backend-test-writer]
   → [spring-api-reviewer ∥ jpa-reviewer ∥ react-reviewer]   ← 병렬 검토
   → 지적 사항 수정 → doc-writer (PR 본문)
```

자세한 흐름과 지시문 예시는 [에이전트 가이드의 역할 분담](./10-agents.md#7-서브에이전트로-역할-분담하기-ai-개발팀-구성)을 참고하세요.

### 4-2. 상황별 추천 조합

| 상황 | 조합 |
|---|---|
| 낯선 코드에 버그 수정 | `code-explorer` → (메인이 수정) → `backend-test-writer` |
| API 변경 PR 전 | `spring-api-reviewer` ∥ `api-contract-checker` |
| 화면 기능 추가 | `frontend-dev` → `react-reviewer` ∥ `frontend-test-writer` |
| DB 스키마 변경 | `migration-reviewer` ∥ `jpa-reviewer` |
| 릴리스 전 점검 | `security-reviewer` ∥ `perf-analyzer` |
| Spring Boot·React 메이저 업그레이드 | `dependency-upgrader` → `build-fixer` → 전체 테스트 |
| CI 빌드가 깨졌을 때 | `build-fixer` |

---

## 5. Java + React 팀을 위한 에이전트 운영 모범 사례

| 원칙 | 설명 |
|---|---|
| **리뷰어는 읽기 전용** | `Read, Grep, Glob`만. 수정은 메인이나 구현 에이전트가 |
| **구현 에이전트는 영역을 나눔** | `backend-dev`는 `backend/`, `frontend-dev`는 `frontend/`만 → 병렬 실행 시 충돌 방지 |
| **팀 규칙은 CLAUDE.md에, 에이전트는 참조** | 에이전트마다 규칙을 복사하지 말고 "backend/CLAUDE.md를 먼저 읽는다"로 연결 |
| **검증 명령을 명시** | `./gradlew test --tests ...`, `npm test -w ...`처럼 정확한 명령 |
| **보고 형식 통일** | 심각/경고/제안 + `파일:줄` + 수정 예시 → 메인이 모아서 처리하기 쉬움 |
| **모델은 역할에 맞게** | 탐색·문서는 빠른 모델, 보안·마이그레이션은 고성능 모델 |
| **스킬과 함께** | 에이전트는 "누가", 스킬은 "어떻게". 예: `backend-dev`가 `add-api-endpoint` 스킬을 따라 구현 ([스킬](./09-skills-catalog.md)) |
| **팀 공유와 버전 관리** | `.claude/agents/`를 커밋, CODEOWNERS로 리뷰. 여러 팀이 쓰면 플러그인으로 배포 |
| **효과 측정** | 리뷰어가 찾은 문제 수, 사람 리뷰에서 추가로 나온 문제 수를 비교해 에이전트 지침 개선 |

### 피해야 할 패턴

| 안티 패턴 | 문제 |
|---|---|
| `java-expert`, `react-expert`처럼 범위가 너무 넓은 에이전트 | description이 모호해 위임 판단이 어렵고, 결과도 일반적 |
| 리뷰어에게 수정 권한 부여 | 검토와 수정이 섞여 무엇이 바뀌었는지 추적 어려움 |
| 처음부터 에이전트 15개 | 관리 부담, 비슷한 역할끼리 혼동. 3개로 시작 |
| 에이전트 정의에 팀 규칙 전체 복사 | 규칙이 바뀌면 여러 곳을 고쳐야 함 |

---

## 6. 참고: 공개된 예시

- [piomin/claude-ai-spring-boot](https://github.com/piomin/claude-ai-spring-boot): Spring Boot용 Claude Code 템플릿으로, 코드 리뷰·JPA 패턴 등의 에이전트와 스킬 예시를 포함 (커뮤니티 자료, 사용 전 검토 필요)
- [obra/superpowers](https://github.com/obra/superpowers): 서브에이전트 중심 개발(`subagent-driven-development`), 병렬 에이전트 배분(`dispatching-parallel-agents`) 방법론 스킬

---

## 7. 실습

1. 시작 세트 3개(`code-explorer`, `jpa-reviewer`, `backend-test-writer`)를 EduHub에 만들기
2. "수강 신청 흐름 찾아줘"를 메인에게 직접 시킬 때와 `code-explorer`에게 맡길 때 `/context` 사용량 비교
3. 수강 취소 기능 PR에 대해 `spring-api-reviewer`, `jpa-reviewer`, `react-reviewer`를 병렬 실행하고 결과를 표로 정리
4. 팀 토론: 우리 팀이 다음에 추가할 에이전트 1개와 그 이유
