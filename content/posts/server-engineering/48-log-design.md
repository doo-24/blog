---
title: "[로그, 모니터링, 관측 가능성] 1편 — 로그 설계: 추적 가능한 로그 만들기"
date: 2026-03-17T19:05:00+09:00
draft: false
tags: ["로그", "MDC", "구조화 로그", "JSON 로그", "서버"]
series: ["로그, 모니터링, 관측 가능성"]
summary: "로그 레벨 선택 기준(DEBUG~FATAL)과 남용 패턴, 구조화된 로그(JSON)와 컨텍스트 필드 설계, MDC(Mapped Diagnostic Context)와 요청 ID 전파, 민감 정보 마스킹과 로그 볼륨 관리까지"
---

프로덕션 장애가 발생했을 때 로그를 열면 두 가지 세계가 펼쳐진다. 하나는 `2024-01-15 03:47:22 ERROR NullPointerException`처럼 무엇이 터졌는지는 알지만 왜 터졌는지, 어떤 사용자가 어떤 요청을 보냈을 때 발생했는지 전혀 알 수 없는 세계다. 다른 하나는 요청 ID 하나로 전체 흐름을 5초 만에 추적해 원인을 특정하는 세계다. 이 두 세계의 차이는 로그를 "찍는" 것과 "설계하는" 것의 차이에서 비롯된다. 로그는 사후 분석 도구이자 시스템의 실시간 상태를 들여다보는 창문이다. 잘 설계된 로그는 장애 대응 시간을 단축하고, 버그 재현을 가능하게 하며, 비정상 패턴을 자동으로 감지할 수 있게 한다. 이번 편에서는 로그 레벨 선택부터 구조화, MDC를 통한 컨텍스트 전파, 민감 정보 처리까지 — 추적 가능한 로그를 만드는 모든 것을 다룬다.

---

## 1. 로그 레벨: 선택 기준과 남용 패턴

로그 레벨은 단순한 분류 태그가 아니다. 레벨은 "이 메시지를 언제 볼 것인가"를 결정하는 운영 정책이다. 레벨을 잘못 선택하면 로그 볼륨이 폭발하거나, 정작 중요한 신호가 노이즈에 묻힌다.

### 각 레벨의 의미와 사용 기준

**TRACE** — 가장 상세한 레벨. 메서드 진입/종료, 변수 값, 루프 반복 상태 등 개발 중에만 필요한 정보다. 프로덕션에서는 절대 활성화하지 않는다. 성능 프로파일링 중에 일시적으로 켰다가 즉시 끄는 용도로 쓴다.

```java
log.trace("Entering calculateDiscount: userId={}, cartTotal={}", userId, cartTotal);
```

**DEBUG** — 개발/스테이징 환경에서 문제를 진단하기 위한 레벨. 비즈니스 흐름에서 중요한 분기점, 외부 시스템 호출 결과, 캐시 히트/미스 여부 등을 기록한다. 프로덕션에서는 기본적으로 비활성화하되, 특정 패키지나 클래스에 대해 동적으로 활성화할 수 있어야 한다.

```java
log.debug("Cache miss for key={}, fetching from DB", cacheKey);
log.debug("Payment gateway response: status={}, transactionId={}", status, txId);
```

**INFO** — 시스템의 정상적인 중요 이벤트. 애플리케이션 시작/종료, 요청 처리 완료, 배치 작업 시작/종료, 중요한 설정 변경 등이 대상이다. 프로덕션에서 기본 레벨이며, 이 레벨만 보고도 시스템이 정상 동작하고 있는지 판단할 수 있어야 한다.

```java
log.info("Order created: orderId={}, userId={}, amount={}, items={}",
    orderId, userId, amount, itemCount);
log.info("Application started on port={}, profile={}", port, activeProfile);
```

**WARN** — 즉각적인 조치는 불필요하지만 주의가 필요한 상황. 재시도 후 성공한 외부 호출, 폴백 로직 실행, 설정값이 기본값으로 대체된 경우, SLA에 근접한 응답 시간 등이다.

```java
log.warn("Retry succeeded after {} attempts: endpoint={}", retryCount, endpoint);
log.warn("Response time approaching threshold: endpoint={}, elapsed={}ms, threshold={}ms",
    endpoint, elapsed, threshold);
```

