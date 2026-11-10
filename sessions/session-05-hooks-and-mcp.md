# 5회차: 훅과 MCP — 품질 자동화와 외부 연결 (FR-07~08)

🧭 ⬅️ 이전: [4회차: 권한·스킬·서브에이전트 — 수강 신청과 동시성 (FR-04~06, NFR-01)](./session-04-permissions-skills-subagents.md) · ➡️ 다음: [6회차: 플러그인·아티팩트 — 마이크로프론트엔드 완성과 알림(FR-09)](./session-06-plugins-artifacts-mfe.md) · 📚 [전체 커리큘럼](../CURRICULUM.md#-회차별-강의-자료)

> **대상**: 4회차까지 진행해 EduHub 수강 신청(FR-04~06)을 구현하고, 권한 설정·스킬·서브에이전트를 써 본 개발자
> **시간**: 120분 (강의 약 40% + 따라하기 약 60%)
> **목표**
> 1. **훅**이 CLAUDE.md·스킬과 어떻게 다른지("부탁"이 아니라 "강제") 설명하고, 이벤트·matcher·종료 코드로 동작을 설계할 수 있다
> 2. EduHub에 **품질 자동화 훅**(자동 포맷, 위험 명령·보호 파일 차단, 마무리 테스트)을 직접 만들어 적용한다
> 3. **MCP**를 프로젝트 범위(`.mcp.json`)로 연결하고, **개발 DB 읽기 전용 계정**으로만 붙이는 이유와 방법을 안다
> 4. MCP와 CLI(`gh`, `psql`)의 차이와 토큰 관점의 선택 기준을 설명할 수 있다
> 5. DB MCP로 스키마와 데이터를 확인하며 **FR-07(내 신청 내역)**, **FR-08(명단·수료 처리)**를 구현하고 PR로 병합한다
>
> **관련 모듈**: 12(훅), 13(커넥터와 MCP)
> **사전 준비**
> - 4회차 과제 PR 병합 완료, `main` 최신화
> - `jq` 설치 확인(`jq --version`). 윈도우는 Git Bash에서 실행되는지 확인 (`winget install jqlang.jq` 등)
> - Docker Desktop 실행, EduHub 개발 DB(PostgreSQL) 컨테이너가 뜨는지 확인
> - `uv` 설치 (`uvx --version`): DB MCP 서버(Postgres MCP Pro) 실행용. 설치 방법은 Step 8 ①
> - (선택) GitHub CLI `gh` 로그인 (`gh auth status`)
> 🖥️ **슬라이드 덱(34장, 발표자 노트 포함)**: https://claude.ai/artifact/SuXXVHg7iSzXiDetXP1LRV — 처음에는 만든 사람만 볼 수 있으며, 다른 사람에게 보여주려면 덱의 공유 메뉴에서 공유해야 합니다. PowerPoint(.pptx)·PDF로 내려받을 수 있습니다.

---

## 진행 순서

| 시간 | 내용 | 형식 |
|---|---|---|
| 0:00 – 0:05 | 지난 회차 복습 & 과제 리뷰 | 강의 |
| 0:05 – 0:25 | 1. 훅: 부탁이 아닌 강제 | 강의 + 시연 |
| 0:25 – 0:40 | 2. MCP: 외부 시스템과 연결하기 | 강의 |
| 0:40 – 0:45 | 3. 1회차 토론 다시 하기: "OpenAPI 문서 누락을 어떻게 막을까?" | 토론 |
| 0:45 – 1:15 | 4. 따라하기 ①: EduHub 품질 자동화 훅 만들기 | 실습 |
| 1:15 – 1:23 | ☕ 휴식 | — |
| 1:23 – 1:53 | 5. 따라하기 ②: DB MCP 연결과 FR-07~08 구현 | 실습 |
| 1:53 – 2:00 | 7. 정리, 퀴즈, 과제 안내 | 강의 |

> 6장(더 해 보기)은 시간이 남으면 수업 중에, 아니면 **과제**로 진행합니다.

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
$hookInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
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

```powershell
# Windows PowerShell 버전
'{"tool_input":{"file_path":"C:\\work\\eduhub\\backend\\src\\main\\resources\\application-prod.yml"}}' |
  powershell.exe -NoProfile -ExecutionPolicy Bypass -File .claude\hooks\protect-prod-config.ps1; "exit=$LASTEXITCODE"   # exit=2
```

> ✅ bash 버전은 위 세 테스트(Linux 경로 차단, Windows 경로 차단, 개발 설정 통과)를 실행해 결과를 확인했습니다.

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

# 3. 1회차 토론 다시 하기: "OpenAPI 문서 누락을 어떻게 막을까?"

1회차 [8-2 토론 과제](./session-01-introduction.md#8-2-토론-과제-7분)의 요구사항을 다시 봅니다: **"새 API를 추가할 때 OpenAPI 문서(`@Operation`)가 빠지지 않게 하고 싶다."**

| 방식 | 1회차 때의 질문 | 지금 우리가 아는 답 |
|---|---|---|
| A. CLAUDE.md | 항상 지켜질까? | 대부분 지키지만 **가끔 놓친다**. 규칙을 알려 주는 역할 |
| B. 슬래시 커맨드 | 사람이 잊으면? | 사람이 실행해야 동작 → 잊으면 끝 |
| C. 스킬 | 품질까지 좋아질까? | 4회차의 `add-api-endpoint` 스킬처럼 **잘 쓰는 법**(설명, 예시, 응답 코드)을 준다 |
| D. 서브에이전트 | 비용과 효과는? | 문서 품질 리뷰는 가능하지만 매번 돌리면 비싸다. PR 전 한 번이면 충분 |
| E. 훅 | "존재"와 "품질"의 차이는? | **`@Operation`이 있는지(존재)는 훅으로 강제**할 수 있다. 설명이 좋은지(품질)는 스킬·리뷰의 몫 |

> **결론: A + C + E.** CLAUDE.md로 알리고, 스킬로 잘 쓰는 법을 주고, 훅으로 빠지면 잡는다. 오늘 따라하기 ①의 "더 해 보기"([6-1](#6-1-openapi-누락-검사-훅))에서 E를 직접 만들어 봅니다.

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

**직접 테스트해 보기** (훅은 프로그램이므로 Claude 없이도 테스트할 수 있습니다)

```bash
echo '{"tool_input":{"file_path":"/x/backend/src/main/resources/application-prod.yml"}}' \
  | bash .claude/hooks/protect-files.sh; echo "exit=$?"        # exit=2
echo '{"tool_input":{"file_path":"/x/backend/src/main/java/A.java"}}' \
  | bash .claude/hooks/protect-files.sh; echo "exit=$?"        # exit=0
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

> ✅ **확인**: PR 본문에 "Claude에게 맡긴 것 / 직접 한 것"을 쓰고, 리뷰어는 **스크립트를 직접 읽고** 승인합니다. 훅은 팀원 모두의 PC에서 실행되기 때문입니다.

---

## 4-7. 자주 겪는 문제 (훅)

| 증상 | 원인과 해결 |
|---|---|
| 훅이 전혀 실행되지 않음 | `/hooks`에서 등록 여부 확인. JSON 문법 오류(`jq . .claude/settings.json`), 이벤트 이름 철자(`PostToolUse`) 확인 |
| `jq: command not found` | jq 설치. 윈도우는 Git Bash의 PATH에 jq가 있는지 확인 |
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

이 작업은 **사람이 직접** 합니다. DB 권한은 Claude에게 맡기지 않습니다. 개발 DB가 **MariaDB**라면 [MariaDB 버전](#개발-db가-mariadb인-팀-step-78의-mariadb-버전)을 따릅니다.

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

```bash
# .env (커밋 금지) 또는 셸 프로필에 추가
export EDUHUB_DB_READONLY_URL="postgresql://eduhub_readonly:각자-정한-비밀번호@localhost:5432/eduhub"
```

`.env.example`에는 **변수 이름만** 추가합니다.

```bash
EDUHUB_DB_READONLY_URL=
```

> ✅ **확인**: `psql "$EDUHUB_DB_READONLY_URL" -c "DELETE FROM enrollment"`가 **권한 오류**로 실패해야 합니다. 쓰기가 막혀 있어야 MCP를 연결합니다.

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

**② 프로젝트 루트에 `.mcp.json` 만들기** (터미널에서는 `claude mcp add --scope project …`로도 만들 수 있습니다)

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

> 💡 `${EDUHUB_DB_READONLY_URL}`은 **Claude Code를 실행한 셸의 환경 변수**에서 읽습니다. `.env` 파일에만 적어 두면 Claude Code가 읽지 못할 수 있으니, 셸 프로필(`~/.zshrc`, `~/.bashrc`, Windows는 사용자 환경 변수)에 넣거나 Claude Code를 시작하기 전에 `export`합니다.

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

```bash
# 개발 DB 컨테이너에 접속 (컨테이너 이름은 팀 설정에 맞게)
docker exec -it eduhub-mariadb mariadb -u root -p
```

```sql
-- 개발 DB에서만 실행. 비밀번호는 팀 공유 금지, 각자 정한다
CREATE USER 'eduhub_readonly'@'%' IDENTIFIED BY '각자-정한-비밀번호';
GRANT SELECT, SHOW VIEW ON eduhub.* TO 'eduhub_readonly'@'%';
SHOW GRANTS FOR 'eduhub_readonly'@'%';   -- SELECT, SHOW VIEW만 있는지 확인
```

> 💡 PostgreSQL처럼 계정 단위로 "기본 읽기 전용 트랜잭션"을 거는 설정 대신, MariaDB에서는 **SELECT 권한만 주는 것**이 핵심 방어선입니다. `eduhub.*`처럼 **필요한 스키마에만** 권한을 줍니다.

```bash
# 셸 프로필(~/.zshrc, ~/.bashrc) 또는 Windows 사용자 환경 변수에 추가 (.env만으로는 Claude Code가 못 읽을 수 있음)
export EDUHUB_DB_RO_USER="eduhub_readonly"
export EDUHUB_DB_RO_PASSWORD="각자-정한-비밀번호"
```

`.env.example`에는 **변수 이름만** 추가합니다: `EDUHUB_DB_RO_USER=`, `EDUHUB_DB_RO_PASSWORD=`

> ✅ **확인**: `mariadb -h 127.0.0.1 -u eduhub_readonly -p eduhub -e "DELETE FROM enrollment"`가 `DELETE command denied` 같은 **권한 오류**로 실패해야 합니다.

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

조회 도구는 `mysql_query` 하나입니다 (README 기준). 권한 허용은 다음과 같습니다.

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

## Step 10. FR-07 내 신청 내역 구현

권한 모드 **Plan**으로 지시합니다.

```text
[목표] FR-07: 로그인한 사원이 자기 신청 내역을 상태별로 조회한다.
[위치] backend/.../enrollment 패키지 (EnrollmentController, EnrollmentQueryService, EnrollmentRepository)
[범위]
- GET /api/me/enrollments?status=CONFIRMED (status 생략 시 전체), 페이지 단위
- 응답 DTO: 신청 ID, 과정 제목, 회차 시작일·장소, 상태, 대기 순번, 신청 일시
- 과정·회차는 fetch join 또는 DTO 프로젝션으로 한 번에 조회 (N+1 금지)
- 사원 ID는 요청 파라미터가 아니라 인증 정보에서 가져온다 (NFR-03)
- OpenAPI 어노테이션(@Operation, 응답 코드)
- Step 9 결과 인덱스가 없으면 Flyway 새 버전 파일로 추가
[하지 않는 것] 화면, FR-08
[완료 기준] ./gradlew test 통과. 쿼리 로그에서 목록 조회 쿼리가 1~2개인지 확인하는 테스트
[진행 방식] 먼저 계획과 쿼리 방식을 보여줘
```

**계획에서 볼 점**
- 엔티티를 그대로 반환하지 않고 DTO를 쓰는가? (2회차 [2-3 사례](./session-02-hands-on-kickoff.md#2-3-사례-테스트는-통과했지만-문제가-있는-코드)의 ①)
- 사원 ID를 인증 정보에서 가져오는가? (같은 사례의 ④)

**기대하는 핵심 코드 (예)**

```java
public record MyEnrollmentResponse(
        Long enrollmentId, String courseTitle, LocalDate startDate, String place,
        EnrollmentStatus status, Integer waitingNo, LocalDateTime appliedAt) {}
```

```java
public interface EnrollmentRepository extends JpaRepository<Enrollment, Long> {

    @Query("""
        select new com.example.eduhub.enrollment.dto.MyEnrollmentResponse(
            e.id, c.title, s.startDate, s.place, e.status, e.waitingNo, e.appliedAt)
        from Enrollment e join e.session s join s.course c
        where e.employeeId = :employeeId
          and (:status is null or e.status = :status)
        order by e.appliedAt desc
        """)
    Page<MyEnrollmentResponse> findMyEnrollments(Long employeeId, EnrollmentStatus status, Pageable pageable);
}
```

**구현 중 확인할 것 (오늘의 훅이 일하는 모습)**
- Java 파일을 수정할 때마다 포맷이 자동으로 맞춰지는가? (훅 ①)
- Claude가 기존 `V*.sql`을 고치려 하면 막히고 새 버전 파일을 만드는가? (훅 ②)
- 응답이 끝날 때 테스트가 돌고, 실패하면 Claude가 이어서 고치는가? (훅 ④)

**MCP로 결과 확인**

```text
eduhub-db로 EXPLAIN을 실행해서 방금 만든 내 신청 내역 쿼리가 인덱스를 쓰는지만 알려줘.
```

> ✅ **확인**: 내 터미널에서 `cd backend && ./gradlew test`를 **직접** 실행해 통과하는지 봅니다. Swagger UI(`/swagger-ui.html`)에서 새 API와 설명이 보이는지도 확인합니다.

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

```text
[목표] FR-08: 관리자가 회차별 신청자·대기자 명단을 보고 수료 처리한다.
[위치] backend/.../enrollment 패키지, 관리자 API는 /api/admin 아래
[범위]
- GET /api/admin/sessions/{sessionId}/roster : 확정자 목록과 대기자 목록(대기 순번 순)
- POST /api/admin/sessions/{sessionId}/completions : 사원 ID 목록을 받아 CONFIRMED → COMPLETED
  - CONFIRMED가 아닌 신청은 수료 처리하지 않고 결과에 실패 사유로 돌려준다
  - 회차 시작일 이전에는 수료 처리할 수 없다 (409)
- 관리자 권한이 없으면 403 (NFR-03)
- OpenAPI 어노테이션
[하지 않는 것] 화면, 알림(FR-09)
[완료 기준] 정상·권한 없음·상태 오류·시작일 이전 테스트 포함, ./gradlew test 통과
[진행 방식] 계획부터. 상태 전이 규칙이 애매하면 질문해줘
```

> 💡 수료 처리 같은 **업무 규칙**(무엇을 수료로 볼지, 취소된 사람은 어떻게 할지)은 Claude가 정하게 두지 말고 팀이 결정해 `docs/requirements.md`에 적습니다. Claude가 질문하면 **사람이 답하는 것**이 구조 통제입니다.

**MCP 활용 예**

```text
eduhub-db로 테스트용 회차 하나를 골라 CONFIRMED/WAITING 건수만 보여줘.
그 회차로 명단 API를 로컬에서 호출했을 때 기대 결과를 정리해줘.
```

> ✅ **확인**: 관리자 토큰과 일반 사원 토큰으로 각각 호출해 200과 403이 나오는지 직접 확인합니다 (팀의 인증 방식에 맞게). 커밋·푸시·PR은 Step 10과 같습니다 (`feat(enrollment): 회차 명단 조회와 수료 처리 API 추가 (FR-08)`).

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
| `uvx: command not found` | uv 미설치. Step 8의 ① 준비. 설치 후 터미널·Claude Code를 다시 시작 |
| `/mcp`에 서버가 failed로 보임 | `uvx --version`, `EDUHUB_DB_READONLY_URL`이 **Claude Code를 실행한 셸**에 있는지(`echo $EDUHUB_DB_READONLY_URL`), DB 컨테이너 실행 여부 확인. 첫 실행은 패키지를 받느라 오래 걸릴 수 있으니 `/mcp`에서 다시 연결 |
| `${EDUHUB_DB_READONLY_URL}`이 글자 그대로 남음 | 환경 변수가 없음. 셸 프로필에 넣고 터미널·앱을 다시 시작 |
| 승인 질문 없이 서버가 안 보임 | 이전에 거부했을 수 있음. `claude mcp reset-project-choices`로 선택 초기화 |
| 윈도우에서 `npx` 서버(Google MCP Toolbox 대안 등)가 실행되지 않음 | `cmd /c npx …` 형태로 감싸기 ([MCP 가이드 5-2](../examples/19-mcp-server-guide.md#5-2-운영체제별-주의)) |
| DB 연결 거부 | Docker DB 컨테이너가 실행 중인지, 포트(5432) 확인 |
| 결과가 너무 커서 경고 | 조회 범위를 좁힌다 ("건수만", "5건만") |
| Claude가 MCP로 데이터를 고치려 함 | 읽기 전용 계정이라 실패해야 정상. 실패하지 않았다면 Step 7 권한을 다시 확인 |

---

# 6. 더 해 보기 (시간이 남으면 수업 중, 아니면 과제)

---

## 6-1. OpenAPI 누락 검사 훅

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

## 6-2. 테스트 출력 줄이기 훅 (개인 설정)

긴 Gradle 테스트 출력이 컨텍스트를 채우는 것을 막습니다. 팀 설정이 아니라 **내 설정**(`~/.claude/settings.json`)에 넣어 실험합니다. 스크립트는 [토큰 최적화 예제](../examples/08-token-optimization.md)의 "훅으로 테스트 출력 줄이기"를 씁니다.

| 비교 | 훅 없이 | 훅 사용 |
|---|---|---|
| `./gradlew test` 실패 시 `/context`의 도구 결과 증가량 | | |
| Claude가 원인을 찾았나? | | |

> ⚠️ 그 예제 훅은 테스트 명령을 자동 승인(`allow`)으로 바꿉니다. 팀 권한 정책과 맞는지 확인하고, 실패 원인을 놓치지 않도록 필터 범위를 조정합니다.

---

## 6-3. SessionStart 훅으로 작업 맥락 넣기

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

# 7. 정리

---

## 7-1. 오늘의 핵심

1. **훅은 강제다**: 반드시 지켜야 하고 기계적으로 검사할 수 있는 규칙은 훅으로. CLAUDE.md(부탁) + 권한(경계) + 훅(강제)을 함께 쓴다
2. **이벤트 · matcher · 종료 코드**: `PreToolUse`는 막고, `PostToolUse`는 알리고, `Stop`은 계속하게 한다. 차단 메시지에는 **대안**을 쓴다
3. **훅은 내 권한으로 실행되는 코드**: 빠르게 만들고, PR로 리뷰하고, `.claude/`에 코드 오너를 둔다
4. **MCP는 개발 DB + 읽기 전용 계정으로만**: 운영 DB와 실제 개인정보는 AI 대화에 들이지 않는다. 데이터 변경은 마이그레이션과 코드로
5. **CLI가 있으면 CLI부터**: `gh`처럼 가벼운 도구를 먼저, 쓰지 않는 MCP 서버는 끄고, 결과는 작게 요청한다
6. ⭐ **자동화가 늘수록 이해가 더 중요하다**: 훅과 MCP가 무엇을 하는지 모르면 팀의 안전장치가 아니라 블랙박스가 된다

---

## 7-2. 확인 퀴즈

1. "Java 파일을 수정하면 항상 포맷을 맞춘다"를 CLAUDE.md가 아니라 훅으로 구현하는 이유는?
2. `PreToolUse` 훅과 `PostToolUse` 훅에서 종료 코드 2는 각각 어떤 결과를 낳는가?
3. 차단 훅의 메시지를 "금지입니다"만 쓰면 어떤 일이 생기나? 어떻게 써야 하나?
4. `guard-bash.sh` 훅이 있으니 `main` 브랜치 보호와 권한 deny는 필요 없을까?
5. `Stop` 훅에서 `stop_hook_active`를 확인하지 않으면 어떤 문제가 생길 수 있나?
6. DB MCP를 개발 DB의 읽기 전용 계정으로만 연결해야 하는 이유 두 가지는?
7. `.mcp.json`에 DB 비밀번호를 쓰면 안 되는 이유와, 대신 쓰는 방법은?
8. GitHub 작업에 GitHub MCP 대신 `gh` CLI를 고른 이유는?

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

</details>

---

## 7-3. 과제 (다음 회차 전까지)

- [ ] 훅 PR(`chore/quality-hooks`), FR-07 PR, FR-08 PR을 리뷰 받고 **Squash merge**까지 완료
- [ ] 6-1 OpenAPI 누락 검사 훅을 추가하고, 1회차 토론의 A+C+E가 모두 갖춰졌는지 팀에서 확인
- [ ] 팀원 한 명의 훅 스크립트를 **직접 읽고** 리뷰 코멘트 1개 이상 남기기
- [ ] 수료 처리 규칙(취소자, 시작일, 부분 수료)을 팀에서 결정해 `docs/requirements.md`에 반영
- [ ] **AI 없이 설명하기**: 오늘 만든 훅 4개가 각각 언제, 무엇을 막거나 실행하는지 팀원에게 설명
- [ ] 회고: `/usage` 결과, 훅이 막아 준 실수 1가지, MCP 결과를 작게 요청해 아낀 경험 1가지

## 7-4. 다음 회차 예고

**6회차 (6주차): 플러그인과 아티팩트 — 마이크로프론트엔드 완성과 알림 (모듈 14, 15)**
- 오늘 만든 훅·스킬·에이전트·MCP 설정을 **팀 플러그인**으로 묶어 배포하기
- course-mfe, enrollment-mfe 분리 완성과 FR-07~08 화면, 알림(FR-09)
- 진행 현황 리포트를 **아티팩트**로 만들어 공유

**미리 읽어 오면 좋은 자료**
- [MCP·플러그인·아티팩트 예제](../examples/04-mcp-plugins-artifacts.md)
- [스킬 카탈로그](../examples/09-skills-catalog.md)
- [Java + React 프로젝트의 에이전트 모범 사례](../examples/18-java-react-agents.md)

---

## 참고 자료

- Claude Code 공식 문서: Hooks reference — https://code.claude.com/docs/en/hooks , MCP — https://code.claude.com/docs/en/mcp
- 교육 저장소: [1회차 자료](./session-01-introduction.md), [2회차 자료](./session-02-hands-on-kickoff.md), [설정·스킬·에이전트·훅 예제](../examples/03-settings-skills-agents-hooks.md), [토큰 최적화](../examples/08-token-optimization.md), [MCP와 훅 개념](../examples/12-mcp-and-hooks-concepts.md), [개념 비교와 도구 선택](../examples/13-choosing-the-right-tool.md), [MCP 서버 완전 가이드](../examples/19-mcp-server-guide.md), [권한](../examples/23-permissions.md), [개발자의 책임](../examples/27-developer-responsibility.md), [GitHub 연결과 버전관리](../examples/06-github-version-control.md)
