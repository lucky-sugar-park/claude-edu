# 예제: 사내 공통 모듈 `corp-common` — Maven 저장소 없이 공유하기

> 관련 모듈: 16 (사내 커스텀 프레임워크 적용) · 관련 회차: [7회차 따라하기 ②](../sessions/session-07-cicd-framework-tokens.md#5-따라하기-②-사내-공통-모듈-적용과-강제)
> **목표**: 7회차 실습에 쓰는 가상의 사내 공통 모듈 `corp-common`의 **전체 소스**와, **사내 Maven 저장소(Nexus 등)가 없을 때** 공통 모듈을 여러 프로젝트가 함께 쓰는 방법을 정리합니다.

---

## 1. 무엇이 들어 있나

| 클래스 | 하는 일 | 쓰는 쪽 코드 |
|---|---|---|
| `ApiResponse<T>`, `ErrorBody` | 모든 API의 공통 응답 `{"success":…, "data":…, "error":{"code","message"}}` | `return ApiResponse.ok(data);` |
| `ErrorCode` (인터페이스) | 업무 오류 코드. 도메인마다 enum으로 구현 (이름 규칙: `도메인_의미`) | `enum CourseErrorCode implements ErrorCode` |
| `BizException` | 업무 오류 예외 | `throw new BizException(CourseErrorCode.COURSE_NOT_FOUND);` |
| `CommonErrorCode` | 도메인과 상관없는 공통 오류 (입력값, 로그인, 권한, 내부 오류) | 자동으로 쓰임 |
| `CorpExceptionHandler` | `BizException`·입력값 검증 실패 → `ApiResponse.fail` + 알맞은 HTTP 상태 | **자동 등록** (따로 만들 필요 없음) |
| `CurrentUser` | 로그인 사용자 (사번, 이름, 역할). 요청 본문의 userId를 믿지 않는다 | 컨트롤러 파라미터 `CurrentUser user` |
| `CurrentUserArgumentResolver` | 컨트롤러의 `CurrentUser` 파라미터를 로그인 토큰(JWT)에서 채움 | **자동 등록** |
| `CorpCommonAutoConfiguration` | 의존성만 추가하면 위 두 가지를 켜는 Spring Boot 자동 구성 | — |

- 좌표: **`com.example.corp:corp-common:1.0.0`** (Java 21, Spring Boot 3.5 기준)
- 실제 회사의 공통 모듈은 사내 SSO 연동 필터(`CorpAuthFilter`) 등을 더 갖고 있지만, 여기서는 EduHub의 JWT 인증(5회차)을 그대로 쓰고 `CurrentUser`만 공통 모듈이 제공합니다. 5회차의 `com.example.eduhub.auth.CurrentUser`와 **필드·메서드가 같으므로** import만 바꾸면 됩니다

---

## 2. Maven 저장소 없이 공유하는 방법

사내 Maven 저장소가 없고, 공개 Maven Central에는 올릴 수 없는 경우의 선택지입니다.

| 방법 | 어떻게 | 좋은 점 | 아쉬운 점 | 맞는 상황 |
|---|---|---|---|---|
| **A. 같은 저장소 + composite build** ⭐ | `libs/corp-common/`에 소스를 두고 `backend/settings.gradle.kts`에 `includeBuild("../libs/corp-common")` | 서버·업로드 없음. CI가 그대로 동작. **Claude가 소스를 직접 읽을 수 있다** (6단계 장치 ④). 좌표가 같아 나중에 저장소로 옮겨도 코드 그대로 | 공통 모듈을 쓰는 저장소마다 소스가 들어간다 | **7회차 실습**, 한 제품을 한 저장소에서 만드는 팀 |
| B. 폴더를 Maven 저장소처럼 | 공통 모듈에서 `publish`로 공유 폴더(예: 사내 파일 서버)에 올리고, 쓰는 쪽은 `maven { url = … }` | 서버 프로그램 없이 **버전별 jar**를 여러 저장소·팀이 공유 | 공유 폴더 권한·백업 관리, CI 러너가 그 폴더에 닿아야 함 | 여러 저장소가 같은 공통 모듈을 쓰는 회사 |
| C. `mavenLocal()` | 각자 `publishToMavenLocal` → `~/.m2` | 설정이 가장 짧다 | **각 PC와 CI마다** 먼저 설치해야 함. "내 PC에서는 되는데"가 생긴다 | 혼자 잠깐 시험할 때만 |
| D. jar 파일을 저장소에 커밋 | `libs/corp-common-1.0.0.jar` + `implementation(files(...))` | 가장 단순 | 바이너리가 Git에 쌓임, 버전·소스 추적 어려움, Claude가 소스를 못 봄 | 권하지 않음 |

> 💡 **A와 B는 같은 좌표(`com.example.corp:corp-common:1.0.0`)를 씁니다.** 지금은 A로 시작하고, 여러 저장소가 쓰게 되면 B로, 사내 Maven 저장소가 생기면 `repositories`의 주소만 바꾸면 됩니다. 쓰는 쪽의 `dependencies`와 Java 코드는 바뀌지 않습니다.

> ⚠️ GitHub Packages(Maven)도 서버 없이 쓸 수 있지만, 읽을 때도 토큰이 필요하고 회사 코드를 외부 서비스에 올리는 것이라 사내 정책 확인이 먼저입니다.

---

## 3. 방법 A: composite build로 연결 (7회차 실습 기준)

```text
eduhub/
├── backend/                       ← 기존 Gradle 빌드 (gradlew 여기)
│   ├── settings.gradle.kts        ← includeBuild("../libs/corp-common") 추가
│   └── build.gradle.kts           ← implementation("com.example.corp:corp-common:1.0.0")
└── libs/
    └── corp-common/               ← 독립 Gradle 빌드 (4장의 파일)
```

```kotlin
// backend/settings.gradle.kts (끝에 추가)
// 사내 공통 모듈: Maven 저장소 대신 같은 저장소의 소스를 함께 빌드 (Gradle composite build)
// dependencies의 com.example.corp:corp-common 을 이 빌드의 결과로 자동 대체한다
includeBuild("../libs/corp-common")
```

```kotlin
// backend/build.gradle.kts (dependencies에 추가)
implementation("com.example.corp:corp-common:1.0.0")
```

**확인**

```bash
cd backend
./gradlew dependencies --configuration runtimeClasspath | grep corp-common
#   +--- com.example.corp:corp-common:1.0.0 -> project :corp-common   ← 저장소가 아니라 libs의 소스로 대체됨
./gradlew :corp-common:test      # 공통 모듈 자체의 테스트 (backend 폴더에서 실행)
./gradlew test                   # EduHub 테스트
./gradlew bootJar                # jar 안에 BOOT-INF/lib/corp-common-1.0.0.jar 가 들어간다
```

- `corp-common`에는 `gradlew`가 없어도 됩니다. backend의 `gradlew`가 함께 빌드합니다
- CI(`ci.yml`의 `working-directory: backend`)와 배포 워크플로우도 그대로 동작합니다. 단, **`libs/**`가 바뀌어도 백엔드가 다시 배포되도록** `deploy-backend.yml`의 `paths`에 `libs/**`를 넣습니다

---

## 4. 전체 소스

**`libs/corp-common/settings.gradle.kts`**

```kotlin
// libs/corp-common/settings.gradle.kts — 사내 공통 모듈(가상). 혼자서도 빌드되는 독립 Gradle 빌드
rootProject.name = "corp-common"
```

**`libs/corp-common/build.gradle.kts`**

```kotlin
// libs/corp-common/build.gradle.kts
// 좌표: com.example.corp:corp-common:1.0.0 — 사내 Maven 저장소가 생겨도 쓰는 쪽 코드는 그대로
plugins {
    `java-library`
}

group = "com.example.corp"
version = "1.0.0"
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }

repositories { mavenCentral() }

dependencies {
    // Spring 버전은 쓰는 쪽(EduHub: Spring Boot 3.5)과 맞춘다. compileOnly: 실제 라이브러리는 쓰는 앱이 가져온다
    compileOnly(platform("org.springframework.boot:spring-boot-dependencies:3.5.16"))
    compileOnly("org.springframework.boot:spring-boot-autoconfigure")
    compileOnly("org.springframework:spring-webmvc")
    compileOnly("org.springframework.security:spring-security-oauth2-jose")
    compileOnly("jakarta.servlet:jakarta.servlet-api")
    compileOnly("org.slf4j:slf4j-api")

    testImplementation(platform("org.springframework.boot:spring-boot-dependencies:3.5.16"))
    testImplementation("org.springframework.boot:spring-boot-starter-web")
    testImplementation("org.springframework.boot:spring-boot-starter-validation")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-oauth2-jose")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

tasks.test { useJUnitPlatform() }
```

**`libs/corp-common/src/main/java/com/example/corp/common/api/ApiResponse.java`**

```java
package com.example.corp.common.api;

import com.example.corp.common.error.ErrorCode;

/**
 * 사내 공통 응답. 모든 API는 이것으로 감싸 반환한다.
 * 성공: {"success":true,"data":{...},"error":null}
 * 실패: {"success":false,"data":null,"error":{"code":"COURSE_NOT_FOUND","message":"..."}}
 */
public record ApiResponse<T>(boolean success, T data, ErrorBody error) {

    public static <T> ApiResponse<T> ok(T data) {
        return new ApiResponse<>(true, data, null);
    }

    /** 돌려줄 데이터가 없는 성공 (취소, 삭제 등) */
    public static ApiResponse<Void> ok() {
        return new ApiResponse<>(true, null, null);
    }

    public static ApiResponse<Void> fail(ErrorCode code) {
        return new ApiResponse<>(false, null, new ErrorBody(code.name(), code.message()));
    }

    /** 같은 코드에 상황별 메시지를 줄 때 (예: 입력값 검증 실패 내용) */
    public static ApiResponse<Void> fail(ErrorCode code, String message) {
        return new ApiResponse<>(false, null, new ErrorBody(code.name(), message));
    }
}
```

**`libs/corp-common/src/main/java/com/example/corp/common/api/ErrorBody.java`**

```java
package com.example.corp.common.api;

/** 실패 응답의 error 부분. code는 ErrorCode의 이름 (화면이 분기할 때 쓰는 값) */
public record ErrorBody(String code, String message) {}
```

**`libs/corp-common/src/main/java/com/example/corp/common/error/ErrorCode.java`**

```java
package com.example.corp.common.error;

import org.springframework.http.HttpStatus;

/**
 * 업무 오류 코드. 각 도메인이 enum으로 구현한다 (예: CourseErrorCode.COURSE_NOT_FOUND).
 * 이름 규칙: 도메인_의미 (대문자, 밑줄). HTTP 상태와 사용자에게 보일 메시지를 함께 정한다.
 */
public interface ErrorCode {
    String name();          // enum이면 자동으로 구현된다

    HttpStatus status();

    String message();
}
```

**`libs/corp-common/src/main/java/com/example/corp/common/error/BizException.java`**

```java
package com.example.corp.common.error;

/** 업무 오류. throw new BizException(CourseErrorCode.COURSE_NOT_FOUND) — CorpExceptionHandler가 ApiResponse.fail로 바꾼다 */
public class BizException extends RuntimeException {
    private final ErrorCode code;

    public BizException(ErrorCode code) {
        super(code.message());
        this.code = code;
    }

    public ErrorCode code() {
        return code;
    }
}
```

**`libs/corp-common/src/main/java/com/example/corp/common/error/CommonErrorCode.java`**

```java
package com.example.corp.common.error;

import org.springframework.http.HttpStatus;

/** 도메인과 상관없는 공통 오류 */
public enum CommonErrorCode implements ErrorCode {
    COMMON_INVALID_INPUT(HttpStatus.BAD_REQUEST, "입력값이 올바르지 않습니다."),
    COMMON_UNAUTHORIZED(HttpStatus.UNAUTHORIZED, "로그인이 필요합니다."),
    COMMON_FORBIDDEN(HttpStatus.FORBIDDEN, "권한이 없습니다."),
    COMMON_INTERNAL_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "일시적인 오류가 발생했습니다.");

    private final HttpStatus status;
    private final String message;

    CommonErrorCode(HttpStatus status, String message) {
        this.status = status;
        this.message = message;
    }

    @Override public HttpStatus status() { return status; }

    @Override public String message() { return message; }
}
```

**`libs/corp-common/src/main/java/com/example/corp/common/web/CorpExceptionHandler.java`**

```java
package com.example.corp.common.web;

import com.example.corp.common.api.ApiResponse;
import com.example.corp.common.error.BizException;
import com.example.corp.common.error.CommonErrorCode;
import java.util.stream.Collectors;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

/** BizException과 입력값 검증 실패를 ApiResponse.fail로 바꾼다. 앱이 따로 만들 필요 없음 (자동 등록) */
@RestControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)   // 앱이 같은 예외를 직접 처리하면 앱 쪽이 먼저
public class CorpExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(CorpExceptionHandler.class);

    @ExceptionHandler(BizException.class)
    public ResponseEntity<ApiResponse<Void>> biz(BizException e) {
        log.info("업무 오류 {}: {}", e.code().name(), e.getMessage());   // 업무 오류는 장애가 아니므로 INFO
        return ResponseEntity.status(e.code().status()).body(ApiResponse.fail(e.code()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> invalid(MethodArgumentNotValidException e) {
        String detail = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .collect(Collectors.joining(", "));
        CommonErrorCode code = CommonErrorCode.COMMON_INVALID_INPUT;
        return ResponseEntity.status(code.status()).body(ApiResponse.fail(code, detail));
    }
}
```

**`libs/corp-common/src/main/java/com/example/corp/common/auth/CurrentUser.java`**

```java
package com.example.corp.common.auth;

import org.springframework.security.oauth2.jwt.Jwt;

/**
 * 현재 로그인한 사용자. 사용자 정보는 이것으로만 얻는다 (요청 본문의 userId를 믿지 않는다).
 * 컨트롤러 메서드에 CurrentUser 파라미터를 두면 자동으로 채워진다 (CurrentUserArgumentResolver).
 * 토큰 클레임: sub = 사번(숫자), name = 이름, role = EMPLOYEE | ADMIN
 */
public record CurrentUser(Long employeeId, String name, String role) {

    public static CurrentUser from(Jwt jwt) {
        return new CurrentUser(Long.valueOf(jwt.getSubject()), jwt.getClaimAsString("name"), jwt.getClaimAsString("role"));
    }

    public boolean isAdmin() {
        return "ADMIN".equals(role);
    }
}
```

**`libs/corp-common/src/main/java/com/example/corp/common/auth/CurrentUserArgumentResolver.java`**

```java
package com.example.corp.common.auth;

import com.example.corp.common.error.BizException;
import com.example.corp.common.error.CommonErrorCode;
import org.springframework.core.MethodParameter;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;

/** 컨트롤러의 CurrentUser 파라미터를 로그인 토큰(JWT)에서 채운다. 로그인하지 않았으면 COMMON_UNAUTHORIZED */
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return CurrentUser.class.equals(parameter.getParameterType());
    }

    @Override
    public CurrentUser resolveArgument(MethodParameter parameter, ModelAndViewContainer mav,
                                       NativeWebRequest request, WebDataBinderFactory binderFactory) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getPrincipal() instanceof Jwt jwt) {
            return CurrentUser.from(jwt);
        }
        throw new BizException(CommonErrorCode.COMMON_UNAUTHORIZED);
    }
}
```

**`libs/corp-common/src/main/java/com/example/corp/common/autoconfigure/CorpCommonAutoConfiguration.java`**

```java
package com.example.corp.common.autoconfigure;

import com.example.corp.common.auth.CurrentUserArgumentResolver;
import com.example.corp.common.web.CorpExceptionHandler;
import java.util.List;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnWebApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/** 의존성만 추가하면 예외 처리기와 CurrentUser 파라미터가 자동으로 켜진다 (Spring Boot 자동 구성) */
@AutoConfiguration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
public class CorpCommonAutoConfiguration {

    @Bean
    CorpExceptionHandler corpExceptionHandler() {
        return new CorpExceptionHandler();
    }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnClass(Jwt.class)
    static class CurrentUserConfiguration implements WebMvcConfigurer {
        @Override
        public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
            resolvers.add(new CurrentUserArgumentResolver());
        }
    }
}
```

**`libs/corp-common/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`**

```text
com.example.corp.common.autoconfigure.CorpCommonAutoConfiguration
```

### 테스트

**`libs/corp-common/src/test/java/com/example/corp/common/web/CorpExceptionHandlerTest.java`**

```java
package com.example.corp.common.web;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.jsonPath;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import com.example.corp.common.api.ApiResponse;
import com.example.corp.common.error.BizException;
import com.example.corp.common.error.ErrorCode;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import org.junit.jupiter.api.Test;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.bind.annotation.*;

class CorpExceptionHandlerTest {

    enum SampleErrorCode implements ErrorCode {
        SAMPLE_NOT_FOUND(HttpStatus.NOT_FOUND, "샘플을 찾을 수 없습니다.");
        private final HttpStatus status; private final String message;
        SampleErrorCode(HttpStatus s, String m) { status = s; message = m; }
        public HttpStatus status() { return status; }
        public String message() { return message; }
    }

    record NameRequest(@NotBlank String name) {}

    @RestController
    static class SampleController {
        @GetMapping("/ok") ApiResponse<String> ok() { return ApiResponse.ok("hello"); }
        @GetMapping("/missing") ApiResponse<String> missing() { throw new BizException(SampleErrorCode.SAMPLE_NOT_FOUND); }
        @PostMapping("/names") ApiResponse<String> create(@Valid @RequestBody NameRequest req) { return ApiResponse.ok(req.name()); }
    }

    private final MockMvc mvc = MockMvcBuilders.standaloneSetup(new SampleController())
            .setControllerAdvice(new CorpExceptionHandler()).build();

    @Test
    void 성공은_success_true와_data() throws Exception {
        mvc.perform(get("/ok"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.success").value(true))
                .andExpect(jsonPath("$.data").value("hello"))
                .andExpect(jsonPath("$.error").doesNotExist());
    }

    @Test
    void BizException은_ErrorCode의_상태와_코드로() throws Exception {
        mvc.perform(get("/missing"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.success").value(false))
                .andExpect(jsonPath("$.error.code").value("SAMPLE_NOT_FOUND"))
                .andExpect(jsonPath("$.error.message").value("샘플을 찾을 수 없습니다."));
    }

    @Test
    void 입력값_검증_실패는_COMMON_INVALID_INPUT() throws Exception {
        mvc.perform(post("/names").contentType(MediaType.APPLICATION_JSON).content("{\"name\":\"\"}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.error.code").value("COMMON_INVALID_INPUT"))
                .andExpect(jsonPath("$.error.message").value(org.hamcrest.Matchers.startsWith("name:")));
    }
}
```

**`libs/corp-common/src/test/java/com/example/corp/common/auth/CurrentUserArgumentResolverTest.java`**

```java
package com.example.corp.common.auth;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import com.example.corp.common.error.BizException;
import java.time.Instant;
import java.util.List;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.security.authentication.TestingAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.jwt.Jwt;

class CurrentUserArgumentResolverTest {
    private final CurrentUserArgumentResolver resolver = new CurrentUserArgumentResolver();

    @AfterEach
    void clear() { SecurityContextHolder.clearContext(); }

    @Test
    void 토큰의_클레임으로_CurrentUser를_만든다() {
        Jwt jwt = Jwt.withTokenValue("t").header("alg", "HS256").subject("1001")
                .claim("name", "김교육").claim("role", "ADMIN")
                .issuedAt(Instant.now()).expiresAt(Instant.now().plusSeconds(60)).build();
        SecurityContextHolder.getContext().setAuthentication(new TestingAuthenticationToken(jwt, null, List.of()));

        CurrentUser user = resolver.resolveArgument(null, null, null, null);

        assertThat(user).isEqualTo(new CurrentUser(1001L, "김교육", "ADMIN"));
        assertThat(user.isAdmin()).isTrue();
    }

    @Test
    void 로그인하지_않았으면_COMMON_UNAUTHORIZED() {
        assertThatThrownBy(() -> resolver.resolveArgument(null, null, null, null))
                .isInstanceOf(BizException.class)
                .extracting(e -> ((BizException) e).code().name()).isEqualTo("COMMON_UNAUTHORIZED");
    }
}
```

---

## 5. 방법 B: 공유 폴더를 Maven 저장소로 쓰기 (선택)

여러 저장소가 같은 공통 모듈을 쓰게 되면, 공통 모듈 저장소에서 **버전을 올릴 때마다** 공유 폴더에 jar를 올립니다.

```kotlin
// libs/corp-common/build.gradle.kts — plugins에 `maven-publish`를 추가하고 아래를 덧붙인다
plugins {
    `java-library`
    `maven-publish`
}

// 서버 없이 폴더를 Maven 저장소처럼: publish → <CORP_REPO_DIR>/com/example/corp/corp-common/1.0.0/
java { withSourcesJar() }   // 소스도 함께 (IDE와 Claude가 읽을 수 있게)
publishing {
    publications { create<MavenPublication>("corp") { from(components["java"]) } }
    repositories { maven { name = "corpFolder"; url = uri(providers.environmentVariable("CORP_REPO_DIR").orElse("../maven-repo")) } }
}
```

```powershell
# 공통 모듈 담당자가 버전을 올린 뒤 (공유 폴더 경로는 예)
$env:CORP_REPO_DIR = '\\fileserver\corp-maven'
.\gradlew publish
```

```kotlin
// 쓰는 쪽 build.gradle.kts — settings.gradle.kts의 includeBuild 대신
repositories {
    mavenCentral()
    maven { url = uri("<공유 폴더 경로>") }   // 예: 네트워크 드라이브로 연결한 Z:/corp-maven
}
dependencies { implementation("com.example.corp:corp-common:1.0.0") }
```

- 한 번 올린 버전(`1.0.0`)은 **덮어쓰지 않습니다.** 고치면 `1.0.1`로 올립니다 (다른 팀이 이미 받은 jar와 내용이 달라지면 원인을 찾기 어렵다)
- 개발자 PC와 CI 러너(특히 self-hosted runner)가 모두 그 공유 폴더를 읽을 수 있어야 합니다

---

## 6. 확인한 것

> ✅ 교재 환경(Linux, Gradle 8.14, Java 21, Spring Boot 3.5.16)에서 확인했습니다.
> - `corp-common` 자체 테스트 5개 통과 (성공 응답 모양, `BizException` → 상태 코드·`error.code`, 입력값 검증 → `COMMON_INVALID_INPUT`, 토큰 → `CurrentUser`, 로그인 없음 → `COMMON_UNAUTHORIZED`)
> - 방법 A: EduHub 백엔드에 `includeBuild`로 연결 → 의존성이 `-> project :corp-common`으로 대체됨, `./gradlew :corp-common:test` 실행됨, bootJar에 `corp-common-1.0.0.jar` 포함
> - EduHub의 `CurrentUser`를 corp-common 것으로 바꾸고 컨트롤러 두 개를 전환해 실제 서버로 확인: 로그인 토큰으로 `GET /api/me/enrollments` → `{"success":true,"data":[…]}`, 토큰 없이 → 401, 없는 과정 `GET /api/v1/courses/99999` → 404 `{"success":false,…,"error":{"code":"COURSE_NOT_FOUND",…}}`
> - 7회차 Step 4의 `CorpRulesTest`는 전환 전 코드에서 세 규칙 모두 위반 목록을 보여 준다(이것이 Step 5의 작업 목록). 응답이 `ApiResponse`로 바뀐 API의 기존 테스트는 JSON 경로를 `$.data…`로 바꾸면 통과
> - 방법 B: 폴더로 `publish` → `includeBuild` 없이 그 폴더를 `repositories`에 넣어 백엔드 컴파일 성공
> - ⚠️ Windows 공유 폴더(UNC 경로·네트워크 드라이브)로의 `publish`와 읽기는 실행해 보지 못했습니다