**ERROR** — 즉각적인 주의가 필요한 오류. 예외가 발생해 요청 처리에 실패한 경우, 외부 의존성이 완전히 실패한 경우, 데이터 정합성 문제가 감지된 경우다. ERROR 로그는 알림(alert)을 트리거해야 하는 신호다.

```java
log.error("Failed to process payment: orderId={}, error={}", orderId, e.getMessage(), e);
```

**FATAL** — 시스템이 계속 동작할 수 없는 치명적 오류. 데이터베이스 연결 풀 고갈, 핵심 설정 파일 누락, OOM 직전 상황 등이다. Logback에서는 `ERROR`와 동일하게 처리되므로, 실제로는 ERROR로 통일하고 알림 심각도를 별도로 관리하는 경우가 많다.

### 레벨 남용 패턴과 안티패턴

**안티패턴 1: INFO 폭탄**

```java
// 나쁜 예 — 루프 내부의 INFO 로그
for (Order order : orders) {
    log.info("Processing order: {}", order.getId()); // 10만 건이면 10만 줄
    processOrder(order);
    log.info("Order processed: {}", order.getId()); // 또 10만 줄
}

// 좋은 예 — 배치 단위로 요약
log.info("Starting batch: count={}", orders.size());
for (Order order : orders) {
    log.debug("Processing order: {}", order.getId());
    processOrder(order);
}
log.info("Batch completed: processed={}, failed={}, elapsed={}ms",
    processed, failed, elapsed);
```

**안티패턴 2: 예외를 삼키는 DEBUG**

```java
// 나쁜 예 — 실제 오류를 DEBUG로 숨김
try {
    externalService.call();
} catch (Exception e) {
    log.debug("External service failed", e); // 프로덕션에서는 보이지도 않는다
}

// 좋은 예 — 비즈니스 임팩트에 따라 레벨 결정
try {
    externalService.call();
} catch (ExternalServiceException e) {
    log.error("External service unavailable, falling back: service={}, error={}",
        serviceName, e.getMessage(), e);
    return fallback();
}
```

**안티패턴 3: 모든 예외를 ERROR로**

```java
// 나쁜 예 — 정상적인 비즈니스 케이스도 ERROR
try {
    return userRepository.findById(id).orElseThrow(() -> new UserNotFoundException(id));
} catch (UserNotFoundException e) {
    log.error("User not found: {}", id, e); // NotFoundException은 WARN이면 충분
}

// 좋은 예
try {
    return userRepository.findById(id).orElseThrow(() -> new UserNotFoundException(id));
} catch (UserNotFoundException e) {
    log.warn("User not found: userId={}", id);
    throw e;
}
```

**안티패턴 4: 동일 이벤트 중복 로깅**

여러 계층(Controller → Service → Repository)에서 동일한 예외를 각각 로깅하면 하나의 오류가 로그에 세 번 나타난다. 원칙은 간단하다 — 예외는 처리하는 계층에서 한 번만 로깅한다. 전파할 때는 로깅하지 않는다.

```java
// 나쁜 예 — 각 계층에서 중복 로깅
@Repository
public User findUser(long id) {
    try { ... } catch (Exception e) {
        log.error("DB error", e); // 1번
        throw new DataAccessException(e);
    }
}

@Service
public User getUser(long id) {
    try { return userRepository.findUser(id); }
    catch (DataAccessException e) {
        log.error("Service error", e); // 2번 — 같은 예외
        throw new UserServiceException(e);
    }
}

// 좋은 예 — 최상위에서 한 번만
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> handle(Exception e, HttpServletRequest request) {
        log.error("Unhandled exception: method={}, uri={}",
            request.getMethod(), request.getRequestURI(), e);
        return ResponseEntity.internalServerError().build();
    }
}
```

---

## 2. 구조화된 로그: JSON과 컨텍스트 필드 설계

텍스트 로그의 가장 큰 문제는 파싱이다. `grep`, `awk`, 정규식으로 로그를 분석하는 것은 한계가 있다. Elasticsearch, Loki, CloudWatch Logs Insights 같은 현대적인 로그 시스템은 JSON 구조화 로그를 기반으로 강력한 필터링과 집계를 제공한다.

