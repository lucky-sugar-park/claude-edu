# 예제: 설정·권한, 스킬, 서브에이전트, 훅

> 관련 모듈: 9, 10, 11, 12
> 예제 프로젝트: [EduHub](./00-sample-project.md)

---

## 모듈 9. 설정과 권한: `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Bash(./gradlew test *)",
      "Bash(./gradlew spotlessApply)",
      "Bash(./gradlew build)",
      "Bash(npm test *)",
      "Bash(npm run lint)",
      "Bash(npm run generate:api)",
      "Bash(git status)",
      "Bash(git diff *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./backend/src/main/resources/application-prod.yml)",
      "Bash(git push --force*)",
      "Bash(./gradlew flywayClean*)"
    ]
  }
}
```

> 규칙 문법과 우선순위의 자세한 설명은 [권한 가이드](./23-permissions.md)를 참고하세요.

**설계 포인트**
- 자주 쓰고 안전한 명령(테스트, 린트, 포맷)은 `allow`로 승인 없이 실행
- 운영 설정, 비밀정보 파일은 `deny`로 **읽기 자체를 차단**
- DB를 비우는 `flywayClean`, 강제 푸시 같은 되돌릴 수 없는 명령은 차단
- 팀 공용은 `.claude/settings.json`(커밋), 개인 설정은 `.claude/settings.local.json`(커밋하지 않음)

---

## 모듈 10. 스킬: `add-api-endpoint`

백엔드부터 프론트까지 "API 하나 추가"에 필요한 절차를 스킬로 표준화합니다.

```
.claude/skills/add-api-endpoint/
├── SKILL.md
├── templates/
│   ├── Controller.java.md
│   └── ServiceTest.java.md
└── scripts/
    └── check-openapi.sh      # 모든 API에 @Operation이 있는지 검사
```

```markdown
---
name: add-api-endpoint
description: EduHub에 새 REST API를 추가할 때 사용한다. 백엔드 Entity/DTO/Service/Controller와 OpenAPI 문서, 테스트, 프론트 API 클라이언트 재생성, MUI 화면 연결까지 한 번에 처리한다. "API 추가", "엔드포인트 만들어줘", "새 기능의 백엔드와 화면" 같은 요청에 사용한다.
---

# API 엔드포인트 추가 절차

## 1. 백엔드
1. 필요하면 Flyway 마이그레이션 `V{다음번호}__{설명}.sql`을 추가한다
2. 엔티티 변경: 연관관계는 LAZY, `@Setter` 금지
3. 요청/응답 DTO는 `dto` 패키지에 Java record로 만든다. Bean Validation 어노테이션 포함
4. Service: `@Transactional(readOnly = true)` 기본, 쓰기 메서드만 `@Transactional`
5. Controller: `templates/Controller.java.md` 형식을 따른다. `@Operation`, `@ApiResponse` 필수
6. 테스트: `templates/ServiceTest.java.md` 형식으로 성공 1개 + 실패 케이스 작성

## 2. 검증
- `./gradlew spotlessApply test`
- `bash .claude/skills/add-api-endpoint/scripts/check-openapi.sh`

## 3. 프론트엔드
1. 백엔드를 실행한 상태에서 `cd frontend && npm run generate:api`
2. 해당 MFE에 `useXxx` 훅 추가 (생성된 `xxxApi` 사용)
3. MUI 컴포넌트로 화면 연결, 로딩/에러 상태 처리

## 4. 마무리
- Swagger UI에서 새 API 확인 방법을 사용자에게 안내한다
- 백엔드/프론트 커밋을 나눈다
```

**사용 예**

```text
교육과정 후기(리뷰) 기능을 추가해줘. 별점(1~5)과 코멘트를 남기고, 과정 상세 화면에 목록을 보여줘.
```

→ 설명(description)이 요청과 맞으므로 Claude가 스킬을 **자동으로** 불러와 절차대로 진행합니다. `/add-api-endpoint`로 직접 부를 수도 있습니다.

**추가 스킬 아이디어**

| 스킬 | 용도 |
|---|---|
| `new-microfrontend` | 새 remote 앱 생성: webpack 설정, shared 설정, shell 라우팅 등록 |
| `jpa-performance-check` | 쿼리 로그 분석, N+1 탐지, 인덱스 제안 |
| `flyway-migration` | 마이그레이션 번호 규칙, 롤백 스크립트 작성 규칙 |

---

## 모듈 11. 서브에이전트

### `.claude/agents/jpa-reviewer.md`

```markdown
---
name: jpa-reviewer
description: JPA/Hibernate 코드 리뷰 전문가. 엔티티, 리포지토리, 서비스 변경 후 성능과 정합성 문제를 검토할 때 사용한다.
tools: Read, Grep, Glob, Bash
---

