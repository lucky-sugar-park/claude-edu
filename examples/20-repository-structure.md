# 모범 사례: Claude Code가 일하기 좋은 저장소(루트 디렉터리) 구조

> 관련 모듈: 8 (7, 9~16과 연결)
> 예제 프로젝트: [EduHub](./00-sample-project.md)
> ⚠️ 파일 위치와 이름은 Claude Code 버전에 따라 바뀔 수 있습니다. 표시된 항목은 2026년 10월 공식 문서 기준이며, 그 밖의 폴더(`docs/progress` 등)는 **팀 관례로 정하는 권장 구조**입니다.

---

## 1. 왜 구조가 중요한가

Claude는 매 세션마다 **처음 온 동료**처럼 저장소를 탐색합니다. 구조가 좋으면:

| 효과 | 설명 |
|---|---|
| **탐색이 빠르고 정확** | 이름과 위치만 보고 무엇이 어디 있는지 안다 → 토큰 절약 |
| **규칙이 자동 적용** | 루트·하위 CLAUDE.md, `.claude/` 설정이 위치에 따라 자동으로 읽힌다 |
| **검증이 쉬움** | 정해진 검증 명령이 있으면 Claude가 스스로 확인한다 |
| **사람과 AI가 같은 지도를 봄** | 사람에게 좋은 구조가 AI에게도 좋은 구조 |

---

## 2. 전체 구조 (EduHub 기준 권장안)

```
eduhub/                              ← 저장소 루트 = Claude Code 작업 디렉터리 (여기서 claude 실행)
│
├── CLAUDE.md                        ★ 항상 읽힘: 프로젝트 개요, 명령어, 공통 규칙, 지도
├── README.md                          사람용 소개 (CLAUDE.md에서 참조 가능)
│
├── .claude/                         ★ Claude Code 프로젝트 설정 (팀 공유)
│   ├── settings.json                  권한, 훅, 환경 변수 (커밋)
│   ├── settings.local.json            개인 설정 (커밋 금지)
│   ├── skills/                        스킬: 작업 절차·템플릿
│   │   ├── add-api-endpoint/SKILL.md
│   │   └── new-microfrontend/SKILL.md
│   ├── agents/                        서브에이전트 정의
│   │   ├── code-explorer.md
│   │   └── jpa-reviewer.md
│   ├── commands/                      (예전 방식) 슬래시 커맨드
│   ├── hooks/                         훅에서 실행할 스크립트 (팀 관례)
│   │   ├── format.sh
│   │   └── protect-files.sh
│   ├── launch.json                    데스크톱 앱 미리보기용 개발 서버 설정
│   └── worktrees/                     병렬 세션용 git worktree (자동 생성, 커밋 금지)
│
├── .mcp.json                        ★ 팀 공용 MCP 서버 (비밀정보는 환경 변수로)
├── .worktreeinclude                   워크트리에 복사할 Git 제외 파일 목록 (예: .env)
│
├── docs/                              사람과 Claude가 함께 보는 문서
│   ├── architecture.md                아키텍처 개요 (CLAUDE.md에서 @로 참조)
│   ├── adr/                           설계 결정 기록 (Architecture Decision Records)
│   ├── design/                        기능별 설계안 (architect 에이전트 산출물)
│   ├── progress/                      작업 진행 메모 (세션 간 맥락 전달)
│   └── runbooks/                      운영·장애 대응 절차
│
├── reference/                         모범 구현 예제 (골든 예제)
│   ├── backend/SampleCrudController.java
│   └── frontend/SampleListPage.tsx
│
├── scripts/                           검증·자동화 스크립트
│   ├── verify.sh                      ★ "이것만 통과하면 된다" 단일 검증 명령
│   └── check-openapi.sh
│
├── backend/
│   ├── CLAUDE.md                      백엔드 전용 규칙 (backend 파일 작업 시 읽힘)
│   └── src/...
│
├── frontend/
│   ├── CLAUDE.md                      프론트 전용 규칙
│   ├── packages/api-client/src/generated/   자동 생성 코드 (수정 금지 표시)
│   └── apps/...
│
├── .github/
│   ├── CODEOWNERS                     .claude/, db/migration/ 등 리뷰어 지정
│   ├── pull_request_template.md       Claude가 PR 본문 작성 시 따름
│   └── workflows/                     CI, Claude 자동 리뷰
│
├── .gitignore
└── .gitattributes                     줄바꿈 규칙 고정 (윈도우·WSL 혼용 대비)
```

