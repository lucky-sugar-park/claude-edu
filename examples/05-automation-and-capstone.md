# 예제: 자동화(헤드리스, CI/CD, Agent SDK)와 캡스톤 주제

> 관련 모듈: 17, 18, 20, 캡스톤
> 예제 프로젝트: [EduHub](./00-sample-project.md)

---

## 모듈 17. 헤드리스 모드 & CI/CD

### 17-1. 헤드리스 모드 (`claude -p`)

```bash
# OpenAPI 어노테이션 누락 검사 결과를 JSON으로 받기
claude -p "backend의 모든 컨트롤러에서 @Operation이 없는 메서드를 찾아 파일과 메서드 이름 목록으로 알려줘" \
  --output-format json > openapi-audit.json

# 실패한 테스트 로그를 넘겨 원인 분석
./gradlew test 2>&1 | tail -100 | claude -p "이 테스트 실패의 원인을 분석하고 수정 방향을 제안해줘"
```

### 17-2. GitHub Actions: PR 자동 리뷰

```yaml
# .github/workflows/claude-review.yml
name: Claude PR Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            이 PR을 EduHub의 CLAUDE.md 규칙에 따라 리뷰해줘.
            특히 다음을 확인해줘.
            - JPA: N+1, EAGER 로딩, 엔티티 직접 반환, 트랜잭션 범위
            - API: OpenAPI 어노테이션 누락, API 변경 시 프론트 클라이언트 재생성 여부
            - 프론트: Module Federation shared 버전 변경, api-client 대신 Axios 직접 사용
            문제는 해당 줄에 인라인 코멘트로 남겨줘.
```

> 액션의 입력값과 버전은 바뀔 수 있으므로 교육 전에 공식 문서로 확인하세요.

**자동화 아이디어**

| 트리거 | 자동화 내용 |
|---|---|
| PR 생성 | 위 리뷰, Swagger 스펙 변경 내역 요약 코멘트 |
| 이슈에 `@claude` 멘션 | 이슈 내용으로 구현 후 PR 생성 |
| 매주 월요일 | 의존성(Spring Boot, MUI) 업데이트 영향 분석 리포트 |
| main 머지 | 릴리스 노트 초안 생성 |

---

## 모듈 18. Claude Agent SDK

**예제: API 계약 테스트 생성 에이전트**

OpenAPI 스펙을 읽고, 각 엔드포인트에 대한 계약 테스트(Spring `MockMvc` 테스트)를 생성하는 사내 도구를 만듭니다.

```ts
// contract-test-agent/index.ts
import { query } from "@anthropic-ai/claude-agent-sdk";

const prompt = `
http://localhost:8080/v3/api-docs 의 OpenAPI 스펙을 읽어줘.
각 엔드포인트마다 backend/src/test/java/.../contract/ 에 MockMvc 기반 계약 테스트를 작성해줘.
- 성공 응답의 스키마 필드 검증
- 문서화된 에러 응답(4xx) 검증
작성 후 ./gradlew test --tests "*ContractTest"로 실행하고, 실패하면 스펙과 구현 중 무엇이 틀렸는지 보고해줘.
`;

for await (const message of query({
  prompt,
  options: {
    cwd: "../eduhub",
    allowedTools: ["Read", "Write", "Edit", "Glob", "Grep", "Bash", "WebFetch"],
    permissionMode: "acceptEdits",
  },
})) {
  if (message.type === "result") console.log(message);
}
```

> SDK의 옵션 이름과 메시지 형식은 버전에 따라 다를 수 있으므로 교육 전에 공식 문서로 확인하세요.

**토론 포인트**
- Claude Code CLI와 SDK의 차이: 같은 에이전트 루프(모듈 1)를 **내 프로그램 안에서** 제어한다
- 어떤 도구를 허용하고 어떤 권한으로 실행할지 코드로 정한다

---

## 모듈 20. 운영 & 거버넌스 체크리스트 (EduHub 팀 기준)

- [ ] 팀 공용 `.claude/settings.json`에 운영 설정·비밀정보 읽기 차단
- [ ] 사용 가능한 MCP 서버 목록 승인 (DB는 읽기 전용 계정만)
- [ ] 사내 플러그인 마켓플레이스만 허용
- [ ] AI가 작성한 코드도 사람 리뷰 1명 이상 필수
- [ ] CI 자동 리뷰 비용 모니터링 (PR당 사용량)

---

## 🏁 캡스톤 프로젝트 주제 예시

모든 주제는 EduHub에 기능을 추가하는 형태이며, 캡스톤 필수 요소(CLAUDE.md, 스킬, 훅, MCP, 플러그인, 아티팩트)를 모두 포함해야 합니다.

| 주제 | 백엔드 | 프론트엔드 | 활용 포인트 |
|---|---|---|---|
| **수료증 발급** | 수료 판정 로직, PDF 생성 API | 마이페이지에 수료증 다운로드 | 스킬로 API 추가 자동화, 아티팩트로 수료증 디자인 시안 |
| **교육 추천** | 수강 이력 기반 추천 API | 새 remote `recommend-mfe` 추가 | `new-microfrontend` 스킬 작성, DB MCP로 데이터 분석 |
| **관리자 통계 대시보드** | 통계 집계 API (JPA 프로젝션, 네이티브 쿼리) | MUI 차트 화면 | 성능 리뷰 서브에이전트, 아티팩트로 대시보드 프로토타입 |
| **알림 기능** | 신청 확정·대기 승격 이벤트, 알림 API | 전역 알림 센터(shell) | 훅으로 이벤트 명명 규칙 강제, CI 자동 리뷰 |