### Logback + Logstash Encoder로 JSON 로그 구성

```xml
<!-- build.gradle -->
implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
```

```xml
<!-- logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="appName" source="spring.application.name"/>
    <springProperty scope="context" name="activeProfile" source="spring.profiles.active"/>

    <appender name="CONSOLE_JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app":"${appName}","env":"${activeProfile}"}</customFields>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <message>message</message>
                <logger>logger</logger>
                <thread>thread</thread>
                <level>level</level>
            </fieldNames>
            <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                <maxDepthPerCause>20</maxDepthPerCause>
                <shortenedClassNameLength>20</shortenedClassNameLength>
                <rootCauseFirst>true</rootCauseFirst>
            </throwableConverter>
        </encoder>
    </appender>

    <!-- 개발 환경: 읽기 쉬운 텍스트 -->
    <springProfile name="local,test">
        <appender name="CONSOLE_TEXT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE_TEXT"/>
        </root>
    </springProfile>

    <!-- 프로덕션: JSON -->
    <springProfile name="prod,staging">
        <root level="INFO">
            <appender-ref ref="CONSOLE_JSON"/>
        </root>
        <logger name="com.example" level="INFO"/>
        <logger name="org.springframework" level="WARN"/>
        <logger name="org.hibernate.SQL" level="WARN"/>
    </springProfile>
</configuration>
```

출력 예시:

```json
{
  "timestamp": "2024-01-15T03:47:22.413+09:00",
  "level": "INFO",
  "thread": "http-nio-8080-exec-3",
  "logger": "c.e.order.OrderService",
  "message": "Order created",
  "app": "order-service",
  "env": "prod",
  "orderId": "ORD-20240115-001",
  "userId": "USR-12345",
  "amount": 59000,
  "itemCount": 3,
  "traceId": "4bf92f3577b34da6",
  "spanId": "00f067aa0ba902b7"
}
```

### 컨텍스트 필드 설계 원칙

좋은 구조화 로그의 핵심은 "나중에 어떤 질문을 할 것인가"를 미리 생각하는 것이다.

**필수 공통 필드**

| 필드 | 설명 | 예시 |
|------|------|------|
| `timestamp` | ISO 8601, 타임존 포함 | `2024-01-15T03:47:22.413+09:00` |
| `level` | 로그 레벨 | `INFO` |
| `app` | 서비스 이름 | `order-service` |
| `env` | 환경 | `prod` |
| `traceId` | 분산 추적 ID | `4bf92f3577b34da6` |
| `requestId` | 단일 요청 ID | `req-uuid-v4` |
| `userId` | 인증된 사용자 ID | `USR-12345` |

**도메인별 필드 예시**

```java
// 주문 도메인
log.info("Order created",
    StructuredArguments.keyValue("orderId", order.getId()),
    StructuredArguments.keyValue("userId", order.getUserId()),
    StructuredArguments.keyValue("amount", order.getTotalAmount()),
    StructuredArguments.keyValue("paymentMethod", order.getPaymentMethod()),
    StructuredArguments.keyValue("itemCount", order.getItems().size())
);

// HTTP 요청/응답
log.info("HTTP request completed",
    StructuredArguments.keyValue("method", request.getMethod()),
    StructuredArguments.keyValue("uri", request.getRequestURI()),
    StructuredArguments.keyValue("status", response.getStatus()),
    StructuredArguments.keyValue("elapsedMs", elapsed),
    StructuredArguments.keyValue("userAgent", request.getHeader("User-Agent"))
);
```

### Logstash Markers로 구조화

`net.logstash.logback.marker.Markers`를 사용하면 Map을 통째로 로그에 첨부할 수 있다.

```java
import static net.logstash.logback.marker.Markers.append;
import static net.logstash.logback.marker.Markers.appendEntries;

@Service
public class OrderService {

    public Order createOrder(CreateOrderRequest request) {
        Map<String, Object> context = Map.of(
            "orderId", generateOrderId(),
            "userId", request.getUserId(),
            "amount", request.getTotalAmount(),
            "itemCount", request.getItems().size()
        );

        log.info(appendEntries(context), "Order created");

        // 개별 필드 추가
        log.info(append("orderId", orderId).and(append("status", "PAID")),
            "Payment confirmed");

        return order;
    }
}
```

