---
title: "[서버 애플리케이션 설계] 8편 — 에러 처리와 예외 전략: 실패를 우아하게"
date: 2026-03-18T00:02:00+09:00
draft: false
tags: ["에러 처리", "예외", "RFC 7807", "재시도", "서버"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "Checked/Unchecked 예외 선택 기준, 글로벌 예외 핸들러와 에러 응답 표준화(RFC 7807), 재시도 정책과 Exponential Backoff, 예외 vs 결과 타입, 방어적 프로그래밍까지"
---

프로덕션 서버에서 에러는 피할 수 없다. 네트워크는 끊기고, DB는 타임아웃을 뱉고, 서드파티 API는 예고 없이 500을 반환한다.

문제는 "에러가 발생하느냐"가 아니라 "에러가 발생했을 때 시스템이 어떻게 반응하느냐"다.

에러 처리를 설계의 핵심 관심사로 보지 않는 팀은 새벽 세 시에 알림을 받는다.

그리고 그 알림의 절반은 예측 가능한 실패였는데 코드가 이를 무시했거나, 잘못된 방식으로 전파했거나, 클라이언트에게 쓸모없는 500 Internal Server Error만 돌려줬기 때문에 발생한다.

이번 편에서는 예외 설계 원칙부터 표준화된 에러 응답, 재시도 전략, 결과 타입 패턴까지 프로덕션에서 통하는 에러 처리 전략을 깊이 있게 다룬다.

---

## 1. Checked vs Unchecked 예외: 잘못된 선택이 코드베이스를 망친다

Java는 예외를 두 가지로 나눈다. Checked 예외는 컴파일러가 처리를 강제하고, Unchecked 예외(RuntimeException 계열)는 개발자 재량에 맡긴다.

이 구분은 단순한 문법 차이가 아니라, **예외의 책임이 누구에게 있는가**라는 설계 철학의 차이다.

### 1.1 Checked 예외의 원래 의도

Checked 예외는 "발신자가 처리 가능한 복구 가능한 상황"을 위해 만들어졌다. `FileNotFoundException`이 대표적이다. 파일이 없을 때 호출자가 대안 경로를 선택하거나 사용자에게 알릴 수 있다는 가정이 깔려 있다.

```java
// Checked 예외가 의도한 사용 패턴
public String readConfig(String path) throws FileNotFoundException {
    File file = new File(path);
    if (!file.exists()) {
        throw new FileNotFoundException("설정 파일을 찾을 수 없습니다: " + path);
    }
    // ...
}

// 호출자가 복구 로직을 갖는다
public AppConfig loadConfig() {
    try {
        return parseConfig(readConfig("/etc/app/config.yaml"));
    } catch (FileNotFoundException e) {
        log.warn("설정 파일 없음, 기본값 사용: {}", e.getMessage());
        return AppConfig.defaults();
    }
}
```

이 패턴이 의미 있는 이유는 호출자가 실제로 **다른 행동**을 할 수 있기 때문이다. 하지만 현실 코드를 보면 Checked 예외의 의도와 다르게 사용되는 경우가 대부분이다.

### 1.2 Checked 예외가 문제가 되는 상황

Spring/JPA 등 현대 프레임워크가 거의 Unchecked 예외를 사용하는 데는 이유가 있다.

**문제 1: 예외 누수(Exception Leakage)**

```java
// 인터페이스가 구현 세부사항을 노출한다
public interface UserRepository {
    User findById(Long id) throws SQLException; // SQL이 인터페이스 계약에 노출됨
}
```

`UserRepository`를 MongoDB로 바꾸는 순간 인터페이스 시그니처가 변경되고, 모든 호출자 코드가 영향을 받는다. 추상화가 무너진다.

**문제 2: 강제 전파로 인한 보일러플레이트**

```java
// 처리할 수 없는데 throws를 달고 전파만 한다
public OrderDto createOrder(OrderRequest req) throws SQLException, IOException {
    // 실제로 SQLException을 복구하는 로직이 없다
    // 그냥 위로 던질 뿐이다
    User user = userRepository.findById(req.getUserId()); // throws SQLException
    // ...
}
```

이런 코드가 쌓이면 스택 전체에 `throws SQLException`이 퍼진다. 아무도 복구하지 않는데 모두가 선언만 한다. 이것은 타입 시스템의 노이즈다.

**문제 3: catch-and-ignore 안티패턴**

```java
// 컴파일러가 잠잠해지길 바라며 예외를 삼킨다
try {
    process();
} catch (SomeCheckedException e) {
    // TODO: 나중에 처리하자
}
```

Checked 예외를 강제하면 개발자들이 이런 식으로 예외를 숨긴다. 예외가 발생해도 시스템은 조용히 잘못된 상태로 계속 동작한다.

### 1.3 실용적 선택 기준

Checked와 Unchecked 중 무엇을 선택할지는 결국 "호출자가 이 예외로 무언가 의미 있는 행동을 할 수 있는가"로 귀결된다. 할 수 있다면 컴파일러가 강제하는 Checked가 유용하고, 그저 위로 전파할 뿐이라면 Unchecked가 코드베이스를 더 깔끔하게 유지한다.

아래 기준으로 판단하면 대부분의 상황에서 명확한 답이 나온다.

```
호출자가 예외를 받았을 때:
  ├── 다른 의미 있는 동작을 할 수 있는가?
  │     ├── YES → Checked 예외 고려
  │     └── NO  → Unchecked 예외 사용
  └── 예외가 구현 세부사항을 노출하는가?
        ├── YES → Unchecked + 도메인 예외로 래핑
        └── NO  → 케이스에 따라 결정
```

실무에서 Checked 예외가 적절한 경우는 생각보다 드물다. **외부 자원 접근(파일, 네트워크)에서 호출자가 실제로 복구 로직을 갖는 경우** 정도다. 대부분의 비즈니스 로직 예외는 Unchecked로 처리하는 편이 낫다.

### 1.4 도메인 예외 계층 설계

예외를 무조건 `RuntimeException`으로 던지는 것도 나쁜 설계다. 예외에 의미와 계층을 부여해야 한다.

```java
// 최상위 도메인 예외
public abstract class AppException extends RuntimeException {
    private final ErrorCode errorCode;

    protected AppException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }

    protected AppException(ErrorCode errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }
}

// 비즈니스 규칙 위반 — 클라이언트 잘못 (4xx)
public class BusinessException extends AppException {
    public BusinessException(ErrorCode errorCode, String message) {
        super(errorCode, message);
    }
}

// 리소스 없음
public class ResourceNotFoundException extends BusinessException {
    public ResourceNotFoundException(String resource, Object id) {
        super(ErrorCode.RESOURCE_NOT_FOUND,
              String.format("%s를 찾을 수 없습니다. id=%s", resource, id));
    }
}

// 서버 내부 오류 — 서버 잘못 (5xx)
public class InfrastructureException extends AppException {
    public InfrastructureException(ErrorCode errorCode, String message, Throwable cause) {
        super(errorCode, message, cause);
    }
}
```

```java
// ErrorCode enum — HTTP 상태 코드와 에러 코드를 함께 관리
public enum ErrorCode {
    // 4xx
    RESOURCE_NOT_FOUND(404, "RESOURCE_NOT_FOUND"),
    INVALID_INPUT(400, "INVALID_INPUT"),
    DUPLICATE_RESOURCE(409, "DUPLICATE_RESOURCE"),
    UNAUTHORIZED(401, "UNAUTHORIZED"),
    FORBIDDEN(403, "FORBIDDEN"),

    // 5xx
    INTERNAL_ERROR(500, "INTERNAL_ERROR"),
    EXTERNAL_API_ERROR(502, "EXTERNAL_API_ERROR"),
    DATABASE_ERROR(500, "DATABASE_ERROR");

    private final int httpStatus;
    private final String code;

    ErrorCode(int httpStatus, String code) {
        this.httpStatus = httpStatus;
        this.code = code;
    }
}
```

이 계층 구조의 장점은 글로벌 예외 핸들러에서 예외 타입만 보면 HTTP 상태 코드와 처리 방식이 결정된다는 점이다.

---

## 2. 글로벌 예외 핸들러와 에러 응답 표준화

### 2.1 왜 표준화가 필요한가

클라이언트 개발자 입장에서 생각해보자. 같은 서버에서 어떤 에러는:

```json
{ "error": "not found" }
```

어떤 에러는:

```json
{ "message": "User not found", "status": 404 }
```

어떤 에러는:

```json
{ "code": "USER_001", "description": "해당 사용자가 존재하지 않습니다" }
```

이런 응답이 섞여 나온다면, 클라이언트는 각 엔드포인트마다 에러 파싱 로직을 다르게 작성해야 한다. 유지보수 비용이 폭발한다.

RFC 7807(Problem Details for HTTP APIs)은 이 문제를 해결하기 위한 IETF 표준이다.

### 2.2 RFC 7807: Problem Details

RFC 7807은 에러 응답에 포함되어야 할 필드를 정의한다.

| 필드 | 타입 | 설명 |
|------|------|------|
| `type` | URI | 에러 유형을 식별하는 URI. 문서 링크로 활용 가능 |
| `title` | string | 에러 유형의 짧은 제목 (변하지 않는 값) |
| `status` | integer | HTTP 상태 코드 |
| `detail` | string | 이 특정 발생에 대한 상세 설명 |
| `instance` | URI | 이 특정 에러 발생을 식별하는 URI |

Content-Type은 `application/problem+json`을 사용한다.

```json
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/invalid-input",
  "title": "입력값이 유효하지 않습니다",
  "status": 400,
  "detail": "이메일 형식이 올바르지 않습니다: 'not-an-email'",
  "instance": "/api/v1/users",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "errors": [
    {
      "field": "email",
      "message": "올바른 이메일 형식이 아닙니다"
    }
  ]
}
```

`errors` 같은 커스텀 필드는 RFC 7807이 허용하는 확장이다. 표준을 지키면서도 필요한 정보를 추가할 수 있다.

### 2.3 Spring에서 글로벌 예외 핸들러 구현

Spring MVC에서는 `@RestControllerAdvice`로 모든 컨트롤러의 예외를 한 곳에서 처리한다.

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ProblemDetail> handleResourceNotFound(
            ResourceNotFoundException ex,
            HttpServletRequest request) {

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
        problem.setType(URI.create("https://api.example.com/problems/resource-not-found"));
        problem.setTitle("리소스를 찾을 수 없습니다");
        problem.setDetail(ex.getMessage());
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("traceId", MDC.get("traceId"));

        log.warn("리소스 없음: path={}, message={}", request.getRequestURI(), ex.getMessage());

        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .contentType(MediaType.APPLICATION_PROBLEM_JSON)
                .body(problem);
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ProblemDetail> handleBusinessException(
            BusinessException ex,
            HttpServletRequest request) {

        HttpStatus status = HttpStatus.resolve(ex.getErrorCode().getHttpStatus());

        ProblemDetail problem = ProblemDetail.forStatus(status);
        problem.setType(URI.create("https://api.example.com/problems/" +
                ex.getErrorCode().getCode().toLowerCase().replace("_", "-")));
        problem.setTitle(ex.getErrorCode().getCode());
        problem.setDetail(ex.getMessage());
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("traceId", MDC.get("traceId"));

        log.warn("비즈니스 예외: code={}, message={}", ex.getErrorCode(), ex.getMessage());

        return ResponseEntity.status(status)
                .contentType(MediaType.APPLICATION_PROBLEM_JSON)
                .body(problem);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidationException(
            MethodArgumentNotValidException ex,
            HttpServletRequest request) {

        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors()
                .stream()
                .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
                .collect(Collectors.toList());

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setType(URI.create("https://api.example.com/problems/invalid-input"));
        problem.setTitle("입력값이 유효하지 않습니다");
        problem.setDetail("요청 데이터 검증에 실패했습니다");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("traceId", MDC.get("traceId"));
        problem.setProperty("errors", fieldErrors);

        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .contentType(MediaType.APPLICATION_PROBLEM_JSON)
                .body(problem);
    }

    // 최후의 보루 — 처리되지 않은 모든 예외
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleUnexpectedException(
            Exception ex,
            HttpServletRequest request) {

        // 예상치 못한 예외는 반드시 ERROR 레벨로 로깅
        log.error("예상치 못한 예외 발생: path={}", request.getRequestURI(), ex);

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        problem.setType(URI.create("https://api.example.com/problems/internal-error"));
        problem.setTitle("서버 내부 오류");
        problem.setDetail("일시적인 오류가 발생했습니다. 잠시 후 다시 시도해주세요.");
        problem.setInstance(URI.create(request.getRequestURI()));
        problem.setProperty("traceId", MDC.get("traceId"));

        // 내부 스택 트레이스나 시스템 정보는 절대 노출하지 않는다
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .contentType(MediaType.APPLICATION_PROBLEM_JSON)
                .body(problem);
    }
}
```

### 2.4 traceId 연동: 에러를 추적 가능하게

에러 응답에 `traceId`를 포함시키면 사용자가 "오류 코드: 4bf92f35"를 전달했을 때 로그에서 즉시 추적할 수 있다. MDC(Mapped Diagnostic Context)를 활용하면 쉽게 구현된다.

```java
@Component
public class TraceIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String traceId = Optional.ofNullable(request.getHeader("X-Trace-Id"))
                .orElse(UUID.randomUUID().toString().replace("-", "").substring(0, 16));

        MDC.put("traceId", traceId);
        response.setHeader("X-Trace-Id", traceId);

        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear(); // 반드시 정리 — 스레드 풀 환경에서 누수 방지
        }
    }
}
```

### 2.5 에러 응답에 절대 포함하면 안 되는 것들

보안 관점에서 에러 응답이 공격자에게 시스템 정보를 노출하는 경우가 많다.

```
// 나쁜 예 — 스택 트레이스 노출
{
  "error": "java.lang.NullPointerException at com.example.service.UserService.findById(UserService.java:45)..."
}

