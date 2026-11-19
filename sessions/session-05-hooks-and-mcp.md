# 5회차: 훅과 MCP — 품질 자동화와 외부 연결 (FR-07~08, 로그인과 화면)

🧭 ⬅️ 이전: [4회차: 권한·스킬·서브에이전트 — 수강 신청과 동시성 (FR-04~06, NFR-01)](./session-04-permissions-skills-subagents.md) · ➡️ 다음: [6회차: 플러그인·아티팩트 — 마이크로프론트엔드 완성과 알림(FR-09)](./session-06-plugins-artifacts-mfe.md) · 📚 [전체 커리큘럼](../CURRICULUM.md#-회차별-강의-자료)

> **대상**: 4회차까지 진행해 EduHub 수강 신청(FR-04~06)을 구현하고, 권한 설정·스킬·서브에이전트를 써 본 개발자
> **시간**: 120분 (강의 약 40% + 따라하기 약 60%)
> **목표**
> 1. **훅**이 CLAUDE.md·스킬과 어떻게 다른지("부탁"이 아니라 "강제") 설명하고, 이벤트·matcher·종료 코드로 동작을 설계할 수 있다
> 2. EduHub에 **품질 자동화 훅**(자동 포맷, 위험 명령·보호 파일 차단, 마무리 테스트)을 직접 만들어 적용한다
> 3. **MCP**를 프로젝트 범위(`.mcp.json`)로 연결하고, **개발 DB 읽기 전용 계정**으로만 붙이는 이유와 방법을 안다
> 4. MCP와 CLI(`gh`, `psql`)의 차이와 토큰 관점의 선택 기준을 설명할 수 있다
> 5. DB MCP로 스키마와 데이터를 확인하며 **인증(JWT)**, **FR-07(내 신청 내역)**, **FR-08(명단·수료 처리)**를 구현하고 PR로 병합한다
> 6. 3회차의 과정 목록 앱에 **로그인과 나머지 화면**(상세·신청, 내 신청 내역, 과정 등록·수정, 명단·수료)을 붙여 **하나의 React 앱**으로 완성한다 (6회차에 이 앱을 마이크로프론트엔드로 쪼갠다)
>
> **관련 모듈**: 12(훅), 13(커넥터와 MCP)
> **사전 준비**
> - 4회차 과제 PR 병합 완료, `main` 최신화
> - `jq` 설치 확인(`jq --version`). 윈도우는 Git Bash에서 실행되는지 확인 (`winget install jqlang.jq` 등)
> - Docker Desktop 실행, EduHub 개발 DB(PostgreSQL) 컨테이너가 뜨는지 확인
> - `uv` 설치 (`uvx --version`): DB MCP 서버(Postgres MCP Pro) 실행용. 설치 방법은 Step 8 ①
> - (선택) GitHub CLI `gh` 로그인 (`gh auth status`)
> - Node.js LTS, `frontend/` 의존성 설치(`npm install`), 3회차의 과정 목록 화면이 `npm run dev --workspace apps/course-mfe`로 뜨는지 확인
> 🖥️ **슬라이드 덱(46장, 발표자 노트 포함)**: https://claude.ai/artifact/SuXXVHg7iSzXiDetXP1LRV — 처음에는 만든 사람만 볼 수 있으며, 다른 사람에게 보여주려면 덱의 공유 메뉴에서 공유해야 합니다. PowerPoint(.pptx)·PDF로 내려받을 수 있습니다.

---

## 진행 순서

| 시간 | 내용 | 형식 |
|---|---|---|
| 0:00 – 0:05 | 지난 회차 복습 & 과제 리뷰 | 강의 |
| 0:05 – 0:22 | 1. 훅: 부탁이 아닌 강제 | 강의 + 시연 |
| 0:22 – 0:35 | 2. MCP: 외부 시스템과 연결하기 (2-5 제조 현장 사례 포함) | 강의 |
| 0:35 – 0:40 | 3. 1회차 토론 다시 하기: "OpenAPI 문서 누락을 어떻게 막을까?" | 토론 |
| 0:40 – 1:05 | 4. 따라하기 ①: EduHub 품질 자동화 훅 만들기 | 실습 |
| 1:05 – 1:12 | ☕ 휴식 | — |
| 1:12 – 1:40 | 5. 따라하기 ②: DB MCP 연결, 인증, FR-07~08 구현 | 실습 |
| 1:40 – 1:55 | 6. 따라하기 ③: 로그인과 화면 완성 — Step 13~14 (로그인) | 실습 |
| — | 6. 따라하기 ③ Step 15~16 (수강생·관리자 화면) | **과제** (6회차 전까지 필수) |
| 1:55 – 2:00 | 8. 정리, 퀴즈, 과제 안내 | 강의 |

> 7장(더 해 보기)은 시간이 남으면 수업 중에, 아니면 **과제**로 진행합니다.
>
> ⏱ 이번 회차는 분량이 많습니다. 시간이 모자라면 강사가 Step 14의 결과를 시연하고 Step 13~16을 모두 과제로 돌려도 됩니다. 6회차는 6장에서 완성한 앱을 마이크로프론트엔드로 쪼개는 것으로 시작하므로, **6장은 6회차 전까지 반드시 끝냅니다.**
>
> ⏱ 따라하기 ②에는 FR-07 전에 **인증 기반 만들기**([Step 9-1](#step-9-1-인증-기반-만들기-nfr-03--fr-07-전에), 약 15분)가 들어 있습니다. 시간이 부족하면 강사가 준비한 인증 브랜치를 병합하고 구조 설명과 확인(curl, 테스트)만 함께 합니다. 인증이 없으면 FR-07("내" 신청 내역)과 FR-08(관리자만)을 제대로 만들 수 없으므로 건너뛰지 않습니다.

---

## 지난 회차 복습 & 과제 리뷰

**4회차 핵심 3문장**
1. **권한(settings.json)**은 Claude가 할 수 있는 일의 **경계**를 정한다 → `.env`, 운영 설정 읽기는 `deny`
2. **스킬**은 반복되는 절차를 Claude가 **필요할 때 꺼내 쓰는 지식**으로 만든다
3. **서브에이전트**는 별도 컨텍스트에서 일하는 역할이다 → 구현과 리뷰를 나누면 메인 컨텍스트가 깨끗해진다

**과제 리뷰 (3분)**: 팀별 PR 하나를 화면에 띄워 확인합니다.

| 확인 항목 | 보는 것 |
|---|---|
| 동시성(NFR-01) | 동시 신청 테스트가 있고, 정원을 넘지 않는가? |
| PR 본문 | "Claude에게 맡긴 것 / 직접 한 것"이 적혀 있는가? |
| 리뷰 에이전트 | 리뷰 에이전트가 찾은 문제 중 실제로 고친 것은? 놓친 것은? |
| 회고 | `/usage` 기록과 토큰을 아낀 방법 하나 |

> 💬 **강사 노트:** 4회차 과제에서 "CLAUDE.md에 포맷 규칙을 적었는데 가끔 안 지켜졌다", "Claude가 `.env`를 읽으려다 권한에 막혔다" 같은 경험을 물어봅니다. 오늘 훅으로 "가끔"을 "항상"으로 바꾸는 이야기의 출발점이 됩니다.

---

# 1. 훅: 부탁이 아닌 강제

> 자세한 내용: [MCP와 훅 개념](../examples/12-mcp-and-hooks-concepts.md#part-2-훅-hooks), [설정·스킬·에이전트·훅 예제](../examples/03-settings-skills-agents-hooks.md#모듈-12-훅)

---

## 1-1. 왜 훅이 필요한가

| | CLAUDE.md · 스킬 | 권한 (settings.json) | **훅** |
|---|---|---|---|
| 성격 | Claude에게 **부탁** | 도구 사용의 **허용·차단 목록** | 정해진 시점에 **반드시 실행되는 명령** |
| 누가 실행 | 모델이 읽고 판단 | 하네스가 규칙과 비교 | **하네스가 항상 실행** |
| 잘하는 것 | 판단이 필요한 규칙, 작업 방법 | "이 도구·파일은 안 됨" | 기계적인 검사, 자동 작업, 맥락을 보는 차단 |
| 한계 | 가끔 놓친다 | 규칙 문법으로 표현할 수 있는 것만 | 스크립트를 직접 만들고 관리해야 한다 |
| EduHub 예 | "Java 코드는 Spotless 형식을 따른다" | `Read(./.env)` deny | 파일 수정마다 `spotlessApply` 실행 |

> **"반드시 지켜야 하고, 기계적으로 검사할 수 있으면 훅"** — 1회차 [8-1 판단 흐름](./session-01-introduction.md#8-1-판단-흐름)의 두 번째 질문입니다.

---

## 1-2. 훅은 언제 실행되나: 주요 이벤트

훅 이벤트는 세션 시작부터 종료까지 Claude Code의 여러 시점에 있습니다. 공식 문서에는 30개가 넘는 이벤트가 있지만, 실무에서 자주 쓰는 것은 아래 정도입니다.

```mermaid
flowchart LR
    SS["SessionStart<br/>세션 시작"] --> UP["UserPromptSubmit<br/>프롬프트 전송 직후"]
    UP --> PRE["PreToolUse<br/>도구 실행 직전"]
    PRE --> T["도구 실행<br/>(Edit, Bash, MCP …)"]
    T --> POST["PostToolUse<br/>도구 실행 직후"]
    POST -- "다음 도구" --> PRE
    POST --> ST["Stop<br/>응답을 마칠 때"]
    ST -- "다음 프롬프트" --> UP
    ST --> SE["SessionEnd<br/>세션 종료"]
```

| 이벤트 | 실행 시점 | EduHub에서의 용도 |
|---|---|---|
| `SessionStart` | 세션 시작·재개 | 현재 브랜치·진행 중인 이슈를 컨텍스트에 넣기, 클라우드에서 의존성 설치 |
| `UserPromptSubmit` | 프롬프트를 보낸 직후 | 프롬프트에 비밀번호·토큰 같은 문자열이 있으면 막기 |
| `PreToolUse` | 도구 실행 **직전** | 운영 접속 명령, 보호 파일 수정 **차단** |
| `PostToolUse` | 도구 실행 **직후** | 수정한 파일 자동 포맷, 규칙 검사 |
| `Stop` | Claude가 응답을 마칠 때 | 백엔드를 고쳤으면 테스트를 돌려 실패 시 계속 고치게 하기 |
| `Notification` | 승인 대기 등 알림 | 데스크톱·메신저 알림 |
| `PreCompact` | 컨텍스트 압축 직전 | 대화 기록 백업 |

> 이벤트 목록은 버전마다 늘어납니다. 전체 목록은 공식 문서 Hooks reference(https://code.claude.com/docs/en/hooks)에서 확인합니다.

---

## 1-3. 설정 구조와 입력·출력

**설정 위치**

| 파일 | 범위 | EduHub 사용 |
|---|---|---|
| `.claude/settings.json` | 프로젝트, **팀 공유 (커밋)** | ⭐ 팀 공용 훅 |
| `.claude/settings.local.json` | 프로젝트, 나만 (커밋 안 함) | 개인 알림 훅 |
| `~/.claude/settings.json` | 내 모든 프로젝트 | 개인 습관 훅 (예: 테스트 출력 줄이기) |

**설정 모양**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/format.sh\"", "timeout": 60 }
        ]
      }
    ]
  }
}
```

| 항목 | 의미 |
|---|---|
| `matcher` | 어떤 도구에 반응할지. `Edit\|Write`처럼 글자·`\|`만 쓰면 **정확히 일치**, 다른 기호가 들어가면 정규식 (예: `mcp__eduhub-db__.*`) |
| `type: "command"` | 셸 명령 실행 |
| `$CLAUDE_PROJECT_DIR` | 세션을 시작한 프로젝트 루트. 어느 폴더에서 실행돼도 스크립트를 찾게 한다 |
| `timeout` | 초 단위 제한 시간 |

**Windows에서의 설정 모양**

Windows(WSL이 아닌 네이티브)에서는 훅 명령을 실행하는 셸이 다릅니다. 공식 문서 기준으로 `command` 문자열은 macOS·Linux에서는 `sh -c`, **Windows에서는 Git Bash**, Git Bash가 없으면 PowerShell로 실행됩니다.

| 상황 | 권장 방식 | 스크립트 |
|---|---|---|
| **Git for Windows 설치** (2회차 준비물, ⭐ 교육 기본) | 위 설정을 **그대로** 사용 (Git Bash가 `bash "…/format.sh"`를 실행) | 팀 공용 `.sh` 하나로 macOS·Linux·Windows 모두 사용 |
| Git Bash 없이 PowerShell만 사용 | `powershell.exe`를 직접 실행하는 형식(exec form) | `.ps1` 스크립트 |
| PowerShell 스크립트를 셸 문자열로 실행 | 훅에 `"shell": "powershell"` 지정 | `.ps1` 스크립트 |

**① Git for Windows가 있을 때 (권장)**: JSON은 위와 같습니다. Windows에서 추가로 챙길 것만 있습니다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/format.sh\"", "timeout": 60 }
        ]
      }
    ]
  }
}
```

- `jq` 설치: `winget install jqlang.jq` (설치 후 Git Bash에서 `jq --version`으로 확인)
- 스크립트 줄바꿈은 **LF**로 저장: `.gitattributes`에 `*.sh text eol=lf`
- **경로는 `\`로 들어옵니다.** Git Bash에서 실행돼도 `tool_input.file_path`는 `C:\work\eduhub\.env`처럼 역슬래시입니다. 경로 패턴을 검사하는 스크립트는 맨 앞에서 `/`로 바꿉니다: `FILE="${FILE//\\//}"` (Step 3, 1-4의 스크립트에 넣어 두었습니다)
- Claude Code가 Git Bash를 못 찾으면 `settings.json`의 `env`에 `"CLAUDE_CODE_GIT_BASH_PATH": "C:\\Program Files\\Git\\bin\\bash.exe"`

**② PowerShell 스크립트를 직접 실행 (exec form, `args` 사용)**: `args`가 있으면 셸 없이 `command`를 실행 파일로 바로 실행하고, `${CLAUDE_PROJECT_DIR}` 자리 표시자는 `command`와 `args` 안에서 그대로 바뀝니다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe",
            "args": ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "${CLAUDE_PROJECT_DIR}/.claude/hooks/format.ps1"],
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

**③ 셸 문자열을 PowerShell로 실행 (`shell: "powershell"`)**

```json
{ "type": "command", "shell": "powershell", "command": "& \"$env:CLAUDE_PROJECT_DIR\\.claude\\hooks\\format.ps1\"" }
```

| 주의 | 설명 |
|---|---|
| PowerShell에서 `$CLAUDE_PROJECT_DIR` 금지 | PowerShell은 이것을 정의되지 않은 변수로 보고 빈 값이 된다. `$env:CLAUDE_PROJECT_DIR` 또는 `${CLAUDE_PROJECT_DIR}`로 쓴다 |
| `.ps1`은 앞에 `&` | 셸 형식에서 경로에 변수가 들어간 스크립트를 실행하려면 `& "경로"` |
| `-ExecutionPolicy Bypass` | 로컬 `.ps1` 실행을 막는 실행 정책을 이 실행에만 풀어 준다 |
| 종료 코드 | 계약은 같다: stdin으로 JSON, `exit 2`면 차단하고 stderr를 전달. Git Bash가 없을 때는 **0과 2 외의 종료 코드는 실패**로 처리된다 |
| 한글 메시지 | Windows PowerShell 5.1은 인코딩 때문에 한글이 깨질 수 있다. PowerShell 7(`pwsh`) 사용, 스크립트는 UTF-8로 저장 |

> 💡 **팀 설정이 섞일 때**: `.claude/settings.json`은 팀 전체가 같이 씁니다. macOS·Linux 팀원이 있으면 **모두 Git Bash + `.sh`**로 통일하는 것이 가장 단순합니다. Windows 전용 PowerShell 훅이 꼭 필요하면 개인 설정 `.claude/settings.local.json`에 둡니다.

> 출처: Claude Code 공식 문서 Hooks reference, Hooks guide, Setup (2026년 11월 확인). ②③의 PowerShell 형식은 문서 예시를 따른 것으로, 이 교재에서는 Windows에서 직접 실행해 보지 못했습니다. 수업 전에 한 번 확인하세요.

**입력과 출력**

```mermaid
sequenceDiagram
    participant C as Claude Code (하네스)
    participant H as 훅 스크립트
    C->>H: stdin으로 JSON 전달<br/>(hook_event_name, tool_name, tool_input …)
    alt 종료 코드 0
        H-->>C: 정상 진행 (stdout JSON으로 세부 지시 가능)
    else 종료 코드 2
        H-->>C: 차단·피드백 (stderr 메시지를 Claude에게 전달)
    else 그 외 종료 코드
        H-->>C: 훅 오류로 표시하고 진행 (차단 아님)
    end
```

**종료 코드 2의 의미는 이벤트마다 다릅니다**

| 이벤트 | 종료 코드 2일 때 |
|---|---|
| `PreToolUse` | 도구 실행을 **막고**, 이유를 Claude에게 전달 → Claude가 다른 방법을 찾는다 |
| `PostToolUse` | 이미 실행된 뒤라 막지는 못하고, 메시지를 Claude에게 전달 → Claude가 고친다 |
| `UserPromptSubmit` | 프롬프트를 **막아** Claude에게 전달되지 않는다 |
| `Stop` | Claude가 멈추지 않고 **계속 작업**한다 |

> 💡 더 세밀하게 제어하려면 종료 코드 0으로 끝내면서 stdout에 JSON을 출력합니다. 예: `Stop` 훅에서 `{"decision": "block", "reason": "테스트 실패. 고친 뒤 끝내세요"}`. 대화 중 `/hooks`로 지금 적용된 훅과 그 출처를 볼 수 있습니다.

---

## 1-4. 시연: 훅이 막고, Claude가 길을 바꾼다 (강사 시연 3분)

**시연에 쓰는 훅 코드** (Step 3의 `protect-files.sh`에서 운영 설정 부분만 떼어 낸 최소 버전)

```bash
#!/usr/bin/env bash
# .claude/hooks/protect-prod-config.sh — PreToolUse(Edit|Write): 운영 설정 파일 수정 차단 (1-4 시연용)
# jq가 없으면 검사할 수 없으므로 통과시키지 않고 막는다 (그냥 두면 보호가 조용히 꺼진다)
command -v jq >/dev/null 2>&1 || { echo "jq가 없어 보호 훅이 검사할 수 없습니다. 사람이 jq를 설치하세요 (Windows: winget install jqlang.jq)." >&2; exit 2; }
FILE=$(jq -r '.tool_input.file_path // empty')
FILE="${FILE//\\//}"   # Windows 경로의 \ 를 / 로 바꿔 아래 패턴이 맞게 한다

if [[ "$FILE" == *application-prod.yml ]]; then
  echo "운영 설정(application-prod.yml)은 사람이 PR로만 수정합니다. 로컬 개발용 변경이면 backend/src/main/resources/application-dev.yml을 수정하세요." >&2
  exit 2
fi
exit 0
```

`.claude/settings.json` (macOS·Linux, Windows + Git Bash)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/protect-prod-config.sh\"" }
        ]
      }
    ]
  }
}
```

<details>
<summary>Windows PowerShell 버전 (Git Bash를 쓰지 않을 때)</summary>

```powershell
# .claude/hooks/protect-prod-config.ps1 — PreToolUse(Edit|Write): 운영 설정 파일 수정 차단 (1-4 시연용)
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8   # 한글 메시지가 깨지지 않게

try {
  $hookInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
} catch {
  exit 0   # 입력을 읽지 못하면 막지 않는다
}
$file = [string]$hookInput.tool_input.file_path

if ($file -match 'application-prod\.yml$') {
  [Console]::Error.WriteLine('운영 설정(application-prod.yml)은 사람이 PR로만 수정합니다. 로컬 개발용 변경이면 backend/src/main/resources/application-dev.yml을 수정하세요.')
  exit 2
}
exit 0
```

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe",
            "args": ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "${CLAUDE_PROJECT_DIR}/.claude/hooks/protect-prod-config.ps1"]
          }
        ]
      }
    ]
  }
}
```

> PowerShell 버전은 공식 문서의 PowerShell 훅 예시 형식을 따랐고, Windows에서 직접 실행해 보지는 못했습니다. 한글이 깨지면 PowerShell 7(`pwsh.exe`)을 쓰고 스크립트를 UTF-8로 저장합니다.

</details>

**Windows 시연용 전체 보호 훅: `protect-files.ps1`**

시연을 Step 3의 **전체 보호 규칙**(운영 설정, `.env`, 생성 코드, 기존 마이그레이션)으로 하려면 아래 PowerShell 스크립트를 씁니다. Step 3의 `protect-files.sh`와 **같은 규칙, 같은 메시지**입니다. Git Bash가 있는 Windows라면 `.sh`를 그대로 써도 됩니다 (1-3 Windows 설정 ①).

```powershell
# .claude/hooks/protect-files.ps1 — PreToolUse(Edit|Write): 보호 파일 수정 차단 (Windows PowerShell 버전)
# Step 3의 protect-files.sh와 같은 규칙. 막을 때는 "이유 + 대신 할 일"을 stderr로 쓰고 exit 2

# 한글 메시지가 깨지지 않도록 콘솔 출력 인코딩을 UTF-8로
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

function Block([string]$message) {
    [Console]::Error.WriteLine($message)
    exit 2
}

# 1) stdin으로 들어온 훅 입력(JSON)에서 수정하려는 파일 경로를 꺼낸다
try {
    $hookInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
} catch {
    exit 0   # 입력을 읽지 못하면 막지 않는다 (훅 오류로 모든 작업이 멈추지 않게)
}
$file = [string]$hookInput.tool_input.file_path
if ([string]::IsNullOrEmpty($file)) { exit 0 }

# 2) 경로 구분자를 / 로 통일 (C:\work\eduhub\.env → C:/work/eduhub/.env)
$path = $file -replace '\\', '/'

# 3) 보호 규칙 (위에서부터 검사. -match는 대소문자를 구분하지 않는다)
if ($path -match 'application-prod\.yml$') {
    Block '운영 설정은 사람이 PR로만 수정합니다. 개발 설정은 application-dev.yml을 수정하세요.'
}
if ($path -match '(^|/)\.env\.example$') {
    exit 0   # .env.example은 변수 이름만 적는 파일이라 허용
}
if ($path -match '(^|/)\.env(\..+)?$') {
    Block '비밀 파일은 수정하지 않습니다. 필요한 변수 이름은 .env.example에 추가하세요.'
}
if ($path -match '/api-client/src/generated/') {
    Block "자동 생성 코드입니다. 'npm run generate:api'로 다시 생성하세요."
}
if ($path -match '/db/migration/V[^/]*\.sql$' -and (Test-Path -LiteralPath $file)) {
    Block '기존 Flyway 마이그레이션은 수정할 수 없습니다. 다음 버전 번호로 새 파일을 추가하세요.'
}
exit 0
```

`.claude/settings.json` (또는 Windows 전용이면 `.claude/settings.local.json`)

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe",
            "args": ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "${CLAUDE_PROJECT_DIR}/.claude/hooks/protect-files.ps1"]
          }
        ]
      }
    ]
  }
}
```

| `protect-files.sh`와 다른 점 | 설명 |
|---|---|
| 입력 읽기 | `jq` 대신 `[Console]::In.ReadToEnd() \| ConvertFrom-Json` (jq 설치 불필요) |
| 경로 구분자 | `$file -replace '\\', '/'`로 `\`를 `/`로 바꾼 뒤 검사 |
| 대소문자 | `-match`는 대소문자를 구분하지 않는다 (Windows 파일 시스템과 같음). bash 버전은 구분한다 |
| 입력이 깨졌을 때 | 막지 않고 `exit 0` (훅 오류 때문에 모든 수정이 멈추지 않게) |
| 한글 | `[Console]::OutputEncoding`을 UTF-8로. **Windows PowerShell 5.1은 스크립트 파일을 "UTF-8 (BOM)"으로 저장**해야 한글 문자열을 바르게 읽는다 (VS Code 오른쪽 아래 인코딩 → "Save with Encoding" → UTF-8 with BOM). PowerShell 7(`pwsh.exe`)은 BOM 없이도 된다 |

**PowerShell로 직접 테스트** (프로젝트 루트에서, Claude 없이)

```powershell
$hook = ".claude\hooks\protect-files.ps1"
function Test-Hook($path) {
  $json = @{ tool_input = @{ file_path = $path } } | ConvertTo-Json -Compress
  $json | powershell.exe -NoProfile -ExecutionPolicy Bypass -File $hook
  "exit=$LASTEXITCODE  ($path)"
}
Test-Hook "C:\work\eduhub\backend\src\main\resources\application-prod.yml"   # 메시지 + exit=2
Test-Hook "C:\work\eduhub\.env"                                                   # 메시지 + exit=2
Test-Hook "C:\work\eduhub\.env.example"                                           # exit=0
Test-Hook "C:\work\eduhub\frontend\packages\api-client\src\generated\api.ts"     # 메시지 + exit=2
Test-Hook "C:\work\eduhub\backend\src\main\java\A.java"                           # exit=0
```

- 마이그레이션 규칙은 **실제로 있는 파일**일 때만 막습니다(`Test-Path`). 저장소의 기존 `V1__…sql` 경로로 시험해 exit=2를 확인합니다
- `ConvertTo-Json`으로 입력을 만들면 경로의 `\`가 JSON에 맞게 자동으로 바뀌어 손으로 이스케이프할 필요가 없습니다

> ⚠️ 이 교재 환경에는 PowerShell이 없어 **스크립트를 직접 실행해 보지 못했습니다.** 규칙 판별(정규식 5개)만 같은 규칙으로 10가지 경로(Windows·Linux 경로, `.env`/`.env.local`/`.env.example`, 생성 코드, 기존·새 마이그레이션, 일반 파일)에 대해 검증했습니다. 시연 전에 위 테스트를 Windows에서 한 번 실행해 주세요.

**시연 전에 직접 테스트** (Claude 없이 훅만 실행)

```bash
# macOS·Linux·Git Bash
echo '{"tool_input":{"file_path":"/x/backend/src/main/resources/application-prod.yml"}}' \
  | bash .claude/hooks/protect-prod-config.sh; echo "exit=$?"     # 메시지 출력 + exit=2
echo '{"tool_input":{"file_path":"C:\\work\\eduhub\\backend\\src\\main\\resources\\application-prod.yml"}}' \
  | bash .claude/hooks/protect-prod-config.sh; echo "exit=$?"     # Windows 경로도 exit=2
echo '{"tool_input":{"file_path":"/x/backend/src/main/resources/application-dev.yml"}}' \
  | bash .claude/hooks/protect-prod-config.sh; echo "exit=$?"     # exit=0
```

**Windows PowerShell에서 테스트** (저장소 루트에서. 위 bash 명령은 줄 끝 `\`, `$?` 때문에 PowerShell·cmd에서는 문법 오류가 납니다)

```powershell
# ① bash 스크립트(.sh)를 Git Bash로 실행해서 테스트 (Git for Windows 설치 시, 교육 기본)
$bash = "C:\Program Files\Git\bin\bash.exe"
'{"tool_input":{"file_path":"/x/backend/src/main/resources/application-prod.yml"}}' | & $bash .claude/hooks/protect-prod-config.sh; "exit=$LASTEXITCODE"   # 메시지 + exit=2
'{"tool_input":{"file_path":"C:\\work\\eduhub\\backend\\src\\main\\resources\\application-prod.yml"}}' | & $bash .claude/hooks/protect-prod-config.sh; "exit=$LASTEXITCODE"   # exit=2
'{"tool_input":{"file_path":"/x/backend/src/main/resources/application-dev.yml"}}' | & $bash .claude/hooks/protect-prod-config.sh; "exit=$LASTEXITCODE"   # exit=0

# ② PowerShell 스크립트(.ps1)를 테스트 (Git Bash 없이 PowerShell 훅을 쓸 때)
$ps1 = ".claude\hooks\protect-prod-config.ps1"
'{"tool_input":{"file_path":"/x/backend/src/main/resources/application-prod.yml"}}' | powershell.exe -NoProfile -ExecutionPolicy Bypass -File $ps1; "exit=$LASTEXITCODE"   # 메시지 + exit=2
'{"tool_input":{"file_path":"C:\\work\\eduhub\\backend\\src\\main\\resources\\application-prod.yml"}}' | powershell.exe -NoProfile -ExecutionPolicy Bypass -File $ps1; "exit=$LASTEXITCODE"   # exit=2
'{"tool_input":{"file_path":"/x/backend/src/main/resources/application-dev.yml"}}' | powershell.exe -NoProfile -ExecutionPolicy Bypass -File $ps1; "exit=$LASTEXITCODE"   # exit=0
```

| bash | PowerShell | 이유 |
|---|---|---|
| `echo '…' \| …` | `'…' \| …` | PowerShell은 문자열만 써도 파이프로 넘어간다 |
| 줄 끝 `\`로 이어 쓰기 | 한 줄로 쓰거나 줄 끝을 `\|`로 끝내기 | `\`는 PowerShell의 줄 이음이 아니다 |
| `echo "exit=$?"` | `"exit=$LASTEXITCODE"` | PowerShell의 `$?`는 True/False. 외부 프로그램의 종료 코드는 `$LASTEXITCODE` |
| `bash …` | `& $bash …` (Git Bash 경로 지정) | PowerShell에서 그냥 `bash`는 WSL의 bash를 실행할 수 있다 (그 안에는 jq가 없을 수 있음) |
| JSON 안의 `C:\\work` | 그대로 `C:\\work` | JSON 문법상 `\`는 `\\`로 써야 하고, 작은따옴표 문자열은 PowerShell이 바꾸지 않는다 |

> cmd(명령 프롬프트)는 작은따옴표를 문자열로 인식하지 않아 위 명령을 그대로 쓸 수 없습니다. Windows에서는 Git Bash 창이나 PowerShell을 씁니다.

> ✅ bash 버전은 위 세 테스트(Linux 경로 차단, Windows 경로 차단, 개발 설정 통과)를 실행해 결과를 확인했습니다. PowerShell 명령은 이 교재 환경에 PowerShell이 없어 실행해 보지 못했으니 시연 전에 한 번 실행해 주세요.

**시연 순서**

1. 위 보호 훅이 등록된 상태에서 Claude에게 `application-prod.yml의 DB URL을 localhost로 바꿔줘`라고 지시합니다
2. `PreToolUse` 훅이 수정을 막고 이유를 전달합니다
3. Claude가 "운영 설정은 수정할 수 없으니 `application-dev.yml`을 바꾸겠다"고 **스스로 방향을 바꾸는** 모습을 보여 줍니다
4. `/hooks`로 등록된 훅과 그 출처(`.claude/settings.json`)를 보여 줍니다

> 💬 **강사 노트:** 훅은 Claude를 "혼내는" 장치가 아니라 **즉각적인 피드백**입니다. 이유를 친절하게 적어 주면 Claude가 올바른 대안을 찾습니다. 메시지를 "금지"로만 쓰지 말고 **"대신 무엇을 하라"**까지 쓰게 합니다.

---

## 1-5. 훅의 보안과 운영 원칙

| 원칙 | 이유와 방법 |
|---|---|
| **훅은 내 권한으로 실행되는 프로그램** | 팀원이 커밋한 훅, 플러그인에 들어 있는 훅도 내 PC에서 내 권한으로 돈다. 내용을 확인하고 쓴다 |
| **훅 변경도 코드 리뷰** | `.claude/`를 CODEOWNERS로 지정해 담당자 리뷰를 받는다 ([저장소 보호 설정](../examples/06-github-version-control.md#2-4-저장소-보호-설정)) |
| **빠르게** | `PreToolUse`·`PostToolUse`는 도구 호출마다 실행된다. 포맷은 수정한 파일만, 전체 테스트는 `Stop`에서 |
| **입력은 신뢰하지 않는다** | `tool_input`의 경로·명령은 따옴표로 감싸고, 예상 밖 값에도 안전하게 끝나게 작성 |
| **차단 훅은 최선의 노력** | 셸 명령은 변형이 무궁무진해 문자열 검사로 100% 막을 수 없다 → **권한 deny + 훅 + 운영 비밀을 개발 PC에 두지 않기**를 함께 쓴다 |
| **실패해도 작업이 멈추지 않게** | 포맷터가 없으면 조용히 넘어가도록(`exit 0`) 만들고, 꼭 막아야 하는 것만 `exit 2` |

> ⚠️ 가장 확실한 환경 분리는 **훅이 아니라 구조**입니다. 운영 DB 접속 정보가 개발 PC에 없으면 훅이 놓쳐도 접속할 수 없습니다 ([개발자 책임 원칙 3](../examples/27-developer-responsibility.md#원칙-3-ai로-유지보수할-때는-운영-환경과-개발테스트-환경을-반드시-분리한다)).

---

# 2. MCP: 외부 시스템과 연결하기

> 자세한 내용: [MCP 서버 완전 가이드](../examples/19-mcp-server-guide.md), [MCP와 훅 개념](../examples/12-mcp-and-hooks-concepts.md)

---

## 2-1. MCP 다시 보기

**MCP(Model Context Protocol)**는 AI 에이전트가 외부 시스템(DB, 이슈 트래커, 브라우저, 사내 API)의 기능을 **도구**로 쓸 수 있게 하는 표준입니다. 1회차에서 "에이전트용 USB-C"라고 소개했습니다.

```mermaid
flowchart LR
    subgraph CC["Claude Code"]
        M["Claude 모델"] <--> H["하네스<br/>(MCP 클라이언트)"]
    end
    H <-- "stdio" --> DB["eduhub-db<br/>MCP 서버 (로컬)"]
    H <-- "HTTP" --> GH["GitHub<br/>MCP 서버 (원격)"]
    H <-- "stdio" --> PW["Playwright<br/>MCP 서버 (로컬)"]
    DB --> PG[("개발 PostgreSQL<br/>읽기 전용 계정")]
    GH --> GHA["GitHub API"]
    PW --> BR["브라우저<br/>(localhost:3000)"]
```

| 서버가 제공하는 것 | 쓰는 법 | 예 |
|---|---|---|
| **도구(Tools)** | 자연어로 요청하면 Claude가 골라 호출 | "enrollment 테이블 상태별 건수 조회해줘" |
| **리소스(Resources)** | `@서버:주소`로 참조 | `@github:issue://42` |
| **프롬프트(Prompts)** | 슬래시 명령으로 실행 | `/mcp__github__pr_review 12` |

---

## 2-2. 연결 범위와 `.mcp.json`

| 범위 | 저장 위치 | 공유 | 쓰는 경우 |
|---|---|---|---|
| `local` (기본) | `~/.claude.json` (프로젝트별) | ✕ | 개인 실험, 개인 토큰이 필요한 서버 |
| `project` | 프로젝트 루트 **`.mcp.json`** | ○ (Git) | ⭐ **팀 공용 서버** |
| `user` | `~/.claude.json` | ✕ | 모든 프로젝트에서 쓰는 개인 서버 |

**`.mcp.json`은 어디에 두나요?** 저장소 루트, 즉 `CLAUDE.md`와 `.claude/` 폴더가 있는 **같은 위치**에 둡니다. `.claude/` 폴더 **안이 아닙니다.**

```text
eduhub/                       ← 저장소 루트 (Claude Code를 시작하는 폴더)
├── .mcp.json                 ← ⭐ 여기 (project 범위 MCP 서버, Git에 커밋)
├── CLAUDE.md
├── .claude/
│   ├── settings.json         ← 권한·훅 (MCP 서버 목록은 여기가 아니다)
│   ├── settings.local.json
│   └── hooks/
├── backend/
└── frontend/
```

| 확인할 것 | 설명 |
|---|---|
| 파일 이름 | 점으로 시작하는 `.mcp.json`. Windows 탐색기에서 만들면 `.mcp.json.txt`가 되기 쉬우니 VS Code나 `claude mcp add --scope project …`로 만든다 |
| 잘못된 위치 | `.claude/.mcp.json`, `backend/.mcp.json`처럼 다른 폴더에 두면 서버가 등록되지 않는다 |
| local·user 범위 | `.mcp.json`이 아니라 홈 폴더의 `~/.claude.json`에 저장된다 (Windows: `C:\Users\<사용자 이름>\.claude.json`) |
| `claude mcp add` | 범위 옵션이 없으면 **local**로 저장된다. `.mcp.json`에 넣으려면 `--scope project` |
| 반영 | 파일을 고친 뒤 Claude Code를 다시 시작하고 `/mcp`에서 확인. 처음 쓸 때 사용 승인 질문이 나온다 |