### 안티패턴: 구조화를 방해하는 나쁜 습관

```java
// 나쁜 예 1 — 메시지에 모든 정보를 문자열로 때려넣기
log.info("Order created: orderId=" + orderId + ", userId=" + userId + ", amount=" + amount);
// → JSON에서 message 필드 하나에 전부 들어가 검색/집계 불가

// 나쁜 예 2 — 일관성 없는 필드명
log.info("user_id={}, order-id={}", userId, orderId);  // 스네이크, 케밥 혼용
log.info("userId={}, orderID={}", userId, orderId);    // 카멜, 혼용

// 좋은 예 — 팀 전체에서 동일한 필드명 규약 적용
// camelCase 통일: userId, orderId, elapsedMs, errorCode
```

---

## 3. MDC: 요청 ID와 컨텍스트 전파

MDC(Mapped Diagnostic Context)는 스레드 로컬 Map으로, 한 번 설정하면 이후 모든 로그에 자동으로 포함된다. 요청 ID를 MDC에 넣으면 해당 요청이 처리되는 동안 발생하는 모든 로그를 요청 ID 하나로 추적할 수 있다.

### Spring Filter로 MDC 자동 설정

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class RequestContextFilter extends OncePerRequestFilter {

    private static final String TRACE_ID_HEADER = "X-Trace-Id";
    private static final String REQUEST_ID_KEY = "requestId";
    private static final String TRACE_ID_KEY = "traceId";
    private static final String USER_ID_KEY = "userId";
    private static final String CLIENT_IP_KEY = "clientIp";
    private static final String METHOD_KEY = "httpMethod";
    private static final String URI_KEY = "uri";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        long startTime = System.currentTimeMillis();
        String requestId = UUID.randomUUID().toString();

        // 업스트림에서 전달된 traceId가 있으면 재사용, 없으면 생성
        String traceId = Optional.ofNullable(request.getHeader(TRACE_ID_HEADER))
            .filter(s -> !s.isBlank())
            .orElse(generateTraceId());

        try {
            MDC.put(REQUEST_ID_KEY, requestId);
            MDC.put(TRACE_ID_KEY, traceId);
            MDC.put(CLIENT_IP_KEY, extractClientIp(request));
            MDC.put(METHOD_KEY, request.getMethod());
            MDC.put(URI_KEY, request.getRequestURI());

            // 인증 정보가 있으면 추가
            extractUserId(request).ifPresent(uid -> MDC.put(USER_ID_KEY, uid));

            // 다운스트림으로 traceId 전파
            response.setHeader(TRACE_ID_HEADER, traceId);

            filterChain.doFilter(request, response);

        } finally {
            long elapsed = System.currentTimeMillis() - startTime;
            log.info("Request completed: status={}, elapsedMs={}",
                response.getStatus(), elapsed);
            MDC.clear(); // 반드시 정리 — 스레드 재사용 시 오염 방지
        }
    }

    private String generateTraceId() {
        return Long.toHexString(System.currentTimeMillis()) +
               Long.toHexString(ThreadLocalRandom.current().nextLong());
    }

    private String extractClientIp(HttpServletRequest request) {
        String forwarded = request.getHeader("X-Forwarded-For");
        if (forwarded != null && !forwarded.isBlank()) {
            return forwarded.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }

    private Optional<String> extractUserId(HttpServletRequest request) {
        // JWT 파싱 또는 SecurityContext에서 추출
        return Optional.ofNullable(request.getAttribute("userId"))
            .map(Object::toString);
    }
}
```

Logback 패턴에 MDC 필드 추가:

```xml
<!-- logback-spring.xml에서 JSON 인코더에 MDC 자동 포함 -->
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <!-- MDC의 모든 키는 자동으로 JSON에 포함됨 -->
    <includeMdcKeyName>requestId</includeMdcKeyName>
    <includeMdcKeyName>traceId</includeMdcKeyName>
    <includeMdcKeyName>userId</includeMdcKeyName>
    <includeMdcKeyName>clientIp</includeMdcKeyName>
