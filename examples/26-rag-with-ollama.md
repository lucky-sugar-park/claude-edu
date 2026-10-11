# 심화: Ollama(오픈소스 LLM)로 RAG 구축하기 — Claude Code와 함께

> 관련 모듈: 선택 심화 모듈 A (13 MCP, 16 커스텀 프레임워크, 18 Agent SDK와 연결)
> 예제 프로젝트: [EduHub](./00-sample-project.md) (Spring Boot + PostgreSQL + React)
> ⚠️ 라이브러리·모델 이름과 설정 키는 버전에 따라 바뀝니다. 아래 코드는 **구조를 이해하기 위한 예시**이므로 실제 적용 전에 Spring AI, Ollama 공식 문서로 최신 사용법을 확인하세요. (2026년 10월 작성)

---

## 1. 개념

### 1-1. RAG란

**RAG(Retrieval-Augmented Generation, 검색 증강 생성)**는 LLM이 답하기 전에 **관련 문서를 먼저 검색해서 함께 넣어 주는** 방식입니다.

```
[질문] "신입사원 필수 교육은 언제까지 들어야 하나요?"
   │
   ▼ ① 검색 (Retrieval): 질문과 의미가 비슷한 문서 조각을 찾는다
[사내 교육 규정 3조 2항, FAQ 12번 …]
   │
   ▼ ② 증강 (Augmentation): 찾은 문서를 질문과 함께 프롬프트에 넣는다
   │
   ▼ ③ 생성 (Generation): LLM이 문서를 근거로 답한다
[답] "입사 후 3개월 이내에 이수해야 합니다 (교육 규정 3조 2항)."
```

### 1-2. 구성 요소

| 구성 요소 | 역할 | 예 |
|---|---|---|
| **문서 수집·분할 (Chunking)** | 문서를 검색하기 좋은 크기로 나눔 | 500~1,000 토큰 단위, 일부 겹치게 |
| **임베딩 모델** | 문장을 의미를 담은 숫자 벡터로 변환 | `nomic-embed-text`, `bge-m3`(다국어) |
| **벡터 저장소** | 벡터를 저장하고 비슷한 것을 빠르게 검색 | **pgvector**(PostgreSQL 확장), Chroma, Qdrant |
| **LLM (생성 모델)** | 검색된 문서를 근거로 답변 생성 | Ollama의 Llama, Qwen, Gemma 계열 등 |
| **오케스트레이션** | 위 과정을 연결하는 코드 | **Spring AI**, LangChain4j(Java), LangChain·LlamaIndex(Python) |

### 1-3. Ollama란

- 오픈소스 LLM을 **내 컴퓨터나 사내 서버에서 쉽게 실행**하게 해 주는 도구
- `ollama pull 모델`로 내려받고, 로컬 API(`http://localhost:11434`)로 호출
- 생성 모델과 임베딩 모델을 모두 제공

---

## 2. 왜 오픈소스 LLM + RAG인가

| 이유 | 설명 |
|---|---|
| **데이터가 밖으로 나가지 않음** | 사내 규정, 고객 정보, 소스 코드를 외부 API로 보내지 않고 사내에서 처리 |
| **최신·사내 지식 반영** | 모델을 다시 학습하지 않고 문서만 갱신하면 답변이 바뀜 |
| **근거 제시** | 어떤 문서를 보고 답했는지 출처를 함께 보여줄 수 있음 → 환각 감소, 검증 가능 |
| **비용 예측** | 사용량 과금 대신 서버 비용 (대신 GPU·운영 비용 고려) |
| **폐쇄망 환경** | 인터넷이 막힌 환경에서도 동작 |

**한계도 알아야 합니다**

| 한계 | 대응 |
|---|---|
| 작은 오픈소스 모델은 추론·한국어 품질이 상용 대형 모델보다 낮을 수 있음 | 업무에 맞는 모델 비교 평가, 다국어 모델 선택 |
| GPU 등 하드웨어 필요, 운영 부담 | 작은 모델로 시작, 사용량에 맞춰 확장 |
| 검색이 틀리면 답도 틀림 | 문서 분할·임베딩 모델·검색 개수 튜닝, 평가 세트 운영 |

> 💡 **역할 구분:** 이 문서에서 **Ollama 모델은 "서비스 안에서 답변하는 AI"**이고, **Claude Code는 "그 서비스를 만드는 개발 도구"**입니다. 둘은 경쟁 관계가 아니라 함께 씁니다.

---

## 3. EduHub에 붙이는 RAG: "교육 도우미" 기능

