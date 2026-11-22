# 예제: 사내 제조 도구 개발 키트 `mfg-tool-kit` — MCP 표준 위에 얹는 custom SDK (Java)

> 관련 모듈: 13 (MCP), 16 (사내 프레임워크), 18 (Agent SDK) · 관련 회차: [5회차 2-5 제조 현장 MCP](../sessions/session-05-hooks-and-mcp.md#2-5-사례-제조-현장의-ai-에이전트와-mcp-meserp-연결), [8회차 2-3 조직의 도구 표준](../sessions/session-08-automation-sdk-governance.md#2-3-조직의-도구-표준-제조-도구-개발-키트를-만들어-그-위에서-개발한다)
> **목표**: 여러 팀이 제조 도구(MCP 서버)를 만들 때 **같은 규칙**을 지키도록, 표준 MCP Java SDK 위에 얇은 사내 키트를 만들고 그 키트로 생산팀 도구 서버를 만들어 실제로 호출해 봅니다. [예제 34](./34-manufacturing-ai-agent-mcp.md)의 `ToolSupport`(팀이 복사해 쓰던 도우미)를 **회사 라이브러리**로 키운 모습입니다.

---

## 1. 키트가 보장하는 것

| # | 표준 | 키트가 하는 일 | 팀이 할 일 |
|---|---|---|---|
| ① | 도구 이름 `<시스템>_<동사>_<대상>` | 규칙에 안 맞으면 **도구를 만드는 순간 실패**. 읽기 전용 도구는 `get·list·find·search·count`만 | 이름 짓기 |
| ② | 설명 4요소 (무엇을 · 언제 · 단위·한도 · 읽기 전용) | `returns`·`whenToUse`가 없으면 실패. 설명 문장을 회사 표준 순서로 조립 | 두 문장 쓰기 |
| ③ | 공통 입력 타입 | `MfgParams.lineId()` 등 형식(정규식·범위)이 같은 스키마 | 골라 쓰기 |
| ④ | 읽기 전용 표시 | MCP `readOnlyHint` 자동 | — |
| ⑤ | 결과 형식 | `{ "data": …, "meta": { tool, source, asOf(+09:00), units, rows, truncated } }` | 데이터만 돌려주기 |
| ⑥ | 결과 건수 한도 | `maxRows`(기본 200)를 넘으면 잘라 내고 `truncated: true` | 한도 정하기 |
| ⑦ | 마스킹 | `mask("materialCost")` → 어느 깊이에 있든 `"***"` | 가릴 필드 지정 |
| ⑧ | 오류 형식 | `{ "error": { code, message, retryable } }` + `isError`. 예상 못 한 예외는 **내부 정보를 숨기고** `INTERNAL_ERROR` | 업무 오류는 `MfgToolException` |
| ⑨ | 감사 로그 | 호출마다 사용자·도구·인자·건수·소요 시간 (JSON Lines, `MFG_AUDIT_LOG`) | — |
| ⑩ | 서버 이름·안내 | `MfgServer.stdio("mes-tools", …)`: 이름 규칙 `<시스템>-tools` 검사 | 한 줄 |

> 💡 **표준을 다시 만들지 않습니다.** 통신(MCP)·입력 스키마 검사·서버 실행은 **MCP Java SDK가 그대로** 합니다(아래 실행 결과의 `input validation failed`는 SDK가 한 검사). 키트는 그 위에 **회사 규칙만** 얹습니다. SDK 버전이 오르면 키트 담당(플랫폼 팀)이 먼저 시험하고 키트 버전을 올립니다.

---

## 2. 구성

```text
mfg/
├── mfg-tool-kit/                ← 플랫폼 팀 소유 (좌표 com.example.mfg:mfg-tool-kit:0.1.0)
│   ├── build.gradle.kts         ← api("io.modelcontextprotocol.sdk:mcp:2.1.0")
│   └── src/main/java/com/example/mfg/toolkit/
│       ├── MfgTool.java         ← 규칙을 담은 도구 빌더 (핵심)
│       ├── MfgParams.java, MfgParam.java, MfgArgs.java   ← 공통 입력 타입과 인자
│       ├── MfgToolException.java                         ← 업무 오류 (code, retryable)
│       ├── AuditLog.java                                 ← 감사 로그
│       └── MfgServer.java                                ← 서버 시작 한 줄
└── mes-line-tools/              ← 생산팀의 도구 서버 (키트를 의존)
    ├── settings.gradle.kts      ← includeBuild("../mfg-tool-kit")
    └── src/main/java/com/example/mes/tools/
        ├── MesLineTools.java    ← 도구 3개: 팀은 "무엇을, 언제, 어디서"만 쓴다
        ├── MesData.java         ← 교육용 메모리 데이터 (실제로는 예제 34의 MesDb)
        └── MesLineToolsServer.java
```

- 키트를 팀에 나눠 주는 방법은 7회차와 같습니다: 같은 저장소 + `includeBuild`, 공유 폴더 Maven 저장소, 사내 Maven 저장소 ([예제 35의 2장](./35-corp-common-library.md#2-maven-저장소-없이-공유하는-방법))
- 키트와 함께 **스킬(`mfg-tool`)과 CLAUDE.md 규칙**을 플러그인으로 배포하면, 각 팀이 Claude Code로 도구를 만들 때도 키트를 쓰게 됩니다 (8회차 2-3 ④)

---

## 3. 팀 쪽 코드: 도구 하나가 이렇게 짧아진다

예제 34의 `get_line_status`는 스키마 JSON, 결과 만들기, 예외 처리를 팀이 직접 썼습니다. 키트를 쓰면 팀이 쓰는 것은 아래가 전부입니다.

```java
MfgTool.readOnly("mes_get_line_status")
        .title("생산 라인 현황")
        .returns("라인 하나의 오늘 양품·불량 수량, 불량률, 비가동 시간")
        .whenToUse("'지금 1라인 어때?', '오늘 2라인 불량률은?' 같은 질문")
        .param(MfgParams.lineId())
        .units(Map.of("todayDefectRate", "%", "todayDowntime", "분"))
        .source("MES")
        .handler(args -> mes.lineStatus(args.lineId()))   // 업무 로직만
        .build();
```

---

## 4. 키트 전체 소스 (`mfg-tool-kit`)

**`mfg-tool-kit/settings.gradle.kts`**

```kotlin
// mfg-tool-kit/settings.gradle.kts — 사내 제조 도구 개발 키트 (플랫폼 팀 소유)
rootProject.name = "mfg-tool-kit"
```

**`mfg-tool-kit/build.gradle.kts`**

```kotlin
// mfg-tool-kit/build.gradle.kts — 좌표 com.example.mfg:mfg-tool-kit:0.1.0
plugins {
    `java-library`
}

group = "com.example.mfg"
version = "0.1.0"
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }

repositories { mavenCentral() }

dependencies {
    api("io.modelcontextprotocol.sdk:mcp:2.1.0")      // 표준 MCP Java SDK를 그대로 쓰고, 그 위에 사내 규칙을 얹는다
    testImplementation(platform("org.junit:junit-bom:5.12.2"))
    testImplementation("org.junit.jupiter:junit-jupiter")
    testImplementation("org.assertj:assertj-core:3.27.3")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.test { useJUnitPlatform() }
```

**`mfg-tool-kit/src/main/java/com/example/mfg/toolkit/MfgTool.java`**

```java
package com.example.mfg.toolkit;

import io.modelcontextprotocol.json.McpJsonDefaults;
import io.modelcontextprotocol.json.McpJsonMapper;
import io.modelcontextprotocol.json.TypeRef;
import io.modelcontextprotocol.server.McpServerFeatures.SyncToolSpecification;
import io.modelcontextprotocol.spec.McpSchema.CallToolResult;
import io.modelcontextprotocol.spec.McpSchema.Tool;
import io.modelcontextprotocol.spec.McpSchema.ToolAnnotations;
import java.time.Clock;
import java.time.OffsetDateTime;
import java.time.ZoneId;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.Collection;
import java.util.LinkedHashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.regex.Pattern;

/**
 * 사내 표준대로 도구를 만드는 빌더. 각 팀은 업무 로직(handler)만 쓰고, 나머지는 키트가 맡는다.
 *
 * <pre>
 * SyncToolSpecification spec = MfgTool.readOnly("mes_get_line_status")
 *         .title("생산 라인 현황")
 *         .returns("진행 중 작업지시, 오늘 양품·불량 수량과 불량률, 오늘 비가동 시간")
 *         .whenToUse("'지금 1라인 어때?' 같은 질문")
 *         .param(MfgParams.lineId())
 *         .units(Map.of("downtime", "분", "defectRate", "%"))
 *         .source("MES")
 *         .handler(args -> repo.lineStatus(args.lineId()))
 *         .build();
 * </pre>
 *
 * 키트가 보장하는 것: ① 이름 규칙 ② 설명 4요소 ③ 공통 입력 타입의 스키마 ④ 읽기 전용 표시
 * ⑤ 결과 형식 {data, meta} ⑥ 결과 건수 한도 ⑦ 마스킹 ⑧ 오류 형식 ⑨ 감사 로그
 */
public final class MfgTool {

    /** 이름 규칙: <시스템>_<동사>_<대상>, 소문자·숫자·밑줄 */
    static final Pattern NAME = Pattern.compile("^(mes|qms|erp|wms|eam)_([a-z]+)_[a-z0-9_]+$");
    /** 읽기 전용 도구가 쓸 수 있는 동사 (set·update·delete 같은 이름은 읽기 전용 도구로 만들 수 없다) */
    static final Set<String> READ_VERBS = Set.of("get", "list", "find", "search", "count");
    static final ZoneId PLANT_ZONE = ZoneId.of("Asia/Seoul");   // 회사 표준 시각대
    static final int DEFAULT_MAX_ROWS = 200;

    @FunctionalInterface
    public interface Handler {
        Object handle(MfgArgs args) throws Exception;
    }

    public static Builder readOnly(String name) {
        return new Builder(name);
    }

    public static final class Builder {
        private final String name;
        private String title;
        private String returns;
        private String whenToUse;
        private String source;
        private final List<MfgParam> params = new ArrayList<>();
        private final Map<String, String> units = new LinkedHashMap<>();
        private final Set<String> masked = new LinkedHashSet<>();
        private int maxRows = DEFAULT_MAX_ROWS;
        private Handler handler;
        private McpJsonMapper json = McpJsonDefaults.getMapper();
        private AuditLog audit;
        private Clock clock = Clock.system(PLANT_ZONE);

        private Builder(String name) { this.name = name; }

        public Builder title(String title) { this.title = title; return this; }

        /** 설명 ① 무엇을 돌려주나 */
        public Builder returns(String returns) { this.returns = returns; return this; }

        /** 설명 ② 언제 쓰나 (예시 질문) — 에이전트가 도구를 고르는 기준 */
        public Builder whenToUse(String whenToUse) { this.whenToUse = whenToUse; return this; }

        /** 설명 ③ 단위 (결과의 meta.units에도 들어간다) */
        public Builder units(Map<String, String> units) { this.units.putAll(new java.util.TreeMap<>(units)); return this; }   // 이름순: 설명 문장이 늘 같게

        /** 데이터 출처 시스템 (결과의 meta.source) */
        public Builder source(String source) { this.source = source; return this; }

        public Builder param(MfgParam param) { this.params.add(param); return this; }

        /** 결과 목록의 최대 건수 (넘으면 잘라 내고 meta.truncated=true) */
        public Builder maxRows(int maxRows) { this.maxRows = maxRows; return this; }

        /** 결과에서 가릴 필드 이름 (원가, 개인정보 등). 어느 깊이에 있든 "***"로 바꾼다 */
        public Builder mask(String... fields) { this.masked.addAll(List.of(fields)); return this; }

        public Builder handler(Handler handler) { this.handler = handler; return this; }

        public Builder json(McpJsonMapper json) { this.json = json; return this; }

        public Builder audit(AuditLog audit) { this.audit = audit; return this; }

        Builder clock(Clock clock) { this.clock = clock; return this; }

        /** 규칙을 어기면 서버가 뜨기 전에(도구를 만드는 순간) 실패한다 */
        public SyncToolSpecification build() {
            List<String> problems = new ArrayList<>();
            var m = name == null ? null : NAME.matcher(name);
            if (m == null || !m.matches()) {
                problems.add("이름 '" + name + "'이(가) 규칙 <시스템>_<동사>_<대상>(예: mes_get_line_status)에 맞지 않습니다");
            } else if (!READ_VERBS.contains(m.group(2))) {
                problems.add("읽기 전용 도구의 동사는 " + READ_VERBS + " 중 하나여야 합니다: " + m.group(2));
            }
            if (isBlank(returns)) problems.add("설명 ① returns(무엇을 돌려주나)가 없습니다");
            if (isBlank(whenToUse)) problems.add("설명 ② whenToUse(언제 쓰나)가 없습니다");
            if (isBlank(source)) problems.add("source(출처 시스템)가 없습니다");
            if (handler == null) problems.add("handler가 없습니다");
            if (maxRows < 1 || maxRows > 1000) problems.add("maxRows는 1~1000이어야 합니다");
            if (!problems.isEmpty()) {
                throw new IllegalStateException("mfg-tool-kit 표준 위반 (" + name + "): " + String.join("; ", problems));
            }

            Tool tool = Tool.builder(name, inputSchema())
                    .title(title == null ? name : title)
                    .description(description())
                    .annotations(ToolAnnotations.builder().readOnlyHint(true).openWorldHint(false).build())
                    .build();
            AuditLog log = audit != null ? audit : AuditLog.fromEnv(json);
            return SyncToolSpecification.builder().tool(tool)
                    .callHandler((exchange, request) -> call(request.arguments(), log))
                    .build();
        }

        /** 설명 4요소를 회사 표준 순서로 이어 붙인다 */
        String description() {
            StringBuilder d = new StringBuilder(returns.strip());
            if (!d.toString().endsWith(".")) d.append('.');
            d.append(" 사용: ").append(whenToUse.strip()).append('.');
            if (!units.isEmpty()) {
                d.append(" 단위: ");
                units.forEach((k, v) -> d.append(k).append('=').append(v).append(", "));
                d.setLength(d.length() - 2);
                d.append('.');
            }
            d.append(" 결과는 최대 ").append(maxRows).append("건. 출처: ").append(source).append(". 읽기 전용.");
            return d.toString();
        }

        Map<String, Object> inputSchema() {
            Map<String, Object> props = new LinkedHashMap<>();
            List<String> required = new ArrayList<>();
            for (MfgParam p : params) {
                props.put(p.name(), p.schema());
                if (p.required()) required.add(p.name());
            }
            Map<String, Object> schema = new LinkedHashMap<>();
            schema.put("type", "object");
            schema.put("properties", props);
            if (!required.isEmpty()) schema.put("required", required);
            schema.put("additionalProperties", false);
            return schema;
        }

        CallToolResult call(Map<String, Object> rawArgs, AuditLog log) {
            long start = System.nanoTime();
            Map<String, Object> args = rawArgs == null ? Map.of() : rawArgs;
            String user = userName();
            try {
                Object tree = json.convertValue(handler.handle(new MfgArgs(args)), new TypeRef<Object>() {});
                Map<String, Object> meta = new LinkedHashMap<>();
                meta.put("tool", name);
                meta.put("source", source);
                meta.put("asOf", OffsetDateTime.now(clock).truncatedTo(ChronoUnit.SECONDS).toString());
                if (!units.isEmpty()) meta.put("units", units);
                tree = limitRows(tree, meta);
                tree = mask(tree);
                Map<String, Object> body = new LinkedHashMap<>();
                body.put("data", tree);
                body.put("meta", meta);
                log.write(new AuditLog.Entry(OffsetDateTime.now(clock), user, name, args, true, null,
                        (int) meta.getOrDefault("rows", 1), elapsed(start)));
                return CallToolResult.builder().addTextContent(json.writeValueAsString(body)).build();
            } catch (MfgToolException e) {
                return error(log, user, args, start, e.code(), e.getMessage(), e.retryable());
            } catch (IllegalArgumentException e) {
                return error(log, user, args, start, "INVALID_INPUT", e.getMessage(), false);
            } catch (Exception e) {
                System.err.println("[mfg-tool-kit] " + name + " 실행 오류: " + e);   // 자세한 원인은 서버 로그에만
                return error(log, user, args, start, "INTERNAL_ERROR", "조회 중 오류가 발생했습니다. 잠시 후 다시 시도하세요.", true);
            }
        }

        private CallToolResult error(AuditLog log, String user, Map<String, Object> args, long start,
                                     String code, String message, boolean retryable) {
            log.write(new AuditLog.Entry(OffsetDateTime.now(clock), user, name, args, false, code, 0, elapsed(start)));
            String text;
            try {
                text = json.writeValueAsString(Map.of("error", Map.of("code", code, "message", message, "retryable", retryable)));
            } catch (Exception ignored) {
                text = code + ": " + message;
            }
            return CallToolResult.builder().addTextContent(text).isError(true).build();
        }

        /** 목록이면(또는 {items: [...]}면) maxRows로 자르고 meta에 건수를 남긴다 */
        @SuppressWarnings("unchecked")
        private Object limitRows(Object tree, Map<String, Object> meta) {
            List<Object> list = null;
            Map<String, Object> holder = null;
            if (tree instanceof List<?> l) {
                list = (List<Object>) l;
            } else if (tree instanceof Map<?, ?> map && map.get("items") instanceof List<?> l) {
                holder = new LinkedHashMap<>((Map<String, Object>) map);
                list = (List<Object>) l;
            }
            if (list == null) return tree;
            boolean truncated = list.size() > maxRows;
            List<Object> kept = truncated ? new ArrayList<>(list.subList(0, maxRows)) : list;
            meta.put("rows", kept.size());
            meta.put("truncated", truncated);
            if (holder == null) return kept;
            holder.put("items", kept);
            return holder;
        }

        @SuppressWarnings("unchecked")
        private Object mask(Object node) {
            if (masked.isEmpty()) return node;
            if (node instanceof Map<?, ?> map) {
                Map<String, Object> out = new LinkedHashMap<>();
                ((Map<String, Object>) map).forEach((k, v) -> out.put(k, masked.contains(k) ? "***" : mask(v)));
                return out;
            }
            if (node instanceof Collection<?> c) {
                return c.stream().map(this::mask).toList();
            }
            return node;
        }

        private static String userName() {
            String u = System.getenv("MFG_USER");   // 원격 서버라면 인증된 사용자로 바꾼다
            return u == null || u.isBlank() ? System.getProperty("user.name", "unknown") : u;
        }

        private static long elapsed(long start) { return (System.nanoTime() - start) / 1_000_000; }

        private static boolean isBlank(String s) { return s == null || s.isBlank(); }
    }

    private MfgTool() {}
}
```

**`mfg-tool-kit/src/main/java/com/example/mfg/toolkit/MfgParam.java`**

```java
package com.example.mfg.toolkit;

import java.util.Map;

/** 도구 입력 하나: 이름 + JSON Schema 조각 + 필수 여부. 직접 만들기보다 MfgParams의 공통 타입을 쓴다 */
public record MfgParam(String name, Map<String, Object> schema, boolean required) {

    /** 같은 형식을 선택 입력으로 */
    public MfgParam optional() {
        return new MfgParam(name, schema, false);
    }
}
```

**`mfg-tool-kit/src/main/java/com/example/mfg/toolkit/MfgParams.java`**

```java
package com.example.mfg.toolkit;

import java.util.LinkedHashMap;
import java.util.Map;

/**
 * 회사 공통 입력 타입. 모든 팀이 같은 이름·형식·검사 규칙을 쓰게 한다.
 * (형식을 바꿀 때는 키트 버전을 올리고 변경 기록에 남긴다)
 */
public final class MfgParams {
    private MfgParams() {}

    /** 라인 ID: L + 숫자 1~3자리 (예: L1, L12) */
    public static MfgParam lineId() {
        return string("lineId", "라인 ID (예: L1)", "^L[0-9]{1,3}$");
    }

    /** 작업지시 번호: WO- + 숫자 4~10자리 (예: WO-1001) */
    public static MfgParam workOrderNo() {
        return string("woNo", "작업지시 번호 (예: WO-1001)", "^WO-[0-9]{4,10}$");
    }

    /** 품번: 영문 대문자·숫자·하이픈 3~20자 (예: BRG-6201) */
    public static MfgParam itemCode() {
        return string("itemCode", "품번 (예: BRG-6201)", "^[A-Z0-9-]{3,20}$");
    }

    /** 최근 며칠: 1 ~ maxDays (조회 기간 한도를 도구마다 정한다) */
    public static MfgParam days(int maxDays) {
        return integer("days", "최근 며칠 (1~" + maxDays + ")", 1, maxDays).optional();
    }

    public static MfgParam string(String name, String description, String pattern) {
        Map<String, Object> s = new LinkedHashMap<>();
        s.put("type", "string");
        s.put("description", description);
        s.put("pattern", pattern);
        return new MfgParam(name, s, true);
    }

    public static MfgParam integer(String name, String description, int min, int max) {
        Map<String, Object> s = new LinkedHashMap<>();
        s.put("type", "integer");
        s.put("description", description);
        s.put("minimum", min);
        s.put("maximum", max);
        return new MfgParam(name, s, true);
    }
}
```

**`mfg-tool-kit/src/main/java/com/example/mfg/toolkit/MfgArgs.java`**

```java
package com.example.mfg.toolkit;

import java.util.Map;

/** 도구 인자를 타입에 맞게 꺼낸다. 형식 검사는 MCP SDK가 입력 스키마로 먼저 한다 */
public final class MfgArgs {
    private final Map<String, Object> values;

    MfgArgs(Map<String, Object> values) {
        this.values = values == null ? Map.of() : values;
    }

    public String str(String name) {
        Object v = values.get(name);
        if (v == null) throw new IllegalArgumentException(name + " 값이 필요합니다.");
        return v.toString();
    }

    public int integer(String name, int defaultValue) {
        Object v = values.get(name);
        return v instanceof Number n ? n.intValue() : defaultValue;
    }

    public String lineId() { return str("lineId"); }

    public String workOrderNo() { return str("woNo"); }

    public String itemCode() { return str("itemCode"); }

    Map<String, Object> asMap() { return values; }
}
```

**`mfg-tool-kit/src/main/java/com/example/mfg/toolkit/MfgToolException.java`**

```java
package com.example.mfg.toolkit;

/**
 * 도구의 업무 오류. 에이전트가 "데이터 없음"과 "실패"를 구분하고, 다시 시도할지 판단할 수 있게 한다.
 * 예: throw new MfgToolException("LINE_NOT_FOUND", "라인 L9을(를) 찾을 수 없습니다.", false)
 */
public class MfgToolException extends RuntimeException {
    private final String code;
    private final boolean retryable;

    public MfgToolException(String code, String message, boolean retryable) {
        super(message);
        this.code = code;
        this.retryable = retryable;
    }

    public String code() { return code; }

    public boolean retryable() { return retryable; }
}
```

**`mfg-tool-kit/src/main/java/com/example/mfg/toolkit/AuditLog.java`**

```java
package com.example.mfg.toolkit;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.time.OffsetDateTime;
import java.util.Map;

/** 감사 로그: 누가 어떤 도구를 어떤 인자로 불렀고, 몇 건을 돌려줬는지. 모든 도구 호출마다 키트가 남긴다 */
public interface AuditLog {

    record Entry(OffsetDateTime at, String user, String tool, Map<String, Object> args,
                 boolean ok, String errorCode, int rows, long millis) {}

    void write(Entry entry);

    /**
     * 기본값: 환경 변수 MFG_AUDIT_LOG 파일에 한 줄씩(JSON Lines), 없으면 stderr.
     * stdio MCP 서버에서 stdout은 MCP 통신 전용이므로 절대 쓰지 않는다.
     */
    static AuditLog fromEnv(io.modelcontextprotocol.json.McpJsonMapper json) {
        String file = System.getenv("MFG_AUDIT_LOG");
        return entry -> {
            try {
                String line = json.writeValueAsString(Map.of(
                        "at", entry.at().toString(), "user", entry.user(), "tool", entry.tool(), "args", entry.args(),
                        "ok", entry.ok(), "errorCode", entry.errorCode() == null ? "" : entry.errorCode(),
                        "rows", entry.rows(), "millis", entry.millis()));
                if (file == null || file.isBlank()) {
                    System.err.println("[audit] " + line);
                } else {
                    Files.writeString(Path.of(file), line + System.lineSeparator(), StandardCharsets.UTF_8,
                            StandardOpenOption.CREATE, StandardOpenOption.APPEND);
                }
            } catch (IOException e) {
                System.err.println("[audit] 기록 실패: " + e.getMessage());
            }
        };
    }
}
```

**`mfg-tool-kit/src/main/java/com/example/mfg/toolkit/MfgServer.java`**

```java
package com.example.mfg.toolkit;

import io.modelcontextprotocol.json.McpJsonDefaults;
import io.modelcontextprotocol.server.McpServer;
import io.modelcontextprotocol.server.McpServerFeatures.SyncToolSpecification;
import io.modelcontextprotocol.server.McpSyncServer;
import io.modelcontextprotocol.server.transport.StdioServerTransportProvider;
import io.modelcontextprotocol.spec.McpSchema.ServerCapabilities;
import java.util.List;

/** 도구 서버 시작을 한 줄로: 이름 규칙(<시스템>-tools)과 안내 문구 형식을 키트가 맞춘다 */
public final class MfgServer {
    private MfgServer() {}

    /** stdio 서버를 띄운다. stdout은 MCP 통신 전용이므로 도구 코드에서 System.out을 쓰지 않는다 */
    public static McpSyncServer stdio(String name, String version, String instructions, List<SyncToolSpecification> tools) {
        if (!name.matches("^(mes|qms|erp|wms|eam)(-[a-z0-9]+)*-tools$")) {
            throw new IllegalStateException("서버 이름은 <시스템>-tools 형식이어야 합니다 (예: mes-tools): " + name);
        }
        var json = McpJsonDefaults.getMapper();
        McpSyncServer server = McpServer.sync(new StdioServerTransportProvider(json))
                .serverInfo(name, version)
                .instructions(instructions + " 모든 도구는 읽기 전용이며 결과는 {data, meta(source, asOf, units)} 형식이다.")
                .capabilities(ServerCapabilities.builder().tools(true).build())
                .tools(tools)
                .build();
        System.err.println(name + " " + version + " 시작 (도구 " + tools.size() + "개, mfg-tool-kit)");
        Runtime.getRuntime().addShutdownHook(new Thread(server::closeGracefully));
        return server;
    }
}
```

### 키트 테스트

**`mfg-tool-kit/src/test/java/com/example/mfg/toolkit/MfgToolTest.java`**

```java
package com.example.mfg.toolkit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import io.modelcontextprotocol.json.McpJsonDefaults;
import io.modelcontextprotocol.json.McpJsonMapper;
import io.modelcontextprotocol.server.McpServerFeatures.SyncToolSpecification;
import io.modelcontextprotocol.spec.McpSchema.CallToolRequest;
import io.modelcontextprotocol.spec.McpSchema.CallToolResult;
import io.modelcontextprotocol.spec.McpSchema.TextContent;
import java.time.Clock;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.stream.IntStream;
import org.junit.jupiter.api.Test;

class MfgToolTest {
    private final McpJsonMapper json = McpJsonDefaults.getMapper();
    private final List<AuditLog.Entry> audit = new ArrayList<>();
    private final Clock clock = Clock.fixed(Instant.parse("2026-11-22T00:00:00Z"), MfgTool.PLANT_ZONE);

    private MfgTool.Builder lineStatus() {
        return MfgTool.readOnly("mes_get_line_status")
                .returns("라인의 오늘 양품·불량 수량")
                .whenToUse("'지금 1라인 어때?' 같은 질문")
                .param(MfgParams.lineId())
                .units(Map.of("defectRate", "%"))
                .source("MES")
                .audit(audit::add)
                .clock(clock);
    }

    private Map<String, Object> call(SyncToolSpecification spec, Map<String, Object> args) throws Exception {
        CallToolResult r = spec.callHandler().apply(null, CallToolRequest.builder(spec.tool().name()).arguments(args).build());
        String text = ((TextContent) r.content().get(0)).text();
        Map<String, Object> body = json.readValue(text, new io.modelcontextprotocol.json.TypeRef<Map<String, Object>>() {});
        body.put("_isError", Boolean.TRUE.equals(r.isError()));
        return body;
    }

    @Test
    void 이름_규칙과_설명_4요소를_어기면_만드는_순간_실패() {
        assertThatThrownBy(() -> MfgTool.readOnly("getLineStatus").returns("x").whenToUse("y").source("MES").handler(a -> 1).build())
                .isInstanceOf(IllegalStateException.class).hasMessageContaining("<시스템>_<동사>_<대상>");
        assertThatThrownBy(() -> MfgTool.readOnly("mes_update_line").returns("x").whenToUse("y").source("MES").handler(a -> 1).build())
                .hasMessageContaining("읽기 전용 도구의 동사");
        assertThatThrownBy(() -> MfgTool.readOnly("mes_get_line_status").source("MES").handler(a -> 1).build())
                .hasMessageContaining("returns").hasMessageContaining("whenToUse");
    }

    @Test
    void 설명과_스키마와_읽기_전용_표시를_표준대로_만든다() {
        SyncToolSpecification spec = lineStatus().handler(a -> Map.of()).build();
        assertThat(spec.tool().description())
                .isEqualTo("라인의 오늘 양품·불량 수량. 사용: '지금 1라인 어때?' 같은 질문. 단위: defectRate=%. 결과는 최대 200건. 출처: MES. 읽기 전용.");
        assertThat(spec.tool().annotations().readOnlyHint()).isTrue();
        assertThat(spec.tool().inputSchema()).containsEntry("required", List.of("lineId"))
                .containsEntry("additionalProperties", false).containsKey("properties");
    }

    @Test
    @SuppressWarnings("unchecked")
    void 결과는_data와_meta로_감싸고_감사_로그를_남긴다() throws Exception {
        SyncToolSpecification spec = lineStatus().handler(a -> Map.of("lineId", a.lineId(), "goodQty", 631)).build();
        Map<String, Object> body = call(spec, Map.of("lineId", "L1"));
        assertThat((Map<String, Object>) body.get("data")).containsEntry("goodQty", 631);
        Map<String, Object> meta = (Map<String, Object>) body.get("meta");
        assertThat(meta).containsEntry("source", "MES").containsEntry("asOf", "2026-11-22T09:00+09:00")
                .containsEntry("tool", "mes_get_line_status");
        assertThat(audit).singleElement().satisfies(e -> {
            assertThat(e.tool()).isEqualTo("mes_get_line_status");
            assertThat(e.args()).containsEntry("lineId", "L1");
            assertThat(e.ok()).isTrue();
        });
    }

    @Test
    @SuppressWarnings("unchecked")
    void 목록은_maxRows로_자르고_표시한다() throws Exception {
        SyncToolSpecification spec = MfgTool.readOnly("qms_list_defects").returns("불량 목록").whenToUse("불량 질문")
                .source("QMS").maxRows(3).audit(audit::add).clock(clock)
                .handler(a -> Map.of("items", IntStream.range(0, 10).boxed().toList())).build();
        Map<String, Object> body = call(spec, Map.of());
        assertThat((List<Object>) ((Map<String, Object>) body.get("data")).get("items")).hasSize(3);
        assertThat((Map<String, Object>) body.get("meta")).containsEntry("rows", 3).containsEntry("truncated", true);
        assertThat(audit.get(0).rows()).isEqualTo(3);
    }

    @Test
    @SuppressWarnings("unchecked")
    void 지정한_필드는_어느_깊이에_있든_가린다() throws Exception {
        record Material(String itemCode, int onHand, double unitCost) {}
        SyncToolSpecification spec = MfgTool.readOnly("erp_get_material").returns("자재 재고").whenToUse("재고 질문")
                .source("ERP").mask("unitCost").audit(audit::add).clock(clock)
                .handler(a -> Map.of("items", List.of(new Material("BRG-6201", 900, 1250.5)))).build();
        Map<String, Object> body = call(spec, Map.of());
        var item = ((List<Map<String, Object>>) ((Map<String, Object>) body.get("data")).get("items")).get(0);
        assertThat(item).containsEntry("itemCode", "BRG-6201").containsEntry("onHand", 900).containsEntry("unitCost", "***");
    }

    @Test
    @SuppressWarnings("unchecked")
    void 오류는_코드와_재시도_여부로() throws Exception {
        SyncToolSpecification notFound = lineStatus()
                .handler(a -> { throw new MfgToolException("LINE_NOT_FOUND", "라인 L9을(를) 찾을 수 없습니다.", false); }).build();
        Map<String, Object> e1 = call(notFound, Map.of("lineId", "L9"));
        assertThat(e1.get("_isError")).isEqualTo(true);
        assertThat((Map<String, Object>) e1.get("error")).containsEntry("code", "LINE_NOT_FOUND").containsEntry("retryable", false);

        SyncToolSpecification broken = lineStatus().handler(a -> { throw new java.sql.SQLException("connection refused (secret-host:3306)"); }).build();
        Map<String, Object> e2 = call(broken, Map.of("lineId", "L1"));
        Map<String, Object> err = (Map<String, Object>) e2.get("error");
        assertThat(err).containsEntry("code", "INTERNAL_ERROR").containsEntry("retryable", true);
        assertThat(err.get("message").toString()).doesNotContain("secret-host");   // 내부 정보는 에이전트에게 보내지 않는다
        assertThat(audit).extracting(AuditLog.Entry::errorCode).containsExactly("LINE_NOT_FOUND", "INTERNAL_ERROR");
    }
}
```

---

## 5. 생산팀 도구 서버 (`mes-line-tools`)

**`mes-line-tools/settings.gradle.kts`**

```kotlin
// mes-line-tools/settings.gradle.kts — 생산팀의 도구 서버. 사내 키트를 함께 빌드 (사내 저장소가 생기면 includeBuild 대신 저장소 주소)
rootProject.name = "mes-line-tools"
includeBuild("../mfg-tool-kit")
```

**`mes-line-tools/build.gradle.kts`**

```kotlin
plugins {
    java
}

java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }

repositories { mavenCentral() }

dependencies {
    implementation("com.example.mfg:mfg-tool-kit:0.1.0")   // 사내 키트 (MCP Java SDK를 함께 가져온다)
}

// java -jar 로 바로 실행되는 실행 JAR (build/libs/mes-line-tools.jar)
tasks.jar {
    archiveFileName = "mes-line-tools.jar"
    manifest { attributes["Main-Class"] = "com.example.mes.tools.MesLineToolsServer" }
    dependsOn(configurations.runtimeClasspath)                    // 키트 jar가 먼저 만들어지도록
    from({ configurations.runtimeClasspath.get().map { if (it.isDirectory) it else zipTree(it) } })
    exclude("META-INF/*.SF", "META-INF/*.DSA", "META-INF/*.RSA")
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}

dependencies {
    testImplementation(platform("org.junit:junit-bom:5.12.2"))
    testImplementation("org.junit.jupiter:junit-jupiter")
    testImplementation("com.tngtech.archunit:archunit-junit5:1.3.0")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}
tasks.test { useJUnitPlatform() }
```

**`mes-line-tools/src/main/java/com/example/mes/tools/MesLineTools.java`**

```java
package com.example.mes.tools;

import com.example.mfg.toolkit.MfgParams;
import com.example.mfg.toolkit.MfgTool;
import io.modelcontextprotocol.server.McpServerFeatures.SyncToolSpecification;
import java.util.List;
import java.util.Map;

/** 생산팀의 MES 도구 3개. 팀이 쓰는 것은 "무엇을, 언제, 어디서 가져오나"뿐이고 규칙은 키트가 맡는다 */
final class MesLineTools {
    private final MesData mes;

    MesLineTools(MesData mes) { this.mes = mes; }

    List<SyncToolSpecification> all() {
        return List.of(lineStatus(), defects(), workOrder());
    }

    private SyncToolSpecification lineStatus() {
        return MfgTool.readOnly("mes_get_line_status")
                .title("생산 라인 현황")
                .returns("라인 하나의 오늘 양품·불량 수량, 불량률, 비가동 시간")
                .whenToUse("'지금 1라인 어때?', '오늘 2라인 불량률은?' 같은 질문")
                .param(MfgParams.lineId())
                .units(Map.of("todayDefectRate", "%", "todayDowntime", "분"))
                .source("MES")
                .handler(args -> mes.lineStatus(args.lineId()))
                .build();
    }

    private SyncToolSpecification defects() {
        return MfgTool.readOnly("mes_list_defects")
                .title("불량 유형별 수량")
                .returns("라인의 최근 불량을 유형별 수량이 많은 순으로")
                .whenToUse("'1라인 불량이 왜 늘었어?', '가장 많은 불량은?' 같은 질문")
                .param(MfgParams.lineId())
                .param(MfgParams.days(7))
                .units(Map.of("qty", "개"))
                .source("MES")
                .maxRows(5)                                        // 상위 5개만 (에이전트에게 긴 목록은 독)
                .handler(args -> Map.of("lineId", args.lineId(), "days", args.integer("days", 1),
                        "items", mes.defects(args.lineId(), args.integer("days", 1))))
                .build();
    }

    private SyncToolSpecification workOrder() {
        return MfgTool.readOnly("mes_get_work_order")
                .title("작업지시 진척")
                .returns("작업지시의 계획·양품 수량, 진척률, 납기")
                .whenToUse("'WO-1001 납기 맞출 수 있어?' 같은 질문")
                .param(MfgParams.workOrderNo())
                .units(Map.of("progress", "%"))
                .source("MES")
                .mask("materialCost")                              // 원가는 에이전트에게 보내지 않는다 (사내 보안 등급)
                .handler(args -> mes.workOrder(args.workOrderNo()))
                .build();
    }
}
```

**`mes-line-tools/src/main/java/com/example/mes/tools/MesData.java`**

```java
package com.example.mes.tools;

import com.example.mfg.toolkit.MfgToolException;
import java.time.LocalDate;
import java.util.List;
import java.util.Map;

/**
 * 교육용 MES 데이터 (메모리). 실제로는 예제 34의 MesDb처럼 MES 복제본을 읽기 전용 계정으로 조회한다.
 * 도구 코드는 이 클래스의 메서드만 부르므로, 바꿔 끼워도 도구 정의는 그대로다.
 */
final class MesData {
    record LineStatus(String lineId, String lineName, int todayGoodQty, int todayDefectQty, double todayDefectRate, int todayDowntime) {}
    record Defect(String defectCode, String defectName, int qty) {}
    record WorkOrder(String woNo, String itemCode, int plannedQty, int goodQty, double progress, LocalDate dueDate, long materialCost) {}

    private static final Map<String, LineStatus> LINES = Map.of(
            "L1", new LineStatus("L1", "조립 1라인", 631, 20, 3.1, 40),
            "L2", new LineStatus("L2", "조립 2라인", 702, 8, 1.1, 0));

    private static final List<Defect> L1_DEFECTS = List.of(
            new Defect("D-SCR", "나사 체결 불량", 15), new Defect("D-SCT", "외관 스크래치", 3),
            new Defect("D-LBL", "라벨 오부착", 1), new Defect("D-GAP", "조립 틈새", 1),
            new Defect("D-NOI", "이음", 1), new Defect("D-ETC", "기타", 1));

    LineStatus lineStatus(String lineId) {
        LineStatus s = LINES.get(lineId);
        if (s == null) throw new MfgToolException("LINE_NOT_FOUND", "라인 " + lineId + "을(를) 찾을 수 없습니다.", false);
        return s;
    }

    List<Defect> defects(String lineId, int days) {
        lineStatus(lineId);                                    // 없는 라인이면 같은 오류
        return "L1".equals(lineId) ? L1_DEFECTS : List.of();
    }

    WorkOrder workOrder(String woNo) {
        if (!"WO-1001".equals(woNo)) throw new MfgToolException("WORK_ORDER_NOT_FOUND", "작업지시 " + woNo + "을(를) 찾을 수 없습니다.", false);
        return new WorkOrder("WO-1001", "PUMP-A100", 1200, 631, 52.6, LocalDate.of(2026, 11, 24), 18_400_000L);
    }
}
```

**`mes-line-tools/src/main/java/com/example/mes/tools/MesLineToolsServer.java`**

```java
package com.example.mes.tools;

import com.example.mfg.toolkit.MfgServer;

/** 생산팀 MES 도구 서버 (stdio). 서버 시작도 키트가 맞춰 준다 */
public final class MesLineToolsServer {
    public static void main(String[] args) {
        MfgServer.stdio("mes-tools", "1.1.0",
                "평택공장 MES의 라인 현황·불량·작업지시를 조회한다.",
                new MesLineTools(new MesData()).all());
    }
}
```

### 표준을 강제하는 테스트 (7회차 장치 ⑤)

**`mes-line-tools/src/test/java/com/example/mes/tools/ToolKitRulesTest.java`**

```java
package com.example.mes.tools;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

/** 사내 표준 강제: 도구는 mfg-tool-kit(MfgTool)으로만 만든다. MCP SDK의 Tool·CallToolResult를 직접 다루면 실패 */
@AnalyzeClasses(packages = "com.example.mes.tools", importOptions = ImportOption.DoNotIncludeTests.class)
class ToolKitRulesTest {

    @ArchTest
    static final ArchRule tools_are_built_with_the_kit =
        noClasses().should().dependOnClassesThat()
                   .haveFullyQualifiedName("io.modelcontextprotocol.spec.McpSchema$Tool")
                   .orShould().dependOnClassesThat()
                   .haveFullyQualifiedName("io.modelcontextprotocol.spec.McpSchema$CallToolResult")
                   .because("도구는 MfgTool로 만든다 (이름·설명·결과 형식·감사 로그를 키트가 보장)");

    @ArchTest
    static final ArchRule no_stdout =
        noClasses().should().accessField(System.class, "out")
                   .because("stdio MCP 서버에서 stdout은 MCP 통신 전용 (로그는 System.err)");
}
```

---

## 6. 빌드와 실행

```bash
cd mfg/mfg-tool-kit && gradle test          # 키트 테스트 6개
cd ../mes-line-tools && gradle test jar      # 표준 강제 테스트 2개 + build/libs/mes-line-tools.jar
export MFG_AUDIT_LOG=$PWD/audit.jsonl MFG_USER=kim.op
java -Dstdout.encoding=UTF-8 -cp build/libs/mes-line-tools.jar scripts/SmokeTest.java
```

```powershell
# Windows PowerShell
$env:MFG_AUDIT_LOG = "$PWD\audit.jsonl"; $env:MFG_USER = "kim.op"
java -Dstdout.encoding=UTF-8 -cp build/libs/mes-line-tools.jar scripts/SmokeTest.java
```

<details>
<summary>scripts/SmokeTest.java (MCP Java SDK 클라이언트로 서버를 띄워 호출)</summary>

```java
// scripts/SmokeTest.java — MCP 클라이언트로 서버를 띄워 도구 목록·설명을 보고 호출해 본다
// 실행: java -cp build/libs/mes-line-tools.jar scripts/SmokeTest.java
import io.modelcontextprotocol.client.McpClient;
import io.modelcontextprotocol.client.McpSyncClient;
import io.modelcontextprotocol.client.transport.ServerParameters;
import io.modelcontextprotocol.client.transport.StdioClientTransport;
import io.modelcontextprotocol.json.McpJsonDefaults;
import io.modelcontextprotocol.spec.McpSchema.CallToolRequest;
import io.modelcontextprotocol.spec.McpSchema.CallToolResult;
import io.modelcontextprotocol.spec.McpSchema.TextContent;
import java.util.Map;

public class SmokeTest {
    public static void main(String[] args) {
        ServerParameters server = ServerParameters.builder("java")
                .args("-Dstderr.encoding=UTF-8", "-jar", "build/libs/mes-line-tools.jar")
                .env(System.getenv())
                .build();
        McpSyncClient client = McpClient.sync(new StdioClientTransport(server, McpJsonDefaults.getMapper())).build();
        try {
            client.initialize();
            client.listTools().tools().forEach(t -> System.out.println("도구 " + t.name() + " (readOnly=" + t.annotations().readOnlyHint() + ")\n  " + t.description()));
            call(client, "mes_get_line_status", Map.of("lineId", "L1"));
            call(client, "mes_list_defects", Map.of("lineId", "L1", "days", 7));
            call(client, "mes_get_work_order", Map.of("woNo", "WO-1001"));
            call(client, "mes_get_line_status", Map.of("lineId", "L9"));
            call(client, "mes_get_line_status", Map.of("lineId", "1라인"));
            call(client, "mes_list_defects", Map.of("lineId", "L1", "days", 99));
        } finally {
            client.closeGracefully();
        }
    }

    private static void call(McpSyncClient client, String name, Map<String, Object> arguments) {
        CallToolResult r = client.callTool(CallToolRequest.builder(name).arguments(arguments).build());
        String text = ((TextContent) r.content().get(0)).text();
        System.out.println("\n▶ " + name + " " + arguments + (Boolean.TRUE.equals(r.isError()) ? " → 오류" : " → 성공") + "\n  " + text);
    }
}
```

</details>

Claude Code에 등록하는 방법은 예제 34와 같습니다 ([7장](./34-manufacturing-ai-agent-mcp.md#7-claude-code또는-다른-mcp-클라이언트에-등록하기)). 키트가 이름 규칙을 지키게 하므로 권한 규칙도 `mcp__<서버>__<시스템>_…` 모양으로 일정해집니다.

```json
{
  "mcpServers": {
    "mes-tools": {
      "type": "stdio",
      "command": "java",
      "args": ["-Dstderr.encoding=UTF-8", "-jar", "tools/mes-line-tools/build/libs/mes-line-tools.jar"],
      "env": { "MFG_AUDIT_LOG": "${MFG_AUDIT_LOG}" }
    }
  }
}
```

```json
{
  "permissions": {
    "allow": ["mcp__mes-tools__mes_get_line_status", "mcp__mes-tools__mes_list_defects", "mcp__mes-tools__mes_get_work_order"]
  }
}
```

---

## 7. 실제 실행 결과

**MCP 클라이언트로 호출** (2026-11-22, 교재 환경)

```text
도구 mes_get_line_status (readOnly=true)
  라인 하나의 오늘 양품·불량 수량, 불량률, 비가동 시간. 사용: '지금 1라인 어때?', '오늘 2라인 불량률은?' 같은 질문. 단위: todayDefectRate=%, todayDowntime=분. 결과는 최대 200건. 출처: MES. 읽기 전용.
도구 mes_list_defects (readOnly=true)
  라인의 최근 불량을 유형별 수량이 많은 순으로. 사용: '1라인 불량이 왜 늘었어?', '가장 많은 불량은?' 같은 질문. 단위: qty=개. 결과는 최대 5건. 출처: MES. 읽기 전용.
도구 mes_get_work_order (readOnly=true)
  작업지시의 계획·양품 수량, 진척률, 납기. 사용: 'WO-1001 납기 맞출 수 있어?' 같은 질문. 단위: progress=%. 결과는 최대 200건. 출처: MES. 읽기 전용.

▶ mes_get_line_status {lineId=L1} → 성공
  {"data":{"lineId":"L1","lineName":"조립 1라인","todayGoodQty":631,"todayDefectQty":20,"todayDefectRate":3.1,"todayDowntime":40},"meta":{"tool":"mes_get_line_status","source":"MES","asOf":"2026-11-22T21:50:51+09:00","units":{"todayDefectRate":"%","todayDowntime":"분"}}}

▶ mes_list_defects {lineId=L1, days=7} → 성공
  {"data":{"days":7,"lineId":"L1","items":[{"defectCode":"D-SCR","defectName":"나사 체결 불량","qty":15},{"defectCode":"D-SCT","defectName":"외관 스크래치","qty":3},{"defectCode":"D-LBL","defectName":"라벨 오부착","qty":1},{"defectCode":"D-GAP","defectName":"조립 틈새","qty":1},{"defectCode":"D-NOI","defectName":"이음","qty":1}]},"meta":{"tool":"mes_list_defects","source":"MES","asOf":"2026-11-22T21:50:51+09:00","units":{"qty":"개"},"rows":5,"truncated":true}}

▶ mes_get_work_order {woNo=WO-1001} → 성공
  {"data":{"woNo":"WO-1001","itemCode":"PUMP-A100","plannedQty":1200,"goodQty":631,"progress":52.6,"dueDate":"2026-11-24","materialCost":"***"},"meta":{"tool":"mes_get_work_order","source":"MES","asOf":"2026-11-22T21:50:51+09:00","units":{"progress":"%"}}}

▶ mes_get_line_status {lineId=L9} → 오류
  {"error":{"message":"라인 L9을(를) 찾을 수 없습니다.","code":"LINE_NOT_FOUND","retryable":false}}

▶ mes_get_line_status {lineId=1라인} → 오류
  Tool (mes_get_line_status) input validation failed: Validation failed: JSON schema validation errors: [/lineId: does not match the regex pattern ^L[0-9]{1,3}$]

▶ mes_list_defects {lineId=L1, days=99} → 오류
  Tool (mes_list_defects) input validation failed: Validation failed: JSON schema validation errors: [/days: must have a maximum value of 7]
```

| 확인 | 결과 |
|---|---|
| 설명 | 세 도구 모두 같은 순서: 무엇을 → 사용 → 단위 → 최대 건수 → 출처 → 읽기 전용 |
| 결과 형식 | 모두 `{data, meta}`, `asOf`는 `+09:00` |
| 건수 한도 | `mes_list_defects`: 불량 6종 중 5개만, `rows: 5, truncated: true` |
| 마스킹 | `materialCost: "***"` |
| 업무 오류 | 없는 라인 → `LINE_NOT_FOUND`, `retryable: false` |
| 입력 검사 | `1라인`(형식), `days: 99`(범위) → **SDK가 실행 전에 거부** (공통 입력 타입의 스키마) |

**감사 로그** (`audit.jsonl`의 처음 4줄)

```json
{"at":"2026-11-22T21:50:51.153817201+09:00","errorCode":"","millis":5,"tool":"mes_get_line_status","rows":1,"user":"kim.op","ok":true,"args":{"lineId":"L1"}}
{"at":"2026-11-22T21:50:51.228563313+09:00","errorCode":"","millis":5,"tool":"mes_list_defects","rows":5,"user":"kim.op","ok":true,"args":{"lineId":"L1","days":7}}
{"at":"2026-11-22T21:50:51.238679470+09:00","errorCode":"","millis":3,"tool":"mes_get_work_order","rows":1,"user":"kim.op","ok":true,"args":{"woNo":"WO-1001"}}
{"at":"2026-11-22T21:50:51.243223721+09:00","errorCode":"LINE_NOT_FOUND","millis":0,"tool":"mes_get_line_status","rows":0,"user":"kim.op","ok":false,"args":{"lineId":"L9"}}
```

> ⚠️ SDK가 입력 검사에서 거부한 호출은 도구 코드까지 오지 않으므로 이 감사 로그에는 남지 않습니다. 거부된 시도까지 기록해야 하면 서버 전송 계층의 로그나 원격 서버의 접근 로그로 보완합니다.

**표준을 어긴 도구는 만들어지지 않는다** (팀이 `updateLineSpeed`라는 도구를 만들려고 할 때)

```text
mfg-tool-kit 표준 위반 (updateLineSpeed): 이름 'updateLineSpeed'이(가) 규칙 <시스템>_<동사>_<대상>(예: mes_get_line_status)에 맞지 않습니다; 설명 ② whenToUse(언제 쓰나)가 없습니다; source(출처 시스템)가 없습니다
```

**표준 강제 테스트**: MCP SDK의 `CallToolResult`를 직접 쓰고 `System.out`에 출력하는 클래스를 일부러 넣으면 `ToolKitRulesTest`의 두 규칙이 모두 실패했고(`was violated`), 그 클래스를 지우면 통과했습니다.

> ✅ 교재 환경(Linux, Gradle 8.14, Java 21, MCP Java SDK 2.1.0)에서 위 결과를 확인했습니다: 키트 테스트 6개, 표준 강제 테스트 2개, 실제 stdio MCP 연결로 도구 목록·호출. ⚠️ Claude Code에 등록해 대화로 호출하는 것과 Windows에서의 실행은 이 예제에서 다시 해 보지 않았습니다 (같은 방식의 예제 34에서는 등록 절차를 안내).

---

## 8. 키트를 운영할 때

| 항목 | 할 일 |
|---|---|
| **버전** | 공통 입력 타입·결과 형식을 바꾸면 키트 버전을 올리고 변경 기록. 결과 형식 변경은 에이전트 프롬프트를 깨뜨릴 수 있으므로 **큰 변경** |
| **SDK 업데이트** | MCP SDK 새 버전은 키트에서 먼저 시험 (이 예제를 만들 때도 `Tool.builder()` 등 일부 API가 이미 deprecated였다) |
| **쓰기 도구** | 키트에는 읽기 전용(`readOnly`)만 있다. 쓰기가 필요하면 `draft`(초안 만들기 → 사람 승인) 같은 별도 빌더를 **거버넌스 승인 후** 추가 |
| **사용자별 권한** | stdio 서버는 실행한 사람 권한. 여러 사람이 쓰면 원격 서버 + 인증으로 바꾸고, 키트의 `MFG_USER` 대신 인증된 사용자를 감사 로그에 |
| **평가** | 도구가 늘면 "질문 → 올바른 도구를 고르는가" 평가 질문 세트를 CI에 (8회차 2-3 ②) |
| **처음부터 크게 ✕** | 이 예제의 키트도 도구 3개에서 겹친 부분만 담았다. 필요가 생길 때 늘린다 |
