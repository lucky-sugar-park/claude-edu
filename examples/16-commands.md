# Claude Code 주요 명령어: 무엇이고, 왜 필요하고, 어떻게 쓰는가

> 관련 모듈: 3 (전 모듈에서 활용)
> 기준: 공식 문서 [CLI reference](https://code.claude.com/docs/en/cli-reference), [Commands](https://code.claude.com/docs/en/commands) (2026년 10월 확인)
> ⚠️ 명령어는 자주 추가·변경됩니다. 대화 중 `/`를 입력하면 지금 쓸 수 있는 명령 목록이 나옵니다.

---

## 1. 명령어의 세 종류

| 종류 | 입력 위치 | 예시 | 용도 |
|---|---|---|---|
| **CLI 명령·옵션** | 터미널(셸)에서 `claude` 실행 시 | `claude -c`, `claude -p "..."` | 세션을 **어떻게 시작할지** 정한다 |
| **슬래시 명령** | Claude Code 대화창에서 `/`로 시작 | `/clear`, `/compact`, `/model` | 세션 **안에서** Claude Code를 제어한다 |
| **입력 접두사** | 대화창에서 | `@파일`, `!명령` | 파일 참조, 셸 명령 바로 실행 |

슬래시 명령은 다시 두 가지로 나뉩니다.
- **내장 명령(built-in)**: Claude Code 자체 기능 (예: `/clear`, `/model`)
- **번들 스킬(bundled skill)**: 기본 제공되는 스킬. 사용자가 부르거나 Claude가 상황에 맞게 사용 (예: `/code-review`, `/simplify`)
- 여기에 **직접 만든 스킬·커맨드**도 `/이름`으로 부를 수 있다 ([스킬](./09-skills-catalog.md))

---

## 2. CLI 명령과 옵션 (터미널에서)

### 2-1. 세션 시작과 이어가기

| 명령 | 하는 일 | 왜 필요한가 |
|---|---|---|
| `claude` | 대화형 세션 시작 | 기본 사용법 |
| `claude "질문"` | 첫 지시를 함께 넣어 시작 | 바로 작업 시작 |
| `claude -c` (`--continue`) | 이 폴더의 가장 최근 대화 이어가기 | 어제 하던 작업 계속 |
| `claude -r` (`--resume`) | 세션 목록에서 골라 이어가기 (ID·이름 지정 가능) | 여러 작업 중 하나로 복귀 |
| `claude -n "이름"` (`--name`) | 세션에 이름 붙이기 → `claude -r 이름`으로 재개 | 세션을 찾기 쉽게 |
| `claude --fork-session` | 이어가되 새 세션으로 분기 (`-r`, `-c`와 함께) | 원래 대화를 보존하며 다른 시도 |
| `claude -w 이름` (`--worktree`) | 격리된 git worktree에서 시작 | 병렬 작업 ([세션 가이드](./15-project-directory-session.md)) |
| `claude --teleport` | 클라우드 세션을 로컬 터미널로 가져오기 | 웹에서 시작한 작업을 로컬에서 마무리 |
| `claude --cloud "작업"` | 새 클라우드 세션 생성 | 오래 걸리는 작업을 클라우드에 맡기기 |
| `claude --desktop` | 현재 폴더를 데스크톱 앱에서 열기 | 시각적 diff로 검토 |

### 2-2. 작업 범위와 동작 조정

| 옵션 | 하는 일 | 예시 |
|---|---|---|
| `--add-dir 경로` | 다른 폴더도 읽기·수정 허용 | `claude --add-dir ../eduhub-frontend` |
| `--model 모델` | 이 세션의 모델 지정 | `claude --model sonnet` |
| `--effort 수준` | 추론 노력 수준 지정 (`low`~`max`) | `claude --effort high` |
| `--permission-mode 모드` | 시작 권한 모드 (`plan`, `acceptEdits` 등) | `claude --permission-mode plan` |
| `--allowedTools` / `--disallowedTools` | 도구 허용·차단 규칙 | `--disallowedTools "Bash(git push *)"` |
| `--append-system-prompt "..."` | 기본 지침에 추가 지시 덧붙이기 | 일회성 규칙 |
| `--mcp-config 파일` | MCP 설정 파일 불러오기 | CI에서 필요한 MCP만 |
| `--safe-mode` | CLAUDE.md, 스킬, 훅, MCP 등 모든 사용자 설정을 끄고 시작 | 설정 문제 원인 찾기 |

### 2-3. 자동화 (헤드리스)

| 옵션 | 하는 일 | 예시 |
|---|---|---|
| `-p` (`--print`) | 대화 없이 한 번 실행하고 결과 출력 | `claude -p "이 함수 설명해줘"` |
| 파이프 입력 | 다른 명령의 출력을 넘김 | `./gradlew test 2>&1 \| tail -100 \| claude -p "실패 원인 분석"` |
| `--output-format` | `text`, `json`, `stream-json` | 결과를 스크립트에서 처리 |
| `--max-turns N` | 에이전트 턴 수 제한 | 무한 반복 방지 |
| `--max-budget-usd N` | 예상 비용 상한 | CI 비용 관리 ([토큰 최적화](./08-token-optimization.md)) |
| `--json-schema` | 지정한 스키마에 맞는 JSON 결과 받기 | 결과를 다른 시스템에 연동 |

### 2-4. 관리 명령

| 명령 | 하는 일 |
|---|---|
| `claude update` | 최신 버전으로 업데이트 |
| `claude doctor` | 설치·설정 진단 (세션 시작 없이) |
| `claude auth login` / `logout` / `status` | 로그인 관리 |
| `claude mcp ...` | MCP 서버 추가·목록·삭제 ([MCP 개념](./12-mcp-and-hooks-concepts.md)) |
| `claude plugin ...` | 플러그인 설치·관리 |
| `claude agents` | 백그라운드 세션을 모아 보고 실행하는 화면 |
| `claude setup-token` | CI·스크립트용 장기 토큰 발급 |
| `claude -v` | 버전 확인 |

---

## 3. 슬래시 명령 (대화창에서)

### 3-1. 꼭 알아야 할 10가지

| 명령 | 하는 일 | 왜 필요한가 | 사용 예 |
|---|---|---|---|
| `/help` | 도움말과 명령 목록 | 모를 때 첫 번째로 | `/help` |
| `/init` | 프로젝트를 분석해 CLAUDE.md 초안 생성 | 프로젝트 시작 시 기본 맥락 만들기 | 새 저장소에서 `/init` |
| `/clear` | 컨텍스트를 비우고 새 대화 시작 | 작업이 바뀌면 이전 맥락 제거 → 품질·비용 개선 | N+1 수정 끝 → `/clear` → 다음 작업 |
| `/compact [지시]` | 대화를 요약해 공간 확보 | 같은 작업을 계속하는데 컨텍스트가 찼을 때 | `/compact 설계 결정과 남은 TODO만 남겨줘` |
| `/context` | 컨텍스트 사용량을 시각화 | 무엇이 공간을 차지하는지 확인 | 응답이 느려지거나 품질이 떨어질 때 |
| `/resume` | 이전 세션 골라 이어가기 | 다른 작업으로 전환 | `/resume` |
| `/rewind` | 대화·코드를 이전 시점으로 되돌리기 (`Esc` 두 번) | 잘못된 방향의 수정 취소 | 리팩터링이 엉망이 됐을 때 |
| `/model` | 모델 변경 | 작업 난이도에 맞는 모델 선택 | 설계는 고성능, 단순 작업은 빠른 모델 |
| `/permissions` | 도구 허용·차단 규칙 관리 | 매번 승인하는 번거로움 해소, 위험 명령 차단 | `./gradlew test` 허용 |
| `/usage` (`/cost`) | 비용, 요금제 사용량, 통계 | 사용량 관리 | 하루 작업 후 확인 |

### 3-2. 작업 흐름별 명령

**① 시작·설정**

| 명령 | 하는 일 |
|---|---|
| `/init` | CLAUDE.md 초안 생성 |
| `/memory` | CLAUDE.md 편집, 자동 메모리 관리 |
| `/config` | 설정 화면 (테마, 모델, 출력 스타일 등) |
| `/status` | 버전, 모델, 계정, 연결 상태 |
| `/doctor` | 설치·설정·CLAUDE.md 문제 진단과 수정 제안 |
| `/terminal-setup` | `Shift + Enter` 줄바꿈 등 터미널 설정 |
| `/login` / `/logout` | 로그인 관리 |

**② 컨텍스트·세션 관리**

| 명령 | 하는 일 |
|---|---|
| `/clear [이름]` | 새 대화 시작 (이전 대화에 이름을 붙여 나중에 `/resume`) |
| `/compact [지시]` | 대화 요약 |
| `/context` | 컨텍스트 사용량 확인 |
| `/resume` | 세션 재개 |
| `/rename [이름]` | 현재 세션 이름 변경 |
| `/branch [이름]` | 현재 지점에서 대화를 분기해 다른 방향 시도 |
| `/rewind` | 이전 시점으로 되돌리기 |
| `/btw [질문]` | 대화에 남기지 않고 곁가지 질문 |
| `/export [파일]` | 대화 내용 내보내기 |
| `/add-dir 경로` | 다른 폴더 접근 허용 |
| `/cd 경로` | 대화를 유지한 채 작업 디렉터리 이동 |

**③ 작업 진행**

| 명령 | 하는 일 |
|---|---|
| `/plan [설명]` | Plan 모드로 들어가 계획부터 세우기 (`Shift + Tab`으로도 전환) |
| `/model`, `/effort` | 모델과 추론 노력 수준 조정 |
| `/fast` | 빠른 모드 켜기/끄기 |
| `/tasks` | 백그라운드 작업·서브에이전트 확인 |
| `/diff` | 지금까지의 변경 내용 검토 |
| `/copy [N]` | 마지막 응답(또는 코드 블록) 복사 |

**④ 검토·품질**

| 명령 | 종류 | 하는 일 |
|---|---|---|
| `/code-review` (`/review`) | 번들 스킬 | 현재 변경, PR, 브랜치의 버그 검토 |
| `/security-review` | 내장 | 현재 브랜치 변경의 보안 취약점 분석 |
| `/simplify` | 번들 스킬 | 변경 코드의 재사용·단순화·효율 개선 |
| `/debug` | 번들 스킬 | 디버그 로그를 켜고 문제 분석 |

**⑤ 확장 기능 관리**

| 명령 | 하는 일 | 관련 |
|---|---|---|
| `/skills` | 사용 가능한 스킬 목록, 토큰 비용 확인 | [스킬](./09-skills-catalog.md) |
| `/agents` | 서브에이전트 만들기·관리 안내 | [에이전트](./10-agents.md) |
| `/hooks` | 훅 설정 확인 | [훅](./12-mcp-and-hooks-concepts.md) |
| `/mcp` | MCP 서버 연결 상태, 인증, 켜기/끄기 | [MCP](./12-mcp-and-hooks-concepts.md) |
| `/plugin` | 플러그인 설치·활성화·관리 | [플러그인](./04-mcp-plugins-artifacts.md) |
| `/output-style` | 응답 스타일 변경 | |

**⑥ 환경 연동**

| 명령 | 하는 일 |
|---|---|
| `/ide` | IDE 연동 관리 |
| `/install-github-app` | Claude GitHub App 설치와 Actions 설정 ([GitHub](./06-github-version-control.md)) |
| `/desktop` | 현재 세션을 데스크톱 앱에서 이어가기 |
| `/teleport` | 클라우드 세션을 이 터미널로 가져오기 |
| `/remote-control` | 이 세션을 claude.ai·모바일에서 원격 조작 가능하게 |
| `/schedule` | 클라우드에서 실행되는 예약 작업(루틴) 만들기 |
| `/loop [간격] [지시]` | 세션이 열려 있는 동안 지시를 반복 실행 |

### 3-3. 입력 접두사

| 입력 | 하는 일 | 예시 |
|---|---|---|
| `/` | 명령·스킬 목록 (글자를 이어 치면 필터링) | `/com` → `/compact`, `/code-review` |
| `@` | 파일·폴더를 맥락에 넣기 (Tab 자동완성) | `@backend/.../EnrollmentService.java 이 클래스 설명해줘` |
| `!` | 셸 명령 바로 실행 (결과가 대화에 포함) | `!git status` |

---

## 4. 상황별 명령 활용 시나리오

### 시나리오 1. 새 프로젝트에 처음 도입

```text
$ cd ~/projects/eduhub
$ claude
> /init                      # CLAUDE.md 초안 생성
> /memory                    # 초안을 열어 팀 규칙 보완
> /permissions               # ./gradlew test, npm test 허용
> /mcp                       # 필요한 MCP 서버 연결 확인
```

### 시나리오 2. 하루 작업 흐름

```text
$ claude -c                                  # 어제 작업 이어가기
> docs/progress/42.md 읽고 남은 일 진행해줘
> /diff                                      # 변경 내용 검토
> /code-review                               # 자체 리뷰
> /clear 수강취소-완료                         # 작업 마무리, 이름 붙여 정리
> /plan 대기자 자동 승격 기능                    # 다음 작업은 계획부터
```

### 시나리오 3. 대화가 길어지고 품질이 떨어질 때

```text
> /context                                   # 무엇이 공간을 차지하는지 확인
> /compact 설계 결정과 남은 TODO만 남겨줘       # 같은 작업 계속이면 요약
  (또는) /clear                               # 작업이 바뀌었으면 비우기
```

### 시나리오 4. 잘못된 방향으로 수정했을 때

```text
(Esc)                                        # 즉시 중단
> /rewind                                    # 수정 전 시점으로 되돌리기
> 다시 지시: EnrollmentService만 수정하고 public 메서드 시그니처는 유지해줘
```

### 시나리오 5. 자동화 (CI)

```bash
claude -p "backend에서 @Operation이 없는 컨트롤러 메서드를 찾아줘" \
  --output-format json --max-turns 10 --max-budget-usd 2 > audit.json
```

### 시나리오 6. 설정 문제 해결

```bash
claude doctor              # 설치·설정 진단
claude --safe-mode         # 사용자 설정을 모두 끄고 실행해 원인 분리
```

```text
> /doctor                  # 세션 안에서 진단과 수정 제안
> /hooks                   # 훅이 원인인지 확인
> /mcp                     # MCP 연결 문제 확인
```

---

## 5. 나만의 명령 만들기

자주 쓰는 지시는 스킬로 만들어 `/이름`으로 부릅니다.

```markdown
<!-- .claude/skills/release-notes/SKILL.md -->
---
name: release-notes
description: 지난 태그 이후 커밋을 분석해 릴리스 노트 초안을 작성한다. 사용자가 릴리스 노트를 요청할 때 사용한다.
---
1. `git log $(git describe --tags --abbrev=0)..HEAD --oneline`으로 변경 목록을 확인한다
2. Conventional Commits 유형별(feat, fix, refactor)로 분류한다
3. 사용자 관점의 설명으로 바꿔 `docs/release-notes/` 아래에 저장한다
```

```text
> /release-notes
```

> 예전 방식인 `.claude/commands/*.md` 파일도 슬래시 명령으로 동작합니다. 새로 만든다면 Claude가 상황에 맞게 스스로 쓸 수도 있는 **스킬** 형식을 권장합니다 ([비교](./13-choosing-the-right-tool.md)).

---

## 6. 실습

1. `/`를 입력해 전체 명령 목록을 훑어보고, 이 문서에 없는 명령 3개를 찾아 용도 조사
2. 시나리오 2의 하루 흐름을 EduHub로 그대로 따라 하기
3. `/context`로 사용량을 확인한 뒤 `/compact`와 `/clear`를 각각 써 보고 차이 비교
4. 일부러 잘못된 수정을 시킨 뒤 `/rewind`로 되돌리기
5. 자주 쓰는 지시 하나를 스킬로 만들어 `/이름`으로 실행