**목표:** 임직원이 교육 규정·과정 설명·FAQ를 자연어로 질문하면 근거와 함께 답하는 기능

```
┌────────── React (enrollment-mfe 또는 새 assistant-mfe) ──────────┐
│  MUI 채팅 화면  ──Axios──▶  POST /api/v1/assistant/ask            │
└────────────────────────────────┬─────────────────────────────────┘
                                 ▼
┌────────────── Spring Boot (EduHub backend) ──────────────────────┐
│  AssistantController → AssistantService (Spring AI ChatClient)   │
│        │ ① 질문 임베딩·유사 문서 검색      │ ③ 답변 생성         │
│        ▼                                   ▼                     │
│  VectorStore (pgvector)               Ollama 생성 모델           │
│        ▲                                                         │
│  IngestionService: 문서 읽기 → 분할 → 임베딩 → 저장 (②)          │
└──────────────────────────────────────────────────────────────────┘
        │                                   │
   PostgreSQL + pgvector              Ollama (localhost:11434)
```

- 이미 쓰는 **PostgreSQL에 pgvector 확장**을 붙이면 새 DB 없이 시작 가능
- Java 팀이므로 **Spring AI**로 구현 (대안: LangChain4j)

---

## 4. 구축 단계

### 4-1. 로컬 환경 (Docker Compose)

```yaml
# docker-compose.rag.yml
services:
  postgres:
    image: pgvector/pgvector:pg16        # pgvector가 포함된 PostgreSQL 이미지
    environment:
      POSTGRES_DB: eduhub
      POSTGRES_USER: eduhub
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports: ["5432:5432"]

  ollama:
    image: ollama/ollama
    ports: ["11434:11434"]
    volumes: ["ollama:/root/.ollama"]
    # GPU 사용 시 런타임 설정 추가 (Ollama 문서 참고)

volumes:
  ollama:
```

```bash
docker compose -f docker-compose.rag.yml up -d

# 모델 내려받기 (예시: 생성 모델 1개 + 다국어 임베딩 모델 1개)
docker compose -f docker-compose.rag.yml exec ollama ollama pull qwen2.5:7b
docker compose -f docker-compose.rag.yml exec ollama ollama pull bge-m3

# 동작 확인
curl http://localhost:11434/api/tags
```

> 모델 선택은 하드웨어와 한국어 품질을 기준으로 **직접 비교 평가**하세요. 임베딩 모델을 바꾸면 벡터 차원이 달라지므로 저장된 벡터를 다시 만들어야 합니다.

### 4-2. 의존성 (Gradle, 예시)

```kotlin
// backend/build.gradle.kts (일부) — 아티팩트 이름과 버전은 Spring AI 문서로 확인
dependencyManagement {
    imports { mavenBom("org.springframework.ai:spring-ai-bom:<버전>") }
}
dependencies {
    implementation("org.springframework.ai:spring-ai-starter-model-ollama")
    implementation("org.springframework.ai:spring-ai-starter-vector-store-pgvector")
    implementation("org.springframework.ai:spring-ai-advisors-vector-store")
}
```

### 4-3. 설정 (`application.yml`, 예시)

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: qwen2.5:7b
          temperature: 0.2          # 규정 답변은 낮게
      embedding:
        options:
          model: bge-m3
    vectorstore:
      pgvector:
        initialize-schema: true     # 개발 환경에서만. 운영은 Flyway로 관리
        dimensions: 1024            # 임베딩 모델의 차원과 일치해야 함
        distance-type: COSINE_DISTANCE
        index-type: HNSW
```

> EduHub는 스키마를 Flyway로 관리하므로(`ddl-auto: validate`), 운영 환경에서는 `initialize-schema: false`로 두고 vector 확장·테이블 생성을 **Flyway 마이그레이션**으로 작성합니다.

### 4-4. 문서 적재 (Ingestion)

```java
@Service
@RequiredArgsConstructor
public class IngestionService {

    private final VectorStore vectorStore;

