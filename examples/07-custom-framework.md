# 예제: 사내 커스텀 프레임워크 적용

> 관련 모듈: 16 (8, 10, 11, 12, 13, 14를 종합)
> 예제 프로젝트: [EduHub](./00-sample-project.md)를 사내 프레임워크 기반으로 바꾼 버전

---

## 1. 문제: Claude는 우리 회사 프레임워크를 모른다

Claude는 Spring Boot, React, MUI처럼 **공개된 기술**은 잘 알지만, 사내 프레임워크는 학습한 적이 없습니다.
아무 준비 없이 맡기면 다음과 같은 일이 생깁니다.

- 사내 공통 클래스 대신 **표준 Spring 방식**으로 코드를 작성한다
- 사내 UI 컴포넌트 대신 **MUI를 직접** 사용한다
- 존재하지 않는 사내 API를 **그럴듯하게 지어낸다**

해결의 핵심은 **"사내 프레임워크 지식을 에이전트가 읽을 수 있는 형태로 제공하고, 규칙은 도구로 강제하는 것"**입니다.

---

## 2. 예시: Acme 사내 프레임워크

### 백엔드: Acme Backend Framework (Spring Boot 기반)

| 표준 방식 | 사내 프레임워크 방식 |
|---|---|
| `@Service` | `@AcmeService` (감사 로그, 권한 체크 자동 적용) |
| `@Transactional` | `@AcmeTx(type = COMMAND)` / `@AcmeTx(type = QUERY)` |
| `ResponseEntity<T>` 반환 | `AcmeResponse<T>` 반환 (공통 응답 형식) |
| `throw new IllegalArgumentException()` | `throw new AcmeBizException(ErrorCode.ENROLL_001)` |
| `JpaRepository` 직접 상속 | `AcmeRepository<E, ID>` 상속 (논리 삭제, 감사 컬럼 자동 처리) |
| 엔티티에 생성일·수정자 직접 작성 | `extends AcmeBaseEntity` |

### 프론트엔드: Acme Frontend Framework (React + MUI 기반)

| 표준 방식 | 사내 프레임워크 방식 |
|---|---|
| `axios.create()` | `@acme/http`의 `acmeClient` (인증, 에러 처리, 로딩 표시 내장) |
| `@mui/material`의 `DataGrid`, `TextField` | `@acme/ui-kit`의 `AcmeGrid`, `AcmeForm` (사내 디자인 시스템) |
| 직접 Module Federation 설정 | `@acme/mfe-runtime`의 `defineRemote()`, `registerRemote()` |
| 라우트 직접 등록 | `pages.config.ts`에 화면 ID 등록 (권한 메뉴와 연동) |

---

## 3. 적용 전략: 6단계로 지식과 규칙을 제공하기

```
① CLAUDE.md       → 항상 알아야 하는 핵심 규칙과 "표준 → 사내" 매핑
② 스킬            → 상세 사용법, 템플릿, 골든 예제 (필요할 때만 로딩)
③ 참조 코드        → "이렇게 만들면 된다"는 모범 구현
④ 문서·소스 접근   → 프레임워크 소스, 사내 문서 MCP
⑤ 강제            → 훅, ArchUnit, ESLint로 규칙 위반 차단
⑥ 배포            → 프레임워크 팀이 플러그인으로 묶어 전사 배포
```

### ① CLAUDE.md: 짧은 핵심 규칙

```markdown
## 사내 프레임워크 (필수)
이 프로젝트는 Acme Framework 기반이다. 표준 Spring/MUI 대신 반드시 사내 프레임워크를 사용한다.
- 서비스: `@AcmeService`, 트랜잭션: `@AcmeTx`, 응답: `AcmeResponse<T>`, 예외: `AcmeBizException(ErrorCode.XXX)`
- 엔티티: `AcmeBaseEntity` 상속, 리포지토리: `AcmeRepository` 상속
- 프론트: HTTP는 `@acme/http`, UI는 `@acme/ui-kit`, MFE는 `@acme/mfe-runtime`
- 사용법을 모르면 추측하지 말고 `acme-backend`, `acme-frontend` 스킬 또는 `reference/` 코드를 확인한다
- 존재 여부가 불확실한 사내 API는 사용하지 말고 사용자에게 질문한다
```

> 💡 상세 설명은 넣지 않습니다. CLAUDE.md는 **"무엇을 써야 하는지"와 "어디를 보면 되는지"**만 알려줍니다.

### ② 스킬: 상세 가이드와 템플릿

```
.claude/skills/acme-backend/
├── SKILL.md                  # 언제 무엇을 읽어야 하는지 안내
├── guides/
│   ├── service-and-tx.md     # @AcmeService, @AcmeTx 상세 규칙
│   ├── error-codes.md        # ErrorCode 체계와 추가 방법
│   └── repository.md         # 논리 삭제, 감사 컬럼 동작
└── templates/
    ├── CrudService.java.md
    └── Controller.java.md
```

```markdown
---
name: acme-backend
description: Acme 사내 백엔드 프레임워크로 Service, Controller, Repository, Entity를 작성하거나 수정할 때 사용한다. @AcmeService, @AcmeTx, AcmeResponse, ErrorCode 사용법과 템플릿을 제공한다.
---

# Acme Backend 사용 가이드
- 서비스·트랜잭션 작성 → guides/service-and-tx.md
- 예외와 에러 코드 추가 → guides/error-codes.md
- 새 CRUD 기능 → templates/CrudService.java.md 를 복사해서 시작
```