```bash
# 저장소 루트에서 위치 확인 (macOS·Linux·Git Bash)
ls -a | grep mcp          # .mcp.json
```

```powershell
# Windows PowerShell
Get-ChildItem -Force -Filter .mcp.json   # 숨김 파일까지 표시
```

```json
{
  "mcpServers": {
    "eduhub-db": {
      "type": "stdio",
      "command": "uvx",
      "args": ["postgres-mcp", "--access-mode=restricted"],
      "env": {
        "DATABASE_URI": "${EDUHUB_DB_READONLY_URL}"
      }
    }
  }
}
```

- 위 예시는 **Postgres MCP Pro**(`postgres-mcp`)를 씁니다. 고른 이유와 다른 선택지는 [Step 8](#step-8-mcpjson에-db-mcp-연결하기)에 있습니다

**`DATABASE_URI`에 들어가는 값의 예** (`.mcp.json`은 JSON이라 주석을 쓸 수 없으므로, 값은 셸 환경 변수에 두고 설명은 여기에 적습니다)

```bash
# ~/.zshrc, ~/.bashrc 또는 Windows 사용자 환경 변수 (.mcp.json에는 ${EDUHUB_DB_READONLY_URL} 글자만 둔다)

# ① PostgreSQL — 위 예시 서버(postgres-mcp)가 읽는 형식
#    postgresql://<읽기 전용 계정>:<비밀번호>@<호스트>:<포트>/<DB 이름>
export EDUHUB_DB_READONLY_URL="postgresql://eduhub_readonly:각자-정한-비밀번호@localhost:5432/eduhub"

# ② 개발 DB가 MariaDB라면
#    - postgres-mcp는 PostgreSQL 전용이라 MariaDB 주소를 넣어도 연결되지 않는다 → MariaDB용 서버(mcp-server-mysql)를 쓴다
#    - MariaDB 주소를 URI로 쓰면 이런 모양이다 (다른 도구·문서에서 참고용):
#        mariadb://eduhub_readonly:각자-정한-비밀번호@127.0.0.1:3306/eduhub
#        JDBC(Spring): jdbc:mariadb://127.0.0.1:3306/eduhub  (계정·비밀번호는 따로)
#    - 하지만 mcp-server-mysql의 MYSQL_CONNECTION_STRING은 URI가 아니라 "mysql 명령줄 형식"이다
export EDUHUB_MARIADB_CONN="--host=127.0.0.1 --port=3306 --user=eduhub_readonly --password=각자-정한-비밀번호 eduhub"
#    같은 뜻의 짧은 형식: -h 127.0.0.1 -P 3306 -u eduhub_readonly -p각자-정한-비밀번호 eduhub
#    비밀번호에 공백·특수문자가 있으면 작은따옴표로: --password='p w#1'
```

MariaDB일 때의 `.mcp.json`

```json
{
  "mcpServers": {
    "eduhub-db": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@benborla29/mcp-server-mysql"],
      "env": {
        "MYSQL_CONNECTION_STRING": "${EDUHUB_MARIADB_CONN}"
      }
    }
  }
}
```

> ⚠️ **MariaDB URI를 `MYSQL_CONNECTION_STRING`에 넣으면 오류 없이 잘못 연결됩니다.** 이 서버(v2.0.9)의 해석 코드를 직접 실행해 보니, `mariadb://eduhub_readonly:…@127.0.0.1:3306/eduhub`를 넣으면 **문자열 전체를 DB 이름으로** 읽고 호스트·계정은 기본값(`127.0.0.1`, **`root`**)을 씁니다. 읽기 전용 계정이 아니라 root로 접속을 시도하게 되므로 반드시 위의 명령줄 형식을 씁니다. 명령줄 형식은 `--host`, `--port`, `--user`, `--password`, `--socket`과 짧은 형식 `-h -P -u -p -S`, 마지막의 DB 이름을 읽습니다.

> 💡 접속 문자열 하나 대신 `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_USER`, `MYSQL_PASS`, `MYSQL_DB`로 나눠 넣는 방법은 [Step 8의 MariaDB 버전](#개발-db가-mariadb인-팀-step-78의-mariadb-버전)에 있습니다. 둘 다 있으면 접속 문자열 쪽 값이 우선합니다.

- `${VAR}`, `${VAR:-기본값}`으로 **환경 변수**를 넣습니다. `.mcp.json`은 커밋되므로 **비밀번호를 파일에 쓰지 않습니다**
- 팀원이 `.mcp.json`의 서버를 처음 쓸 때 **사용 승인 질문**이 나옵니다 (모르는 서버가 몰래 실행되지 않게 하는 장치)
- 대화 중 `/mcp`로 연결 상태 확인, 인증, 재연결, 끄기·켜기를 합니다
- 터미널에서는 `claude mcp add --scope project …`, `claude mcp list`, `claude mcp get <이름>`으로 관리합니다

> ⚠️ PostgreSQL MCP 서버는 여러 구현이 있고, 초기의 공식 참조 서버 `@modelcontextprotocol/server-postgres`는 **2025년 5월에 보관(archived)되어 관리가 끝났습니다.** 예전 자료에 많이 나오지만 새로 쓰지 않습니다. 어떤 서버를 쓰든 출처와 유지보수 상태(최근 릴리스, 이슈)를 확인하고 고릅니다.

---

## 2-3. DB MCP는 "개발 DB + 읽기 전용 계정"으로만

```mermaid
flowchart LR
    subgraph OK["✅ 허용"]
        A["Claude Code"] --> M["eduhub-db MCP"] --> R["eduhub_readonly 계정<br/>SELECT만"] --> D[("개발 DB<br/>localhost Docker<br/>가짜 데이터")]
    end
    subgraph NO["⛔ 금지"]
        X["Claude Code"] -.-x P[("운영 DB<br/>실제 개인정보")]
    end
```

| 지켜야 할 것 | 이유 |
|---|---|
| **개발 DB에만** 연결 | 운영 DB에는 실제 사용자 개인정보가 있다. 조회 결과가 AI 대화로 들어가는 것 자체가 정책 위반일 수 있다 |
| **읽기 전용 계정** | MCP 도구가 `DELETE`·`UPDATE`를 실행할 수 있으면, 잘못된 판단이나 프롬프트 인젝션 한 번이 데이터 손실이 된다 |
| 데이터 변경은 **Flyway 마이그레이션과 코드로** | 변경 이력이 Git에 남고, 리뷰와 테스트를 거친다 |
| 접속 정보는 **환경 변수**로 | `.mcp.json`과 저장소에는 변수 이름만 |
| 조회 결과는 **작게** | 큰 결과는 그대로 컨텍스트를 채운다. "상위 10건", "건수만"처럼 범위를 좁혀 요청 |

> 💬 **강사 노트:** "운영 DB를 봐야 원인을 알 수 있는 장애는요?"라는 질문이 나옵니다. 그런 조회는 **사람이** 운영 절차(접근 승인, 감사 로그)에 따라 하고, 필요한 결과만 **마스킹해서** Claude에게 줍니다.

---

## 2-4. MCP와 CLI, 무엇을 쓸까

Claude Code는 Bash로 `gh`, `psql` 같은 CLI를 바로 쓸 수 있습니다. MCP가 항상 더 나은 것은 아닙니다.

| 기준 | CLI (`gh`, `psql`) | MCP 서버 |
|---|---|---|
| 준비 | 이미 설치·로그인된 도구 그대로 | 서버 설정, 승인 필요 |
| 토큰 | **도구 정의가 추가되지 않는다.** 출력은 명령 옵션으로 줄이기 쉽다 (`--json`, `--limit`) | 도구 이름이 세션에 등록된다. 도구 검색(기본 켜짐) 덕분에 상세 정의는 필요할 때만 로드되지만, 결과가 크면 컨텍스트를 많이 쓴다 |
| 권한 제어 | `Bash(gh pr view *)`처럼 명령 패턴으로 | `mcp__서버__도구` 단위로 허용·차단 |
| 인증 | CLI의 로그인 상태 | 서버가 OAuth·토큰 처리 |
| 유리한 경우 | 자주 쓰는 개발 도구, 스크립트로 충분한 일 | CLI가 없는 SaaS, OAuth가 필요한 서비스, 여러 AI 도구에서 같은 연동을 재사용, 조직이 허용 서버를 관리 |

**EduHub의 선택**

| 대상 | 선택 | 이유 |
|---|---|---|
| GitHub (PR, 이슈) | **`gh` CLI** | 이미 설치·로그인되어 있고, 토큰을 덜 쓴다. 데스크톱 앱의 PR 기능도 `gh`를 쓴다 |
| 개발 DB 조회 | **DB MCP** (읽기 전용) | 스키마·데이터 조회를 도구 단위로 허용하고, 쓰기 자체가 불가능한 계정으로 묶을 수 있다 |
| 화면 확인 (6주차) | Playwright MCP | 브라우저 조작은 CLI로 하기 어렵다 |

> 💡 공식 비용 관리 문서도 **"가능하면 CLI, 쓰지 않는 MCP 서버는 `/mcp`에서 끄기"**를 권합니다 ([토큰 최적화](../examples/08-token-optimization.md)).

---

## 2-5. 사례: 제조 현장의 AI 에이전트와 MCP (MES·ERP 연결)

MCP는 Claude만의 기능이 아니라 **공개 표준**입니다. 한 번 만든 MCP 서버는 Claude, 사내 챗봇, 자동화 작업 등 **MCP를 지원하는 여러 에이전트가 함께 씁니다.** 제조 현장에서 이것이 어떤 서비스가 되는지 봅니다. 자세한 아키텍처와 **Java 구현 코드(실행 검증)**는 [제조 현장의 AI 에이전트와 MCP](../examples/34-manufacturing-ai-agent-mcp.md)에 있습니다.

**흩어진 현장 정보 → 도구 → 서비스**

| 시스템 | 도구 예 (읽기 전용) | 이 도구로 만드는 서비스 |
|---|---|---|
| **MES** | `get_line_status`(라인 현황), `get_work_order_progress`(작업지시 진척·예상 완료·납기 위험), `get_defect_pareto`(불량 파레토), `list_work_orders_due`(납기 임박) | 생산 현황 질의 비서, 납기 위험 조기 경보, 불량 원인 분석 보조, 교대 인수인계 리포트 |
| **ERP** | `get_material_availability`(가용 재고·부족분·입고 예정), `get_bom`, `get_sales_order_due` | 자재 결품 예측, 납기 위험의 자재 원인 확인 |
| **QMS·CMMS** | `get_inspection_results`, `get_maintenance_history` | 품질 이상 해석, 설비 이상 대응 가이드 |
| **문서** | `search_work_instructions` (작업표준서·매뉴얼 RAG) | 알람·불량 대응 절차 안내 |

**아키텍처 한눈에**

```mermaid
flowchart LR
    A["에이전트<br/>Claude·사내 챗봇·정기 자동화"] --> G["공통 관문<br/>인증·권한·감사 로그"]
    G --> M1["mes-tools"] & M2["erp-tools"] & M3["qms·docs 도구"]
    M1 -- "SELECT<br/>읽기 전용 계정" --> MES[("MES 복제본")]
    M2 -- "GET<br/>조회 전용 토큰" --> ERP["ERP API"]
    MES -. "집계 데이터만" .- OT["OT 영역: PLC·SCADA·Historian<br/>에이전트가 직접 접근·제어하지 않음"]
```

| 원칙 | 내용 |
|---|---|
| **읽기 전용부터** | MES는 운영 DB가 아니라 복제본과 읽기 전용 계정, ERP는 DB가 아니라 공식 API와 조회 전용 토큰 |
| **업무 단위 도구, 계산은 서버에서** | "라인 현황", "납기 위험"처럼 현장 용어로. 진척률·불량률·예상 완료는 서버가 계산해 AI가 추측하지 않게 |
| **OT는 건드리지 않는다** | 설비 제어·설정 변경 도구는 만들지 않는다. 설비 데이터는 이미 승인된 경로(Historian 집계)로만 |
| **에이전트는 제안까지** | 작업지시 변경, 발주는 사람이 기존 시스템의 결재 흐름으로 실행 |
| **개념 검증은 stdio, 확산은 원격 서버** | 여러 사람이 쓰면 중앙 원격 MCP 서버 + 사용자별 권한·감사 로그 |

**예: 납기 위험 조기 경보** — `3일 안에 납기인 작업지시 중 위험한 건과 이유를 알려줘. 자재도 확인해줘`

```text
list_work_orders_due → WO-1001 (52.6%, 내일 납기), WO-1002 (60%)
get_work_order_progress(WO-1001) → 시간당 79개, 남은 569개, 예상 완료 오늘 18:00
get_material_availability(BRG-6201, 1200) → 가용 300, 입고 500 예정, 그래도 400 부족
→ 에이전트: "생산 속도는 납기 안, 하지만 베어링 400개 부족 → 구매 담당과 입고 일정 조정 검토" (결정은 사람)
```

**Java 도구 구현 핵심** (전체 코드는 예제 문서 6장)

```java
// MES 도구 하나 = 이름 + 입력 스키마(JSON Schema) + 설명 + 읽기 전용 힌트 + 실행 함수
Tool tool = Tool.builder("get_work_order_progress", json, """
        {"type":"object",
         "properties":{"woNo":{"type":"string","pattern":"^[A-Za-z0-9_-]{1,30}$"}},
         "required":["woNo"]}
        """)
        .description("작업지시 하나의 진척률, 최근 시간당 생산량, 예상 완료 시각과 납기 위험 여부를 조회한다. "
                + "납기를 맞출 수 있는지 물을 때 사용. 읽기 전용.")
        .annotations(READ_ONLY)
        .build();

SyncToolSpecification.builder().tool(tool).callHandler((ex, req) -> handle(() -> {
    var wo = db.select("SELECT … FROM work_order w LEFT JOIN production_result r … WHERE w.wo_no = ?", str(req, "woNo"));
    // 진척률·시간당 생산량·예상 완료·atRisk를 서버에서 계산해 JSON으로 돌려준다
    return ok(result);
})).build();
```

> 💬 **강사 노트:** 시간이 부족하면 서비스 표와 아키텍처 그림만 5분 안에 보여 주고, Java 코드는 예제 문서로 안내합니다. 제조업 수강생에게는 "우리 공장에서 가장 먼저 만들고 싶은 읽기 전용 도구 3개"를 과제로 내면 좋습니다. 6회차 플러그인(도구 묶음 배포), 8회차 Agent SDK(매일 아침 자동 리포트)와 이어집니다.

---

# 3. 1회차 토론 다시 하기: "OpenAPI 문서 누락을 어떻게 막을까?"

1회차 [8-2 토론 과제](./session-01-introduction.md#8-2-토론-과제-7분)의 요구사항을 다시 봅니다: **"새 API를 추가할 때 OpenAPI 문서(`@Operation`)가 빠지지 않게 하고 싶다."**

| 방식 | 1회차 때의 질문 | 지금 우리가 아는 답 |
|---|---|---|
| A. CLAUDE.md | 항상 지켜질까? | 대부분 지키지만 **가끔 놓친다**. 규칙을 알려 주는 역할 |
| B. 슬래시 커맨드 | 사람이 잊으면? | 사람이 실행해야 동작 → 잊으면 끝 |
| C. 스킬 | 품질까지 좋아질까? | 4회차의 `add-api-endpoint` 스킬처럼 **잘 쓰는 법**(설명, 예시, 응답 코드)을 준다 |
| D. 서브에이전트 | 비용과 효과는? | 문서 품질 리뷰는 가능하지만 매번 돌리면 비싸다. PR 전 한 번이면 충분 |
| E. 훅 | "존재"와 "품질"의 차이는? | **`@Operation`이 있는지(존재)는 훅으로 강제**할 수 있다. 설명이 좋은지(품질)는 스킬·리뷰의 몫 |

> **결론: A + C + E.** CLAUDE.md로 알리고, 스킬로 잘 쓰는 법을 주고, 훅으로 빠지면 잡는다. 오늘 따라하기 ①의 "더 해 보기"([7-1](#7-1-openapi-누락-검사-훅))에서 E를 직접 만들어 봅니다.

> 💬 **강사 노트:** 5분만 씁니다. 조별로 "1회차 때 우리 조의 답"과 "지금의 답"이 어떻게 달라졌는지 한 문장씩 말하게 합니다.

---

# 4. 따라하기 ①: EduHub 품질 자동화 훅 만들기

> **목표**: 다음 4개의 훅을 `.claude/settings.json`에 넣고, 실제로 동작하는 것을 확인한 뒤 PR로 병합합니다.

| # | 이벤트 | matcher | 하는 일 |
|---|---|---|---|
| ① | `PostToolUse` | `Edit\|Write` | 수정한 `.java` → Spotless, `.ts/.tsx` → Prettier |
| ② | `PreToolUse` | `Edit\|Write` | 운영 설정·`.env`·기존 Flyway 마이그레이션·생성된 API 클라이언트 수정 차단 |
| ③ | `PreToolUse` | `Bash` | 운영 접속, `flywayClean`, 강제 푸시, `main` 직접 푸시 차단 |
| ④ | `Stop` | (없음) | 백엔드 코드가 바뀌었으면 테스트 실행, 실패하면 계속 고치게 하기 |

```mermaid
flowchart TD
    E["Claude가 Edit 요청"] --> P2{"② 보호 파일?"}
    P2 -- "예" --> B1["exit 2: 차단 + 대안 안내"]
    P2 -- "아니오" --> ED["파일 수정"]
    ED --> F1["① 자동 포맷"]
    BA["Claude가 Bash 요청"] --> P3{"③ 위험 명령?"}
    P3 -- "예" --> B2["exit 2: 차단 + 대안 안내"]
    P3 -- "아니오" --> RUN["명령 실행"]
    F1 --> END["응답 마무리"]
    RUN --> END
    END --> S4{"④ 백엔드 변경 +<br/>테스트 실패?"}
    S4 -- "예" --> CONT["decision: block<br/>→ Claude가 계속 고침"]
    S4 -- "아니오" --> DONE["완료"]
```

---

## Step 1. 브랜치 만들기와 준비 확인

```bash
git switch main && git pull
git switch -c chore/quality-hooks
jq --version                    # 1.6 이상이면 OK
mkdir -p .claude/hooks
```

```powershell
# Windows PowerShell (5.1은 &&를 지원하지 않으므로 ; 로 잇는다)
git switch main; git pull
git switch -c chore/quality-hooks
jq --version                    # 없으면 winget install jqlang.jq 후 터미널을 새로 연다
New-Item -ItemType Directory -Force .claude\hooks | Out-Null
```

> 💡 **Windows 사용자**: 훅 스크립트(`.sh`)와 이 회차의 `echo '…' | bash .claude/hooks/….sh` 테스트 명령은 **Git Bash 창**에서 그대로 실행합니다 (1-3 Windows 설정 ①). PowerShell에서 테스트할 때는 각 Step의 **PowerShell 테스트 명령**(Git Bash 경로를 지정해 `.sh`를 실행하거나 `.ps1`을 실행)을 씁니다. bash 명령을 PowerShell·cmd에 그대로 붙여 넣으면 문법 오류가 납니다. PowerShell만 쓴다면 아래 `.ps1` 버전을 씁니다.

**이 회차의 훅 스크립트: bash 버전과 Windows PowerShell 버전**

| 훅 | bash (macOS·Linux·Git Bash) | Windows PowerShell | 실린 곳 |
|---|---|---|---|
| 운영 설정 보호 (시연용 최소 버전) | `protect-prod-config.sh` | `protect-prod-config.ps1` | 1-4 |
| 자동 포맷 | `format.sh` | `format.ps1` | Step 2 (PowerShell 버전은 접힌 부분) |
| 보호 파일 전체 | `protect-files.sh` | `protect-files.ps1` | Step 3 (bash), 1-4 (PowerShell) |

PowerShell 버전은 모두 `"command": "powershell.exe"`, `"args": ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "${CLAUDE_PROJECT_DIR}/.claude/hooks/<이름>.ps1"]` 형식으로 등록합니다 (1-3 Windows 설정 ②).

**Windows에서 jq 설치하기**

훅 스크립트(`.sh`)는 Claude Code가 **Git Bash**로 실행하므로, jq는 "Git Bash에서 `jq`가 실행되는 상태"여야 합니다.

| 방법 | 명령 | 비고 |
|---|---|---|
| ⭐ **winget** (Windows 10·11 기본 포함) | `winget install jqlang.jq` | 사용자 PATH에 등록된다. 설치 후 **터미널·Claude Code·데스크톱 앱을 다시 시작** |
| Scoop | `scoop install jq` | Scoop을 쓰는 사람 |
| Chocolatey | `choco install jq` | 관리자 PowerShell 필요 |
| 직접 받기 (설치 도구를 쓸 수 없을 때) | jq 공식 사이트(jqlang.org)의 Windows용 실행 파일(`jq-windows-amd64.exe`)을 받아 이름을 `jq.exe`로 바꾸고 PATH에 있는 폴더에 둔다 | Git Bash에서는 `~/bin` 폴더(`C:\Users\<나>\bin`)에 두면 PATH에 잡힌다 (없으면 만든다) |

```bash
# 설치 후 "Git Bash"를 새로 열어 확인
jq --version          # jq-1.7.1 처럼 나오면 OK
which jq              # 어디에 설치됐는지
echo '{"tool_input":{"file_path":"C:\\work\\a.java"}}' | jq -r '.tool_input.file_path'   # C:\work\a.java
```

```powershell
# PowerShell에서 확인 (which에 해당하는 명령)
jq --version
Get-Command jq            # Source 열에 jq.exe의 위치가 나온다
where.exe jq              # 같은 결과를 경로만 출력. PowerShell에서는 where가 다른 명령(Where-Object)이라 반드시 .exe까지 쓴다
```

```bat
:: 명령 프롬프트(cmd)에서 확인
jq --version
where jq
```

| 셸 | `which jq`에 해당하는 명령 |
|---|---|
| Git Bash | `which jq` (그대로) |
| PowerShell | `Get-Command jq` 또는 `where.exe jq` |
| 명령 프롬프트(cmd) | `where jq` |

> 💡 PowerShell·cmd에서는 보이는데 Git Bash에서 안 보인다면, 훅이 실행되는 쪽은 **Git Bash**이므로 아래 표의 방법으로 Git Bash에서 보이게 만듭니다.

| 증상 | 원인과 해결 |
|---|---|
| PowerShell에서는 되는데 Git Bash에서 `jq: command not found` | jq를 설치하기 **전부터** 열어 둔 Git Bash 창. 창을 새로 연다. 그래도 안 되면 PowerShell의 `Get-Command jq`로 실제 위치를 보고 Git Bash의 `which jq`와 비교하고, 직접 받은 `jq.exe`를 `~/bin`에 둔다 |
| 터미널에서는 되는데 훅에서만 `jq: command not found` 또는 "jq가 없어 보호 훅이 검사할 수 없습니다" | Claude Code(또는 데스크톱 앱)를 jq 설치 **전에** 실행했다. Claude Code를 다시 시작한다 |
| 회사 PC라 winget·관리자 권한을 쓸 수 없음 | "직접 받기" 방법으로 `~/bin`에 둔다. 그래도 어렵다면 jq가 필요 없는 PowerShell 훅(1-4 `protect-files.ps1`의 `ConvertFrom-Json` 방식)을 쓴다 |

> ✅ **확인**: `git branch --show-current`가 `chore/quality-hooks`이고, `jq --version`이 버전을 출력하면 됩니다. 윈도우에서는 훅이 Git Bash로 실행되는지 강사와 함께 확인합니다.

---

## Step 2. 훅 ① 자동 포맷 (PostToolUse)

권한 모드를 **Plan**으로 두고 지시합니다.

```text
[목표] Claude가 파일을 수정할 때마다 해당 파일이 자동으로 포맷되게 하고 싶어.
[위치] .claude/hooks/format.sh 새로 만들고, .claude/settings.json의 hooks.PostToolUse에 등록
[범위]
- matcher는 Edit|Write
- stdin JSON의 tool_input.file_path를 jq로 읽는다
- *.java → backend 폴더에서 ./gradlew spotlessApply -q
- *.ts, *.tsx → frontend의 prettier로 그 파일만 --write
- 포맷 도구가 없거나 실패해도 작업이 멈추지 않게 exit 0
[하지 않는 것] 다른 훅, 다른 설정 변경
[완료 기준] 스크립트에 실행 권한, settings.json이 올바른 JSON
[진행 방식] 먼저 스크립트와 설정 내용을 보여줘
```

**기대하는 결과 (예)**

```bash
#!/usr/bin/env bash
# .claude/hooks/format.sh — PostToolUse(Edit|Write): 수정한 파일만 포맷
FILE=$(jq -r '.tool_input.file_path // empty')
[ -z "$FILE" ] && exit 0

case "$FILE" in
  *.java)
    (cd "$CLAUDE_PROJECT_DIR/backend" && ./gradlew spotlessApply -q) >/dev/null 2>&1 ;;
  *.ts|*.tsx)
    npx --prefix "$CLAUDE_PROJECT_DIR/frontend" prettier --write "$FILE" >/dev/null 2>&1 ;;
esac
exit 0
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/format.sh\"", "timeout": 120 }
        ]
      }
    ]
  }
}
```

> ✅ `format.sh`는 가짜 `gradlew`·`npx`로 바꿔 실행해 확인했습니다: `.java`면 `backend`에서 `spotlessApply`, `.tsx`면 `prettier --write <그 파일>`, 그 밖의 파일·경로 없는 입력은 아무것도 하지 않고 모두 `exit 0`.

<details>
<summary>Windows PowerShell 버전: <code>format.ps1</code> (Git Bash를 쓰지 않을 때)</summary>

```powershell
# .claude/hooks/format.ps1 — PostToolUse(Edit|Write): 수정한 파일만 포맷 (Windows PowerShell 버전)
# Step 2의 format.sh와 같은 동작. 포맷 도구가 없거나 실패해도 작업이 멈추지 않도록 항상 exit 0

# 1) stdin으로 들어온 훅 입력(JSON)에서 수정한 파일 경로를 꺼낸다
try {
    $hookInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
} catch {
    exit 0
}
$file = [string]$hookInput.tool_input.file_path
$root = $env:CLAUDE_PROJECT_DIR
if ([string]::IsNullOrEmpty($file) -or [string]::IsNullOrEmpty($root)) { exit 0 }

# 2) 확장자에 따라 포맷 (출력은 모두 버린다: PostToolUse 출력이 컨텍스트를 채우지 않게)
try {
    if ($file -match '\.java$') {
        # -p 로 backend 폴더를 프로젝트로 지정 → 현재 폴더를 바꿀 필요가 없다
        $backend = Join-Path $root 'backend'
        & (Join-Path $backend 'gradlew.bat') -p $backend spotlessApply -q *> $null
    }
    elseif ($file -match '\.tsx?$') {
        # npx.ps1은 실행 정책에 막힐 수 있어 npx.cmd를 직접 부른다
        & npx.cmd --prefix (Join-Path $root 'frontend') prettier --write $file *> $null
    }
} catch {
    # gradlew.bat·npx.cmd가 없으면 여기로 온다. 포맷만 건너뛴다
}
exit 0
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "powershell.exe",
            "args": ["-NoProfile", "-ExecutionPolicy", "Bypass", "-File", "${CLAUDE_PROJECT_DIR}/.claude/hooks/format.ps1"],
            "timeout": 120
          }
        ]
      }
    ]
  }
}
```

| `format.sh`와 다른 점 | 설명 |
|---|---|
| 입력 읽기 | `jq` 대신 `ConvertFrom-Json` (jq 설치 불필요) |
| Gradle 실행 | `./gradlew` 대신 `gradlew.bat`, `cd` 대신 `-p <backend 폴더>` |
| npx 실행 | `npx.cmd`를 직접 호출 (`npx.ps1`이 실행 정책에 막히는 것을 피한다) |
| 출력 버리기 | `>/dev/null 2>&1` 대신 `*> $null` |
| 확장자 비교 | `-match '\.tsx?$'`는 대소문자를 구분하지 않는다 (`A.JAVA`도 포맷) |

**PowerShell로 직접 테스트** (프로젝트 루트에서)

```powershell
$env:CLAUDE_PROJECT_DIR = (Get-Location).Path
'{"tool_input":{"file_path":"C:\\work\\eduhub\\backend\\src\\main\\java\\A.java"}}' |
  powershell.exe -NoProfile -ExecutionPolicy Bypass -File .claude\hooks\format.ps1; "exit=$LASTEXITCODE"   # exit=0, A.java 포맷
'{"tool_input":{"file_path":"C:\\work\\eduhub\\README.md"}}' |
  powershell.exe -NoProfile -ExecutionPolicy Bypass -File .claude\hooks\format.ps1; "exit=$LASTEXITCODE"   # exit=0, 아무것도 안 함
```

> ⚠️ 이 교재 환경에서는 PowerShell을 실행할 수 없어 `format.ps1`을 **직접 실행해 보지 못했습니다.** 수업 전에 Windows에서 위 테스트를 한 번 실행하고, 일부러 들여쓰기를 망친 `.java` 파일이 정리되는지 확인해 주세요.

</details>

> 💡 `spotlessApply`는 프로젝트 전체를 검사하므로 프로젝트가 커지면 느려질 수 있습니다. 느리면 Spotless의 파일 지정 옵션이나 google-java-format을 그 파일에만 실행하는 방식으로 바꿉니다 (공식 문서 확인 필요).

**동작 확인**

```text
@backend/src/main/java/com/example/eduhub/common/HealthController.java 의 health 메서드에
들여쓰기를 일부러 엉망으로 한 주석 한 줄을 추가해줘.
```

> ✅ **확인**: 수정 직후 파일을 열어 보면 들여쓰기가 정리되어 있어야 합니다. `/hooks`에서 PostToolUse 훅이 프로젝트 설정 출처로 보이는지 확인합니다. 확인이 끝나면 `/rewind`로 주석을 되돌립니다.

---

## Step 3. 훅 ② 보호 파일 수정 차단 (PreToolUse: Edit|Write)

```text
[목표] 사람이 관리해야 하는 파일을 Claude가 수정하지 못하게 막고 싶어.
[위치] .claude/hooks/protect-files.sh, settings.json의 hooks.PreToolUse (matcher: Edit|Write)
[범위] 다음 경우 stderr에 "이유 + 대신 할 일"을 쓰고 exit 2
- application-prod.yml → "운영 설정은 사람이 PR로만 수정. 개발은 application-dev.yml"
- .env, .env.* (단 .env.example은 허용) → "비밀 파일은 수정하지 않는다. .env.example에 변수 이름만"
- db/migration/V*.sql 중 이미 존재하는 파일 → "기존 마이그레이션은 수정 금지. 새 버전 파일 추가"
- frontend/packages/api-client/src/generated/ → "생성 코드. npm run generate:api 사용"
[완료 기준] 각 경우를 echo로 JSON을 흘려 넣는 셸 테스트 4개가 모두 exit 2, 일반 파일은 exit 0
[진행 방식] 계획부터
```

**기대하는 결과 (예)**

```bash
#!/usr/bin/env bash
# .claude/hooks/protect-files.sh — PreToolUse(Edit|Write): 보호 파일 수정 차단
# jq가 없으면 검사할 수 없으므로 통과시키지 않고 막는다 (그냥 두면 보호가 조용히 꺼진다)
command -v jq >/dev/null 2>&1 || { echo "jq가 없어 보호 훅이 검사할 수 없습니다. 사람이 jq를 설치하세요 (Windows: winget install jqlang.jq)." >&2; exit 2; }
FILE=$(jq -r '.tool_input.file_path // empty')
FILE="${FILE//\\//}"   # Windows 경로의 \ 를 / 로 (Git Bash에서 실행돼도 경로는 \ 로 들어온다)
block() { echo "$1" >&2; exit 2; }

case "$FILE" in
  *application-prod.yml)
    block "운영 설정은 사람이 PR로만 수정합니다. 개발 설정은 application-dev.yml을 수정하세요." ;;
  *.env.example) ;;
  */.env|*/.env.*)
    block "비밀 파일은 수정하지 않습니다. 필요한 변수 이름은 .env.example에 추가하세요." ;;
  */api-client/src/generated/*)
    block "자동 생성 코드입니다. 'npm run generate:api'로 다시 생성하세요." ;;
esac

if [[ "$FILE" == */db/migration/V*.sql && -f "$FILE" ]]; then
  block "기존 Flyway 마이그레이션은 수정할 수 없습니다. 다음 버전 번호로 새 파일을 추가하세요."
fi
exit 0
```

> 💡 Windows PowerShell 버전(`protect-files.ps1`) 전체와 테스트 방법은 [1-4](#1-4-시연-훅이-막고-claude가-길을-바꾼다-강사-시연-3분)에 있습니다.

**직접 테스트해 보기** (훅은 프로그램이므로 Claude 없이도 테스트할 수 있습니다)

```bash
echo '{"tool_input":{"file_path":"/x/backend/src/main/resources/application-prod.yml"}}' \
  | bash .claude/hooks/protect-files.sh; echo "exit=$?"        # exit=2
echo '{"tool_input":{"file_path":"/x/backend/src/main/java/A.java"}}' \
  | bash .claude/hooks/protect-files.sh; echo "exit=$?"        # exit=0
```

**이 명령은 무엇을 하나요?** 훅은 Claude Code가 stdin으로 JSON을 넘겨주면 실행되는 평범한 스크립트입니다. `echo '{"tool_input":…}'`로 Claude Code가 넘겨줄 입력을 흉내 내 스크립트에 흘려 넣고(`|`), `echo "exit=$?"`로 종료 코드(2 = 차단, 0 = 통과)를 확인합니다. 설정에 등록하기 전에 **스크립트의 판단만 따로** 시험하는 단위 테스트입니다.

**어디서 실행하나요?** 위 명령은 bash 문법(줄 끝 `\` 이음, `$?`)입니다. macOS·Linux 터미널, Windows는 **Git Bash 창**에서 **저장소 루트로 이동한 뒤** 실행합니다. PowerShell·cmd에 그대로 붙여 넣으면 문법 오류가 납니다. PowerShell에서 시험하려면 Git Bash의 bash를 직접 지정합니다.

```powershell
# Windows PowerShell (저장소 루트에서)
$bash = "C:\Program Files\Git\bin\bash.exe"
'{"tool_input":{"file_path":"/x/backend/src/main/resources/application-prod.yml"}}' | & $bash .claude/hooks/protect-files.sh; "exit=$LASTEXITCODE"   # exit=2
'{"tool_input":{"file_path":"/x/backend/src/main/java/A.java"}}' | & $bash .claude/hooks/protect-files.sh; "exit=$LASTEXITCODE"   # exit=0

# PowerShell 버전(protect-files.ps1)을 쓰는 경우
$ps1 = ".claude\hooks\protect-files.ps1"
'{"tool_input":{"file_path":"C:\\work\\eduhub\\backend\\src\\main\\resources\\application-prod.yml"}}' | powershell.exe -NoProfile -ExecutionPolicy Bypass -File $ps1; "exit=$LASTEXITCODE"   # exit=2
'{"tool_input":{"file_path":"C:\\work\\eduhub\\backend\\src\\main\\java\\A.java"}}' | powershell.exe -NoProfile -ExecutionPolicy Bypass -File $ps1; "exit=$LASTEXITCODE"   # exit=0
```

> PowerShell 명령은 이 교재 환경에 PowerShell이 없어 실행해 보지 못했습니다. PowerShell에서 그냥 `bash`라고 쓰면 WSL의 bash가 실행될 수 있어(그 안에는 jq가 없을 수 있음) 경로를 직접 지정했습니다.

**테스트가 에러로 끝날 때**

| 증상 | 원인 | 해결 |
|---|---|---|
| `No such file or directory`, `exit=127` | 저장소 루트가 아닌 곳에서 실행했거나, 스크립트를 아직 만들지 않음 | 저장소 루트로 `cd`, `.claude/hooks/protect-files.sh`가 있는지 확인 |
| `syntax error near unexpected token $'in\r'` | 스크립트가 CRLF 줄바꿈으로 저장됨 (Windows) | LF로 다시 저장, `.gitattributes`에 `*.sh text eol=lf` |
| `jq가 없어 보호 훅이 검사할 수 없습니다`, 일반 파일도 `exit=2` | jq 미설치 (위 jq 확인 줄이 막은 것) | jq 설치 (Step 1). 의도한 동작입니다 |
| PowerShell·cmd에서 문법 오류 | bash 문법을 다른 셸에서 실행 | Git Bash 창 또는 위 PowerShell 명령 |

```bash
# (선택) jq가 없을 때도 막히는지 확인: PATH를 비워 jq를 못 찾게 하고 실행
echo '{"tool_input":{"file_path":"/x/backend/src/main/java/A.java"}}' \
  | env PATH=/nonexistent /bin/bash .claude/hooks/protect-files.sh; echo "exit=$?"   # 안내 메시지 + exit=2
```

```powershell
# 같은 확인을 Windows PowerShell에서 (Git Bash 안에서 PATH를 비워 실행)
'{"tool_input":{"file_path":"/x/backend/src/main/java/A.java"}}' | & $bash -c 'env PATH=/nonexistent /bin/bash .claude/hooks/protect-files.sh'; "exit=$LASTEXITCODE"   # 안내 메시지 + exit=2
```

> ✅ **확인**: Claude에게 `application-prod.yml에 로그 레벨 설정을 추가해줘`라고 지시하면 수정이 막히고, Claude가 이유를 읽고 다른 방법(개발 설정 수정 또는 사람에게 확인 요청)을 제안해야 합니다.

---

## Step 4. 훅 ③ 위험 명령 차단 (PreToolUse: Bash)

```text
[목표] 운영 환경 접속과 되돌릴 수 없는 명령을 Bash 단계에서 막고 싶어.
[위치] .claude/hooks/guard-bash.sh, settings.json의 hooks.PreToolUse (matcher: Bash)
[범위] tool_input.command에 다음이 있으면 exit 2 + 이유와 대안
- 운영 호스트·DB 이름: eduhub-prod, prod-db (우리 팀 명명 규칙)
- flywayClean
- git push --force, git push -f
- git push origin main (main 직접 푸시)
[하지 않는 것] 일반 명령을 느리게 만드는 무거운 검사
[완료 기준] Step 3처럼 셸 테스트로 차단·통과 확인
```

**기대하는 결과 (예)**

```bash
#!/usr/bin/env bash
# .claude/hooks/guard-bash.sh — PreToolUse(Bash): 위험 명령 차단 (최선의 노력용 안전망)
# jq가 없으면 검사할 수 없으므로 통과시키지 않고 막는다 (그냥 두면 보호가 조용히 꺼진다)
command -v jq >/dev/null 2>&1 || { echo "jq가 없어 보호 훅이 검사할 수 없습니다. 사람이 jq를 설치하세요 (Windows: winget install jqlang.jq)." >&2; exit 2; }
CMD=$(jq -r '.tool_input.command // empty')
block() { echo "$1" >&2; exit 2; }

case "$CMD" in
  *eduhub-prod*|*prod-db*)
    block "운영 환경 접속 명령은 실행할 수 없습니다. 개발 DB(localhost)나 테스트(Testcontainers)를 사용하세요." ;;
  *flywayClean*)
    block "flywayClean은 DB를 모두 지웁니다. 개발 DB 초기화가 필요하면 사람에게 요청하세요." ;;
  *"push --force"*|*"push -f"*)
    block "강제 푸시는 금지입니다. 새 커밋으로 수정하세요." ;;
  *"push origin main"*)
    block "main에 직접 푸시하지 않습니다. 작업 브랜치를 푸시하고 PR을 만드세요." ;;
esac
exit 0
```

**직접 테스트해 보기**

```bash
# macOS·Linux·Git Bash (저장소 루트에서)
echo '{"tool_input":{"command":"git push --force origin feature/x"}}' | bash .claude/hooks/guard-bash.sh; echo "exit=$?"   # exit=2
echo '{"tool_input":{"command":"git push origin main"}}' | bash .claude/hooks/guard-bash.sh; echo "exit=$?"               # exit=2
echo '{"tool_input":{"command":"./gradlew flywayClean"}}' | bash .claude/hooks/guard-bash.sh; echo "exit=$?"              # exit=2
echo '{"tool_input":{"command":"./gradlew test"}}' | bash .claude/hooks/guard-bash.sh; echo "exit=$?"                     # exit=0
```

```powershell
# Windows PowerShell (Git Bash로 .sh 실행)
$bash = "C:\Program Files\Git\bin\bash.exe"
'{"tool_input":{"command":"git push --force origin feature/x"}}' | & $bash .claude/hooks/guard-bash.sh; "exit=$LASTEXITCODE"   # exit=2
'{"tool_input":{"command":"git push origin main"}}' | & $bash .claude/hooks/guard-bash.sh; "exit=$LASTEXITCODE"               # exit=2
'{"tool_input":{"command":"./gradlew flywayClean"}}' | & $bash .claude/hooks/guard-bash.sh; "exit=$LASTEXITCODE"              # exit=2
'{"tool_input":{"command":"./gradlew test"}}' | & $bash .claude/hooks/guard-bash.sh; "exit=$LASTEXITCODE"                     # exit=0
```

> 💡 **jq 확인 줄**: 보호 훅 세 개(`protect-prod-config.sh`, `protect-files.sh`, `guard-bash.sh`)는 맨 앞에서 jq가 있는지 확인합니다. jq가 없으면 `jq -r …`가 실패해 경로·명령이 빈 값이 되고, 어떤 규칙에도 걸리지 않아 **`exit 0`(통과)으로 끝납니다.** 보호가 조용히 꺼지는 것이므로 막는 쪽(`exit 2`)을 택했습니다. jq가 없는 PC에서는 `guard-bash.sh`가 Claude의 모든 Bash 명령을 막으므로 **jq는 사람이 직접 설치**합니다. 포맷(`format.sh`)·알림(`check-openapi.sh`)처럼 막는 목적이 아닌 훅은 jq가 없으면 아무것도 하지 않고 넘어가도 괜찮습니다.

> ⚠️ 이 훅은 **안전망**입니다. `git push origin HEAD:main`처럼 다른 형태의 명령은 빠져나갈 수 있습니다. 그래서 GitHub의 **`main` 브랜치 보호**(직접 푸시·강제 푸시 금지)와 권한 설정의 `deny`(예: `Bash(git push --force *)`)를 함께 씁니다. 하나만 믿지 않는 것이 핵심입니다.

> ✅ **확인**: Claude에게 `git push --force 해줘`라고 지시해 차단되는지 봅니다.

---

## Step 5. 훅 ④ 마무리 테스트 (Stop)

Claude가 "끝났습니다"라고 하기 전에 백엔드 테스트가 통과하는지 확인하게 합니다.

```text
[목표] Claude가 응답을 마칠 때 백엔드 코드가 바뀌었으면 테스트를 돌리고, 실패하면 계속 고치게 하고 싶어.
[위치] .claude/hooks/stop-test.sh, settings.json의 hooks.Stop
[범위]
- 입력 JSON의 stop_hook_active가 true면 바로 exit 0 (무한 반복 방지)
- git status --porcelain -- backend/src 결과가 비어 있으면 exit 0
- cd backend && ./gradlew test -q, 실패하면 실패 줄(FAILED, Exception) 20줄만 골라
  {"decision":"block","reason":"..."} JSON을 stdout으로 출력
- timeout 300초
[완료 기준] 테스트를 일부러 깨뜨렸을 때 Claude가 멈추지 않고 고치는지 확인
```

**기대하는 결과 (예)**

```bash
#!/usr/bin/env bash
# .claude/hooks/stop-test.sh — Stop: 백엔드가 바뀌었으면 테스트 실행
INPUT=$(cat)
# Stop 훅 때문에 이미 한 번 더 진행 중이면 반복하지 않는다
[ "$(echo "$INPUT" | jq -r '.stop_hook_active // false')" = "true" ] && exit 0

cd "$CLAUDE_PROJECT_DIR" || exit 0
[ -z "$(git status --porcelain -- backend/src)" ] && exit 0

if ! OUT=$(cd backend && ./gradlew test -q 2>&1); then
  FAILS=$(echo "$OUT" | grep -E 'FAILED|Exception' | head -20)
  jq -n --arg r "백엔드 테스트가 실패했습니다. 원인을 고친 뒤 마무리하세요.
$FAILS" '{decision: "block", reason: $r}'
fi
exit 0
```

| 장점 | 비용과 주의 |
|---|---|
| "테스트 통과했습니다"라는 말이 아니라 **실제 결과**로 마무리 | 백엔드를 바꿀 때마다 응답 끝에 테스트 시간이 붙는다 |
| 실패 줄만 골라 넘겨 **토큰을 아낀다** | 테스트가 오래 걸리면 `--tests`로 변경 영역만, 또는 알림만 하는 방식으로 |
| 리뷰 전에 깨진 코드가 PR로 가지 않는다 | `stop_hook_active` 확인이 없으면 실패가 계속될 때 반복될 수 있다 (공식 문서의 Stop 입력 필드 확인 필요) |

> ✅ **확인**: Claude에게 `EnrollmentServiceTest의 정원 검사 기대값을 일부러 틀리게 바꿔줘`라고 한 뒤, 응답이 끝날 때 테스트 실패로 Claude가 다시 작업을 이어가는지 봅니다. 확인 후 `/rewind`나 `git restore`로 원래대로 돌립니다.

> 💬 **강사 노트:** 시간이 부족하면 Step 5는 시연으로 대신하고 과제로 넘깁니다. "Stop 훅은 테스트 시간만큼 매 응답이 느려진다"는 비용을 꼭 같이 이야기합니다.

---

## Step 6. 전체 설정 확인, 커밋, PR

**완성된 `.claude/settings.json`의 hooks 부분 (예)**

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/protect-files.sh\"" }]
      },
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/guard-bash.sh\"" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/format.sh\"", "timeout": 120 }]
      }
    ],
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/stop-test.sh\"", "timeout": 300 }]
      }
    ]
  }
}
```

> 💡 4회차에 만든 `permissions` 항목은 그대로 두고 `hooks` 항목만 추가합니다. 설정 파일을 고치면 보통 자동으로 다시 읽히지만, 반영이 안 되면 `/hooks`로 확인하고 세션을 다시 시작합니다.

**CLAUDE.md에 한 줄 추가** (Claude가 훅의 존재를 미리 알면 막히기 전에 피해 갑니다)

```markdown
## 자동화 (훅)
- 파일 수정 시 자동 포맷, 보호 파일·위험 명령은 훅이 차단한다. 차단 메시지의 안내를 따른다
```

**diff 검토 체크리스트**

- [ ] 스크립트 4개에 실행 권한이 있고(`chmod +x`), 모두 마지막에 `exit 0`으로 끝나는가?
- [ ] 차단 메시지에 **이유와 대안**이 함께 있는가?
- [ ] `settings.json`이 올바른 JSON이고 4회차 `permissions`가 지워지지 않았는가? (`jq . .claude/settings.json`)
- [ ] 운영 호스트 이름 외에 **실제 운영 접속 정보**가 스크립트에 들어가지 않았는가?
- [ ] `.github/CODEOWNERS`에 `/.claude/` 담당자가 있는가? (없으면 이번 PR에 추가)

```bash
chmod +x .claude/hooks/*.sh
git add .claude/ CLAUDE.md .github/CODEOWNERS
git diff --staged
git commit -m "chore(claude): 자동 포맷·보호 파일·위험 명령·마무리 테스트 훅 추가"
git push -u origin chore/quality-hooks
```

> 💡 **Windows**: `chmod +x`는 필요 없습니다(Git Bash에서는 실행해도 무해). 설정에서 `bash "…/format.sh"`처럼 **bash로 직접 실행**하므로 실행 권한이 없어도 동작합니다. 대신 `.gitattributes`에 `*.sh text eol=lf`를 두어 줄바꿈이 CRLF로 바뀌지 않게 합니다.

> ✅ **확인**: PR 본문에 "Claude에게 맡긴 것 / 직접 한 것"을 쓰고, 리뷰어는 **스크립트를 직접 읽고** 승인합니다. 훅은 팀원 모두의 PC에서 실행되기 때문입니다.

---

## 4-7. 자주 겪는 문제 (훅)

| 증상 | 원인과 해결 |
|---|---|
| 훅이 전혀 실행되지 않음 | `/hooks`에서 등록 여부 확인. JSON 문법 오류(`jq . .claude/settings.json`), 이벤트 이름 철자(`PostToolUse`) 확인 |
| `jq: command not found`, 또는 보호 훅이 "jq가 없어 … 검사할 수 없습니다"로 모든 수정·명령을 막음 | jq 설치. 윈도우는 Step 1의 "Windows에서 jq 설치하기"와 Git Bash에서 `which jq`로 확인. 설치 후 Claude Code를 다시 시작 |
| 스크립트를 찾지 못함 | 상대 경로 대신 `"$CLAUDE_PROJECT_DIR/.claude/hooks/…"` 사용 |
| 차단했는데 Claude가 같은 시도를 반복 | 메시지에 **대안**을 쓴다. "금지"만 쓰면 다른 우회 방법을 찾으려 한다 |
| 작업이 눈에 띄게 느려짐 | PostToolUse에서 전체 빌드를 돌리고 있지 않은지 확인. 무거운 작업은 Stop으로 |
| Stop 훅이 계속 반복됨 | `stop_hook_active` 확인 누락. 테스트가 환경 문제로 계속 실패하면 사람이 개입 |
| 윈도우에서 줄바꿈 오류(`\r`) | 스크립트를 LF로 저장. `.gitattributes`에 `*.sh text eol=lf` |
| 윈도우에서 보호 파일이 막히지 않음 | 경로가 `C:\...\.env`처럼 `\`로 들어와 `*/.env` 패턴이 맞지 않음. 스크립트 앞에서 `FILE="${FILE//\\//}"`로 바꾼다 |
| 윈도우 PowerShell 훅에서 경로가 비어 있음 | `$CLAUDE_PROJECT_DIR` 대신 `$env:CLAUDE_PROJECT_DIR` 또는 `${CLAUDE_PROJECT_DIR}` (1-3 Windows 설정) |

---

## ☕ 휴식 (8분)

---

# 5. 따라하기 ②: DB MCP 연결과 FR-07~08 구현

> **목표**: 개발 DB에 **읽기 전용** MCP를 연결하고, 실제 스키마와 데이터를 확인하며 FR-07(내 신청 내역), FR-08(명단·수료 처리)을 구현해 PR로 병합합니다. 오늘 만든 훅이 그대로 동작하는 것도 함께 확인합니다.

| 요구사항 | 내용 (2회차 [3-3](./session-02-hands-on-kickoff.md#3-3-기능-요구사항-개략)) |
|---|---|
| FR-07 | 내 신청 내역(확정, 대기, 취소, 수료)을 조회한다 |
| FR-08 | 관리자는 회차별 신청자·대기자 명단을 보고 수료 처리한다 |

---

## Step 7. 개발 DB에 읽기 전용 계정 만들기

이 작업은 **사람이 직접** 합니다. DB 권한은 Claude에게 맡기지 않습니다. 개발 DB가 **MariaDB**라면 계정은 아래 [MariaDB용 SQL](#mariadb용-읽기-전용-계정-sql)로 만들고, 환경 변수와 `.mcp.json`은 [MariaDB 버전](#개발-db가-mariadb인-팀-step-78의-mariadb-버전)을 따릅니다.

```bash
# 개발 DB 컨테이너에 접속 (컨테이너 이름은 팀 설정에 맞게)
docker exec -it eduhub-postgres psql -U eduhub -d eduhub
```

```sql
-- 개발 DB에서만 실행. 비밀번호는 팀 공유 금지, 각자 정한다
CREATE ROLE eduhub_readonly LOGIN PASSWORD '각자-정한-비밀번호';
GRANT CONNECT ON DATABASE eduhub TO eduhub_readonly;
GRANT USAGE ON SCHEMA public TO eduhub_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO eduhub_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO eduhub_readonly;
ALTER ROLE eduhub_readonly SET default_transaction_read_only = on;
```

#### MariaDB용 읽기 전용 계정 SQL

```bash
# 개발 DB 컨테이너에 root로 접속 (컨테이너 이름은 팀 설정에 맞게)
docker exec -it eduhub-mariadb mariadb -u root -p
```

```sql
-- 개발 DB에서만 실행. 비밀번호는 팀 공유 금지, 각자 정한다
-- ① 계정 만들기 + 안전 제한 (동시 접속 5개, 쿼리 하나당 최대 10초)
CREATE USER 'eduhub_readonly'@'%' IDENTIFIED BY '각자-정한-비밀번호'
  WITH MAX_USER_CONNECTIONS 5
       MAX_STATEMENT_TIME 10;

-- ② eduhub 스키마에만 읽기 권한 (앞으로 만들 테이블에도 자동 적용)
GRANT SELECT, SHOW VIEW ON eduhub.* TO 'eduhub_readonly'@'%';

-- ③ 확인: USAGE(접속만)와 eduhub의 SELECT, SHOW VIEW만 보여야 한다
SHOW GRANTS FOR 'eduhub_readonly'@'%';
```

**PostgreSQL SQL과 무엇이 대응하나**

| PostgreSQL | MariaDB | 설명 |
|---|---|---|
| `CREATE ROLE … LOGIN PASSWORD '…'` | `CREATE USER 'eduhub_readonly'@'%' IDENTIFIED BY '…'` | MariaDB 계정은 **'이름'@'접속 호스트'** 한 쌍이다. 내 PC에서 Docker 컨테이너로 접속하면 호스트가 컨테이너 네트워크 주소로 보이므로 `'%'`(어디서나)를 쓴다. **개발 DB에서만** |
| `GRANT CONNECT …`, `GRANT USAGE ON SCHEMA …` | 필요 없음 | DB에 권한을 주면 접속할 수 있다 |
| `GRANT SELECT ON ALL TABLES …` + `ALTER DEFAULT PRIVILEGES …` | `GRANT SELECT, SHOW VIEW ON eduhub.*` | **DB 단위 권한**이라 나중에 Flyway로 만든 테이블에도 그대로 적용된다. `SHOW VIEW`는 뷰 정의 조회용 |
| `ALTER ROLE … SET default_transaction_read_only = on` | 계정 단위 설정 없음 | MariaDB는 **SELECT만 준 권한**이 핵심 방어선이다. 읽기 전용 트랜잭션은 MCP 서버가 건다 (`mcp-server-mysql` 기본값) |
| (참고: `ALTER ROLE … SET statement_timeout`) | `WITH MAX_STATEMENT_TIME 10`, `MAX_USER_CONNECTIONS 5` | 큰 조인·전체 스캔 같은 무거운 쿼리와 접속 폭주를 막는 보조 장치. MariaDB 전용 문법 (MySQL에는 없음) |

```bash
# 확인: 쓰기는 권한 오류로 실패해야 한다 (macOS·Linux·Git Bash, mariadb 클라이언트가 있을 때)
mariadb -h 127.0.0.1 -u eduhub_readonly -p eduhub -e "SELECT COUNT(*) FROM enrollment"   # 성공
mariadb -h 127.0.0.1 -u eduhub_readonly -p eduhub -e "DELETE FROM enrollment"
# ERROR 1142 (42000): DELETE command denied to user 'eduhub_readonly'@'…' for table `eduhub`.`enrollment`
```

```powershell
# Windows PowerShell (mariadb 클라이언트가 없으면 컨테이너 안의 클라이언트로)
docker exec -it eduhub-mariadb mariadb -u eduhub_readonly -p eduhub -e "SELECT COUNT(*) FROM enrollment"   # 성공
docker exec -it eduhub-mariadb mariadb -u eduhub_readonly -p eduhub -e "DELETE FROM enrollment"           # ERROR 1142 … denied
```

**MariaDB에서 자주 막히는 곳**

| 증상 | 원인 | 해결 (root로 실행) |
|---|---|---|
| 비밀번호가 맞는데 `ERROR 1045 Access denied for user 'eduhub_readonly'@'localhost'` | 기본 설치가 만든 **익명 사용자** `''@'localhost'`가 `'%'`보다 먼저 맞는다 | `SELECT user, host FROM mysql.user WHERE user = '';`로 확인하고, 나온 행마다 `DROP USER ''@'localhost';` (호스트 이름 행도 같은 방식) |
| `SHOW DATABASES`에 `test`가 보이고, 읽기 전용 계정으로 `test`에 테이블을 만들 수 있다 | 기본 설치가 **모든 사용자(PUBLIC)**에게 `test` DB 권한을 준다. `SHOW GRANTS`에는 나오지 않는다 | ``REVOKE ALL PRIVILEGES ON `test`.* FROM PUBLIC;``, ``REVOKE ALL PRIVILEGES ON `test\_%`.* FROM PUBLIC;``, `DROP DATABASE IF EXISTS test;` |
| `CREATE USER`에서 문법 오류 | MariaDB가 아니라 MySQL이다 | `WITH MAX_USER_CONNECTIONS …` 줄을 빼고 실행 (MySQL은 쿼리 시간 제한 방식이 다르다) |

> 위 두 가지 기본 설정은 주로 **직접 설치한 MariaDB**(설치 프로그램, 패키지)에 남아 있습니다. 공식 Docker 이미지로 만든 개발 DB라도 `SHOW DATABASES`와 익명 사용자 확인은 한 번 해 봅니다.

> ✅ MariaDB 10.11에서 위 SQL을 실행해 확인했습니다: `SELECT` 성공, `DELETE`·`UPDATE`·`CREATE`·`DROP`은 `ERROR 1142`로 거부, 다른 DB(`mysql`) 조회 거부, 계정을 만든 **뒤에** 생긴 테이블도 조회 가능, 제한 시간을 넘긴 쿼리는 `max_statement_time exceeded`로 중단. 익명 사용자와 `test` DB 문제도 실제로 재현한 뒤 위 해결 방법으로 풀리는 것을 확인했습니다.

```bash
# macOS·Linux·WSL: 셸 프로필(~/.zshrc, ~/.bashrc)에 추가 (.env로 관리하려면 아래 "접속 정보를 .env로 관리하려면")
export EDUHUB_DB_READONLY_URL="postgresql://eduhub_readonly:각자-정한-비밀번호@localhost:5432/eduhub"
```

```powershell
# Windows PowerShell
# ① 지금 열린 창에서만 (창을 닫으면 사라짐)
$env:EDUHUB_DB_READONLY_URL = "postgresql://eduhub_readonly:각자-정한-비밀번호@localhost:5432/eduhub"

# ② 내 계정에 영구 저장 (새로 여는 터미널과 앱부터 적용 → Claude Code·데스크톱 앱을 다시 시작)
[Environment]::SetEnvironmentVariable("EDUHUB_DB_READONLY_URL",
  "postgresql://eduhub_readonly:각자-정한-비밀번호@localhost:5432/eduhub", "User")

# 확인
$env:EDUHUB_DB_READONLY_URL
```

> 💡 Windows에서는 화면으로도 설정할 수 있습니다: 시작 메뉴에서 "환경 변수" 검색 → **계정의 환경 변수 편집** → 사용자 변수 **새로 만들기**. 어떤 방법이든 **이미 실행 중인 Claude Code·데스크톱 앱은 새 값을 모르므로** 다시 시작합니다.

`.env.example`에는 **변수 이름만** 추가합니다.

```bash
EDUHUB_DB_READONLY_URL=
```

> ✅ **확인**: `psql "$EDUHUB_DB_READONLY_URL" -c "DELETE FROM enrollment"`가 **권한 오류**로 실패해야 합니다. 쓰기가 막혀 있어야 MCP를 연결합니다.
>
> Windows에 `psql`이 없다면 DB 컨테이너 안의 `psql`로 확인합니다 (PowerShell·Git Bash 모두 같은 명령):
> ```powershell
> docker exec -it eduhub-postgres psql "postgresql://eduhub_readonly:각자-정한-비밀번호@localhost:5432/eduhub" -c "DELETE FROM enrollment"
> ```
> 컨테이너 안에서의 `localhost`는 컨테이너 자신이라 그대로 접속됩니다. PowerShell에 설정한 값을 쓰려면 `psql $env:EDUHUB_DB_READONLY_URL -c "…"`(로컬에 psql이 있을 때).

### 접속 정보를 `.env`로 관리하려면

`.mcp.json`의 `${EDUHUB_DB_READONLY_URL}`은 **Claude Code를 실행한 프로세스의 환경 변수**에서 값을 가져옵니다. 프로젝트의 `.env` 파일을 자동으로 읽지는 않으므로, `.env`를 쓰려면 **`.env`를 환경 변수로 올려 주는 단계**가 필요합니다.

```bash
# .env (프로젝트 루트, .gitignore에 반드시 포함. 저장소에는 값 없는 .env.example만)
EDUHUB_DB_READONLY_URL=postgresql://eduhub_readonly:각자-정한-비밀번호@localhost:5432/eduhub
# MariaDB 팀이라면
EDUHUB_DB_RO_USER=eduhub_readonly
EDUHUB_DB_RO_PASSWORD=각자-정한-비밀번호
```

**방법 1. Claude Code를 시작할 때 `.env`를 올리기**

```bash
# macOS·Linux·WSL·Git Bash: .env의 KEY=VALUE를 모두 환경 변수로 올린 뒤 실행
set -a; source .env; set +a
claude
```

```powershell
# Windows PowerShell: 이 창(프로세스)에만 올린 뒤 실행. 빈 줄과 # 주석은 건너뛰고, 값의 따옴표는 벗긴다
Get-Content .env | ForEach-Object {
  if ($_ -match '^\s*([^#=\s][^=]*)=(.*)$') {
    $value = $matches[2].Trim().Trim('"').Trim("'")
    [Environment]::SetEnvironmentVariable($matches[1].Trim(), $value, 'Process')
  }
}
claude
```

> 💡 자주 쓴다면 위 PowerShell 코드를 `scripts\load-env.ps1`로 저장하고 `. .\scripts\load-env.ps1; claude`처럼 실행합니다 (앞의 `. `은 "현재 창에 적용"하는 dot-sourcing).

**방법 2. 폴더에 들어가면 자동으로: direnv** (macOS·Linux·WSL)
- `direnv`를 설치하고 프로젝트 루트의 `.envrc`에 `dotenv` 한 줄을 쓴 뒤 `direnv allow` → 그 폴더에서 실행하는 `claude`가 `.env` 값을 받습니다
- 네이티브 Windows에서는 방법 1이나 사용자 환경 변수를 씁니다

**방법 3. MCP 서버가 `.env`를 직접 읽기**
- `mcp-server-mysql`(MariaDB 예시)은 시작할 때 `dotenv`로 **자기가 실행된 폴더의 `.env`**를 읽습니다 (v2.0.9 코드 확인). 이 경우 `.env`에 `MYSQL_HOST`, `MYSQL_USER`, `MYSQL_PASS`, `MYSQL_DB`를 직접 쓰면 됩니다. 단, Claude Code가 서버를 어느 폴더에서 실행하는지는 확인이 필요하고, `.env`에 `ALLOW_INSERT_OPERATION=true` 같은 값이 섞이면 **쓰기가 허용될 수 있으니** 주의합니다
- 직접 만든 Node 서버([나만의 MCP 서버](../examples/32-custom-mcp-server-mariadb.md))는 Node 20.6 이상의 `node --env-file=.env dist/index.js`로 읽을 수 있습니다

| 실행 방법 | 권장 |
|---|---|
| 터미널(macOS·Linux·WSL)에서 `claude` | `.env` + direnv, 또는 `set -a; source .env; set +a` |
| 터미널(Windows PowerShell)에서 `claude` | `.env` + 방법 1의 PowerShell 코드 |
| **데스크톱 앱** (터미널 없이 실행) | **사용자 환경 변수**(Step 7의 Windows ②, macOS는 셸 프로필) 후 앱 다시 시작 |

> ⚠️ `.env`도 셸 프로필도 **디스크에 평문으로 저장**됩니다. `.env`는 `.gitignore`와 4회차 `deny: Read(./.env)`로 커밋·읽기를 막고, 셸 환경 변수는 Claude의 Bash가 `echo`로 볼 수 있다는 점을 기억합니다. 어느 쪽이든 **개발 DB의 읽기 전용 계정**만 두고 운영 접속 정보는 두지 않는 것이 핵심입니다.

---

## Step 8. `.mcp.json`에 DB MCP 연결하기

```bash
git switch main && git pull                  # 훅 PR이 병합된 main
git switch -c feature/fr-07-my-enrollments
```

**어떤 PostgreSQL MCP 서버를 쓰나?** (2026년 11월 기준, 각 저장소 README로 확인)

| 서버 | 실행 방법 | 읽기 전용 | 상태 | EduHub |
|---|---|---|---|---|
| **Postgres MCP Pro** (`postgres-mcp`, crystaldba) | `uvx` · `pipx` · Docker | ✅ `--access-mode=restricted`: 읽기 전용 트랜잭션 + 실행 시간 제한 | 관리 중 (최신 v0.3.0) | ⭐ **이 교재의 기준** |
| Google MCP Toolbox (`@toolbox-sdk/server`) | `npx` | 서버 자체 보장은 README에 없음 → DB 계정 권한에 의존 | 관리 중 | 대안 (아래 참고) |
| `@modelcontextprotocol/server-postgres` | `npx` | 읽기 전용 트랜잭션 | ❌ **2025년 5월 보관, 관리 종료** | 쓰지 않음 |

Postgres MCP Pro를 고른 이유: **서버가 스스로 읽기 전용을 보장하는 모드**가 있어 Step 7의 읽기 전용 계정과 **이중으로** 막을 수 있고, 테이블 구조 조회·실행 계획(`explain_query`)·인덱스 분석 도구가 있어 FR-07의 인덱스 확인(NFR-02)에도 쓸 수 있습니다.

**① 준비: `uv` 설치** (`uvx`가 `postgres-mcp`를 내려받아 실행합니다. Python은 uv가 알아서 준비합니다)

```bash
# macOS
brew install uv
# Windows (PowerShell)
winget install --id=astral-sh.uv -e
# Linux·WSL (설치 스크립트는 https://docs.astral.sh/uv/ 의 안내를 따릅니다)

uvx --version     # 설치 확인
```

**② 프로젝트 루트에 `.mcp.json` 만들기**: 파일 위치는 저장소 루트의 `eduhub/.mcp.json`입니다 (`.claude/` 안이 아님, [2-2의 위치 그림](#2-2-연결-범위와-mcpjson)). 터미널에서는 `claude mcp add --scope project …`로도 만들 수 있습니다

```json
{
  "mcpServers": {
    "eduhub-db": {
      "type": "stdio",
      "command": "uvx",
      "args": ["postgres-mcp", "--access-mode=restricted"],
      "env": {
        "DATABASE_URI": "${EDUHUB_DB_READONLY_URL}"
      }
    }
  }
}
```

| 항목 | 의미 |
|---|---|
| `"command": "uvx"`, `"args": ["postgres-mcp", …]` | PyPI의 `postgres-mcp` 패키지를 받아 MCP 서버로 실행 |
| `--access-mode=restricted` | ⭐ **읽기 전용 트랜잭션**으로만 SQL 실행, 실행 시간 제한. README 예시는 `unrestricted`로 되어 있으니 **반드시 바꿔서** 씁니다 |
| `"DATABASE_URI"` | 이 서버가 읽는 접속 문자열 환경 변수 이름 (README 기준) |
| `"${EDUHUB_DB_READONLY_URL}"` | Step 7에서 각자 `.env`·셸에 둔 **읽기 전용 계정** URL을 넣는다. 비밀번호는 파일에 쓰지 않는다 |

> 💡 `${EDUHUB_DB_READONLY_URL}`은 **Claude Code를 실행한 셸의 환경 변수**에서 읽습니다. `.env` 파일에만 적어 두면 Claude Code가 읽지 못할 수 있으니, 셸 프로필(`~/.zshrc`, `~/.bashrc`, Windows는 사용자 환경 변수)에 넣거나 Claude Code를 시작하기 전에 `export`합니다. Windows 명령과 `.env`로 관리하는 방법은 Step 7의 Windows 예시와 [접속 정보를 .env로 관리하려면](#접속-정보를-env로-관리하려면)을 봅니다.

<details>
<summary>다른 실행 방법: Docker로 실행 (uv를 설치하고 싶지 않을 때)</summary>

```json
{
  "mcpServers": {
    "eduhub-db": {
      "type": "stdio",
      "command": "docker",
      "args": ["run", "-i", "--rm", "-e", "DATABASE_URI", "crystaldba/postgres-mcp", "--access-mode=restricted"],
      "env": {
        "DATABASE_URI": "${EDUHUB_DB_READONLY_URL}"
      }
    }
  }
}
```

- README에 따르면 이 이미지는 접속 주소의 `localhost`를 컨테이너 밖의 호스트 주소(macOS·Windows는 `host.docker.internal`, Linux는 `172.17.0.1`)로 자동으로 바꿔 줍니다. 그래서 `EDUHUB_DB_READONLY_URL`은 그대로 `localhost`로 둡니다

</details>

<details>
<summary>대안: Google MCP Toolbox (npx로 실행)</summary>

```json
{
  "mcpServers": {
    "eduhub-db": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@toolbox-sdk/server", "--prebuilt=postgres", "--stdio"]
    }
  }
}
```

- README는 접속 정보를 환경 변수로 넣으라고만 안내하며, 변수 이름(`POSTGRES_HOST`, `POSTGRES_USER` 등)은 Prebuilt Tools Reference 문서에서 확인해야 합니다 (이 교재에서는 확인하지 못함)
- 서버가 읽기 전용을 따로 보장한다는 설명이 README에 없으므로, **Step 7의 읽기 전용 계정이 유일한 방어선**입니다

</details>

**③ 조회 도구는 매번 묻지 않게 허용** (`.claude/settings.json`의 `permissions.allow`에 추가)

Postgres MCP Pro의 도구는 `list_schemas`, `list_objects`, `get_object_details`, `execute_sql`, `explain_query`, `get_top_queries`, `analyze_workload_indexes`, `analyze_query_indexes`, `analyze_db_health`입니다 (README 기준). Claude Code에서는 `mcp__<서버 이름>__<도구 이름>`으로 부릅니다. 조회에 쓰는 것만 허용합니다.

```json
{
  "permissions": {
    "allow": [
      "mcp__eduhub-db__list_schemas",
      "mcp__eduhub-db__list_objects",
      "mcp__eduhub-db__get_object_details",
      "mcp__eduhub-db__execute_sql",
      "mcp__eduhub-db__explain_query"
    ]
  }
}
```

> 💡 `execute_sql`을 묻지 않고 허용해도 되는 이유: **서버의 restricted 모드 + DB 계정의 읽기 전용 권한**이라는 두 겹의 보호가 있기 때문입니다. 둘 중 하나라도 빠졌다면 허용 목록에 넣지 않습니다. 실제 도구 이름은 연결 후 `/mcp`에서 확인하고 다르면 맞춰 고칩니다.

**MCP 도구 이름은 어디서 오나?**

권한 규칙에 쓰는 `mcp__eduhub-db__execute_sql` 같은 이름은 **세 부분을 Claude Code가 합친 것**입니다. 우리 저장소 어디에도 도구가 "정의"되어 있지 않습니다.

```text
mcp__eduhub-db__execute_sql
│    │          └ ③ 도구 이름: MCP 서버 프로그램(패키지 코드)이 정의
│    └ ② 서버 이름: 내가 .mcp.json의 mcpServers 아래에 붙인 키 이름
└ ① 접두사: Claude Code의 명명 규칙 (MCP 도구라는 표시)
```

| 부분 | 정의된 곳 | 바꿀 수 있나 |
|---|---|---|
| ① `mcp__` | Claude Code 규칙 | ✕ |
| ② `eduhub-db` | `.mcp.json`의 `"eduhub-db": { … }` 키 | ○ 바꾸면 도구 이름도 바뀐다 (권한 규칙도 같이 고칠 것) |
| ③ `execute_sql`, `mysql_query` 등 | **MCP 서버 패키지의 소스 코드** | ✕ 서버 작성자가 정한다 |

③은 **서버가 실행된 뒤에** Claude Code에 전달됩니다.

```mermaid
sequenceDiagram
    participant CC as Claude Code
    participant S as MCP 서버 (예: mcp-server-mysql)
    participant DB as 개발 DB
    CC->>S: .mcp.json의 command 실행 (npx/uvx …)
    S->>DB: 시작하면서 접속 확인
    CC->>S: tools/list (도구 목록 요청)
    S-->>CC: [{ name: "mysql_query", description, inputSchema }]
    Note over CC: 이름 앞에 mcp__eduhub-db__ 를 붙여 Claude에게 제공
```

예를 들어 MariaDB 예시의 `@benborla29/mcp-server-mysql`은 소스 코드(`index.ts`)에 `mysql_query` **하나만** 고정으로 등록합니다. 환경 변수로 이름이나 개수가 바뀌지 않고, 쓰기 허용 옵션을 켜면 설명 문구만 바뀝니다 (2026년 11월, GitHub 소스 확인).

**③ 도구 이름(`execute_sql` 등)은 어디서 확인하나?**

도구 이름은 우리 저장소 파일에 없으므로, **서버가 실제로 보내 준 목록**을 보거나 **서버의 문서·코드**를 봅니다. 권한 규칙에 쓸 이름은 ①을 기준으로 정합니다.

| 방법 | 어떻게 | 보이는 것 | 특징 |
|---|---|---|---|
| ① ⭐ Claude Code의 `/mcp` | 대화창에서 `/mcp` → `eduhub-db` 선택 → **View tools**(도구 보기) | 지금 연결된 서버가 보낸 **실제 도구 목록**과 각 도구의 설명 | 설치한 버전 그대로라 가장 정확하다. 서버가 `connected` 상태여야 보인다 |
| ② Claude에게 묻기 | "eduhub-db MCP 서버가 제공하는 도구 이름과 한 줄 설명만 표로 보여줘. 도구는 실행하지 마" | ①과 같은 목록 (Claude가 받은 도구 정보) | 간편하지만 Claude가 정리한 결과이므로 권한 규칙에 옮길 때는 ①로 다시 확인 |
| ③ 서버의 README·문서 | Postgres MCP Pro는 README의 도구 표(`list_schemas`, `execute_sql` …), `mcp-server-mysql`은 README·소스의 도구 설명 | 서버 작성자가 문서에 적은 목록 | **연결 전에** 미리 볼 수 있다. 버전이 바뀌면 실제와 다를 수 있다 |
| ④ 서버 소스 코드 | 도구를 등록하는 부분을 찾는다: TypeScript SDK는 `registerTool("이름", …)`(예전 코드는 `server.tool(…)`), Python(FastMCP)은 `@mcp.tool` 데코레이터가 붙은 함수 이름, Java SDK는 `Tool.builder("이름", …)` | 이름이 정해지는 바로 그 코드 | 7-4에서 직접 만든 서버의 도구 이름이 정해지는 곳과 같다 |
| ⑤ MCP Inspector | `npx @modelcontextprotocol/inspector uvx postgres-mcp --access-mode=restricted` (환경 변수 `DATABASE_URI` 설정 후) → 브라우저 화면의 **Tools** 탭 | `tools/list` 응답 원문 (이름, 설명, 입력 스키마) | Claude 없이 서버만 띄워 확인. 도구를 직접 호출해 볼 수도 있다 |

- 이름이 정해지는 원리는 7-4 ③에서 실제 메시지로 볼 수 있습니다: 클라이언트가 `tools/list`를 보내면 서버가 `{"name": "get_remaining_seats", …}`처럼 도구 목록을 돌려주고, Claude Code가 그 `name` 앞에 `mcp__<서버 이름>__`을 붙입니다 ([7-4 ③ 동작 순서](#7-4-나만의-mcp-서버-만들기-mariadb-조회-도구))
- `/mcp`에서 서버가 `failed`이면 도구 목록이 오지 않습니다. 이때는 ③(README)으로 이름을 확인해 두고, 연결 문제를 먼저 해결합니다 (5-7 자주 겪는 문제)
- 권한 규칙에 쓴 이름이 실제 도구 이름과 한 글자라도 다르면 **오류 없이 무시**되고, 그 도구를 쓸 때마다 승인 질문이 나옵니다. 승인 질문이 계속 나오면 이름부터 ①과 대조합니다

> 이 교재 환경에서는 Postgres MCP Pro를 실행해 보지 못해(`uvx` 실행이 막힘) 도구 목록은 README 기준입니다. `/mcp`의 메뉴 이름은 Claude Code 버전에 따라 다를 수 있습니다. ⑤의 Inspector 명령도 직접 실행해 보지 못했습니다(7-4에서는 직접 만든 서버로 `tools/list`를 확인함).

> 💡 **권한 규칙은 도구를 만들지 않습니다.** `permissions.allow`에 이름을 적어도 도구가 생기지 않습니다. 서버가 실행되어 도구 목록을 보내야 도구가 있고, 권한 규칙은 그 도구를 "묻지 않고 써도 된다"고 정할 뿐입니다.

**⚠️ 자주 하는 오타: 오류 없이 무시됩니다**

```json
{ "permission": { "allow": ["mcp_eduhub-db_mysql_query"] } }
```

| 잘못된 곳 | 올바른 형태 | 틀리면 |
|---|---|---|
| `"permission"` | **`"permissions"`** (s) | 설정 항목 전체가 무시됨 |
| `mcp_eduhub-db_mysql_query` (밑줄 1개) | **`mcp__eduhub-db__mysql_query`** (밑줄 **2개**씩) | 어떤 도구와도 맞지 않음 |

```json
{
  "permissions": {
    "allow": ["mcp__eduhub-db__mysql_query"]
  }
}
```

틀려도 오류가 나지 않고 **매번 승인을 묻는 상태로 남아서** 알아차리기 어렵습니다. 고친 뒤 `/permissions`에서 규칙이 보이는지 확인합니다.

**도구가 안 보일 때 점검 순서**

```mermaid
flowchart TD
    A["/mcp 를 연다"] --> B{"eduhub-db가<br/>목록에 있나?"}
    B -- "없음" --> B1["① 서버가 등록되지 않음<br/>.mcp.json이 프로젝트 루트에 있나?<br/>승인 질문을 거부했나? → claude mcp reset-project-choices<br/>Claude Code를 그 폴더에서 다시 시작"]
    B -- "있음" --> C{"상태가<br/>connected?"}
    C -- "failed / 연결 중" --> C1["② 서버가 시작 직후 종료됨<br/>대부분 DB 접속 실패 (아래 직접 실행으로 확인)"]
    C -- "connected" --> D["③ eduhub-db 선택 → 도구 목록 보기<br/>여기 보이는 이름이 정답"]
```

| 단계 | 확인 | 해결 |
|---|---|---|
| ① 서버가 목록에 없음 | `.mcp.json` 위치(프로젝트 루트), JSON 문법(`jq . .mcp.json`), 승인 여부 | `claude mcp reset-project-choices` 후 다시 시작하고 승인 |
| ② failed | 서버는 **시작하면서 DB 접속을 시험하고, 실패하면 종료**한다 (mcp-server-mysql 소스 기준). 서버가 꺼지면 도구 목록도 없다 | DB 컨테이너 실행, 호스트·포트, 환경 변수(`EDUHUB_DB_RO_USER`, `EDUHUB_DB_RO_PASSWORD`)가 **Claude Code를 실행한 셸**에 있는지, 변수 이름 `MYSQL_PASS` 확인 |
| ② failed (첫 실행) | `npx`/`uvx`가 패키지를 내려받느라 연결 제한 시간을 넘김 | `/mcp`에서 다시 연결, 또는 `MCP_TIMEOUT`을 늘림 |
| ③ connected인데 Claude가 도구를 모른다고 함 | 도구 검색이 켜져 있으면 도구 정의는 필요할 때 불러온다 | "eduhub-db의 도구 목록을 알려줘"라고 물어본다 |

**서버를 직접 실행해 원인 보기** (Claude Code 없이, 내 터미널에서)

```bash
# MariaDB 예시 (mcp-server-mysql). 접속에 성공하면 "Database connection test successful",
# 실패하면 "Fatal error during server startup"과 원인이 출력된다. 확인 후 Ctrl+C
MYSQL_HOST=127.0.0.1 MYSQL_PORT=3306 MYSQL_DB=eduhub \
MYSQL_USER="$EDUHUB_DB_RO_USER" MYSQL_PASS="$EDUHUB_DB_RO_PASSWORD" \
  npx -y @benborla29/mcp-server-mysql
```

```powershell
# Windows PowerShell: 이 창에만 설정하고 실행 (확인 후 Ctrl+C)
$env:MYSQL_HOST = "127.0.0.1"; $env:MYSQL_PORT = "3306"; $env:MYSQL_DB = "eduhub"
$env:MYSQL_USER = $env:EDUHUB_DB_RO_USER; $env:MYSQL_PASS = $env:EDUHUB_DB_RO_PASSWORD
npx -y @benborla29/mcp-server-mysql
```

> 💡 MySQL 8 계열에서 인증 방식 오류가 나면 이 패키지의 문서(docs/TROUBLESHOOTING.md)는 `caching_sha2_password` 지원 확인이나 `mysql_native_password` 계정 사용을 안내합니다. MariaDB에서의 동작은 이 문서에 따로 적혀 있지 않습니다.


**④ 연결하고 확인하기**

1. Claude Code를 다시 시작하면 `.mcp.json`의 서버를 쓸지 묻는 질문이 나옵니다 → 내용을 확인하고 승인
2. `/mcp`로 `eduhub-db`가 연결됨(connected) 상태인지 확인

> ✅ **확인**: `eduhub-db로 테이블 목록만 보여줘`라고 했을 때 `course`, `course_session`, `enrollment`, `employee`(팀 스키마에 맞게)가 나오면 됩니다.

### 개발 DB가 MariaDB인 팀 (Step 7~8의 MariaDB 버전)

EduHub의 기준 DB는 PostgreSQL이지만, 회사 표준이 MariaDB라면 Step 7~8을 아래처럼 바꿉니다. **원칙은 같습니다: 개발 DB + 읽기 전용 계정 + 서버의 읽기 전용 설정, 비밀번호는 환경 변수로.**

**MariaDB용 MCP 서버 선택** (2026년 11월 기준, 각 저장소 README로 확인)

| 서버 | 실행 방법 | 읽기 전용 | 상태 | EduHub |
|---|---|---|---|---|
| **mcp-server-mysql** (`@benborla29/mcp-server-mysql`) | `npx` | ✅ 기본값이 읽기 전용: 쓰기 허용 옵션(`ALLOW_INSERT/UPDATE/DELETE/DDL_OPERATION`)이 모두 `false`, 읽기 전용 트랜잭션 강제 | 관리 중 | ⭐ **`.mcp.json` 예시 기준**. MySQL 서버이며 README에 MariaDB 지원은 명시되어 있지 않음 → **수업 전 연결 확인 필요** |
| **MariaDB 공식 플러그인** (`mariadb/ai-plugins`의 `dev` 플러그인) | Claude Code 플러그인. MariaDB Shell에 들어 있는 MCP 서버를 실행 | 서버 설정이 아니라 **읽기 전용 전용 계정**으로 보장하라고 안내 | 관리 중 (MariaDB 공식) | 대안 (6회차 플러그인과 연결) |
| `MariaDB/mcp` (이전 공식 서버) | git clone + `uv run` | `MCP_READ_ONLY` (기본 `true`, 화이트리스트 방식) | ❌ **2026년 11월 3일 지원 중단**, ai-plugins로 이동 | 쓰지 않음 |

**① 읽기 전용 계정 만들기 (Step 7의 MariaDB 버전, 사람이 직접)**

계정을 만드는 SQL, PostgreSQL과의 대응, 확인 방법, 자주 막히는 곳은 Step 7의 [MariaDB용 읽기 전용 계정 SQL](#mariadb용-읽기-전용-계정-sql)에 있습니다. 핵심은 **SELECT 권한만, `eduhub.*`처럼 필요한 스키마에만** 주는 것입니다.

```bash
# macOS·Linux·WSL: 셸 프로필(~/.zshrc, ~/.bashrc)에 추가 (.env로 관리하려면 "접속 정보를 .env로 관리하려면")
export EDUHUB_DB_RO_USER="eduhub_readonly"
export EDUHUB_DB_RO_PASSWORD="각자-정한-비밀번호"
```

```powershell
# Windows PowerShell: 내 계정에 영구 저장 → Claude Code·데스크톱 앱 다시 시작
[Environment]::SetEnvironmentVariable("EDUHUB_DB_RO_USER", "eduhub_readonly", "User")
[Environment]::SetEnvironmentVariable("EDUHUB_DB_RO_PASSWORD", "각자-정한-비밀번호", "User")
# 지금 창에서 바로 쓰려면: $env:EDUHUB_DB_RO_USER = "eduhub_readonly"; $env:EDUHUB_DB_RO_PASSWORD = "각자-정한-비밀번호"
```

`.env.example`에는 **변수 이름만** 추가합니다: `EDUHUB_DB_RO_USER=`, `EDUHUB_DB_RO_PASSWORD=`

> ✅ **확인**: `mariadb -h 127.0.0.1 -u eduhub_readonly -p eduhub -e "DELETE FROM enrollment"`가 `DELETE command denied` 같은 **권한 오류**로 실패해야 합니다.
>
> Windows에 `mariadb` 클라이언트가 없다면 DB 컨테이너 안에서 확인합니다 (PowerShell·Git Bash 모두 같은 명령):
> ```powershell
> docker exec -it eduhub-mariadb mariadb -u eduhub_readonly -p eduhub -e "DELETE FROM enrollment"
> ```

**② `.mcp.json` (mcp-server-mysql)**

```json
{
  "mcpServers": {
    "eduhub-db": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@benborla29/mcp-server-mysql"],
      "env": {
        "MYSQL_HOST": "127.0.0.1",
        "MYSQL_PORT": "3306",
        "MYSQL_USER": "${EDUHUB_DB_RO_USER}",
        "MYSQL_PASS": "${EDUHUB_DB_RO_PASSWORD}",
        "MYSQL_DB": "eduhub"
      }
    }
  }
}
```

| 항목 | 의미 |
|---|---|
| `MYSQL_HOST`, `MYSQL_PORT`, `MYSQL_DB` | 개발 DB 주소와 스키마. 비밀이 아니므로 파일에 써도 된다 (운영 주소는 절대 금지) |
| `MYSQL_USER`, `MYSQL_PASS` | **읽기 전용 계정**. 환경 변수로 넣는다. 변수 이름이 `MYSQL_PASSWORD`가 아니라 **`MYSQL_PASS`**인 점에 주의 (README 기준) |
| `ALLOW_INSERT_OPERATION` 등 | **쓰지 않는다.** 기본값 `false`가 읽기 전용. `true`로 바꾸면 Claude가 데이터를 바꿀 수 있게 된다 |
| `MYSQL_DISABLE_READ_ONLY_TRANSACTIONS` | **쓰지 않는다.** 기본값 `false`(읽기 전용 트랜잭션 강제) |

조회 도구는 `mysql_query` 하나입니다 (소스 코드 기준, 이름이 어디서 오는지는 위의 [MCP 도구 이름은 어디서 오나?](#step-8-mcpjson에-db-mcp-연결하기) 참고). 권한 허용은 다음과 같습니다.

```json
{
  "permissions": {
    "allow": ["mcp__eduhub-db__mysql_query"]
  }
}
```

> ⚠️ README의 지원 대상은 "MySQL 5.7+"이고 MariaDB는 명시되어 있지 않습니다. MariaDB는 MySQL 프로토콜과 호환되어 보통 연결되지만, 이 교재에서는 직접 확인하지 못했습니다. **수업 전에 `/mcp` 연결과 `eduhub-db로 테이블 목록만 보여줘`를 한 번 확인**하고, 문제가 있으면 아래 공식 플러그인을 씁니다.

<details>
<summary>대안: MariaDB 공식 Claude Code 플러그인 (mariadb/ai-plugins)</summary>

MariaDB가 직접 관리하는 방식입니다. `.mcp.json`에 서버를 적는 대신 **플러그인**을 설치하고, 접속 정보는 MariaDB Shell에 등록합니다 (README 기준).

```text
/plugin marketplace add mariadb/ai-plugins
/plugin install dev@mariadb
```

```bash
# 컴퓨터마다 한 번: MariaDB Shell에 접속(연결)을 등록. ①에서 만든 읽기 전용 계정을 쓴다
mariadb-shell -- mcp setup
```

| 알아 둘 점 | 내용 |
|---|---|
| 첫 실행 | 플러그인이 MariaDB Shell을 내려받아 압축을 푼다. 30초 연결 제한을 넘기면 `~/.claude/settings.json`에 `{ "env": { "MCP_TIMEOUT": "120000" } }` |
| 비밀번호 | `.mcp.json`이나 환경 변수가 아니라 MariaDB Shell의 비밀 저장소에 저장 |
| 읽기 전용 | 서버 옵션이 아니라 **권한이 최소인 전용 계정**으로 보장하라고 안내한다 → ①의 계정 사용 |
| 도구 | 43개(`db.*`, `util.*`, `sandbox.*` 등). 조회는 `db.execute_sql` 등. Claude Code에서의 실제 이름은 `/mcp`에서 확인 |
| 함께 들어오는 훅 | Bash에서 `mariadb`/`mysql` CLI로 SQL을 실행하면 막는 PreToolUse 훅이 포함 (`MARIADB_PLUGIN_ALLOW_CLI=1`로 해제) |
| 라이선스 | GPL-2.0 |

> 💡 도구가 많은 플러그인은 그만큼 컨텍스트와 승인 관리 부담이 큽니다. 설치 전에 플러그인 내용(훅, MCP 설정)을 읽어 보는 것은 6회차 "플러그인 설치 전 검토"의 실습이기도 합니다.

</details>

> 💡 백엔드도 MariaDB를 쓴다면 JDBC 드라이버(`org.mariadb.jdbc:mariadb-java-client`), Flyway의 MariaDB 지원 모듈, 테스트의 `MariaDBContainer`(Testcontainers)를 함께 맞춰야 합니다. 이 부분은 Plan 모드로 Claude와 계획을 세워 진행합니다.


---

## Step 9. MCP로 스키마와 데이터 확인하기

구현 전에 **실제 DB가 엔티티와 같은지**부터 봅니다. 결과는 작게 요청합니다.

```text
eduhub-db로 다음만 조회해줘. 행 데이터는 최대 5건까지만.
1. enrollment 테이블의 컬럼, 타입, 인덱스
2. status별 건수
3. 이 결과와 @backend/src/main/java/com/example/eduhub/enrollment/Enrollment.java 를 비교해서 다른 점
```

| 확인할 것 | 왜 |
|---|---|
| `enrollment(employee_id, status)`에 인덱스가 있는가? | FR-07은 "내 신청 내역"을 사원별로 조회한다 → 없으면 Flyway로 추가 (NFR-02) |
| 상태 값이 `CONFIRMED`, `WAITING`, `CANCELED`, `COMPLETED`인가? | 2회차 도메인 모델과 일치하는지 |
| 엔티티와 컬럼이 다른 곳은? | 다르면 마이그레이션이 누락됐거나 엔티티가 틀렸다 |

> 💡 MCP 결과도 컨텍스트에 들어갑니다. "전부 보여줘"가 아니라 **"건수만", "5건만"**으로 요청하는 것이 토큰 습관입니다. 조회가 끝나면 이 세션은 구현 단계로 넘어가기 전에 `/compact 스키마 확인 결과만 남겨줘`로 정리합니다.

---

## Step 9-1. 인증 기반 만들기 (NFR-03) — FR-07 전에

4회차에서 `employeeId`를 서비스 인자로 받고 `TODO(NFR-03)`를 남겼습니다. FR-07은 **"로그인한 사원 본인"**의 내역이고 FR-08은 **"관리자만"** 쓰는 기능이므로, 지금 인증을 붙입니다. 로그인 화면은 이 회차의 [6장 따라하기 ③](#6-따라하기-③-로그인과-화면-완성-하나의-react-앱)에서 만듭니다.

**먼저 사람이 정할 것** (Claude에게 맡기지 않는 설계 결정)

| 결정 | EduHub의 선택 | 이유 |
|---|---|---|
| 인증 방식 | **JWT(Bearer 토큰)**, 서버는 세션을 저장하지 않음(stateless) | 6회차에 shell·course-mfe·enrollment-mfe 여러 앱이 같은 API를 부른다. 요청마다 `Authorization` 헤더 하나로 확인 |
| 로그인 방법 | **교육용 개발 로그인**: 사번 + 비밀번호(BCrypt 해시) | 실제 회사는 사내 SSO(OIDC 등)를 쓴다. 토큰을 받은 **이후** 구조(검증, 사용자 꺼내기, 역할)는 같다 |
| 토큰 서명 | HS256 공유 비밀키, **환경 변수** `EDUHUB_JWT_SECRET` (32바이트 이상) | 비밀은 저장소에 두지 않는다 (NFR-03). 운영은 비밀 저장소에서 주입 |
| 공개 범위 | 로그인·과정 조회(GET)는 누구나. 과정 등록·수정·회차 추가·종료(3회차 FR-03, `/api/v1/courses/**`의 GET 외)와 `/api/admin/**`는 ADMIN. 나머지는 로그인 필요 | FR-01·02는 둘러보기, 신청부터 로그인. 3회차에서 "인증은 나중에"로 미룬 FR-03의 관리자 구분을 여기서 채운다 |
| 남의 신청 취소 | **404** (존재 여부를 숨김) | 8회차 보안 점검과 같은 규칙 |
| 사용자 정보를 꺼내는 곳 | `CurrentUser.from(jwt)` **한 곳** | 요청 본문의 `employeeId`를 믿지 않는다 (2회차 2-3 사례 ④). 7회차의 사내 `CurrentUser` 규칙과 이어진다 |

**지시** (새 세션, Plan 모드)

```text
[목표] NFR-03 인증 기반: 개발용 로그인으로 JWT를 발급하고, 모든 API에서 토큰으로 사용자를 확인한다. 4회차의 TODO(NFR-03)를 해결한다.
[위치] backend/ (새 패키지 auth), build.gradle.kts, application.yml, db/migration
[범위]
- 의존성: spring-boot-starter-security, spring-boot-starter-oauth2-resource-server (JWT 검증·발급에 Spring 기본 기능 사용, 직접 만든 필터 금지)
- Flyway 다음 버전: employee에 employee_no(유일), password_hash, role(EMPLOYEE|ADMIN) 추가
- POST /api/auth/login {employeeNo, password} → {accessToken, expiresIn, user}. 틀리면 사번·비밀번호 구분 없이 401
- GET /api/me → 현재 사용자
- JWT: HS256, 비밀키는 환경 변수 EDUHUB_JWT_SECRET, 만료 60분, sub=사원 ID, role 클레임
- 권한: POST /api/auth/login·GET /api/v1/courses/** 공개, 그 밖의 /api/v1/courses/**(FR-03 등록·수정·회차·종료)와 /api/admin/** 는 ADMIN, 나머지 인증 필요. CSRF 끔(헤더 토큰), 세션 STATELESS
- CurrentUser.from(Jwt) 하나로 사용자를 꺼내고, 기존 신청·취소 컨트롤러의 TODO(NFR-03)를 이것으로 바꾼다
- dev 프로필에서만 개발용 계정 3개(사원 2, 관리자 1) 생성
- 테스트: 로그인 성공·실패, 토큰 없음·위조 401, 신청자=토큰의 사원, 남의 신청 취소 404, 과정 등록 401/403/201, 관리자 API 403
[하지 않는 것] 회원가입, 비밀번호 변경, 리프레시 토큰, FR-07·08 API(Step 10·11), 프론트 화면(6장)
[완료 기준] ./gradlew test 통과, 아래 curl 확인
[진행 방식] 바꿀 파일 목록과 SecurityFilterChain 설정을 먼저 보여줘
```

**계획 검토 포인트**
- `OncePerRequestFilter`를 직접 만들어 토큰을 파싱하는 계획이면, Spring Security의 리소스 서버(`oauth2ResourceServer().jwt()`)를 쓰게 합니다 (검증 코드를 직접 만들면 만료·서명 검사를 빠뜨리기 쉽다)
- 비밀키가 `application.yml`에 **값으로** 들어가 있으면 거절합니다 (환경 변수만)
- 로그인 실패 메시지가 "없는 사번입니다"처럼 **어느 쪽이 틀렸는지** 알려 주면 고치게 합니다
- 3회차 FR-03의 쓰기 API(`POST`·`PUT /api/v1/courses/**`)가 `anyRequest().authenticated()`에 걸려 **사원 토큰으로도 과정을 등록할 수 있게** 되어 있지 않은지 봅니다. `GET`만 공개하고 나머지는 `ADMIN`이어야 합니다 (실제로 이 교재의 첫 설정에서 빠뜨렸던 구멍)

**기대하는 결과 (핵심 코드)** — 아래 코드는 실제로 빌드하고 테스트한 것입니다 (검증 내용은 이 Step 끝)

```sql
-- db/migration/V{다음 번호}__employee_login.sql
ALTER TABLE employee ADD COLUMN employee_no   VARCHAR(20);
ALTER TABLE employee ADD COLUMN password_hash VARCHAR(100);
ALTER TABLE employee ADD COLUMN role          VARCHAR(20) NOT NULL DEFAULT 'EMPLOYEE';
CREATE UNIQUE INDEX ux_employee_no ON employee (employee_no);
```

```java
// auth/SecurityConfig.java — 누가 어떤 API를 부를 수 있는가
package com.example.eduhub.auth;

import org.springframework.context.annotation.*;
import org.springframework.http.HttpMethod;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    SecurityFilterChain api(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())                                   // 쿠키가 아니라 Authorization 헤더로 인증 → CSRF 대상 아님
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.POST, "/api/auth/login").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/courses/**").permitAll()   // 과정 조회는 로그인 없이
                .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/api/health").permitAll()
                .requestMatchers("/api/v1/courses/**").hasRole("ADMIN")               // FR-03 등록·수정·회차 추가·종료 (GET 외)
                .requestMatchers("/api/admin/**").hasRole("ADMIN")                    // FR-08 명단·수료
                .anyRequest().authenticated())                                       // 그 밖(신청·취소·내 내역)은 로그인 필요
            .oauth2ResourceServer(rs -> rs.jwt(jwt -> jwt.jwtAuthenticationConverter(roleConverter())))
            .httpBasic(b -> b.disable())
            .formLogin(f -> f.disable())
            .build();
    }

    /** 토큰의 role 클레임("ADMIN")을 스프링 권한("ROLE_ADMIN")으로 */
    private JwtAuthenticationConverter roleConverter() {
        JwtGrantedAuthoritiesConverter roles = new JwtGrantedAuthoritiesConverter();
        roles.setAuthoritiesClaimName("role");
        roles.setAuthorityPrefix("ROLE_");
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(roles);
        return converter;
    }

    @Bean PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

```java
// auth/JwtConfig.java
package com.example.eduhub.auth;

import com.nimbusds.jose.jwk.source.ImmutableSecret;
import java.nio.charset.StandardCharsets;
import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.*;
import org.springframework.security.oauth2.jose.jws.MacAlgorithm;
import org.springframework.security.oauth2.jwt.*;

/** HS256(공유 비밀키) JWT 발급(JwtEncoder)과 검증(JwtDecoder) */
@Configuration
public class JwtConfig {
    private final SecretKey key;

    JwtConfig(@Value("${eduhub.jwt.secret}") String secret) {
        byte[] bytes = secret.getBytes(StandardCharsets.UTF_8);
        if (bytes.length < 32) throw new IllegalStateException("EDUHUB_JWT_SECRET은 32바이트 이상이어야 합니다.");
        this.key = new SecretKeySpec(bytes, "HmacSHA256");
    }

    @Bean JwtEncoder jwtEncoder() { return new NimbusJwtEncoder(new ImmutableSecret<>(key)); }

    @Bean JwtDecoder jwtDecoder() { return NimbusJwtDecoder.withSecretKey(key).macAlgorithm(MacAlgorithm.HS256).build(); }
}
```

<details>
<summary>TokenService.java, AuthController.java, CurrentUser.java, DevDataLoader.java</summary>

```java
// auth/TokenService.java — 토큰 발급
package com.example.eduhub.auth;

import com.example.eduhub.employee.Employee;
import java.time.Duration;
import java.time.Instant;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.oauth2.jose.jws.MacAlgorithm;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.stereotype.Service;

@Service
public class TokenService {
    private final JwtEncoder encoder;
    private final Duration ttl;

    TokenService(JwtEncoder encoder, @Value("${eduhub.jwt.ttl-minutes:60}") long ttlMinutes) {
        this.encoder = encoder; this.ttl = Duration.ofMinutes(ttlMinutes);
    }

    public String issue(Employee e) {
        Instant now = Instant.now();
        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("eduhub")
                .subject(String.valueOf(e.getId()))      // sub = 사원 ID → CurrentUser.employeeId()
                .claim("name", e.getName())
                .claim("role", e.getRole().name())       // EMPLOYEE | ADMIN
                .issuedAt(now)
                .expiresAt(now.plus(ttl))
                .build();
        JwsHeader header = JwsHeader.with(MacAlgorithm.HS256).build();
        return encoder.encode(JwtEncoderParameters.from(header, claims)).getTokenValue();
    }

    public long ttlSeconds() { return ttl.toSeconds(); }
}
```

```java
// auth/AuthController.java
package com.example.eduhub.auth;

import com.example.eduhub.employee.EmployeeRepository;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/api")
public class AuthController {
    public record LoginRequest(@NotBlank String employeeNo, @NotBlank String password) {}
    public record LoginResponse(String accessToken, long expiresIn, CurrentUser user) {}

    private final EmployeeRepository employees;
    private final PasswordEncoder passwordEncoder;
    private final TokenService tokens;

    AuthController(EmployeeRepository employees, PasswordEncoder passwordEncoder, TokenService tokens) {
        this.employees = employees; this.passwordEncoder = passwordEncoder; this.tokens = tokens;
    }

    /** 사번·비밀번호가 맞으면 JWT를 발급. 사번이 없든 비밀번호가 틀리든 같은 401 (어느 쪽이 틀렸는지 알려 주지 않는다) */
    @PostMapping("/auth/login")
    public LoginResponse login(@Valid @RequestBody LoginRequest req) {
        var e = employees.findByEmployeeNo(req.employeeNo())
                .filter(x -> x.getPasswordHash() != null && passwordEncoder.matches(req.password(), x.getPasswordHash()))
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.UNAUTHORIZED, "사번 또는 비밀번호가 올바르지 않습니다."));
        return new LoginResponse(tokens.issue(e), tokens.ttlSeconds(), new CurrentUser(e.getId(), e.getName(), e.getRole().name()));
    }

    /** 새로고침 후 화면이 "지금 누구로 로그인했는지" 확인할 때 */
    @GetMapping("/me")
    public CurrentUser me(@AuthenticationPrincipal Jwt jwt) { return CurrentUser.from(jwt); }
}
```

```java
// auth/CurrentUser.java
package com.example.eduhub.auth;

import org.springframework.security.oauth2.jwt.Jwt;

/** 토큰에서 꺼낸 현재 사용자. 컨트롤러는 이것으로만 사용자를 얻는다 (요청 본문의 ID를 믿지 않는다) */
public record CurrentUser(Long employeeId, String name, String role) {
    public static CurrentUser from(Jwt jwt) {
        return new CurrentUser(Long.valueOf(jwt.getSubject()), jwt.getClaimAsString("name"), jwt.getClaimAsString("role"));
    }
    public boolean isAdmin() { return "ADMIN".equals(role); }
}
```

```java
// auth/DevDataLoader.java (dev 프로필 전용) — 계정 3개, 과정 3개, 이미 시작한 회차와 확정자 2명
package com.example.eduhub.auth;

import com.example.eduhub.course.*;
import com.example.eduhub.employee.*;
import com.example.eduhub.enrollment.EnrollmentService;
import java.time.LocalDate;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Component;

/** dev 프로필에서만: 개발용 계정과 과정 데이터. 운영에는 절대 실리지 않는다 */
@Component
@Profile("dev")
class DevDataLoader implements CommandLineRunner {
    private final EmployeeRepository employees;
    private final CourseRepository courses;
    private final CourseSessionRepository sessions;
    private final EnrollmentService enrollments;
    private final PasswordEncoder encoder;

    DevDataLoader(EmployeeRepository employees, CourseRepository courses, CourseSessionRepository sessions,
                  EnrollmentService enrollments, PasswordEncoder encoder) {
        this.employees = employees; this.courses = courses; this.sessions = sessions;
        this.enrollments = enrollments; this.encoder = encoder;
    }

    @Override
    public void run(String... args) {
        if (employees.count() > 0) return;
        String pw = encoder.encode("dev-password");   // 개발용 공통 비밀번호
        Employee kim = employees.save(new Employee("E1001", "김사원", "개발1팀", pw, Employee.Role.EMPLOYEE));
        Employee lee = employees.save(new Employee("E1002", "이사원", "개발2팀", pw, Employee.Role.EMPLOYEE));
        employees.save(new Employee("A0001", "박관리", "인사팀", pw, Employee.Role.ADMIN));
        Course spring = courses.save(new Course("Spring Boot 입문", "BACKEND", "스프링 부트로 REST API 만들기"));
        Course jpa = courses.save(new Course("JPA 실전", "BACKEND", "N+1과 영속성 컨텍스트"));
        sessions.save(new CourseSession(spring, LocalDate.now().plusDays(7), "본관 3층", 1));   // 정원 1: 대기 흐름 확인용
        sessions.save(new CourseSession(spring, LocalDate.now().plusDays(30), "온라인", 20));
        sessions.save(new CourseSession(jpa, LocalDate.now().plusDays(14), "본관 5층", 10));
        // 이미 시작한 회차와 확정자 2명: 수료 처리(FR-08) 확인용
        Course git = courses.save(new Course("Git 협업 기초", "TOOL", "브랜치, PR, 충돌 해결"));
        CourseSession started = sessions.save(new CourseSession(git, LocalDate.now().minusDays(3), "본관 2층", 10));
        enrollments.enroll(started.getId(), kim.getId());
        enrollments.enroll(started.getId(), lee.getId());
    }
}
```

</details>

**컨트롤러: `TODO(NFR-03)`를 토큰의 사용자로 바꾼다**

```java
// enrollment/EnrollmentController.java — 4회차 컨트롤러의 TODO(NFR-03)를 토큰의 사용자로 바꾼 것
package com.example.eduhub.enrollment;

import com.example.eduhub.auth.CurrentUser;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

@RestController
public class EnrollmentController {
    private final EnrollmentService service;
    EnrollmentController(EnrollmentService service) { this.service = service; }

    /** FR-04: 신청자는 요청 본문이 아니라 토큰에서 (NFR-03) */
    @PostMapping("/api/sessions/{sessionId}/enrollments")
    @ResponseStatus(HttpStatus.CREATED)
    public EnrollmentService.EnrollmentResult enroll(@PathVariable Long sessionId, @AuthenticationPrincipal Jwt jwt) {
        return service.enroll(sessionId, CurrentUser.from(jwt).employeeId());
    }

    /** FR-06: 본인 신청만 취소 */
    @DeleteMapping("/api/enrollments/{enrollmentId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void cancel(@PathVariable Long enrollmentId, @AuthenticationPrincipal Jwt jwt) {
        service.cancel(enrollmentId, CurrentUser.from(jwt).employeeId());
    }
}
```

```yaml
# application.yml (추가)
eduhub:
  jwt:
    secret: ${EDUHUB_JWT_SECRET}        # 32바이트 이상. 저장소에 쓰지 않는다 (환경 변수)
    ttl-minutes: 60
```

```bash
# 비밀키는 각자 만들어 셸 프로필(또는 .env를 셸에 올리는 방법, Step 7 참고)에. .env.example에는 변수 이름만
export EDUHUB_JWT_SECRET="$(openssl rand -base64 48)"
```

```powershell
# Windows PowerShell
$env:EDUHUB_JWT_SECRET = [Convert]::ToBase64String((1..48 | ForEach-Object { Get-Random -Maximum 256 }))
```

**테스트**

```java
// src/test/.../auth/AuthControllerTest.java — 실제로 로그인해 받은 토큰으로 호출
package com.example.eduhub.auth;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import com.example.eduhub.employee.Employee;
import com.example.eduhub.employee.EmployeeRepository;
import com.jayway.jsonpath.JsonPath;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
class AuthControllerTest {
    @Autowired MockMvc mvc;
    @Autowired EmployeeRepository employees;
    @Autowired PasswordEncoder encoder;

    @BeforeEach
    void setUp() {
        employees.save(new Employee("T0001", "테스트사원", "QA", encoder.encode("pw-1234"), Employee.Role.EMPLOYEE));
    }

    private String login(String employeeNo, String password) {
        return """
            {"employeeNo":"%s","password":"%s"}""".formatted(employeeNo, password);
    }

    @Test
    void 사번과_비밀번호가_맞으면_토큰을_발급하고_그_토큰으로_내_정보를_본다() throws Exception {
        String body = mvc.perform(post("/api/auth/login").contentType(MediaType.APPLICATION_JSON).content(login("T0001", "pw-1234")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.user.name").value("테스트사원"))
                .andExpect(jsonPath("$.user.role").value("EMPLOYEE"))
                .andReturn().getResponse().getContentAsString();
        String token = JsonPath.read(body, "$.accessToken");

        mvc.perform(get("/api/me").header("Authorization", "Bearer " + token))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.name").value("테스트사원"));
    }

    @Test
    void 비밀번호가_틀리면_401() throws Exception {
        mvc.perform(post("/api/auth/login").contentType(MediaType.APPLICATION_JSON).content(login("T0001", "wrong")))
                .andExpect(status().isUnauthorized());
    }

    @Test
    void 토큰이_없거나_위조되면_401() throws Exception {
        mvc.perform(get("/api/me")).andExpect(status().isUnauthorized());
        mvc.perform(get("/api/me").header("Authorization", "Bearer abc.def.ghi")).andExpect(status().isUnauthorized());
    }

    @Test
    void 과정_조회는_로그인_없이_된다() throws Exception {
        mvc.perform(get("/api/v1/courses")).andExpect(status().isOk());
    }
}
```

<details>
<summary>TestAuth.java, EnrollmentSecurityTest.java — 토큰 발급 없이 jwt()로 "이 사람으로 로그인한 요청" 만들기</summary>

```java
// src/test/.../TestAuth.java — Step 10·11 테스트도 함께 쓴다
package com.example.eduhub;

import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;

import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.test.web.servlet.request.RequestPostProcessor;

/** 테스트에서 "이 사람으로 로그인한 요청"을 만든다. 토큰 발급 없이 검증된 JWT를 흉내 낸다 */
public final class TestAuth {
    private TestAuth() {}

    public static RequestPostProcessor as(Long employeeId, String role) {
        return jwt().jwt(j -> j.subject(String.valueOf(employeeId)).claim("role", role))
                    .authorities(new SimpleGrantedAuthority("ROLE_" + role));
    }
}
```

```java
// src/test/.../enrollment/EnrollmentSecurityTest.java
package com.example.eduhub.enrollment;

import static com.example.eduhub.TestAuth.as;
import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import com.example.eduhub.course.*;
import com.example.eduhub.employee.*;
import com.jayway.jsonpath.JsonPath;
import java.time.LocalDate;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
class EnrollmentSecurityTest {
    @Autowired MockMvc mvc;
    @Autowired EmployeeRepository employees;
    @Autowired CourseRepository courses;
    @Autowired CourseSessionRepository sessions;
    @Autowired EnrollmentRepository enrollments;

    Long kimId, leeId, sessionId;

    @BeforeEach
    void setUp() {
        kimId = employees.save(new Employee("T1001", "김사원", "개발1팀", null, Employee.Role.EMPLOYEE)).getId();
        leeId = employees.save(new Employee("T1002", "이사원", "개발2팀", null, Employee.Role.EMPLOYEE)).getId();
        Course c = courses.save(new Course("Spring Boot 입문", "BACKEND", null));
        sessionId = sessions.save(new CourseSession(c, LocalDate.now().plusDays(7), "본관", 1)).getId();
    }

    private Integer enroll(Long employeeId) throws Exception {
        String body = mvc.perform(post("/api/sessions/{id}/enrollments", sessionId).with(as(employeeId, "EMPLOYEE")))
                .andExpect(status().isCreated()).andReturn().getResponse().getContentAsString();
        return JsonPath.read(body, "$.enrollmentId");
    }

    @Test
    void 로그인하지_않으면_신청할_수_없다() throws Exception {
        mvc.perform(post("/api/sessions/{id}/enrollments", sessionId)).andExpect(status().isUnauthorized());
    }

    @Test
    void 신청자는_요청이_아니라_토큰의_사원이다() throws Exception {
        Integer id = enroll(kimId);
        assertThat(enrollments.findById(id.longValue()).orElseThrow().getEmployeeId()).isEqualTo(kimId);
    }

    @Test
    void 남의_신청은_취소할_수_없다_404() throws Exception {
        Integer id = enroll(kimId);
        mvc.perform(delete("/api/enrollments/{id}", id).with(as(leeId, "EMPLOYEE"))).andExpect(status().isNotFound());
        mvc.perform(delete("/api/enrollments/{id}", id).with(as(kimId, "EMPLOYEE"))).andExpect(status().isNoContent());
    }

    @Test
    void 정원이_차면_대기_그리고_취소하면_대기자가_확정된다() throws Exception {
        Integer first = enroll(kimId);
        String body = mvc.perform(post("/api/sessions/{id}/enrollments", sessionId).with(as(leeId, "EMPLOYEE")))
                .andExpect(jsonPath("$.status").value("WAITING"))
                .andExpect(jsonPath("$.waitingNo").value(1))
                .andReturn().getResponse().getContentAsString();
        Integer second = JsonPath.read(body, "$.enrollmentId");

        mvc.perform(delete("/api/enrollments/{id}", first).with(as(kimId, "EMPLOYEE"))).andExpect(status().isNoContent());
        assertThat(enrollments.findById(second.longValue()).orElseThrow().getStatus()).isEqualTo(EnrollmentStatus.CONFIRMED);
    }

    @Test
    void 과정_등록_수정은_관리자만_한다() throws Exception {
        String course = """
            {"title":"새 과정","category":"BACKEND","description":"설명"}""";
        mvc.perform(post("/api/v1/courses").contentType(MediaType.APPLICATION_JSON).content(course))
                .andExpect(status().isUnauthorized());
        mvc.perform(post("/api/v1/courses").contentType(MediaType.APPLICATION_JSON).content(course).with(as(kimId, "EMPLOYEE")))
                .andExpect(status().isForbidden());
        mvc.perform(post("/api/v1/courses").contentType(MediaType.APPLICATION_JSON).content(course).with(as(kimId, "ADMIN")))
                .andExpect(status().isCreated());
    }

    @Test
    void 관리자_API는_사원이_부르면_403() throws Exception {
        mvc.perform(get("/api/admin/sessions/{id}/roster", sessionId).with(as(kimId, "EMPLOYEE"))).andExpect(status().isForbidden());
    }
}
```

</details>

> 💡 테스트용 비밀키는 `src/test/resources/application-test.yml`에 **테스트 전용 값**으로 둡니다(`eduhub.jwt.secret: test-secret-…`, 32바이트 이상). 운영 비밀이 아니므로 커밋해도 됩니다. DB는 3회차에 정한 테스트 방식(Testcontainers 등)을 그대로 씁니다.

**직접 확인** (백엔드를 dev 프로필로 실행한 뒤)

```bash
# 로그인 → 토큰
TOKEN=$(curl -s -X POST localhost:8080/api/auth/login -H 'Content-Type: application/json' \
  -d '{"employeeNo":"E1001","password":"dev-password"}' | jq -r .accessToken)
curl -s -o /dev/null -w "%{http_code}\n" localhost:8080/api/me                                  # 401
curl -s localhost:8080/api/me -H "Authorization: Bearer $TOKEN"                                  # {"employeeId":1,"name":"김사원","role":"EMPLOYEE","admin":false}
curl -s -o /dev/null -w "%{http_code}\n" localhost:8080/api/admin/sessions/1/roster -H "Authorization: Bearer $TOKEN"   # 403
curl -s -o /dev/null -w "%{http_code}\n" -X POST localhost:8080/api/v1/courses -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"title":"t","category":"c"}'                         # 403 (사원은 과정 등록 불가)
```

```powershell
# Windows PowerShell
$login = Invoke-RestMethod -Method Post -Uri http://localhost:8080/api/auth/login -ContentType 'application/json' -Body '{"employeeNo":"E1001","password":"dev-password"}'
Invoke-RestMethod -Uri http://localhost:8080/api/me -Headers @{ Authorization = "Bearer $($login.accessToken)" }
```

| 개발용 계정 (dev 프로필) | 사번 | 비밀번호 | 역할 |
|---|---|---|---|
| 김사원 | `E1001` | `dev-password` | EMPLOYEE |
| 이사원 | `E1002` | `dev-password` | EMPLOYEE |
| 박관리 | `A0001` | `dev-password` | ADMIN |

> ✅ 이 교재 환경에서 Spring Boot 3.5.16 + PostgreSQL 16 + Flyway로 위 코드를 실행해 확인했습니다: 이 Step의 테스트 10개 통과(로그인 성공·실패, 토큰 없음·위조 401, 과정 조회 공개, 신청자=토큰의 사원, 남의 신청 취소 404, 정원 초과 시 대기와 취소 후 자동 확정, 과정 등록 401/403/201, 관리자 API 403). `/api/v1/courses/**`의 `hasRole("ADMIN")` 줄을 일부러 지우면 "과정 등록은 관리자만" 테스트가 실패하는 것도 확인했습니다. 과정·신청 쪽 코드는 3~4회차 교재와 같은 규칙으로 만든 축약본이라 팀 코드와 클래스 이름이 다를 수 있습니다.

> 💬 **강사 노트:** "왜 JWT를 직접 파싱하지 않나요?"라는 질문이 나오면, 서명·만료·알고리즘 검사를 빠뜨린 직접 구현이 흔한 보안 사고라는 점을 말합니다. 인증처럼 **검증된 표준 구현이 있는 영역은 그것을 쓰게** 지시하는 것도 개발자의 판단입니다 (2회차 2-2 아키텍처 지식 지도의 "보안").

---

## Step 10. FR-07 내 신청 내역 구현

권한 모드 **Plan**으로 지시합니다.

```text
[목표] FR-07: 로그인한 사원이 자기 신청 내역을 상태별로 조회한다.
[위치] backend/.../enrollment 패키지 (EnrollmentController, MyEnrollmentQueryService, EnrollmentRepository)
[범위]
- GET /api/me/enrollments?status=WAITING (status 생략 시 전체), 최근 신청순 목록
- 응답 DTO MyEnrollmentResponse: 신청 ID, 회차 ID, 과정 제목, 회차 시작일·장소, 상태, 대기 순번(대기일 때만), 신청 일시
- 과정·회차·대기 순번을 쿼리 한 번으로 (N+1 금지). 대기 순번은 저장하지 않고 계산 (4회차 4-3)
- 사원 ID는 요청 파라미터가 아니라 인증 정보에서 가져온다: CurrentUser.from(jwt).employeeId() (NFR-03, Step 9-1)
- OpenAPI 어노테이션(@Operation, 응답 코드)
- Step 9 결과 인덱스가 없으면 Flyway 새 버전 파일로 추가
[하지 않는 것] 화면(6장), FR-08
[완료 기준] ./gradlew test 통과. 내 것만 나옴, 상태 필터, 대기 순번, 실행한 SQL이 1개인지 확인하는 테스트
[진행 방식] 먼저 계획과 쿼리 방식을 보여줘
```

**계획에서 볼 점**
- 엔티티를 그대로 반환하지 않고 DTO를 쓰는가? (2회차 [2-3 사례](./session-02-hands-on-kickoff.md#2-3-사례-테스트는-통과했지만-문제가-있는-코드)의 ①)
- 사원 ID를 인증 정보에서 가져오는가? (같은 사례의 ④)
- 목록을 가져온 뒤 행마다 과정·대기 순번을 다시 조회하는 계획이면(N+1) 쿼리 한 번으로 바꾸게 합니다

> 💡 **페이지 대신 전체 목록**으로 정했습니다. 한 사원의 신청은 많아야 수십 건이고, 화면(6장)도 상태 필터만 씁니다. 늘어나면 그때 페이지를 붙입니다. 이런 결정을 지시문에 적어 두면 Claude가 습관처럼 `Pageable`을 붙이지 않습니다.

**기대하는 결과 (핵심 코드)** — 아래 코드는 실제로 빌드하고 테스트한 것입니다

```java
// enrollment/MyEnrollmentResponse.java
package com.example.eduhub.enrollment;

import java.time.LocalDate;
import java.time.LocalDateTime;

/** FR-07 응답. 엔티티를 그대로 내보내지 않는다 */
public record MyEnrollmentResponse(
        Long enrollmentId, Long sessionId, String courseTitle, LocalDate startDate, String place,
        EnrollmentStatus status, Integer waitingNo, LocalDateTime appliedAt) {

    /** JPQL 생성자 표현식용: 대기 순번(rank)은 쿼리에서 함께 계산하고, 대기가 아니면 null로 둔다 */
    public MyEnrollmentResponse(Long enrollmentId, Long sessionId, String courseTitle, LocalDate startDate, String place,
                                EnrollmentStatus status, Long waitingRank, LocalDateTime appliedAt) {
        this(enrollmentId, sessionId, courseTitle, startDate, place, status,
             status == EnrollmentStatus.WAITING ? waitingRank.intValue() : null, appliedAt);
    }
}
```

```java
// enrollment/EnrollmentRepository.java (FR-07 부분)
    /** FR-07: 과정·회차·대기 순번까지 쿼리 한 번으로 (N+1 없음). 대기 순번 = 같은 회차에서 나보다 먼저(또는 같이) 신청한 대기자 수 */
    @Query("""
        select new com.example.eduhub.enrollment.MyEnrollmentResponse(
            e.id, s.id, c.title, s.startDate, s.place, e.status,
            (select count(w) from Enrollment w
              where w.session = s and w.status = com.example.eduhub.enrollment.EnrollmentStatus.WAITING
                and (w.appliedAt < e.appliedAt or (w.appliedAt = e.appliedAt and w.id <= e.id))),
            e.appliedAt)
        from Enrollment e join e.session s join s.course c
        where e.employeeId = :employeeId and (:status is null or e.status = :status)
        order by e.appliedAt desc, e.id desc
        """)
    List<MyEnrollmentResponse> findMyEnrollments(@Param("employeeId") Long employeeId, @Param("status") EnrollmentStatus status);
```

```java
// enrollment/MyEnrollmentQueryService.java
package com.example.eduhub.enrollment;

import java.util.List;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/** FR-07: 조회 전용 */
@Service
public class MyEnrollmentQueryService {
    private final EnrollmentRepository enrollments;
    MyEnrollmentQueryService(EnrollmentRepository enrollments) { this.enrollments = enrollments; }

    @Transactional(readOnly = true)
    public List<MyEnrollmentResponse> find(Long employeeId, EnrollmentStatus status) {
        return enrollments.findMyEnrollments(employeeId, status);
    }
}
```

```java
// enrollment/EnrollmentController.java (추가: 생성자에 MyEnrollmentQueryService도 받는다)
    /** FR-07 (Step 10): 내 신청 내역. status를 주면 그 상태만 */
    @GetMapping("/api/me/enrollments")
    public List<MyEnrollmentResponse> getMyEnrollments(@RequestParam(required = false) EnrollmentStatus status,
                                           @AuthenticationPrincipal Jwt jwt) {
        return myEnrollments.find(CurrentUser.from(jwt).employeeId(), status);
    }
```

<details>
<summary>MyEnrollmentsTest.java — 내 것만, 상태 필터, 대기 순번, SQL 1개</summary>

```java
// src/test/.../enrollment/MyEnrollmentsTest.java
package com.example.eduhub.enrollment;

import static com.example.eduhub.TestAuth.as;
import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import com.example.eduhub.course.*;
import com.example.eduhub.employee.*;
import jakarta.persistence.EntityManager;
import java.time.LocalDate;
import org.hibernate.SessionFactory;
import org.hibernate.stat.Statistics;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

/** FR-07 내 신청 내역 */
@SpringBootTest(properties = "spring.jpa.properties.hibernate.generate_statistics=true")  // 실행한 SQL 수를 센다
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
class MyEnrollmentsTest {
    @Autowired MockMvc mvc;
    @Autowired EmployeeRepository employees;
    @Autowired CourseRepository courses;
    @Autowired CourseSessionRepository sessions;
    @Autowired EnrollmentService enrollmentService;
    @Autowired EntityManager em;

    Long kim, lee, park;

    @BeforeEach
    void setUp() {
        kim = employees.save(new Employee("T2001", "김사원", "개발1팀", null, Employee.Role.EMPLOYEE)).getId();
        lee = employees.save(new Employee("T2002", "이사원", "개발2팀", null, Employee.Role.EMPLOYEE)).getId();
        park = employees.save(new Employee("T2003", "박사원", "개발3팀", null, Employee.Role.EMPLOYEE)).getId();
        Course spring = courses.save(new Course("Spring Boot 입문", "BACKEND", null));
        Course jpa = courses.save(new Course("JPA 실전", "BACKEND", null));
        Long full = sessions.save(new CourseSession(spring, LocalDate.now().plusDays(7), "본관", 1)).getId();
        Long open = sessions.save(new CourseSession(jpa, LocalDate.now().plusDays(14), "온라인", 10)).getId();
        enrollmentService.enroll(full, kim);   // 확정
        enrollmentService.enroll(full, lee);   // 대기 1번
        enrollmentService.enroll(full, park);  // 대기 2번
        enrollmentService.enroll(open, park);  // 확정
    }

    @Test
    void 내_신청만_최근_신청순으로_과정_정보와_함께_보여_준다() throws Exception {
        mvc.perform(get("/api/me/enrollments").with(as(park, "EMPLOYEE")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.length()").value(2))
                .andExpect(jsonPath("$[0].courseTitle").value("JPA 실전"))
                .andExpect(jsonPath("$[0].status").value("CONFIRMED"))
                .andExpect(jsonPath("$[0].waitingNo").doesNotExist())
                .andExpect(jsonPath("$[1].courseTitle").value("Spring Boot 입문"))
                .andExpect(jsonPath("$[1].status").value("WAITING"))
                .andExpect(jsonPath("$[1].waitingNo").value(2));
        mvc.perform(get("/api/me/enrollments").with(as(kim, "EMPLOYEE")))
                .andExpect(jsonPath("$.length()").value(1));
    }

    @Test
    void 상태로_거를_수_있다() throws Exception {
        mvc.perform(get("/api/me/enrollments").param("status", "WAITING").with(as(park, "EMPLOYEE")))
                .andExpect(jsonPath("$.length()").value(1))
                .andExpect(jsonPath("$[0].status").value("WAITING"));
    }

    @Test
    void 로그인하지_않으면_401() throws Exception {
        mvc.perform(get("/api/me/enrollments")).andExpect(status().isUnauthorized());
    }

    @Test
    void 목록은_쿼리_한_번으로_조회한다_N플러스1_없음() throws Exception {
        em.flush();
        em.clear();
        Statistics stats = em.getEntityManagerFactory().unwrap(SessionFactory.class).getStatistics();
        stats.clear();

        mvc.perform(get("/api/me/enrollments").with(as(park, "EMPLOYEE"))).andExpect(status().isOk());

        assertThat(stats.getPrepareStatementCount()).isEqualTo(1);
    }
}
```

</details>

> ✅ 이 교재 환경(Spring Boot 3.5.16, Hibernate 6, PostgreSQL 16)에서 테스트 4개가 통과했습니다. 쿼리를 "엔티티로 모두 조회한 뒤 행마다 회차·과정을 읽는" 방식으로 바꾸면 SQL 개수 테스트와 대기 순번·상태 필터 테스트가 실패하는 것도 확인했습니다. 실행한 SQL 수는 Hibernate 통계(`generate_statistics`)로 셉니다.

**구현 중 확인할 것 (오늘의 훅이 일하는 모습)**
- Java 파일을 수정할 때마다 포맷이 자동으로 맞춰지는가? (훅 ①)
- Claude가 기존 `V*.sql`을 고치려 하면 막히고 새 버전 파일을 만드는가? (훅 ②)
- 응답이 끝날 때 테스트가 돌고, 실패하면 Claude가 이어서 고치는가? (훅 ④)

**MCP로 결과 확인**

```text
eduhub-db로 EXPLAIN을 실행해서 방금 만든 내 신청 내역 쿼리가 인덱스를 쓰는지만 알려줘.
```

> ✅ **확인**: 내 터미널에서 `cd backend && ./gradlew test`를 **직접** 실행해 통과하는지 봅니다. Swagger UI(`/swagger-ui.html`)에서 새 API와 설명이 보이는지도 확인합니다. 토큰으로 직접 불러 봅니다: `curl -s localhost:8080/api/me/enrollments -H "Authorization: Bearer $TOKEN"` (토큰은 Step 9-1처럼 받는다)

```bash
git add backend/
git diff --staged
git commit -m "feat(enrollment): 내 신청 내역 조회 API 추가 (FR-07)"
git push -u origin feature/fr-07-my-enrollments
gh pr create --fill          # 또는 GitHub 화면의 Compare & pull request
```

---

## Step 11. FR-08 명단 조회와 수료 처리

FR-07 PR을 리뷰 요청한 뒤 **새 세션**(`/clear`)에서 시작합니다. 한 세션 = 한 작업입니다.

```bash
git switch main && git pull
git switch -c feature/fr-08-roster-completion
```

**먼저 사람이 정할 것** (업무 규칙은 Claude가 정하게 두지 않는다)

| 결정 | EduHub의 선택 (예) | 이유 |
|---|---|---|
| 언제부터 수료 처리할 수 있나 | 회차 **시작일 당일부터**. 그 전이면 409 | 교육을 시작하지도 않은 사람을 수료로 만들 수 없다 |
| 여러 명을 한 번에 처리하다 일부가 안 되면 | **되는 사람만 처리**하고, 안 된 사람은 사유와 함께 돌려준다 | 한 명 때문에 전체가 실패하면 관리자가 다시 골라야 한다 |
| 수료할 수 있는 상태 | `CONFIRMED`만. 대기·취소·이미 수료는 사유를 돌려준다 | 대기자는 교육을 받지 않았다 |
| 명단에 보일 것 | 확정자(수료 포함)와 대기자(대기 순번 순), **사번·이름** | 관리자는 사원 ID 숫자로는 사람을 알아볼 수 없다 |

> 💡 위 표는 예시입니다. 팀이 다르게 정해도 되지만, **정한 내용을 `docs/requirements.md`에 적고** 지시문에 넣습니다. Claude가 질문하면 **사람이 답하는 것**이 구조 통제입니다.

```text
[목표] FR-08: 관리자가 회차별 신청자·대기자 명단을 보고 수료 처리한다.
[위치] backend/.../enrollment 패키지 (RosterService, AdminEnrollmentController), 관리자 API는 /api/admin 아래
[범위]
- GET /api/admin/sessions/{sessionId}/roster : 과정 제목, 시작일, 정원, 확정자(수료 포함)와 대기자(대기 순번 순). 사번·이름 포함
- POST /api/admin/sessions/{sessionId}/completions {employeeIds: [...]} : CONFIRMED → COMPLETED
  - 결과는 {completed: [사원 ID], failed: [{employeeId, reason}]}. CONFIRMED가 아니면 처리하지 않고 사유를 돌려준다
  - 회차 시작일 전이면 409, 빈 목록은 400
- 관리자 권한이 없으면 403 (Step 9-1의 /api/admin/** 규칙)
- OpenAPI 어노테이션
[하지 않는 것] 화면(6장), 알림(FR-09)
[완료 기준] 정상·권한 없음·상태 오류(대기·신청 없음·이미 수료)·시작일 이전·빈 목록 테스트, ./gradlew test 통과
[진행 방식] 계획부터. 위 규칙 외에 애매한 점이 있으면 질문해줘
```

**기대하는 결과 (핵심 코드)** — 아래 코드는 실제로 빌드하고 테스트한 것입니다

```java
// enrollment/RosterEntry.java
package com.example.eduhub.enrollment;

/** FR-08 명단 한 줄. waitingNo는 대기자에게만 */
public record RosterEntry(Long enrollmentId, Long employeeId, String employeeNo, String name,
                          EnrollmentStatus status, Integer waitingNo) {
    public RosterEntry(Long enrollmentId, Long employeeId, String employeeNo, String name, EnrollmentStatus status) {
        this(enrollmentId, employeeId, employeeNo, name, status, null);
    }
    RosterEntry withWaitingNo(int no) { return new RosterEntry(enrollmentId, employeeId, employeeNo, name, status, no); }
}
```

```java
// enrollment/EnrollmentRepository.java (FR-08 부분)
    /** FR-08: 명단은 사번·이름과 함께. Enrollment는 사원 ID만 가지므로 Employee와 ON으로 조인 */
    @Query("""
        select new com.example.eduhub.enrollment.RosterEntry(e.id, m.id, m.employeeNo, m.name, e.status)
        from Enrollment e join Employee m on m.id = e.employeeId
        where e.session.id = :sessionId and e.status in :statuses
        order by e.appliedAt, e.id
        """)
    List<RosterEntry> findRoster(@Param("sessionId") Long sessionId, @Param("statuses") Collection<EnrollmentStatus> statuses);

    List<Enrollment> findBySessionIdAndEmployeeIdIn(Long sessionId, Collection<Long> employeeIds);
```

```java
// enrollment/RosterService.java
package com.example.eduhub.enrollment;

import com.example.eduhub.common.ConflictException;
import com.example.eduhub.common.NotFoundException;
import com.example.eduhub.course.CourseSession;
import com.example.eduhub.course.CourseSessionRepository;
import java.time.LocalDate;
import java.util.*;
import java.util.stream.IntStream;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/** FR-08: 회차 명단과 수료 처리 (관리자) */
@Service
public class RosterService {
    public record RosterResponse(Long sessionId, String courseTitle, LocalDate startDate, int capacity,
                                 List<RosterEntry> confirmed, List<RosterEntry> waiting) {}
    public record Failure(Long employeeId, String reason) {}
    public record CompletionResult(List<Long> completed, List<Failure> failed) {}

    private final EnrollmentRepository enrollments;
    private final CourseSessionRepository sessions;
    RosterService(EnrollmentRepository enrollments, CourseSessionRepository sessions) { this.enrollments = enrollments; this.sessions = sessions; }

    /** 확정자(수료 포함)와 대기자(대기 순번 순) */
    @Transactional(readOnly = true)
    public RosterResponse roster(Long sessionId) {
        CourseSession s = session(sessionId);
        List<RosterEntry> confirmed = enrollments.findRoster(sessionId, List.of(EnrollmentStatus.CONFIRMED, EnrollmentStatus.COMPLETED));
        List<RosterEntry> waiting = enrollments.findRoster(sessionId, List.of(EnrollmentStatus.WAITING));
        return new RosterResponse(sessionId, s.getCourse().getTitle(), s.getStartDate(), s.getCapacity(), confirmed,
                IntStream.range(0, waiting.size()).mapToObj(i -> waiting.get(i).withWaitingNo(i + 1)).toList());
    }

    /** CONFIRMED만 COMPLETED로. 나머지는 처리하지 않고 사유를 돌려준다. 회차 시작일 전이면 409 */
    @Transactional
    public CompletionResult complete(Long sessionId, List<Long> employeeIds) {
        CourseSession s = session(sessionId);
        if (LocalDate.now().isBefore(s.getStartDate())) {
            throw new ConflictException("회차 시작일(" + s.getStartDate() + ") 전에는 수료 처리할 수 없습니다.");
        }
        Map<Long, List<Enrollment>> byEmployee = new HashMap<>();
        for (Enrollment e : enrollments.findBySessionIdAndEmployeeIdIn(sessionId, employeeIds)) {
            byEmployee.computeIfAbsent(e.getEmployeeId(), k -> new ArrayList<>()).add(e);
        }
        List<Long> completed = new ArrayList<>();
        List<Failure> failed = new ArrayList<>();
        for (Long employeeId : new LinkedHashSet<>(employeeIds)) {
            List<Enrollment> mine = byEmployee.getOrDefault(employeeId, List.of());
            Optional<Enrollment> confirmed = mine.stream().filter(e -> e.getStatus() == EnrollmentStatus.CONFIRMED).findFirst();
            if (confirmed.isPresent()) { confirmed.get().complete(); completed.add(employeeId); }
            else failed.add(new Failure(employeeId, reason(mine)));
        }
        return new CompletionResult(completed, failed);
    }

    private static String reason(List<Enrollment> mine) {
        if (mine.stream().anyMatch(e -> e.getStatus() == EnrollmentStatus.COMPLETED)) return "이미 수료 처리됨";
        if (mine.stream().anyMatch(e -> e.getStatus() == EnrollmentStatus.WAITING)) return "대기 중인 신청";
        if (!mine.isEmpty()) return "취소된 신청";
        return "이 회차 신청 내역 없음";
    }

    private CourseSession session(Long id) { return sessions.findById(id).orElseThrow(() -> new NotFoundException("회차 " + id)); }
}
```

```java
// enrollment/AdminEnrollmentController.java
package com.example.eduhub.enrollment;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotEmpty;
import java.util.List;
import org.springframework.web.bind.annotation.*;

/** FR-08. /api/admin/** 는 SecurityConfig에서 ADMIN 역할만 허용한다 */
@RestController
@RequestMapping("/api/admin/sessions/{sessionId}")
public class AdminEnrollmentController {
    public record CompletionRequest(@NotEmpty List<Long> employeeIds) {}
    private final RosterService service;
    AdminEnrollmentController(RosterService service) { this.service = service; }

    @GetMapping("/roster")
    public RosterService.RosterResponse getRoster(@PathVariable Long sessionId) { return service.roster(sessionId); }

    @PostMapping("/completions")
    public RosterService.CompletionResult complete(@PathVariable Long sessionId, @Valid @RequestBody CompletionRequest req) {
        return service.complete(sessionId, req.employeeIds());
    }
}
```

<details>
<summary>RosterTest.java — 명단, 403, 부분 처리와 사유, 시작일 전 409, 빈 목록 400</summary>

```java
// src/test/.../enrollment/RosterTest.java
package com.example.eduhub.enrollment;

import static com.example.eduhub.TestAuth.as;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

import com.example.eduhub.course.*;
import com.example.eduhub.employee.*;
import java.time.LocalDate;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

/** FR-08 명단과 수료 처리 */
@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional
class RosterTest {
    @Autowired MockMvc mvc;
    @Autowired EmployeeRepository employees;
    @Autowired CourseRepository courses;
    @Autowired CourseSessionRepository sessions;
    @Autowired EnrollmentService enrollmentService;

    Long admin, kim, lee, park, started, upcoming;

    @BeforeEach
    void setUp() {
        admin = employees.save(new Employee("T3000", "박관리", "인사팀", null, Employee.Role.ADMIN)).getId();
        kim = employees.save(new Employee("T3001", "김사원", "개발1팀", null, Employee.Role.EMPLOYEE)).getId();
        lee = employees.save(new Employee("T3002", "이사원", "개발2팀", null, Employee.Role.EMPLOYEE)).getId();
        park = employees.save(new Employee("T3003", "박사원", "개발3팀", null, Employee.Role.EMPLOYEE)).getId();
        Course c = courses.save(new Course("Git 협업 기초", "TOOL", null));
        started = sessions.save(new CourseSession(c, LocalDate.now().minusDays(1), "본관", 1)).getId();   // 어제 시작
        upcoming = sessions.save(new CourseSession(c, LocalDate.now().plusDays(7), "본관", 10)).getId();
        enrollmentService.enroll(started, kim);   // 확정
        enrollmentService.enroll(started, lee);   // 대기 1번
        enrollmentService.enroll(upcoming, kim);
    }

    private String ids(Long... ids) {
        return "{\"employeeIds\":" + java.util.Arrays.toString(ids) + "}";
    }

    @Test
    void 명단은_확정자와_대기자를_사번_이름과_함께_보여_준다() throws Exception {
        mvc.perform(get("/api/admin/sessions/{id}/roster", started).with(as(admin, "ADMIN")))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.courseTitle").value("Git 협업 기초"))
                .andExpect(jsonPath("$.confirmed.length()").value(1))
                .andExpect(jsonPath("$.confirmed[0].name").value("김사원"))
                .andExpect(jsonPath("$.confirmed[0].employeeNo").value("T3001"))
                .andExpect(jsonPath("$.waiting[0].name").value("이사원"))
                .andExpect(jsonPath("$.waiting[0].waitingNo").value(1));
    }

    @Test
    void 사원은_명단을_볼_수_없다_403() throws Exception {
        mvc.perform(get("/api/admin/sessions/{id}/roster", started).with(as(kim, "EMPLOYEE"))).andExpect(status().isForbidden());
    }

    @Test
    void 확정자만_수료_처리하고_나머지는_사유를_돌려준다() throws Exception {
        mvc.perform(post("/api/admin/sessions/{id}/completions", started).with(as(admin, "ADMIN"))
                        .contentType(MediaType.APPLICATION_JSON).content(ids(kim, lee, park)))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.completed[0]").value(kim))
                .andExpect(jsonPath("$.failed.length()").value(2))
                .andExpect(jsonPath("$.failed[0].reason").value("대기 중인 신청"))
                .andExpect(jsonPath("$.failed[1].reason").value("이 회차 신청 내역 없음"));
        // 같은 사람을 다시 수료 처리하면 실패 사유로
        mvc.perform(post("/api/admin/sessions/{id}/completions", started).with(as(admin, "ADMIN"))
                        .contentType(MediaType.APPLICATION_JSON).content(ids(kim)))
                .andExpect(jsonPath("$.failed[0].reason").value("이미 수료 처리됨"));
        mvc.perform(get("/api/admin/sessions/{id}/roster", started).with(as(admin, "ADMIN")))
                .andExpect(jsonPath("$.confirmed[0].status").value("COMPLETED"));
    }

    @Test
    void 회차_시작일_전에는_수료_처리할_수_없다_409() throws Exception {
        mvc.perform(post("/api/admin/sessions/{id}/completions", upcoming).with(as(admin, "ADMIN"))
                        .contentType(MediaType.APPLICATION_JSON).content(ids(kim)))
                .andExpect(status().isConflict());
    }

    @Test
    void 빈_목록은_400() throws Exception {
        mvc.perform(post("/api/admin/sessions/{id}/completions", started).with(as(admin, "ADMIN"))
                        .contentType(MediaType.APPLICATION_JSON).content("{\"employeeIds\":[]}"))
                .andExpect(status().isBadRequest());
    }
}
```

</details>

> ✅ 이 교재 환경에서 테스트 5개가 통과했습니다. 시작일 검사를 일부러 빼면 "시작일 전 409" 테스트가 실패하는 것도 확인했습니다. `Enrollment`는 사원을 연관관계가 아니라 ID(`employeeId`)로만 가지므로, 명단 쿼리는 `join Employee m on m.id = e.employeeId`(Hibernate 6의 엔티티 조인)로 사번·이름을 함께 가져옵니다.

**MCP 활용 예**

```text
eduhub-db로 테스트용 회차 하나를 골라 CONFIRMED/WAITING 건수만 보여줘.
그 회차로 명단 API를 로컬에서 호출했을 때 기대 결과를 정리해줘.
```

> ✅ **확인**: 관리자 토큰과 일반 사원 토큰으로 각각 호출해 200과 403이 나오는지 직접 확인합니다. 토큰은 Step 9-1처럼 `A0001`(관리자), `E1001`(사원)로 로그인해 받습니다. dev 프로필의 4번 회차(Git 협업 기초)는 이미 시작한 회차라 수료 처리가 되고, 1번 회차는 시작 전이라 409가 나옵니다. 커밋·푸시·PR은 Step 10과 같습니다 (`feat(enrollment): 회차 명단 조회와 수료 처리 API 추가 (FR-08)`).

```bash
ADMIN=$(curl -s -X POST localhost:8080/api/auth/login -H 'Content-Type: application/json' \
  -d '{"employeeNo":"A0001","password":"dev-password"}' | jq -r .accessToken)
curl -s localhost:8080/api/admin/sessions/4/roster -H "Authorization: Bearer $ADMIN" | jq '.confirmed[].name'
curl -s -X POST localhost:8080/api/admin/sessions/4/completions -H "Authorization: Bearer $ADMIN" \
  -H 'Content-Type: application/json' -d '{"employeeIds":[1,3]}'
# {"completed":[1],"failed":[{"employeeId":3,"reason":"이 회차 신청 내역 없음"}]}
```

---

## Step 12. GitHub는 `gh` CLI로: MCP와 비교해 보기

PR 확인은 `gh`로 해 봅니다.

```text
gh로 내가 연 PR 목록과 각 PR의 CI 상태만 한 줄씩 요약해줘.
```

Claude는 `gh pr list --author @me --json …` 같은 명령을 씁니다. 같은 일을 GitHub MCP로 할 수도 있지만, `gh`가 이미 있으면 **도구 정의가 추가되지 않고** 출력도 옵션으로 줄일 수 있어 가볍습니다.

| 관찰 | 기록 |
|---|---|
| `/context`에서 MCP 도구 항목은 얼마나 차지하나? | |
| `/mcp`에서 쓰지 않는 서버가 켜져 있지 않은가? | |
| 오늘 MCP 결과 중 가장 컸던 것은? 더 작게 요청할 수 있었나? | |

---

## 5-7. 자주 겪는 문제 (MCP)

| 증상 | 원인과 해결 |
|---|---|
| `/mcp`에 서버는 있는데 도구가 안 보임 | 서버가 failed 상태(대부분 DB 접속 실패로 시작 직후 종료). Step 8의 "도구가 안 보일 때 점검 순서"와 서버 직접 실행으로 원인 확인 |
| `permissions.allow`에 적었는데 계속 승인을 물음 | `"permissions"`(s), `mcp__서버__도구`(밑줄 2개) 오타 확인. `/permissions`에서 규칙 확인 |
| `uvx: command not found` | uv 미설치. Step 8의 ① 준비. 설치 후 터미널·Claude Code를 다시 시작 |
| `/mcp`에 서버가 failed로 보임 | `uvx --version`, `EDUHUB_DB_READONLY_URL`이 **Claude Code를 실행한 셸**에 있는지(`echo $EDUHUB_DB_READONLY_URL`), DB 컨테이너 실행 여부 확인. 첫 실행은 패키지를 받느라 오래 걸릴 수 있으니 `/mcp`에서 다시 연결 |
| `${EDUHUB_DB_READONLY_URL}`이 글자 그대로 남음 | 환경 변수가 없음. 셸 프로필에 넣고 터미널·앱을 다시 시작 |
| 승인 질문 없이 서버가 안 보임 | 이전에 거부했을 수 있음. `claude mcp reset-project-choices`로 선택 초기화 |
| 윈도우에서 `npx` 서버(Google MCP Toolbox 대안 등)가 실행되지 않음 | `cmd /c npx …` 형태로 감싸기 ([MCP 가이드 5-2](../examples/19-mcp-server-guide.md#5-2-운영체제별-주의)) |
| DB 연결 거부 | Docker DB 컨테이너가 실행 중인지, 포트(5432) 확인 |
| 결과가 너무 커서 경고 | 조회 범위를 좁힌다 ("건수만", "5건만") |
| Claude가 MCP로 데이터를 고치려 함 | 읽기 전용 계정이라 실패해야 정상. 실패하지 않았다면 Step 7 권한을 다시 확인 |

---

# 6. 따라하기 ③: 로그인과 화면 완성 (하나의 React 앱)

> **목표**: 3회차에 만든 과정 목록 앱(`frontend/apps/course-mfe`, 포트 3001)에 **로그인과 나머지 화면**을 붙여, FR-01~08을 브라우저에서 처음부터 끝까지 쓸 수 있는 **하나의 React 앱**으로 완성합니다. 마이크로프론트엔드(MFE) 분리는 6회차에 이 앱을 쪼개며 합니다.

> ⏱ 수업에서는 **Step 13~14(준비와 로그인)**까지 진행하고, Step 15~16은 **과제**로 완성합니다. 6회차는 이 앱을 shell·course-mfe·enrollment-mfe로 쪼개는 것으로 시작하므로, **6회차 전까지 Step 16까지** 끝나 있어야 합니다.

---

## 6-1. 무엇을 만드나

**왜 먼저 하나의 앱으로 만드나**

| 하나의 앱으로 먼저 | 이유 |
|---|---|
| 화면과 인증에만 집중 | Module Federation, 공유 의존성(singleton), 포트 3개 같은 신경 쓸 것이 없다 |
| 분리 전의 "정답"이 생긴다 | 6회차에 쪼갠 뒤 같은 확인 표(6-2)로 **기능이 그대로인지** 비교할 수 있다 |
| 실제 현장과 같은 순서 | MFE는 대개 이미 동작하는 앱을 나누면서 도입한다. 무엇이 깨지는지 직접 겪어 봐야 경계 규칙이 와닿는다 |

> 💡 앱 이름이 `course-mfe`인 것은 2회차 뼈대의 이름 그대로입니다. 6회차에 이 앱의 과정 화면은 course-mfe(remote)로 남고, 로그인·레이아웃은 shell로, 신청 화면은 enrollment-mfe로 옮깁니다. 그래서 **폴더를 처음부터 6회차에 갈 곳별로 나눠 둡니다.**

**화면 지도** (개발용 계정: 사원 `E1001`·`E1002`, 관리자 `A0001`, 비밀번호 `dev-password`)

| 경로 | 화면 | 요구사항 | 누가 | 파일 (6회차에 갈 곳) |
|---|---|---|---|---|
| `/login` | 로그인 | NFR-03 | 누구나 | `auth/LoginPage.tsx` (shell) |
| `/courses` | 과정 목록 (3회차) + 행을 누르면 상세 | FR-01 | 누구나 | `pages/CourseListPage.tsx` (course-mfe) |
| `/courses/:courseId` | 과정 상세, 회차별 **신청 버튼** | FR-02, 04, 05 | 누구나 (신청은 로그인) | `pages/CourseDetailPage.tsx` (course-mfe), `enrollment/EnrollButton.tsx` (enrollment-mfe) |
| `/my/enrollments` | 내 신청 내역(상태 필터), 취소 | FR-06, 07 | 로그인 | `enrollment/MyEnrollmentsPage.tsx` (enrollment-mfe) |
| `/admin/courses/new`, `/admin/courses/:courseId/edit` | 과정 등록·수정, 회차 추가, 과정 종료 | FR-03 | 관리자 | `pages/CourseFormPage.tsx` (course-mfe) |
| `/admin/sessions/:sessionId/roster` | 회차 명단, 수료 처리 | FR-08 | 관리자 | `enrollment/RosterPage.tsx` (enrollment-mfe) |

```text
frontend/apps/course-mfe/src/
├── main.tsx, App.tsx, theme.ts   # 시작점, 상단 바·라우팅, MUI 테마        → 6회차: shell
├── api/
│   ├── http.ts                   # 토큰 보관, 토큰 붙이기, 401 처리 (+ 테스트) → 6회차: packages/api-client
│   ├── courseApi.ts              # 3회차 파일 (http를 쓰도록 한 줄 변경)
│   └── clients.ts                # authApi, enrollmentApi, adminEnrollmentApi
├── auth/                         # AuthContext, RequireAuth, LoginPage       → 6회차: shell
├── hooks/, pages/                # 3회차 목록 + 상세, 과정 등록·수정          → 6회차: course-mfe에 남음
└── enrollment/                   # 신청 버튼, 내 신청 내역, 명단 (+ 테스트)    → 6회차: enrollment-mfe
```

```mermaid
flowchart LR
    subgraph APP["apps/course-mfe :3001 (하나의 앱)"]
        AUTH["auth/<br/>AuthContext · RequireAuth · LoginPage"]
        PAGES["pages/<br/>목록 · 상세 · 과정 등록·수정"]
        ENR["enrollment/<br/>신청 버튼 · 내 신청 내역 · 명단"]
        HTTP["api/http.ts<br/>토큰 첨부 · 401 → 이벤트"]
    end
    PAGES --> HTTP
    ENR --> HTTP
    AUTH --> HTTP
    HTTP -- "/api (Vite 프록시)<br/>Authorization: Bearer …" --> BE["backend :8080"]
    HTTP -. "401 → eduhub:unauthorized" .-> AUTH
```

---

## Step 13. 준비: 브랜치, 의존성, API 클라이언트, 프록시

```bash
git switch main && git pull                  # Step 9-1·10·11 PR이 병합된 main
git switch -c feature/frontend-auth-screens
cd frontend
npm install --workspace apps/course-mfe react-router-dom axios
npm install -D --workspace apps/course-mfe @testing-library/user-event
```

**① API 클라이언트 다시 만들기**: 백엔드를 dev 프로필로 띄운 뒤(Step 9-1의 "직접 확인") 3회차의 `npm run generate:api`를 다시 실행합니다. 생성된 `packages/api-client/src/generated/api.ts`에 `AuthApi`(login, me), `EnrollmentApi`(enroll, cancel, getMyEnrollments), `AdminEnrollmentApi`(getRoster, complete), `CourseApi`의 등록·수정 메서드가 생겼는지 봅니다. **메서드 이름은 컨트롤러 메서드 이름에서 오므로 실제 이름을 확인**하고 아래 코드와 다르면 맞춥니다.

> ⚠️ 생성된 코드(`generated/`)는 **손으로 고치지 않습니다.** 토큰 처리 같은 손으로 쓰는 코드는 앱의 `src/api/`에 둡니다 (6회차에 패키지로 옮긴다).

**② 개발 서버 프록시**: 3회차 💡에서 미뤄 둔 CORS 문제를 **프록시**로 정합니다. 화면은 `/api/...`(같은 origin)로 부르고, 개발 서버가 8080으로 넘깁니다. 빌드 결과를 확인하는 `preview`에도 같게 둡니다.

```ts
/// <reference types="vitest/config" />
// frontend/apps/course-mfe/vite.config.ts — 5회차: /api 프록시 추가 (3회차 설정 + server/preview.proxy)
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  // /api 요청은 개발 서버가 백엔드로 넘긴다 → 브라우저 입장에서는 같은 origin이라 CORS 설정이 필요 없다
  server: { port: 3001, strictPort: true, proxy: { "/api": "http://localhost:8080" } },
  preview: { port: 3001, strictPort: true, proxy: { "/api": "http://localhost:8080" } },
  test: {
    environment: "jsdom",
    setupFiles: ["./src/setupTests.ts"],
  },
});
```

**③ 기존 테스트의 타입 오류 고치기**: 생성된 클라이언트의 `getCourses`는 Axios 응답 전체(`AxiosResponse`) 타입을 돌려주므로, 3회차 `useCourses.test.ts`의 `mockResolvedValue(pageOf(...))`가 `npm run build`(`tsc --noEmit`)에서 "status, statusText, headers, config가 없다"는 타입 오류를 냅니다. 테스트에 필요 없는 필드이므로 `as never`로 맞춥니다.

```ts
// frontend/apps/course-mfe/src/hooks/useCourses.test.ts (한 줄 변경)
getCourses.mockResolvedValue(pageOf(['Spring Boot 입문', 'JPA 실전']) as never);   // ★ AxiosResponse의 나머지 필드는 테스트에 필요 없다
```

> 💡 3회차 교재의 이 테스트는 응답을 단순하게 흉내 낸 스텁 기준으로 확인한 것이라 이 오류가 드러나지 않았습니다. 실제로 생성한 클라이언트로 빌드하면 나오므로, 여기서 함께 고칩니다.

> ✅ **확인**: `npm run build --workspace apps/course-mfe`와 `npm test --workspace apps/course-mfe`가 통과하면 커밋합니다 (`chore(frontend): API 클라이언트 재생성과 /api 프록시`).

---

## Step 14. 로그인: 토큰 처리, 로그인 상태, 보호 라우트

새 세션(또는 `/clear`)에서 Plan 모드로 지시합니다.

```text
[목표] 로그인(NFR-03): 사번·비밀번호로 로그인하고, 이후 모든 API 호출에 토큰을 붙인다. 로그인·관리자 전용 화면을 막는다.
[위치] @frontend/apps/course-mfe/src (api/, auth/, App.tsx, main.tsx, theme.ts)
[범위]
- src/api/http.ts: tokenStore(localStorage: eduhub.accessToken, eduhub.user), Axios 인스턴스 http
  - 요청 인터셉터: 토큰이 있으면 Authorization: Bearer
  - 응답 인터셉터: 401이면 토큰 삭제 + window 이벤트 eduhub:unauthorized. 403은 로그아웃하지 않는다
- 생성된 *Api는 new XxxApi(config, undefined, http)로 src/api 안에서만 만든다 (courseApi.ts는 http를 쓰도록 변경, 나머지는 clients.ts)
- src/auth/AuthContext.tsx: user, expired, login, logout. eduhub:unauthorized를 들으면 로그인 화면으로. 새로고침 시 토큰이 있으면 /api/me로 확인
- src/auth/RequireAuth.tsx: 로그인 안 했으면 /login(원래 경로를 state.from에), admin이면 관리자 확인
- src/auth/LoginPage.tsx: 로그인 후 원래 가려던 화면으로. 실패하면 "사번 또는 비밀번호가 올바르지 않습니다." 만료로 왔으면 안내
- App.tsx: 상단 바(교육과정, 로그인 시 내 신청 내역, 관리자면 과정 등록, 이름·로그아웃/로그인)와 라우팅. main.tsx: ThemeProvider(theme.ts) → BrowserRouter → AuthProvider
- 테스트(Vitest): http — 토큰 첨부, 토큰 없으면 헤더 없음, 401이면 토큰 삭제·이벤트, 403은 유지
[하지 않는 것] 상세·신청·관리자 화면(Step 15·16), 백엔드 수정, generated/ 수정
[완료 기준] npm run build·npm test (apps/course-mfe) 통과, 로그인·로그아웃·보호 화면 이동이 브라우저에서 동작
[진행 방식] 바꿀 파일 목록과 이유를 먼저 보여줘
```

**계획 검토 포인트**
- 화면마다 `axios.create()`나 `fetch`로 토큰을 붙이는 계획이면 `http` 하나로 모으게 합니다 (한 곳이라야 401 처리와 6회차 분리가 쉽다)
- 403(권한 없음)에도 로그아웃시키는 계획이면 고치게 합니다. 관리자 화면을 잘못 연 사원이 로그아웃되면 안 된다
- 토큰을 `generated/` 안에 넣거나 생성 코드를 고치는 계획이면 거절합니다

**기대하는 결과 (핵심 코드)** — 아래 코드는 실제로 빌드·테스트하고 브라우저로 확인한 것입니다 (6-2의 확인 표)

```ts
// frontend/apps/course-mfe/src/api/http.ts — 토큰 보관과 모든 API 호출이 쓰는 Axios 인스턴스
import axios from 'axios';
import type { CurrentUser } from '@eduhub/api-client';

const TOKEN_KEY = 'eduhub.accessToken';
const USER_KEY = 'eduhub.user';

/** 토큰 보관소 (localStorage). 새로고침해도 로그인이 유지된다 */
export const tokenStore = {
  get: (): string | null => localStorage.getItem(TOKEN_KEY),
  user: (): CurrentUser | null => {
    const raw = localStorage.getItem(USER_KEY);
    return raw ? (JSON.parse(raw) as CurrentUser) : null;
  },
  save: (token: string, user: CurrentUser) => {
    localStorage.setItem(TOKEN_KEY, token);
    localStorage.setItem(USER_KEY, JSON.stringify(user));
  },
  clear: () => {
    localStorage.removeItem(TOKEN_KEY);
    localStorage.removeItem(USER_KEY);
  },
};

/** 로그인이 필요하다는 신호. AuthContext가 듣고 로그인 화면으로 보낸다 */
export const UNAUTHORIZED_EVENT = 'eduhub:unauthorized';

/** 모든 API 호출이 쓰는 Axios 인스턴스. 생성된 *Api 클래스의 세 번째 인자로 넘긴다 */
export const http = axios.create();

// 요청마다 토큰을 붙인다
http.interceptors.request.use((config) => {
  const token = tokenStore.get();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// 401(토큰 없음·만료·위조)이면 저장된 토큰을 지우고 알린다. 403(권한 없음)은 로그아웃하지 않는다
http.interceptors.response.use(
  (response) => response,
  (error) => {
    if (axios.isAxiosError(error) && error.response?.status === 401) {
      tokenStore.clear();
      window.dispatchEvent(new CustomEvent(UNAUTHORIZED_EVENT));
    }
    return Promise.reject(error);
  },
);
```

```ts
// frontend/apps/course-mfe/src/api/courseApi.ts — 5회차: 세 번째 인자로 http(토큰을 붙이는 인스턴스)를 넘긴다
import { Configuration, CourseApi } from '@eduhub/api-client';
import { http } from './http';

// 생성된 클라이언트의 인스턴스는 여기 한 곳에서만 만든다 (basePath ''= 같은 origin, Vite 프록시가 백엔드로)
export const courseApi = new CourseApi(new Configuration({ basePath: '' }), undefined, http);
```

```ts
// frontend/apps/course-mfe/src/api/clients.ts — 인증·신청·관리자 API 인스턴스 (만드는 곳은 여기 한 곳)
import { AdminEnrollmentApi, AuthApi, Configuration, EnrollmentApi } from '@eduhub/api-client';
import { http } from './http';

const config = new Configuration({ basePath: '' });
export const authApi = new AuthApi(config, undefined, http);
export const enrollmentApi = new EnrollmentApi(config, undefined, http);
export const adminEnrollmentApi = new AdminEnrollmentApi(config, undefined, http);
```

<details>
<summary>auth: AuthContext.tsx, RequireAuth.tsx, LoginPage.tsx</summary>

```tsx
// frontend/apps/course-mfe/src/auth/AuthContext.tsx — 로그인 상태는 이 한 곳에서 관리한다
import { createContext, useCallback, useContext, useEffect, useState, type ReactNode } from 'react';
import { useNavigate } from 'react-router-dom';
import type { CurrentUser } from '@eduhub/api-client';
import { authApi } from '../api/clients';
import { tokenStore, UNAUTHORIZED_EVENT } from '../api/http';

interface AuthState {
  user: CurrentUser | null;
  /** 토큰이 만료·위조되어 로그아웃된 경우 true (로그인 화면에 안내를 띄운다) */
  expired: boolean;
  login: (employeeNo: string, password: string) => Promise<void>;
  logout: () => void;
}
const AuthContext = createContext<AuthState | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const navigate = useNavigate();
  const [user, setUser] = useState<CurrentUser | null>(tokenStore.user());
  const [expired, setExpired] = useState(false);

  const login = useCallback(async (employeeNo: string, password: string) => {
    const { data } = await authApi.login({ employeeNo, password });
    tokenStore.save(data.accessToken, data.user);
    setExpired(false);
    setUser(data.user);
  }, []);

  const logout = useCallback(() => {
    tokenStore.clear();
    setUser(null);
    navigate('/login');
  }, [navigate]);

  // 어느 화면에서든 401이 나면 http가 이벤트를 보낸다 → 로그인 화면으로
  useEffect(() => {
    const onUnauthorized = () => {
      // 보호 화면이면 RequireAuth도 로그인으로 보낸다. 어느 쪽이 먼저 옮기든 안내가 남도록 expired를 상태로 둔다
      setExpired(true);
      setUser(null);
      navigate('/login', { replace: true, state: { from: window.location.pathname } });
    };
    window.addEventListener(UNAUTHORIZED_EVENT, onUnauthorized);
    return () => window.removeEventListener(UNAUTHORIZED_EVENT, onUnauthorized);
  }, [navigate]);

  // 새로고침 후에도 토큰이 아직 유효한지 서버에 확인 (만료됐으면 401 → 위 이벤트)
  useEffect(() => {
    if (tokenStore.get()) authApi.me().then((r) => setUser(r.data)).catch(() => undefined);
  }, []);

  return <AuthContext.Provider value={{ user, expired, login, logout }}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('AuthProvider 안에서만 쓸 수 있습니다.');
  return ctx;
}
```

```tsx
// frontend/apps/course-mfe/src/auth/RequireAuth.tsx — 로그인(·관리자)이 필요한 라우트를 감싼다
import type { ReactNode } from 'react';
import { Navigate, useLocation } from 'react-router-dom';
import { Alert } from '@mui/material';
import { useAuth } from './AuthContext';

export function RequireAuth({ children, admin = false }: { children: ReactNode; admin?: boolean }) {
  const { user } = useAuth();
  const location = useLocation();
  if (!user) return <Navigate to="/login" replace state={{ from: location.pathname }} />;
  if (admin && user.role !== 'ADMIN') return <Alert severity="warning">관리자만 사용할 수 있는 화면입니다.</Alert>;
  return <>{children}</>;
}
```

```tsx
// frontend/apps/course-mfe/src/auth/LoginPage.tsx
import { useState, type FormEvent } from 'react';
import { useLocation, useNavigate } from 'react-router-dom';
import { Alert, Button, Paper, Stack, TextField, Typography } from '@mui/material';
import { useAuth } from './AuthContext';

export default function LoginPage() {
  const { login, expired } = useAuth();
  const navigate = useNavigate();
  const state = useLocation().state as { from?: string } | null;
  const [employeeNo, setEmployeeNo] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState(false);

  const submit = async (e: FormEvent) => {
    e.preventDefault();
    setError(false);
    try {
      await login(employeeNo, password);
      navigate(state?.from ?? '/courses', { replace: true });   // 원래 가려던 화면으로
    } catch {
      setError(true);
    }
  };

  return (
    <Paper sx={{ p: 4, maxWidth: 400, mx: 'auto', mt: 6 }}>
      <form onSubmit={submit}>
        <Stack spacing={2}>
          <Typography variant="h5">로그인</Typography>
          {expired && <Alert severity="info">로그인이 만료되었습니다. 다시 로그인하세요.</Alert>}
          {error && <Alert severity="error">사번 또는 비밀번호가 올바르지 않습니다.</Alert>}
          <TextField label="사번" value={employeeNo} onChange={(e) => setEmployeeNo(e.target.value)} required />
          <TextField label="비밀번호" type="password" value={password} onChange={(e) => setPassword(e.target.value)} required />
          <Button type="submit" variant="contained">로그인</Button>
        </Stack>
      </form>
    </Paper>
  );
}
```

</details>

<details>
<summary>main.tsx, theme.ts, App.tsx (완성본: Step 15·16의 화면 라우트까지 포함)</summary>

```tsx
// frontend/apps/course-mfe/src/main.tsx — 앱 시작점: 테마 → 라우터 → 로그인 상태 → 화면
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { CssBaseline, ThemeProvider } from '@mui/material';
import { AuthProvider } from './auth/AuthContext';
import { theme } from './theme';
import App from './App';

createRoot(document.getElementById('root')!).render(
  <ThemeProvider theme={theme}>
    <CssBaseline />
    <BrowserRouter>
      <AuthProvider>
        <App />
      </AuthProvider>
    </BrowserRouter>
  </ThemeProvider>,
);
```

```ts
// frontend/apps/course-mfe/src/theme.ts — MUI 테마는 한 곳에서 (6회차에 shell로 옮긴다)
import { createTheme } from '@mui/material';

export const theme = createTheme({ palette: { primary: { main: '#B5591C' } } });
```

```tsx
// frontend/apps/course-mfe/src/App.tsx — 상단 바와 라우팅. 로그인·관리자 확인은 RequireAuth로
import { Link as RouterLink, Navigate, Route, Routes } from 'react-router-dom';
import { AppBar, Button, Container, Toolbar, Typography } from '@mui/material';
import { useAuth } from './auth/AuthContext';
import { RequireAuth } from './auth/RequireAuth';
import LoginPage from './auth/LoginPage';
import { CourseListPage } from './pages/CourseListPage';
import CourseDetailPage from './pages/CourseDetailPage';
import CourseFormPage from './pages/CourseFormPage';
import MyEnrollmentsPage from './enrollment/MyEnrollmentsPage';
import RosterPage from './enrollment/RosterPage';

export default function App() {
  const { user, logout } = useAuth();
  const isAdmin = user?.role === 'ADMIN';
  return (
    <>
      <AppBar position="static">
        <Toolbar sx={{ gap: 1 }}>
          <Typography variant="h6" sx={{ flexGrow: 1 }}>EduHub</Typography>
          <Button color="inherit" component={RouterLink} to="/courses">교육과정</Button>
          {user && <Button color="inherit" component={RouterLink} to="/my/enrollments">내 신청 내역</Button>}
          {isAdmin && <Button color="inherit" component={RouterLink} to="/admin/courses/new">과정 등록</Button>}
          {user
            ? <><Typography variant="body2">{user.name}{isAdmin ? ' (관리자)' : ''}</Typography><Button color="inherit" onClick={logout}>로그아웃</Button></>
            : <Button color="inherit" component={RouterLink} to="/login">로그인</Button>}
        </Toolbar>
      </AppBar>
      <Container sx={{ mt: 3 }}>
        <Routes>
          <Route path="/" element={<Navigate to="/courses" replace />} />
          <Route path="/login" element={<LoginPage />} />
          <Route path="/courses" element={<CourseListPage />} />
          <Route path="/courses/:courseId" element={<CourseDetailPage />} />
          <Route path="/my/enrollments" element={<RequireAuth><MyEnrollmentsPage /></RequireAuth>} />
          <Route path="/admin/courses/new" element={<RequireAuth admin><CourseFormPage /></RequireAuth>} />
          <Route path="/admin/courses/:courseId/edit" element={<RequireAuth admin><CourseFormPage /></RequireAuth>} />
          <Route path="/admin/sessions/:sessionId/roster" element={<RequireAuth admin><RosterPage /></RequireAuth>} />
        </Routes>
      </Container>
    </>
  );
}
```

</details>

<details>
<summary>테스트: http.test.ts (토큰 첨부, 401 처리, 403은 로그아웃 안 함)</summary>

```ts
// frontend/apps/course-mfe/src/api/http.test.ts
import type { AxiosAdapter, InternalAxiosRequestConfig } from 'axios';
import { beforeEach, expect, test, vi } from 'vitest';
import { http, tokenStore, UNAUTHORIZED_EVENT } from './http';

// 네트워크 대신 응답을 흉내 내는 어댑터: 받은 요청을 기록하고 정해진 상태 코드로 답한다
let sent: InternalAxiosRequestConfig[] = [];
const respondWith = (status: number): AxiosAdapter => async (config) => {
  sent.push(config);
  const response = { data: {}, status, statusText: String(status), headers: {}, config };
  if (status >= 400) throw Object.assign(new Error(`HTTP ${status}`), { isAxiosError: true, response, config });
  return response;
};

beforeEach(() => {
  sent = [];
  localStorage.clear();
});

test('저장된 토큰을 Authorization 헤더로 붙인다', async () => {
  tokenStore.save('abc.def.ghi', { employeeId: 1, name: '김사원', role: 'EMPLOYEE' });
  await http.get('/api/me', { adapter: respondWith(200) });
  expect(sent[0].headers.Authorization).toBe('Bearer abc.def.ghi');
});

test('토큰이 없으면 헤더를 붙이지 않는다', async () => {
  await http.get('/api/v1/courses', { adapter: respondWith(200) });
  expect(sent[0].headers.Authorization).toBeUndefined();
});

test('401이면 토큰을 지우고 로그인 필요 이벤트를 보낸다', async () => {
  tokenStore.save('expired', { employeeId: 1, name: '김사원', role: 'EMPLOYEE' });
  const listener = vi.fn();
  window.addEventListener(UNAUTHORIZED_EVENT, listener);

  await expect(http.get('/api/me', { adapter: respondWith(401) })).rejects.toThrow('HTTP 401');

  expect(tokenStore.get()).toBeNull();
  expect(listener).toHaveBeenCalledTimes(1);
  window.removeEventListener(UNAUTHORIZED_EVENT, listener);
});

test('403(권한 없음)은 로그아웃시키지 않는다', async () => {
  tokenStore.save('valid', { employeeId: 1, name: '김사원', role: 'EMPLOYEE' });
  await expect(http.get('/api/admin/sessions/1/roster', { adapter: respondWith(403) })).rejects.toThrow('HTTP 403');
  expect(tokenStore.get()).toBe('valid');
});
```

</details>

> 💡 `AuthContext`의 `expired`를 **상태로** 두는 이유(실제로 겪은 문제): 401 이벤트에서 `navigate('/login', { state: { expired: true } })`만 하면 "로그인이 만료되었습니다" 안내가 사라졌습니다. React Router 7은 화면 이동을 낮은 우선순위(transition)로 처리해서, `setUser(null)`이 먼저 반영되고 `RequireAuth`가 `/login`으로 **다시** 보내며 이동 상태를 덮어썼기 때문입니다. 누가 먼저 화면을 옮기든 안내가 남도록 만료 여부를 컨텍스트 상태로 바꿨습니다.

> ⚠️ **화면의 권한 확인은 편의일 뿐입니다.** `RequireAuth`나 "로그인 후 신청" 버튼은 길을 안내할 뿐이고, 실제로 막는 것은 백엔드(401·403, Step 9-1)입니다. 화면만 막고 API를 열어 두면 누구나 curl로 부를 수 있습니다.

> 🔐 **토큰을 localStorage에 두는 것의 위험**: 페이지에 악성 스크립트가 들어오면(XSS) 토큰을 읽어 갈 수 있습니다. 교육에서는 구조를 단순하게 하려고 localStorage를 쓰고 토큰 만료를 60분으로 짧게 둡니다. 실제 서비스에서는 `HttpOnly` 쿠키, 짧은 만료 + 리프레시 토큰, 사내 SSO를 함께 검토합니다 (8회차 보안 점검에서 다시 봅니다).

**직접 확인** (원칙 ②: 사람이 직접 실행할 수 있어야 한다)

```bash
# 백엔드: Step 9-1의 dev 프로필 (EDUHUB_JWT_SECRET 설정 후)
cd frontend && npm run dev --workspace apps/course-mfe     # http://localhost:3001
```

- 상단 바의 **로그인** → `E1001` / `dev-password` → 이름이 보이고 **내 신청 내역** 메뉴가 생긴다
- 비밀번호를 틀리면 "사번 또는 비밀번호가 올바르지 않습니다."
- 로그아웃 후 주소창에 `/my/enrollments` → 로그인 화면 → 로그인하면 **원래 가려던 곳**으로 (화면은 Step 15에서 만든다)
- 개발자 도구 Application 탭의 Local Storage에서 `eduhub.accessToken`을 확인하고, Network 탭에서 요청에 `Authorization` 헤더가 붙는지 본다

커밋: `feat(frontend): 로그인과 토큰 처리, 보호 라우트 (NFR-03)`

---

## Step 15. 수강생 화면: 과정 상세, 신청, 내 신청 내역 (FR-02, FR-04~07)

```text
[목표] 수강생이 과정 상세에서 회차를 신청하고, 내 신청 내역에서 확인·취소한다
[위치] @frontend/apps/course-mfe/src/pages, @frontend/apps/course-mfe/src/enrollment
[범위]
- CourseListPage(3회차): 행을 누르면 /courses/:courseId. useNavigate를 쓰므로 기존 화면 테스트는 MemoryRouter로 감싼다
- CourseDetailPage(FR-02): 과정 정보, 종료 표시, 회차 표(시작일, 장소, 남은 자리/정원). 회차마다 EnrollButton(종료된 과정이면 없음).
  관리자면 "과정 수정" 버튼과 회차별 "명단" 링크
- enrollment/EnrollButton({ sessionId }): 로그인 안 했으면 "로그인 후 신청"(/login 링크).
  201 → "신청이 확정되었습니다" 또는 "대기 N번으로 등록되었습니다", 409 → "이미 신청한 회차입니다"
- enrollment/MyEnrollmentsPage(FR-07, FR-06): 상태 필터(전체·확정·대기·취소·수료), 표(과정, 시작일, 장소, 상태 칩 — 대기는 "대기 N번"),
  확정·대기는 확인 후 취소하고 목록을 다시 불러온다
- App.tsx에 /courses/:courseId, /my/enrollments(RequireAuth) 라우트
- 테스트(Vitest): EnrollButton — 로그인 안 함, 대기 순번, 409
[하지 않는 것] 관리자 화면(Step 16), 백엔드 수정
[완료 기준] npm run build·npm test 통과, 6-2 확인 표의 1~6이 브라우저에서 동작
[진행 방식] 계획 먼저
```

**기대하는 결과 (핵심 코드)**

```tsx
// frontend/apps/course-mfe/src/pages/CourseListPage.tsx — 5회차: 행을 누르면 상세로 (바뀐 곳에 ★)
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';                       // ★
import {
  Alert, CircularProgress, Stack, Table, TableBody, TableCell, TableHead, TablePagination, TableRow, TextField,
} from '@mui/material';
import { useCourses } from '../hooks/useCourses';

export function CourseListPage() {
  const navigate = useNavigate();                                       // ★
  const [keyword, setKeyword] = useState('');
  const [page, setPage] = useState(0); // MUI TablePagination도 0부터 시작
  const { data, loading, error } = useCourses({ keyword, page, size: 10 });

  if (error) return <Alert severity="error">과정 목록을 불러오지 못했습니다.</Alert>;

  return (
    <Stack spacing={2}>
      <TextField
        label="과정명 검색"
        value={keyword}
        onChange={(e) => {
          setKeyword(e.target.value);
          setPage(0); // 검색어가 바뀌면 첫 페이지부터
        }}
      />
      {loading ? (
        <CircularProgress />
      ) : (
        <Table>
          <TableHead>
            <TableRow>
              <TableCell>과정명</TableCell>
              <TableCell>분류</TableCell>
              <TableCell>회차 수</TableCell>
            </TableRow>
          </TableHead>
          <TableBody>
            {data?.content.map((c) => (
              <TableRow key={c.id} hover sx={{ cursor: 'pointer' }} onClick={() => navigate(`/courses/${c.id}`)}>  {/* ★ */}
                <TableCell>{c.title}</TableCell>
                <TableCell>{c.category}</TableCell>
                <TableCell>{c.sessionCount}</TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      )}
      <TablePagination
        component="div"
        count={data?.totalElements ?? 0}
        page={page}
        rowsPerPage={10}
        rowsPerPageOptions={[10]}
        onPageChange={(_, p) => setPage(p)}
      />
    </Stack>
  );
}
```

```tsx
// frontend/apps/course-mfe/src/pages/CourseDetailPage.tsx — FR-02 과정 상세와 회차, 회차마다 신청 버튼
import { useEffect, useState } from 'react';
import { Link as RouterLink, useParams } from 'react-router-dom';
import { Alert, Button, Chip, CircularProgress, Stack, Table, TableBody, TableCell, TableHead, TableRow, Typography } from '@mui/material';
import type { CourseDetailResponse } from '@eduhub/api-client';
import { courseApi } from '../api/courseApi';
import { useAuth } from '../auth/AuthContext';
import EnrollButton from '../enrollment/EnrollButton';

export default function CourseDetailPage() {
  const { courseId } = useParams();
  const { user } = useAuth();
  const isAdmin = user?.role === 'ADMIN';
  const [course, setCourse] = useState<CourseDetailResponse>();
  const [error, setError] = useState(false);

  useEffect(() => {
    courseApi.getCourse(Number(courseId)).then((r) => setCourse(r.data)).catch(() => setError(true));
  }, [courseId]);

  if (error) return <Alert severity="error">과정 정보를 불러오지 못했습니다.</Alert>;
  if (!course) return <CircularProgress />;
  const closed = course.status === 'CLOSED';
  return (
    <Stack spacing={2}>
      <Stack direction="row" spacing={2} alignItems="center">
        <Typography variant="h5">{course.title}</Typography>
        {closed && <Chip label="종료" size="small" />}
        {isAdmin && <Button component={RouterLink} to={`/admin/courses/${course.id}/edit`} size="small">과정 수정</Button>}
      </Stack>
      <Typography color="text.secondary">{course.description}</Typography>
      <Table>
        <TableHead>
          <TableRow><TableCell>시작일</TableCell><TableCell>장소</TableCell><TableCell>남은 자리</TableCell><TableCell /></TableRow>
        </TableHead>
        <TableBody>
          {course.sessions.map((s) => (
            <TableRow key={s.id}>
              <TableCell>{s.startDate}</TableCell>
              <TableCell>{s.place}</TableCell>
              <TableCell>{s.remaining} / {s.capacity}</TableCell>
              <TableCell>
                <Stack direction="row" spacing={1} alignItems="center">
                  {!closed && <EnrollButton sessionId={s.id} />}
                  {isAdmin && <Button component={RouterLink} to={`/admin/sessions/${s.id}/roster`} size="small">명단</Button>}
                </Stack>
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </Stack>
  );
}
```

```tsx
// frontend/apps/course-mfe/src/enrollment/EnrollButton.tsx — FR-04·05 신청 버튼 (과정 상세의 회차 행에 들어간다)
import { useState } from 'react';
import { Link as RouterLink } from 'react-router-dom';
import { Alert, Button, Stack } from '@mui/material';
import axios from 'axios';
import type { EnrollmentResult } from '@eduhub/api-client';
import { enrollmentApi } from '../api/clients';
import { useAuth } from '../auth/AuthContext';

export default function EnrollButton({ sessionId }: { sessionId: number }) {
  const { user } = useAuth();
  const [result, setResult] = useState<EnrollmentResult>();
  const [error, setError] = useState<string>();
  const [busy, setBusy] = useState(false);

  // 로그인하지 않았으면 신청 대신 로그인으로 (화면 안내일 뿐, 최종 판단은 백엔드가 401로 한다)
  if (!user) {
    return <Button component={RouterLink} to="/login" size="small">로그인 후 신청</Button>;
  }
  if (result) {
    return result.status === 'CONFIRMED'
      ? <Alert severity="success">신청이 확정되었습니다.</Alert>
      : <Alert severity="info">대기 {result.waitingNo}번으로 등록되었습니다.</Alert>;
  }

  const enroll = async () => {
    setBusy(true);
    setError(undefined);
    try {
      setResult((await enrollmentApi.enroll(sessionId)).data);
    } catch (e) {
      setError(axios.isAxiosError(e) && e.response?.status === 409 ? '이미 신청한 회차입니다.' : '신청하지 못했습니다.');
    } finally {
      setBusy(false);
    }
  };

  return (
    <Stack direction="row" spacing={1} alignItems="center">
      <Button variant="contained" size="small" disabled={busy} onClick={enroll}>신청</Button>
      {error && <Alert severity="warning">{error}</Alert>}
    </Stack>
  );
}
```

<details>
<summary>MyEnrollmentsPage.tsx</summary>

```tsx
// frontend/apps/course-mfe/src/enrollment/MyEnrollmentsPage.tsx — FR-07 내 신청 내역(상태별) + FR-06 취소
import { useCallback, useEffect, useState } from 'react';
import { Alert, Button, Chip, CircularProgress, MenuItem, Stack, Table, TableBody, TableCell, TableHead, TableRow, TextField, Typography } from '@mui/material';
import type { EnrollmentStatus, MyEnrollmentResponse } from '@eduhub/api-client';
import { enrollmentApi } from '../api/clients';

const LABEL: Record<EnrollmentStatus, string> = { CONFIRMED: '확정', WAITING: '대기', CANCELED: '취소', COMPLETED: '수료' };

export default function MyEnrollmentsPage() {
  const [status, setStatus] = useState<EnrollmentStatus | ''>('');
  const [rows, setRows] = useState<MyEnrollmentResponse[]>();
  const [error, setError] = useState(false);

  const load = useCallback(() => {
    enrollmentApi.getMyEnrollments(status || undefined).then((r) => setRows(r.data)).catch(() => setError(true));
  }, [status]);
  useEffect(load, [load]);

  const cancel = async (id: number) => {
    if (!window.confirm('신청을 취소할까요? 대기 1순위가 있으면 자동으로 확정됩니다.')) return;
    await enrollmentApi.cancel(id);
    load();
  };

  if (error) return <Alert severity="error">신청 내역을 불러오지 못했습니다.</Alert>;
  if (!rows) return <CircularProgress />;
  return (
    <Stack spacing={2}>
      <Typography variant="h5">내 신청 내역</Typography>
      <TextField select label="상태" value={status} onChange={(e) => setStatus(e.target.value as EnrollmentStatus | '')} sx={{ width: 160 }}>
        <MenuItem value="">전체</MenuItem>
        {Object.entries(LABEL).map(([value, label]) => <MenuItem key={value} value={value}>{label}</MenuItem>)}
      </TextField>
      {rows.length === 0 && <Alert severity="info">신청한 과정이 없습니다.</Alert>}
      <Table>
        <TableHead><TableRow><TableCell>과정</TableCell><TableCell>시작일</TableCell><TableCell>장소</TableCell><TableCell>상태</TableCell><TableCell /></TableRow></TableHead>
        <TableBody>
          {rows.map((r) => (
            <TableRow key={r.enrollmentId}>
              <TableCell>{r.courseTitle}</TableCell>
              <TableCell>{r.startDate}</TableCell>
              <TableCell>{r.place}</TableCell>
              <TableCell>
                <Chip size="small" label={r.status === 'WAITING' ? `대기 ${r.waitingNo}번` : LABEL[r.status]} />
              </TableCell>
              <TableCell>
                {(r.status === 'CONFIRMED' || r.status === 'WAITING') && (
                  <Button size="small" color="error" onClick={() => cancel(r.enrollmentId)}>취소</Button>
                )}
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </Stack>
  );
}
```

</details>

<details>
<summary>테스트: EnrollButton.test.tsx, CourseListPage.test.tsx(MemoryRouter로 감싸기)</summary>

```tsx
// frontend/apps/course-mfe/src/enrollment/EnrollButton.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter } from 'react-router-dom';
import { beforeEach, expect, test, vi } from 'vitest';
import { enrollmentApi } from '../api/clients';
import { useAuth } from '../auth/AuthContext';
import EnrollButton from './EnrollButton';

vi.mock('../api/clients', () => ({ enrollmentApi: { enroll: vi.fn() } }));
vi.mock('../auth/AuthContext', () => ({ useAuth: vi.fn() }));
const enroll = vi.mocked(enrollmentApi.enroll);
const mockUseAuth = vi.mocked(useAuth);
const loggedIn = () =>
  mockUseAuth.mockReturnValue({ user: { employeeId: 1, name: '김사원', role: 'EMPLOYEE' }, expired: false, login: vi.fn(), logout: vi.fn() });

beforeEach(() => {
  enroll.mockReset();
  mockUseAuth.mockReturnValue({ user: null, expired: false, login: vi.fn(), logout: vi.fn() });
});

const renderButton = () => render(<MemoryRouter><EnrollButton sessionId={7} /></MemoryRouter>);

test('로그인하지 않았으면 신청 대신 로그인으로 안내한다', () => {
  renderButton();
  expect(screen.getByRole('link', { name: '로그인 후 신청' })).toHaveAttribute('href', '/login');
});

test('정원이 차 있으면 대기 순번을 보여 준다', async () => {
  loggedIn();
  enroll.mockResolvedValue({ data: { enrollmentId: 3, status: 'WAITING', waitingNo: 2 } } as never);

  renderButton();
  await userEvent.click(screen.getByRole('button', { name: '신청' }));

  expect(enroll).toHaveBeenCalledWith(7);
  expect(await screen.findByText('대기 2번으로 등록되었습니다.')).toBeInTheDocument();
});

test('이미 신청했으면(409) 그 이유를 보여 준다', async () => {
  loggedIn();
  enroll.mockRejectedValue(Object.assign(new Error('409'), { isAxiosError: true, response: { status: 409 } }));

  renderButton();
  await userEvent.click(screen.getByRole('button', { name: '신청' }));

  expect(await screen.findByText('이미 신청한 회차입니다.')).toBeInTheDocument();
});
```

```tsx
// frontend/apps/course-mfe/src/pages/CourseListPage.test.tsx — 5회차: useNavigate를 쓰므로 MemoryRouter로 감싼다 (★)
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';                       // ★
import { expect, test, vi } from 'vitest';
import { useCourses } from '../hooks/useCourses';
import { CourseListPage } from './CourseListPage';

// 화면 테스트는 훅을 mock한다. API 호출 방식은 useCourses.test.ts가 따로 검증한다
vi.mock('../hooks/useCourses');
const mockUseCourses = vi.mocked(useCourses);

test('과정 목록을 표로 보여 준다', () => {
  mockUseCourses.mockReturnValue({
    data: {
      content: [{ id: 1, title: 'Spring Boot 입문', category: 'BACKEND', status: 'OPEN', sessionCount: 2 }],
      page: 0, size: 10, totalElements: 1,
    },
    loading: false,
    error: undefined,
  });

  render(<MemoryRouter><CourseListPage /></MemoryRouter>);   // ★

  expect(screen.getByText('Spring Boot 입문')).toBeInTheDocument();
  expect(screen.getByText('BACKEND')).toBeInTheDocument();
});

test('불러오기에 실패하면 에러 메시지를 보여 준다', () => {
  mockUseCourses.mockReturnValue({ data: undefined, loading: false, error: new Error('fail') });

  render(<MemoryRouter><CourseListPage /></MemoryRouter>);   // ★

  expect(screen.getByRole('alert')).toHaveTextContent('과정 목록을 불러오지 못했습니다.');
});
```

</details>

**diff 검토 체크리스트**
- [ ] API 호출은 `src/api`의 인스턴스로만 하는가? 화면에서 `axios.create`·`fetch`·`localhost:8080`을 쓰지 않았는가?
- [ ] 신청 결과 문구가 백엔드 응답(`status`, `waitingNo`)에서 나오는가? 화면이 정원을 직접 계산하지 않는가?
- [ ] 취소 후 목록을 **다시 불러오는가**? (대기자 자동 확정은 서버가 한다)
- [ ] `enrollment/` 폴더의 파일이 `pages/`를 import하지 않는가? (6회차에 서로 다른 앱이 된다)

커밋: `feat(frontend): 과정 상세·신청·내 신청 내역 화면 (FR-02, FR-04~07)`

---

## Step 16. 관리자 화면: 과정 등록·수정 (FR-03), 명단·수료 (FR-08)

```text
[목표] 관리자가 과정을 등록·수정·종료하고, 회차 명단에서 수료 처리한다
[위치] @frontend/apps/course-mfe/src/pages/CourseFormPage.tsx, @frontend/apps/course-mfe/src/enrollment/RosterPage.tsx, App.tsx
[범위]
- CourseFormPage(FR-03): /admin/courses/new(등록 → 수정 화면으로 이동), /admin/courses/:courseId/edit(수정, 회차 목록·추가, 과정 종료).
  종료된 과정은 입력을 막고 "종료된 과정입니다". 서버 오류 메시지(409 등)를 그대로 보여 준다
- RosterPage(FR-08): 과정명·시작일, 확정자(이름·사번, 체크박스, 수료자는 "수료"), 대기자(대기 순번 순).
  선택한 사원 수료 처리 → "N명을 수료 처리했습니다." + 처리하지 않은 사원과 사유. 409(시작 전)면 서버가 준 이유. 403이면 "관리자만 볼 수 있습니다."
- App.tsx: 두 화면 라우트는 RequireAuth admin
- 테스트(Vitest): RosterPage — 명단 표시, 수료 결과, 409 이유
[하지 않는 것] 백엔드 수정
[완료 기준] npm run build·npm test 통과, 6-2 확인 표 전체가 브라우저에서 동작
[진행 방식] 계획 먼저
```

**기대하는 결과 (핵심 코드)**

<details>
<summary>CourseFormPage.tsx (FR-03)</summary>

```tsx
// frontend/apps/course-mfe/src/pages/CourseFormPage.tsx — FR-03 과정 등록·수정, 회차 추가, 과정 종료 (관리자)
import { useEffect, useState, type FormEvent } from 'react';
import { useNavigate, useParams } from 'react-router-dom';
import { Alert, Button, CircularProgress, Divider, Stack, TextField, Typography } from '@mui/material';
import axios from 'axios';
import type { CourseDetailResponse, CourseRequest } from '@eduhub/api-client';
import { courseApi } from '../api/courseApi';

const empty: CourseRequest = { title: '', category: '', description: '' };

export default function CourseFormPage() {
  const { courseId } = useParams();                 // 없으면 등록, 있으면 수정
  const id = courseId ? Number(courseId) : undefined;
  const navigate = useNavigate();
  const [form, setForm] = useState<CourseRequest>(empty);
  const [course, setCourse] = useState<CourseDetailResponse>();
  const [session, setSession] = useState({ startDate: '', place: '', capacity: 10 });
  const [message, setMessage] = useState<string>();
  const [error, setError] = useState<string>();

  const load = () => {
    if (!id) return;
    courseApi.getCourse(id).then(({ data }) => {
      setCourse(data);
      setForm({ title: data.title, category: data.category, description: data.description ?? '' });
    }).catch(() => setError('과정 정보를 불러오지 못했습니다.'));
  };
  useEffect(load, [id]);

  // 서버가 돌려준 이유(409 등)를 그대로 보여 준다
  const run = async (action: () => Promise<unknown>, done: string) => {
    setError(undefined);
    setMessage(undefined);
    try {
      await action();
      setMessage(done);
    } catch (e) {
      setError(axios.isAxiosError(e) && e.response?.data?.message ? e.response.data.message : '저장하지 못했습니다.');
    }
  };

  const save = (e: FormEvent) => {
    e.preventDefault();
    if (!id) {
      void run(async () => {
        const { data } = await courseApi.createCourse(form);
        navigate(`/admin/courses/${data.id}/edit`, { replace: true });   // 등록 후 회차를 추가할 수 있게 수정 화면으로
      }, '등록했습니다.');
    } else {
      void run(() => courseApi.updateCourse(id, form), '저장했습니다.');
    }
  };

  const addSession = (e: FormEvent) => {
    e.preventDefault();
    void run(async () => {
      await courseApi.addSession(id!, session);
      load();
    }, '회차를 추가했습니다.');
  };

  const close = () => {
    if (!window.confirm('과정을 종료할까요? 종료하면 수정·회차 추가·신청을 할 수 없습니다.')) return;
    void run(async () => {
      await courseApi.closeCourse(id!);
      load();
    }, '과정을 종료했습니다.');
  };

  if (id && !course && !error) return <CircularProgress />;
  const closed = course?.status === 'CLOSED';
  return (
    <Stack spacing={2} sx={{ maxWidth: 640 }}>
      <Typography variant="h5">{id ? '과정 수정' : '과정 등록'}</Typography>
      {message && <Alert severity="success">{message}</Alert>}
      {error && <Alert severity="error">{error}</Alert>}
      {closed && <Alert severity="info">종료된 과정입니다.</Alert>}
      <form onSubmit={save}>
        <Stack spacing={2}>
          <TextField label="과정명" value={form.title} onChange={(e) => setForm({ ...form, title: e.target.value })} required disabled={closed} />
          <TextField label="분류" value={form.category} onChange={(e) => setForm({ ...form, category: e.target.value })} required disabled={closed} />
          <TextField label="설명" value={form.description} onChange={(e) => setForm({ ...form, description: e.target.value })} multiline minRows={2} disabled={closed} />
          <Button type="submit" variant="contained" disabled={closed}>{id ? '저장' : '등록'}</Button>
        </Stack>
      </form>

      {id && course && (
        <>
          <Divider />
          <Typography variant="h6">회차 ({course.sessions.length}개)</Typography>
          {course.sessions.map((s) => <Typography key={s.id}>{s.startDate} · {s.place} · 정원 {s.capacity}</Typography>)}
          {!closed && (
            <form onSubmit={addSession}>
              <Stack direction="row" spacing={1}>
                <TextField label="시작일" type="date" value={session.startDate} onChange={(e) => setSession({ ...session, startDate: e.target.value })}
                  required slotProps={{ inputLabel: { shrink: true } }} />
                <TextField label="장소" value={session.place} onChange={(e) => setSession({ ...session, place: e.target.value })} required />
                <TextField label="정원" type="number" value={session.capacity} onChange={(e) => setSession({ ...session, capacity: Number(e.target.value) })}
                  required slotProps={{ htmlInput: { min: 1 } }} />
                <Button type="submit">회차 추가</Button>
              </Stack>
            </form>
          )}
          <Divider />
          <Button color="error" onClick={close} disabled={closed} sx={{ alignSelf: 'flex-start' }}>과정 종료</Button>
        </>
      )}
    </Stack>
  );
}
```

</details>

```tsx
// frontend/apps/course-mfe/src/enrollment/RosterPage.tsx — FR-08 회차 명단과 수료 처리 (관리자)
import { useCallback, useEffect, useState } from 'react';
import { useParams } from 'react-router-dom';
import { Alert, Button, Checkbox, Chip, CircularProgress, List, ListItem, ListItemText, Stack, Typography } from '@mui/material';
import axios from 'axios';
import type { CompletionResult, RosterResponse } from '@eduhub/api-client';
import { adminEnrollmentApi } from '../api/clients';

/** 서버가 돌려준 오류 메시지(409 등)를 우선 보여 준다 */
const messageOf = (e: unknown, fallback: string) =>
  axios.isAxiosError(e) && e.response?.data?.message ? (e.response.data.message as string) : fallback;

export default function RosterPage() {
  const sessionId = Number(useParams().sessionId);
  const [roster, setRoster] = useState<RosterResponse>();
  const [error, setError] = useState<string>();
  const [checked, setChecked] = useState<number[]>([]);
  const [result, setResult] = useState<CompletionResult>();

  const load = useCallback(() => {
    adminEnrollmentApi.getRoster(sessionId).then((r) => setRoster(r.data))
      .catch((e) => setError(axios.isAxiosError(e) && e.response?.status === 403 ? '관리자만 볼 수 있습니다.' : '명단을 불러오지 못했습니다.'));
  }, [sessionId]);
  useEffect(load, [load]);

  const toggle = (id: number) => setChecked((c) => (c.includes(id) ? c.filter((x) => x !== id) : [...c, id]));
  const complete = async () => {
    setError(undefined);
    try {
      setResult((await adminEnrollmentApi.complete(sessionId, { employeeIds: checked })).data);
      setChecked([]);
      load();
    } catch (e) {
      setError(messageOf(e, '수료 처리하지 못했습니다.'));
    }
  };

  if (!roster) return error ? <Alert severity="error">{error}</Alert> : <CircularProgress />;
  return (
    <Stack spacing={2}>
      <Typography variant="h5">{roster.courseTitle} · {roster.startDate} 명단</Typography>
      {error && <Alert severity="error">{error}</Alert>}
      {result && (
        <Alert severity={result.failed.length ? 'warning' : 'success'}>
          {result.completed.length}명을 수료 처리했습니다.
          {result.failed.map((f) => <div key={f.employeeId}>처리하지 않음: 사원 {f.employeeId} ({f.reason})</div>)}
        </Alert>
      )}
      <Typography variant="h6">확정 {roster.confirmed.length}명 / 정원 {roster.capacity}</Typography>
      <List dense>
        {roster.confirmed.map((r) => (
          <ListItem key={r.enrollmentId} disablePadding>
            <Checkbox checked={checked.includes(r.employeeId)} onChange={() => toggle(r.employeeId)}
              disabled={r.status === 'COMPLETED'} inputProps={{ 'aria-label': `${r.name} 선택` }} />
            <ListItemText primary={`${r.name} (${r.employeeNo})`} />
            {r.status === 'COMPLETED' && <Chip size="small" label="수료" />}
          </ListItem>
        ))}
      </List>
      <Typography variant="h6">대기 {roster.waiting.length}명</Typography>
      <List dense>
        {roster.waiting.map((r) => <ListItem key={r.enrollmentId}><ListItemText primary={`대기 ${r.waitingNo}번 · ${r.name} (${r.employeeNo})`} /></ListItem>)}
      </List>
      <Button variant="contained" disabled={checked.length === 0} onClick={complete} sx={{ alignSelf: 'flex-start' }}>
        선택한 {checked.length}명 수료 처리
      </Button>
    </Stack>
  );
}
```

<details>
<summary>테스트: RosterPage.test.tsx</summary>

```tsx
// frontend/apps/course-mfe/src/enrollment/RosterPage.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter, Route, Routes } from 'react-router-dom';
import { beforeEach, expect, test, vi } from 'vitest';
import { adminEnrollmentApi } from '../api/clients';
import RosterPage from './RosterPage';

vi.mock('../api/clients', () => ({ adminEnrollmentApi: { getRoster: vi.fn(), complete: vi.fn() } }));
const getRoster = vi.mocked(adminEnrollmentApi.getRoster);
const complete = vi.mocked(adminEnrollmentApi.complete);

const roster = {
  sessionId: 4, courseTitle: 'Git 협업 기초', startDate: '2026-11-15', capacity: 10,
  confirmed: [{ enrollmentId: 1, employeeId: 1, employeeNo: 'E1001', name: '김사원', status: 'CONFIRMED' as const }],
  waiting: [{ enrollmentId: 2, employeeId: 2, employeeNo: 'E1002', name: '이사원', status: 'WAITING' as const, waitingNo: 1 }],
};

beforeEach(() => {
  getRoster.mockReset();
  complete.mockReset();
  getRoster.mockResolvedValue({ data: roster } as never);
});

const renderPage = () =>
  render(<MemoryRouter initialEntries={['/admin/sessions/4/roster']}>
    <Routes><Route path="/admin/sessions/:sessionId/roster" element={<RosterPage />} /></Routes>
  </MemoryRouter>);

test('확정자와 대기 순번을 이름과 함께 보여 준다', async () => {
  renderPage();
  expect(await screen.findByText('김사원 (E1001)')).toBeInTheDocument();
  expect(screen.getByText('대기 1번 · 이사원 (E1002)')).toBeInTheDocument();
  expect(getRoster).toHaveBeenCalledWith(4);
});

test('선택한 사원을 수료 처리하고 결과를 보여 준다', async () => {
  complete.mockResolvedValue({ data: { completed: [1], failed: [] } } as never);
  renderPage();
  await userEvent.click(await screen.findByLabelText('김사원 선택'));
  await userEvent.click(screen.getByRole('button', { name: '선택한 1명 수료 처리' }));

  expect(complete).toHaveBeenCalledWith(4, { employeeIds: [1] });
  expect(await screen.findByText('1명을 수료 처리했습니다.')).toBeInTheDocument();
});

test('시작 전 회차(409)면 서버가 준 이유를 보여 준다', async () => {
  complete.mockRejectedValue(Object.assign(new Error('409'), {
    isAxiosError: true, response: { status: 409, data: { message: '회차 시작일(2026-11-25) 전에는 수료 처리할 수 없습니다.' } },
  }));
  renderPage();
  await userEvent.click(await screen.findByLabelText('김사원 선택'));
  await userEvent.click(screen.getByRole('button', { name: '선택한 1명 수료 처리' }));

  expect(await screen.findByText('회차 시작일(2026-11-25) 전에는 수료 처리할 수 없습니다.')).toBeInTheDocument();
});
```

</details>

커밋 후 PR을 올립니다: `feat(frontend): 과정 관리·명단·수료 화면 (FR-03, FR-08)`. PR 본문에 6-2 확인 표를 직접 해 본 결과를 적습니다.

---

## 6-2. 데모 흐름으로 확인하기

백엔드(dev 프로필)와 `npm run dev --workspace apps/course-mfe`를 띄우고 http://localhost:3001 에서 순서대로 해 봅니다. dev 프로필은 시작할 때 과정 3개(1: Spring Boot 입문 — 1번 회차 정원 1명, 2: JPA 실전, 3: Git 협업 기초 — 이미 시작한 4번 회차에 김사원·이사원 확정)를 넣습니다.

| # | 할 일 | 기대 결과 |
|---|---|---|
| 1 | 로그인하지 않고 `/courses` → **Spring Boot 입문** 클릭 | 상세와 회차 표, 회차마다 **로그인 후 신청** |
| 2 | 로그인하지 않고 `/my/enrollments` | `/login`으로 이동 → `E1001` 로그인 → **내 신청 내역**으로 돌아오고 Git 협업 기초(확정)가 보임 |
| 3 | `E1001`로 1번 회차(본관 3층, 정원 1) 신청 | "신청이 확정되었습니다." |
| 4 | 로그아웃 → `E1002`로 같은 회차 신청 | "대기 1번으로 등록되었습니다.", 내 신청 내역 상태 필터 **대기**에서 대기 1번 한 줄 |
| 5 | `E1002`로 `/admin/sessions/1/roster` | "관리자만 사용할 수 있는 화면입니다." (백엔드도 403) |
| 6 | `E1001`로 Spring Boot 입문 신청 **취소** | 상태가 **취소** → `E1002`로 로그인하면 **확정**(대기자 자동 확정) |
| 7 | `A0001`로 상단 **과정 등록** → 과정 저장 → 회차 추가 | 수정 화면으로 이동, "회차를 추가했습니다." |
| 8 | 같은 화면에서 **과정 종료** | "종료된 과정입니다", 저장 버튼 비활성. 상세에는 **종료** 표시, 신청 버튼 없음 |
| 9 | `A0001`로 1번 회차 명단 → 이사원 선택 → 수료 처리 | 시작 전이라 "회차 시작일(…) 전에는 수료 처리할 수 없습니다." (409) |
| 10 | `A0001`로 `/admin/sessions/4/roster` → 김사원 선택 → 수료 처리 | "1명을 수료 처리했습니다.", 김사원 옆에 **수료** |
| 11 | `E1001`로 내 신청 내역 | Git 협업 기초가 **수료** |
| 12 | 개발자 도구에서 `localStorage`의 `eduhub.accessToken`을 아무 값으로 바꾸고 새로고침 | 로그인 화면 + "로그인이 만료되었습니다", 토큰 삭제됨 |

> 💡 다시 처음부터 해 보려면 개발 DB를 비우고(예: 컨테이너의 `eduhub` DB를 지우고 다시 만들기) 백엔드를 다시 시작합니다. 개발용 데이터는 계정이 하나도 없을 때만 들어갑니다.

> ✅ 이 교재 환경에서 위 코드를 Step 9-1~11 백엔드(Spring Boot 3.5 + PostgreSQL 16)와 함께 실행하고, 브라우저(Chromium, Playwright)로 위 흐름(세분화한 20단계)을 개발 서버 모드와 빌드·미리보기 모드 모두에서 확인했습니다. 단위 테스트 14개(3회차 4개 + http 4, EnrollButton 3, RosterPage 3), 타입 검사, 빌드도 통과했습니다 (Vite 8.3, React 18.3, React Router 7.18, MUI 6.5, Vitest 5.1). 개발 서버 모드에서 네 번 중 한 번은 마지막 단계(만료 토큰 확인 뒤 다시 로그인)에서 브라우저 자동화가 시간 초과로 멈췄고, 다시 실행하면 통과해 원인을 확정하지 못했습니다. API 클라이언트는 openapi-generator가 만드는 것과 같은 모양의 스텁으로 대신했습니다.

## 6-3. 자주 겪는 문제 (화면)

| 증상 | 원인과 해결 |
|---|---|
| 브라우저 콘솔에 CORS 오류 | API를 `http://localhost:8080`으로 직접 부름. 상대 경로(`basePath: ''`) + Vite `server.proxy`·`preview.proxy`의 `/api` (Step 13) |
| `npm run dev`는 되는데 `npm run preview`에서 API가 404 | `preview.proxy`를 빠뜨림. 빌드 결과도 같은 프록시가 필요하다 |
| 로그인했는데 어떤 화면만 401 | 그 화면이 토큰을 붙이지 않는 Axios 인스턴스나 `fetch`를 씀. 모든 *Api는 `src/api`에서 `http`로 만든다 |
| 관리자 화면을 연 사원이 로그아웃됨 | 응답 인터셉터가 403에도 토큰을 지움. 401만 로그아웃 |
| 토큰이 만료돼도 "만료" 안내 없이 로그인 화면만 나옴 | React Router 7의 transition 때문에 이동 상태가 덮어써짐. 만료 여부를 `AuthContext` 상태로 (Step 14의 💡) |
| `useNavigate() may be used only in the context of a <Router> component` (테스트에서) | 화면 테스트가 라우터 없이 렌더링함. `MemoryRouter`로 감싼다 (Step 15) |
| `npm run build`에서 `useCourses.test.ts` 타입 오류 | 생성 클라이언트의 `AxiosResponse` 타입. `as never` (Step 13 ③) |
| 수료 처리가 항상 409 | 회차 시작일 전. dev 데이터의 4번 회차(이미 시작)로 확인한다 |
| 새로고침하면 로그인이 풀림 | `tokenStore`에 저장하지 않거나 `AuthProvider`의 초깃값을 `tokenStore.user()`로 두지 않음 |

---

# 7. 더 해 보기 (시간이 남으면 수업 중, 아니면 과제)

---

## 7-1. OpenAPI 누락 검사 훅

3장 토론의 E안을 직접 만듭니다.

```text
[목표] 컨트롤러를 수정했을 때 매핑 어노테이션(@GetMapping 등) 수와 @Operation 수가 다르면 Claude에게 알려주고 싶어.
[위치] .claude/hooks/check-openapi.sh, PostToolUse (matcher: Edit|Write)
[범위] 파일명이 *Controller.java일 때만 검사. 다르면 stderr에 "어느 메서드에 @Operation이 없는지"를 쓰고 exit 2
[완료 기준] @Operation을 하나 지운 컨트롤러로 셸 테스트 시 exit 2
```

```bash
#!/usr/bin/env bash
# .claude/hooks/check-openapi.sh — PostToolUse(Edit|Write): 컨트롤러 OpenAPI 누락 검사 (단순 버전)
FILE=$(jq -r '.tool_input.file_path // empty')
[[ "$FILE" != *Controller.java ]] && exit 0
MAPPINGS=$(grep -cE '@(Get|Post|Put|Patch|Delete)Mapping' "$FILE")
OPS=$(grep -c '@Operation' "$FILE")
if [ "$MAPPINGS" -gt "$OPS" ]; then
  echo "$FILE: API 메서드 ${MAPPINGS}개 중 @Operation이 ${OPS}개입니다. 빠진 메서드에 @Operation(summary=...)을 추가하세요." >&2
  exit 2
fi
exit 0
```

> 💡 PostToolUse의 종료 코드 2는 수정을 되돌리지 않고 **Claude에게 알려 고치게** 합니다. 이 훅은 "존재"만 검사합니다. 설명의 "품질"은 스킬과 리뷰가 맡습니다.

---

## 7-2. 테스트 출력 줄이기 훅 (개인 설정)

**무엇을 하나요?** Claude가 `./gradlew test` 같은 테스트 명령을 실행하면 수백 줄의 빌드·테스트 출력이 그대로 **도구 결과로 컨텍스트에 들어갑니다.** 이 훅은 Claude가 테스트 명령을 실행하기 **직전에(PreToolUse)** 명령을 살짝 바꿔서, 전체 출력은 로그 파일에 저장하고 Claude에게는 **실패 부분과 요약만** 보여 주게 합니다.

```mermaid
flowchart LR
    A["Claude: cd backend && ./gradlew test<br/>실행 요청"] --> H{"PreToolUse 훅<br/>filter-test-output.sh"}
    H -->|테스트 명령이면| B["바뀐 명령 실행<br/>전체 출력 → 로그 파일<br/>화면 → 실패 줄 + 요약 + exit 코드"]
    H -->|그 밖의 명령| C["원래 명령 그대로"]
    B --> R["Claude가 받는 결과: 수십 줄"]
```

**왜 개인 설정(`~/.claude/settings.json`)인가?** 이 훅은 출력을 숨기고 테스트 명령을 **자동 승인**합니다. 팀 전체에 바로 적용하기에는 영향이 크므로 먼저 **내 PC에서만** 실험하고, 효과가 확인되면 팀에서 논의합니다.

**오늘 할 일**: 같은 실패를 **훅 없이 한 번, 훅을 켜고 한 번** 실행해 컨텍스트 사용량을 비교합니다 (약 15분).

### ① 실험 준비: 테스트 하나를 일부러 깨기

```bash
git switch main && git pull
git switch -c experiment/test-output-hook      # 실험용 브랜치. 병합하지 않는다
```

`backend/src/test/.../EnrollmentServiceTest.java`에서 정원 검사 테스트의 기대값 하나를 **직접** 틀리게 바꿉니다 (예: `WAITING` → `CONFIRMED`). Claude에게 시키지 않고 직접 바꾸는 이유는 두 번의 측정 조건을 똑같이 맞추기 위해서입니다.

### ② 기준 측정: 훅 없이

1. Claude Code에서 `/clear`로 새 대화를 시작합니다
2. `/context`를 실행해 **Messages** 항목의 토큰 수를 적습니다 (시작값)
3. 아래 지시를 그대로 보냅니다

   ```text
   다음 명령을 그대로 한 번만 실행하고, 실패한 테스트 이름과 원인을 한 줄로 알려줘.
   다른 명령이나 옵션은 추가하지 마.
   cd backend && ./gradlew test
   ```

4. 답을 받은 뒤 다시 `/context`를 실행해 **Messages** 토큰 수를 적습니다 (끝값). 끝값 − 시작값이 이번 실행에서 늘어난 양입니다
5. Claude가 원인(어느 테스트, 기대값과 실제값)을 맞혔는지 적습니다

### ③ 훅 설치 (내 PC에만)

**스크립트 만들기**: 홈 폴더의 `.claude/hooks/`에 저장합니다 (macOS·Linux: `~/.claude/hooks/`, Windows: `C:\Users\<사용자 이름>\.claude\hooks\`). 줄바꿈은 **LF**로 저장합니다.

```bash
mkdir -p ~/.claude/hooks
# 아래 내용을 ~/.claude/hooks/filter-test-output.sh 로 저장 (VS Code 등)
```

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/filter-test-output.sh — PreToolUse(matcher: Bash), 내 개인 설정에만 등록
# 테스트 명령의 전체 출력은 로그 파일에 저장하고, Claude에게는 실패 부분과 요약만 보여 준다
input=$(cat)
command -v jq >/dev/null 2>&1 || exit 0          # jq가 없으면 아무것도 바꾸지 않는다 (명령은 원래대로 실행)
cmd=$(printf '%s' "$input" | jq -r '.tool_input.command // empty')

# 대상: ./gradlew … test …, cd backend && ./gradlew test, npm test, npm --prefix frontend test
# 파이프(|)·세미콜론(;)·리다이렉트(>)가 섞인 명령은 건드리지 않는다
w="[A-Za-z0-9_.:=*'\"/-]+"
re="^(cd $w && )?(\./gradlew( $w)* test|npm( --prefix $w)? (run )?test)( $w)*$"
[[ "$cmd" =~ $re ]] || exit 0

log="${TMPDIR:-/tmp}/claude-test-output.log"
new="{ $cmd ; } > \"$log\" 2>&1; rc=\$?
grep -E -A 6 'FAILED|FAIL |Exception|Error:|tests completed|Tests: |BUILD (SUCCESSFUL|FAILED)' \"$log\" | head -80
echo \"--- exit=\$rc · 전체 로그: $log (\$(wc -l < \"$log\")줄)\"
exit \$rc"

printf '%s' "$input" | jq --arg c "$new" '{
  hookSpecificOutput: {
    hookEventName: "PreToolUse",
    permissionDecision: "allow",
    permissionDecisionReason: "테스트 출력 줄이기 (개인 훅)",
    updatedInput: (.tool_input + {command: $c})
  }
}'
```

| 부분 | 하는 일 |
|---|---|
| `re=…` (정규식) | `./gradlew … test …`, `cd backend && ./gradlew test`, `npm test`, `npm --prefix frontend test`만 바꾼다. `\|`, `;`, `>`가 섞인 명령과 `./gradlew build` 같은 다른 명령은 **원래대로** 실행된다 |
| `{ 원래 명령 ; } > "$log" 2>&1` | 원래 명령을 그대로 실행하되, 화면 대신 **로그 파일**에 모두 저장한다 |
| `grep -E -A 6 '…' \| head -80` | 실패 줄(`FAILED`, `Exception` 등)과 그 아래 6줄, 요약(`tests completed`, `BUILD …`)만 최대 80줄 보여 준다 |
| `echo "--- exit=… · 전체 로그: …"`, `exit $rc` | **원래 종료 코드를 그대로 돌려준다.** 실패한 테스트가 "성공"으로 보이지 않게 하고, 더 필요하면 Claude가 로그 파일을 읽을 수 있게 위치를 알려 준다 |
| `permissionDecision: "allow"` | 바뀐 명령을 **묻지 않고 실행**한다. 정규식에 맞는 테스트 명령에만 적용된다 |
| `updatedInput` | Claude가 요청한 명령 대신 실행할 명령. 나머지 입력(설명 등)은 그대로 둔다 |

**스크립트만 먼저 시험하기** (Claude 없이):

```bash
# macOS·Linux·Git Bash
echo '{"tool_input":{"command":"cd backend && ./gradlew test"}}' | bash ~/.claude/hooks/filter-test-output.sh | jq -r '.hookSpecificOutput.updatedInput.command'
#  → { cd backend && ./gradlew test ; } > "/tmp/claude-test-output.log" 2>&1; rc=$? … 처럼 바뀐 명령이 보이면 성공
echo '{"tool_input":{"command":"git status"}}' | bash ~/.claude/hooks/filter-test-output.sh; echo "exit=$?"
#  → 아무것도 출력되지 않고 exit=0 (테스트 명령이 아니면 건드리지 않음)
```

```powershell
# Windows PowerShell (Git Bash로 실행, ~는 bash가 홈 폴더로 바꾼다)
$bash = "C:\Program Files\Git\bin\bash.exe"
'{"tool_input":{"command":"cd backend && ./gradlew test"}}' | & $bash -c 'bash ~/.claude/hooks/filter-test-output.sh'
'{"tool_input":{"command":"git status"}}' | & $bash -c 'bash ~/.claude/hooks/filter-test-output.sh'; "exit=$LASTEXITCODE"   # 출력 없음, exit=0
```

**내 설정에 등록하기**: `~/.claude/settings.json` (Windows: `C:\Users\<사용자 이름>\.claude\settings.json`). 파일이 없으면 새로 만들고, 이미 있으면 `hooks` 부분만 **합칩니다** (다른 설정은 지우지 않음).

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "bash ~/.claude/hooks/filter-test-output.sh" }
        ]
      }
    ]
  }
}
```

- 저장한 뒤 Claude Code에서 `/hooks`를 열어 `PreToolUse`에 이 훅이 **User settings** 출처로 보이는지 확인합니다. 보이지 않으면 Claude Code를 다시 시작합니다
- 팀 설정(`.claude/settings.json`)의 `guard-bash.sh`와 함께 실행됩니다. 두 훅은 서로 영향을 주지 않습니다 (위험 명령 검사는 원래 명령으로 합니다)
- Windows는 훅이 Git Bash로 실행되며 `jq`가 필요합니다 (Step 1). jq가 없으면 이 훅은 아무것도 바꾸지 않고 넘어갑니다

### ④ 훅을 켜고 같은 측정

②와 똑같이 합니다: `/clear` → `/context` (시작값) → 같은 지시 → `/context` (끝값). 이번에는 Claude가 받는 결과가 아래 **모양**이어야 합니다.

```text
EnrollmentServiceTest > 정원이_차면_대기자로_등록된다() FAILED
    org.opentest4j.AssertionFailedError: expected: <WAITING> but was: <CONFIRMED>
        at ...