</encoder>
```

이제 별도 코드 없이 모든 로그에 `requestId`, `traceId`, `userId`가 자동 포함된다:

```json
{"timestamp":"2024-01-15T03:47:22.413+09:00","level":"INFO","message":"Cache miss for key=product:123","requestId":"a1b2c3d4-...","traceId":"4bf92f3577b34da6","userId":"USR-12345","clientIp":"1.2.3.4"}
{"timestamp":"2024-01-15T03:47:22.521+09:00","level":"INFO","message":"Order created","requestId":"a1b2c3d4-...","traceId":"4bf92f3577b34da6","userId":"USR-12345","orderId":"ORD-001"}
```

### 비동기 환경에서 MDC 전파

MDC는 스레드 로컬이므로 `@Async`, `CompletableFuture`, 가상 스레드 환경에서는 별도 처리가 필요하다.

```java
// MDC 컨텍스트를 복사하는 Executor 래퍼
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> mdcContext = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (mdcContext != null) {
                    MDC.setContextMap(mdcContext);
                }
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}

// Spring @Async 설정
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setTaskDecorator(new MdcTaskDecorator()); // MDC 전파
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        return executor;
    }
}
```

```java
// CompletableFuture 사용 시
Map<String, String> mdcContext = MDC.getCopyOfContextMap();

CompletableFuture.supplyAsync(() -> {
    MDC.setContextMap(mdcContext); // 자식 스레드에 MDC 복원
    try {
        return externalService.fetch();
    } finally {
        MDC.clear();
    }
}, executor);
```

### 마이크로서비스 간 traceId 전파

서비스 A → 서비스 B → 서비스 C 호출 체인에서 동일한 `traceId`를 유지하려면 HTTP 클라이언트에서 헤더를 자동으로 전파해야 한다.

```java
// RestTemplate 인터셉터
public class TraceIdInterceptor implements ClientHttpRequestInterceptor {
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                                        ClientHttpRequestExecution execution)
            throws IOException {
        String traceId = MDC.get("traceId");
        if (traceId != null) {
            request.getHeaders().set("X-Trace-Id", traceId);
        }
        String requestId = MDC.get("requestId");
        if (requestId != null) {
            request.getHeaders().set("X-Request-Id", requestId);
        }
        return execution.execute(request, body);
    }
}

// WebClient 필터 (Reactive)
WebClient webClient = WebClient.builder()
    .filter((request, next) -> {
        String traceId = MDC.get("traceId");
        ClientRequest modified = ClientRequest.from(request)
            .header("X-Trace-Id", traceId != null ? traceId : "")
            .build();
        return next.exchange(modified);
    })
    .build();
```

---

## 4. 민감 정보 마스킹과 로그 볼륨 관리

### 민감 정보 마스킹

로그에 개인정보(PII), 결제 정보, 비밀번호가 그대로 남는 것은 보안 사고의 직접적인 원인이다. GDPR, 개인정보보호법 위반이기도 하다. 마스킹은 로깅 시점에, 소스 코드 레벨에서 적용해야 한다.

**도메인 객체에 toString 마스킹 적용**

```java
public class PaymentInfo {
    private String cardNumber;  // 1234-5678-9012-3456
    private String cvv;
    private String cardholderName;

    @Override
    public String toString() {
        // 마지막 4자리만 노출
        String maskedCard = "****-****-****-" +
            cardNumber.substring(cardNumber.length() - 4);
        return "PaymentInfo{card=" + maskedCard + ", cvv=***, name=" + cardholderName + "}";
    }
}
```

**Logback의 PatternLayoutEncoder 마스킹 (최후 수단)**

```xml
<!-- 정규식으로 민감 데이터 패턴 마스킹 — 성능 주의 -->
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <jsonGeneratorDecorator class="net.logstash.logback.mask.MaskingJsonGeneratorDecorator">
        <defaultMask>****</defaultMask>
        <valueMasker class="net.logstash.logback.mask.RegexValueMasker">
            <!-- 신용카드 번호 패턴 -->
            <regex>\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b</regex>
            <mask>****-****-****-####</mask>
        </valueMasker>
        <pathMasker>
            <!-- JSON 필드명 기반 마스킹 -->
            <path>password</path>
            <path>cvv</path>
            <path>ssn</path>
            <path>secretKey</path>
        </pathMasker>
    </jsonGeneratorDecorator>
