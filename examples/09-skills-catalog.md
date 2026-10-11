# 예제: 스킬 모범 사례와 공개 스킬 소개

> 관련 모듈: 10 (14, 16과 연결)
> 대상 기술 스택: Java, Spring Boot, JPA/Hibernate, OpenAPI/Swagger, React, Microfrontend, MUI, Axios

---

## 1. 좋은 스킬의 조건

| 원칙 | 설명 | 나쁜 예 → 좋은 예 |
|---|---|---|
| **description이 핵심** | Claude는 설명만 보고 스킬을 쓸지 정한다. "무엇을 하는지 + 언제 쓰는지"를 담는다 | "API 스킬" → "새 REST API를 추가할 때 사용. Controller·DTO·OpenAPI 문서·테스트·프론트 클라이언트 재생성까지 처리" |
| **하나의 스킬, 하나의 작업** | 여러 일을 섞으면 선택이 부정확해진다 | "백엔드 전부" → `add-api-endpoint`, `flyway-migration` 분리 |
| **본문은 짧게, 상세는 파일로** | `SKILL.md`는 목차 역할, 상세 가이드는 하위 파일로 → 필요한 것만 읽힘 | 500줄 SKILL.md → 50줄 SKILL.md + `guides/*.md` |
| **결정적인 작업은 스크립트로** | 검사·변환·생성은 모델보다 코드가 정확하고 토큰도 적게 든다 | "어노테이션 누락을 찾아라" → `scripts/check-openapi.sh` |
| **예제 코드 포함** | 설명보다 템플릿·골든 예제가 효과적 | `templates/Controller.java.md` |
| **검증 단계 포함** | 끝에 "무엇으로 확인하는지"를 명시 | "`./gradlew test`와 Swagger UI로 확인" |
| **버전 관리와 리뷰** | 스킬도 코드처럼 PR로 관리 | `.claude/skills/`에 CODEOWNERS 지정 |

**스킬 작성 도우미:** Anthropic의 `skill-creator` 스킬(아래 공개 스킬 참고)을 설치하면 "Spring Boot API 추가 스킬을 만들어줘"처럼 대화로 스킬을 만들고 개선할 수 있습니다.

---

## 2. 기술 스택별 추천 스킬 사례 (직접 만들기)

### 2-1. 백엔드 (Java, Spring Boot, JPA/Hibernate, OpenAPI)