**점진적 로딩의 효과:** 가이드 전체가 수십 페이지여도, Claude는 필요한 파일만 읽으므로 컨텍스트를 아낍니다.

### ③ 참조 코드 (골든 예제)

```
reference/
├── backend/SampleCrudController.java   # 프레임워크 팀이 관리하는 모범 구현
└── frontend/SampleListPage.tsx
```

```text
reference/backend/SampleCrudController.java와 같은 방식으로 수강 후기 CRUD를 만들어줘.
```

> 💡 긴 설명보다 **잘 만든 예제 코드 하나**가 더 효과적인 경우가 많습니다.

### ④ 프레임워크 문서와 소스에 접근시키기

| 방법 | 설명 |
|---|---|
| 소스 jar / 소스 저장소 | 프레임워크 소스를 함께 클론하거나 sources jar 경로를 알려 주면 Claude가 직접 읽고 확인 |
| 사내 문서 MCP 서버 | 사내 위키나 프레임워크 문서 검색 도구를 MCP로 제공 ([MCP 예제](./04-mcp-plugins-artifacts.md)) |
| 변경 이력 | 프레임워크 버전 업그레이드 시 CHANGELOG를 스킬에 포함 |

### ⑤ 규칙 강제: 지침이 아니라 검사로

**백엔드: ArchUnit 테스트**

```java
@AnalyzeClasses(packages = "com.acme.eduhub")
class AcmeArchitectureTest {

    @ArchTest
    static final ArchRule services_use_acme_service =
        classes().that().resideInAPackage("..service..")
                 .should().beAnnotatedWith(AcmeService.class);

    @ArchTest
    static final ArchRule no_spring_transactional =
        noClasses().should().beAnnotatedWith(org.springframework.transaction.annotation.Transactional.class)
                   .because("@AcmeTx를 사용해야 합니다");
}
```

**프론트엔드: ESLint**

```js
// eslint.config.js (일부)
rules: {
  "no-restricted-imports": ["error", {
    paths: [
      { name: "axios", message: "@acme/http의 acmeClient를 사용하세요." },
      { name: "@mui/x-data-grid", message: "@acme/ui-kit의 AcmeGrid를 사용하세요." },
    ],
  }],
}
```

**훅으로 연결:** 파일을 수정할 때마다 ESLint와 ArchUnit을 돌려, 위반하면 Claude가 바로 오류 메시지를 보고 고치게 합니다 ([훅 예제](./03-settings-skills-agents-hooks.md#모듈-12-훅)).

**리뷰 서브에이전트:** `acme-compliance-reviewer`를 만들어 "표준 방식이 섞였는지"를 검토합니다.

### ⑥ 플러그인으로 전사 배포

```
acme-framework-kit/              # 프레임워크 팀이 관리
├── .claude-plugin/plugin.json   # version을 프레임워크 버전과 맞춤 (예: 5.2.0)
├── skills/acme-backend/
├── skills/acme-frontend/
├── agents/acme-compliance-reviewer.md
├── hooks/hooks.json
└── .mcp.json                    # 사내 문서 검색 MCP
```

- 프레임워크가 업그레이드되면 플러그인도 함께 업데이트 → 모든 팀이 같은 최신 지식으로 일한다
- 각 프로젝트의 CLAUDE.md에는 프로젝트 고유 규칙만 남긴다

---

## 4. 역할 분담

| 역할 | 책임 |
|---|---|
| **프레임워크 팀** | 스킬, 참조 코드, ArchUnit/ESLint 규칙, 플러그인 작성과 버전 관리 |
| **서비스 개발 팀** | 플러그인 설치, 프로젝트 CLAUDE.md 작성, 잘못 생성된 사례를 프레임워크 팀에 피드백 |
| **플랫폼/보안 팀** | 사내 MCP 서버 운영, 권한 정책 |

---

## 5. 단계적 도입 로드맵

| 단계 | 할 일 | 성공 기준 |
|---|---|---|
| 1. 파일럿 | 한 팀에서 CLAUDE.md + 참조 코드로 시작 | 생성 코드가 사내 프레임워크를 쓰는 비율 측정 |
| 2. 스킬화 | 자주 틀리는 부분을 스킬과 템플릿으로 보강 | 수정 요청 횟수 감소 |
| 3. 강제 | ArchUnit, ESLint, 훅 적용 | 규칙 위반이 CI에서 0건 |
| 4. 확산 | 플러그인으로 묶어 전사 배포 | 여러 팀이 같은 플러그인 사용 |

---

## 6. 실습

1. 플러그인 없이 "수강 후기 CRUD를 만들어줘" → 표준 Spring/MUI 코드가 나오는 것 확인
2. CLAUDE.md만 추가 후 재실행 → 개선 정도 비교
3. 스킬과 참조 코드 추가 후 재실행 → 비교
4. ArchUnit/ESLint 훅을 켜고, 일부러 `@Transactional`을 쓰게 지시 → 차단되고 스스로 고치는 과정 관찰
5. 결과를 표로 정리해 아티팩트로 공유
