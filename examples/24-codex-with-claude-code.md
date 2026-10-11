# Codex와 Claude Code를 함께 사용하기: 왜, 어떻게

> 관련 모듈: 14, 17 (도구 비교는 [22번 문서](./22-claude-code-vs-codex-vs-google.md))
> 기준: [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc), [openai/codex](https://github.com/openai/codex) (2026년 10월 확인)
> ⚠️ 두 도구 모두 변화가 빠릅니다. 명령 이름과 옵션은 설치된 버전의 `--help`와 공식 문서로 확인하세요.

---

## 1. 왜 함께 쓰나

| 이유 | 설명 |
|---|---|
| **교차 리뷰** | 같은 모델이 쓰고 검토하면 같은 맹점을 공유하기 쉽다. **다른 회사의 모델이 검토하면** 놓친 문제를 찾을 가능성이 커진다 |
| **설계 반론 받기** | 한쪽이 만든 설계를 다른 쪽이 일부러 반박하게 해 가정과 트레이드오프를 점검 |
| **막혔을 때 다른 시도** | 한 도구가 같은 버그에서 계속 헤매면, 다른 도구에 넘겨 새 관점으로 시도 |
| **팀원의 선호와 계정 상황** | 팀원마다 익숙한 도구나 가진 구독이 다를 수 있다 |
| **가용성** | 한쪽의 사용 한도나 장애 시 다른 쪽으로 계속 작업 |
| **도구 평가** | 같은 과제를 두 도구로 해 보고 팀 기준으로 비교 |

> 💡 "두 개를 쓰면 무조건 좋다"는 아닙니다. 비용과 관리 부담이 늘어나므로 **교차 리뷰처럼 효과가 분명한 지점**에 쓰는 것이 핵심입니다.

---

## 2. 함께 쓰는 방법 5가지

| 방법 | 형태 | 난이도 | 추천 용도 |
|---|---|---|---|
| **A. 저장소 자산 공유** | 같은 저장소를 두 도구가 각자 사용 | 쉬움 | 모든 경우의 기반 |
| **B. 공식 플러그인 (codex-plugin-cc)** | Claude Code 안에서 Codex 호출 | 쉬움 | **교차 리뷰, 작업 넘기기** |
| **C. MCP로 연결** | Codex를 Claude Code의 도구로 (또는 반대로) | 중간 | 에이전트끼리 직접 위임 |
| **D. PR 단계 교차 리뷰** | 한쪽이 PR, 다른 쪽이 GitHub에서 리뷰 | 쉬움 | 팀 협업 흐름 |
| **E. 스크립트·CI 파이프라인** | `claude -p`와 Codex 비대화형 실행을 연결 | 중간 | 자동화 |

---

### 방법 A. 저장소 자산 공유 (기반)

두 도구가 같은 규칙과 도구를 쓰도록 저장소를 구성합니다.

| 자산 | 구성 |
|---|---|
| **지침** | 공통 규칙은 `AGENTS.md`(Codex가 읽음). `CLAUDE.md`는 `@AGENTS.md`로 가져오고 Claude 전용 내용만 추가 |
| **스킬** | Agent Skills 형식(`SKILL.md`)으로 작성. 각 도구의 스킬 폴더 위치는 다르므로 공식 문서 확인 후 연결(복사 또는 링크) |
| **외부 연동** | MCP 서버는 두 도구 모두 지원 → 같은 서버를 각자 등록 (`claude mcp add`, `codex mcp add`) |
| **검증** | `scripts/verify.sh` 하나로 → 어느 도구든 같은 기준으로 확인 |
| **설계·진행 기록** | `docs/design/`, `docs/progress/` → 도구를 바꿔도 맥락 유지 |

```markdown
<!-- CLAUDE.md -->
@AGENTS.md

## Claude Code 전용
- 서브에이전트: .claude/agents/
- 훅·권한: .claude/settings.json
```

```markdown
<!-- AGENTS.md (공통) -->
# EduHub
- 백엔드: Spring Boot 3, JPA. 규칙은 backend/AGENTS.md
- 프론트: React MFE, MUI. 규칙은 frontend/AGENTS.md
- 검증: ./scripts/verify.sh 통과 필수
- 생성 코드(frontend/packages/api-client/src/generated/) 수정 금지
```

> 하위 디렉터리 규칙도 같은 방식으로 `backend/AGENTS.md` + `backend/CLAUDE.md`(`@AGENTS.md`)로 구성할 수 있습니다. 각 도구가 지침 파일을 읽는 규칙은 공식 문서로 확인하세요.

---

### 방법 B. 공식 플러그인: Claude Code 안에서 Codex 쓰기 ⭐

OpenAI가 공개한 Claude Code용 플러그인(Apache-2.0)입니다. 내 컴퓨터의 Codex CLI를 감싸서 동작하므로 Codex 설치·로그인·설정을 그대로 사용합니다.

**준비물:** Node.js 18.18 이상, ChatGPT 계정(무료 포함) 또는 OpenAI API 키 (Codex 사용량에서 차감)

**설치**

```text
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

`/codex:setup`이 Codex 설치·인증 상태를 확인합니다. 필요하면 `npm install -g @openai/codex`로 설치하고 `!codex login`으로 로그인합니다.

**명령**

| 명령 | 하는 일 | 사용 예 |
|---|---|---|
| `/codex:review` | 커밋하지 않은 변경(또는 `--base <브랜치>` 대비 변경)을 Codex가 **읽기 전용으로 리뷰** | 구현 후 PR 전에 |
| `/codex:adversarial-review` | 설계 선택·가정·트레이드오프에 **반론을 제기하는 리뷰**. 집중할 주제를 글로 지정 가능 | "동시성 처리 방식에 집중해서 반박해줘" |
| `/codex:rescue` | 작업을 Codex에게 넘겨 조사·수정 시도 (`--resume`, `--fresh`, `--model`, `--effort`) | Claude가 같은 버그에서 막혔을 때 |
| `/codex:transfer` | 현재 Claude Code 세션을 Codex 스레드로 넘기고 `codex resume` 명령 출력 | Codex로 작업을 이어가고 싶을 때 |
| `/codex:status` / `/codex:result` / `/codex:cancel` | 백그라운드 작업 상태 확인, 결과 보기, 취소 | `--background`로 실행한 경우 |
| `/codex:setup` | 설치·인증 확인, 리뷰 게이트 켜기/끄기 | |

**리뷰 게이트 (선택, 실험적):** `/codex:setup --enable-review-gate`로 켜면, Claude가 응답을 마치려 할 때 `Stop` 훅이 Codex 리뷰를 실행하고 문제가 있으면 Claude가 계속 고치게 합니다. **두 에이전트가 오래 주고받으며 사용량을 빠르게 소모할 수 있으므로**, 지켜볼 수 있을 때만 켜세요. ([훅 개념](./12-mcp-and-hooks-concepts.md))

**EduHub 활용 흐름**

```text
> 수강 취소 기능 구현해줘 (Claude Code가 구현, 테스트 통과)
> /codex:review --background                    # Codex가 교차 리뷰
> /codex:adversarial-review 대기자 승격의 동시성 처리와 트랜잭션 경계를 집중적으로 반박해줘
> /codex:result                                 # 결과 확인
> Codex 리뷰 결과 중 타당한 것만 반영해줘. 반영하지 않는 항목은 이유를 정리해줘.
```

> 💡 리뷰 결과는 **"검증할 주장"**이지 명령이 아닙니다. 어느 모델의 지적이든 근거를 확인하고 반영 여부를 판단하세요.

---

### 방법 C. MCP로 연결

**C-1. Codex를 Claude Code의 MCP 도구로**

Codex CLI는 자신을 MCP 서버로 실행하는 기능을 제공합니다(실험적). Claude Code에 등록하면 Claude가 필요할 때 Codex에게 작업을 맡길 수 있습니다.

```bash
# 명령 이름은 버전에 따라 다를 수 있음 → codex --help로 확인
claude mcp add codex -- codex mcp-server
```

**C-2. Claude Code를 Codex의 MCP 도구로**

```bash
claude mcp serve        # Claude Code를 stdio MCP 서버로 실행
```

Codex의 MCP 설정에 위 명령을 서버로 등록하면 반대 방향도 가능합니다. (Codex 쪽 등록 방법은 Codex 공식 문서 확인)

**주의:** 에이전트가 다른 에이전트를 부르면 **권한과 비용을 추적하기 어려워집니다.** 샌드박스·승인 설정이 꺼진 상태로 실행되지 않는지 확인하고, 신뢰하는 저장소에서만 사용하세요. 대부분의 팀에는 방법 B가 더 단순합니다.

---

### 방법 D. PR 단계 교차 리뷰

```
Claude Code로 구현 → PR 생성
   ├─ Claude Code GitHub Action 자동 리뷰 (모듈 17)
   └─ Codex의 GitHub 코드 리뷰 (@codex 멘션 등, Codex 공식 문서 확인)
→ 사람이 두 리뷰를 참고해 최종 판단
```

- 도구 설치 없이 **팀 단위로 바로 적용** 가능
- 반대로 Codex가 만든 PR을 Claude Code로 리뷰할 수도 있다 (`/code-review <PR번호>`)

---

### 방법 E. 스크립트·CI 파이프라인

```bash
# 예시 개념: 구현은 Claude Code, 2차 리뷰는 Codex (명령 옵션은 각 도구 문서 확인)
claude -p "이슈 #42를 구현하고 ./scripts/verify.sh를 통과시켜줘" --permission-mode acceptEdits
codex exec "현재 브랜치의 변경을 리뷰하고 문제를 목록으로 출력해줘" > codex-review.md
claude -p "codex-review.md의 지적 중 타당한 것만 반영하고, 반영하지 않은 항목은 이유를 적어줘"
```

- 무인 실행이므로 권한을 최소로 ([권한](./23-permissions.md)), 비용 상한을 설정
- 결과는 반드시 사람이 PR에서 검토

---

## 3. 역할 분담 패턴

| 패턴 | 구현 | 검토 | 특징 |
|---|---|---|---|
| **작성자-검토자** | Claude Code | Codex (`/codex:review`) | 가장 단순하고 효과적 |
| **반론자** | Claude Code가 설계 | Codex가 반박 (`adversarial-review`) | 설계 단계 품질 향상 |
| **구조 요원** | Claude Code가 시도 | 막히면 Codex에 넘김 (`rescue`) | 디버깅 정체 해소 |
| **병렬 경쟁** | 같은 과제를 두 도구가 각자 워크트리에서 | 사람이 더 나은 결과 선택 | 비용 2배, 중요한 과제·도구 평가에 |
| **반대 방향** | Codex가 구현 | Claude Code가 리뷰 (서브에이전트 리뷰어 활용) | 팀원 선호에 따라 |

---

## 4. 주의 사항

| 주의 | 설명 | 대책 |
|---|---|---|
| **비용 증가** | 두 서비스의 사용량·구독 비용 | 교차 리뷰는 중요한 변경에만, 백그라운드 실행·결과만 확인 |
| **무한 루프** | 리뷰 게이트, 에이전트 간 상호 호출 | 지켜볼 때만 켜기, 반복 횟수·예산 제한 |
| **동시 수정 충돌** | 두 도구가 같은 작업 공간을 동시에 수정 | 리뷰는 읽기 전용으로, 구현은 워크트리 분리 |
| **지침 불일치** | `CLAUDE.md`와 `AGENTS.md` 내용이 달라짐 | 공통 내용은 `AGENTS.md` 한 곳에, `CLAUDE.md`는 가져오기 |
| **보안·정책** | 코드가 **두 회사의 서비스로** 전송됨 | 사내 보안 정책, 데이터 처리 조건을 두 서비스 모두 확인 |
| **판단 책임** | 두 AI의 의견이 다를 때 | 최종 판단은 사람. 근거(테스트, 재현)로 결정 |

---

## 5. 실습

1. 저장소에 `AGENTS.md`를 만들고 `CLAUDE.md`에서 `@AGENTS.md`로 가져오기 → 두 도구에서 같은 규칙이 적용되는지 확인
2. codex-plugin-cc를 설치하고, 수강 취소 기능 구현 후 `/codex:review`와 Claude Code의 `/code-review` 결과 비교
3. `/codex:adversarial-review`로 동시성 설계에 대한 반론을 받고, 타당한 지적만 반영
4. 일부러 어려운 버그를 Claude Code에 맡겼다가 막히면 `/codex:rescue`로 넘겨 결과 비교
5. 팀 토론: 우리 팀에서 교차 리뷰를 적용할 기준(어떤 PR에, 언제) 정하기