// 나쁜 예 — SQL 쿼리 노출
{
  "error": "ERROR: column \"pasword\" does not exist at character 22\nQuery: SELECT id, pasword FROM users WHERE..."
}

// 나쁜 예 — 내부 경로 노출
{
  "error": "FileNotFoundException: /opt/app/configs/database-prod.yaml"
}
```

스택 트레이스는 공격자에게 클래스 구조, 라이브러리 버전, 취약 지점을 알려준다. 프로덕션에서는 사용자에게 `traceId`만 주고, 상세 정보는 서버 로그에만 남긴다.

---

## 3. 재시도 정책: 실패를 기회로

### 3.1 일시적 실패와 영구적 실패를 구분하라

모든 실패가 재시도 대상이 아니다. 재시도 전에 반드시 물어야 할 질문이 있다: **"같은 요청을 다시 보내면 성공할 가능성이 있는가?"**

```
실패 유형 분류:
  ├── 일시적(Transient) 실패 → 재시도 가능
  │     ├── 네트워크 타임아웃
  │     ├── 503 Service Unavailable
  │     ├── 429 Too Many Requests
  │     └── DB Connection Pool 일시 고갈
  │
  └── 영구적(Permanent) 실패 → 재시도 불가
        ├── 400 Bad Request (요청 자체가 잘못됨)
        ├── 401 Unauthorized (인증 실패)
        ├── 404 Not Found (리소스가 없음)
        └── 비즈니스 규칙 위반
