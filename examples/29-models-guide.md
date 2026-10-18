# 모델 가이드: Claude Code에서 쓰는 모델, 특징, 토큰 사용, 선택 모범 사례

> 관련 모듈: 1, 11, 19 (1회차 6-2와 연결)
> 기준: Claude Code 공식 문서 [Model configuration](https://code.claude.com/docs/en/model-config), [Manage costs](https://code.claude.com/docs/en/costs), Anthropic 모델 가격표 (2026년 10월 확인)
> ⚠️ 모델 라인업과 가격은 자주 바뀝니다. 교육 전에 `/model` 선택 화면과 공식 문서로 최신 내용을 확인하세요.

---

## 1. Claude 모델 한눈에 보기

| 별칭 | 모델 | 이런 일에 | 입력 / 출력 (100만 토큰당) | Sonnet 대비 | 기본 추론 수준 |
|---|---|---|---|---|---|
| `fable` | **Fable 5.1** | 가장 어려운 일: 장시간 자율 작업, 원인 불명 장애의 근본 원인 분석, 아키텍처 결정 같은 모호한 문제 | $10 / $50 | 5배 | high |
| `opus` | **Opus 5.5** | 복잡한 설계, 여러 단계 추론, 위험 영역 검토 | $4 / $20 | 2배 | medium |
| `sonnet` | **Sonnet 5.5** | 대부분의 일상 코딩: 구현, 버그 수정, 테스트, 리뷰 | $2 / $10 | 1배 | medium |
| `haiku` | **Haiku 5.5** | 빠르고 저렴한 일: 탐색, 요약, 분류, 단순 반복 | $0.10 / $0.50 (10만 토큰 이하) | 약 1/20 | medium |

**특수 별칭**

| 별칭 | 의미 |
|---|---|
| `opusplan` | Plan 모드에서는 Opus로 계획하고, 실행은 Sonnet으로 자동 전환 → "설계는 똑똑하게, 구현은 효율적으로"를 설정 하나로 |
| `best` | Fable을 쓸 수 있으면 Fable, 아니면 Opus |
| `default` | 계정의 기본 모델로 되돌림 |
| `sonnet[1m]`, `opus[1m]` | 100만 토큰 컨텍스트 지정 (현재 세대는 이미 기본 100만 토큰이라 효과 없음) |

**알아둘 점**
- **기본 모델**: Pro, Max, Team, Enterprise, API 계정은 **Opus 5.5**가 기본입니다. Fable은 기본값이 되지 않으며 직접 골라야 합니다 (사용량 크레딧으로 과금될 수 있어 처음 쓸 때 확인 창이 뜸)
- **컨텍스트**: 네 모델 모두 기본 100만 토큰. 자동 압축은 약 96만 7천 토큰 근처에서 일어납니다 (`/autocompact`로 조정 가능)
- **Haiku 5.5**: 10만 토큰을 넘는 요청은 단가가 높아집니다 ($0.50 / $2.50)
- **안전 대체(fallback)**: Fable, Opus 5.5, Sonnet 5.5는 보안·생물 관련으로 분류된 요청을 다른 모델로 다시 실행하거나 거절할 수 있습니다
- **클라우드 제공사**: Bedrock, Foundry 등에서는 별칭이 이전 버전을 가리킬 수 있습니다. 버전을 고정하려면 전체 모델 ID(예: `claude-opus-5-5`)를 씁니다

---

## 2. 토큰 사용량을 결정하는 요소

```
비용(또는 요금제 한도 소모) ≈ 모델 단가 × (입력 토큰 + 출력 토큰)
```

| 요소 | 영향 |
|---|---|
| **모델 단가** | 같은 작업도 Opus는 Sonnet의 2배, Fable은 5배, Haiku는 약 1/20 |
| **추론 수준 (effort)** | 높을수록 생각 토큰(출력으로 과금)과 도구 호출이 늘어납니다. `low`는 도구 호출이 적고 답이 간결합니다 |
| **컨텍스트 크기** | 매 요청마다 대화 전체를 보내므로 긴 세션일수록 비쌉니다 ([토큰 최적화](./08-token-optimization.md)) |
| **프롬프트 캐시** | 반복되는 앞부분은 싸게 처리되지만, **캐시는 모델별로 따로**입니다. 세션 중간에 모델을 바꾸면 대화 전체를 다시 처리합니다 |
| **재작업** | 싼 모델이 틀려서 다시 시키면 총비용이 오히려 늘어납니다 |

### 추론 수준 (effort)

| 수준 | 공식 문서 가이드 |
|---|---|
| `low` | 빠르게 주고받고 사람이 검토하는 작업 |
| `medium` | 범위가 분명한 일상 작업. Opus 5.5의 출발점 (Opus 5.5 `medium`이 이전 세대 Opus 5 `high`와 같거나 낫다고 설명) |
| `high` | 검증이나 예외 상황을 꼼꼼히 다뤄야 하는 작업 |
| `xhigh` | 더 깊은 추론, 더 높은 비용 |
| `max` | 정말 어려운 문제. 과하게 생각하고 효과가 줄어들 수 있음 |

설정: `/effort medium`, `/model` 화면의 좌우 화살표, `claude --effort high`, 한 턴만 깊게 생각시키려면 프롬프트에 `ultrathink`

---

## 3. ChatGPT(OpenAI) 진영 모델

| 질문 | 답 |
|---|---|
| Claude Code에서 GPT 모델을 직접 쓸 수 있나? | **아니요.** Claude Code는 Claude 모델을 실행합니다 |
| 그럼 함께 쓰려면? | ① **Codex CLI**를 따로 쓰거나 ② Claude Code 안에서 **OpenAI 공식 플러그인 codex-plugin-cc**로 Codex를 호출 (`/codex:review`, `/codex:adversarial-review`, `/codex:rescue --model …`) ([함께 쓰기](./24-codex-with-claude-code.md)) |
| 어떤 GPT 모델이 있나? | 2026년 10월 기준 Codex는 **주력(GPT-6.1 Sol 등) · 경량(Luna 등) · 코딩 특화(codex 계열)** 모델로 나뉘어 있고, Codex CLI 기본 모델이 GPT-6.1 Sol로 바뀌었다는 보도가 있습니다. 다만 OpenAI 공식 자료로 직접 확인하지 못했고 이름·가격이 자주 바뀌므로 **Codex의 모델 선택 화면과 OpenAI 공식 문서로 확인**하세요 |
| 비용은? | ChatGPT 요금제로 로그인하면 요금제 한도 안에서, API 키로 쓰면 토큰당 과금 |

**GPT 모델을 함께 쓰는 대표 경우**: 다른 회사 모델의 관점으로 **교차 리뷰**, 설계에 대한 **반론 받기**, Claude가 막힌 버그를 **다른 모델로 재시도**

---

## 4. 어느 경우에 어떤 모델? (모범 사례)

### 4-1. 작업별 추천

| 작업 | 추천 모델 | 추론 수준 | 이유 |
|---|---|---|---|
| 원인 불명 장애, 큰 아키텍처 결정, 장시간 자율 작업 | Fable 5.1 (꼭 필요할 때) 또는 Opus | high | 틀리면 비용이 가장 큰 판단 |
| 설계명세, 구현계획, 복잡한 리팩터링 설계 | **Opus** 또는 `opusplan` | medium~high | 넓은 맥락과 깊은 추론 |
| 보안, DB 마이그레이션, 동시성 검토 | **Opus** | high | 놓치면 운영 사고 |
| 일반 구현, 원인이 분명한 버그 수정, 테스트 작성 | **Sonnet** | medium | 계획이 구체적이고 테스트로 검증 가능 |
| 일반 코드 리뷰 | **Sonnet** (위험 영역만 Opus) | medium~high | 체크리스트 중심 검토 |
| 코드 탐색, 호출 흐름 찾기, 로그·문서 요약, PR 본문 | **Haiku** (서브에이전트) | low~medium | 정답이 분명하고 요약만 돌려받음 |
| CI 자동화 (PR 요약, 실패 원인 분석, 릴리스 노트) | Haiku 또는 Sonnet | low~medium | 대량·반복 실행 |
| 교차 리뷰, 설계 반론 | Claude + **Codex(GPT)** | — | 다른 모델의 맹점 보완 |

### 4-2. 구성 패턴

| 패턴 | 구성 | 적합한 상황 |
|---|---|---|
| **단일 모델** | 메인 `sonnet` | 개인 일상 개발, 비용 우선 |
| **opusplan** | Plan 모드 Opus → 실행 Sonnet | 설정 하나로 설계/구현 분리 |
| **메인 고성능 + 서브에이전트 분업** | 메인 Opus (설계·판단), `implementer`·`reviewer` Sonnet, `code-explorer` Haiku, `critical-reviewer` Opus | 설계 → 계획 → 구현 → 검토 자동 진행 ([설정 예시](./18-java-react-agents.md)) |
| **교차 모델 리뷰** | Claude 구현 + Codex 리뷰 | 중요한 PR, 품질 우선 |

### 4-3. 모범 사례

1. **기본값 확인**: 기본이 Opus라는 것을 모르고 모든 작업을 Opus로 하는 경우가 많습니다. 공식 문서도 예상보다 비용이 큰 경우는 대개 "정리하지 않은 긴 세션"과 "기본 모델을 Opus로 둔 것"이라고 설명합니다
2. **추론 수준부터 조절**: 모델을 바꾸기 전에 같은 모델에서 `/effort`를 낮춰 봅니다. 상위 모델의 낮은 추론 수준이 하위 모델보다 나은 경우가 많습니다
3. **역할별 모델 명시**: 서브에이전트 정의에 `model:`을 적어 두면 메인 모델을 바꿔도 서브에이전트 비용이 흔들리지 않습니다 (명시하지 않으면 메인 모델을 따라감)
4. **세션 중간 모델 전환 최소화**: 캐시가 모델별로 따로라 전환할 때마다 대화 전체를 다시 처리합니다. 작업 단위로(`/clear` 직후) 바꿉니다
5. **완료당 비용으로 판단**: "요청 1건"이 아니라 "기능 1건 완료"에 든 사용량과 재작업으로 비교합니다
6. **측정 후 결정**: 대표 작업 5~10개로 모델 조합을 비교합니다 (테스트 첫 통과율, 사람 리뷰에서 추가로 나온 문제 수, 완료당 사용량)
7. **조직 관리**: 관리자는 `availableModels`(쓸 수 있는 모델 제한), `maxEffortLevel`(추론 수준 상한)로 비용을 통제할 수 있습니다

---

## 5. 설정 방법

| 무엇을 | 방법 |
|---|---|
| 세션 모델 | `/model sonnet` (Enter: 기본값으로 저장, `s`: 이 세션만) |
| 시작할 때 | `claude --model opusplan` |
| 항상 쓰는 기본값 | 설정 파일 `"model": "sonnet"` 또는 환경 변수 `ANTHROPIC_MODEL` |
| 추론 수준 | `/effort medium`, `claude --effort high`, `CLAUDE_CODE_EFFORT_LEVEL` |
| 서브에이전트 | 정의 파일 머리말 `model: haiku` / `sonnet` / `opus` / `inherit` |
| 모든 서브에이전트 기본값 | 환경 변수 `CLAUDE_CODE_SUBAGENT_MODEL=sonnet` |
| 과부하 시 대체 모델 | `claude --fallback-model sonnet,haiku` 또는 설정 `"fallbackModel"` |
| 스킬 실행 중에만 | 스킬 머리말 `model:` (그 턴 동안만 적용) |

**우선순위 (세션 모델)**: `/model` → `--model` → `ANTHROPIC_MODEL` → 설정 파일 `model` → 계정 기본값

---

## 6. 실습

1. `/model`로 현재 기본 모델과 추론 수준을 확인하고, 일상 작업용으로 `sonnet` 또는 `opusplan`으로 바꿔 보기
2. 같은 과제("과정 목록 N+1 수정")를 Opus `medium`, Sonnet `medium`, Sonnet `high`로 각각 수행하고 `/usage`로 사용량, 테스트 통과 여부, 수정 횟수 비교
3. `code-explorer`를 Haiku로, `implementer`를 Sonnet으로 정의하고 `/usage`의 서브에이전트 사용 비중 확인
4. (선택) codex-plugin-cc로 같은 변경을 Codex에게 리뷰받아 Claude 리뷰 결과와 비교

---

## 참고 자료

- [Claude Code: Model configuration](https://code.claude.com/docs/en/model-config) · [Manage costs effectively](https://code.claude.com/docs/en/costs) · [Sub-agents](https://code.claude.com/docs/en/sub-agents)
- [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc)
- OpenAI 모델 관련 보도 (2차 자료, 공식 확인 필요): [Codex CLI 0.161.0 기본 모델 GPT-6.1 Sol](https://sessionwatcher.com/news/codex-cli-0-161-0-gpt-6-1-sol-default-model), [Codex pricing (October 2026)](https://www.aieatingtheworld.com/articles/codex-pricing)
