# 비교: Claude Code vs OpenAI Codex vs Google(Antigravity, Gemini CLI, Jules)

> 관련 모듈: 1, 2 (도입 검토, 20과 연결)
> ⚠️ **2026년 10월 기준**이며, 이 분야는 몇 달 단위로 크게 바뀝니다. 실제 도입 결정 전에는 각 사의 **공식 문서와 요금 정책**을 반드시 확인하세요. 모델 성능·가격·사용 한도는 자주 바뀌고 출처마다 다르므로 이 문서에서는 다루지 않습니다.

---

## 1. 각 제품 소개

### Claude Code (Anthropic)

- 터미널(CLI), IDE 확장, 데스크톱 앱, 웹(클라우드), 모바일에서 쓰는 코딩 에이전트
- 지침 파일 `CLAUDE.md`, 스킬, 서브에이전트, 훅, MCP, 플러그인, Agent SDK
- 이 교육과정 전체가 다루는 도구

### OpenAI Codex

- OpenAI의 코딩 에이전트. **Codex CLI**는 Apache-2.0 오픈소스로 로컬에서 실행
- 사용 형태: CLI, IDE 확장(VS Code 등), 데스크톱 앱, 클라우드 에이전트(Codex Web)
- ChatGPT 계정 로그인(유료 요금제 포함) 또는 API 키로 사용
- 지침 파일로 `AGENTS.md`를 사용하고, 스킬과 MCP를 지원

### Google: Antigravity, Gemini CLI, Jules

| 제품 | 형태 | 현황 (2026년 10월) |
|---|---|---|
| **Antigravity** | 에이전트 중심 IDE·데스크톱 앱. 여러 에이전트를 관리하는 Agent Manager, 내장 브라우저 | Google의 주력 개발 에이전트 플랫폼 (Antigravity 2.0) |
| **Antigravity CLI** | Antigravity와 같은 구조를 공유하는 터미널 에이전트 | 2026년 5월 발표, Gemini CLI의 후속 |
| **Gemini CLI** | 오픈소스(Apache-2.0) 터미널 에이전트 | 2026년 6월 18일부터 개인(무료·AI Pro·Ultra) 사용자 서비스 종료, Antigravity CLI로 전환. 기업 라이선스·유료 API 키 사용자는 계속 사용 가능 |
| **Jules** | GitHub 저장소를 받아 클라우드 VM에서 비동기로 작업하고 PR을 만드는 에이전트 | 서비스 중. 세부 기능·요금은 공식 사이트 확인 필요 |

Google 발표에 따르면 Antigravity CLI는 Gemini CLI의 Agent Skills, Hooks, Subagents, Extensions(Antigravity plugins로 이름 변경)를 이어받았습니다.

---

## 2. 개념 대응표: "같은 개념, 다른 이름"

세 회사의 도구는 **구조가 점점 비슷해지고 있습니다.** 이 교육과정에서 배운 개념은 대부분 다른 도구에도 그대로 적용됩니다.

| 개념 | Claude Code | OpenAI Codex | Google (Antigravity / Gemini CLI) |
|---|---|---|---|
| 프로젝트 지침 파일 | `CLAUDE.md` | `AGENTS.md` | `GEMINI.md` (Gemini CLI) 등 |
| 스킬 (Agent Skills) | ○ `.claude/skills/` | ○ (`.agents/skills` 등) | ○ |
| MCP | ○ | ○ | ○ |
| 서브에이전트 | ○ | (공식 문서 확인) | ○ |
| 훅 | ○ | (공식 문서 확인) | ○ |
| 확장 묶음 | 플러그인 | (공식 문서 확인) | Extensions → Antigravity plugins |
| 터미널 | ○ | ○ (오픈소스) | Antigravity CLI / Gemini CLI(오픈소스) |
| IDE | VS Code, JetBrains 확장 | VS Code 계열 확장 | Antigravity(자체 IDE) |
| 데스크톱 앱 | ○ | ○ | Antigravity |
| 클라우드 비동기 작업 | 웹(claude.ai/code) | Codex Web | Jules |
| 병렬 에이전트 관리 | 데스크톱 멀티 세션, 에이전트 뷰 | (공식 문서 확인) | Agent Manager |
| 헤드리스·CI | `claude -p`, GitHub Actions | ○ | Gemini CLI용 GitHub Action 등 |
| 개발용 SDK | Claude Agent SDK | ○ | ○ |

> "(공식 문서 확인)" 항목은 이번 조사에서 공식 자료로 확인하지 못한 부분입니다. 지원하지 않는다는 뜻이 아닙니다.