```

400을 재시도하면 영원히 실패한다. 오히려 불필요한 부하만 늘린다. 재시도 로직에는 반드시 **재시도 가능 여부 판단 로직**이 포함되어야 한다.

### 3.2 Exponential Backoff 원리

단순 재시도(fixed interval)는 위험하다. 서버가 과부하 상태일 때 모든 클라이언트가 동시에 재시도하면 thundering herd 문제가 발생해 서버가 더 힘들어진다.

Exponential Backoff는 재시도 간격을 지수적으로 늘린다.

```
재시도 간격 = base_delay * (2 ^ attempt)

attempt=0: 1초
attempt=1: 2초
attempt=2: 4초
attempt=3: 8초
attempt=4: 16초
...
```

하지만 모든 클라이언트가 같은 공식을 쓰면 여전히 동시에 재시도한다. 여기서 **Jitter**가 필요하다.

### 3.3 Jitter: 동시성 폭풍을 막아라

Jitter는 재시도 간격에 무작위성을 추가한다. AWS에서 제안한 "Full Jitter" 전략이 가장 널리 사용된다.

```
// Full Jitter
재시도 간격 = random(0, min(max_delay, base_delay * 2^attempt))

// Equal Jitter
cap = min(max_delay, base_delay * 2^attempt)
재시도 간격 = cap/2 + random(0, cap/2)