</encoder>
```

**마스킹 유틸리티 클래스**

```java
public final class LogMasker {

    private LogMasker() {}

    // 이메일: test@example.com → t***@example.com
    public static String email(String email) {
        if (email == null || !email.contains("@")) return "***";
        String[] parts = email.split("@");
        String local = parts[0];
        String masked = local.charAt(0) + "***";
        return masked + "@" + parts[1];
    }

    // 전화번호: 010-1234-5678 → 010-****-5678
    public static String phone(String phone) {
        if (phone == null) return "***";
        return phone.replaceAll("(\\d{3})[- ]?(\\d{3,4})[- ]?(\\d{4})", "$1-****-$3");
    }

    // 이름: 홍길동 → 홍*동
    public static String name(String name) {
        if (name == null || name.length() < 2) return "*";
        if (name.length() == 2) return name.charAt(0) + "*";
        return name.charAt(0) + "*".repeat(name.length() - 2) + name.charAt(name.length() - 1);
    }

    // 카드번호 마지막 4자리만
    public static String cardNumber(String cardNumber) {
        if (cardNumber == null || cardNumber.length() < 4) return "****";
        return "****-****-****-" + cardNumber.replaceAll("[^0-9]", "")
            .substring(cardNumber.replaceAll("[^0-9]", "").length() - 4);
    }
}

// 사용 예
log.info("User registered: email={}, name={}, phone={}",
    LogMasker.email(user.getEmail()),
    LogMasker.name(user.getName()),
    LogMasker.phone(user.getPhone()));
```

### 로그 볼륨 관리

로그가 너무 많으면 스토리지 비용이 증가하고, 정작 중요한 로그가 묻힌다. 볼륨 관리는 비용 문제이자 신호 대비 노이즈(SNR) 문제다.

**샘플링으로 고빈도 로그 줄이기**

```java
@Component
public class SamplingLogger {

    private final AtomicLong counter = new AtomicLong(0);
    private static final int SAMPLE_RATE = 100; // 100건 중 1건만 로깅

    public void logWithSampling(String message, Object... args) {
        long count = counter.incrementAndGet();
        if (count % SAMPLE_RATE == 0) {
            log.info("[sampled:{}/{}] " + message,
                ArrayUtils.addAll(new Object[]{count, SAMPLE_RATE}, args));
        }
    }
}

// 헬스체크 엔드포인트같은 고빈도 요청에 적용
public class HealthCheckFilter extends OncePerRequestFilter {
    private final AtomicLong healthCheckCount = new AtomicLong(0);

    @Override
    protected void doFilterInternal(...) {
        if (request.getRequestURI().equals("/actuator/health")) {
            long count = healthCheckCount.incrementAndGet();
            if (count % 100 == 0) {
                log.debug("Health check: count={}", count);
            }
            // 로깅 없이 바로 통과
            filterChain.doFilter(request, response);
            return;
        }
        // 일반 요청은 정상 처리
        filterChain.doFilter(request, response);
    }
}
```

**동적 로그 레벨 변경 (Spring Boot Actuator)**

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: loggers, health, info
  endpoint:
    loggers:
      enabled: true
```

```bash
# 특정 패키지의 로그 레벨을 런타임에 DEBUG로 변경
curl -X POST http://localhost:8080/actuator/loggers/com.example.payment \
  -H 'Content-Type: application/json' \
  -d '{"configuredLevel": "DEBUG"}'

# 원복
curl -X POST http://localhost:8080/actuator/loggers/com.example.payment \
  -H 'Content-Type: application/json' \
  -d '{"configuredLevel": "INFO"}'
```

**로그 압축과 롤링 정책**

```xml
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/var/log/app/application.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <!-- 일별 롤링, 최대 파일 크기 100MB -->
        <fileNamePattern>/var/log/app/application.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <!-- 30일치 보관, 총 5GB 초과 시 오래된 것부터 삭제 -->
        <maxHistory>30</maxHistory>
        <totalSizeCap>5GB</totalSizeCap>
    </rollingPolicy>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

**비동기 어펜더로 I/O 성능 개선**

```xml
<!-- 로그 I/O가 요청 처리를 블로킹하지 않도록 비동기 처리 -->
<appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <discardingThreshold>0</discardingThreshold> <!-- 큐 가득 차도 버리지 않음 -->
    <queueSize>512</queueSize>
    <includeCallerData>false</includeCallerData> <!-- 스택 트레이스 계산 비용 절약 -->
    <appender-ref ref="FILE"/>
