# 8회차: 자동화·Agent SDK·거버넌스 — 성능·보안 점검과 AI 없이 핫픽스 훈련 (NFR-02, NFR-03)

🧭 ⬅️ 이전: [7회차: CI/CD·환경 분리 완성·사내 프레임워크·토큰 최적화 (NFR-05, NFR-06)](./session-07-cicd-framework-tokens.md) · ➡️ 다음: [9회차: Wrap-up — 1~8회차 다시 점검하고 캡스톤 준비하기](./session-09-wrap-up.md) · 📚 [전체 커리큘럼](../CURRICULUM.md#-회차별-강의-자료)

> **대상**: 7회차까지 EduHub에 CI/CD와 스테이징 배포를 갖춘 팀
> **시간**: 120분 (강의 약 40%, 따라하기 약 60%)
> **목표**
> 1. `claude -p` 헤드리스 실행과 GitHub Actions로 **사람 없이 도는 Claude 작업**을 설계하고, 비용·권한을 통제한다
> 2. Claude Agent SDK로 **사내 도구 안에 에이전트를 넣는 경우**와 그렇지 않은 경우를 구분한다
> 3. 조직 단위 **거버넌스**(관리 설정, 승인된 MCP·플러그인, 사용량 모니터링, 감사)의 큰 그림을 이해한다
> 4. EduHub의 **성능(NFR-02)**을 숫자로 측정하고 개선 전후를 비교한다
> 5. EduHub의 **보안(NFR-03)**을 점검하고 PR 자동 보안 리뷰를 붙인다
> 6. **AI 없이 핫픽스 훈련**으로 "시스템의 주인은 개발자"를 직접 증명하고, 사후 분석에서 Claude를 활용한다
>
> **관련 모듈**: 17(헤드리스 & CI/CD), 18(Claude Agent SDK), 20(운영 & 거버넌스) + 선택 심화 A(RAG)
> **사전 준비**
> - 7회차 결과: `main` 병합 시 스테이징에 자동 배포되는 파이프라인, 런북(`docs/runbook.md`), 이전 버전으로 되돌리는 롤백 절차
> - 과정 데이터 1만 건 시드: 수업 중 Claude Code로 `scripts/seed-10k.sql`을 만들어 넣습니다 ([Step 1-3](#1-3-1만-건-시드-sql은-claude-code에게-만들게-하고-실행은-내가)). 로컬 PostgreSQL(`docker compose up -d postgres`)이 떠 있어야 합니다
> - [k6](https://k6.io/) 설치 (`k6 version`으로 확인, 방법은 [Step 2-3](#2-3-k6-설치)). 회사 PC에서 설치가 막히면 강사 PC의 결과를 함께 봅니다
> - 저장소 Settings → Secrets에 `ANTHROPIC_API_KEY` 등록 권한 (팀장 1명)
> 🖥️ **슬라이드 덱(37장, 발표자 노트 포함)**: https://claude.ai/artifact/GtdvAoFTYMtxTvStG13Zme — 처음에는 만든 사람만 볼 수 있으며, 다른 사람에게 보여주려면 덱의 공유 메뉴에서 공유해야 합니다. PowerPoint(.pptx)·PDF로 내려받을 수 있습니다.

---

## 진행 순서

| 시간 | 내용 | 형식 |
|---|---|---|
| 0:00 – 0:05 | 지난 회차 복습 & 과제 리뷰 | 강의 |
| 0:05 – 0:20 | 1. 헤드리스 실행과 CI 자동화 (모듈 17) | 강의 + 시연 |
| 0:20 – 0:30 | 2. Claude Agent SDK: 내 프로그램 안의 에이전트 (모듈 18), 조직의 도구 표준(제조 도구 개발 키트) | 강의 |
| 0:30 – 0:40 | 3. 운영 & 거버넌스 (모듈 20) | 강의 + 토론 |
| 0:40 – 0:48 | ☕ 휴식 (스테이징이 정상인지 각 팀이 확인) | — |
| 0:48 – 1:08 | 4. 따라하기 ①: 성능 측정과 개선 (NFR-02) | 실습 |
| 1:08 – 1:23 | 5. 따라하기 ②: 보안 점검과 PR 자동 보안 리뷰 (NFR-03) | 실습 |
| 1:23 – 1:53 | 6. 따라하기 ③: ⭐ AI 없이 핫픽스 훈련 + Claude와 사후 분석 | 실습 |
| 1:53 – 2:00 | 8. 정리, 과제, Wrap-up·캡스톤 안내 | 강의 |

> 7장(SDK 미니 봇, RAG 맛보기)은 시간이 남으면 수업 중에, 아니면 **과제**로 진행합니다.
> 💬 **강사 노트:** 핫픽스 훈련(6장)은 **절대 줄이지 않습니다.** 시간이 밀리면 1~3장의 시연과 5장의 CI 리뷰 설정을 줄입니다.

---

## 지난 회차 복습 & 과제 리뷰

**7회차 핵심 3문장**
1. 빌드는 **한 번**, 배포는 같은 결과물(실습은 jar, 운영에서는 보통 컨테이너 이미지)을 **여러 환경에**
2. 운영 배포는 **사람의 승인**과 파이프라인으로만. Claude Code는 파이프라인을 **만드는 것을 돕지만**, 운영에 직접 배포하지 않는다
3. 런북은 **AI 없이 사람이 그대로 따라 할 수 있어야** 한다

**과제 확인 (팀별 1분)**
- [ ] 스테이징 자동 배포가 동작하는가? 지금 스테이징 버전(스테이징 PC의 `current.txt`)과 직전 버전은?
- [ ] `docs/runbook.md`에 배포·롤백 절차가 명령어 수준으로 적혀 있는가?
- [ ] 이번 주 `/usage` 기록과 토큰을 아낀 방법 1개

> 💬 **강사 노트:** 오늘 6장의 핫픽스 훈련은 이 런북을 그대로 씁니다. 런북이 부실한 팀은 휴식 시간에 "롤백 명령" 한 줄이라도 채워 두라고 미리 알려 줍니다. (어떤 버그를 넣을지는 말하지 않습니다)

---

# 1. 헤드리스 실행과 CI 자동화 (모듈 17)

---

## 1-1. 대화형 vs 헤드리스

| 구분 | 대화형 (`claude`, 데스크톱 앱) | 헤드리스 (`claude -p "지시"`) |
|---|---|---|
| 누가 보나 | 사람이 화면을 보며 승인·수정 | **아무도 보지 않음**. 결과만 남음 |
| 권한 승인 | 승인 창에서 사람이 판단 | 승인할 사람이 없음 → **미리 허용한 도구만** 쓰게 해야 함 |
| 출력 | 화면 | 표준 출력: `text`(기본), `json`(결과+사용량), `stream-json`(진행 이벤트) |
| 어울리는 일 | 설계, 구현, 디버깅 | PR 리뷰, CI 실패 분석, 릴리스 노트, 정기 리포트 |

```bash
# 결과를 JSON으로 받아 사용량까지 기록 (CI 비용 추적에 사용)
claude -p "backend에서 @Operation이 없는 컨트롤러 메서드를 찾아 목록으로 알려줘. 코드는 수정하지 마." \
  --output-format json \
  --permission-mode dontAsk \
  --max-turns 8 > openapi-audit.json

# 파이프로 로그를 넘겨 분석
./gradlew test 2>&1 | tail -100 | claude -p "이 테스트 실패의 원인과 수정 방향을 5줄로 요약해줘"
```

> 💡 헤드리스의 3대 안전장치: **허용 범위를 좁힌다**(`--permission-mode dontAsk` + 꼭 필요한 명령만 `--allowedTools`), **턴 수 제한**(`--max-turns`), **분석만 하고 수정·푸시는 사람이**. `dontAsk`에서도 **작업 폴더 안의 파일 읽기·검색은 허용**되므로 `Read`·`Grep`·`Glob`을 따로 허용하지 않습니다. 범위 없이 `--allowedTools "Read"`처럼 허용하면 **작업 폴더 밖**(러너의 다른 폴더)까지 읽을 수 있습니다 ([2-2 예제 ①](#예제-①-javascript-agent-sdk로-테스트-실패-원인-분석-봇)에서 실제로 확인). 비용 상한은 `--max-budget-usd`로도 걸 수 있습니다. ([빌드·배포 파이프라인 4-2](../examples/28-build-deploy-pipeline.md#4-2-ci-실패-원인-자동-분석))

---

## 1-2. 파이프라인 안의 Claude

```mermaid
flowchart LR
    PR["PR 생성·갱신"] --> CI["CI<br/>빌드·테스트·린트"]
    PR --> RV["Claude PR 리뷰<br/>(claude-code-action)<br/>읽기 전용, 코멘트만"]
    CI -- 실패 --> AN["Claude 실패 분석<br/>(claude -p)<br/>원인 요약 코멘트"]
    CI -- 성공 --> HR["사람 리뷰·승인"]
    RV --> HR
    AN --> DEV["개발자가 수정"]
    DEV --> PR
    HR --> MG["Squash merge"] --> STG["스테이징 자동 배포"]
    STG --> APV{"사람 승인"} --> PROD["운영 배포"]
```

| 자동화 | 트리거 | 도구 | 권한 |
|---|---|---|---|
| PR 자동 리뷰 | `pull_request` | `anthropics/claude-code-action@v1` | `contents: read`, `pull-requests: write` |
| CI 실패 원인 분석 | CI 실패 | `claude -p` + `gh` | 읽기 전용 도구, 코멘트만 |
| 이슈 `@claude` 멘션 처리 | 이슈 코멘트 | `claude-code-action` | 브랜치·PR 생성까지 (사람 리뷰 필수) |
| 릴리스 노트 초안 | 태그 생성 | `claude -p` | 읽기 전용 |

> ⚠️ 액션의 입력값과 버전은 바뀔 수 있으므로 설정 전에 공식 문서로 확인합니다. 터미널에서 `/install-github-app`을 실행하면 설정을 안내받을 수 있습니다.

---

## 1-3. CI에서의 비용·보안 통제

| 위험 | 통제 방법 |
|---|---|
| PR마다 큰 저장소 전체를 읽어 비용 폭증 | 프롬프트에 **"변경된 파일만"** 명시, `--max-turns`, 비용 상한, 작은 모델(Sonnet·Haiku) |
| 외부 포크 PR이 시크릿을 빼냄 | `if: github.event.pull_request.head.repo.full_name == github.repository` 로 외부 PR 제외 |
| PR 본문·코드 주석의 숨은 지시(프롬프트 인젝션) | Claude 단계는 **읽기 전용 + 코멘트만**. 푸시·배포 권한 없음 |
| 워크플로우 파일을 몰래 바꿔 권한 확대 | `.github/workflows/`에 CODEOWNERS 지정 |
| 사용량을 아무도 모름 | `--output-format json` 결과의 사용량을 아티팩트로 저장, 주간 리포트 |

> 💬 **강사 노트:** "CI에 Claude를 붙이면 리뷰어가 필요 없나요?" → 아닙니다. Claude 리뷰는 **보조 리뷰어**입니다. 승인 버튼은 사람만 누른다는 7주 규칙을 그대로 유지합니다.

---

# 2. Claude Agent SDK: 내 프로그램 안의 에이전트 (모듈 18)

---

## 2-1. CLI·Actions·SDK 중 무엇을 쓸까

Claude Code를 움직이는 **같은 에이전트 루프**(1회차 2-3)를 내 프로그램 안에서 코드로 제어하는 것이 Agent SDK입니다.

| 상황 | 추천 |
|---|---|
| 개발자가 직접 쓰는 작업 | Claude Code (터미널·IDE·데스크톱) |
| CI에서 정해진 시점에 한 번 실행 | `claude -p` 또는 `claude-code-action` |
| **사내 도구·서비스 안에** 에이전트 기능을 넣어야 함 (사내 포털 버튼, 슬랙 봇, 배치 서버) | **Agent SDK** (TypeScript `@anthropic-ai/claude-agent-sdk`, Python `claude-agent-sdk`) |
| 에이전트 루프가 필요 없는 단순 생성·분류 | SDK 대신 Claude API(Messages API) 직접 호출 (차이는 [2-2](#2-2-claude-api를-직접-호출하는-것과-agent-sdk를-쓰는-것은-무엇이-다른가)) |

## 2-2. Claude API를 직접 호출하는 것과 Agent SDK를 쓰는 것은 무엇이 다른가

둘 다 "내 코드에서 Claude를 부른다"는 점은 같지만, **누가 무엇을 맡는지**가 다릅니다. 한 줄로 정리하면:

- **Claude API(Messages API)**: 내 코드가 **질문 한 번을 보내고 답 한 번을 받습니다.** 자료를 모으는 것, 도구를 실행하는 것, 다시 물을지 정하는 것은 모두 **내 코드**가 합니다
- **Agent SDK**: **Claude Code를 라이브러리로** 쓰는 것입니다. 목표를 주면 SDK 안의 에이전트 루프가 파일을 읽고, 명령을 실행하고, 다시 묻기를 **스스로 반복**합니다. 내 코드는 허용 범위(도구·권한·폴더·턴 수)를 정하고 결과를 받습니다

```mermaid
flowchart LR
    subgraph A["① Claude API 직접 호출"]
        direction TB
        A1["내 코드<br/>git log 실행 → 텍스트로"] -- "요청 1번<br/>(프롬프트 + 자료)" --> A2["Claude API<br/>POST /v1/messages"]
        A2 -- "응답 1번<br/>(텍스트)" --> A3["내 코드<br/>결과 저장·표시"]
    end
    subgraph B["② Agent SDK"]
        direction TB
        B1["내 코드<br/>query(목표, 허용 도구·폴더·턴 수)"] --> B2["Agent SDK<br/>(Claude Code의 에이전트 루프)"]
        B2 -- "요청" --> B3["Claude API"]
        B3 -- "도구 사용 요청<br/>(Bash: git log)" --> B2
        B2 -- "권한 확인 후 실행" --> B4["내장 도구<br/>Read · Grep · Bash…"]
        B4 -- "결과" --> B2
        B2 -- "result 메시지<br/>(답 + 사용량)" --> B5["내 코드"]
    end
```

> 💡 **Agent SDK도 안에서는 Claude API를 호출합니다.** Agent SDK는 Claude API를 대신하는 다른 서비스가 아니라, API 위에 Claude Code의 루프·도구·권한·컨텍스트 관리를 얹은 것입니다. 그래서 요금도 똑같이 **API 토큰 요금**이고, 한 번 실행하는 동안 API를 **여러 번** 부릅니다.

**비교**

| 비교 | ① Claude API 직접 호출 | ② Agent SDK |
|---|---|---|
| 설치하는 것 | Anthropic SDK (API 클라이언트) — Java `com.anthropic:anthropic-java`, TypeScript `@anthropic-ai/sdk`, Python `anthropic` 등 | Agent SDK — TypeScript `@anthropic-ai/claude-agent-sdk`, Python `claude-agent-sdk` (**Java용은 없음**) |
| 내 코드가 부르는 것 | `client.messages().create(...)` → 답 하나 | `query(prompt, options)` → 진행 메시지가 차례로, 마지막에 `result` |
| 한 번 실행 = API 호출 | **1번** (도구를 쓰면 내가 루프를 돌리는 만큼) | 목표를 이룰 때까지 **여러 번** (`maxTurns`로 상한) |
| 자료 (코드·로그·DB) | **내 코드가 모아서** 프롬프트에 넣는다. Claude는 내 PC·서버를 볼 수 없다 | Claude가 **스스로** 파일을 읽고 명령을 실행해 찾는다 (허용한 범위 안에서) |
| 도구 | 없음. 필요하면 내가 정의하고(tool use), **실행도 내 코드가** | **내장 도구**(Read, Write, Edit, Bash, Grep, Glob, WebSearch 등) + MCP 서버 + 서브에이전트 |
| 권한·안전 | 내 코드가 정한다 (Claude는 내가 넘긴 것만 본다) | `allowedTools`·`disallowedTools`·권한 모드·작업 폴더·훅으로 정한다 (Claude Code의 권한 체계와 같음) |
| 대화 기록·컨텍스트 | 내가 `messages` 배열로 관리 | SDK가 관리 (세션 이어가기, 길어지면 자동 압축) |
| 결과 형식 | 내가 정한 형식 그대로 (JSON 스키마로 고정할 수도 있음) | 마지막 답 텍스트 + 사용량·비용·턴 수 |
| 토큰 | 프롬프트 + 자료 + 답만 | 그 위에 Claude Code의 시스템 지시, 도구 설명, 도구 결과가 **매 턴** 더해진다 → 같은 일이면 보통 더 많다 |
| 실행 환경 | API에 접속되는 곳이면 어디든 (Spring 서버 안 등) | 작업 폴더(저장소 체크아웃 등)가 있는 서버. 내부에서 Claude Code를 실행 |
| 결과가 매번 같은가 | 입력이 같으면 비슷하다 | Claude가 무엇을 읽을지 스스로 정하므로 실행마다 경로가 달라질 수 있다 |

**같은 일을 두 방식으로: 릴리스 노트 초안** ([7-1](#7-1-sdk-미니-봇-릴리스-노트-초안)의 SDK 봇과 같은 일)

```ts
// ① Claude API 직접 호출 — git log는 내 코드가 실행하고, Claude에게는 한 번 묻는다
import Anthropic from "@anthropic-ai/sdk";
import { execFileSync } from "node:child_process";

const from = process.argv[2] ?? "v0.7.0";

// 자료는 내 코드가 모은다 (Claude는 저장소를 볼 수 없다)
const log = execFileSync("git", ["log", "--no-merges", "--pretty=format:%h %s", `${from}..HEAD`], {
  encoding: "utf8",
});

const client = new Anthropic(); // ANTHROPIC_API_KEY 환경 변수
const response = await client.beta.messages.create({
  model: "claude-opus-5-5",
  max_tokens: 16000,
  betas: ["server-side-fallback-2026-07-01"],
  fallbacks: "default", // 안전 분류기가 거절하면 다른 모델로 자동 재시도
  system: "너는 릴리스 노트 작성자다. 주어진 커밋 목록만 근거로 쓴다.",
  messages: [
    {
      role: "user",
      content: `다음 커밋으로 릴리스 노트 초안을 마크다운으로 써줘.
- feat / fix / perf / 기타로 분류, 요구사항 ID(FR-xx, NFR-xx)가 있으면 함께 표시
- 사용자에게 보이는 변경을 먼저, 내부 변경은 마지막에 한 줄씩

<commits>
${log}
</commits>`,
    },
  ],
});

// 결과 처리도 내 코드가
if (response.stop_reason === "refusal") {
  console.error("거절됨:", response.stop_details?.category);
  process.exit(1);
}
for (const block of response.content) {
  if (block.type === "text") console.log(block.text);
}
console.error(`입력 ${response.usage.input_tokens} / 출력 ${response.usage.output_tokens} 토큰`);
```

```ts
// ② Agent SDK — "git log로 확인해서"까지 Claude에게 맡긴다 (전체 코드는 7-1)
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: `${from}..HEAD 사이의 커밋을 git log로 확인하고 릴리스 노트 초안을 써줘 ...`,
  options: {
    cwd: process.cwd(),                                   // Claude가 읽을 수 있는 폴더
    allowedTools: ["Bash(git log *)", "Bash(git show *)"], // 명령은 git 조회만 (읽기는 cwd 안이면 기본 허용)
    disallowedTools: ["Write", "Edit"],
    permissionMode: "dontAsk",                            // 허용 안 된 것은 묻지 않고 거부
    maxTurns: 10,                                         // API 호출 반복 상한
  },
})) {
  if (message.type === "result") console.log(message);
}
```

| 차이 | ① API | ② Agent SDK |
|---|---|---|
| `git log`를 누가 실행하나 | 내 코드 (`execFileSync`) | Claude가 Bash 도구로 (허용한 `git log *`만) |
| 커밋 메시지만으로 부족하면 | 그대로 쓴다 (내가 넘긴 것만 본다) | `git show`로 변경 내용을 더 읽어 볼 수 있다 |
| 고를 이유 | 자료가 **미리 정해져** 있다 → 싸고, 빠르고, 결과가 예측 가능 | **무엇을 볼지 미리 정하기 어렵다** → 탐색을 맡긴다 |

> 💡 **Java(Spring) 개발자라면**: Agent SDK는 TypeScript·Python만 있으므로, **Spring 서비스 안에서는 Claude API를 직접 호출하는 방식(①)이 기본**입니다. 에이전트 루프가 꼭 필요하면 TypeScript·Python으로 작은 도구를 따로 두거나(아래 예제 ①), Java SDK의 도구 사용(tool use)으로 루프를 직접 짭니다. Spring에 넣는 모양은 아래 예제 ②.

**어느 쪽을 고를까**

| 상황 | 고를 것 | 예 |
|---|---|---|
| 입력이 정해져 있고 답 하나면 된다 (분류·요약·추출·번역·초안) | **① API** | 문의 메일 분류, 회의록 요약, 불량 보고서에서 항목 뽑기 |
| 정해진 순서가 있고 Claude가 단계 사이에 판단만 한다 | **① API** + 내 코드의 흐름 (필요하면 tool use) | "조회 → 요약 → 사람 승인 → 저장" 업무 흐름 |
| 무엇을 읽고 실행할지 **Claude가 찾아가야** 한다 (코드베이스·로그·파일 탐색) | **② Agent SDK** | CI 실패 원인 찾기, 저장소를 읽고 문서 갱신 제안, 여러 파일에 걸친 점검 |
| 사람이 한 번 쓰고 끝나는 작업 | SDK 대신 Claude Code 또는 `claude -p` (2-1) | |

> ⚠️ **헷갈리기 쉬운 것**
> - **"Agent SDK = Claude API의 Java·TypeScript 클라이언트"가 아닙니다.** API 클라이언트는 Anthropic SDK(`anthropic-java`, `@anthropic-ai/sdk`)이고, Agent SDK는 그 위의 **에이전트 실행기**입니다
> - **API만으로도 에이전트를 만들 수 있습니다.** 도구를 정의하고 "도구 요청 → 내 코드가 실행 → 결과를 돌려주기"를 반복하면 됩니다. Anthropic SDK에는 이 반복을 대신해 주는 **Tool Runner**도 있습니다. 다만 도구는 모두 내가 만들어야 하고, 파일 읽기·명령 실행 같은 내장 도구는 없습니다
> - **Agent SDK가 더 똑똑한 것이 아닙니다.** 같은 모델을 씁니다. 차이는 **누가 자료를 모으고 도구를 실행하느냐**이고, 그만큼 토큰·실행 시간·예측 가능성이 달라집니다
> - 참고: 루프와 실행 환경(컨테이너)까지 Anthropic이 운영해 주는 **Managed Agents**(베타)라는 방식도 있습니다. 이 과정에서는 다루지 않습니다

> 💡 **시작은 ①부터.** "에이전트를 만들자"보다 "이 일에 필요한 자료를 내 코드가 모을 수 있나?"를 먼저 묻습니다. 모을 수 있으면 ①이 싸고 빠르고 테스트하기 쉽습니다. 2-4의 제조 도구 키트도 같은 생각입니다: 자료를 가져오는 일(MCP 도구)은 코드가 정확하게 하고, Claude는 판단을 맡습니다.

> ✅ 이 교재 환경에서 위 TypeScript 두 예제를 `@anthropic-ai/sdk` 1.14.0, `@anthropic-ai/claude-agent-sdk` 0.3.332로 타입 검사(TypeScript 7.0, `strict`)했습니다. Agent SDK는 아래 예제 ①로 실제 실행했고, ① API 방식은 API 키가 없어 **실제 호출은 하지 못했습니다.** 수업 전에 강사 키로 ①을 한 번 실행해 출력과 토큰 수를 확인하고, ②의 `result` 메시지(사용량·비용·턴 수)와 비교해 보여 주면 토큰 차이를 바로 볼 수 있습니다.

### 예제 ① JavaScript: Agent SDK로 "테스트 실패 원인 분석" 봇

CI나 사내 서버에서 테스트가 실패하면, 결과 XML과 코드를 **Claude가 스스로 찾아 읽고** 원인과 수정 방향을 보고하는 작은 도구입니다. 무엇을 읽어야 할지 미리 정하기 어려우므로 Agent SDK가 맞는 일입니다 (위 "어느 쪽을 고를까" 표). 코드는 고치지 않습니다.

```json
// tools/test-doctor/package.json
{
  "name": "eduhub-test-doctor",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "start": "node index.mjs"
  },
  "dependencies": {
    "@anthropic-ai/claude-agent-sdk": "^0.3.332"
  }
}
```

```js
// tools/test-doctor/index.mjs — 실패한 백엔드 테스트의 원인을 찾아 보고하는 사내 도구 (읽기 전용)
// 실행: node index.mjs <저장소 폴더>   (ANTHROPIC_API_KEY 환경 변수 필요)
import { query } from "@anthropic-ai/claude-agent-sdk";
import { resolve } from "node:path";

const repoDir = resolve(process.argv[2] ?? "."); // Claude가 볼 수 있는 폴더 (저장소 체크아웃)

const prompt = `
backend/build/test-results/test/ 아래의 JUnit 결과(XML)에서 실패한 테스트를 찾고,
각 실패의 원인을 테스트 코드와 대상 코드를 읽어 분석해줘.

[보고 형식] 실패한 테스트마다:
- 테스트 이름
- 원인 (파일:줄, 3줄 이내)
- 수정 방향 (코드는 고치지 말 것)
실패가 없으면 "실패 없음"과 확인한 파일 목록만.
`;

const run = query({
  prompt,
  options: {
    cwd: repoDir,
    // ① 쓸 수 있는 도구를 좁힌다
    //    Read·Grep·Glob은 cwd 안에서는 허용 목록 없이도 쓸 수 있다. 여기에 "Read"처럼 범위 없이 넣으면
    //    cwd 밖(서버의 다른 폴더)까지 허용되므로 넣지 않는다. 명령은 git 조회만 허용
    allowedTools: ["Bash(git log *)", "Bash(git diff *)"],
    disallowedTools: ["Write", "Edit", "WebFetch", "WebSearch"],
    // 허용되지 않은 것(cwd 밖 읽기, 다른 명령)은 묻지 않고 거부 (서버에는 승인할 사람이 없으므로)
    permissionMode: "dontAsk",
    // ② 한 번 실행에 쓸 수 있는 양을 제한한다
    maxTurns: 15,
    maxBudgetUsd: 0.5,
    // ③ 개인 PC 설정(~/.claude)이 섞이지 않게 저장소의 설정(.claude/, CLAUDE.md)만 읽는다
    settingSources: ["project"],
  },
});

for await (const message of run) {
  // 진행 상황: Claude가 어떤 도구를 쓰는지 한 줄씩 (stderr)
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "tool_use") {
        console.error(`  → ${block.name} ${JSON.stringify(block.input).slice(0, 80)}`);
      }
    }
  }

  // 마지막 결과 (stdout)
  if (message.type === "result") {
    if (message.subtype === "success") {
      console.log(message.result);
    } else {
      // error_max_turns, error_max_budget_usd, error_during_execution …
      console.error(`실패: ${message.subtype}`, message.errors);
      process.exitCode = 1;
    }
    console.error(
      `턴 ${message.num_turns}회 · 비용 $${message.total_cost_usd.toFixed(4)} · ` +
        `입력 ${message.usage.input_tokens} (+캐시 읽기 ${message.usage.cache_read_input_tokens}) / ` +
        `출력 ${message.usage.output_tokens} 토큰`,
    );
  }
}
```

```bash
cd tools/test-doctor
npm install
node index.mjs ../..          # 저장소 루트를 넘긴다 (먼저 ./gradlew test로 결과 XML을 만들어 둔다)
```

| 옵션 | 하는 일 | 빼면 |
|---|---|---|
| `cwd` | Claude가 읽을 수 있는 폴더 (저장소 체크아웃) | 실행한 폴더가 기준이 된다 |
| `allowedTools` | 묻지 않고 쓸 수 있는 도구. 명령은 **`git log`·`git diff`만** | — |
| `permissionMode: "dontAsk"` | 허용되지 않은 것은 **묻지 않고 거부**. 서버에는 승인 버튼을 누를 사람이 없다 | 승인을 기다리거나 실패 |
| `disallowedTools` | 쓰기·웹 도구를 확실히 금지 (2중 안전장치) | — |
| `maxTurns`, `maxBudgetUsd` | 반복 횟수·비용 상한. 넘으면 `result`가 `error_max_turns`·`error_max_budget_usd` | 탐색이 길어지면 비용이 계속 늘어난다 |
| `settingSources: ["project"]` | 저장소의 `.claude/`·`CLAUDE.md`만 읽는다 | 서버 계정의 `~/.claude` 설정이 섞일 수 있다 |

> ⚠️ **`allowedTools`에 `"Read"`, `"Glob"`을 범위 없이 넣지 않습니다.** 이 교재 환경에서 처음에 `allowedTools: ["Read", "Grep", "Glob", …]`으로 실행했더니, 작업 폴더에서 결과 파일을 찾지 못한 Claude가 **작업 폴더 밖의 상위 폴더까지 `Glob`으로 검색해** 다른 프로젝트의 테스트 결과를 찾아냈습니다. `cwd`는 기본 위치일 뿐 **울타리가 아닙니다.** 읽기 도구는 `cwd` 안에서는 허용 목록이 없어도 쓸 수 있으므로 목록에서 빼고, `dontAsk`로 나머지를 거부하게 했습니다 (위 코드). 서버에서 더 단단히 막으려면 전용 계정·컨테이너 안에서 실행합니다.

**실제 실행 결과** (이 교재 환경, 정원 비교를 `<=`로 잘못 쓴 작은 예제 저장소)

```text
  → Bash {"command":"cd backend/build/test-results/test/ 2>&1 && ls -la && grep -l -E '<(
  → Bash {"command":"cd …
  → Read {"file_path":"…/backend/build/test-results/test/TEST-com.example.eduhub.SeatCounterTest.xml"
  → Read {"file_path":"…/backend/src/test/java/com/example/eduhub/SeatCounterTest.java"
  → Read {"file_path":"…/backend/src/main/java/com/example/eduhub/SeatCounter.java"
실패한 테스트는 1개입니다. …

**테스트 이름:** `SeatCounterTest.정원이_차면_신청할_수_없다()`
**원인:** `backend/src/main/java/com/example/eduhub/SeatCounter.java:5`
- `canEnroll()`이 `enrolled <= capacity`로 비교합니다. 그래서 정원이 꽉 찬 경우(`enrolled == capacity`)에도 `true`를 반환합니다. …
**수정 방향:** 비교 연산자를 `<=`에서 `<`로 바꾸면 됩니다(`enrolled < capacity`). …
턴 6회 · 비용 $0.3166 · 입력 10 (+캐시 읽기 126713) / 출력 1327 토큰
```

- `Bash(cd … && ls …)`는 허용 목록(`git log`·`git diff`)에 없어 **거부**되었고, Claude는 `Read`로 방법을 바꿨습니다
- 입력 토큰 대부분이 **캐시 읽기**(Claude Code의 시스템 지시·도구 설명)입니다. 작은 저장소에서도 한 번에 약 $0.3: API 한 번 호출(①)보다 비싼 이유가 이것입니다 (2-2 비교표의 "토큰")

### 예제 ② Java(Spring): Claude API로 "수강 문의 분류·답변 초안"

수강 문의 한 건을 받아 **분류, 한 줄 요약, 담당자 확인 필요 여부, 답변 초안**을 돌려주는 Spring 서비스입니다. 입력이 정해져 있고 답 하나면 되므로 API를 한 번 호출합니다. 응답을 **Java record로 바로 받도록**(structured output) 해서 JSON을 직접 파싱하지 않습니다.

```kotlin
// backend/build.gradle.kts (추가)
dependencies {
    implementation("com.anthropic:anthropic-java:3.3.0")   // Claude API 클라이언트 (Anthropic Java SDK)
}
```

```java
// ClaudeConfig.java
package com.example.eduhub.inquiry;

import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/** Claude API 클라이언트는 애플리케이션에 하나만 만들어 재사용한다 (연결 풀을 공유) */
@Configuration
public class ClaudeConfig {

    @Bean(destroyMethod = "close")
    public AnthropicClient anthropicClient() {
        // ANTHROPIC_API_KEY 환경 변수에서 키를 읽는다. 키를 코드·application.yml에 넣지 않는다
        return AnthropicOkHttpClient.fromEnv();
    }
}
```

```java
// InquiryTriage.java — 결과 타입
package com.example.eduhub.inquiry;

import com.fasterxml.jackson.annotation.JsonPropertyDescription;

/** Claude가 돌려줄 결과의 모양. 이 record에서 JSON 스키마가 만들어지고, 응답이 이 타입으로 바로 바뀐다 */
public record InquiryTriage(
        @JsonPropertyDescription("문의 종류") Category category,
        @JsonPropertyDescription("문의 내용 한 줄 요약") String summary,
        @JsonPropertyDescription("담당자가 직접 답해야 하면 true (환불, 개인정보, 불만 등)") boolean needsHuman,
        @JsonPropertyDescription("담당자가 다듬어 보낼 답변 초안, 3문장 이내") String replyDraft) {

    public enum Category { ENROLLMENT, CANCELLATION, SCHEDULE, CERTIFICATE, OTHER }
}
```

```java
// InquiryTriageService.java
package com.example.eduhub.inquiry;

import com.anthropic.client.AnthropicClient;
import com.anthropic.errors.AnthropicIoException;
import com.anthropic.errors.AnthropicServiceException;
import com.anthropic.errors.RateLimitException;
import com.anthropic.models.messages.Message;
import com.anthropic.models.messages.MessageCreateParams;
import com.anthropic.models.messages.StopReason;
import com.anthropic.models.messages.StructuredMessage;
import com.anthropic.models.messages.StructuredMessageCreateParams;
import org.springframework.stereotype.Service;

/**
 * 수강 문의 한 건을 분류하고 답변 초안을 만든다.
 * 에이전트 루프가 필요 없는 일(입력이 정해져 있고 답 하나)이므로 Claude API를 한 번 호출한다.
 */
@Service
public class InquiryTriageService {

    private static final String SYSTEM = """
            너는 EduHub 교육 운영팀의 문의 담당 보조다.
            문의를 분류하고, 담당자가 다듬어 보낼 답변 초안을 쓴다.
            모르는 정보(일정, 금액, 규정)는 지어내지 말고 "담당자가 확인 후 안내" 라고 쓴다.
            """;

    private final AnthropicClient client;

    public InquiryTriageService(AnthropicClient client) {
        this.client = client;
    }

    public InquiryTriage triage(String inquiry) {
        StructuredMessageCreateParams<InquiryTriage> params = MessageCreateParams.builder()
                .model("claude-opus-5-5")
                .maxTokens(4000L)
                .system(SYSTEM)
                .outputConfig(InquiryTriage.class)          // 응답을 InquiryTriage JSON으로 고정
                // 문의 내용은 사용자가 쓴 데이터: 태그로 감싸 지시와 구분한다
                .addUserMessage("<inquiry>\n" + inquiry + "\n</inquiry>")
                .build();

        StructuredMessage<InquiryTriage> response;
        try {
            response = client.messages().create(params);
        } catch (RateLimitException e) {
            throw new IllegalStateException("요청이 많습니다. 잠시 후 다시 시도하세요.", e);   // 429 (SDK가 이미 2번 재시도한 뒤)
        } catch (AnthropicServiceException e) {
            throw new IllegalStateException("Claude API 오류: " + e.statusCode(), e);            // 400·401·5xx 등
        } catch (AnthropicIoException e) {
            throw new IllegalStateException("Claude API에 연결할 수 없습니다.", e);               // 네트워크
        }

        Message raw = response.rawMessage();
        if (raw.stopReason().filter(StopReason.REFUSAL::equals).isPresent()) {
            throw new IllegalStateException("요청이 거절되었습니다. 담당자가 직접 확인하세요.");
        }
        if (raw.stopReason().filter(StopReason.MAX_TOKENS::equals).isPresent()) {
            throw new IllegalStateException("답이 잘렸습니다. maxTokens를 늘리세요.");
        }

        return response.content().stream()
                .flatMap(block -> block.text().stream())
                .map(text -> text.text())                   // 이미 InquiryTriage 타입
                .findFirst()
                .orElseThrow(() -> new IllegalStateException("응답에 결과가 없습니다."));
    }
}
```

<details>
<summary>Spring 없이 한 번 실행해 보기: <code>InquiryTriageDemo.java</code></summary>

```java
package com.example.eduhub.inquiry;

import com.anthropic.client.AnthropicClient;

/** Spring 없이 한 번 실행해 보는 용도: ./gradlew run (ANTHROPIC_API_KEY 필요) */
public class InquiryTriageDemo {

    public static void main(String[] args) throws Exception {
        AnthropicClient client = new ClaudeConfig().anthropicClient();
        try {
            InquiryTriageService service = new InquiryTriageService(client);
            InquiryTriage result = service.triage("""
                    안녕하세요. 다음 주 화요일 'Spring 트랜잭션 심화' 2회차를 신청했는데
                    회사 일정 때문에 참석이 어렵습니다. 취소하고 3회차로 옮길 수 있나요?
                    """);
            System.out.println("분류: " + result.category());
            System.out.println("요약: " + result.summary());
            System.out.println("담당자 확인: " + result.needsHuman());
            System.out.println("답변 초안: " + result.replyDraft());
        } finally {
            client.close();   // Spring에서는 @Bean(destroyMethod = "close")가 대신 한다
        }
    }
}
```

</details>

| 코드 | 이유 |
|---|---|
| 클라이언트를 `@Bean` 하나로 | 요청마다 만들면 연결을 매번 새로 연다. SDK가 429·5xx·네트워크 오류를 **기본 2번 재시도**한다 |
| `outputConfig(InquiryTriage.class)` | record에서 JSON 스키마를 만들고, 응답이 그 스키마를 지키도록 API가 보장한다. `@JsonPropertyDescription`이 각 칸의 설명 |
| `<inquiry>` 태그 | 문의 내용은 **사용자가 쓴 데이터**. "이전 지시를 무시하고…" 같은 문장이 있어도 지시와 섞이지 않게 구분한다 |
| `stopReason` 확인 | `refusal`(거절)·`max_tokens`(잘림)이면 결과를 쓰지 않는다 |
| 예외를 종류별로 | 429는 잠시 후 다시, 400·401은 설정 문제, 네트워크는 연결 문제. EduHub에서는 7회차 `BizException`·`ErrorCode`로 바꿔 던진다 |
| 결과는 **초안** | `needsHuman`이 `true`거나 아니거나, 답변은 담당자가 확인한 뒤 보낸다 (금액·일정은 지어내지 않게 지시) |

> ✅ 이 교재 환경에서 위 Java 코드를 `anthropic-java` 3.3.0, Spring 6.2.19, Java 21로 컴파일했습니다. 실행은 API 키가 없어 `401 authentication_error`가 났고, 서비스의 예외 처리(`AnthropicServiceException` → "Claude API 오류: 401")가 동작하는 것까지 확인했습니다. ⚠️ **정상 응답(분류 결과)은 확인하지 못했습니다.** 수업 전에 강사 키로 `InquiryTriageDemo`를 한 번 실행해 결과를 확인합니다.

> 💡 **두 예제의 인증**: 위 Agent SDK 예제가 이 교재 환경에서 키 없이 실행된 것은 SDK가 내부에서 Claude Code를 실행하고, 그 Claude Code가 **이미 로그인된 인증**을 썼기 때문입니다. 사내 도구로 서버에 배포할 때는 두 예제 모두 **API 키(`ANTHROPIC_API_KEY`)를 비밀 저장소·환경 변수로** 주고, 개인 로그인을 쓰지 않습니다.

## 2-3. SDK 에이전트의 동작 시퀀스

```mermaid
sequenceDiagram
    participant App as 사내 도구 (내 코드)
    participant SDK as Agent SDK
    participant M as Claude 모델
    participant T as 도구 (Read, Grep, Bash…)
    App->>SDK: query(prompt, options: 허용 도구·권한·작업 폴더)
    loop 목표를 이룰 때까지 (maxTurns 이내)
        SDK->>M: 지시 + 컨텍스트
        M-->>SDK: 도구 사용 요청
        SDK->>SDK: 권한 확인 (허용 목록·권한 모드)
        SDK->>T: 도구 실행
        T-->>SDK: 결과
    end
    M-->>SDK: 최종 답변
    SDK-->>App: result 메시지 (결과·사용량)
```

**SDK로 만들 때 코드로 정해야 하는 것**

| 항목 | 예 (EduHub 릴리스 노트 봇) |
|---|---|
| 허용 도구 | 명령은 `Bash(git log *)`만, 권한 모드 `dontAsk`. 읽기는 작업 폴더 안만 (`Read`·`Grep`·`Glob`을 범위 없이 허용하지 않음, [예제 ①](#예제-①-javascript-agent-sdk로-테스트-실패-원인-분석-봇)) — 쓰기 도구는 금지 |
| 작업 폴더 | 저장소 체크아웃 경로 하나 |
| 턴 수·비용 상한 | 턴 10회 |
| 결과 처리 | `result` 메시지를 받아 사내 위키나 PR 코멘트로 |
| 비밀 | API 키는 환경 변수·비밀 저장소. 코드에 넣지 않음 |

> 자세한 예제: [Agent SDK 예제](../examples/05-automation-and-capstone.md#모듈-18-claude-agent-sdk), [SDK 동작 시퀀스](../examples/10-agents.md#5-agent-sdk로-만든-커스텀-에이전트의-동작-시퀀스). 직접 만들어 보기는 [7-1](#7-1-sdk-미니-봇-릴리스-노트-초안)에서.

## 2-4. 조직의 도구 표준: "제조 도구 개발 키트"를 만들어 그 위에서 개발한다

5회차 [2-5](./session-05-hooks-and-mcp.md#2-5-사례-제조-현장의-ai-에이전트와-mcp-meserp-연결)에서 MES·ERP 정보를 가져오는 MCP 도구를 만들었습니다. 회사 전체로 넓히면 생산·품질·설비·물류 팀이 **저마다 도구를 만들게** 됩니다. 이때 각 팀이 마음대로 만들면 금방 문제가 생깁니다.

| 팀마다 제각각이면 | 예 | 결과 |
|---|---|---|
| 이름·설명이 다르다 | `get_line_status`, `lineInfo`, `라인상태조회` | 에이전트가 **비슷한 도구 중 엉뚱한 것**을 고른다 |
| 단위·시각·코드 표기가 다르다 | 분/초, KST/UTC, 라인 `L1`/`LINE-01` | 숫자를 잘못 비교·합산한다 (사람도 모른 채) |
| 오류 형식이 다르다 | 예외 그대로, 빈 결과, `"error"` 문자열 | 에이전트가 실패를 "데이터 없음"으로 오해한다 |
| 권한·감사가 빠진다 | 쓰기 도구, 결과 건수 제한 없음, 조회 기록 없음 | 보안 사고·운영 DB 부하, 사고가 나도 추적 불가 |
| 같은 코드를 팀마다 다시 짠다 | DB 연결, 인자 검사, 마스킹 | 비용 증가, 고칠 때 모든 팀을 찾아다녀야 함 |

그래서 **조직이 도구 개발 표준과 그것을 담은 키트(SDK)를 만들고, 각 팀은 그 위에서 도구를 만드는 것**이 바람직합니다. 7회차의 사내 공통 모듈 `corp-common`(공통 응답·예외·`CurrentUser`)과 같은 생각을 **에이전트 도구**에 적용하는 것입니다.

### ① 무엇을 직접 만들고, 무엇은 표준을 그대로 쓰나

> ⚠️ **"사내 에이전트 SDK"를 처음부터 새로 만들지 않습니다.** 에이전트와 도구가 주고받는 규약(MCP), 도구 서버를 만드는 라이브러리(MCP Java·TypeScript·Python SDK), 에이전트 루프(Claude Agent SDK)는 이미 표준이 있습니다. 직접 만들면 표준이 바뀔 때마다 따라가느라 비용이 들고, 다른 에이전트·도구와 연결도 끊깁니다. 조직이 만드는 것은 **표준 위에 얹는 얇은 키트와 규칙**입니다.

```mermaid
flowchart TB
    subgraph USE["쓰는 쪽 (에이전트)"]
        A1["Claude Code<br/>(개발자)"]
        A2["Agent SDK 앱<br/>매일 아침 납기 위험 리포트"]
        A3["사내 챗봇 등<br/>MCP를 지원하는 다른 에이전트"]
    end
    subgraph TOOLS["각 팀이 만드는 도구 서버 (MCP 서버)"]
        T1["mes-tools<br/>생산팀"]
        T2["qms-tools<br/>품질팀"]
        T3["erp-tools<br/>구매·물류팀"]
    end
    subgraph KIT["조직이 만드는 것: 제조 도구 개발 키트 (mfg-tool-kit)"]
        K1["도구 정의 도우미<br/>이름·설명·스키마 규칙 검사"]
        K2["공통 입력 타입<br/>라인 ID·품번·기간"]
        K3["공통 결과 형식<br/>단위·기준 시각·출처"]
        K4["읽기 전용 기본·권한<br/>결과 건수·시간 제한·마스킹"]
        K5["감사 로그<br/>누가·무엇을·몇 건"]
        K6["테스트 도구<br/>계약 테스트·평가 질문 세트"]
    end
    subgraph STD["표준 (그대로 사용)"]
        S1["MCP 프로토콜"]
        S2["MCP Java SDK 등"]
        S3["Claude Agent SDK"]
    end
    A1 & A2 & A3 -- "MCP" --> T1 & T2 & T3
    T1 & T2 & T3 --> KIT
    KIT --> S2
    A2 --> S3
    S2 --- S1
```

| 층 | 누가 | 내용 |
|---|---|---|
| **표준** | 외부 (그대로 사용) | MCP 프로토콜, MCP SDK, Claude Agent SDK. 버전 업데이트는 키트 담당이 먼저 시험 |
| **제조 도구 개발 키트** | **플랫폼(공통) 팀** | 아래 ②의 표준을 코드로 담은 라이브러리 + 템플릿 + 문서 |
| **도구 서버** | 각 업무 팀 | 키트로 자기 시스템(MES·QMS·ERP)의 도구를 만든다. 업무 지식은 이 팀에 있다 |
| **에이전트** | 각 서비스 팀 | 승인된 도구 서버만 연결해서 쓴다 (3장 거버넌스) |

### ② 표준에 담을 것

| 항목 | 표준 예 | 이유 |
|---|---|---|
| **도구 이름** | `<시스템>_<동사>_<대상>`: `mes_get_line_status`, `qms_list_defects` | 여러 서버를 함께 붙여도 이름이 겹치지 않고, 에이전트가 어느 시스템인지 안다 |
| **설명** | ① 무엇을 돌려주나 ② 언제 쓰나(예시 질문) ③ 단위·기간 한도 ④ "읽기 전용" — 네 가지를 반드시 | 에이전트는 **설명을 보고 도구를 고른다** (7회차 스킬의 description과 같은 원리) |
| **공통 입력 타입** | 라인 ID 형식, 품번 형식, 기간(`from`~`to`, 최대 31일) | 팀마다 검사 규칙이 다르면 같은 값이 어떤 도구에서는 실패 |
| **공통 결과 형식** | `{ "data": …, "meta": { "source": "MES", "asOf": "2026-11-22T09:00+09:00", "units": {"downtime":"분"} } }` | 숫자의 **단위·기준 시각·출처**를 모르면 사람도 에이전트도 잘못 판단 |
| **오류** | 사람이 읽을 메시지 + 재시도해도 되는지 + 오류 코드 (`isError` 결과) | 실패를 "데이터 없음"으로 오해하지 않게 |
| **안전** | **읽기 전용이 기본**, 쓰기 도구는 별도 승인·사람 확인 흐름, 결과 최대 건수·조회 시간 제한, 개인정보·원가 마스킹 | 에이전트는 사람보다 훨씬 자주, 많이 부른다 |
| **감사 로그** | 사용자·도구·인자·결과 건수·소요 시간을 별도 저장소에 | 사고 추적, 사용량·비용 파악 |
| **버전** | 도구의 입력·결과 형식을 바꾸면 버전을 올리고 변경 기록 | 그 도구를 쓰는 에이전트·프롬프트가 조용히 깨지지 않게 |
| **테스트** | 계약 테스트(입력 검사·결과 형식), **평가 질문 세트**("1라인 지금 어때?" → 이 도구를 고르고 이 값을 답하는가) | 도구가 많아질수록 "에이전트가 올바른 도구를 고르는가"가 품질의 핵심 |

### ③ 예시: 실제로 만든 키트 `mfg-tool-kit`

예제 34의 [`ToolSupport`](../examples/34-manufacturing-ai-agent-mcp.md#6-5-공통-도우미-toolsupportjava)(결과·오류 만들기, 인자 꺼내기)가 바로 키트의 씨앗입니다. 팀마다 이것을 복사해 쓰는 대신, 플랫폼 팀이 규칙까지 넣어 **라이브러리로** 배포합니다. 이 교재에는 MCP Java SDK 위에 만든 **실행되는 키트와, 그것으로 만든 생산팀 도구 서버**가 있습니다 → [예제 36: mfg-tool-kit](../examples/36-mfg-tool-kit.md).

```java
// 생산팀이 키트로 만드는 도구 하나 (예제 36의 MesLineTools.java)
// 키트가 대신 해 주는 것: 이름 규칙 검사, 설명 4요소 검사·조립, 라인 ID 형식 검사, 읽기 전용 표시,
//                      결과를 {data, meta} 형식으로, 건수 제한, 마스킹, 감사 로그, 예외 → 표준 오류
SyncToolSpecification lineStatus = MfgTool.readOnly("mes_get_line_status")
        .title("생산 라인 현황")
        .returns("라인 하나의 오늘 양품·불량 수량, 불량률, 비가동 시간")
        .whenToUse("'지금 1라인 어때?', '오늘 2라인 불량률은?' 같은 질문")
        .param(MfgParams.lineId())                        // 공통 입력 타입 (형식 검사 포함)
        .units(Map.of("todayDefectRate", "%", "todayDowntime", "분"))
        .source("MES")
        .handler(args -> mes.lineStatus(args.lineId()))   // 팀이 쓰는 것은 업무 로직뿐
        .build();
```

에이전트가 받는 결과 (실제 실행):

```json
{"data":{"lineId":"L1","lineName":"조립 1라인","todayGoodQty":631,"todayDefectQty":20,"todayDefectRate":3.1,"todayDowntime":40},
 "meta":{"tool":"mes_get_line_status","source":"MES","asOf":"2026-11-22T21:50:51+09:00","units":{"todayDefectRate":"%","todayDowntime":"분"}}}
```

- 표준을 어긴 도구(`updateLineSpeed`, 설명 없음)는 **만드는 순간 실패**하고, 원가 같은 필드는 `mask(...)`로 `"***"`, 긴 목록은 `maxRows`로 잘리고, 모든 호출이 감사 로그에 남습니다
- MCP 통신·입력 스키마 검사·서버 실행은 MCP Java SDK가 그대로 합니다. 키트는 그 위에 회사 규칙만 얹습니다

> ✅ 예제 36의 키트(테스트 6개)와 도구 서버(표준 강제 테스트 2개)를 교재 환경에서 빌드하고, 실제 MCP 클라이언트로 stdio 연결해 도구 목록·호출·오류·감사 로그를 확인했습니다 (Java 21, MCP Java SDK 2.1.0).

### ④ 표준을 지키게 하는 법: 7회차 6단계 장치를 그대로

| 장치 | 제조 도구 키트에서 |
|---|---|
| ① CLAUDE.md | 도구 서버 저장소마다 "도구는 `mfg-tool-kit`으로만 만든다, 이름 규칙, 읽기 전용 기본" |
| ② 스킬 | `mfg-tool` 스킬: 새 도구 만들기 절차, 설명 작성법, 템플릿 |
| ③ 참조 코드 | 플랫폼 팀이 만든 모범 도구 하나 (예: `mes_get_line_status`) |
| ④ 소스·문서 | 키트 소스와 사용 문서 (7회차처럼 소스를 Claude가 읽을 수 있게) |
| ⑤ 강제 | ArchUnit: "도구는 `MfgTool`로만 만든다(MCP SDK의 `Tool`·`CallToolResult` 직접 사용 금지)", "`System.out` 금지" (예제 36의 `ToolKitRulesTest`), "DB는 읽기 전용 DataSource만"; CI의 계약 테스트·평가 질문 세트 |
| ⑥ 배포 | 키트는 사내 저장소(없으면 7회차 예제 35의 방법), 스킬·CLAUDE.md는 **플러그인**(6회차)으로 각 팀에 |

여기에 **승인된 도구 서버 목록**(어떤 MCP 서버를 어떤 에이전트가 쓸 수 있는지)을 더하면 3장의 거버넌스와 이어집니다.

### ⑤ 에이전트 쪽 표준도 함께: Agent SDK 설정을 사내 기본값으로

도구만큼 **도구를 쓰는 에이전트 앱**도 팀마다 제각각이 되기 쉽습니다. Agent SDK 앱을 만들 때 쓰는 **사내 기본 설정**을 정해 둡니다 (2-3 표의 "코드로 정해야 하는 것"을 회사 기준으로).

| 항목 | 사내 기본값 예 |
|---|---|
| 연결할 수 있는 도구 서버 | 승인 목록에 있는 MCP 서버만 |
| 허용 도구 | 읽기 도구만. 쓰기는 별도 승인 |
| 턴 수·비용 상한 | 턴 20회, 실행당 비용 한도 |
| 결과 처리 | 답변에 **근거(어느 도구의 어떤 값)**를 함께 남기기 |
| 기록 | 실행마다 사용량·호출한 도구를 감사 로그로 |

### ⑥ 처음부터 크게 만들지 않는다

1. 한 팀이 도구 **2~3개를 먼저** 만들어 써 본다 (5회차·예제 34처럼)
2. 두 번째 팀이 만들 때 **겹치는 부분**(인자 검사, 결과 형식, 감사 로그)을 키트로 뽑는다
3. 규칙은 **문서 → 템플릿 → 강제(테스트)** 순서로 단단하게 한다
4. 키트 담당(플랫폼 팀)과 변경 절차(버전, 변경 기록, 각 팀 공지)를 정한다

> 💬 **강사 노트:** 제조업 수강생에게 "우리 회사에서 도구를 만들 팀이 몇 개인가, 그 팀들이 공통으로 쓰는 코드(라인 ID, 품번, 근무 캘린더)는 무엇인가"를 묻고 키트의 첫 후보를 3개 적게 합니다. 시간이 부족하면 그림과 ② 표만 보여 주고 나머지는 읽어 오게 합니다.

---

# 3. 운영 & 거버넌스 (모듈 20)

---

## 3-1. 거버넌스는 왜 필요한가

개인이 쓸 때는 "조심해서 쓰면" 되지만, 조직 전체가 쓰면 **사람마다 설정이 달라** 사고가 납니다. 거버넌스는 "모두가 지켜야 하는 최소 규칙을 **강제**하는 장치"입니다.

```mermaid
flowchart TB
    ORG["🏢 조직: 관리 설정 (managed settings)<br/>누구도 덮어쓸 수 없음<br/>운영 비밀 deny, bypass 모드 금지, 승인된 MCP·마켓플레이스만"]
    TEAM["👥 팀: 프로젝트 .claude/settings.json (커밋, 리뷰)<br/>권한·훅·스킬·에이전트"]
    ME["👤 개인: .claude/settings.local.json, ~/.claude/settings.json<br/>개인 선호"]
    CI["⚙️ CI·SDK: 실행 옵션 (--allowedTools, --max-turns)"]
    ORG --> TEAM --> ME
    ORG --> CI
```

> 우선순위: **관리 설정 > CLI 옵션 > 프로젝트 개인 > 프로젝트 공유 > 사용자**. 단, **deny는 어느 단계에 있든 이깁니다.** ([권한 3-1](../examples/23-permissions.md#3-1-설정하는-곳))

## 3-2. 거버넌스 체크리스트

| 영역 | 해야 할 일 | Claude Code에서의 수단 |
|---|---|---|
| **보안 정책** | 운영 설정·비밀 읽기 금지, 위험 명령 금지 | 관리 설정의 `permissions.deny`, 필요하면 `allowManagedPermissionRulesOnly` |
| **권한 모드** | 승인 없이 다 하는 모드 금지 | `permissions.disableBypassPermissionsMode`, `permissions.disableAutoMode`를 `"disable"`로 |
| **외부 연결** | 승인된 MCP 서버만 | `managed-mcp.json`, `allowedMcpServers`·`deniedMcpServers` ([MCP 서버 가이드](../examples/19-mcp-server-guide.md)) |
| **플러그인** | 사내 마켓플레이스만 허용 | 관리 설정의 마켓플레이스 제한 (설정 키는 공식 문서 확인 필요) |
| **비용·사용량** | 팀·사람·용도별 사용량 확인, 이상 사용 감지 | 관리 콘솔의 사용량 화면, `/usage`, 헤드리스 JSON 출력, OpenTelemetry 텔레메트리 내보내기 (공식 문서 확인 필요) |
| **데이터 정책** | 운영 데이터·개인정보를 AI 도구에 넣지 않음 | 환경 분리, 마스킹 데이터, DB MCP는 개발 DB 읽기 전용 계정 |
| **감사** | 누가 무엇을 바꿨는지 추적 | Git 이력(`Co-Authored-By`), PR 리뷰 기록, CI 로그, `.claude/` CODEOWNERS |
| **품질 문화** | AI 코드도 사람 리뷰 필수 | 브랜치 보호, 필수 리뷰 1명, CI 필수 체크 |
| **AI 없이 운영할 역량** | 런북, 핫픽스 절차, 기능 플래그, **정기 훈련** | 오늘 6장 |

**관리 설정 예시 (조직 보안 담당이 배포)**

```json
{
  "permissions": {
    "deny": [
      "Read(**/.env)",
      "Read(**/.env.*)",
      "Read(**/application-prod.yml)",
      "Bash(kubectl * --context prod*)",
      "Bash(*deploy-backend*)"
    ],
    "disableBypassPermissionsMode": "disable"
  }
}
```

> 관리 설정은 claude.ai 관리 화면(서버 관리 설정), MDM·OS 정책(macOS `com.anthropic.claudecode`, Windows `HKLM\SOFTWARE\Policies\ClaudeCode`), 또는 시스템 경로의 파일(macOS `/Library/Application Support/ClaudeCode/`, Linux·WSL `/etc/claude-code/`, Windows `C:\Program Files\ClaudeCode\`의 `managed-settings.json`)로 배포합니다. 위치별 예시와 우선순위, `/status`로 확인하는 법은 [4회차 1-2](./session-04-permissions-skills-subagents.md#관리-설정은-어디에-어떻게-두나)를 봅니다.

> 💬 **강사 노트 (토론 3분):** "우리 회사에서 Claude Code를 도입한다면 **관리 설정으로 강제해야 할 것 3가지**는?" 조별로 한 줄씩. 흔한 답: 운영 비밀 deny, bypass 금지, 승인된 MCP만. "강제할 것"과 "팀 자율로 둘 것"을 나누는 것이 핵심입니다.

---

## ☕ 휴식 (8분)

> 각 팀은 스테이징 URL에 접속해 과정 목록과 수강 신청이 정상인지 확인합니다. **6장 훈련 전 정상 상태를 알아 두는 것**이 장애 판단의 기준이 됩니다.

---

# 4. 따라하기 ①: 성능 측정과 개선 (NFR-02)

> **목표**: "과정 목록 API는 데이터 1만 건에서 응답 300ms 이내, N+1 쿼리 없음"(NFR-02)을 **숫자로** 확인하고 개선합니다.
> **원칙**: 성능은 "느낌"이 아니라 **측정 → 원인 → 개선 → 재측정**입니다. Claude에게 "빠르게 해줘"라고 하지 않고, 측정 결과와 함께 원인 분석을 맡깁니다.

```mermaid
flowchart LR
    A["① 기준 측정<br/>쿼리 수·응답 시간"] --> B["② 원인 분석<br/>N+1, 인덱스, 페이징"]
    B --> C["③ 개선<br/>DTO 조회·배치 로딩·인덱스"]
    C --> D["④ 재측정<br/>같은 조건으로"]
    D --> E{"NFR-02 충족?"}
    E -- 아니오 --> B
    E -- 예 --> F["⑤ 쿼리 수 테스트로 고정<br/>→ PR"]
```

---

## Step 1. 브랜치와 측정 준비

### 1-1. 브랜치

```bash
git switch main && git pull
git switch -c perf/nfr-02-course-list
```

### 1-2. 쿼리 수를 볼 수 있게 설정

`backend/src/main/resources/application-dev.yml`에 측정용 설정이 없으면 **직접** 추가합니다 (세 줄이라 Claude에게 맡기는 것보다 빠릅니다).

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true   # 실행된 쿼리 수 집계 (측정할 때만, 운영 프로필에는 넣지 않음)
logging:
  level:
    org.hibernate.SQL: debug                       # 실행되는 SQL 출력
    org.hibernate.stat: debug                      # 요청별 통계
```

### 1-3. 1만 건 시드: SQL은 Claude Code에게 만들게 하고, 실행은 내가

과정 1만 건을 손으로 넣을 수는 없습니다. **"무엇을 몇 건, 어떤 규칙으로"는 사람이 정하고**, SQL 작성은 Claude Code에게 맡깁니다. 실행은 **접속한 DB가 내 로컬 DB인지 확인한 뒤 내가** 합니다.

**① Claude Code에게 지시** (EduHub 저장소에서, 기본 모드)

```text
[목표] 과정 목록 API(GET /api/v1/courses)의 성능 측정용 시드 SQL을 만들어줘.
[위치] scripts/seed-10k.sql (새 파일). 스키마는 backend/src/main/resources/db/migration/ 의 Flyway 파일을 읽고 맞춰줘.
[데이터]
- course 10,000건. 제목은 검색어로 찾을 수 있게 "주제 + 난이도 + 번호" (예: "JPA 성능 튜닝 입문 1"),
  주제 8개를 골고루, category는 주제에 맞게, 10건 중 1건은 CLOSED
- 과정마다 course_session 2~3개 (시작일·장소·정원은 규칙적으로 다르게)
- enrollment·employee는 넣지 않음
[규칙]
- PostgreSQL generate_series로 짧게 (INSERT 1만 줄 금지)
- 시드로 넣은 행에 표시를 남기고(description이 '[seed-10k]'로 시작), 다시 실행하면 그 표시가 있는 것만 지우고 새로 넣기
  (직접 만든 데이터는 건드리지 않기, 외래 키 순서대로 지우기)
- BEGIN/COMMIT으로 묶고, 마지막에 건수를 확인하는 SELECT
- 파일 맨 위에 "dev·로컬 전용, 운영·스테이징 실행 금지" 주석
[하지 않는 것] SQL 실행, Flyway 마이그레이션 파일 추가·수정, 다른 파일 수정
[완료 기준] 파일만 만들고, 지우는 조건과 넣는 건수를 요약해서 보여줘
```

> 💡 **왜 Flyway 마이그레이션(`V5__seed.sql`)으로 만들지 않나**: 마이그레이션은 **모든 환경**(스테이징·운영 포함)에서 자동으로 실행됩니다. 측정용 가짜 데이터가 운영 DB에 들어가면 안 되므로, 시드는 `scripts/`에 두고 **사람이 로컬에서만** 실행합니다.

**② Claude가 만든 결과 예** (이 교재 환경에서 위 지시로 만든 것과 같은 내용)

```sql
-- scripts/seed-10k.sql — 성능 측정용 시드 (dev·로컬 DB 전용, 운영·스테이징에서 실행 금지)
-- 과정 10,000건 + 과정마다 회차 2~3개(약 25,000건). 여러 번 실행해도 결과가 같다.
-- 시드로 넣은 과정은 description이 '[seed-10k]'로 시작한다 (이 표시로만 지우고 다시 넣는다)
-- 실행: psql -h localhost -U eduhub -d eduhub -v ON_ERROR_STOP=1 -f scripts/seed-10k.sql

BEGIN;

-- 0) 이전에 넣은 시드만 지운다 (직접 만든 과정·신청은 건드리지 않는다)
DELETE FROM enrollment
 WHERE session_id IN (SELECT s.id FROM course_session s
                        JOIN course c ON c.id = s.course_id
                       WHERE c.description LIKE '[seed-10k]%');
DELETE FROM course_session
 WHERE course_id IN (SELECT id FROM course WHERE description LIKE '[seed-10k]%');
DELETE FROM course WHERE description LIKE '[seed-10k]%';

-- 1) 과정 10,000건: 분야 8개 × 주제를 섞어 검색어(keyword)로 찾을 수 있는 제목을 만든다
INSERT INTO course (title, category, description, status)
SELECT format('%s %s %s', topic.name, level.name, n),
       topic.category,
       format('[seed-10k] %s 과정 %s번 (성능 측정용 데이터)', topic.name, n),
       CASE WHEN n % 10 = 0 THEN 'CLOSED' ELSE 'OPEN' END
  FROM generate_series(1, 10000) AS n
  JOIN LATERAL (
        SELECT (ARRAY['Spring 트랜잭션', 'JPA 성능 튜닝', 'React 기초', 'Kubernetes 운영',
                      '데이터 분석 SQL', '리더십 코칭', '보안 기본', '클라우드 아키텍처'])[1 + n % 8] AS name,
               (ARRAY['BACKEND', 'BACKEND', 'FRONTEND', 'INFRA',
                      'DATA', 'LEADERSHIP', 'SECURITY', 'INFRA'])[1 + n % 8] AS category
       ) AS topic ON true
  JOIN LATERAL (
        SELECT (ARRAY['입문', '실무', '심화'])[1 + (n / 8) % 3] AS name
       ) AS level ON true;

-- 2) 회차: 과정마다 2~3개 (id 기준으로 결정해 매번 같게)
INSERT INTO course_session (course_id, start_date, place, capacity, status)
SELECT c.id,
       DATE '2026-12-01' + ((c.id + k * 7) % 120)::int,
       (ARRAY['본사 3층 교육장', '연수원 A동', '온라인'])[1 + (c.id + k) % 3],
       10 + (c.id % 4) * 10,
       'OPEN'
  FROM course c
  CROSS JOIN generate_series(1, 3) AS k
 WHERE c.description LIKE '[seed-10k]%'
   AND (k <= 2 OR c.id % 2 = 0);

COMMIT;

-- 3) 확인 (결과를 눈으로 본다)
SELECT count(*) AS seed_courses FROM course WHERE description LIKE '[seed-10k]%';
SELECT count(*) AS seed_sessions
  FROM course_session s JOIN course c ON c.id = s.course_id
 WHERE c.description LIKE '[seed-10k]%';
SELECT category, count(*) FROM course WHERE description LIKE '[seed-10k]%' GROUP BY category ORDER BY category;
```

**③ 실행 전에 검토** (사람이 직접)

- [ ] 컬럼 이름·NOT NULL·CHECK가 Flyway 스키마(`V1__course.sql`)와 맞는가? (`capacity > 0` 등)
- [ ] `DELETE`의 조건이 **시드 표시(`[seed-10k]`)로만** 걸려 있는가? 조건 없는 `DELETE`·`TRUNCATE`가 없는가?
- [ ] 지우는 순서가 외래 키 순서인가? (`enrollment` → `course_session` → `course`)
- [ ] `BEGIN`/`COMMIT`으로 묶여 있어, 중간에 실패하면 아무것도 바뀌지 않는가?
- [ ] 파일을 Flyway 폴더(`db/migration/`)가 아니라 `scripts/`에 만들었는가?

**④ 실행** (먼저 서버를 한 번 띄워 Flyway가 테이블을 만든 상태여야 합니다)

```bash
docker compose up -d postgres

# 접속한 DB가 "내 로컬"인지 먼저 확인 (host가 localhost, database가 eduhub)
psql -h localhost -U eduhub -d eduhub -c "\conninfo"

psql -h localhost -U eduhub -d eduhub -v ON_ERROR_STOP=1 -f scripts/seed-10k.sql
```

```powershell
# Windows에 psql이 없으면: 파일을 컨테이너에 복사해서 컨테이너 안의 psql로 실행
docker compose cp scripts/seed-10k.sql postgres:/tmp/seed-10k.sql
docker compose exec postgres psql -U eduhub -d eduhub -v ON_ERROR_STOP=1 -f /tmp/seed-10k.sql
```

**기대 결과** (이 교재 환경에서 실제로 실행한 결과, 0.3초)

```text
DELETE 0
DELETE 0
DELETE 0
INSERT 0 10000
INSERT 0 25000
COMMIT
 seed_courses
--------------
        10000

 seed_sessions
---------------
         25000

  category  | count
------------+-------
 BACKEND    |  2500
 DATA       |  1250
 FRONTEND   |  1250
 INFRA      |  2500
 LEADERSHIP |  1250
 SECURITY   |  1250
```

- 한 번 더 실행하면 `DELETE`가 시드 건수만큼 지우고 다시 넣어, **건수가 그대로** 10,000 / 25,000입니다 (두 번 실행해 확인했습니다)
- `-v ON_ERROR_STOP=1`이 없으면 오류가 나도 다음 줄을 계속 실행합니다. 꼭 붙입니다

> ⚠️ **운영·스테이징 DB에서 실행하지 않습니다.** `.env`나 환경 변수의 DB 주소가 스테이징으로 되어 있으면 그대로 그쪽에 들어갑니다. 실행 전 `\conninfo`로 확인하는 습관을 들입니다. 4회차 권한 설정에서 `Bash(psql *)`는 **deny**라 Claude는 psql을 실행할 수 없습니다. 그래서 SQL은 Claude가 쓰고, 실행은 사람이 합니다.

**⑤ (참고) MariaDB를 쓰는 프로젝트라면**

EduHub는 PostgreSQL이지만, 회사 프로젝트가 MariaDB(MySQL 계열)라면 같은 시드를 **MariaDB 문법으로** 만들어야 합니다. 지시문의 `[규칙]`만 바꿉니다.

```text
[규칙]
- MariaDB 10.11용. 숫자 만들기는 SEQUENCE 엔진 테이블(seq_1_to_10000)로 (INSERT 1만 줄 금지)
- 시드 표시('[seed-10k]')로만 지우고 다시 넣기, 외래 키 순서대로 (DELETE … JOIN)
- START TRANSACTION/COMMIT으로 묶고, 마지막에 건수 확인 SELECT
- 파일은 scripts/seed-10k-mariadb.sql, 맨 위에 "dev·로컬 전용, 운영·스테이징 실행 금지" 주석
```

| PostgreSQL 판 | MariaDB 판 | 이유 |
|---|---|---|
| `generate_series(1, 10000)` | `seq_1_to_10000` 테이블 | MariaDB에는 `generate_series`가 없다. SEQUENCE 엔진이 `seq_<시작>_to_<끝>` 이름의 가상 테이블을 만들어 준다 (기본으로 켜져 있음) |
| `(ARRAY['a','b'])[i]` | `ELT(i, 'a', 'b')` | 배열 문법이 없다. `ELT`는 i번째 값 |
| `format('%s %s', a, b)` | `CONCAT(a, ' ', b)` | |
| `n / 8` (정수 나눗셈) | `seq DIV 8` | MariaDB의 `/`는 소수가 나온다 |
| `DATE '2026-12-01' + n` | `DATE_ADD('2026-12-01', INTERVAL n DAY)` | |
| `CASE WHEN … THEN … ELSE … END` | `IF(조건, 참, 거짓)` (CASE도 됨) | |
| `DELETE … WHERE id IN (SELECT …)` | `DELETE s FROM course_session s JOIN course c …` | 같은 뜻. MariaDB에서는 JOIN 형태가 흔하고 빠르다 |
| `BEGIN` | `START TRANSACTION` | 테이블이 InnoDB여야 트랜잭션이 된다 |

**결과 예** (이 교재 환경에서 MariaDB 10.11로 실행한 파일)

```sql
-- scripts/seed-10k-mariadb.sql — 성능 측정용 시드, MariaDB 10.x·11.x용 (dev·로컬 DB 전용, 운영·스테이징에서 실행 금지)
-- 과정 10,000건 + 과정마다 회차 2~3개(약 25,000건). 여러 번 실행해도 결과가 같다.
-- 시드로 넣은 과정은 description이 '[seed-10k]'로 시작한다 (이 표시로만 지우고 다시 넣는다)
-- 숫자 1~10000은 MariaDB의 SEQUENCE 엔진 테이블(seq_1_to_10000)로 만든다 (기본으로 켜져 있음)
-- 실행: mariadb -h 127.0.0.1 -u eduhub -p --default-character-set=utf8mb4 eduhub < scripts/seed-10k-mariadb.sql

START TRANSACTION;

-- 0) 이전에 넣은 시드만 지운다 (직접 만든 과정·신청은 건드리지 않는다)
DELETE e FROM enrollment e
  JOIN course_session s ON s.id = e.session_id
  JOIN course c ON c.id = s.course_id
 WHERE c.description LIKE '[seed-10k]%';
DELETE s FROM course_session s
  JOIN course c ON c.id = s.course_id
 WHERE c.description LIKE '[seed-10k]%';
DELETE FROM course WHERE description LIKE '[seed-10k]%';

-- 1) 과정 10,000건: 분야 8개 × 주제를 섞어 검색어(keyword)로 찾을 수 있는 제목을 만든다
--    ELT(i, a, b, …)는 i번째 값 (PostgreSQL의 ARRAY[…][i]에 해당)
INSERT INTO course (title, category, description, status)
SELECT CONCAT(ELT(1 + seq % 8, 'Spring 트랜잭션', 'JPA 성능 튜닝', 'React 기초', 'Kubernetes 운영',
                               '데이터 분석 SQL', '리더십 코칭', '보안 기본', '클라우드 아키텍처'),
              ' ', ELT(1 + (seq DIV 8) % 3, '입문', '실무', '심화'),
              ' ', seq),
       ELT(1 + seq % 8, 'BACKEND', 'BACKEND', 'FRONTEND', 'INFRA',
                        'DATA', 'LEADERSHIP', 'SECURITY', 'INFRA'),
       CONCAT('[seed-10k] ',
              ELT(1 + seq % 8, 'Spring 트랜잭션', 'JPA 성능 튜닝', 'React 기초', 'Kubernetes 운영',
                               '데이터 분석 SQL', '리더십 코칭', '보안 기본', '클라우드 아키텍처'),
              ' 과정 ', seq, '번 (성능 측정용 데이터)'),
       IF(seq % 10 = 0, 'CLOSED', 'OPEN')
  FROM seq_1_to_10000;

-- 2) 회차: 과정마다 2~3개 (id 기준으로 결정해 매번 같게)
INSERT INTO course_session (course_id, start_date, place, capacity, status)
SELECT c.id,
       DATE_ADD('2026-12-01', INTERVAL (c.id + k.seq * 7) % 120 DAY),
       ELT(1 + (c.id + k.seq) % 3, '본사 3층 교육장', '연수원 A동', '온라인'),
       10 + (c.id % 4) * 10,
       'OPEN'
  FROM course c
  CROSS JOIN seq_1_to_3 AS k
 WHERE c.description LIKE '[seed-10k]%'
   AND (k.seq <= 2 OR c.id % 2 = 0);

COMMIT;

-- 3) 확인 (결과를 눈으로 본다)
SELECT COUNT(*) AS seed_courses FROM course WHERE description LIKE '[seed-10k]%';
SELECT COUNT(*) AS seed_sessions
  FROM course_session s JOIN course c ON c.id = s.course_id
 WHERE c.description LIKE '[seed-10k]%';
SELECT category, COUNT(*) AS cnt FROM course WHERE description LIKE '[seed-10k]%' GROUP BY category ORDER BY category;
```

**실행**

```bash
# bash (Git Bash·macOS·Linux): 파일을 입력으로
mariadb -h 127.0.0.1 -u eduhub -p --default-character-set=utf8mb4 eduhub -e "status" | grep -E "Current database|Connection"   # 로컬인지 먼저 확인
mariadb -h 127.0.0.1 -u eduhub -p --default-character-set=utf8mb4 eduhub < scripts/seed-10k-mariadb.sql
```

```powershell
# PowerShell: "<"(입력 리디렉션)을 쓸 수 없으므로 클라이언트의 source 명령으로
mariadb -h 127.0.0.1 -u eduhub -p --default-character-set=utf8mb4 eduhub -e "source scripts/seed-10k-mariadb.sql"

# 클라이언트가 없고 MariaDB가 Docker 컨테이너(서비스 이름 mariadb)라면
docker compose cp scripts/seed-10k-mariadb.sql mariadb:/tmp/seed.sql
docker compose exec mariadb mariadb -u eduhub -p --default-character-set=utf8mb4 eduhub -e "source /tmp/seed.sql"
```

- `-p`만 쓰면 비밀번호를 물어봅니다. `-peduhub`처럼 붙여 쓰면 명령 기록에 비밀번호가 남으니 로컬 실습에서만
- 예전 버전이나 MySQL 클라이언트는 명령 이름이 `mysql`입니다 (옵션은 같음)

**기대 결과** (`-t` 옵션을 붙이면 표 모양으로 나온다)

```text
+--------------+
| seed_courses |
+--------------+
|        10000 |
+--------------+
+---------------+
| seed_sessions |
+---------------+
|         25000 |
+---------------+
+------------+------+
| category   | cnt  |
+------------+------+
| BACKEND    | 2500 |
| DATA       | 1250 |
| FRONTEND   | 1250 |
| INFRA      | 2500 |
| LEADERSHIP | 1250 |
| SECURITY   | 1250 |
+------------+------+
```

> ⚠️ **`--default-character-set=utf8mb4`를 꼭 붙입니다.** 이 교재 환경의 MariaDB 클라이언트는 옵션이 없으면 `latin1`로 접속했고, 그 상태로 넣은 한글 제목 "직접 만든 과정"이 `ì§ì ‘ ë§Œë“  ê³¼ì •`처럼 깨져 저장되었습니다. 오류 없이 **조용히 깨지므로** 넣은 뒤 `SELECT title FROM course LIMIT 3`으로 한글을 눈으로 확인합니다. 테이블도 `DEFAULT CHARSET=utf8mb4`로 만들어져 있어야 합니다.

> ✅ 이 교재 환경(MariaDB 10.11.14, InnoDB, EduHub의 V1~V3 테이블을 MariaDB 문법으로 옮긴 스키마)에서 확인했습니다: 0.3초에 10,000 / 25,000건, 두 번 실행해도 건수 같음, 직접 넣은 과정 1건과 그 신청은 남고 시드 회차의 신청만 지워짐, `< 파일`과 `-e "source 파일"` 두 방식 모두 동작. ⚠️ Windows PowerShell과 Docker 컨테이너에서의 실행, MariaDB 11.x는 해 보지 못했습니다. EduHub 백엔드를 MariaDB로 바꾸는 방법(드라이버, Flyway 스크립트)은 이 과정의 범위 밖입니다.

### 1-4. 서버 실행 (시드를 넣은 뒤 그대로 띄워 둡니다)

```bash
cd backend && ./gradlew bootRun --args='--spring.profiles.active=dev'
```

---

## Step 2. 기준 측정 (Before)

같은 API를 두 가지로 잽니다: **① 요청 한 번에 쿼리가 몇 개 나가나**(N+1 확인), **② 여러 사람이 동시에 부를 때 얼마나 빨리 응답하나**(k6).

### 2-1. 쿼리 수: 요청 한 번, 로그로 세기

다른 터미널에서 한 번 호출합니다.

```bash
curl -s "http://localhost:8080/api/v1/courses?page=0&size=20" > /dev/null
```

서버 로그에서 이번 요청에 찍힌 `org.hibernate.SQL` 줄을 봅니다. 이 교재 환경(시드 1만 건)에서는 이렇게 나왔습니다.

```text
select c1_0.id,c1_0.category,… from course c1_0 where (? is null or lower(c1_0.title) like …) order by c1_0.id offset ? rows fetch first ? rows only   ← 목록 1번
select count(c1_0.id) from course c1_0 where (? is null or lower(c1_0.title) like …)                                                           ← 전체 건수 1번
select count(cs1_0.id) from course_session cs1_0 left join course c1_0 on … where c1_0.id=?                                                    ← 과정마다 1번 × 20
…
Session Metrics {
    … nanoseconds spent executing 22 JDBC statements;
```

**22개 = 목록 1 + 전체 건수 1 + 회차 수 20.** 목록에 보이는 과정 20개마다 "회차가 몇 개인지" 쿼리가 따로 나갑니다. 이것이 **N+1**입니다 (2회차 2-2). 한 페이지를 100건으로 늘리면 102개가 됩니다.

> 💡 줄 수를 세기 어려우면 `Session Metrics`의 `executing NN JDBC statements`를 봅니다. 요청마다 한 번씩 찍힙니다.

### 2-2. k6란: "여러 사람이 동시에 쓰는 상황"을 만들어 주는 도구

**k6**(케이식스)는 Grafana Labs가 만든 무료(오픈소스) **부하 테스트 도구**입니다. 브라우저 대신 **가상 사용자**를 여러 명 만들어 API를 계속 호출하고, 응답 시간을 모아 통계를 냅니다.

- 시나리오는 **JavaScript 파일**로 씁니다 (어떤 주소를, 몇 명이, 얼마 동안). 하지만 Node.js에서 도는 것이 아니라, k6 **실행 파일 하나**가 스크립트를 읽어 실행합니다. Node.js·npm 설치가 필요 없습니다
- `curl`로 한 번 재는 것은 "혼자 쓸 때"의 시간입니다. 실제 서비스는 **여러 명이 동시에** 쓰고, 그때 DB 연결이 모자라거나 쿼리가 몰려 느려집니다. k6는 그 상황을 만듭니다
- 비슷한 도구: JMeter(화면에서 설정, Java), Gatling(Scala·Java 코드), nGrinder(네이버, 웹 화면). k6는 **스크립트가 짧고 CI에 넣기 쉬워서** 이 과정에서 씁니다

```mermaid
flowchart LR
    S["perf/course-list.js<br/>(시나리오)"] --> K6["k6 run"]
    K6 --> V1["VU 1"] & V2["VU 2"] & V3["… VU 20"]
    V1 & V2 & V3 -- "GET /api/v1/courses (반복)" --> API["Spring Boot :8080"]
    API --> DB[("PostgreSQL<br/>과정 1만 건")]
    K6 --> R["결과 요약<br/>p95 응답 시간 · 실패율 · 기준 통과 여부"]
```

**꼭 알아야 할 용어**

| 용어 | 뜻 | 예 |
|---|---|---|
| **VU** (Virtual User) | 가상 사용자. 각자 스크립트의 `default` 함수를 **끝나면 다시** 반복한다 | `vus: 20` = 20명이 동시에 |
| **iteration** | VU가 `default` 함수를 한 번 끝까지 실행한 것 | 요청 1번 + 0.5초 쉬기 = 1 iteration |
| **think time** | 사용자가 화면을 보는 시간. `sleep(0.5)` | 없으면 쉬지 않고 두드리는 "최대 부하" |
| **http_req_duration** | 요청을 보내고 응답을 다 받을 때까지 걸린 시간 | NFR-02의 "응답 시간" |
| **p95** (95번째 백분위수) | 응답 시간을 빠른 순으로 줄 세웠을 때 **95% 지점**의 값. "100번 중 95번은 이 시간 안에" | `p(95)=127ms` |
| **threshold** | 통과 기준. 넘으면 결과에 ✗, k6의 **종료 코드가 99**(실패) | `'p(95)<300'` |
| **check** | 응답 하나하나가 맞는지 확인 (상태 코드, 건수). 기준(threshold)과 달리 실패해도 종료 코드는 그대로 | `'status 200'` |

> 💡 **왜 평균(avg)이 아니라 p95인가**: 평균은 아주 빠른 요청들이 아주 느린 요청을 가립니다. 평균 58ms여도 20명 중 1명은 0.5초를 기다릴 수 있습니다. "대부분의 사용자가 겪는 최악"을 보려고 p95(또는 p99)로 기준을 정합니다.

### 2-3. k6 설치

| PC | 설치 명령 | 비고 |
|---|---|---|
| ⭐ Windows (교육 기본) | `winget install k6 --source winget` | Windows 10·11 기본 포함. 설치 후 **터미널을 새로 연다** |
| Windows (Chocolatey) | `choco install k6` | 관리자 PowerShell |
| macOS | `brew install k6` | |
| Linux (Debian·Ubuntu) | k6 공식 문서 Install의 apt 저장소 등록 후 `sudo apt-get install k6` | |
| 설치 도구를 쓸 수 없을 때 | GitHub `grafana/k6` 저장소 Releases에서 내 OS용 압축 파일(`k6-v…-windows-amd64.zip`)을 받아 풀고, `k6.exe`가 있는 폴더를 PATH에 추가 | 받은 파일은 회사 보안 정책에 따라 IT 확인 |
| Docker만 있을 때 | `docker run --rm -i grafana/k6 run - < perf/course-list.js` | 컨테이너 안에서는 `localhost`가 내 PC가 아니다 → `-e BASE_URL=http://host.docker.internal:8080` |

설치 확인:

```bash
k6 version
# k6 v1.x.x (go1.xx, windows/amd64)
```

> ⚠️ 6회차의 nginx처럼, 회사 PC의 보안 프로그램이 `k6.exe`를 막을 수 있습니다. 끄지 말고, 실행 파일 경로와 출처(Grafana 공식 배포)를 근거로 **IT에 예외 등록을 요청**합니다. 허용 전까지는 강사 PC의 결과를 함께 봅니다.

### 2-4. 시나리오 작성: `perf/course-list.js`

```javascript
// perf/course-list.js — NFR-02: 과정 목록 API 응답 시간 측정
import http from 'k6/http';
import { check, sleep } from 'k6';

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export const options = {
  vus: 20,          // 가상 사용자(VU) 20명이 동시에
  duration: '30s',  // 30초 동안 반복
  thresholds: {
    http_req_duration: ['p(95)<300'],  // NFR-02: 95%의 요청이 300ms 안에
    http_req_failed: ['rate<0.01'],    // 실패율 1% 미만
  },
};

// VU 한 명이 반복해서 실행하는 내용 (끝나면 처음부터 다시)
export default function () {
  const page = Math.floor(Math.random() * 50);   // 같은 페이지만 부르면 캐시 효과로 실제보다 빨라 보인다
  const res = http.get(`${BASE_URL}/api/v1/courses?page=${page}&size=20`);
  check(res, {
    'status 200': (r) => r.status === 200,
    '20건': (r) => r.json('content').length === 20,
  });
  sleep(0.5);   // 사용자가 화면을 보는 시간 (0.5초 쉬고 다음 요청)
}
```

| 부분 | 하는 일 |
|---|---|
| `import http from 'k6/http'` | k6에 들어 있는 HTTP 모듈 (npm 패키지가 아니다. `axios`·`fetch`는 쓸 수 없다) |
| `__ENV.BASE_URL` | 실행할 때 `-e BASE_URL=…`로 주소를 바꿀 수 있게 (스테이징 주소로 잘못 돌리지 않게 기본값은 localhost) |
| `options.vus`, `duration` | 몇 명이, 얼마 동안 |
| `options.thresholds` | **NFR-02를 그대로 코드로**. 통과·실패를 사람이 판단하지 않고 k6가 판정한다 |
| `export default function` | VU 한 명이 반복하는 일: 페이지 하나를 골라 목록 조회 |
| `check(...)` | 응답이 200이고 20건인지. 에러 페이지를 빠르게 돌려받고 "빠르다"고 착각하지 않게 |
| `sleep(0.5)` | 사람처럼 0.5초 쉬기. 20명 × 초당 약 2번 = **초당 약 35~40건** 정도의 부하 |

**사람이 몰리는 상황도 하나 더**: 수강 신청을 연 직후처럼 **50명이 쉬지 않고 검색**하는 시나리오입니다.

```javascript
// perf/course-list-peak.js — 사람이 몰릴 때: 수강 신청 오픈 직후처럼 쉬지 않고 검색
import http from 'k6/http';
import { check } from 'k6';

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export const options = {
  vus: 50,          // 동시에 50명
  duration: '20s',
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const page = Math.floor(Math.random() * 50);
  // 검색어까지 넣는다: 제목 LIKE 검색 + 전체 건수 count 쿼리가 함께 돈다
  const res = http.get(`${BASE_URL}/api/v1/courses?keyword=jpa&page=${page}&size=20`);
  check(res, {
    'status 200': (r) => r.status === 200,
    '20건': (r) => r.json('content').length === 20,
  });
  // sleep 없음: 응답을 받자마자 다음 요청
}
```

> 💡 시나리오 초안은 Claude에게 맡겨도 됩니다 ("이 API로 20 VU 30초, p95<300ms 기준의 k6 스크립트"). 단 **VU 수·시간·기준값은 사람이 정한** NFR에서 옵니다. k6 문법은 버전에 따라 바뀌므로 Claude가 쓴 스크립트는 한 번 실행해 보고 씁니다.

### 2-5. 실행하고 결과 읽기

```bash
# 서버(Step 1-4)가 떠 있는 상태에서, 저장소 루트에서
k6 run perf/course-list.js
k6 run perf/course-list-peak.js

# 결과를 파일로도 남기기 (PR에 붙일 근거)
k6 run perf/course-list.js 2>&1 | tee perf/before-normal.txt          # PowerShell: | Tee-Object perf\before-normal.txt
```

실행하면 30초 동안 진행 막대가 움직이고, 끝나면 요약이 나옵니다. 이 교재 환경(Linux, k6 v1.4.2, 백엔드·PostgreSQL 16이 같은 PC)의 **실제 결과**입니다.

```text
  █ THRESHOLDS

    http_req_duration
    ✓ 'p(95)<300' p(95)=127.42ms

    http_req_failed
    ✓ 'rate<0.01' rate=0.00%

  █ TOTAL RESULTS

    checks_succeeded...: 100.00% 2164 out of 2164
    ✓ status 200
    ✓ 20건

    HTTP
    http_req_duration..............: avg=58.26ms  min=29.42ms  med=46.49ms  max=499.55ms p(90)=80.09ms p(95)=127.42ms
    http_req_failed................: 0.00%  0 out of 1082
    http_reqs......................: 1082   35.462316/s
```

| 줄 | 읽는 법 |
|---|---|
| `THRESHOLDS`의 ✓·✗ | **NFR-02 통과 여부.** 하나라도 ✗이면 마지막에 `thresholds on metrics 'http_req_duration' have been crossed`가 나오고 종료 코드 99 |
| `p(95)=127.42ms` | 100번 중 95번은 127ms 안에 응답 |
| `avg`·`med`·`max` | 평균·가운데 값·가장 느린 것. `max`가 튀는 것은 한두 번일 수 있으니 p95와 함께 본다 |
| `http_req_failed 0.00%` | 오류 응답(4xx·5xx·연결 실패) 비율 |
| `http_reqs 35.46/s` | 초당 처리한 요청 수 (처리량) |
| `checks 100%` | 응답 내용 확인(200, 20건)이 모두 맞았다 |

**사람이 몰릴 때**(`course-list-peak.js`) 결과는 달랐습니다.

```text
    http_req_duration
    ✗ 'p(95)<300' p(95)=670.26ms
    http_req_duration..............: avg=520.38ms min=51.61ms med=517.55ms max=1.25s p(90)=615.96ms p(95)=670.26ms
    http_reqs......................: 1943   95.074501/s
time="…" level=error msg="thresholds on metrics 'http_req_duration' have been crossed"
```

**측정 기록표** (PR 본문에 그대로 붙입니다. 아래는 이 교재 환경의 Before 예)

| 항목 | Before (교재 환경 예) | After |
|---|---|---|
| 요청 1회당 select 수 (`size=20`) | 22 (목록 1 + 건수 1 + 회차 20) | |
| 평소 부하 p95 (20 VU, 0.5초 쉬기, 30초) | 127ms ✓ | |
| 몰릴 때 p95 (50 VU, 쉬지 않음, 검색어, 20초) | 670ms ~ 1.02s ✗ (세 번 측정) | |
| 실패율 | 0% | |
| k6 threshold 통과 | 평소 ✓ / 몰릴 때 ✗ | |

> 💡 **"평소에는 통과하는데 왜 고치나?"** NFR-02는 "300ms 이내" **그리고 "N+1 없음"**입니다. 내 PC처럼 DB가 바로 옆에 있으면 작은 쿼리 20개도 빨라서 평소 부하는 통과합니다. 하지만 ① 요청 하나가 쿼리 22개를 쓰는 동안 DB 연결을 오래 붙잡아, 사람이 몰리면 연결을 기다리는 줄이 생기고(위 결과), ② 운영처럼 DB가 다른 서버에 있으면 쿼리마다 네트워크 왕복이 더해집니다. **쿼리 수는 PC와 상관없이 같으므로** 가장 믿을 만한 지표입니다.

### 2-6. 측정할 때 지킬 것

| 규칙 | 이유 |
|---|---|
| **같은 조건으로 Before·After** (같은 PC, 같은 시드, 같은 스크립트) | 조건이 다르면 개선 효과인지 알 수 없다 |
| 서버를 띄운 뒤 **한 번 돌려 버리고**(워밍업) 두 번째부터 기록 | 처음 몇 초는 JVM이 코드를 최적화하는 중이라 느리다 |
| **2~3번 재서** 범위로 적는다 | 같은 조건에서도 결과가 흔들린다 (교재 환경의 몰릴 때 p95: 670ms, 914ms, 1.02s) |
| 측정 중에는 IDE 빌드·브라우저 탭 등 무거운 작업을 하지 않는다 | 내 PC가 서버·DB·k6를 모두 돌리고 있다 |
| 내 PC의 숫자는 **비교용**이다 | 운영 성능은 운영과 같은 구성의 스테이징에서, 정해진 시간에 잰다 |
| ⚠️ **스테이징·운영에 승인 없이 부하를 주지 않는다** | 다른 사람의 테스트·실제 사용자에게 장애가 된다. `BASE_URL` 기본값을 localhost로 둔 이유 |

> ✅ 이 교재 환경(Linux, k6 v1.4.2, Spring Boot 3.5 백엔드, PostgreSQL 16, 위 시드)에서 이 Step의 명령을 차례로 실행해 위 결과를 얻었습니다: 시드 SQL 두 번 실행해도 건수 같음, 요청 1회당 SQL 22개, 평소 부하 p95 127ms·143ms(✓), 몰릴 때 p95 670ms~1.02s(✗, 종료 코드 99). SQL 로그를 끄고 재도 평소 부하 p95는 143ms로 비슷했습니다. ⚠️ k6는 Linux에서 공식 소스로 빌드해 썼고, Windows의 `winget`·`choco` 설치와 PowerShell의 `docker compose cp` 실행은 해 보지 못했습니다 (k6 공식 문서 Install의 방식). 시드 SQL은 Claude Code가 아니라 교재 작성 중에 만든 것이며, 수업에서 Claude가 만든 결과는 모양이 다를 수 있으니 ③의 검토를 꼭 합니다.

---

## Step 3. 원인 분석은 Claude에게, 판단은 내가

권한 모드를 **Plan**으로 두고 측정 결과와 함께 지시합니다.

```text
[목표] 과정 목록 API(GET /api/v1/courses)의 N+1 쿼리와 응답 시간을 개선하려고 해. 원인 분석과 개선 방안만 먼저 알려줘.

[위치]
- @backend/src/main/java/com/example/eduhub/course/CourseController.java
- @backend/src/main/java/com/example/eduhub/course/CourseService.java
- @backend/src/main/java/com/example/eduhub/course/CourseRepository.java

[측정 결과]
- 데이터 1만 건, page size 20 요청 1회에 SQL 22개 (목록 1 + 전체 건수 1 + 과정마다 회차 수 20)
- k6 평소 부하(20 VU, 0.5초 쉬기): p95 ○○○ms / 몰릴 때(50 VU, 쉬지 않음, 검색어): p95 ○○○ms (기준 300ms 초과)
- 아래는 요청 1회의 SQL 로그 일부:
(로그 20줄 붙여넣기)

[범위 / 하지 않는 것]
- API 응답 형식(JSON 필드)은 바꾸지 않는다
- 캐시 도입은 하지 않는다 (이번 범위 아님)

[완료 기준]
- 요청 1회당 SQL 3개 이하, k6 두 시나리오 모두 p95 < 300ms
- ./gradlew test 통과

[진행 방식] 개선 방안을 2~3가지 비교표(장점·단점·페이징 영향)로 보여줘. 구현은 내가 고른 뒤에.
```

**Claude가 흔히 제시하는 방안과 판단 기준**

| 방안 | 장점 | 주의 (개발자가 알아야 할 것) |
|---|---|---|
| `JOIN FETCH` / `@EntityGraph`로 회차를 함께 조회 | 코드 변경이 작다 | **컬렉션 fetch join + 페이징**을 같이 쓰면 Hibernate가 전체를 메모리로 가져와 페이징한다(경고 로그 `HHH90003004`). 1만 건에서 오히려 느려질 수 있다 |
| `hibernate.default_batch_fetch_size` (또는 `@BatchSize`) | 페이징 유지, N+1 → 1+1 | 전역 설정이라 다른 쿼리에도 영향 |
| **DTO 조회 쿼리** (목록에 필요한 필드와 남은 자리 수만 `select new ...` 또는 집계 쿼리) | 필요한 데이터만, 페이징 정상 | 쿼리 코드가 늘어난다. 응답 형식과 맞는지 확인 |
| 인덱스 추가 (`course(category)`, `course_session(course_id)`, `enrollment(session_id, status)`) | 검색·집계가 빨라진다 | Flyway 마이그레이션으로만 추가. 쓰기 성능 영향 확인 |

> 💡 위 표의 "주의" 칸이 **2회차 2-2의 아키텍처 지식**입니다. 테스트는 세 방안 모두 통과하지만, fetch join + 페이징 문제는 지식이 있어야 보입니다.

---

## Step 4. 개선 구현

방안을 고른 뒤(예: DTO 조회 + 인덱스) 권한 모드를 **Accept edits**로 바꿔 지시합니다.

```text
DTO 조회 + 인덱스 방안으로 구현해줘.
- CourseRepository에 목록용 조회 쿼리 추가 (과정 정보 + 회차 수 + 남은 자리 합계), 페이징 유지
- Flyway 마이그레이션 V{다음 번호}__add_course_list_indexes.sql 추가
- 요청 1회의 select 수를 검증하는 테스트를 추가해줘 (Hibernate Statistics 사용)
완료 기준: ./gradlew test 통과
```

쿼리 수를 테스트로 고정하면 **나중에 누군가(AI 포함) N+1을 다시 만들어도 CI가 잡습니다.**

```java
@SpringBootTest
@Transactional
class CourseListQueryCountTest {

    @Autowired EntityManagerFactory emf;
    @Autowired CourseService courseService;

    @Test
    void 과정_목록_조회는_쿼리_3개_이하() {
        Statistics stats = emf.unwrap(SessionFactory.class).getStatistics();
        stats.setStatisticsEnabled(true);
        stats.clear();

        courseService.findCourses(PageRequest.of(0, 20), null);

        assertThat(stats.getPrepareStatementCount()).isLessThanOrEqualTo(3);
    }
}
```

```sql
-- V12__add_course_list_indexes.sql (번호는 팀 저장소에 맞춤)
CREATE INDEX IF NOT EXISTS idx_course_category ON course (category);
CREATE INDEX IF NOT EXISTS idx_course_session_course_id ON course_session (course_id);
CREATE INDEX IF NOT EXISTS idx_enrollment_session_status ON enrollment (session_id, status);
```

**diff 검토 체크리스트**

- [ ] API 응답 JSON 형식이 바뀌지 않았는가? (프론트 `api-client` 재생성이 필요 없어야 함)
- [ ] 페이징이 DB에서 일어나는가? (로그에 `limit`/`offset`, `HHH90003004` 경고 없음)
- [ ] 마이그레이션이 **새 파일**로 추가되었는가? (기존 Flyway 파일 수정 금지)
- [ ] `generate_statistics` 같은 측정 설정이 **운영 프로필에 들어가지 않았는가?**

---

## Step 5. 재측정과 PR

```bash
k6 run perf/course-list.js          # 같은 조건으로 재측정 (워밍업 1번 후, 2~3번), 기록표 After 칸 채우기
k6 run perf/course-list-peak.js
cd backend && ./gradlew test        # 내 터미널에서 직접
git add backend/src perf/ scripts/seed-10k.sql
git diff --staged                   # 측정용 로그 설정이 섞이지 않았는지 확인
git commit -m "perf(course): 과정 목록 N+1 제거와 인덱스 추가 (NFR-02)"
git push -u origin perf/nfr-02-course-list
```

PR 본문에 **측정 기록표(Before/After)**와 "Claude에게 맡긴 것 / 직접 한 것"을 적습니다.

> ✅ **확인**: k6 threshold가 통과(`✓ http_req_duration`)하고, 쿼리 수 테스트가 CI에서 통과하면 NFR-02 완료입니다.

---

# 5. 따라하기 ②: 보안 점검과 PR 자동 보안 리뷰 (NFR-03)

> **NFR-03**: 로그인한 사용자만 신청 가능, **본인 신청만 취소 가능**, 비밀 정보는 저장소에 두지 않음

## Step 1. 보안 리뷰 실행

```bash
git switch main && git pull
git switch -c fix/nfr-03-security
```

새 세션(`/clear`)에서 다음 순서로 진행합니다.

1. 내장 명령 `/security-review`로 현재 브랜치 변경의 보안 점검을 돌려 봅니다 (변경이 없으면 결과가 적을 수 있습니다)
2. 저장소 전체 관점의 점검은 읽기 전용으로 지시합니다. 보안 점검 전용 `security-reviewer` 에이전트를 예시를 참고해 만들어 두면 반복해서 쓸 수 있습니다 ([security-reviewer 예시](../examples/18-java-react-agents.md#3-5-security-reviewer-리뷰-읽기-전용-고성능-모델))

```text
[목표] EduHub의 NFR-03(인증·인가, 입력 검증, 비밀 정보) 점검. 수정하지 말고 발견 사항만 보고해줘.

[위치]
- @backend/src/main/java/com/example/eduhub/enrollment/
- @backend/src/main/java/com/example/eduhub/common/

[점검 항목]
1. 수강 신청 취소 API가 "본인 신청만" 취소하게 막는가? (다른 사람의 enrollmentId로 호출하면?)
2. 요청 DTO에 입력 검증(@Valid, @NotNull, @Size 등)이 있는가?
3. 신청자 ID를 요청 본문에서 받고 있지 않은가? (인증 정보에서 가져와야 함)
4. 에러 응답에 스택트레이스·SQL 같은 내부 정보가 노출되지 않는가?

[완료 기준] 항목별로 파일:줄, 위험도(높음/중간/낮음), 재현 방법, 수정 방향을 표로
```

> ⚠️ `.env`, `application-prod.yml`은 권한 규칙으로 읽기가 막혀 있어야 합니다. 점검 중 Claude가 이 파일을 읽으려다 거부되면 **정상**입니다.

## Step 2. 가장 위험한 것부터 고치기: 본인 신청만 취소

흔히 발견되는 코드와 수정 방향입니다.

```java
// Before: 누구나 아무 신청이나 취소할 수 있음
@Transactional
public void cancel(Long enrollmentId) {
    Enrollment e = enrollmentRepository.findById(enrollmentId)
        .orElseThrow(() -> new EnrollmentNotFoundException(enrollmentId));
    e.cancel();
    waitingPromoter.promoteNext(e.getSession());
}

// After: 인증된 사용자 본인의 신청인지 확인
@Transactional
public void cancel(Long enrollmentId, Long currentEmployeeId) {
    Enrollment e = enrollmentRepository.findById(enrollmentId)
        .orElseThrow(() -> new EnrollmentNotFoundException(enrollmentId));
    if (!e.isOwnedBy(currentEmployeeId)) {
        // 존재 여부를 숨기려면 404, 명시하려면 403 — 팀 규칙으로 정한다
        throw new EnrollmentNotFoundException(enrollmentId);
    }
    e.cancel();
    waitingPromoter.promoteNext(e.getSession());
}
```

```text
cancel()에 본인 확인을 추가해줘. currentEmployeeId는 컨트롤러에서 인증 정보로 받아 넘겨.
테스트: (1) 본인 취소 성공 (2) 다른 사람 신청 취소 시 404 (3) 취소 후 대기 1순위 승급 유지.
완료 기준: ./gradlew test 통과. 응답 상태 코드는 404로 통일.
```

## Step 3. 비밀 정보 스캔

### 3-1. 비밀 정보 스캔이란

**비밀 정보(secret)**는 그것만 있으면 누군가 **내 대신 시스템에 들어갈 수 있는 값**입니다: API 키, DB 비밀번호, 클라우드 접근 키(AWS·Azure), JWT 서명 키, GitHub 토큰, 개인 키(`-----BEGIN PRIVATE KEY-----`), 사내 시스템 계정. **비밀 정보 스캔**은 저장소의 파일과 **커밋 이력 전체**를 뒤져 이런 값이 들어 있는지 찾아 주는 검사입니다.

| 왜 필요한가 | 설명 |
|---|---|
| **커밋은 지워도 이력에 남는다** | 키를 넣고 다음 커밋에서 지워도, 그 커밋을 받아 간 사람·백업·포크에는 그대로 있다. `git log -p`로 누구나 볼 수 있다 |
| **공개 저장소는 몇 분 안에 수집된다** | 공개 저장소에 올라간 키를 찾는 자동 수집기가 늘 돌고 있다. "잠깐 올렸다 지웠다"도 안전하지 않다 |
| **사람 눈으로는 못 찾는다** | 설정 파일 한 줄, 테스트 코드, 주석, 로그 파일에 섞인다. Claude가 디버깅하며 넣은 값일 수도 있다 |
| **비공개 저장소도 위험하다** | 저장소를 볼 수 있는 사람이 모두 키를 갖게 된다. 퇴사자, 외주, 나중에 공개로 바꾸는 경우 |

**어떻게 찾나**: ① 서비스마다 정해진 **형식**(AWS 접근 키는 `AKIA`로 시작하는 20자, GitHub 토큰은 `ghp_`로 시작 등)으로 찾고, ② 형식이 없는 값은 `password:`, `secret-key:` 같은 **이름 옆의 무작위해 보이는 문자열**(엔트로피가 높은 값)로 찾습니다. 그래서 가짜 값이 걸리기도 하고(오탐), 형식이 특이한 사내 비밀번호는 못 찾기도 합니다. 스캔은 **안전망**이고, 1차 방어는 7회차의 "비밀은 환경 변수·비밀 저장소에, 저장소에는 변수 이름만"입니다.

**세 겹으로 막는다**

```mermaid
flowchart LR
    A["① 내 PC<br/>gitleaks로 직접 스캔<br/>(커밋·푸시 전)"] --> B["② 푸시할 때<br/>GitHub Push protection<br/>비밀이 있으면 푸시 거부"]
    B --> C["③ 저장소 전체<br/>GitHub Secret scanning<br/>이력까지 검사해 알림"]
    C --> D["발견되면<br/>키 폐기·교체 → 이력 정리"]
```

| 단계 | 도구 | 언제 | 비용·조건 |
|---|---|---|---|
| ① 내 PC | **gitleaks** (오픈소스) | 내가 실행할 때, 또는 CI에서 | 무료, 어떤 저장소든 |
| ② 푸시할 때 | GitHub **Push protection** | `git push` 하는 순간 | 저장소 종류에 따라 다름 (3-3) |
| ③ 저장소 전체 | GitHub **Secret scanning** | 켜는 순간 이력 전체, 이후 새 커밋마다 | 저장소 종류에 따라 다름 (3-3) |

### 3-2. gitleaks로 직접 스캔하기 (어떤 저장소든 됨)

**설치** (gitleaks 공식 README의 방식)

| PC | 방법 |
|---|---|
| macOS | `brew install gitleaks` |
| Windows | GitHub `gitleaks/gitleaks` 저장소 **Releases**에서 `gitleaks_…_windows_x64.zip`을 받아 풀고 `gitleaks.exe` 폴더를 PATH에 (회사 보안 정책에 따라 IT 확인) |
| Docker | `docker run --rm -v "${PWD}:/repo" zricethezav/gitleaks:latest git /repo` |
| Go가 있으면 | `go install github.com/zricethezav/gitleaks/v8@latest` |

**실행** (저장소 루트에서)

```bash
gitleaks git --redact .      # 커밋 이력 전체 검사 ← 기본으로 이것
gitleaks dir --redact .      # 지금 있는 파일만 검사 (커밋하지 않은 파일 포함)
# 예전 명령 gitleaks detect --source . --redact 도 아직 동작한다 (v8.19부터 git·dir로 나뉨)
```

- `--redact`: 찾은 값을 화면에 `REDACTED`로 가려서 출력합니다. **이 옵션 없이 실행한 결과를 Claude에게 붙여 넣지 않습니다** (비밀 값이 그대로 대화에 들어갑니다)
- 종료 코드: 찾으면 **1**, 없으면 **0** → CI 단계에 그대로 넣으면 비밀이 있을 때 빌드가 실패합니다
- 결과를 파일로: `--report-format json --report-path leaks.json` (이 파일은 커밋하지 않습니다)

**실제로 해 보면** (이 교재 환경에서 만든 연습 저장소: AWS 키 모양의 가짜 값을 `application-dev.yml`에 커밋하고, 다음 커밋에서 지움)

```text
$ gitleaks dir --redact .
INF no leaks found                          ← 지금 파일에는 없다

$ gitleaks git --redact -v .
Finding:     access-key: REDACTED
RuleID:      generic-api-key
File:        backend/src/main/resources/application-dev.yml
Line:        3
Commit:      e5aeb2d…                        ← 지운 커밋 이전의 커밋에 남아 있다
Finding:     secret-key: REDACTED
…
INF 3 commits scanned.
WRN leaks found: 2                          ← 종료 코드 1
```

**지금 파일을 지워도 이력에는 남는다**는 것을 눈으로 확인하는 것이 이 실습의 핵심입니다. 그래서 발견되면 **키를 먼저 폐기**합니다 (3-4).

**오탐 다루기**: 테스트용 가짜 값이 걸리면 그 줄 끝에 `# gitleaks:allow` 주석을 달거나, 저장소 루트의 `.gitleaksignore`에 결과의 `Fingerprint` 값을 한 줄씩 적습니다. 무엇을 왜 예외로 했는지 PR에 남깁니다.

### 3-3. GitHub에서 켜기: Secret Protection과 Push protection

> 💡 **"Settings에 Secret scanning이 없다"면 정상일 수 있습니다.** 두 가지가 바뀌었습니다.
> 1. **메뉴 이름**: 예전의 "Code security and analysis" 화면이 지금은 저장소 Settings 왼쪽 메뉴의 **"Security and quality"**(화면에 따라 "Security") 묶음 아래 **Advanced Security**입니다. Secret scanning과 Push protection은 그 안의 **Secret Protection** 항목에 들어 있습니다
> 2. **쓸 수 있는 저장소**: 아래 표처럼 **개인 계정의 Private 저장소에서는 Secret scanning을 켤 수 없습니다** (메뉴 자체가 보이지 않거나 켤 수 없게 나옵니다). EduHub를 개인 계정에 Private으로 만들었다면 이 경우입니다

| 저장소 | Secret scanning | Push protection (저장소 단위) |
|---|---|---|
| **Public** (개인·조직 모두) | ✅ 무료로 **자동** 실행 | ✅ 무료로 켤 수 있음 (기본은 꺼짐) |
| **개인 계정의 Private** | ❌ 없음 | ❌ 없음 → 3-2의 gitleaks + 아래 "나를 위한 Push protection"(공개 저장소에만 적용)으로 |
| **조직의 Private** (Team·Enterprise 요금제) | 유료 **GitHub Secret Protection**을 켜야 함 | 〃 |

(GitHub 문서 "About secret scanning", "Enabling secret scanning for your repository", "Enabling push protection for your repository" 기준, 2026년 11월 확인)

**저장소에서 켜는 순서** (Public 저장소, 또는 Secret Protection이 있는 조직 저장소)

1. 저장소 첫 화면 → 탭 줄 오른쪽 **⚙ Settings**
2. 왼쪽 메뉴 **"Security and quality"**(또는 "Security") 묶음 → **Advanced Security**
3. **Secret Protection** 오른쪽 **Enable** → 영향 안내를 읽고 **Enable Secret Protection** (이것이 Secret scanning을 켭니다. Public 저장소는 이미 켜져 있을 수 있습니다)
4. 같은 Secret Protection 영역에서 **Push protection** 오른쪽 **Enable**
5. 결과 보기: 저장소 상단 **Security and quality**(또는 Security) 탭 → **Secret scanning** 알림 목록

**개인 계정 설정: "나를 위한 Push protection"** (요금제와 상관없음, 기본으로 켜져 있음)

1. 화면 오른쪽 위 프로필 사진 → **⚙ Settings** (저장소 Settings가 아니라 **내 계정** Settings)
2. 왼쪽 "Security" 묶음 → **Code security**
3. "User" 아래 **Push protection for yourself**가 켜져 있는지 확인

이 설정은 **내가 공개 저장소에** 비밀을 푸시하는 것을 막아 줍니다. 비공개 저장소는 막지 않습니다.

**Push protection이 막으면 이렇게 보입니다**

```text
$ git push
remote: error: GH013: Repository rule violations found for refs/heads/fix/nfr-03-security.
remote: - GITHUB PUSH PROTECTION
remote:   ——————————————————————————————————————
remote:     Resolve the following violations before pushing again
remote:
remote:     - Push cannot contain secrets
remote:     …  (어느 커밋·파일·줄인지, 그리고 허용 요청 URL)
```

(메시지 형식은 GitHub 문서 "Push protection from the command line"의 설명을 따른 예이며, 줄 모양은 조금 다를 수 있습니다.)

- **고치는 법**: 그 비밀이 들어간 커밋을 고칩니다 (아직 푸시 전이니 `git commit --amend` 또는 `git rebase -i`로 해당 줄 제거 → 다시 푸시). 마지막 커밋에서만 지우면 **이전 커밋에 남아 있어 계속 막힙니다**
- **우회(bypass)**: 출력된 URL에서 이유를 고르면 푸시할 수 있습니다. "테스트용(It's used in tests)"·"오탐(It's a false positive)"은 닫힌 알림, "나중에 고침(I'll fix it later)"은 열린 알림이 남고, 관리자에게 메일이 갑니다. **진짜 키라면 우회하지 않습니다**

### 3-4. 발견되면: 순서가 중요하다

| 순서 | 할 일 | 이유 |
|---|---|---|
| 1 | **키를 폐기·교체** (AWS 콘솔, DB 비밀번호 변경, 토큰 삭제 후 재발급) | 이력에서 지우는 동안에도 이미 복사된 키는 쓸 수 있다. 폐기만이 확실하다 |
| 2 | 새 키는 환경 변수·비밀 저장소로 (7회차 Step 6) | 같은 실수 반복 방지 |
| 3 | 그 키로 무엇이 접근되었는지 확인 (클라우드·DB 접속 기록) | 이미 쓰였을 수 있다 |
| 4 | (필요하면) 이력에서 지우기 (`git filter-repo` 등, 팀 합의 후) | 모든 사람이 다시 받아야 하는 큰 작업. 1을 했다면 급하지 않다 |
| 5 | 사후 분석: 왜 들어갔나, 어느 단계에서 막을 수 있었나 | 6장 사후 분석과 같은 방식 |

> ⚠️ Claude에게 "이 키 지워줘"라고 맡기면 **파일에서만** 지우고 끝날 수 있습니다. 폐기·교체는 사람이 해당 시스템에서 직접 하고, Claude에게는 "이 값을 쓰는 곳을 환경 변수로 바꿔줘"를 맡깁니다.

**EduHub에서 할 일** (오늘 실습)

1. `gitleaks git --redact .`로 저장소 이력 전체 검사 → 결과를 PR 본문에 "발견 0건" 또는 처리 내역으로
2. 저장소가 Public이면 Secret Protection·Push protection 켜기, 개인 Private이면 내 계정의 "Push protection for yourself"만 확인
3. (선택) CI에 gitleaks 단계 추가 (종료 코드 1이면 실패)

> ✅ 이 교재 환경에서 gitleaks v8.28.0(공식 소스를 Go로 빌드)으로 위 연습 저장소를 검사해 `dir`은 0건(종료 코드 0), `git`은 지운 커밋 이전의 2건(종료 코드 1)이 나오는 것을 확인했습니다. 예전 명령 `detect --source .`도 같은 결과였습니다. 오탐 처리도 확인했습니다: `# gitleaks:allow` 주석을 단 줄은 제외(종료 코드 1 → 0), `.gitleaksignore`에 Fingerprint를 적으면 이력의 2건이 제외(1 → 0). ⚠️ GitHub 설정 화면은 이 환경에서 열어 보지 못했습니다. 메뉴 이름·순서·요금제 조건은 GitHub 공식 문서를 따랐고, 화면에 따라 조금 다를 수 있으니 수업 전에 강사 저장소에서 한 번 확인합니다.

## Step 4. PR 자동 보안 리뷰 붙이기

7회차 CI 워크플로우 옆에 리뷰 워크플로우를 추가합니다. (`ANTHROPIC_API_KEY`는 팀장이 저장소 Secrets에 등록)

```yaml
# .github/workflows/claude-review.yml
name: Claude PR Review
on:
  pull_request:
    types: [opened, synchronize]
permissions:
  contents: read
  pull-requests: write
jobs:
  review:
    if: github.event.pull_request.head.repo.full_name == github.repository   # 외부 포크 PR 제외
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            CLAUDE.md 규칙에 따라 이 PR의 변경 파일만 리뷰해줘.
            특히 NFR-03을 확인해줘: 본인 확인 없는 수정·취소, 입력 검증 누락, 비밀 정보, 내부 정보 노출.
            JPA는 N+1과 엔티티 직접 반환을 확인해줘.
            문제는 해당 줄에 인라인 코멘트로, 문제가 없으면 한 줄 요약만.
```

> ⚠️ 액션 입력값(`prompt`, `claude_args` 등)은 버전에 따라 다를 수 있으므로 공식 문서로 확인합니다. 비용을 줄이려면 리뷰 범위를 "변경 파일만"으로 한정하고 턴 수를 제한합니다.

```bash
git add backend .github/workflows/claude-review.yml
git commit -m "fix(enrollment): 본인 신청만 취소하도록 인가 추가 (NFR-03)"
git push -u origin fix/nfr-03-security
```

> ✅ **확인**: PR을 열면 몇 분 안에 Claude 리뷰 코멘트가 달리고, 사람 리뷰어 1명이 승인해야 병합됩니다. Claude 코멘트가 지적한 것 중 **틀린 지적**이 있으면 근거와 함께 답글로 반박합니다. 그것도 리뷰입니다.

---

# 6. 따라하기 ③: ⭐ AI 없이 핫픽스 훈련 + Claude와 사후 분석

> **"AI 없이는 고칠 수 없는 시스템은 운영할 수 없는 시스템이다."** (1회차 7-3)
> 7주 동안 Claude와 만든 EduHub를, 오늘은 **Claude 없이** 고치고 배포하고 되돌립니다. 그리고 끝난 뒤에 Claude와 함께 원인을 분석하고 재발을 막습니다.

---

## 6-1. 훈련 규칙

| 항목 | 규칙 |
|---|---|
| 시나리오 | 강사가 각 팀 스테이징에 **운영 장애처럼 보이는 버그**를 넣은 버전을 배포합니다 (버그 커밋을 `main`에 병합 → 7회차 파이프라인이 배포. 무엇인지 알려 주지 않음) |
| 시간 | **20분** 타임박스 (탐지·완화·수정·배포) + 사후 분석 10분 |
| 금지 | Claude Code, 웹 Claude, 다른 AI 도구 (데스크톱 앱 종료) |
| 허용 | IDE, 로그, `git`, 런북, CI/CD 파이프라인, 팀원과 대화, 공식 문서 검색 |
| 역할 | **지휘(1)**: 상황 판단·결정·기록 / **분석(1~2)**: 로그·코드로 원인 위치 / **수정·배포(1)**: hotfix 브랜치, 빌드, 배포 |
| 성공 기준 | ① 15분 안에 **완화**(롤백 등)로 사용자 영향 차단 ② 원인 수정 PR 병합 후 스테이징 재배포 ③ 타임라인 기록 |

> 💬 **강사 노트 (버그 시나리오 예시, 팀마다 다르게):**
> - 취소해도 대기 1순위가 승급되지 않음 (승급 메서드 호출 누락)
> - 특정 분류(`category=null`) 과정이 있으면 과정 목록이 500 (NPE)
> - 신청 마감 시간 비교가 UTC/KST 차이로 9시간 어긋남
> - 마이그레이션에서 `NOT NULL` 컬럼 추가로 신규 신청이 실패
> 버그 커밋은 일반 기능 커밋처럼 보이게 메시지를 쓰고, 이전 정상 버전(커밋 SHA 앞 12자리)을 기록해 둡니다. 막힌 팀에는 5분마다 힌트 하나(로그 위치 → 패키지 → 클래스).

---

## 6-2. 훈련 흐름

```mermaid
flowchart TD
    D["🚨 장애 감지<br/>스테이징 오류·사용자 신고"] --> T["지휘: 타임라인 기록 시작<br/>(시각, 증상, 결정)"]
    T --> Q{"즉시 완화 가능?<br/>롤백·기능 플래그"}
    Q -- 예 --> RB["런북대로 이전 버전으로 롤백<br/>→ 사용자 영향 차단 확인"]
    Q -- 아니오 --> AN
    RB --> AN["원인 위치 파악<br/>로그 → 구조 지식 → 최근 커밋 (git log, git diff)"]
    AN --> HF["hotfix/… 브랜치에서 최소 수정<br/>직접 빌드·테스트"]
    HF --> PR["긴급 PR → 리뷰 1명 → 병합"]
    PR --> DP["파이프라인으로 스테이징 재배포"]
    DP --> V["정상 확인 → 종료 선언"]
    V --> PM["📝 사후 분석 (여기서부터 Claude 사용)"]
```

## 6-3. 단계별 명령 (Claude 없이)

**① 감지와 완화**: 런북의 롤백 절차를 그대로 실행합니다. 예 (팀 런북에 맞게):

```powershell
# 스테이징 PC: 지금 버전과 직전 정상 버전 확인
Get-Content C:\eduhub-staging\backend\deploy-history.log -Tail 5
Get-Content C:\eduhub-staging\backend\previous.txt
```

```bash
# 런북의 롤백 ① 파이프라인으로 (빌드 없이 직전 버전, staging 승인 후 실행)
gh run list --workflow deploy-backend.yml --limit 5
gh workflow run deploy-backend.yml -f rollback=true
```

```powershell
# 런북의 롤백 ② 스테이징 PC에서 직접 (GitHub·러너 장애, 승인자 부재 시)
.\scripts\deploy-backend.ps1 -Root C:\eduhub-staging\backend -Rollback
```

**② 원인 위치 파악**: 2회차에 말한 "구조를 아는 개발자"의 시간입니다.

```powershell
# 스테이징 PC: 백엔드 로그에서 예외와 위치 찾기 (마지막 3건, 뒤 20줄까지)
Select-String -Path C:\eduhub-staging\backend\logs\eduhub.log -Pattern "ERROR" -Context 0,20 | Select-Object -Last 3
```

```bash
# 최근 바뀐 것 보기
git log --oneline -10
git show <의심 커밋> --stat
git diff <이전-정상-커밋> HEAD -- backend/src/main/java/com/example/eduhub/enrollment/
```

**③ 수정·빌드·배포**

```bash
git switch main && git pull
git switch -c hotfix/waiting-promotion
# IDE에서 직접 최소 수정 (리팩터링 금지, 원인만)
cd backend && ./gradlew test --tests "*Enrollment*"
git add -p                                   # 수정한 부분만 골라 담기
git commit -m "fix(enrollment): 취소 시 대기자 승급 누락 수정"
git push -u origin hotfix/waiting-promotion
# 긴급 PR → 리뷰 1명 → Squash merge → 파이프라인이 스테이징 배포
```

**지휘가 기록하는 타임라인 (예)**

| 시각 | 사건·결정 |
|---|---|
| 14:02 | 신고: 취소했는데 대기자가 확정되지 않음 |
| 14:05 | 이전 버전으로 롤백 결정 → 14:08 롤백 완료, 정상 확인 |
| 14:12 | 원인 위치: `EnrollmentService.cancel()`에서 `promoteNext` 호출 누락 (커밋 abc123) |
| 14:16 | hotfix PR 병합, 14:19 재배포, 14:20 종료 선언 |

---

## 6-4. 사후 분석: 이제 Claude를 씁니다

훈련이 끝나면 데스크톱 앱을 다시 열고, **새 세션**에서 사후 분석을 합니다. AI는 사후 정리 단계에서 가장 큰 도움이 됩니다. ([개발자 책임 원칙 2](../examples/27-developer-responsibility.md#긴급-핫픽스-흐름))

```text
[목표] 오늘 스테이징 장애의 사후 분석(포스트모템)과 재발 방지 작업.

[자료]
- 타임라인: (지휘 기록 붙여넣기)
- 원인 커밋: abc123, 수정 커밋: def456
- @docs/runbook.md

[해줄 일]
1. 원인 커밋과 수정 커밋을 비교해서 근본 원인을 3줄로 설명해줘
2. 이 버그를 잡았어야 할 테스트가 왜 없었는지 분석하고, 회귀 테스트를 추가해줘 (취소 → 대기 1순위 확정)
3. docs/postmortem/2026-xx-xx-waiting-promotion.md 를 작성해줘: 요약, 영향, 타임라인, 근본 원인, 잘된 점, 개선할 점, 후속 조치
4. 런북에서 오늘 막혔던 부분을 고칠 제안 (직접 고치지 말고 제안만)

[진행 방식] 1번과 2번의 계획부터 보여줘.
```

**포스트모템 원칙**
- **사람을 탓하지 않습니다** (blameless). "누가"가 아니라 "어떤 장치가 없어서" 통과했는지
- 후속 조치는 **장치**로: 회귀 테스트, CI 체크, 리뷰 체크리스트, 런북 수정
- AI가 만든 커밋이었다면: 그 PR의 리뷰에서 무엇을 놓쳤는지도 기록합니다

> ✅ **확인**: 회귀 테스트가 추가된 PR, `docs/postmortem/` 문서, 런북 개선 제안. 10회차 캡스톤 발표에서 **"우리 팀의 핫픽스 시간"**을 함께 발표합니다.

---

## 6-5. 자주 겪는 문제

| 증상 | 원인과 해결 |
|---|---|
| 롤백 명령을 몰라 시간이 다 감 | 런북에 명령어가 없음 → 사후 분석의 1순위 개선 항목 |
| 이전 정상 버전을 모름 | 스테이징 PC의 `previous.txt`·`deploy-history.log`, Actions 실행 이력 확인. 롤백은 한 단계만이므로 더 이전은 `-Version <버전>` |
| 로그에 원인이 안 보임 | 로그 레벨·요청 ID 부족 → 후속 조치: 예외 로깅, 요청 추적 ID |
| 고치다가 리팩터링까지 함 | 핫픽스는 **최소 수정**. 정리는 사후에 별도 PR |
| 긴급 PR 리뷰를 건너뜀 | 긴급해도 리뷰 1명 + CI. 브랜치 보호가 막아 주는지 확인 |
| k6 결과가 매번 크게 다름 | 서버 워밍업 후 측정, 같은 PC·같은 데이터·같은 VU로 비교 |
| `k6`: 명령을 찾을 수 없음 | 설치 후 **터미널을 새로** 연다 (PATH 갱신). Releases 압축 파일을 쓴 경우 `k6.exe` 폴더를 PATH에 |
| k6에서 `connection refused` | 서버가 안 떠 있거나 포트가 다름. Docker로 k6를 돌리면 `localhost`가 컨테이너 자신 → `-e BASE_URL=http://host.docker.internal:8080` |
| k6 check `20건` 실패 | 시드가 없거나 페이지 번호가 데이터보다 큼. 시드 확인 SQL(1-3 ④)로 건수 확인 |
| 시드 실행 시 `relation "course" does not exist` | Flyway가 아직 테이블을 만들지 않음 → 서버를 한 번 띄운 뒤 다시 |
| 시드 실행 시 `violates check constraint` 등 | Claude가 쓴 SQL이 스키마와 다름 → 오류 메시지를 그대로 붙여 고치게 하고, `BEGIN`으로 묶여 있으니 DB는 그대로 |
| Claude PR 리뷰가 달리지 않음 | Secrets 이름, 워크플로우 `permissions`, 외부 포크 조건, Actions 로그 확인 |

---

# 7. 더 해 보기 (시간이 남으면 수업 중, 아니면 과제)

---

## 7-1. SDK 미니 봇: 릴리스 노트 초안

`main`에 병합된 커밋 목록으로 릴리스 노트 초안을 만드는 작은 사내 도구입니다. 쓰기 도구는 허용하지 않습니다.

```ts
// tools/release-notes/index.ts
import { query } from "@anthropic-ai/claude-agent-sdk";

const from = process.argv[2] ?? "v0.7.0";   // 이전 릴리스 태그

const prompt = `
${from}..HEAD 사이의 커밋을 git log로 확인하고 릴리스 노트 초안을 마크다운으로 써줘.
- feat / fix / perf / 기타로 분류, 요구사항 ID(FR-xx, NFR-xx)가 있으면 함께 표시
- 사용자에게 보이는 변경을 먼저, 내부 변경은 마지막에 한 줄씩
- 파일은 수정하지 말고 결과만 출력해줘
`;

for await (const message of query({
  prompt,
  options: {
    cwd: process.cwd(),
    allowedTools: ["Bash(git log *)", "Bash(git show *)"], // 읽기는 cwd 안이면 기본 허용 (2-2 예제 ①)
    disallowedTools: ["Write", "Edit"],
    permissionMode: "dontAsk",
    maxTurns: 10,
  },
})) {
  if (message.type === "result") console.log(message);
}
```

> SDK의 옵션 이름과 메시지 형식은 버전에 따라 다를 수 있으므로 공식 문서(Agent SDK)로 확인합니다. 출력 결과는 사람이 검토한 뒤 GitHub Release에 붙입니다.

**생각해 볼 점**: 이 봇은 `claude -p` 한 줄로도 만들 수 있습니다. **SDK가 필요한 순간**은 언제일까요? (예: 사내 포털 버튼에서 실행, 결과를 사내 시스템 API로 저장, 사용자별 권한 제어)

## 7-2. CI 실패 자동 분석 붙이기

[빌드·배포 파이프라인 4-2](../examples/28-build-deploy-pipeline.md#4-2-ci-실패-원인-자동-분석)의 워크플로우를 EduHub CI에 추가하고, 일부러 테스트를 하나 깨뜨린 PR로 분석 코멘트가 달리는지 확인합니다. 분석 결과가 **맞았는지** 팀이 판정해 기록합니다.

## 7-3. 선택 심화 A: RAG 맛보기

EduHub에 "교육 도우미"(과정 안내 문서에 대한 질의응답)를 붙이고 싶다면, 사내 데이터를 외부로 보내지 않는 **오픈소스 LLM(Ollama) + RAG** 구성을 검토합니다. 이번 과정에서는 개념과 구성만 봅니다. ([오픈소스 LLM으로 RAG 구축](../examples/26-rag-with-ollama.md))

---

# 8. 정리

---

## 8-1. 오늘의 핵심

1. **헤드리스·CI의 Claude는 보조 리뷰어·분석가**: 읽기 전용 도구, 턴 수·비용 제한, 코멘트만. 승인과 배포는 사람
2. **SDK는 사내 도구 안에 에이전트가 필요할 때**: 허용 도구·권한·작업 폴더를 코드로 정한다. 여러 팀이 도구를 만들면 **표준(MCP·SDK) 위에 얇은 사내 도구 키트**를 두고 그 위에서 만든다 (2-4)
3. **거버넌스 = 최소 규칙의 강제**: 관리 설정(deny, bypass 금지, 승인된 MCP·마켓플레이스), 사용량 모니터링, 감사, 리뷰 문화
4. **성능은 측정 → 원인 → 개선 → 재측정**, 결과는 쿼리 수 테스트로 고정한다. 방안 선택은 아키텍처 지식이 있는 개발자가
5. ⭐ **AI 없이 고치고, 배포하고, 되돌릴 수 있어야** 시스템의 주인이다. AI는 사후 분석과 재발 방지에서 힘을 발휘한다

---

## 8-2. 확인 퀴즈

1. 헤드리스(`claude -p`)로 CI에서 Claude를 돌릴 때 반드시 걸어야 할 통제 세 가지는?
2. 외부 포크에서 온 PR에 Claude 리뷰를 돌리면 안 되는 이유는?
3. 같은 "릴리스 노트 만들기"를 `claude -p`가 아니라 Agent SDK로 만들어야 하는 경우는?
4. 관리 설정(managed settings)이 프로젝트 `.claude/settings.json`과 다른 점은?
5. 과정 목록 N+1을 고칠 때 `JOIN FETCH`를 페이징과 함께 쓰면 생기는 문제는?
6. N+1을 고친 뒤 "쿼리 수 테스트"를 추가하는 이유는?
7. 핫픽스 훈련에서 원인 수정보다 **롤백을 먼저** 하는 이유는?
8. 포스트모템의 후속 조치를 "더 조심하자"가 아니라 무엇으로 정해야 하나?

<details>
<summary>정답 보기</summary>

1. 허용 도구를 읽기 전용으로(`--allowedTools`, `--permission-mode dontAsk`), 턴 수·비용 제한(`--max-turns`, `--max-budget-usd`), 결과는 코멘트만 하고 수정·푸시·배포 권한을 주지 않기
2. 시크릿(`ANTHROPIC_API_KEY`)이 노출될 수 있고, PR 내용에 숨은 지시(프롬프트 인젝션)로 의도하지 않은 동작을 할 수 있기 때문
3. 사내 포털·봇·서비스 안에서 실행해야 하거나, 결과를 사내 시스템에 저장하거나, 사용자별 권한·도구를 코드로 제어해야 할 때
4. 조직이 배포하고 누구도 덮어쓸 수 없다. 우선순위가 가장 높고, deny 규칙·모드 금지·승인된 MCP 같은 정책을 강제할 수 있다
5. 컬렉션 fetch join과 페이징을 함께 쓰면 Hibernate가 결과 전체를 메모리로 가져와 페이징한다(`HHH90003004` 경고). 데이터가 많으면 오히려 느려지고 메모리를 많이 쓴다
6. 나중에 누군가(AI 포함) 코드를 바꿔 N+1이 다시 생겨도 CI에서 자동으로 잡기 위해
7. 사용자 영향을 가장 빨리 막을 수 있고, 원인 분석과 수정을 서두르다 생기는 2차 실수를 줄이기 때문
8. 장치로: 회귀 테스트, CI 체크, 리뷰 체크리스트, 런북 수정, 모니터링·알림

</details>

---

## 8-3. 과제 (9회차 Wrap-up 전까지)

- [ ] NFR-02 PR 병합: 측정 기록표(Before/After)와 쿼리 수 테스트 포함
- [ ] NFR-03 PR 병합: 본인 확인, 입력 검증, gitleaks 결과, Claude PR 리뷰 워크플로우
- [ ] 핫픽스 훈련 사후 분석: 회귀 테스트 PR, `docs/postmortem/…md`, 런북 개선 반영
- [ ] 팀 거버넌스 제안 1쪽: "우리 회사에 Claude Code를 도입한다면 관리 설정으로 강제할 것 / 팀 자율로 둘 것"
- [ ] (선택) 7-1 SDK 미니 봇 또는 7-2 CI 실패 분석
- [ ] (선택, 제조업) 2-4: 우리 회사 도구 키트의 첫 후보 3개(여러 팀이 함께 쓰는 입력 타입·결과 형식·안전 규칙)와 도구 이름 규칙 초안 1쪽
- [ ] 7주 회고 준비: 주차별 `/usage` 기록, 가장 효과가 컸던 토큰 절약 습관, Claude에게 맡긴 것 / 직접 한 것의 변화

## 8-4. 다음 회차 예고

**9주차: Wrap-up — 1~8회차 다시 점검하고 캡스톤 준비하기**
- 1~8회차 핵심을 한 장의 지도로 다시 연결하고, 개인 자가 진단·종합 퀴즈로 빈 곳 찾기
- 팀 EduHub 점검: 요구사항 달성 현황, Claude Code 구성, 보안·환경, 남은 일 목록
- 캡스톤 리허설: 짝 팀과 AI 없이 구조 설명과 직접 수정 연습

**10주차: 캡스톤 발표**
- 팀별 데모: EduHub 기능(FR)과 비기능 품질(NFR) 결과
- **구조 이해도 검증 (NFR-08)**: AI 없이 아키텍처와 핵심 파일 역할 설명, 강사가 고른 변경 하나를 직접 수정·빌드
- Claude Code 활용 사례, 핫픽스 훈련 결과와 포스트모템, 7주 사용량과 토큰 절약 회고

**미리 읽어 오면 좋은 자료**
- [9회차 개인 자가 진단표](./session-09-wrap-up.md#2-1-개인-자가-진단표) (수업 전에 채워 오기)
- [캡스톤 주제와 평가](../examples/05-automation-and-capstone.md#-캡스톤-프로젝트-주제-예시)
- [바이브 코딩 시대, 개발자의 책임](../examples/27-developer-responsibility.md)
- [2회차 3-7. 어떻게 진행하나 (평가 기준)](./session-02-hands-on-kickoff.md#3-7-어떻게-진행하나)

---

## 참고 자료

- Claude Code 공식 문서: Headless mode, GitHub Actions, Agent SDK, Settings(관리 설정), Managed MCP, Monitoring — https://code.claude.com/docs
- 교육 저장소: [자동화와 캡스톤](../examples/05-automation-and-capstone.md), [빌드·배포 파이프라인](../examples/28-build-deploy-pipeline.md), [권한](../examples/23-permissions.md), [토큰 최적화](../examples/08-token-optimization.md), [개발자의 책임](../examples/27-developer-responsibility.md), [에이전트](../examples/10-agents.md), [Java·React 에이전트](../examples/18-java-react-agents.md), [MCP 서버 가이드](../examples/19-mcp-server-guide.md), [RAG with Ollama](../examples/26-rag-with-ollama.md)
- 이전 회차: [1회차](./session-01-introduction.md), [2회차](./session-02-hands-on-kickoff.md)
- 도구: [k6](https://k6.io/), [gitleaks](https://github.com/gitleaks/gitleaks)