// Decorrelated Jitter (AWS 권장)
재시도 간격 = random(base_delay, 이전_간격 * 3)
```

실제 구현 예시를 보자.

```java
public class RetryPolicy {
    private final int maxAttempts;
    private final long baseDelayMs;
    private final long maxDelayMs;
    private final Set<Class<? extends Throwable>> retryableExceptions;
    private final Random random = new Random();

    public RetryPolicy(int maxAttempts, long baseDelayMs, long maxDelayMs,
                       Set<Class<? extends Throwable>> retryableExceptions) {
        this.maxAttempts = maxAttempts;
        this.baseDelayMs = baseDelayMs;
        this.maxDelayMs = maxDelayMs;
        this.retryableExceptions = retryableExceptions;
    }

    public <T> T execute(Supplier<T> operation) {
        int attempt = 0;
        Throwable lastException = null;

        while (attempt < maxAttempts) {
            try {
                return operation.get();
            } catch (Exception e) {
                if (!isRetryable(e)) {
                    throw e; // 재시도 불가 예외는 즉시 전파
                }
                lastException = e;
                attempt++;

                if (attempt < maxAttempts) {
                    long delay = calculateDelay(attempt);
                    log.warn("재시도 예정: attempt={}/{}, delay={}ms, cause={}",
                            attempt, maxAttempts, delay, e.getMessage());
                    sleep(delay);
                }
            }
        }

        throw new MaxRetryExceededException(
                "최대 재시도 횟수(" + maxAttempts + ")를 초과했습니다", lastException);
    }

    private long calculateDelay(int attempt) {
        // Exponential Backoff with Full Jitter
        long exponentialDelay = (long) (baseDelayMs * Math.pow(2, attempt - 1));
        long cappedDelay = Math.min(maxDelayMs, exponentialDelay);
        return (long) (random.nextDouble() * cappedDelay); // Full Jitter
    }

    private boolean isRetryable(Throwable e) {
        return retryableExceptions.stream()
                .anyMatch(retryableType -> retryableType.isInstance(e));
    }

