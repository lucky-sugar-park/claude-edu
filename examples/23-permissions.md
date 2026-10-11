# 권한(Permissions): 사용자가 Claude Code에게 주는 권한 — 종류, 설정 방법, 모범 사례

> 관련 모듈: 9 (12, 20과 연결)
> 기준: 공식 문서 [Configure permissions](https://code.claude.com/docs/en/permissions), [Permission modes](https://code.claude.com/docs/en/permission-modes) (2026년 10월 확인). 세부 동작은 버전에 따라 바뀔 수 있습니다.

---

## 1. 왜 권한이 중요한가

Claude Code는 **내 컴퓨터에서 내 권한으로** 파일을 고치고 명령을 실행합니다. 권한 설정은 "Claude가 **무엇을, 어디까지, 묻지 않고** 할 수 있는가"를 정하는 장치입니다.

| 권한이 너무 좁으면 | 권한이 너무 넓으면 |
|---|---|
| 매번 승인 요청 → 흐름이 끊김 | 실수로 중요한 파일 삭제, 비밀정보 노출 |
| 승인 피로 → 내용을 보지 않고 "예"를 누르게 됨 | 프롬프트 인젝션에 취약 (외부 문서의 숨은 지시) |
| 에이전트의 장점이 사라짐 | 되돌리기 어려운 작업(푸시, 배포) 발생 |

**목표:** 안전한 일은 **자동으로**, 위험한 일은 **확인 후**, 금지된 일은 **절대 못 하게**.

---

## 2. 어떤 권한이 있나

### 2-1. 기본 동작 (Manual 모드 기준)

| 도구 종류 | 예 | 승인 필요? | "다시 묻지 않기" 선택 시 |
|---|---|---|---|
| **읽기 전용** | 파일 읽기, 검색(Grep) | 아니오 (작업 디렉터리·추가 디렉터리 안에서) | — |
| **셸 명령** | `Bash` | 예 (기본 제공 읽기 전용 명령 일부 제외) | 이 저장소에서 해당 명령은 계속 허용 |
| **파일 수정** | `Edit`, `Write` | 예 | 세션이 끝날 때까지 |
| **웹 가져오기** | `WebFetch` | 예 (사전 승인된 문서 도메인 제외) | 이 저장소에서 해당 도메인 계속 허용 |
| **웹 검색** | `WebSearch` | 예 | 이 저장소에서 계속 허용 |
| **MCP 도구** | `mcp__서버__도구` | 설정에 따라 | |

### 2-2. 권한 규칙 세 종류

| 규칙 | 의미 | 예 |
|---|---|---|
| **allow** | 묻지 않고 실행 | 테스트, 린트, 포맷 명령 |
| **ask** | 매번 확인 | `git push`, 배포 스크립트 |
| **deny** | 실행 차단 | `.env` 읽기, 강제 푸시, 운영 DB 접속 |

**평가 순서: deny → ask → allow.** 먼저 걸리는 규칙이 결과를 정합니다.
- 넓은 deny(`Bash(aws *)`)는 좁은 allow(`Bash(aws s3 ls)`)보다 **항상 우선**한다. allow로 deny에 예외를 만들 수 없다
- 어느 설정 파일에서든 **deny가 하나라도 있으면** 다른 곳의 allow로 풀 수 없다

### 2-3. 권한 모드

| 모드 | 동작 | 추천 상황 |
|---|---|---|
| `default` (화면 표시: **Manual**) | 처음 쓰는 도구마다 승인 요청 | 처음 사용, 중요한 저장소 |
| `acceptEdits` | 작업 디렉터리 안의 파일 수정과 `mkdir`, `mv`, `cp` 같은 기본 파일 명령은 자동 승인 | 믿을 수 있는 작업을 빠르게 |
| `plan` | 읽기·탐색과 읽기 전용 명령만, 소스 수정 안 함 | 계획 세우기, 코드 분석 |
| `auto` | 일상적인 승인 요청 없이 진행. 실행 전에 분류기가 요청 의도와 맞는지 검사 | 반복 작업을 덜 끊기게 (분류기를 믿을 수 있는 환경) |
| `dontAsk` | 승인이 필요한 호출은 모두 자동 거부, 미리 허용된 것만 실행 | 무인 실행, 엄격한 자동화 |
| `bypassPermissions` | 승인 요청 생략 (일부 보호 장치만 유지) | **컨테이너·VM 같은 격리 환경에서만** |

- 전환: 대화 중 `Shift + Tab`, 시작 시 `claude --permission-mode plan`
- 기본 모드 지정: 설정 파일의 `defaultMode`
- `bypassPermissions`는 `.git`, `.claude` 같은 보호 경로 쓰기도 묻지 않으므로 특히 주의
- 조직에서 `bypassPermissions`나 `auto`를 못 쓰게 하려면 `permissions.disableBypassPermissionsMode`, `permissions.disableAutoMode`를 `"disable"`로 설정 (보통 관리 설정에)

### 2-4. 접근 범위: 작업 디렉터리

| 방법 | 설명 |
|---|---|
| 기본 | `claude`를 실행한 폴더와 그 하위 |
| `--add-dir 경로` / `/add-dir` | 다른 폴더의 파일 접근 추가 (설정은 대부분 불러오지 않음) |
| `additionalDirectories` (설정) | 추가 폴더를 영구 설정 |
| `/cd 경로` | 세션을 다른 폴더로 이동 (그 폴더의 설정 적용) |
| `blockReadsOutsideWorkingDirectories` | 작업 디렉터리 밖 읽기를 모든 모드에서 차단 |

---

## 3. 어떻게 설정하나

### 3-1. 설정하는 곳

| 방법 | 범위 | 언제 |
|---|---|---|
| **승인 요청 화면에서 "다시 묻지 않기"** | 저장소 또는 세션 | 사용하면서 자연스럽게 |
| **`/permissions`** | 선택한 설정 파일 | 규칙 확인·추가·삭제, 대화 중에도 즉시 반영 |
| **`.claude/settings.json`** | 프로젝트 (팀 공유, 커밋) | 팀 공통 규칙 |
| **`.claude/settings.local.json`** | 프로젝트 (개인, 커밋 금지) | 개인 선호 |
| **`~/.claude/settings.json`** | 내 모든 프로젝트 | 개인 기본값 |
| **관리 설정 (managed settings)** | 조직 전체 | 회사 보안 정책. 누구도 덮어쓸 수 없음 |
| **CLI 옵션** | 이번 실행만 | `--allowedTools`, `--disallowedTools`, `--permission-mode` |

**우선순위:** 관리 설정 > CLI 옵션 > 프로젝트 개인 > 프로젝트 공유 > 사용자. 단, **deny는 어느 단계에 있든 이긴다.**

### 3-2. 규칙 문법

```
도구이름                 → 그 도구 전체
도구이름(지정자)          → 특정 사용만
```

| 도구 | 규칙 예시 | 의미 |
|---|---|---|
| **Bash** | `Bash(npm run build)` | 정확히 이 명령 |
| | `Bash(npm run *)` | `npm run`으로 시작하는 모든 명령 (`*`는 하위 명령 뒤에) |
| | `Bash(git push *)` | 모든 `git push` |
| **Read / Edit** | `Read(./.env)` | 현재 폴더의 `.env` 읽기 |
| | `Read(./secrets/**)` | `secrets` 폴더 전체 |
| | `Edit(/src/**/*.ts)` | 프로젝트 설정 기준 `src` 아래 `.ts` 수정 |
| | `Read(~/.ssh/**)` | 홈 디렉터리의 `.ssh` |
| | `Read(//etc/**)` | 절대 경로 (슬래시 두 개) |
| **WebFetch** | `WebFetch(domain:docs.spring.io)` | 특정 도메인 |
| | `WebFetch(domain:*.example.com)` | 하위 도메인 전체 |
| **MCP** | `mcp__github` 또는 `mcp__github__*` | 해당 서버의 모든 도구 |
| | `mcp__github__create_issue` | 특정 도구 |
| **서브에이전트** | `Agent(code-explorer)` | 특정 서브에이전트 사용 |

**경로 패턴 주의**
| 패턴 | 의미 |
|---|---|
| `//경로` | 파일 시스템 루트부터의 **절대 경로** |
| `~/경로` | 홈 디렉터리 기준 |
| `/경로` | **설정 파일 위치 기준** (프로젝트 설정이면 프로젝트 루트). 절대 경로가 아님에 주의 |
| `경로`, `./경로` | 현재 디렉터리 기준 |

- 윈도우 경로는 `/c/Users/...` 형태로 바뀌어 비교된다 (예: `//c/**/.env`)
- `Read` deny는 같은 경로의 `Edit`·`Write`도 막고, `cat`·`head` 같은 셸 파일 명령에도 최대한 적용된다. 하지만 **모든 우회를 완벽히 막지는 못하므로**, 중요한 비밀은 샌드박스나 파일 위치 분리와 함께 보호한다
- 경로 규칙은 `Read`와 `Edit`로 작성한다 (`Write`로 쓴 경로 규칙은 적용되지 않음)

### 3-3. EduHub 팀 설정 예시

```json
// .claude/settings.json (팀 공유)
{
  "permissions": {
    "defaultMode": "default",
    "allow": [
      "Bash(./gradlew test *)",
      "Bash(./gradlew spotlessApply)",
      "Bash(./gradlew build)",
      "Bash(npm run *)",
      "Bash(npm test *)",
      "Bash(./scripts/verify.sh)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git commit *)",
      "WebFetch(domain:docs.spring.io)",
      "WebFetch(domain:mui.com)",
      "mcp__eduhub-db__query"
    ],
    "ask": [
      "Bash(git push *)",
      "Bash(docker *)",
      "mcp__github__create_pull_request"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./backend/src/main/resources/application-prod.yml)",
      "Read(~/.ssh/**)",
      "Edit(./frontend/packages/api-client/src/generated/**)",
      "Bash(git push --force*)",
      "Bash(./gradlew flywayClean*)",
      "mcp__github__merge_pull_request"
    ]
  }
}
```

| 구분 | 설계 의도 |
|---|---|
| allow | 테스트·빌드·린트·조회처럼 **안전하고 자주 쓰는 것** → 흐름이 끊기지 않게 |
| ask | 외부에 영향을 주는 것(푸시, PR 생성, Docker) → 사람이 확인 |
| deny | 비밀정보, 생성 코드 수정, 되돌릴 수 없는 명령 → 절대 금지 |

### 3-4. 개인 설정 예시

```json
// .claude/settings.local.json (커밋 금지)
{
  "permissions": {
    "defaultMode": "acceptEdits",
    "allow": ["Bash(./gradlew bootRun)"]
  }
}
```

---

## 4. 함께 알아둘 안전장치

| 장치 | 설명 |
|---|---|
| **작업 공간 신뢰(Workspace trust)** | 처음 여는 폴더에서는 신뢰 확인 창이 뜬다. 프로젝트 `.claude/settings.json`의 allow 규칙과 추가 디렉터리는 **신뢰를 수락한 뒤에만** 적용된다 → 남의 저장소가 권한을 몰래 넓히지 못함 |
| **MCP 서버 승인** | 프로젝트 `.mcp.json`의 서버는 처음 사용할 때 승인 요청 |
| **훅** | 권한 규칙으로 표현하기 어려운 검사를 스크립트로 (`PreToolUse`에서 차단) ([훅](./12-mcp-and-hooks-concepts.md)) |
| **샌드박스** (`/sandbox`) | 운영체제 수준에서 셸 명령의 파일·네트워크 접근을 제한. 권한 규칙과 **함께 쓰면 이중 방어** (프롬프트 인젝션으로 판단이 흔들려도 샌드박스는 유지) |
| **체크포인트·되돌리기** | 파일 수정은 `/rewind`로 되돌릴 수 있다 (단, 외부로 나간 행동은 되돌릴 수 없음) |
| **관리 설정** | 조직이 규칙을 강제. `allowManagedPermissionRulesOnly`로 관리 설정의 규칙만 쓰게 할 수도 있음 |

> ⚠️ **비대화형 실행(`claude -p`, Agent SDK)에는 신뢰 확인 창이 없습니다.** CI에서는 `--allowedTools`, `--disallowedTools`, `--permission-mode`로 필요한 권한만 명시적으로 지정하세요.

---

## 5. 상황별 권장 설정

| 상황 | 권장 |
|---|---|
| 처음 도입하는 팀 | Manual 모드 + 테스트·린트만 allow + 비밀정보 deny. 1~2주 사용하며 자주 승인하는 명령을 allow로 옮기기 |
| 숙련된 개인 개발 | `acceptEdits` + 팀 규칙. 푸시·배포는 ask |
| 코드 분석·설계 | `plan` 모드 |
| 클라우드 세션, 컨테이너 | 격리 환경이므로 더 넓게 허용 가능. 그래도 비밀정보 deny는 유지 |
| CI (헤드리스) | `--permission-mode dontAsk` + 필요한 도구만 `--allowedTools` |
| 리뷰 전용 서브에이전트 | `tools: Read, Grep, Glob`로 읽기 전용 |
| 회사 전체 | 관리 설정으로 `bypassPermissions` 금지, 비밀 경로 deny, 허용 MCP 서버 목록 |

---

## 6. 모범 사례

1. **deny부터 정한다**: 절대 안 되는 것(비밀정보, 운영 환경, 강제 푸시)을 먼저 막는다
2. **allow는 구체적으로**: `Bash(*)` 대신 `Bash(./gradlew test *)`
3. **외부 영향은 ask**: 푸시, PR 생성, 메시지 전송, 배포
4. **팀 규칙은 커밋하고 리뷰**: `.claude/settings.json`에 CODEOWNERS 지정
5. **승인 요청을 읽는다**: 습관적인 "예"가 가장 위험하다. 자주 묻는 안전한 명령은 allow로 옮겨 피로를 줄인다
6. **`bypassPermissions`는 격리 환경에서만**
7. **이중 방어**: 권한 규칙 + 훅 + 샌드박스 + 비밀정보를 저장소 밖에 두기
8. **주기적으로 점검**: `/permissions`로 쌓인 규칙을 확인하고 불필요한 것 정리

---

## 7. 실습

1. `/permissions`를 열어 현재 규칙과 각 규칙이 어느 파일에서 왔는지 확인
2. 위 EduHub 팀 설정을 적용하고 다음을 확인
   - `./gradlew test`는 묻지 않고 실행되는가
   - `git push`는 확인을 요청하는가
   - `.env`를 읽어 달라고 하면 차단되는가
3. `Shift + Tab`으로 모드를 바꿔 가며 같은 수정 요청에 대한 동작 차이 관찰 (Manual → acceptEdits → plan)
4. 사용자 설정에 allow, 프로젝트 설정에 같은 명령의 deny를 넣고 어느 쪽이 이기는지 확인
5. CI용 명령 작성: `claude -p "..." --permission-mode dontAsk --allowedTools "Read" "Grep" "Bash(./gradlew test *)"`