12 tests completed, 1 failed
BUILD FAILED in 9s
--- exit=1 · 전체 로그: /tmp/claude-test-output.log (256줄)
```

### ⑤ 결과 기록 (회고에 붙이기)

| 비교 | 훅 없이 | 훅 사용 |
|---|---|---|
| `/context` Messages 시작값 → 끝값 | | |
| 이번 실행으로 늘어난 토큰 (끝값 − 시작값) | | |
| Claude가 받은 출력 줄 수 (대략) | | |
| Claude가 실패한 테스트와 원인을 맞혔나? | | |
| Claude가 실패를 "실패"로 인식했나? (exit 코드) | | |

> 💡 **효과가 작게 나올 수 있습니다.** Gradle의 기본 출력은 원래 짧은 편이라, 실패 테스트가 하나면 차이가 수백 토큰에 그칠 수 있습니다. 차이는 실패가 여러 개일 때, `--info`처럼 로그가 긴 옵션을 쓸 때, 프론트 테스트(`npm test`)에서 크게 납니다. **숫자로 확인하고 판단하는 것**이 이 실습의 목적입니다 (7회차 토큰 측정과 같은 방식).

### ⑥ 정리

```bash
git restore backend/src/test       # 일부러 깬 테스트 되돌리기
git switch main
git branch -D experiment/test-output-hook
```

- 훅을 계속 쓸지 정합니다. 그만 쓰려면 `~/.claude/settings.json`에서 위 `PreToolUse` 항목을 지웁니다
- 팀에 제안하려면 ⑤의 표를 근거로 PR을 만들고, **자동 승인(`allow`)을 팀 정책으로 받아들일지**를 리뷰에서 함께 정합니다

| 주의 | 설명 |
|---|---|
| 자동 승인 | 정규식에 맞는 테스트 명령은 **묻지 않고 실행**된다. 팀 권한 정책에서 테스트 명령을 `ask`로 두었다면 이 훅이 그 질문을 건너뛴다 |
| 원인을 놓칠 때 | 필터에 걸리지 않는 형식의 오류(예: 컴파일 오류 메시지)는 안 보일 수 있다. 출력 끝의 로그 위치를 보고 Claude가 `/tmp/claude-test-output.log`를 읽게 하거나, `grep` 패턴에 `error:`를 더한다 |
| 로그 파일 | 실행할 때마다 덮어쓴다. Git Bash의 `/tmp`는 Windows 사용자 임시 폴더다 |
| 종료 코드 | 처음 나온 예제(`… 2>&1 \| grep … \| head -100`)는 파이프 마지막 명령(`head`)의 종료 코드 0이 남아 **실패한 테스트도 성공처럼** 보였다. 이 버전은 원래 종료 코드를 돌려준다 |

> ✅ 이 교재 환경에서 가짜 `gradlew`(200줄 출력, 실패·성공 두 경우)로 확인했습니다: 실패 시 실패 줄과 요약만 나오고 `exit=1`, 성공 시 `BUILD SUCCESSFUL` 한 줄과 `exit=0`, `--tests` 옵션도 동작, 파이프·세미콜론·리다이렉트가 섞인 명령과 `./gradlew build`, `git status`는 바뀌지 않음. Claude Code에 등록해 실제 세션에서 실행해 보지는 못했으니 ③의 `/hooks` 확인과 ④의 출력 모양을 꼭 확인하세요.

---

## 7-3. SessionStart 훅으로 작업 맥락 넣기

세션을 시작할 때 현재 브랜치와 최근 커밋을 Claude에게 알려 줍니다.

```bash
#!/usr/bin/env bash
# .claude/hooks/session-context.sh — SessionStart: stdout 내용이 컨텍스트로 들어간다
cd "$CLAUDE_PROJECT_DIR" || exit 0
echo "현재 브랜치: $(git branch --show-current)"
echo "최근 커밋:"; git log --oneline -5
exit 0
```

> 💡 SessionStart 훅의 출력은 매 세션 컨텍스트에 들어가므로 **짧게** 유지합니다 (공식 문서의 SessionStart 출력 처리 방식 확인 필요).

---

## 7-4. 나만의 MCP 서버 만들기 (MariaDB 조회 도구)

범용 SQL 서버 대신, **정해진 질문만 할 수 있는 전용 도구**를 직접 만들어 등록해 봅니다. SQL은 코드에 고정하고 Claude는 값(`sessionId` 등)만 보내므로 더 안전하고 결과도 작습니다. 이 절에서는 **MCP 서버가 무엇이고 어떻게 동작하며 어떤 순서로 만드는지**를 먼저 이해하고, 가장 작은 서버를 직접 실행해 봅니다. DB에 연결하는 실제 서버의 전체 코드는 마지막의 링크를 따라 합니다.

### ① MCP 서버란

**MCP 서버 = AI 에이전트에게 "도구"를 제공하는 별도의 프로그램**입니다. Claude Code는 MCP **클라이언트**로서 서버를 실행(또는 접속)하고, 서버가 가진 도구 목록을 받아 Claude(모델)에게 보여 줍니다. Claude가 도구를 쓰겠다고 하면 Claude Code가 서버에 실행을 요청하고, 결과를 다시 Claude에게 전달합니다.

```mermaid
flowchart LR
    U["사용자<br/>1번 회차 남은 자리는?"] --> M["Claude (모델)<br/>도구를 고르고<br/>인자를 정한다"]
    M -->|"get_remaining_seats<br/>{sessionId: 1}"| C["Claude Code<br/>(MCP 클라이언트)<br/>권한 확인 · 메시지 전달"]
    C -->|"JSON-RPC 요청<br/>tools/call"| S["내 MCP 서버<br/>(별도 프로세스)"]
    S -->|"고정 SQL 실행"| D[("개발 DB<br/>읽기 전용 계정")]
    D --> S
    S -->|"JSON-RPC 응답<br/>content"| C
    C --> M
    M --> U