    private void sleep(long ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException ie) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("재시도 대기 중 인터럽트 발생", ie);
        }
    }
}
```

### 3.4 Spring Retry와 Resilience4j 활용

실무에서는 직접 구현보다 검증된 라이브러리를 쓰는 편이 낫다.

**Spring Retry:**

```java
@Service
public class ExternalApiService {

    @Retryable(
        retryFor = {HttpServerErrorException.class, ResourceAccessException.class},
        noRetryFor = {HttpClientErrorException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2.0, random = true)
    )
    public ApiResponse callExternalApi(String payload) {
        // HTTP 호출 로직
        return restTemplate.postForObject(url, payload, ApiResponse.class);
    }

    @Recover
    public ApiResponse recover(Exception ex, String payload) {
        // 모든 재시도 실패 후 폴백 로직
        log.error("외부 API 호출 최종 실패: payload={}", payload, ex);
        return ApiResponse.fallback();
    }
}
```

**Resilience4j (더 세밀한 제어가 필요할 때):**

```java
@Configuration
public class ResilienceConfig {

    @Bean
    public RetryRegistry retryRegistry() {
        RetryConfig config = RetryConfig.custom()
                .maxAttempts(3)
                .waitDuration(Duration.ofMillis(500))
                .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                        500,   // 초기 대기 시간 (ms)
                        2.0,   // 배수
                        0.5    // 랜덤화 인자
                ))
                .retryOnException(ex ->
                        ex instanceof TransientException ||
                        ex instanceof SocketTimeoutException)
                .ignoreExceptions(BusinessException.class)
                .build();

        return RetryRegistry.of(config);
    }
}

@Service
@RequiredArgsConstructor
public class PaymentService {

    private final RetryRegistry retryRegistry;
    private final PaymentGateway paymentGateway;

    public PaymentResult processPayment(PaymentRequest request) {
        Retry retry = retryRegistry.retry("payment-gateway");

        // 재시도 이벤트 모니터링
        retry.getEventPublisher()
                .onRetry(event -> log.warn("결제 재시도: attempt={}", event.getNumberOfRetryAttempts()))
                .onError(event -> log.error("결제 최종 실패"));

        return Retry.decorateSupplier(retry,
                () -> paymentGateway.charge(request))
                .get();
    }
}
```

### 3.5 멱등성(Idempotency): 재시도의 전제 조건

재시도를 안전하게 하려면 작업이 **멱등(idempotent)**해야 한다. 같은 요청을 여러 번 보내도 결과가 동일해야 한다.

```
GET, DELETE, PUT → 멱등
POST → 기본적으로 멱등하지 않음 (같은 요청 두 번 = 두 개의 리소스 생성)
```

POST를 재시도 가능하게 만들려면 `Idempotency-Key`를 사용한다.

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResult> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    // 같은 idempotencyKey로 이미 처리된 결과가 있으면 반환
    return idempotencyStore.get(idempotencyKey)
            .map(cached -> ResponseEntity.ok(cached))
            .orElseGet(() -> {
                PaymentResult result = paymentService.process(request);
                idempotencyStore.store(idempotencyKey, result, Duration.ofHours(24));
                return ResponseEntity.status(HttpStatus.CREATED).body(result);
            });
}
```

클라이언트는 네트워크 실패 시 동일한 `Idempotency-Key`로 재시도하면, 서버는 중복 처리 없이 이전 결과를 반환한다.

---

## 4. 예외 vs 결과 타입: 제어 흐름으로서의 에러

### 4.1 예외를 제어 흐름에 사용하면 안 되는 이유

예외를 발생시키는 것은 비용이 크다. JVM에서 예외 생성 시 스택 트레이스를 캡처하는 작업이 수행되며, 이는 일반 메서드 호출보다 수십~수백 배 느리다.

더 큰 문제는 가독성이다.

```java
// 나쁜 예 — 예외를 제어 흐름으로 사용
public boolean isUserEligible(Long userId) {
    try {
        User user = userService.findById(userId); // 없으면 예외 던짐
        return user.getAge() >= 18;
    } catch (UserNotFoundException e) {
        return false; // 예외가 분기의 조건으로 사용됨
    }
}
```

이 코드는 "사용자가 존재하지 않을 수 있다"는 사실이 예외로 숨겨져 있다. 코드를 읽는 사람은 `findById`가 예외를 던질 수 있다는 것을 알아야 하고, 그 예외가 여기서 잡힌다는 것도 알아야 한다. 이것은 명시적이지 않다.

### 4.2 Optional: 값이 없을 수도 있다는 명시적 표현

Java의 `Optional`은 "값이 없을 수 있다"는 의도를 타입으로 표현한다.

```java
// findById는 Optional을 반환한다
public Optional<User> findById(Long userId) {
    return userRepository.findById(userId);
}

// 호출자는 값의 부재를 명시적으로 처리해야 한다
public boolean isUserEligible(Long userId) {
    return findById(userId)
            .map(user -> user.getAge() >= 18)
            .orElse(false);
}
```