    /** 교육 규정·FAQ 마크다운 문서를 분할해 벡터 저장소에 적재한다. */
    public int ingest(List<Resource> files) {
        TokenTextSplitter splitter = new TokenTextSplitter();   // 기본 분할 설정 (필요 시 크기 조정)
        List<Document> chunks = new ArrayList<>();

        for (Resource file : files) {
            String text = readText(file);
            Document doc = new Document(text, Map.of(
                    "source", file.getFilename(),          // 출처 표시용
                    "category", "education-policy"));      // 검색 필터용
            chunks.addAll(splitter.apply(List.of(doc)));
        }
        vectorStore.add(chunks);                            // 임베딩 생성 + 저장
        return chunks.size();
    }
}
```

**분할 요령**
- 조항·질문 단위처럼 **의미 단위**를 지키도록 문서 구조(제목)를 활용
- 출처, 문서 종류, 갱신일 같은 **메타데이터**를 함께 저장 → 출처 표시, 필터링, 최신 문서 우선
- 문서가 바뀌면 해당 문서의 벡터를 지우고 다시 적재

### 4-5. 질의 (Retrieval + Generation)

```java
@Service
public class AssistantService {

    private final ChatClient chatClient;

    public AssistantService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
                .defaultSystem("""
                        너는 EduHub 교육 도우미다. 제공된 문서 내용만 근거로 한국어로 답한다.
                        문서에 없는 내용은 "확인된 규정이 없습니다"라고 답하고 추측하지 않는다.
                        답변 끝에 근거 문서 이름을 표시한다.
                        """)
                .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
                        .searchRequest(SearchRequest.builder()
                                .topK(4)                     // 가져올 문서 조각 수
                                .similarityThreshold(0.6)    // 너무 관련 없는 조각 제외
                                .build())
                        .build())
                .build();
    }

    public String ask(String question) {
        return chatClient.prompt().user(question).call().content();
    }
}
```

```java
@RestController
@RequestMapping("/api/v1/assistant")
@RequiredArgsConstructor
@Tag(name = "Assistant", description = "교육 도우미 API")
public class AssistantController {

    private final AssistantService assistantService;

    @Operation(summary = "교육 관련 질문에 답변")
    @PostMapping("/ask")
    public AskResponse ask(@Valid @RequestBody AskRequest request) {
        return new AskResponse(assistantService.ask(request.question()));
    }
}
```

> API 이름(`QuestionAnswerAdvisor`, `SearchRequest` 빌더 등)은 Spring AI 버전에 따라 달라질 수 있습니다. 응답 스트리밍, 출처 목록을 별도 필드로 돌려주기 등은 공식 문서의 RAG 가이드를 참고하세요.

### 4-6. 프론트엔드 (React + MUI + Axios)

```tsx
export default function AssistantChat() {
  const [question, setQuestion] = useState("");
  const [answer, setAnswer] = useState<string>();
  const [loading, setLoading] = useState(false);

  const ask = async () => {
    setLoading(true);
    try {
      const { data } = await assistantApi.ask({ question });   // OpenAPI로 생성한 클라이언트
      setAnswer(data.answer);
    } finally {
      setLoading(false);
    }
  };

  return (
    <Stack spacing={2}>
      <TextField label="교육 관련 질문" value={question} onChange={(e) => setQuestion(e.target.value)} />
      <Button variant="contained" onClick={ask} disabled={!question || loading}>질문하기</Button>
      {loading && <LinearProgress />}
      {answer && <Alert severity="info" sx={{ whiteSpace: "pre-line" }}>{answer}</Alert>}
    </Stack>
  );
}
```

---

## 5. 품질 평가와 개선

| 단계 | 방법 |
|---|---|
| **평가 세트 만들기** | 실제 질문 30~50개와 정답·근거 문서 목록 |
| **검색 평가** | 정답 근거 문서가 상위 k개 안에 들어오는 비율 |
| **답변 평가** | 정확성, 근거 일치, "모른다"를 제대로 말하는지 |
| **튜닝 순서** | ① 문서 분할 크기·방식 → ② 임베딩 모델 → ③ topK·임계값 → ④ 프롬프트 → ⑤ 생성 모델 |

| 흔한 문제 | 원인 | 개선 |
|---|---|---|
| 엉뚱한 문서를 근거로 답함 | 분할이 너무 크거나 작음, 임베딩 모델이 한국어에 약함 | 의미 단위 분할, 다국어 임베딩 모델 |
| 문서에 있는데 "모른다" | topK가 작거나 임계값이 높음 | topK 늘리기, 임계값 낮추기, 키워드 검색 병행(하이브리드) |
| 문서에 없는 내용을 지어냄 | 프롬프트 제약 부족, 모델 크기 | 시스템 프롬프트 강화, 근거 없으면 답하지 않게 |
| 오래된 규정으로 답함 | 문서 갱신 누락 | 메타데이터에 갱신일, 재적재 파이프라인 |

---

## 6. Claude Code로 RAG를 만드는 방법

### 6-1. 개발 흐름

```text
[계획] /plan EduHub에 Spring AI + Ollama + pgvector 기반 교육 도우미 RAG 기능을 추가하려고 해.
       backend/CLAUDE.md 규칙(DTO, OpenAPI, Flyway)을 따르고, 단계별 구현 계획을 세워줘.

