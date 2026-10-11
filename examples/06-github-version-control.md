# 예제: GitHub 연결과 버전관리 모범사례

> 관련 모듈: 7 (17, 18과 연결)
> 예제 프로젝트: [EduHub](./00-sample-project.md)

---

## 1. Claude Code와 GitHub를 연결하는 방법

| 환경 | 연결 방법 | 할 수 있는 일 |
|---|---|---|
| **터미널, IDE, 데스크톱(로컬)** | 로컬의 `git` 인증(SSH 키, 자격 증명) 그대로 사용. GitHub 작업은 `gh` CLI 설치 후 `gh auth login` | 커밋, 푸시, `gh`로 PR 생성·이슈 조회 |
| **웹(클라우드)** | claude.ai에서 GitHub 계정 연결 + 저장소에 Claude GitHub App 설치 | 클라우드 세션이 저장소를 클론하고 브랜치에 푸시, PR 생성 |
| **GitHub MCP 서버** | `.mcp.json` 또는 `claude mcp add`로 연결 ([예제](./04-mcp-plugins-artifacts.md)) | 이슈·PR·리뷰 코멘트를 도구로 조회·작성 |
| **GitHub Actions** | 터미널에서 `/install-github-app` 실행(앱 설치와 워크플로우 생성 안내), 또는 워크플로우를 직접 작성 | PR 자동 리뷰, 이슈에서 `@claude` 멘션으로 작업 지시 ([예제](./05-automation-and-capstone.md)) |

> 명령 이름과 설치 절차는 바뀔 수 있으므로 교육 전에 공식 문서로 확인하세요.

**보안 원칙**
- 토큰과 GitHub App 권한은 **필요한 저장소, 필요한 권한만** 부여한다
- Actions에서 API 키는 반드시 `secrets`로 관리한다
- 외부인이 작성한 이슈나 PR 내용도 Claude에게는 "입력"이다. 공개 저장소에서는 누가 Claude를 호출할 수 있는지 제한한다

---

## 2. 버전관리 모범사례 (AI 에이전트와 함께 일할 때)

### 2-1. 브랜치 전략: GitHub Flow

```
main (보호 브랜치, 항상 배포 가능)
 ├── feature/42-cancel-enrollment
 ├── fix/57-n-plus-one-course-list
 └── chore/update-spring-boot-3.4
```

- `main`에 직접 커밋하지 않는다. **작업 하나 = 브랜치 하나 = PR 하나**
- 브랜치 이름에 이슈 번호를 넣어 추적한다
- Claude에게 작업을 맡길 때도 "브랜치를 먼저 만들고 시작해줘"를 습관화한다

### 2-2. 커밋

| 원칙 | 이유 | Claude에게 줄 지시 |
|---|---|---|
| **작고 의미 있는 단위로** | 리뷰와 되돌리기가 쉽다 | "백엔드와 프론트를 별도 커밋으로 나눠줘" |
| **Conventional Commits** | 릴리스 노트 자동화, 변경 성격 파악 | CLAUDE.md에 형식 명시: `feat(enrollment): ...` |
| **커밋 전 검증** | 깨진 커밋이 남지 않는다 | "테스트와 spotlessApply 통과 후 커밋해줘" |
| **AI 참여 표시** | 리뷰어가 AI 작성 코드임을 안다 | Claude Code가 기본으로 넣는 `Co-Authored-By` 줄을 유지 |
| **비밀정보 커밋 금지** | 한 번 푸시되면 이력에 남는다 | `.gitignore`에 `.env`, `application-local.yml` + 권한 설정으로 읽기 차단 |

### 2-3. Pull Request

- PR 템플릿(`.github/pull_request_template.md`)을 두면 Claude가 그 형식에 맞춰 본문을 작성한다

```markdown
## 변경 내용
## API 변경 (Swagger 경로)
## 테스트 방법
## 스크린샷 (화면 변경 시)
## 체크리스트
- [ ] 백엔드 테스트 통과
- [ ] API 클라이언트 재생성
- [ ] Flyway 마이그레이션 추가 시 리뷰어 지정
```

- PR은 **리뷰 가능한 크기**로: 변경 400줄 이내를 목표로, 크면 Claude에게 나눠 달라고 한다
- AI가 만든 PR도 **사람의 리뷰가 필수**. Claude의 자동 리뷰는 보조 수단이다

### 2-4. 저장소 보호 설정

| 설정 | 목적 |
|---|---|
| 보호 브랜치(`main`): 직접 푸시 금지, 강제 푸시 금지 | 에이전트의 실수로부터 `main` 보호 |
| 머지 전 필수 체크: 빌드, 테스트, 린트 | 검증되지 않은 코드 차단 |
| 필수 리뷰 1명 이상 | AI 코드도 사람이 확인 |
| `CODEOWNERS` | `db/migration/`은 DBA, `packages/api-client`는 프론트 리드가 리뷰 |

```
# .github/CODEOWNERS
/backend/src/main/resources/db/migration/  @eduhub/dba
/frontend/packages/                       @eduhub/frontend-leads
/.claude/                                 @eduhub/platform
```

> 💡 `.claude/` 설정(권한, 훅, 스킬)도 팀 전체에 영향을 주므로 코드 오너를 지정해 리뷰를 받습니다.

### 2-5. 되돌리기 전략

| 상황 | 방법 |
|---|---|
| 방금 Claude가 한 수정이 마음에 들지 않음 (커밋 전) | Claude Code의 체크포인트·되돌리기(rewind) |
| 커밋했지만 푸시 전 | `git reset`, `git commit --amend` (내 브랜치에서만) |
| 이미 푸시되어 공유됨 | `git revert`로 되돌리는 커밋 추가. 공유 브랜치에서 이력 재작성 금지 |

---

## 3. 실습: 이슈부터 머지까지

```text
GitHub 이슈 #42를 읽고 처리해줘.
1. main에서 feature/42-cancel-enrollment 브랜치를 만들어줘
2. 구현 후 테스트를 통과시키고, 백엔드·프론트를 별도 커밋으로 나눠줘 (Conventional Commits)
3. PR 템플릿에 맞춰 PR을 만들고 이슈 #42를 연결해줘 (Closes #42)
```

1. Claude가 만든 PR을 동료가 리뷰하고 코멘트 남기기
2. "리뷰 코멘트를 반영해줘"로 수정 요청 → 추가 커밋 확인
3. CI 통과, 리뷰 승인 후 머지 (Squash merge 권장)