```

| 개념 | 설명 |
|---|---|
| **호스트·클라이언트** | Claude Code(또는 Claude 데스크톱, 사내 에이전트)가 클라이언트다. 서버마다 연결 하나를 맺는다 |
| **서버** | 기능을 제공하는 프로그램. 어떤 언어로 만들어도 된다 (규약만 지키면 된다) |
| **서버가 제공하는 3가지** | **도구(Tools)**: 모델이 골라 실행하는 함수 · **리소스(Resources)**: 읽을 수 있는 데이터(`@서버:주소`) · **프롬프트(Prompts)**: 슬래시 명령이 되는 템플릿. 이 실습은 **도구만** 만든다 |
| **도구 하나** | 이름 + 설명 + 입력 스키마 + (힌트) + 실행 함수. **설명은 모델이 도구를 고르는 근거**다 |
| **실행 위치** | stdio 서버는 **내 PC에서 내 권한으로** 실행된다. 그래서 DB 접속은 읽기 전용 계정으로 한다 |

### ② 관련 표준

| 표준 | 무엇을 정하나 | 이 실습에서 |
|---|---|---|
| **MCP 사양** (modelcontextprotocol.io) | 메시지 종류(`initialize`, `tools/list`, `tools/call` …), 순서, 형식. **날짜로 버전**을 붙인다 (예: `2025-06-18`, `2025-11-25`) | 연결할 때 클라이언트와 서버가 버전을 맞춘다 (③) |
| **JSON-RPC 2.0** | 메시지의 겉모양. **요청**(`id`, `method`, `params`) → **응답**(같은 `id`, `result` 또는 `error`), 응답이 없는 **알림**(`id` 없음) | 모든 MCP 메시지가 이 형식이다 |
| **JSON Schema** | 도구 입력의 형식(타입, 필수, 범위) | `inputSchema`. SDK가 실행 전에 입력을 검증한다 |
| **전송: stdio** | 클라이언트가 서버를 **자식 프로세스로 실행**하고 stdin/stdout으로 **한 줄에 메시지 하나**씩 주고받는다 | ⭐ 이 실습. `.mcp.json`의 `command`, `args` |
| **전송: Streamable HTTP** | 서버가 URL로 떠 있고 HTTP POST로 주고받는다 (필요하면 스트리밍) | 팀이 함께 쓰는 원격 서버 (2-5 제조 사례의 중앙 서버) |
| **인가: OAuth 2.1** | HTTP 서버에서 사용자를 인증·인가하는 방법 | 원격 서버를 사내에 배포할 때 |

> 💡 사양을 직접 구현할 필요는 없습니다. 공식 **SDK**(TypeScript `@modelcontextprotocol/sdk`, Java `io.modelcontextprotocol.sdk:mcp`, Python 등)가 JSON-RPC, 버전 협상, 입력 검증을 처리하고, 우리는 **도구 정의와 실행 함수**만 씁니다.

### ③ 동작 순서: 연결부터 도구 호출까지

```mermaid
sequenceDiagram
    participant CC as Claude Code (클라이언트)
    participant S as 내 MCP 서버
    Note over CC,S: 세션 시작 (.mcp.json을 읽고 서버 실행)
    CC->>S: initialize (내 버전·기능·이름)
    S-->>CC: 결과: 맞춘 버전, 서버 기능(tools), serverInfo
    CC->>S: notifications/initialized (알림, 응답 없음)
    CC->>S: tools/list
    S-->>CC: 도구 목록 (이름, 설명, inputSchema, annotations)
    Note over CC: 사용자가 질문 → Claude가 도구 선택
    CC->>S: tools/call {name, arguments}
    S-->>CC: 결과 {content: [...], isError}