★ = Claude Code가 **자동으로 인식**하는 핵심 위치

---

## 3. 영역별 설명

### 3-1. `CLAUDE.md` (루트): 저장소의 "지도"

루트 CLAUDE.md는 **목차와 지도** 역할을 합니다. 상세 내용은 다른 파일로 보내고 위치만 알려줍니다.

```markdown
# EduHub
사내 교육 신청 시스템. Spring Boot 백엔드 + React 마이크로프론트엔드 모노레포.

## 지도
- backend/: Spring Boot (규칙: backend/CLAUDE.md)
- frontend/: React MFE (규칙: frontend/CLAUDE.md)
- docs/architecture.md: 아키텍처 개요
- docs/adr/: 주요 설계 결정과 이유
- reference/: 새 코드를 만들 때 따라야 할 모범 예제
- 자동 생성 코드: frontend/packages/api-client/src/generated/ (직접 수정 금지)

## 검증
- 모든 작업은 `./scripts/verify.sh`가 통과해야 완료

## 작업 기록
- 여러 세션에 걸친 작업은 docs/progress/{이슈번호}-{이름}.md에 진행 상황을 남긴다
- 설계안은 docs/design/에 저장한다
```

### 3-2. `.claude/`: Claude Code 설정 폴더

| 항목 | 용도 | 커밋 | 관련 문서 |
|---|---|---|---|
| `settings.json` | 팀 권한·훅·환경 변수 | ○ | [설정](./03-settings-skills-agents-hooks.md) |
| `settings.local.json` | 개인 설정 | ✕ | |
| `skills/` | 작업 절차 | ○ | [스킬](./09-skills-catalog.md) |
| `agents/` | 서브에이전트 | ○ | [에이전트](./18-java-react-agents.md) |
| `commands/` | 예전 방식 커맨드 (새로 만들면 스킬 권장) | ○ | [명령어](./16-commands.md) |
| `hooks/` (팀 관례) | 훅 스크립트를 모아 두는 곳 | ○ | [훅](./12-mcp-and-hooks-concepts.md) |
| `launch.json` | 데스크톱 앱에서 앱 미리보기 시 실행할 개발 서버 | ○ | [병렬 작업](./17-parallel-work-terminal-vs-desktop.md) |
| `worktrees/` | 병렬 세션의 워크트리 (기본 위치) | ✕ | [세션](./15-project-directory-session.md) |

> 💡 `.claude/` 변경은 팀 전체의 AI 동작을 바꿉니다. **CODEOWNERS로 리뷰어를 지정**하세요.

### 3-3. `docs/`: Claude의 "장기 기억"

세션은 끝나면 대화를 잊지만, **파일은 남습니다**. 그래서 `docs/`는 Claude가 일하기 위한 핵심 작업 공간입니다.

| 폴더 | 누가 쓰나 | 언제 읽나 |
|---|---|---|
| `architecture.md` | 사람 (Claude가 초안 작성 가능) | 큰 변경, 새 기능 설계 시 (CLAUDE.md에서 `@docs/architecture.md`로 연결 가능) |
| `adr/` | 사람 + Claude | "왜 이렇게 했지?" 확인할 때. 예: `0003-use-module-federation.md` |
| `design/` | `architect` 에이전트, Plan 모드 결과 | 구현 에이전트가 설계를 따를 때 |
| `progress/` | Claude가 작업하며 갱신 | 다음 세션이 이어서 작업할 때 |
| `runbooks/` | 사람 | 장애 대응, 배포 작업 시 |