| 스킬 이름 | 언제 쓰나 | 포함할 내용 |
|---|---|---|
| `add-api-endpoint` | 새 REST API 추가 | 계층별 템플릿, OpenAPI 어노테이션 규칙, 테스트 템플릿, 클라이언트 재생성 ([전체 예제](./03-settings-skills-agents-hooks.md#모듈-10-스킬-add-api-endpoint)) |
| `jpa-entity-design` | 엔티티·연관관계 설계 | LAZY 원칙, 양방향 연관관계 편의 메서드, `@Setter` 금지, equals/hashCode 규칙 |
| `jpa-performance` | 느린 조회, N+1 의심 | 쿼리 로그 켜는 법, Hibernate Statistics로 쿼리 수 검증 테스트, `@EntityGraph`/fetch join/`@BatchSize` 선택 기준 |
| `flyway-migration` | 스키마 변경 | 파일 명명 규칙, 기존 파일 수정 금지, 대용량 테이블 변경 주의 사항, 엔티티 매핑 검증 |
| `openapi-docs` | API 문서 보완 | `@Operation`/`@ApiResponse`/`@Schema` 규칙, 에러 응답 문서화, 누락 검사 스크립트 |
| `spring-test-strategy` | 테스트 작성 | 계층별 테스트 종류(`@WebMvcTest`, `@DataJpaTest`, Testcontainers), Fixture 사용법 |
| `spring-boot-upgrade` | 버전 업그레이드 | 마이그레이션 체크리스트, 지원 중단 API 대체표, 단계별 검증 |

**예시: `jpa-performance/SKILL.md`**

```markdown
---
name: jpa-performance
description: JPA/Hibernate 조회 성능 문제(N+1, 느린 목록 조회, 과도한 쿼리)를 진단하고 고칠 때 사용한다. 쿼리 수를 테스트로 측정하고 fetch 전략을 선택하는 절차를 제공한다.
---

# JPA 성능 진단 절차

1. **재현**: `guides/query-count-test.md`의 방법으로 현재 쿼리 수를 측정하는 테스트를 먼저 작성한다
2. **원인 파악**: 지연 로딩 연관관계를 반복문에서 접근하는 코드를 찾는다
3. **해결책 선택** (`guides/fetch-strategy.md` 참고)
   - 단건 + 연관 1개 → fetch join
   - 목록 + 페이징 → `@EntityGraph`는 컬렉션 페이징 불가 → `@BatchSize` 또는 DTO 프로젝션
   - 화면 전용 조회 → DTO 프로젝션 쿼리
4. **검증**: 1번 테스트의 쿼리 수가 줄었는지 확인하고, 기존 테스트 전체 통과 확인
```

### 2-2. 프론트엔드 (React, Microfrontend, MUI, Axios)

| 스킬 이름 | 언제 쓰나 | 포함할 내용 |
|---|---|---|
| `new-microfrontend` | 새 remote 앱 추가 | webpack Module Federation 템플릿, shared singleton 목록, shell 라우트 등록, 로컬 실행 포트 규칙 |
| `mfe-shared-deps` | 공유 의존성 버전 변경 | shell과 모든 remote 동시 변경 절차, 버전 불일치 검사 스크립트 |
| `api-client-usage` | 프론트에서 API 호출 | 클라이언트 재생성 명령, 커스텀 훅 패턴, 에러는 인터셉터에 맡기는 규칙 |
| `mui-page` | 새 화면 작성 | 사내 테마 사용, 목록/상세/폼 화면 템플릿, 로딩·에러·빈 상태 처리 |
| `react-testing` | 컴포넌트 테스트 | React Testing Library 규칙, API 모킹(MSW 등) 방법 |

**예시: `new-microfrontend/SKILL.md`**

```markdown
---
name: new-microfrontend
description: EduHub 프론트엔드에 새 마이크로프론트엔드(remote 앱)를 추가할 때 사용한다. Module Federation 설정, shared 의존성, shell 라우팅 등록, 개발 서버 포트 할당을 처리한다.
---

# 새 remote 추가 절차
1. `templates/remote-app/`을 `apps/{이름}-mfe/`로 복사한다
2. webpack 설정의 `name`을 camelCase(`{이름}Mfe`)로 바꾼다
3. shared 설정은 `templates/shared.js`를 그대로 사용한다 (버전을 바꾸지 않는다)
4. 포트는 `ports.md`에서 다음 빈 번호를 사용한다
5. shell의 `remotes`와 라우트에 등록한다
6. `scripts/check-shared-versions.mjs`로 버전 일치를 확인한다
7. `npm run dev`로 shell에서 화면이 뜨는지 확인한다
```

### 2-3. 풀스택 (백엔드 ↔ 프론트 연결)

| 스킬 이름 | 언제 쓰나 | 포함할 내용 |
|---|---|---|
| `fullstack-feature` | 화면 + API가 함께 필요한 기능 | `add-api-endpoint` → 클라이언트 재생성 → `mui-page` 순서 안내, 커밋 분리 규칙 |
| `api-breaking-change` | 기존 API 변경 | 버전 전략(`/v2`), 사용 중인 MFE 찾기, 하위 호환 확인 |

---

## 3. 공개 스킬 소개

> ⚠️ 공개 스킬은 **다른 사람이 작성한 지침과 스크립트**입니다. 설치 전에 내용을 직접 읽어 보고, 라이선스와 관리 상태를 확인하세요. 스크립트가 포함된 스킬은 특히 주의합니다.
> 아래 정보는 2026년 10월 기준이며 바뀔 수 있습니다.

### 3-1. Anthropic 공식: [anthropics/skills](https://github.com/anthropics/skills)

```text
/plugin marketplace add anthropics/skills
/plugin install example-skills@anthropic-agent-skills
/plugin install document-skills@anthropic-agent-skills
```

| 스킬 | 내용 | 교육과정 활용 |
|---|---|---|
| `skill-creator` | 스킬을 만들고 개선하는 것을 돕는 스킬 | 모듈 10 실습에서 스킬 작성 도우미로 사용 |
| `mcp-builder` | MCP 서버 개발 가이드 | 모듈 13에서 EduHub MCP 서버 만들기 |
| `webapp-testing` | 브라우저로 웹 앱을 테스트 | React MFE 화면 동작 확인 |
| `frontend-design` | 완성도 높은 프론트엔드 UI 디자인 | MUI 화면 시안 개선 |
| `web-artifacts-builder` | 복잡한 웹 아티팩트 제작 | 모듈 15 대시보드·프로토타입 |
| `claude-api` | Claude API·SDK 사용 가이드 | 모듈 18 Agent SDK 실습 |
| `docx`, `pptx`, `xlsx`, `pdf` | 문서 파일 생성·편집 | 교육 자료를 PPT로 만들기, 리포트 작성 |
| `doc-coauthoring`, `internal-comms` | 문서 공동 작성, 사내 공지 작성 | 릴리스 노트, 장애 보고서 |

`template/` 폴더에 새 스킬의 시작 템플릿이, `spec/` 폴더에 Agent Skills 사양이 있습니다.

### 3-2. 개발 방법론 스킬: [obra/superpowers](https://github.com/obra/superpowers)

MIT 라이선스. 설계 → 계획 → 구현 → 검증으로 이어지는 **개발 프로세스를 스킬로 구성**한 모음입니다.

```text
/plugin install superpowers@claude-plugins-official
```

| 스킬 | 내용 |
|---|---|
| `test-driven-development` | 테스트 먼저 작성하는 절차 |
| `systematic-debugging` | 추측하지 않고 원인을 좁혀 가는 디버깅 절차 |
| `verification-before-completion` | "완료" 전에 반드시 검증하는 습관 |
| `brainstorming`, `writing-plans`, `executing-plans` | 요구사항 정리와 계획 수립·실행 |
| `using-git-worktrees`, `finishing-a-development-branch` | 브랜치·워크트리 작업 흐름 |
| `subagent-driven-development`, `dispatching-parallel-agents` | 서브에이전트에 작업을 나눠 맡기는 방법 |
| `requesting-code-review`, `receiving-code-review` | 코드 리뷰 요청과 반영 |
| `writing-skills` | 스킬 작성법 |

**교육 활용:** 모듈 5(워크플로우), 6(TDD, 디버깅), 11(서브에이전트)에서 "잘 만든 스킬은 이렇게 생겼다"는 읽기 자료로 활용하기 좋습니다.

### 3-3. Spring Boot·React 관련 커뮤니티 스킬

검색으로 찾을 수 있는 커뮤니티 스킬 예시입니다. **품질과 관리 상태가 검증되지 않았으므로** 그대로 쓰기보다 참고해서 사내 스킬을 만드는 용도로 권장합니다.

| 저장소 | 내용 |
|---|---|
| [piomin/claude-ai-spring-boot](https://github.com/piomin/claude-ai-spring-boot) | Spring Boot용 Claude Code 템플릿. 코드 리뷰, JPA 패턴, 로깅 등의 스킬과 에이전트 |
| [jeffallan/claude-skills](https://github.com/jeffallan/claude-skills) | 여러 기술의 스킬 모음. `spring-boot-engineer` 스킬 포함 |
| [joseptr1285/skills](https://github.com/joseptr1285/skills) | `react` 스킬(훅, 성능 최적화) 포함 |

스킬 목록을 모아 둔 커뮤니티 사이트(레지스트리)도 여러 곳 있지만, 출처가 다양하므로 원본 저장소와 라이선스를 반드시 확인합니다.

---

## 4. 공개 스킬 도입 절차 (팀 정책 예시)

1. **탐색**: 공식 저장소 → 신뢰할 수 있는 저장소 순으로 찾는다
2. **검토**: `SKILL.md`와 모든 스크립트를 읽는다. 외부로 데이터를 보내는 코드가 없는지 확인한다
3. **라이선스 확인**: 사내 사용·수정이 가능한지 확인한다
4. **포크와 맞춤화**: 사내 저장소로 가져와 EduHub 규칙에 맞게 수정한다
5. **배포**: 사내 플러그인 마켓플레이스로 배포한다 ([플러그인 예제](./04-mcp-plugins-artifacts.md))

---

## 5. 실습

1. `skill-creator`를 설치하고, 대화로 `jpa-performance` 스킬을 만들기
2. 만든 스킬로 EduHub의 N+1 문제(P1)를 해결하고, 스킬이 자동으로 선택되는지 확인
3. `superpowers`의 `systematic-debugging` 스킬을 읽고, 우리 팀 디버깅 절차와 비교해 토론
4. 커뮤니티 Spring Boot 스킬 하나를 골라 위 "도입 절차"에 따라 검토 보고서 작성