```

**실제로 오가는 메시지** (아래 ⑤의 서버에 직접 보내서 얻은 결과. 읽기 쉽게 줄였습니다)

```text
→ {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"manual-test","version":"0.1"}}}
← {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2025-06-18","capabilities":{"tools":{"listChanged":true}},"serverInfo":{"name":"eduhub-hello","version":"1.0.0"}}}

→ {"jsonrpc":"2.0","method":"notifications/initialized"}

→ {"jsonrpc":"2.0","id":2,"method":"tools/list"}
← {"jsonrpc":"2.0","id":2,"result":{"tools":[{"name":"get_remaining_seats",
     "description":"교육 회차의 남은 자리 수를 조회한다. 회차 ID를 알 때 사용. 읽기 전용.",
     "inputSchema":{"type":"object","properties":{"sessionId":{"type":"integer", … ,"description":"회차 ID"}},"required":["sessionId"]},
     "annotations":{"readOnlyHint":true}}]}}

→ {"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_remaining_seats","arguments":{"sessionId":1}}}
← {"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"{\"sessionId\":1,\"title\":\"Spring Boot 입문\",\"remaining\":0}"}]}}

→ {"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"get_remaining_seats","arguments":{"sessionId":99}}}
← {"jsonrpc":"2.0","id":4,"result":{"content":[{"type":"text","text":"회차 99이(가) 없습니다."}],"isError":true}}

→ {"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"get_remaining_seats","arguments":{"sessionId":-1}}}
← {"jsonrpc":"2.0","id":5,"result":{"content":[{"type":"text","text":"MCP error -32602: Input validation error: … expected number to be >0 at sessionId"}],"isError":true}}
```

| 관찰 | 의미 |
|---|---|
| 응답의 `id`가 요청의 `id`와 같다 | JSON-RPC는 `id`로 요청과 응답의 짝을 맞춘다. 실제로 응답 순서가 요청 순서와 다르게 오기도 했다 |
| `protocolVersion` | 클라이언트가 보낸 버전을 서버가 지원하면 그대로, 아니면 서버가 지원하는 최신 버전으로 답한다 (모르는 버전 `2099-01-01`을 보내면 `2025-11-25`로 답함, TypeScript SDK 1.32) |
| `tools/list`의 `inputSchema` | 코드에 쓴 zod 스키마(또는 Java의 JSON 문자열)가 **JSON Schema**로 바뀌어 전달된다. Claude는 이것을 보고 인자를 만든다 |
| 결과 `content` | 도구 결과는 **텍스트 블록 목록**. JSON을 문자열로 넣어 보내는 것이 보통이다 |
| `isError: true` | "회차 없음" 같은 **업무 오류**는 예외가 아니라 결과로 돌려준다. Claude가 읽고 사용자에게 설명할 수 있다 |
| `sessionId: -1` | 입력 검증은 **SDK가 실행 함수 전에** 한다. 내 코드는 올바른 입력만 받는다 |

### ④ 만드는 절차

| 단계 | 할 일 | 결과물 | 상세 (TypeScript / Java) |
|---|---|---|---|
| 1. 도구 설계 | 어떤 질문에 답할지 정하고 **도구 이름·입력·출력**을 정한다. SQL은 코드에 고정, 결과는 필요한 열·최대 건수만 | 도구 목록 표 | [32번 1장](../examples/32-custom-mcp-server-mariadb.md#1-왜-직접-만드나-범용-sql-서버-vs-전용-도구) |
| 2. 프로젝트 준비 | SDK와 DB 드라이버 설치, 접속 정보는 **환경 변수**로 | `package.json` / `build.gradle.kts` | [32번 3장](../examples/32-custom-mcp-server-mariadb.md#3-준비) · [33번](../examples/33-custom-mcp-server-mariadb-java.md) |
| 3. 서버 만들기 + 도구 등록 | 서버 이름·버전, 도구마다 이름·**설명**·입력 스키마·`readOnlyHint`·실행 함수 | `index.ts` / `*.java` | [32번 4장](../examples/32-custom-mcp-server-mariadb.md#4-도구-만들기) |
| 4. 전송 연결 | stdio로 시작. **stdout에는 아무것도 출력하지 않는다** (로그는 stderr) | 실행 가능한 서버 | ⑤ 코드의 ③ |
| 5. Claude 없이 테스트 | 원시 메시지 보내기(⑤), MCP Inspector, 스모크 테스트 스크립트로 정상·오류 입력 확인 | 테스트 결과 | [32번 5장](../examples/32-custom-mcp-server-mariadb.md#5-빌드하고-claude-없이-먼저-테스트) |
| 6. Claude Code에 등록 | 저장소 루트 `.mcp.json`에 실행 명령, 권한에 `mcp__서버__도구` | `.mcp.json`, `settings.json` | [32번 6장](../examples/32-custom-mcp-server-mariadb.md#6-내-mcp-서버를-claude-code에-등록하기) |
| 7. 사용하며 다듬기 | Claude가 엉뚱한 도구를 고르면 **설명**을, 결과가 크면 **출력**을 고친다 | 개선 PR | [32번 7~8장](../examples/32-custom-mcp-server-mariadb.md#8-설계보안-체크리스트) |

### ⑤ 핵심 예시 코드: 가장 작은 MCP 서버

DB 없이 메모리 데이터로 답하는 도구 하나짜리 서버입니다. 모든 MCP 서버는 **① 서버 만들기 → ② 도구 등록 → ③ 전송 연결** 세 부분으로 되어 있고, 실제 DB 서버는 ②의 실행 함수 안에서 DB를 조회하는 것만 다릅니다.

**TypeScript** (`@modelcontextprotocol/sdk` 1.32, `zod` 4)

```js
#!/usr/bin/env node
// 가장 작은 MCP 서버: 도구 하나, DB 없이 메모리 데이터
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// ① 서버 만들기: 이름과 버전 (initialize 응답의 serverInfo가 된다)
const server = new McpServer({ name: "eduhub-hello", version: "1.0.0" });

// 예제 데이터 (실제로는 DB·API를 조회하는 자리)
const sessions = { 1: { title: "Spring Boot 입문", capacity: 2, confirmed: 2 } };

// ② 도구 등록: 이름 + 설명 + 입력 스키마 + 실행 함수
server.registerTool(
  "get_remaining_seats",
  {
    description: "교육 회차의 남은 자리 수를 조회한다. 회차 ID를 알 때 사용. 읽기 전용.",
    inputSchema: { sessionId: z.number().int().positive().describe("회차 ID") },
    annotations: { readOnlyHint: true },
  },
  async ({ sessionId }) => {
    const s = sessions[sessionId];
    if (!s) {
      return { isError: true, content: [{ type: "text", text: `회차 ${sessionId}이(가) 없습니다.` }] };
    }
    const result = { sessionId, title: s.title, remaining: s.capacity - s.confirmed };
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  },
);

// ③ 전송 방식 연결: stdio (stdout = MCP 메시지 전용, 로그는 stderr로)
await server.connect(new StdioServerTransport());
console.error("eduhub-hello 시작");
```

```bash
# 실행 준비 (빈 폴더에서)
npm init -y
npm pkg set type=module
npm install @modelcontextprotocol/sdk zod
# 위 코드를 hello-server.mjs로 저장
```

**Java** (`io.modelcontextprotocol.sdk:mcp` 2.1.0, JDK 21)

```java
import io.modelcontextprotocol.json.McpJsonDefaults;
import io.modelcontextprotocol.json.McpJsonMapper;
import io.modelcontextprotocol.server.McpServer;
import io.modelcontextprotocol.server.McpServerFeatures.SyncToolSpecification;
import io.modelcontextprotocol.server.transport.StdioServerTransportProvider;
import io.modelcontextprotocol.spec.McpSchema.CallToolResult;
import io.modelcontextprotocol.spec.McpSchema.ServerCapabilities;
import io.modelcontextprotocol.spec.McpSchema.Tool;
import io.modelcontextprotocol.spec.McpSchema.ToolAnnotations;
import java.util.Map;

/** 가장 작은 MCP 서버 (Java): 도구 하나, DB 없이 메모리 데이터 */
public class HelloServer {
    record Session(String title, int capacity, int confirmed) {}
    static final Map<Long, Session> SESSIONS = Map.of(1L, new Session("Spring Boot 입문", 2, 2));

    public static void main(String[] args) {
        McpJsonMapper json = McpJsonDefaults.getMapper();

        // ② 도구 정의: 이름 + 입력 스키마(JSON Schema) + 설명 + 힌트
        Tool tool = Tool.builder("get_remaining_seats", json, """
                {"type":"object",
                 "properties":{"sessionId":{"type":"integer","minimum":1,"description":"회차 ID"}},
                 "required":["sessionId"]}
                """)
            .description("교육 회차의 남은 자리 수를 조회한다. 회차 ID를 알 때 사용. 읽기 전용.")
            .annotations(ToolAnnotations.builder().readOnlyHint(true).build())
            .build();

        // ② 실행 함수: 인자를 받아 결과(content)를 돌려준다
        SyncToolSpecification spec = SyncToolSpecification.builder()
            .tool(tool)
            .callHandler((exchange, request) -> {
                long id = ((Number) request.arguments().get("sessionId")).longValue();
                Session s = SESSIONS.get(id);
                if (s == null) {
                    return CallToolResult.builder().addTextContent("회차 " + id + "이(가) 없습니다.").isError(true).build();
                }
                String text = "{\"sessionId\":" + id + ",\"title\":\"" + s.title() + "\",\"remaining\":" + (s.capacity() - s.confirmed()) + "}";
                return CallToolResult.builder().addTextContent(text).build();
            })
            .build();

        // ①③ 서버 만들기 + stdio 전송 연결 (System.out 금지, 로그는 System.err)
        McpServer.sync(new StdioServerTransportProvider(json))
            .serverInfo("eduhub-hello", "1.0.0")
            .capabilities(ServerCapabilities.builder().tools(true).build())
            .tools(spec)
            .build();
        System.err.println("eduhub-hello 시작");
    }
}
```

> Java는 [33번 예제](../examples/33-custom-mcp-server-mariadb-java.md)의 Gradle 프로젝트에서 `gradle jar`로 만든 jar에 SDK가 함께 들어 있으므로 `java -cp build/libs/eduhub-mcp.jar HelloServer.java`처럼 소스 파일 하나를 바로 실행할 수 있습니다.

| 부분 | TypeScript | Java | 핵심 |
|---|---|---|---|
| ① 서버 만들기 | `new McpServer({ name, version })` | `McpServer.sync(…).serverInfo(name, version)` | `initialize` 응답의 `serverInfo`가 된다 |
| ② 도구 등록 | `server.registerTool(이름, { description, inputSchema, annotations }, 함수)` | `Tool.builder(이름, json, 스키마)` + `SyncToolSpecification.builder().callHandler(…)` | `tools/list`에 나가는 정보 + `tools/call`이 오면 실행할 함수 |
| 입력 스키마 | zod (`z.number().int().positive()`) | JSON Schema 문자열 | SDK가 JSON Schema로 내보내고, 들어온 값을 검증한다 |
| 결과 | `{ content: [{ type: "text", text }] }`, 오류는 `isError: true` | `CallToolResult.builder().addTextContent(…).isError(…)` | 업무 오류도 결과로 돌려준다 |
| ③ 전송 연결 | `server.connect(new StdioServerTransport())` | `new StdioServerTransportProvider(json)` | stdout은 MCP 메시지 전용. 로그는 `console.error` / `System.err` |

**직접 확인: 서버에 원시 메시지 보내 보기** (Claude 없이 ③의 메시지를 그대로 재현)

```bash
# msgs.jsonl: 한 줄에 메시지 하나 (③에서 → 로 표시한 요청들)
cat > msgs.jsonl <<'EOF'
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"manual-test","version":"0.1"}}}
{"jsonrpc":"2.0","method":"notifications/initialized"}
{"jsonrpc":"2.0","id":2,"method":"tools/list"}
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_remaining_seats","arguments":{"sessionId":1}}}
{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"get_remaining_seats","arguments":{"sessionId":99}}}
{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"get_remaining_seats","arguments":{"sessionId":-1}}}
EOF