`Optional`의 올바른 사용 원칙:
- 반환 타입으로만 사용한다 (파라미터, 필드에는 사용하지 않는다)
- `get()`은 `isPresent()` 확인 없이 절대 호출하지 않는다
- `orElseThrow()`로 예외 전환 시 의미 있는 예외를 사용한다

```java
// Optional에서 의미 있는 예외로 전환
User user = userRepository.findById(userId)
        .orElseThrow(() -> new ResourceNotFoundException("User", userId));
```

### 4.3 Result 타입: 성공과 실패를 일급 시민으로

Rust의 `Result<T, E>`, Haskell의 `Either` 패턴에서 영감을 받은 Result 타입은 성공과 실패를 모두 타입으로 표현한다. Java에는 기본 제공이 없지만, vavr 라이브러리나 직접 구현으로 사용할 수 있다.

```java
// 간단한 Result 타입 구현
public sealed interface Result<T> {

    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(AppException error) implements Result<T> {}

    static <T> Result<T> success(T value) {
        return new Success<>(value);
    }

    static <T> Result<T> failure(AppException error) {
        return new Failure<>(error);
    }

    default boolean isSuccess() {
        return this instanceof Success;
    }

    default T getOrThrow() {
        return switch (this) {
            case Success<T> s -> s.value();
            case Failure<T> f -> throw f.error();
        };
    }

    default <U> Result<U> map(Function<T, U> mapper) {
        return switch (this) {
            case Success<T> s -> Result.success(mapper.apply(s.value()));
            case Failure<T> f -> Result.failure(f.error());
        };
    }

    default <U> Result<U> flatMap(Function<T, Result<U>> mapper) {
        return switch (this) {
            case Success<T> s -> mapper.apply(s.value());
            case Failure<T> f -> Result.failure(f.error());
        };
    }
}
```

```java
// Result 타입 활용 예
@Service
public class TransferService {

    public Result<TransferReceipt> transfer(TransferRequest request) {
        return validateRequest(request)
                .flatMap(this::findSourceAccount)
                .flatMap(source -> findTargetAccount(request.getTargetId())
                        .map(target -> Pair.of(source, target)))
                .flatMap(pair -> deductBalance(pair.getFirst(), request.getAmount()))
                .flatMap(source -> addBalance(source, request))
                .map(this::createReceipt);
    }

    private Result<TransferRequest> validateRequest(TransferRequest request) {
        if (request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            return Result.failure(new BusinessException(
                    ErrorCode.INVALID_INPUT, "이체 금액은 0보다 커야 합니다"));
        }
        return Result.success(request);
    }
}

// 컨트롤러에서의 처리
@PostMapping("/transfer")
public ResponseEntity<?> transfer(@RequestBody TransferRequest request) {
    Result<TransferReceipt> result = transferService.transfer(request);
    return switch (result) {
        case Result.Success<TransferReceipt> s ->
                ResponseEntity.ok(s.value());
        case Result.Failure<TransferReceipt> f ->
                throw f.error(); // GlobalExceptionHandler가 처리
    };
}
```

### 4.4 예외와 Result 타입의 선택 기준

```
어떤 것을 써야 하는가?

예외가 적합한 경우:
  - 호출자가 거의 항상 성공을 기대하는 경우
  - 실패가 진짜 예외적인 상황인 경우 (DB 다운, 디스크 꽉 참)
  - 여러 레이어를 관통해 전파되어야 하는 경우

Result 타입이 적합한 경우:
  - 실패가 정상적인 비즈니스 흐름의 일부인 경우
  - 호출자가 성공/실패에 따라 다른 행동을 해야 하는 경우
  - 함수형 체이닝으로 파이프라인을 구성하는 경우
```

---

## 5. 방어적 프로그래밍: 나쁜 데이터가 시스템을 죽이기 전에

### 5.1 입력 검증의 계층

외부에서 들어오는 모든 데이터는 신뢰하지 않는다. 이것은 보안뿐 아니라 안정성의 문제다.

```
┌──────────────────────────────────┐
│           Controller Layer        │  ← HTTP 요청 파라미터 검증 (Bean Validation)
├──────────────────────────────────┤
│           Service Layer           │  ← 비즈니스 규칙 검증
├──────────────────────────────────┤
│           Domain Layer            │  ← 도메인 불변식(Invariant) 검증
├──────────────────────────────────┤
│        Repository Layer           │  ← DB 제약 조건
└──────────────────────────────────┘
```

각 계층이 자신이 담당하는 검증을 수행한다. 상위 계층이 검증했다고 하위 계층이 믿어서는 안 된다.

