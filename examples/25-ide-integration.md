# IDE 연결: VS Code, IntelliJ(JetBrains), Eclipse에서 Claude Code 쓰기

> 관련 모듈: 3
> 기준: 공식 문서 [VS Code](https://code.claude.com/docs/en/vs-code), [JetBrains](https://code.claude.com/docs/en/jetbrains) (2026년 10월 확인). 단축키·설정 이름은 버전에 따라 바뀔 수 있습니다.

---

## 1. 한눈에 보기

| IDE | 공식 지원 | 연결 방식 | 핵심 기능 |
|---|---|---|---|
| **VS Code** (Cursor 등 VS Code 계열 포함) | ○ 확장(Extension) | 그래픽 채팅 패널 (CLI 내장) | 인라인 diff, 선택 영역 공유, `@`멘션, 계획 검토, 여러 대화 탭, 체크포인트 |
| **IntelliJ IDEA 등 JetBrains** | ○ 플러그인 | IDE 터미널에서 실행하는 CLI와 연결 | IDE diff 뷰어, 선택 영역 공유, 진단(오류·경고) 공유 |
| **Eclipse** | ✕ (공식 플러그인 없음) | Eclipse 터미널·외부 터미널에서 CLI 사용 | 파일 수정은 동작, IDE 연동 기능은 없음 |

> 💡 **IDE 연동의 핵심은 "IDE와 Claude Code가 서로 정보를 주고받는 것"**입니다. 연결되면 내가 선택한 코드와 열린 파일이 자동으로 맥락이 되고, Claude의 변경은 IDE의 diff 화면으로 보이며, IDE가 찾은 컴파일 오류를 Claude가 읽을 수 있습니다.

---

## 2. VS Code (및 Cursor 등 VS Code 계열)

### 2-1. 설치

- **준비물**: VS Code 1.94.0 이상, Claude 유료 구독(Pro, Max, Team, Enterprise) 또는 Claude Console 계정
- 확장 보기(`Ctrl + Shift + X`)에서 **"Claude Code"** 검색 → 설치 (게시자: Anthropic)
- Cursor 등 다른 VS Code 계열 편집기에서도 같은 방법으로 설치 (Open VSX 레지스트리에도 등록됨)
- 처음 열 때 Anthropic 계정으로 로그인

### 2-2. 사용 방법

| 동작 | 방법 |
|---|---|
| Claude 열기 | 편집기 오른쪽 위 **Spark 아이콘** 클릭 (파일이 열려 있을 때), 또는 왼쪽 활동 표시줄의 Spark 아이콘 |
| 명령 찾기 | 명령 팔레트(`Ctrl + Shift + P`)에서 "Claude Code" 입력 |
| 선택한 코드 전달 | 코드를 선택하면 Claude가 자동으로 봄. `Alt + K`(Mac `Option + K`)로 `@파일#5-10` 형태 참조 삽입 |
| 변경 검토 | 제안된 수정을 diff로 보고 수락·거절 (변경 단위로도 가능) |
| 계획 검토 | Plan 모드의 계획을 편집·승인한 뒤 구현 |
| 되돌리기 | 메시지에 마우스를 올려 되돌리기 버튼 → 대화 분기 / 코드만 되돌리기 / 둘 다 |

### 2-3. 주요 단축키 (Windows/Linux 기준, Mac은 `Ctrl` → `Cmd`)

| 동작 | 단축키 |
|---|---|
| 편집기 ↔ Claude 포커스 전환 | `Ctrl + Esc` |
| 새 대화를 편집기 탭으로 열기 | `Ctrl + Shift + Esc` |
| 현재 파일·선택 영역 참조 삽입 | `Alt + K` |
| 닫은 Claude 탭 다시 열기 | `Ctrl + Shift + T` |

### 2-4. 확장(그래픽 패널) vs 터미널에서 CLI

| 기능 | CLI | VS Code 확장 |
|---|---|---|
| 명령·스킬 | 전체 | 일부 (`/`로 확인) |
| MCP 설정 | ○ | ○ (`/mcp`) |
| 체크포인트 | ○ | ○ |
| `!` 셸 명령 단축 | ○ | ✕ |
| Tab 자동완성 | ○ | ✕ |

- CLI 전용 기능이 필요하면 VS Code 통합 터미널(`` Ctrl + ` ``)에서 `claude` 실행 → 자동으로 IDE와 연동 (diff, 진단 공유)
- **주의:** 확장은 채팅 패널용 CLI를 내장하지만, 터미널에서 `claude` 명령을 쓰려면 **CLI를 따로 설치**해야 함
- 확장과 CLI는 **대화 기록을 공유** → 확장에서 하던 대화를 터미널에서 `claude --resume`으로 이어갈 수 있음
- 그래픽 패널 대신 CLI 화면을 선호하면 설정 **Use Terminal**(`claudeCode.useTerminal`)을 켬
- 여러 대화를 탭·창으로 동시에 열 수 있음 (탭의 점 색: 파랑 = 승인 대기, 주황 = 완료)
- `/plugins`로 플러그인 설치·관리 화면 사용 가능

---

## 3. IntelliJ IDEA (JetBrains IDE)

> Java·Spring Boot 개발자에게 가장 중요한 연동입니다. IntelliJ IDEA, WebStorm, PyCharm, Android Studio, GoLand 등 대부분의 JetBrains IDE에서 동작합니다.

### 3-1. 설치 (두 가지 모두 필요)

1. **Claude Code CLI 설치** (플러그인은 CLI를 포함하지 않음)
2. **JetBrains Marketplace에서 "Claude Code" 플러그인 설치** → IDE 재시작

`claude`가 PATH에 없으면 플러그인 설정의 **Claude command**에 전체 경로를 지정합니다.

### 3-2. 사용 방법

| 방법 | 설명 |
|---|---|
| IDE 안에서 | IDE 통합 터미널에서 `claude` 실행 → 모든 연동 기능 자동 활성화 |
| 빠른 실행 | `Ctrl + Esc`(Mac `Cmd + Esc`) 또는 Claude Code 버튼 |
| 외부 터미널에서 | 외부 터미널(Windows Terminal, tmux 등)에서 `claude` 실행 후 **`/ide`** → "Connected to IntelliJ IDEA." 확인 |

**연동되는 기능**

| 기능 | 설명 |
|---|---|
| **IDE diff 뷰어** | Claude의 변경이 IntelliJ의 diff 화면으로 열림 (`/config`의 Diff tool: `auto` / `terminal`) |
| **선택 영역 공유** | 현재 선택한 코드와 열린 파일이 자동으로 맥락이 됨 |
| **파일 참조 단축키** | `Alt + Ctrl + K`(Mac `Cmd + Option + K`) → `@src/.../EnrollmentService.java#L1-99` 삽입 |
| **진단 공유** | IntelliJ가 표시하는 컴파일 오류·경고를 Claude가 읽을 수 있음 |

### 3-3. 설정과 문제 해결

| 상황 | 해결 |
|---|---|
| `Esc`로 Claude 작업이 중단되지 않음 | Settings → Tools → Terminal에서 "Move focus to the editor with Escape" 해제 |
| WSL에서 실행 | 플러그인 Claude command를 `wsl -d Ubuntu -- bash -lic "claude"`로 설정 |
| WSL2에서 "No available IDEs detected" | Windows 방화벽에서 WSL2 대역 허용, 또는 WSL2 미러 네트워킹(`networkingMode=mirrored`) |
| JetBrains 원격 개발(Remote Development) | 플러그인을 **원격 호스트**에 설치 (Settings → Plugins (Host)) |
| 플러그인이 동작하지 않음 | 프로젝트 루트에서 `claude` 실행, 플러그인 활성화 확인, IDE 완전 재시작 |

> ⚠️ **보안 주의:** JetBrains IDE에서 `acceptEdits`나 `auto` 모드를 쓰면, Claude가 IDE가 자동 실행하는 설정 파일을 수정해 셸 명령 승인을 우회할 가능성이 있습니다. 공식 문서는 JetBrains에서는 **Manual 모드**를 권장합니다. ([권한 가이드](./23-permissions.md))

---

## 4. Eclipse

### 4-1. 현황

- **Anthropic 공식 Eclipse 플러그인은 없습니다** (2026년 10월 기준, 공식 GitHub에 기능 요청 이슈가 있음)
- Eclipse Marketplace에 **커뮤니티(비공식) 플러그인**이 있으나 Anthropic과 무관하며, 지원 OS 등 제약이 있을 수 있습니다
- 별개 플랫폼인 **Eclipse Theia**(웹 기반 IDE 프레임워크)는 Claude Code 연동을 발표했습니다

### 4-2. Eclipse에서 쓰는 방법

**방법 1. Eclipse 내장 터미널에서 CLI 실행 (권장)**

1. Claude Code CLI 설치
2. Eclipse에서 **Window → Show View → Terminal** (Terminal 뷰가 없으면 Eclipse Marketplace에서 TM Terminal 설치)
3. 터미널을 열고 **프로젝트 루트**로 이동해 `claude` 실행
4. Claude가 파일을 수정하면 Eclipse에서 **F5(Refresh)** 또는 자동 새로고침 설정(Preferences → General → Workspace → "Refresh using native hooks or polling")

**방법 2. 외부 터미널 + Eclipse 나란히 사용**

- Windows Terminal·tmux에서 `claude`를 실행하고 Eclipse는 편집·디버깅에 사용
- 변경 검토는 `/diff`, `git diff`, 또는 Eclipse의 Git Staging·Compare 화면 활용

**방법 3. 다른 도구와 병행**

- 변경 검토와 시각적 diff가 중요하면 **데스크톱 앱**을 함께 사용 ([환경 비교](./14-environments-and-terminals.md))
- Eclipse 사용자도 Claude 작업용으로만 VS Code 확장이나 IntelliJ를 보조로 쓰는 방법

**Eclipse 사용 시 팁**

| 팁 | 이유 |
|---|---|
| CLAUDE.md에 "빌드는 Gradle/Maven 명령으로" 명시 | Claude는 Eclipse 빌더가 아니라 명령으로 검증함 |
| Eclipse 전용 파일(`.project`, `.classpath`, `.settings/`)은 수정 금지 규칙 | 자동 생성·IDE 설정 파일 보호 |
| 외부 수정 후 Refresh 습관화 | Eclipse 작업 공간과 실제 파일이 어긋나는 것 방지 |

---

## 5. Java + React 팀 추천 구성

| 역할 | 추천 |
|---|---|
| 백엔드(Spring Boot) 개발자 | **IntelliJ IDEA + Claude Code 플러그인** (IDE 터미널에서 `claude`) |
| 프론트엔드(React) 개발자 | **VS Code + Claude Code 확장** |
| Eclipse 사용 팀 | Eclipse 터미널에서 CLI + 필요 시 데스크톱 앱으로 diff 검토 |
| 풀스택으로 한 저장소 | 저장소 루트에서 실행 (어느 IDE든 같은 CLAUDE.md·설정 적용) |

> 💡 IDE가 달라도 **CLAUDE.md, `.claude/` 설정, 스킬, 에이전트는 똑같이 적용**됩니다. 팀원마다 다른 IDE를 써도 AI 동작 규칙은 하나로 유지됩니다.

---

## 6. 실습

1. VS Code에 확장을 설치하고, `CourseController.java`의 메서드를 선택한 뒤 `Alt + K`로 참조를 넣어 "OpenAPI 어노테이션 보완해줘" → diff로 검토·수락
2. IntelliJ에 플러그인을 설치하고 IDE 터미널에서 `claude` 실행 → 일부러 컴파일 오류를 만든 뒤 "IDE에 표시된 오류를 확인하고 고쳐줘"
3. 외부 터미널에서 `claude` → `/ide`로 IntelliJ에 연결해 보기
4. (Eclipse 사용자) Eclipse 터미널에서 `claude`를 실행해 기능을 구현하고, Refresh 후 Eclipse에서 결과 확인
5. VS Code 확장에서 시작한 대화를 터미널에서 `claude --resume`으로 이어가기