</appender>
```

### 로그 보존 정책과 비용 최적화

모든 로그를 동일한 기간 동안 보관할 필요는 없다. 레벨과 도메인에 따라 보존 기간을 다르게 설정한다.

| 레벨/종류 | 보존 기간 | 스토리지 티어 |
|-----------|-----------|---------------|
| ERROR, WARN | 90일 | Hot (즉시 검색) |
| INFO (비즈니스 이벤트) | 30일 | Hot |
| INFO (HTTP 접근 로그) | 7일 | Hot, 이후 Cold |
| DEBUG | 3일 | Hot |
| 감사 로그 (Audit) | 5년 | Warm/Cold + 암호화 |

Elasticsearch ILM(Index Lifecycle Management) 또는 AWS CloudWatch Logs의 보존 정책, Loki의 retention 설정으로 자동화할 수 있다.

---

## 5. 실전: 로그 설계 체크리스트

설계가 완료되면 아래 항목으로 점검한다.

**레벨 검토**
- [ ] 루프 내부에 INFO 로그가 없는가?
- [ ] 예외를 삼키면서 DEBUG로 로깅하는 코드가 없는가?
- [ ] 동일 예외를 여러 계층에서 중복 로깅하지 않는가?
- [ ] 비즈니스 정상 케이스(404, 사용자 미존재 등)를 ERROR로 로깅하지 않는가?

**구조화 검토**
- [ ] 프로덕션 환경에서 JSON으로 출력되는가?
- [ ] 모든 로그에 `traceId`, `requestId`가 포함되는가?
- [ ] 필드명이 팀 전체에서 일관성 있게 사용되는가?
- [ ] 문자열 연결(+)이 아닌 구조화 인자를 사용하는가?

**보안 검토**
- [ ] 비밀번호, 카드번호, 주민번호가 평문으로 로깅되지 않는가?
- [ ] 이메일, 전화번호 등 PII가 마스킹되어 있는가?
- [ ] Authorization 헤더가 로그에 남지 않는가?

**운영 검토**
- [ ] 고빈도 엔드포인트(헬스체크, 메트릭)에 샘플링이 적용되어 있는가?
- [ ] 로그 롤링과 보존 정책이 설정되어 있는가?
- [ ] 비동기 어펜더를 사용하는가?
- [ ] 런타임 로그 레벨 변경이 가능한가?

---

로그는 시스템이 스스로 하는 이야기다. 잘 설계된 로그는 장애 순간에 조용히 빛을 발한다. 구조화되지 않은 텍스트 더미가 아니라, 요청 흐름을 추적하고 원인을 특정할 수 있는 데이터로 만드는 것 — 이것이 로그 설계의 핵심이다. 다음 편에서는 이 로그를 기반으로 메트릭을 수집하고 대시보드를 구성하는 모니터링 설계를 다룬다.

---

## 참고 자료

1. **Logback 공식 문서** — [https://logback.qos.ch/documentation.html](https://logback.qos.ch/documentation.html) — Appender, Layout, Filter 설정의 기준 문서
2. **logstash-logback-encoder GitHub** — [https://github.com/logfellow/logstash-logback-encoder](https://github.com/logfellow/logstash-logback-encoder) — JSON 구조화 로그 인코더 사용법과 마스킹 API
3. **The Twelve-Factor App: Logs** — [https://12factor.net/logs](https://12factor.net/logs) — 로그를 이벤트 스트림으로 처리하는 원칙
4. **OWASP Logging Cheat Sheet** — [https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) — 보안 관점의 로깅 가이드라인과 민감 정보 처리
5. **Spring Boot Actuator: Loggers** — [https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.loggers](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.loggers) — 런타임 로그 레벨 변경 공식 문서
6. **Google Cloud: Best practices for application logging** — [https://cloud.google.com/logging/docs/structured-logging](https://cloud.google.com/logging/docs/structured-logging) — 구조화 로그 필드 설계와 클라우드 로깅 통합 가이드