```java
// 컨트롤러 계층: Bean Validation
public record CreateUserRequest(
    @NotBlank(message = "이름은 필수입니다")
    @Size(min = 2, max = 50, message = "이름은 2자 이상 50자 이하여야 합니다")
    String name,

    @NotBlank(message = "이메일은 필수입니다")
    @Email(message = "올바른 이메일 형식이 아닙니다")
    String email,

    @NotNull(message = "나이는 필수입니다")
    @Min(value = 0, message = "나이는 0 이상이어야 합니다")
    @Max(value = 150, message = "나이가 올바르지 않습니다")
    Integer age
) {}

// 도메인 계층: 불변식 검증
public class User {
    private final String email;
    private final int age;

    public User(String email, int age) {
        // 도메인 객체는 자신의 불변식을 스스로 지킨다
        Objects.requireNonNull(email, "이메일은 null일 수 없습니다");
        if (!isValidEmail(email)) {
            throw new IllegalArgumentException("올바르지 않은 이메일: " + email);
        }
        if (age < 0 || age > 150) {
            throw new IllegalArgumentException("올바르지 않은 나이: " + age);
        }
        this.email = email;
        this.age = age;
    }
}
```

### 5.2 Null 방어: NullPointerException은 예방 가능하다

NPE는 Java에서 가장 흔한 런타임 예외다. 대부분은 방어적 코딩으로 예방할 수 있다.

```java
// @NonNull 어노테이션으로 의도 명시 (Lombok, Spring, Jakarta)
public OrderService(@NonNull OrderRepository orderRepository,
                    @NonNull UserService userService) {
    this.orderRepository = orderRepository; // Lombok이 null 체크 코드 생성
    this.userService = userService;
}

// 메서드 진입 시 빠른 실패(Fail-Fast)
public Order cancelOrder(Long orderId, String reason) {
    Objects.requireNonNull(orderId, "orderId는 null일 수 없습니다");
    Objects.requireNonNull(reason, "취소 사유는 필수입니다");

    // 이하 로직에서 orderId, reason이 null이 아님을 보장
    Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order", orderId));
    // ...
}
```

### 5.3 Fail-Fast 원칙

문제를 발견한 즉시 실패하는 것이 나중에 조용히 실패하는 것보다 훨씬 낫다.

```java
// 나쁜 예 — 잘못된 값으로 계속 진행
public BigDecimal calculateDiscount(int quantity, BigDecimal price) {
    if (quantity < 0) {
        quantity = 0; // 조용히 수정
    }
    return price.multiply(BigDecimal.valueOf(quantity * 0.1));
}

// 좋은 예 — 즉시 실패
public BigDecimal calculateDiscount(int quantity, BigDecimal price) {
    if (quantity < 0) {
        throw new IllegalArgumentException("수량은 음수일 수 없습니다: " + quantity);
    }
    if (price.compareTo(BigDecimal.ZERO) < 0) {
        throw new IllegalArgumentException("가격은 음수일 수 없습니다: " + price);
    }
    return price.multiply(BigDecimal.valueOf(quantity * 0.1));
}
```

나쁜 예에서 버그가 발생하면 "왜 할인이 0원이지?"라는 디버깅 지옥이 시작된다. 좋은 예에서는 예외와 함께 즉시 원인이 드러난다.

### 5.4 방어적 복사: 변경 가능한 객체의 캡슐화

```java
// 나쁜 예 — 외부에서 내부 상태를 변경할 수 있음
public class Order {
    private final List<OrderItem> items;

    public Order(List<OrderItem> items) {
        this.items = items; // 외부 리스트의 참조를 그대로 저장
    }

    public List<OrderItem> getItems() {
        return items; // 내부 리스트를 직접 노출
    }
}

// Order 밖에서 내부 상태 변경 가능
List<OrderItem> items = new ArrayList<>();
Order order = new Order(items);
items.add(fakeItem); // order 내부 상태가 변경됨!
order.getItems().clear(); // order 내부 상태를 비움!

// 좋은 예 — 방어적 복사
public class Order {
    private final List<OrderItem> items;

    public Order(List<OrderItem> items) {
        this.items = List.copyOf(items); // 불변 복사본 저장
    }

    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items); // 읽기 전용 뷰 반환
    }
}
```

### 5.5 에러 로깅 전략: 신호 대 잡음비

모든 예외를 ERROR로 로깅하면 정작 중요한 에러가 노이즈에 묻힌다.

```java
// 로그 레벨 전략
// ERROR: 즉각적인 대응이 필요한 상황 (알림 연동 대상)
log.error("결제 처리 실패 — 즉각 확인 필요: orderId={}", orderId, ex);

// WARN: 비정상적이지만 자동 복구 가능한 상황
log.warn("외부 API 호출 실패, 재시도 예정: attempt={}, endpoint={}", attempt, endpoint);

// INFO: 중요한 비즈니스 이벤트
log.info("주문 생성: orderId={}, userId={}, amount={}", orderId, userId, amount);

// DEBUG: 개발/디버깅용 상세 정보 (프로덕션에서는 비활성화)
log.debug("캐시 미스: key={}", cacheKey);
```