너는 JPA와 Hibernate 성능 전문가다. 변경된 백엔드 코드를 다음 관점으로 검토한다.

1. N+1 쿼리 가능성 (목록 조회에서 지연 로딩 연관관계 접근)
2. EAGER 로딩 사용 여부
3. 트랜잭션 범위와 readOnly 설정
4. 엔티티 직접 반환 여부 (DTO 변환 누락)
5. 동시성 문제 (정원, 재고 같은 수량 변경에 락이 있는지)
6. Flyway 마이그레이션과 엔티티 매핑 불일치

결과는 "심각 / 경고 / 제안"으로 나누고, 파일 경로와 줄 번호, 수정 예시 코드를 함께 제시한다.
코드는 직접 수정하지 않는다.
```

### `.claude/agents/mfe-integration-checker.md`

```markdown
---
name: mfe-integration-checker
description: 마이크로프론트엔드 통합 검사. Module Federation 설정, shared 의존성 버전, MUI 테마 사용을 점검할 때 사용한다.
tools: Read, Grep, Glob
---

shell과 모든 remote의 package.json, webpack 설정을 비교해 다음을 보고한다.
- shared singleton 의존성의 버전 불일치
- remote에서 테마를 새로 만드는 코드(createTheme)
- remote끼리 직접 import하는 코드
- Axios를 직접 생성하는 코드 (api-client를 써야 함)
```

**사용 예**

```text
수강 취소 기능 구현이 끝났으면 jpa-reviewer와 mfe-integration-checker로 병렬 검토해줘.
```

→ 각 서브에이전트가 **별도 컨텍스트**에서 검토하고 요약만 돌려주므로, 메인 대화의 컨텍스트가 지저분해지지 않습니다.

---

## 모듈 12. 훅

### 파일을 수정하면 자동 포맷 (PostToolUse)

`.claude/settings.json`에 추가합니다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "bash .claude/hooks/format.sh" }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "bash .claude/hooks/protect-files.sh" }
        ]
      }
    ]
  }
}
```

```bash
# .claude/hooks/format.sh
#!/usr/bin/env bash
# 훅 입력(JSON)은 stdin으로 들어온다
FILE=$(jq -r '.tool_input.file_path // empty')

case "$FILE" in
  *.java)
    (cd backend && ./gradlew spotlessApply -q) ;;
  *.ts|*.tsx)
    npx --prefix frontend prettier --write "$FILE" >/dev/null ;;
esac
exit 0
```

### 수정 금지 파일 보호 (PreToolUse)

```bash
# .claude/hooks/protect-files.sh
#!/usr/bin/env bash
FILE=$(jq -r '.tool_input.file_path // empty')

if [[ "$FILE" == *"/api-client/src/generated/"* ]]; then
  echo "자동 생성 코드는 직접 수정할 수 없습니다. npm run generate:api를 사용하세요." >&2
  exit 2   # 종료 코드 2: 도구 실행을 막고 메시지를 Claude에게 전달
fi

if [[ "$FILE" == *"/db/migration/V"* && -f "$FILE" ]]; then
  echo "기존 Flyway 마이그레이션은 수정할 수 없습니다. 새 버전 파일을 추가하세요." >&2
  exit 2
fi
exit 0
```

**핵심 메시지:** CLAUDE.md의 "생성 코드는 수정하지 않는다"는 **부탁**이고, 이 훅은 **강제**입니다. 두 가지를 함께 쓰면 Claude가 규칙을 미리 알고, 실수해도 막힙니다.