**진행 메모 예시** (`docs/progress/42-cancel-enrollment.md`)는 [세션 가이드](./15-project-directory-session.md#4-4-세션-사이에-맥락-넘기기)를 참고하세요.

> `progress/`를 커밋할지는 팀이 정합니다. 커밋하면 다른 사람·세션과 공유되고, 작업이 끝나면 정리합니다. 개인 메모로만 쓰려면 `.gitignore`에 추가합니다.

### 3-4. `reference/`: 모범 예제

- "이 파일처럼 만들어줘"라고 지시할 수 있는 **작고 완성도 높은 예제**
- 실제 서비스 코드에서 고르면 시간이 지나며 나쁜 패턴이 섞이므로, 별도 폴더에서 관리하고 리뷰
- 사내 프레임워크가 있다면 특히 효과적 ([커스텀 프레임워크](./07-custom-framework.md))

### 3-5. `scripts/verify.sh`: 단일 검증 명령

Claude에게 **"이것만 통과하면 완료"**라는 기준을 주는 것이 가장 강력한 품질 장치입니다.

```bash
#!/usr/bin/env bash
# scripts/verify.sh — 변경된 영역만 빠르게 검증
set -euo pipefail

changed=$(git diff --name-only origin/main...HEAD; git diff --name-only)

if echo "$changed" | grep -q '^backend/'; then
  echo "▶ backend"
  (cd backend && ./gradlew spotlessCheck test -q)
fi

if echo "$changed" | grep -q '^frontend/'; then
  echo "▶ frontend"
  (cd frontend && npm run lint --silent && npm test --silent)
fi

echo "✅ verify passed"
```

- 출력은 짧게 (토큰 절약), 실패 시에만 자세히
- CI도 같은 스크립트를 사용 → 로컬과 CI 결과 일치
- CLAUDE.md, 스킬, 에이전트 정의에서 모두 이 명령을 참조

### 3-6. 자동 생성 코드는 분리하고 표시

| 방법 | 예 |
|---|---|
| 별도 폴더 | `packages/api-client/src/generated/` |
| 파일 머리말 | `// AUTO-GENERATED. DO NOT EDIT. Run: npm run generate:api` |
| CLAUDE.md에 명시 | "생성 코드는 직접 수정 금지, 재생성 명령 사용" |
| 훅으로 강제 | PreToolUse 훅으로 수정 차단 ([예제](./03-settings-skills-agents-hooks.md#모듈-12-훅)) |
| 읽기 제외 (선택) | 권한 설정으로 읽기 차단 → 토큰 절약 (대신 스펙 위치 안내) |

---

## 4. 무엇을 커밋하고 무엇을 제외하나

```gitignore
# Claude Code — 개인·임시 파일
.claude/settings.local.json
.claude/worktrees/

# 비밀정보
.env
.env.*
!.env.example

# 빌드 산출물 (Claude가 탐색하지 않도록)
backend/build/
frontend/**/dist/
frontend/**/node_modules/
```

| 커밋 ○ | 커밋 ✕ |
|---|---|
| `CLAUDE.md`(모든 계층), `.claude/settings.json`, `skills/`, `agents/`, `hooks/` | `.claude/settings.local.json` |
| `.mcp.json` (비밀정보 없이 `${VAR}`만) | `.env`, 토큰, 인증 정보 |
| `docs/`, `reference/`, `scripts/` | `.claude/worktrees/` |
| `.env.example` (변수 이름만) | 빌드 산출물, 의존성 폴더 |

---

## 5. 구조 설계 원칙

| 원칙 | 설명 |
|---|---|
| **루트에서 실행** | 작업 디렉터리 = 저장소 루트. 하위 폴더에서만 작업하면 공통 설정이 빠질 수 있음 |
| **계층형 CLAUDE.md** | 루트는 지도와 공통 규칙, 하위 폴더는 영역 규칙 → 필요한 규칙만 읽힘 |
| **예측 가능한 이름** | `course/`, `enrollment/`처럼 도메인 이름 그대로. 약어·임의 이름 피하기 |
| **도메인별 패키지** | 계층별(`controller/`, `service/`)보다 도메인별이 관련 코드를 한 곳에서 찾기 쉬움 |
| **파일은 적당히 작게** | 수천 줄 파일은 읽기 비용이 크고 수정 정확도도 떨어짐 |
| **생성물·산출물 분리** | 빌드 결과, 생성 코드, 의존성은 명확히 분리하고 탐색 대상에서 제외 |
| **문서는 코드 가까이** | 모듈별 README, 하위 CLAUDE.md |
| **검증 명령은 하나로** | `scripts/verify.sh` |
| **AI 산출물의 자리 지정** | 설계안, 진행 메모, 리포트를 둘 위치를 CLAUDE.md에 명시 → 루트가 어지러워지지 않음 |

---

## 6. 상황별 구성

### 6-1. 멀티 저장소 (백엔드·프론트 분리)

```
~/work/eduhub-workspace/        ← 여기서 claude 실행 (또는 각 저장소에서 --add-dir)
├── CLAUDE.md                   두 저장소의 관계, 공통 규칙
├── eduhub-backend/             (Git 저장소, 자체 CLAUDE.md·.claude/)
└── eduhub-frontend/            (Git 저장소, 자체 CLAUDE.md·.claude/)
```

- 주로 한 저장소에서 작업하면 그 저장소에서 실행하고 `--add-dir ../eduhub-frontend`
- `--add-dir`로 추가한 폴더는 **파일 접근만 허용**되고, 그 폴더의 `.claude/` 설정 대부분은 자동으로 불러오지 않는다는 점에 유의
- 공통 규칙이 많다면 **플러그인**으로 묶어 양쪽에 설치

### 6-2. 마이크로프론트엔드가 저장소별로 분리

- remote 저장소마다 CLAUDE.md에 "shared 의존성 버전은 shell 저장소의 기준을 따른다"를 명시
- shell 저장소의 `docs/`에 MFE 통합 규칙을 두고, 각 remote의 CLAUDE.md에서 위치 안내

### 6-3. 사용자 범위 구성 (`~/.claude/`)

```
~/.claude/
├── CLAUDE.md          개인 선호 (응답 언어, 설명 스타일)
├── settings.json      개인 기본 권한, 모델
├── skills/            어느 프로젝트에서나 쓰는 개인 스킬
└── agents/            개인 서브에이전트
```

프로젝트 규칙은 프로젝트에, 개인 취향은 사용자 범위에 둡니다. 팀 규칙을 개인 설정에 넣으면 다른 팀원에게 적용되지 않습니다.

---

## 7. 새 프로젝트에 적용하는 순서

1. 저장소 루트에서 `claude` 실행 → `/init`으로 CLAUDE.md 초안 생성
2. 루트 CLAUDE.md를 "지도 + 명령어 + 공통 규칙"으로 다듬고, 영역별 하위 CLAUDE.md 분리
3. `scripts/verify.sh` 작성, CLAUDE.md에 등록
4. `.claude/settings.json`(권한), `.gitignore`(local 설정, worktrees) 정리
5. `docs/` 구조 만들기 (architecture, adr, design, progress)
6. 반복 작업이 보이면 `skills/`, `agents/`, `hooks/` 추가
7. `.github/CODEOWNERS`에 `.claude/`, `CLAUDE.md` 리뷰어 지정

---

## 8. 실습

1. EduHub에 위 구조를 적용하고, 적용 전후로 "이 프로젝트 구조를 설명해줘"를 실행해 정확도와 `/context` 사용량 비교
2. `scripts/verify.sh`를 만들고 CLAUDE.md에 등록한 뒤, 기능 구현을 시켜 Claude가 스스로 검증하는지 확인
3. `docs/progress/` 메모로 세션을 바꿔 가며 작업 이어가기
4. 팀 토론: 우리 저장소에서 바꿔야 할 구조 3가지 선정
