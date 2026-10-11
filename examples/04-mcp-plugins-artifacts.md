# 예제: 커넥터(MCP), 플러그인, 아티팩트

> 관련 모듈: 13, 14, 15
> 예제 프로젝트: [EduHub](./00-sample-project.md)

---

## 모듈 13. 커넥터와 MCP

### 13-1. 프로젝트 공용 MCP 설정: `.mcp.json`

```json
{
  "mcpServers": {
    "eduhub-db": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${EDUHUB_DB_READONLY_URL}"]
    },
    "github": {
      "type": "http",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  }
}
```

> 서버 패키지와 URL은 예시입니다. 실제 교육 전에 사용할 MCP 서버의 최신 설치 방법을 확인하세요.

- DB 접속 정보는 환경 변수로 넣고 파일에 직접 쓰지 않는다
- DB는 **읽기 전용 계정**으로 연결한다 (최소 권한)

### 13-2. 활용 예시

```text
eduhub-db에서 지난달 수강 신청이 가장 많은 과정 5개와 회차별 정원 대비 신청률을 조회해줘.
```

```text
enrollment 테이블의 실제 인덱스를 확인하고, EnrollmentRepository의 쿼리 메서드들과 비교해서 빠진 인덱스를 Flyway 마이그레이션으로 제안해줘.
```

```text
GitHub에서 "bug" 라벨이 붙은 열린 이슈를 가져와서, 각 이슈가 백엔드/프론트 중 어디 문제인지 분류해줘.
```

### 13-3. 직접 만드는 MCP 서버: EduHub API 래핑

EduHub의 OpenAPI 스펙을 활용해, 운영자가 Claude 앱에서 자연어로 교육 현황을 조회하는 MCP 서버를 만듭니다.

```ts
// eduhub-mcp/src/index.ts (TypeScript MCP SDK 사용 예시)
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import axios from "axios";

const api = axios.create({ baseURL: process.env.EDUHUB_API_URL });
const server = new McpServer({ name: "eduhub", version: "1.0.0" });

server.tool(
  "search_courses",
  "카테고리나 키워드로 교육과정을 검색한다",
  { category: z.string().optional(), keyword: z.string().optional() },
  async ({ category, keyword }) => {
    const { data } = await api.get("/api/v1/courses", { params: { category, keyword } });
    return { content: [{ type: "text", text: JSON.stringify(data.content, null, 2) }] };
  },
);

server.tool(
  "get_session_status",
  "교육 회차의 정원, 신청 인원, 대기 인원을 조회한다",
  { sessionId: z.number() },
  async ({ sessionId }) => {
    const { data } = await api.get(`/api/v1/sessions/${sessionId}/status`);
    return { content: [{ type: "text", text: JSON.stringify(data) }] };
  },
);

await server.connect(new StdioServerTransport());
```

```bash
claude mcp add eduhub -- node eduhub-mcp/dist/index.js
```

**토론 포인트**
- 조회 도구만 먼저 열고, 신청·취소처럼 **변경하는 도구는 승인 절차**를 거치게 한다
- 도구 설명(description)이 좋아야 Claude가 적절한 때에 도구를 고른다 (스킬의 description과 같은 원리)

---

## 모듈 14. 플러그인: `eduhub-dev-kit`

모듈 9~13에서 만든 결과물을 하나의 플러그인으로 묶어 다른 팀 프로젝트에도 배포합니다.

```
eduhub-dev-kit/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── add-api-endpoint/
│   └── new-microfrontend/
├── agents/
│   ├── jpa-reviewer.md
│   └── mfe-integration-checker.md
├── commands/
│   └── release-notes.md
├── hooks/
│   └── hooks.json
└── .mcp.json
```

```json
// .claude-plugin/plugin.json
{
  "name": "eduhub-dev-kit",
  "version": "1.0.0",
  "description": "Spring Boot + React 마이크로프론트엔드 팀용 개발 키트: API 추가 스킬, JPA 리뷰어, 자동 포맷 훅",
  "author": { "name": "Platform Team" }
}
```

**사내 마켓플레이스로 배포**

```json
// (마켓플레이스 저장소) .claude-plugin/marketplace.json
{
  "name": "company-plugins",
  "owner": { "name": "Platform Team" },
  "plugins": [
    {
      "name": "eduhub-dev-kit",
      "source": "./plugins/eduhub-dev-kit",
      "description": "Spring Boot + React MFE 개발 키트"
    }
  ]
}
```

```text
/plugin marketplace add company/claude-plugins
/plugin install eduhub-dev-kit@company-plugins
```

> 플러그인 구조와 명령은 버전에 따라 바뀔 수 있으므로 교육 전에 공식 문서로 확인하세요.

**토론 포인트**
- 어떤 것은 플러그인에 넣고, 어떤 것은 프로젝트 `.claude/`에 남길까? → 여러 프로젝트에 공통이면 플러그인, 이 저장소에만 해당하면 프로젝트
- 버전을 올릴 때 팀에 어떻게 알릴까?

---

## 모듈 15. 아티팩트

EduHub 개발 과정에서 나오는 결과물을 아티팩트로 만들어 공유합니다.

| 아티팩트 | 요청 예시 | 공유 대상 |
|---|---|---|
| **API 문서 페이지** | "/v3/api-docs 스펙을 읽어서 프론트 개발자용 API 가이드 페이지를 만들어줘. 엔드포인트별 요청·응답 예시와 에러 코드를 포함해줘" | 프론트 개발자, 협력사 |
| **ERD 다이어그램** | "JPA 엔티티를 분석해서 ERD를 그려줘. 연관관계 방향과 fetch 전략을 표시해줘" | 신규 입사자, 설계 리뷰 |
| **MFE 아키텍처 다이어그램** | "shell과 remote의 관계, shared 의존성, API 호출 흐름을 다이어그램으로 보여줘" | 아키텍처 리뷰 |
| **성능 개선 리포트** | "N+1 수정 전후의 쿼리 수와 응답 시간을 비교하는 리포트를 차트와 함께 만들어줘" | 팀 리드, 관리자 |
| **교육 신청 현황 대시보드** | "eduhub-db에서 과정별 신청률을 조회해서 대시보드를 만들어줘" | 교육 담당자 |
| **화면 프로토타입** | "수강 취소 확인 Dialog를 MUI 스타일로 프로토타입을 만들어줘. 기한이 지난 경우 화면도 보여줘" | 기획자, 디자이너 |

**실습 흐름**

1. 요청: "N+1 수정 결과 리포트를 만들어줘"
2. 미리보기 확인
3. 피드백: "관리자용이니 기술 용어를 줄이고, 맨 위에 핵심 수치 3개를 크게 보여줘"
4. 수정 확인 후 링크로 공유

> ⚠️ DB 조회 결과로 만든 대시보드는 개인정보(임직원 이름, 사번)가 포함되지 않았는지 확인한 뒤 공유합니다.