> 💡 **Agent Skills**는 공개된 사양으로 정리되어 여러 회사 도구가 지원합니다([anthropics/skills의 spec](https://github.com/anthropics/skills)). 잘 만든 스킬은 다른 도구로 옮겨 쓰기 쉽습니다.

---

## 3. 비교 관점별 정리

| 관점 | Claude Code | OpenAI Codex | Google |
|---|---|---|---|
| **사용 형태의 폭** | CLI·IDE·데스크톱·웹·모바일 모두 | CLI·IDE·데스크톱·웹 | IDE 중심(Antigravity) + CLI + 비동기(Jules) |
| **오픈소스 여부** | CLI는 비공개 | CLI 오픈소스(Apache-2.0) | Gemini CLI 오픈소스, Antigravity CLI는 공식 자료에서 오픈소스 언급 없음 |
| **확장 생태계** | 스킬·에이전트·훅·MCP·플러그인·마켓플레이스 | 스킬·MCP 등 | 스킬·훅·서브에이전트·플러그인·MCP |
| **모델 선택** | Claude 모델 | OpenAI 모델 | Gemini 모델 중심 (일부 다른 모델 지원 보도 있음) |
| **기업 관리** | 관리 설정, 허용 MCP·플러그인 관리 등 | 기업 요금제 | Google Cloud·Gemini Code Assist 기업 라이선스 |
| **최근 변화의 크기** | 기능 추가가 잦음 | 릴리스 주기가 매우 짧음 | 제품 개편(Gemini CLI → Antigravity CLI)이 있었음 |

---

## 4. 어떻게 선택할까: 평가 기준

특정 도구를 "가장 좋다"고 단정하기보다, **우리 팀 기준으로 직접 평가**하는 것을 권장합니다.

| 기준 | 확인할 질문 |
|---|---|
| **실제 과제 성능** | 우리 코드베이스(Spring Boot + React)의 실제 이슈 5~10개를 각 도구로 해결해 보면? |
| **보안·정책** | 코드가 어디로 전송되나? 데이터 보존 정책, 사내 보안 심사 통과 여부, 클라우드 실행 위치 |
| **기업 관리** | 권한·MCP·플러그인을 중앙에서 통제할 수 있나? 감사 로그는? |
| **개발 환경 적합성** | 팀이 쓰는 IDE(IntelliJ, VS Code), OS(윈도우·WSL), 터미널 습관과 맞나? |
| **확장성** | 사내 프레임워크 지식(스킬), 사내 시스템 연동(MCP), 자동화(CI)를 얼마나 쉽게? |
| **비용** | 요금제, 사용 한도, 팀 규모에 따른 총비용 |
| **안정성·지원** | 제품 개편이 잦은가? 기업 지원 채널이 있나? |
| **이식성** | 다른 도구로 옮길 때 지침·스킬을 재사용할 수 있나? |

### 평가 실습 템플릿

| 과제 | Claude Code | Codex | Google | 메모 |
|---|---|---|---|---|
| N+1 수정 (P1) | 성공/실패, 소요 시간, 수정 횟수 | | | |
| 수강 취소 기능 (풀스택) | | | | |
| MFE 테마 문제 (P5) | | | | |
| 사내 프레임워크 규칙 준수 | | | | |
| 보안 리뷰 품질 | | | | |

---

## 5. 여러 도구를 함께 쓰는 팀을 위한 팁

팀원마다 다른 도구를 쓰거나, 도구를 바꿀 가능성이 있다면 **지식 자산을 도구 중립적으로** 관리합니다.

| 자산 | 도구 중립적으로 관리하는 방법 |
|---|---|
| 프로젝트 지침 | 공통 내용을 `AGENTS.md`에 두고, `CLAUDE.md`에서 `@AGENTS.md`로 가져온 뒤 Claude 전용 내용만 추가 (각 도구의 지침 파일 규칙은 공식 문서 확인) |
| 작업 절차 | **Agent Skills 형식**(`SKILL.md`)으로 작성 → 여러 도구에서 재사용 |
| 외부 연동 | **MCP 서버**로 만들기 → 표준이라 여러 도구에서 사용 |
| 검증 | `scripts/verify.sh` 같은 **일반 스크립트**로 → 어느 도구든 실행 가능 |
| 문서·설계 기록 | `docs/`의 마크다운 → 도구와 무관 |

```markdown
<!-- CLAUDE.md 예시 -->
@AGENTS.md

## Claude Code 전용
- 서브에이전트: .claude/agents/ 참고
- 훅: .claude/settings.json 참고
```

> 💡 이 교육과정에서 배운 **원리**(에이전트 루프, 컨텍스트 관리, 좋은 프롬프트, 검증 수단 제공, 최소 권한, 스킬·MCP 설계)는 도구가 바뀌어도 그대로 통합니다.

---

## 6. 토론 주제

1. 우리 팀이 코딩 에이전트를 선택할 때 가장 중요한 기준 3가지는?
2. 여러 도구를 허용할 것인가, 하나로 표준화할 것인가? 장단점은?
3. 지침·스킬·MCP를 도구 중립적으로 관리하려면 저장소 구조를 어떻게 바꿔야 할까? ([저장소 구조](./20-repository-structure.md))

---

## 참고 자료

- Claude Code 공식 문서: https://code.claude.com/docs
- OpenAI Codex CLI 저장소: https://github.com/openai/codex
- Google Cloud 블로그, "Choosing Antigravity or Gemini CLI" (2026-02-04): https://cloud.google.com/blog/topics/developers-practitioners/choosing-antigravity-or-gemini-cli
- Gemini CLI → Antigravity CLI 전환 공지 (2026-05-19): https://github.com/google-gemini/gemini-cli/discussions/27274
- Agent Skills 사양: https://github.com/anthropics/skills