[구현] 1단계: docker-compose.rag.yml과 pgvector Flyway 마이그레이션을 만들어줘.
       2단계: IngestionService와 적재 테스트를 만들어줘. docs/education/*.md를 적재 대상으로 해.
       3단계: AssistantService·Controller와 @WebMvcTest를 만들어줘.
       4단계: API 클라이언트 재생성 후 MUI 채팅 화면을 만들어줘.

[평가] eval/questions.csv의 질문으로 검색 정확도를 측정하는 테스트를 만들고 결과를 표로 보여줘.
```

### 6-2. Claude Code 확장 기능 활용

| 기능 | 활용 |
|---|---|
| **스킬** `rag-pipeline` | 분할 규칙, 메타데이터 규칙, 평가 절차를 표준화 ([스킬](./09-skills-catalog.md)) |
| **서브에이전트** `rag-evaluator` | 평가 세트 실행 → 실패 사례 분석 → 튜닝 제안 (읽기 전용 + 테스트 실행) |
| **훅** | 임베딩 모델 설정이 바뀌면 "재적재 필요" 경고 |
| **아티팩트** | 평가 결과를 차트 리포트로 공유 ([아티팩트](./04-mcp-plugins-artifacts.md#모듈-15-아티팩트)) |

### 6-3. 거꾸로: 사내 RAG를 Claude Code의 도구로 쓰기

만든 RAG 검색을 **MCP 서버**로 감싸면, Claude Code가 개발 중에 **사내 문서(사내 프레임워크 가이드, 코딩 표준, 장애 사례)**를 검색할 수 있습니다.

```
Claude Code ──MCP──▶ eduhub-docs MCP 서버 ──▶ pgvector 검색 (사내 문서)
```

```ts
// 개념 예시: 검색 결과(문서 조각)만 돌려주고, 답변 생성은 Claude가 한다
server.tool(
  "search_internal_docs",
  "사내 프레임워크 가이드, 코딩 표준, 장애 사례 문서를 의미 기반으로 검색한다",
  { query: z.string(), topK: z.number().default(5) },
  async ({ query, topK }) => {
    const res = await fetch(`${RAG_URL}/api/v1/docs/search`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ query, topK }),
    });
    return { content: [{ type: "text", text: await res.text() }] };
  },
);
```

- [사내 커스텀 프레임워크 적용](./07-custom-framework.md)의 "④ 문서·소스 접근"을 RAG로 구현하는 방법
- 문서 원문은 사내에 두고, **필요한 조각만** Claude에게 전달 → 컨텍스트 절약
- 어떤 문서를 외부 모델(Claude)에 보낼 수 있는지 **사내 정책으로 범위 지정**

---

## 7. 운영·보안 체크리스트

- [ ] 문서 접근 권한: 사용자 권한에 따라 검색 대상 필터링 (부서·직급 메타데이터)
- [ ] 개인정보·민감정보 문서는 적재 전에 제외 또는 마스킹
- [ ] 프롬프트 인젝션: 문서 안의 "지시문"을 따르지 않도록 시스템 프롬프트로 제한
- [ ] 문서 갱신 시 자동 재적재 (배치 또는 이벤트)
- [ ] 질문·답변 로그와 피드백(좋아요/싫어요) 수집 → 평가 세트 보강
- [ ] Ollama 서버 접근 제한 (사내망, 인증 프록시)
- [ ] 모델·임베딩 버전을 기록하고 바꿀 때 평가 세트로 회귀 테스트

---

## 8. 실습

1. Docker Compose로 pgvector와 Ollama를 띄우고 모델 2개(생성, 임베딩)를 내려받기
2. Claude Code에게 계획 → 단계별 구현을 맡겨 "교육 도우미" API 완성
3. 사내 교육 규정 예시 문서 10개를 적재하고 질문 20개로 평가
4. 분할 크기, topK, 임베딩 모델을 바꿔 가며 정확도 비교 → 결과를 아티팩트로 공유
5. (심화) RAG 검색을 MCP 서버로 만들어 Claude Code에 연결하고, "사내 프레임워크 가이드에 따라 구현해줘" 지시 시 검색 도구를 쓰는지 확인