4xx 에러(클라이언트 잘못)를 ERROR로 찍으면 알림 피로도가 높아진다.

클라이언트가 잘못된 요청을 보내는 것은 서버가 해결할 문제가 아니다. WARN으로 찍거나, 빈도가 높으면 아예 DEBUG로 처리하는 것이 나을 수 있다.

### 5.6 예외 연쇄(Exception Chaining): 원인을 잃지 마라

하위 계층의 예외를 상위 계층 예외로 래핑할 때 원인을 반드시 유지해야 한다.

```java
// 나쁜 예 — 원인을 잃음
try {
    jdbcTemplate.update(sql, params);
} catch (DataAccessException e) {
    throw new InfrastructureException(ErrorCode.DATABASE_ERROR, "DB 저장 실패");
    // 원본 DataAccessException이 사라짐 — 디버깅이 어려워짐
}

// 좋은 예 — 원인을 유지
try {
    jdbcTemplate.update(sql, params);
} catch (DataAccessException e) {
    throw new InfrastructureException(ErrorCode.DATABASE_ERROR, "DB 저장 실패", e);
    // cause에 원본 예외가 연결됨 — 로그에서 전체 스택 트레이스 확인 가능
}
```

로그에서 `Caused by:`가 보이는 이유가 바로 이 원인 연쇄 덕분이다. 원인을 잃으면 "왜 DB 에러가 났는지"를 알 수 없다.

---

## 6. 종합: 에러 처리 아키텍처 전체 그림

지금까지 다룬 내용을 하나의 흐름으로 정리하면 다음과 같다.

```
[HTTP 요청]
    │
    ▼
[Controller]
    │  Bean Validation (@Valid)
    │  MethodArgumentNotValidException → GlobalExceptionHandler (400)
    │
    ▼
[Service]
    │  비즈니스 규칙 검증
    │  BusinessException (도메인 예외)
    │
    ├─── [외부 API 호출]
    │         │  일시적 실패 → RetryPolicy (Exponential Backoff + Jitter)
    │         │  최종 실패 → InfrastructureException
    │
    ├─── [DB 접근]
    │         │  DataAccessException → InfrastructureException으로 래핑
    │
    ▼
[GlobalExceptionHandler]
    │
    ├── ResourceNotFoundException → 404 Problem Detail
    ├── BusinessException         → 4xx Problem Detail
    ├── InfrastructureException   → 5xx Problem Detail (상세 정보 숨김)
    └── Exception (기타)          → 500 Problem Detail + ERROR 로그

[HTTP 응답] → application/problem+json (RFC 7807)
```

이 아키텍처에서:
- **예외는 의미를 가진다** — 예외 타입만 보면 원인과 처리 방법이 보인다
- **에러 응답은 표준화된다** — 클라이언트는 항상 같은 형식을 기대할 수 있다
- **재시도는 안전하다** — 멱등성이 보장되고, Jitter로 thundering herd를 막는다
- **원인은 추적 가능하다** — traceId와 예외 연쇄로 로그에서 전체 맥락을 볼 수 있다
- **내부 정보는 보호된다** — 스택 트레이스, SQL, 시스템 경로가 응답에 노출되지 않는다

에러 처리는 코드의 "행복한 경로"만큼 중요하다. 오히려 시스템의 성숙도는 실패 시 어떻게 동작하는지에서 드러난다.

처음부터 에러를 일급 시민으로 설계하는 팀과 나중에 땜질하는 팀의 차이는 프로덕션에서 명확하게 나타난다.

---

## 참고 자료

- **"Effective Java, 3rd Edition"** — Joshua Bloch. 아이템 69-77이 예외 처리에 집중. Checked vs Unchecked 판단 기준의 원전이라 할 수 있다.
- **RFC 7807: Problem Details for HTTP APIs** — IETF (Nottingham, Wilde). 에러 응답 표준화의 근거 문서. https://www.rfc-editor.org/rfc/rfc7807
- **"Release It!, 2nd Edition"** — Michael T. Nygard. Bulkhead, Circuit Breaker, 재시도 패턴을 실제 장애 사례와 함께 설명한다. 안정성 패턴의 바이블.
- **"Designing Distributed Systems"** — Brendan Burns. 분산 환경에서의 실패 처리, 멱등성, 재시도 전략을 컨테이너 중심으로 설명한다.
- **AWS 공식 블로그 — "Exponential Backoff And Jitter"** — Marc Brooker. Full Jitter, Decorrelated Jitter 비교 실험 포함. https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
- **"Domain-Driven Design"** — Eric Evans. 도메인 예외 계층 설계와 불변식(Invariant) 개념의 기반이 되는 고전.