node hello-server.mjs < msgs.jsonl                                             # TypeScript
(cat msgs.jsonl; sleep 3) | java -cp build/libs/eduhub-mcp.jar HelloServer.java   # Java
```

```powershell
# Windows PowerShell (msgs.jsonl은 같은 내용으로 VS Code 등에서 만든다)
Get-Content msgs.jsonl | node hello-server.mjs
```

- 응답 5개가 ③의 ← 줄과 같은 모양이면 성공입니다. `eduhub-hello 시작`은 stderr로 나온 로그입니다
- Java는 입력이 끝나면 남은 응답을 보내기 전에 종료될 수 있어 `sleep 3`으로 잠깐 기다립니다. PowerShell에서 Java 서버를 시험할 때는 33번 예제의 스모크 테스트(SDK 클라이언트)를 씁니다
- Claude Code에 붙여 보려면 저장소 루트 `.mcp.json`에 `"eduhub-hello": { "command": "node", "args": ["tools/hello/hello-server.mjs"] }`처럼 등록하고(경로는 저장소 루트 기준), `/mcp`에서 연결을 확인한 뒤 "1번 회차 남은 자리 알려줘"라고 물어봅니다

> ✅ 위 TypeScript·Java 코드와 메시지는 이 교재 환경에서 실제로 실행해 얻은 결과입니다 (TypeScript SDK 1.32.2 + zod 4.6.7 / Node.js 22, Java SDK 2.1.0 / JDK 21). 빈 폴더에 위 "실행 준비" 명령으로 설치한 뒤 실행하는 것까지 확인했습니다. Claude Code 화면에서 이 서버를 등록해 쓰는 과정은 직접 확인하지 못했습니다.

### ⑥ 좋은 도구를 만드는 원칙과 흔한 실수

| 원칙 | 왜 | 흔한 실수 |
|---|---|---|
| **업무 단위**로 만든다 ("회차 현황", "사원별 신청 내역") | Claude가 테이블 구조를 몰라도 된다 | `run_sql(query)` 같은 범용 도구를 또 만든다 |
| **설명에 무엇을, 언제** 쓰는지 쓴다 | Claude가 도구를 고르는 유일한 근거 | "회차 조회" 한 마디만 쓴다 → 다른 도구와 헷갈린다 |
| **SQL은 코드에 고정**, 값만 받아 바인딩 | SQL 인젝션·무거운 쿼리 차단 | 입력 문자열을 SQL에 이어 붙인다 |
| **결과는 작게** (필요한 열, `LIMIT`, 요약값) | 결과는 컨텍스트에 들어가 토큰을 쓴다 | 테이블 전체를 돌려준다 |
| **읽기 전용**: DB 계정 + 읽기 전용 세션 + `readOnlyHint` | 힌트는 표시일 뿐, 강제는 계정 권한 | 쓰기 계정으로 연결하고 힌트만 단다 |
| **업무 오류는 `isError` 결과**로 | Claude가 사용자에게 이유를 설명할 수 있다 | 예외를 던져 서버가 죽는다 |
| **stdout 금지** (stdio 서버) | stdout은 MCP 메시지 통로 | `console.log`, `System.out.println`, 콘솔 로거가 메시지를 깨뜨려 연결이 끊긴다 |

### ⑦ 이제 실제 DB 서버로

⑤의 실행 함수 안에서 메모리 데이터 대신 **개발 DB(MariaDB)를 읽기 전용 계정으로 조회**하면 실제 서버가 됩니다. 도구 두 개(`get_session_status`, `find_enrollments_by_employee`), DB 연결, 테스트, `.mcp.json` 등록, 권한 설정까지의 전체 과정은 다음 예제를 따라 합니다.

- [나만의 MCP 서버 만들기 (TypeScript)](../examples/32-custom-mcp-server-mariadb.md)
- [나만의 MCP 서버 만들기 (Java, MCP Java SDK)](../examples/33-custom-mcp-server-mariadb-java.md)
- 같은 방식을 제조 현장에 적용한 예: [제조 현장 AI 에이전트와 MCP](../examples/34-manufacturing-ai-agent-mcp.md) (MES는 DB, ERP는 REST API로 가져오는 도구 5개)

---

# 8. 정리

---

## 8-1. 오늘의 핵심

1. **훅은 강제다**: 반드시 지켜야 하고 기계적으로 검사할 수 있는 규칙은 훅으로. CLAUDE.md(부탁) + 권한(경계) + 훅(강제)을 함께 쓴다
2. **이벤트 · matcher · 종료 코드**: `PreToolUse`는 막고, `PostToolUse`는 알리고, `Stop`은 계속하게 한다. 차단 메시지에는 **대안**을 쓴다
3. **훅은 내 권한으로 실행되는 코드**: 빠르게 만들고, PR로 리뷰하고, `.claude/`에 코드 오너를 둔다
4. **MCP는 개발 DB + 읽기 전용 계정으로만**: 운영 DB와 실제 개인정보는 AI 대화에 들이지 않는다. 데이터 변경은 마이그레이션과 코드로
5. **CLI가 있으면 CLI부터**: `gh`처럼 가벼운 도구를 먼저, 쓰지 않는 MCP 서버는 끄고, 결과는 작게 요청한다
6. **권한 확인은 백엔드가, 화면은 안내만**: 화면의 로그인·관리자 확인은 길 안내일 뿐이고 실제로 막는 것은 401·403이다. 토큰 처리는 `http` 한 곳에 모은다
7. ⭐ **자동화가 늘수록 이해가 더 중요하다**: 훅과 MCP가 무엇을 하는지 모르면 팀의 안전장치가 아니라 블랙박스가 된다

---

## 8-2. 확인 퀴즈

1. "Java 파일을 수정하면 항상 포맷을 맞춘다"를 CLAUDE.md가 아니라 훅으로 구현하는 이유는?
2. `PreToolUse` 훅과 `PostToolUse` 훅에서 종료 코드 2는 각각 어떤 결과를 낳는가?
3. 차단 훅의 메시지를 "금지입니다"만 쓰면 어떤 일이 생기나? 어떻게 써야 하나?
4. `guard-bash.sh` 훅이 있으니 `main` 브랜치 보호와 권한 deny는 필요 없을까?
5. `Stop` 훅에서 `stop_hook_active`를 확인하지 않으면 어떤 문제가 생길 수 있나?
6. DB MCP를 개발 DB의 읽기 전용 계정으로만 연결해야 하는 이유 두 가지는?
7. `.mcp.json`에 DB 비밀번호를 쓰면 안 되는 이유와, 대신 쓰는 방법은?
8. GitHub 작업에 GitHub MCP 대신 `gh` CLI를 고른 이유는?
9. 화면에서 관리자 메뉴를 숨겼는데도 백엔드에서 `/api/v1/courses`의 POST를 ADMIN으로 막아야 하는 이유는?
10. API가 401을 돌려주면 로그아웃시키는데, 403에는 로그아웃시키지 않는 이유는?

<details>
<summary>정답 보기</summary>

1. CLAUDE.md는 부탁이라 가끔 놓치지만, 훅은 하네스가 도구 실행 후 **매번 반드시** 실행하기 때문
2. `PreToolUse`: 도구 실행을 막고 이유를 Claude에게 전달. `PostToolUse`: 이미 실행되어 막지는 못하고 메시지를 Claude에게 전달해 고치게 한다
3. Claude가 다른 우회 방법을 찾으며 같은 시도를 반복할 수 있다. **이유와 대신 할 일**을 함께 쓴다 (예: "운영 설정은 PR로만. 개발은 application-dev.yml")
4. 필요하다. 셸 명령은 변형이 많아 문자열 검사로 다 막을 수 없다. 훅 + 권한 deny + 브랜치 보호(+ 운영 비밀을 개발 PC에 두지 않기)를 **겹쳐서** 쓴다
5. 테스트가 계속 실패하면 Stop 훅이 계속 작업을 이어가게 해 반복될 수 있다 (시간과 토큰 낭비)
6. 운영 데이터(개인정보)가 AI 대화로 들어가는 것을 막기 위해, 잘못된 판단이나 프롬프트 인젝션으로 데이터가 바뀌거나 지워지는 것을 막기 위해
7. `.mcp.json`은 Git에 커밋되어 이력에 영원히 남는다. `${EDUHUB_DB_READONLY_URL}`처럼 **환경 변수**를 쓰고, 값은 커밋하지 않는 `.env`나 셸 프로필에 둔다
8. 이미 설치·로그인되어 있고, 도구 정의를 추가하지 않으며, `--json`·`--limit` 등으로 출력을 줄일 수 있어 토큰을 덜 쓴다
9. 화면은 안내일 뿐이라 누구나 curl이나 개발자 도구로 API를 직접 부를 수 있다. 막는 것은 백엔드다 (Step 9-1, 6장의 ⚠️)
10. 401은 "누구인지 모름"(토큰 없음·만료·위조)이라 다시 로그인해야 하지만, 403은 "누구인지는 알지만 권한 없음"이라 로그인 상태를 유지하고 안내만 한다

</details>

---

## 8-3. 과제 (다음 회차 전까지)

- [ ] 훅 PR(`chore/quality-hooks`), 인증 PR, FR-07 PR, FR-08 PR을 리뷰 받고 **Squash merge**까지 완료
- [ ] ⭐ **6장 Step 15~16 완성**(수강생·관리자 화면)하고 6-2 확인 표 12개를 직접 확인한 뒤 PR 병합. **6회차는 이 앱에서 시작합니다**
- [ ] 7-1 OpenAPI 누락 검사 훅을 추가하고, 1회차 토론의 A+C+E가 모두 갖춰졌는지 팀에서 확인
- [ ] 팀원 한 명의 훅 스크립트를 **직접 읽고** 리뷰 코멘트 1개 이상 남기기
- [ ] 수료 처리 규칙(취소자, 시작일, 부분 수료)을 팀에서 결정해 `docs/requirements.md`에 반영
- [ ] **AI 없이 설명하기**: 오늘 만든 훅 4개가 각각 언제, 무엇을 막거나 실행하는지 팀원에게 설명
- [ ] 회고: `/usage` 결과, 훅이 막아 준 실수 1가지, MCP 결과를 작게 요청해 아낀 경험 1가지

## 8-4. 다음 회차 예고

**6회차 (6주차): 플러그인과 아티팩트 — 마이크로프론트엔드 완성과 알림 (모듈 14, 15)**
- 오늘 만든 훅·스킬·에이전트·MCP 설정을 **팀 플러그인**으로 묶어 배포하기
- 오늘 완성한 **하나의 React 앱을 shell·course-mfe·enrollment-mfe로 쪼개기** (Module Federation, 워크트리 병렬 세션), 알림(FR-09)
- 진행 현황 리포트를 **아티팩트**로 만들어 공유

**미리 읽어 오면 좋은 자료**
- [MCP·플러그인·아티팩트 예제](../examples/04-mcp-plugins-artifacts.md)
- [스킬 카탈로그](../examples/09-skills-catalog.md)
- [Java + React 프로젝트의 에이전트 모범 사례](../examples/18-java-react-agents.md)

---

## 참고 자료

- Claude Code 공식 문서: Hooks reference — https://code.claude.com/docs/en/hooks , MCP — https://code.claude.com/docs/en/mcp
- 교육 저장소: [1회차 자료](./session-01-introduction.md), [2회차 자료](./session-02-hands-on-kickoff.md), [설정·스킬·에이전트·훅 예제](../examples/03-settings-skills-agents-hooks.md), [토큰 최적화](../examples/08-token-optimization.md), [MCP와 훅 개념](../examples/12-mcp-and-hooks-concepts.md), [개념 비교와 도구 선택](../examples/13-choosing-the-right-tool.md), [MCP 서버 완전 가이드](../examples/19-mcp-server-guide.md), [권한](../examples/23-permissions.md), [개발자의 책임](../examples/27-developer-responsibility.md), [GitHub 연결과 버전관리](../examples/06-github-version-control.md)
