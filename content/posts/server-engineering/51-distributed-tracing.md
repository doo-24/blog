---
title: "[로그, 모니터링, 관측 가능성] 4편 — 분산 트레이싱: 마이크로서비스 요청 추적"
date: 2026-03-17T19:02:00+09:00
draft: false
tags: ["OpenTelemetry", "Jaeger", "분산 트레이싱", "Trace", "서버"]
series: ["로그, 모니터링, 관측 가능성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 7
summary: "분산 트레이싱 개념(Trace, Span, Context Propagation), OpenTelemetry 표준과 자동/수동 계측, Jaeger/Zipkin 설정과 트레이스 분석, 샘플링 전략과 성능 오버헤드 관리까지"
---

마이크로서비스 아키텍처에서 하나의 HTTP 요청은 수십 개의 서비스를 거쳐 처리된다. 결제 API를 호출했을 때 응답이 3초나 걸렸다면, 문제는 인증 서비스인가, 재고 서비스인가, 아니면 그 사이의 메시지 큐인가? 로그를 뒤지는 것만으로는 이 질문에 답할 수 없다. 분산 트레이싱은 바로 이 "요청이 어디서 얼마나 걸렸는가"를 시각화하고 추적하는 기술이다.

---

## 분산 트레이싱의 핵심 개념

### Trace, Span, 그리고 그 관계

**Trace**는 하나의 요청이 시스템 전체를 통과하는 전체 여정이다.

**Span**은 그 여정에서 하나의 작업 단위다. "주문 서비스에서 DB 쿼리를 실행한 0.3초"가 하나의 Span이다.

Trace는 여러 Span의 트리 구조로 이루어진다. 최상위 Span을 **Root Span**이라 부르고, 그 아래에 자식 Span들이 중첩된다.

```
Trace ID: abc-123
│
└─ [Root Span] API Gateway - /api/orders (총 1850ms)
   ├─ [Span] Auth Service - validateToken (120ms)
   ├─ [Span] Order Service - createOrder (1500ms)
   │  ├─ [Span] DB - INSERT orders (80ms)
   │  ├─ [Span] Inventory Service - checkStock (900ms)
   │  │  └─ [Span] Cache - Redis GET (5ms)
   │  └─ [Span] Payment Service - charge (500ms)
   └─ [Span] Notification Service - sendEmail (80ms)
```

이 구조 하나로 병목이 Inventory Service의 900ms임을 즉시 파악할 수 있다.

### Span의 내부 구조

각 Span은 다음 정보를 담는다.

```
Span {
  traceId:    "abc-123-def-456"   // 전체 Trace 식별자
  spanId:     "7f8a9b0c"          // 이 Span의 식별자
  parentSpanId: "3d4e5f6a"        // 부모 Span (없으면 Root)
  operationName: "inventory.checkStock"
  startTime:  1710000000000       // Unix milliseconds
  duration:   900                 // ms
  status:     OK | ERROR
  attributes: {                   // 사용자 정의 메타데이터
    "http.method": "GET",
    "http.url": "/api/inventory/SKU-001",
    "db.statement": "SELECT ...",
    "user.id": "usr-789"
  }
  events: [                       // Span 내 시점 기록
    { time: +50ms, name: "cache_miss" },
    { time: +55ms, name: "db_query_start" }
  ]
}
```

`attributes`와 `events`가 트레이싱의 진짜 가치를 만든다. 단순한 타이밍 정보를 넘어 "어떤 쿼리가 느렸는가", "어떤 사용자의 요청인가"까지 추적할 수 있다.

### Context Propagation

분산 트레이싱의 가장 핵심적인 메커니즘이다.

서비스 A가 서비스 B를 호출할 때, Trace 정보를 HTTP 헤더에 담아 전달한다. 받는 쪽은 이 헤더를 읽어 동일한 Trace 하위의 새 Span을 생성한다.

**W3C Trace Context** 표준 헤더:

W3C가 표준화한 이 헤더 형식 덕분에 서로 다른 벤더의 계측 라이브러리를 사용하는 서비스들도 Trace를 이어받을 수 있다.

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate:  vendor1=opaqueValue1
```

`traceparent` 형식: `{version}-{traceId}-{parentSpanId}-{flags}`

`flags`의 마지막 비트는 샘플링 여부다. `01`이면 이 요청은 수집 대상이다.

이 헤더가 없으면 서비스는 새 Trace를 시작한다. 있으면 기존 Trace를 이어받는다.

**Baggage**는 Trace Context와 함께 전파되는 key-value 데이터다. `userId`나 `sessionId`처럼 모든 Span에 공통으로 붙여야 하는 값을 한 번만 설정하면 자동으로 전파된다.

---

## OpenTelemetry 표준

### OpenTelemetry가 해결한 문제

과거에는 Zipkin, Jaeger, Datadog, New Relic 각각의 SDK를 따로 써야 했다. 벤더 변경 시 계측 코드 전체를 다시 작성해야 했다.

OpenTelemetry(OTel)는 이 문제를 해결하는 **벤더 중립적 표준**이다. 계측 코드는 한 번만 작성하고, 백엔드(Jaeger, Zipkin, Datadog 등)는 설정만으로 교체한다.

OTel의 핵심 구성:

```
애플리케이션 코드
    ↓ (계측)
OTel API/SDK
    ↓ (수집)
OTel Collector
    ↓ (내보내기)
Jaeger / Zipkin / Datadog / ...
```

### OTel 핵심 컴포넌트

**TracerProvider**: Tracer 인스턴스를 생성하는 팩토리다. 전역 싱글톤으로 설정한다.

**Tracer**: Span을 생성하는 주체다. 서비스당 하나를 사용하는 것이 일반적이다.

**SpanProcessor**: Span이 종료될 때 처리 방식을 정의한다. `BatchSpanProcessor`가 프로덕션 표준이다.

**Exporter**: 수집된 Span 데이터를 외부로 내보낸다. OTLP, Jaeger, Zipkin 등 다양한 프로토콜을 지원한다.

### Java/Spring Boot 자동 계측 설정

자동 계측(Auto Instrumentation)은 코드 변경 없이 JVM 에이전트로 동작한다. 가장 빠른 시작 방법이다.

**의존성 추가** (`pom.xml`):

```xml
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>2.3.0</version>
</dependency>
```

**application.yml 설정**:

```yaml
otel:
  service:
    name: order-service
  exporter:
    otlp:
      endpoint: http://otel-collector:4318
      protocol: http/protobuf
  traces:
    exporter: otlp
  metrics:
    exporter: otlp
  logs:
    exporter: otlp
  propagators: tracecontext,baggage
  instrumentation:
    spring-webmvc:
      enabled: true
    jdbc:
      enabled: true
    redis:
      enabled: true
    kafka:
      enabled: true
```

JVM 에이전트 방식 (코드 변경 불필요):

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-service \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4318 \
     -jar app.jar
```

이것만으로 HTTP 요청, DB 쿼리, Redis 호출, Kafka 메시지 등 대부분의 Span이 자동 생성된다.

### 수동 계측: 비즈니스 로직 추적

자동 계측이 잡지 못하는 비즈니스 로직은 수동으로 계측한다.

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final Tracer tracer;  // OTel Tracer 주입
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;

    public Order createOrder(CreateOrderRequest request) {
        // 현재 활성 Span에 비즈니스 속성 추가
        Span currentSpan = Span.current();
        currentSpan.setAttribute("order.userId", request.getUserId());
        currentSpan.setAttribute("order.itemCount", request.getItems().size());

        // 재고 확인 - 별도 Span으로 추적
        Span inventorySpan = tracer.spanBuilder("inventory.validate")
                .setSpanKind(SpanKind.INTERNAL)
                .startSpan();

        try (Scope scope = inventorySpan.makeCurrent()) {
            for (OrderItem item : request.getItems()) {
                inventorySpan.setAttribute("item.sku", item.getSku());
                boolean available = inventoryClient.checkStock(item.getSku(), item.getQty());
                if (!available) {
                    inventorySpan.setStatus(StatusCode.ERROR, "Out of stock: " + item.getSku());
                    throw new OutOfStockException(item.getSku());
                }
            }
        } catch (Exception e) {
            inventorySpan.recordException(e);
            throw e;
        } finally {
            inventorySpan.end();
        }

        // 결제 처리
        Span paymentSpan = tracer.spanBuilder("payment.charge")
                .setSpanKind(SpanKind.CLIENT)
                .startSpan();

        try (Scope scope = paymentSpan.makeCurrent()) {
            paymentSpan.addEvent("payment_started", Attributes.of(
                    AttributeKey.doubleKey("amount"), request.getTotalAmount()
            ));

            PaymentResult result = paymentClient.charge(request.getPaymentInfo());

            paymentSpan.addEvent("payment_completed");
            paymentSpan.setAttribute("payment.transactionId", result.getTransactionId());

            return buildOrder(request, result);
        } catch (PaymentException e) {
            paymentSpan.setStatus(StatusCode.ERROR, e.getMessage());
            paymentSpan.recordException(e);
            throw e;
        } finally {
            paymentSpan.end();
        }
    }
}
```

**`@WithSpan` 어노테이션**: 더 간단한 수동 계측 방법이다.

```java
@Service
public class PricingService {

    @WithSpan("pricing.calculate")
    public BigDecimal calculatePrice(
            @SpanAttribute("item.sku") String sku,
            @SpanAttribute("item.qty") int quantity) {
        // 이 메서드 실행 전체가 하나의 Span으로 자동 래핑됨
        return basePrice(sku).multiply(BigDecimal.valueOf(quantity));
    }
}
```

### Baggage를 통한 컨텍스트 전파

사용자 ID처럼 전체 추적 흐름에서 공통으로 필요한 데이터는 Baggage로 전파한다.

```java
@Component
public class RequestContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String userId = httpRequest.getHeader("X-User-Id");

        if (userId != null) {
            // Baggage에 userId 설정 - 하위 모든 Span에 자동 전파
            Baggage baggage = Baggage.current().toBuilder()
                    .put("user.id", userId)
                    .build();

            // 현재 Span에도 속성 추가
            Span.current().setAttribute("user.id", userId);

            try (BaggageScope scope = baggage.makeCurrent()) {
                chain.doFilter(request, response);
            }
        } else {
            chain.doFilter(request, response);
        }
    }
}
```

---

## Jaeger 설정과 트레이스 분석

### OTel Collector 아키텍처

직접 Jaeger로 보내는 것보다 OTel Collector를 중간에 두는 것이 프로덕션 표준이다.

```
서비스들 → OTel Collector → Jaeger (시각화)
                          → Prometheus (메트릭)
                          → Elasticsearch (로그)
```

Collector를 쓰면 백엔드를 교체해도 애플리케이션 코드를 건드리지 않아도 된다.

**OTel Collector 설정** (`otel-collector-config.yaml`):

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
    send_batch_max_size: 2048

  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

  # 민감 정보 제거
  attributes/redact:
    actions:
      - key: "db.statement"
        action: update
        value: "[REDACTED]"
      - key: "http.request.header.authorization"
        action: delete

  # 서비스 이름 정규화
  resource:
    attributes:
      - key: service.environment
        value: "production"
        action: upsert

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true

  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, attributes/redact, resource]
      exporters: [jaeger, otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

**Docker Compose 통합 구성**:

```yaml
version: "3.9"

services:
  jaeger:
    image: jaegertracing/all-in-one:1.56
    ports:
      - "16686:16686"   # Jaeger UI
      - "14250:14250"   # gRPC (Collector -> Jaeger)
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
      SPAN_STORAGE_TYPE: badger
      BADGER_EPHEMERAL: "false"
      BADGER_DIRECTORY_VALUE: /badger/data
      BADGER_DIRECTORY_KEY: /badger/key
    volumes:
      - jaeger-data:/badger

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.97.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus metrics
    depends_on:
      - jaeger

volumes:
  jaeger-data:
```

### Jaeger UI에서 트레이스 분석

**지연 시간 이상 탐지**: Jaeger UI의 "Search" 탭에서 `minDuration=1s`로 필터링하면 느린 요청만 골라볼 수 있다.

**에러 트레이스 탐지**: `tags=error:true` 필터로 에러가 발생한 요청만 추려낸다.

**Trace Comparison**: Jaeger는 두 Trace를 나란히 비교하는 기능을 제공한다. 배포 전/후 성능을 직접 비교할 수 있다.

**Service Dependency Graph**: Jaeger UI의 "System Architecture" 탭은 서비스 간 호출 관계를 자동으로 시각화한다. 예상치 못한 의존성을 발견하는 데 유용하다.

### 실전 트레이스 분석 패턴

**N+1 쿼리 탐지**: 루프 안에서 DB 호출이 반복되면 Trace에서 같은 Span이 수십 번 반복되는 패턴으로 나타난다.

```
Order Service (2000ms)
├── DB SELECT orders (5ms)
├── DB SELECT user WHERE id=1 (4ms)   ← N+1 시작
├── DB SELECT user WHERE id=2 (4ms)
├── DB SELECT user WHERE id=3 (4ms)
...
└── DB SELECT user WHERE id=100 (4ms)  ← 총 100번 반복
```

이 패턴을 보면 즉시 `JOIN`이나 배치 조회로 개선해야 함을 알 수 있다.

**직렬 호출 vs 병렬 호출**: Trace 뷰에서 Span들이 순차적으로 나열되면 병렬화 기회가 있다는 신호다.

```
Before (순차, 총 900ms):
├── [Inventory Check   300ms]
├──────────────────── [User Profile  300ms]
└────────────────────────────────── [Price Calc  300ms]

After (병렬, 총 300ms):
├── [Inventory Check   300ms]
├── [User Profile      300ms]
└── [Price Calc        300ms]
```

---

## 샘플링 전략과 성능 오버헤드 관리

### 왜 샘플링이 필요한가

초당 10,000 요청을 처리하는 서비스에서 모든 요청을 추적하면 어떻게 될까?

트레이싱 자체의 CPU 오버헤드는 약 1~3%이지만, 네트워크와 스토리지 비용이 폭발한다. Span 하나에 평균 1KB라고 하면 초당 10MB, 하루 864GB다.

샘플링은 "어떤 요청을 추적할 것인가"를 결정하는 전략이다.

### Head-based Sampling (헤드 기반 샘플링)

요청이 시작될 때 추적 여부를 결정한다.

**장점**: 구현이 단순하고 오버헤드가 낮다.

**단점**: 에러나 지연이 발생할지 미리 알 수 없으므로, 중요한 요청을 놓칠 수 있다.

OTel SDK 설정:

```java
@Configuration
public class TracingConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        // 확률적 샘플링: 10%만 추적
        Sampler sampler = Sampler.traceIdRatioBased(0.1);

        // 또는 ParentBased 샘플링: 부모의 결정을 따름
        Sampler parentBasedSampler = Sampler.parentBased(
                Sampler.traceIdRatioBased(0.1)
        );

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .setSampler(parentBasedSampler)
                .addSpanProcessor(BatchSpanProcessor.builder(
                        OtlpGrpcSpanExporter.builder()
                                .setEndpoint("http://otel-collector:4317")
                                .build()
                ).build())
                .build();

        return OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .setPropagators(ContextPropagators.create(
                        W3CTraceContextPropagator.getInstance()
                ))
                .buildAndRegisterGlobal();
    }
}
```

### Tail-based Sampling (테일 기반 샘플링)

요청이 완료된 후 전체 Trace를 보고 수집 여부를 결정한다.

**장점**: 에러나 지연이 있는 요청을 100% 수집하고, 정상 요청만 샘플링한다.

**단점**: Trace 전체를 메모리에 잠시 유지해야 하므로 OTel Collector 메모리를 더 사용한다.

OTel Collector에서 설정:

```yaml
processors:
  tail_sampling:
    # 의사결정 대기 시간 (Trace 완료 대기)
    decision_wait: 10s
    num_traces: 50000

    policies:
      # 에러 요청은 100% 수집
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]

      # 느린 요청은 100% 수집 (500ms 이상)
      - name: slow-requests-policy
        type: latency
        latency:
          threshold_ms: 500

      # 특정 서비스는 100% 수집
      - name: payment-service-policy
        type: string_attribute
        string_attribute:
          key: service.name
          values: ["payment-service"]

      # 나머지는 5% 확률 샘플링
      - name: default-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 5
```

### 커스텀 샘플링: 중요 요청 강제 수집

비즈니스적으로 중요한 요청(VIP 사용자, 결제 요청)은 항상 추적해야 한다.

```java
public class BusinessAwareSampler implements Sampler {

    @Override
    public SamplingResult shouldSample(
            Context parentContext,
            String traceId,
            String name,
            SpanKind spanKind,
            Attributes attributes,
            List<LinkData> parentLinks) {

        // 결제 관련 요청은 항상 수집
        String httpTarget = attributes.get(SemanticAttributes.HTTP_TARGET);
        if (httpTarget != null && httpTarget.startsWith("/api/payment")) {
            return SamplingResult.create(
                    SamplingDecision.RECORD_AND_SAMPLE,
                    Attributes.of(
                            AttributeKey.booleanKey("force.sampled"), true
                    )
            );
        }

        // VIP 사용자 요청은 항상 수집
        String userId = attributes.get(AttributeKey.stringKey("user.id"));
        if (userId != null && isVipUser(userId)) {
            return SamplingResult.recordAndSample();
        }

        // 나머지는 10% 샘플링
        return Sampler.traceIdRatioBased(0.1)
                .shouldSample(parentContext, traceId, name, spanKind, attributes, parentLinks);
    }

    @Override
    public String getDescription() {
        return "BusinessAwareSampler";
    }
}
```

### 성능 오버헤드 측정과 최적화

**BatchSpanProcessor 튜닝**: 기본값보다 배치 크기를 키우면 네트워크 호출 횟수가 줄어든다.

```java
BatchSpanProcessor processor = BatchSpanProcessor.builder(exporter)
        .setMaxQueueSize(8192)          // 큐 크기 (기본값: 2048)
        .setMaxExportBatchSize(1024)    // 배치 크기 (기본값: 512)
        .setExporterTimeout(Duration.ofSeconds(30))
        .setScheduleDelay(Duration.ofMillis(200))  // 내보내기 주기
        .build();
```

**오버헤드 벤치마크** 결과 (실측 기준):

| 설정 | CPU 오버헤드 | 메모리 오버헤드 | 지연 추가 |
|------|------------|--------------|---------|
| 자동 계측 + 100% 샘플링 | ~3% | ~50MB | ~0.5ms |
| 자동 계측 + 10% 샘플링 | ~0.5% | ~20MB | ~0.05ms |
| 수동 계측 + 1% 샘플링 | < 0.1% | ~10MB | 무시 가능 |

프로덕션에서 10% 이하 샘플링이 대부분의 경우 충분하다. 에러/지연 요청은 Tail-based Sampling으로 보완한다.

---

## 안티패턴: 흔히 저지르는 실수들

### 안티패턴 1: Span에 민감 정보 기록

```java
// 잘못된 예
span.setAttribute("user.password", request.getPassword());
span.setAttribute("credit.card.number", payment.getCardNumber());
span.setAttribute("db.statement", "SELECT * FROM users WHERE password='" + pwd + "'");
```

Jaeger/Zipkin은 별도의 접근 제어 없이 모든 Span 데이터를 볼 수 있다. 민감 정보는 절대 Span에 넣지 않는다.

올바른 방법:

```java
// 올바른 예
span.setAttribute("user.id", user.getId());          // ID만, 비밀번호 X
span.setAttribute("credit.card.last4", card.getLast4());  // 마지막 4자리만
span.setAttribute("db.operation", "SELECT users");         // 쿼리 템플릿만
```

### 안티패턴 2: 모든 것에 Span을 붙이기

```java
// 과도한 계측 - 성능 저하 유발
public List<Product> getProducts() {
    Span span1 = tracer.spanBuilder("list.create").startSpan();
    List<Product> result = new ArrayList<>();
    span1.end();

    for (Product p : dbProducts) {
        Span span2 = tracer.spanBuilder("item.add").startSpan();
        result.add(p);  // 수십만 번 실행될 수 있는 단순 연산
        span2.end();
    }
    return result;
}
```

Span은 의미 있는 작업 단위에만 사용한다. DB 쿼리, 외부 API 호출, 캐시 조회처럼 I/O가 발생하거나 비즈니스적으로 의미 있는 곳에만 붙인다.

### 안티패턴 3: Context를 쓰레드 경계에서 잃어버리기

```java
// 잘못된 예 - Trace Context가 쓰레드를 넘으면 끊긴다
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(() -> {
    // 여기서 새 Span을 만들면 부모 Span과 연결이 끊김
    Span span = tracer.spanBuilder("async.work").startSpan();
    doWork();
    span.end();
});
```

올바른 방법: Context를 명시적으로 전달한다.

```java
// 올바른 예
Context currentContext = Context.current();  // 현재 Context 캡처

executor.submit(() -> {
    // 캡처한 Context를 사용해 Span 생성
    Span span = tracer.spanBuilder("async.work")
            .setParent(currentContext)  // 명시적 부모 설정
            .startSpan();
    try (Scope scope = span.makeCurrent()) {
        doWork();
    } finally {
        span.end();
    }
});
```

### 안티패턴 4: 에러를 기록하지 않고 Span 종료

```java
// 잘못된 예 - 에러가 발생해도 Span 상태가 OK
try {
    result = callExternalService();
} catch (Exception e) {
    throw e;  // Span에 에러를 기록하지 않음
} finally {
    span.end();
}
```

에러는 반드시 Span에 기록한다:

```java
// 올바른 예
try {
    result = callExternalService();
} catch (Exception e) {
    span.setStatus(StatusCode.ERROR, e.getMessage());
    span.recordException(e);  // 스택 트레이스까지 저장
    throw e;
} finally {
    span.end();
}
```

---

## 로그와 트레이싱 연결 (Correlation)

트레이싱의 진짜 힘은 로그와 연결할 때 나온다.

로그 메시지에 `traceId`와 `spanId`를 포함시키면, Jaeger에서 Span을 보다가 해당 시점의 로그를 바로 조회할 수 있다.

**Logback MDC 자동 주입** (OTel Java 에이전트가 자동으로 처리):

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service": "order-service"}</customFields>
            <!-- OTel 에이전트가 자동으로 MDC에 traceId, spanId를 주입 -->
        </encoder>
    </appender>
</configuration>
```

출력 로그 예시:

```json
{
  "timestamp": "2026-03-17T10:00:00.123Z",
  "level": "ERROR",
  "message": "재고 부족: SKU-001",
  "service": "order-service",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "userId": "usr-789"
}
```

Jaeger에서 느린 Span을 발견 → `traceId`로 Elasticsearch/Loki에서 해당 로그 즉시 조회 → 원인 파악. 이것이 현대적인 관측 가능성의 실전 워크플로다.

---

## 참고 자료

- [OpenTelemetry 공식 문서](https://opentelemetry.io/docs/) — API/SDK 레퍼런스, 자동 계측 설정 가이드
- [Jaeger 공식 문서](https://www.jaegertracing.io/docs/) — 배포 아키텍처, 스토리지 백엔드 설정
- [W3C Trace Context 표준](https://www.w3.org/TR/trace-context/) — traceparent/tracestate 헤더 명세
- [OpenTelemetry Java 계측 가이드](https://opentelemetry.io/docs/instrumentation/java/) — Spring Boot 통합, 수동 계측 예제
- Dapper, a Large-Scale Distributed Systems Tracing Infrastructure (Google, 2010) — 분산 트레이싱의 기원이 된 구글 논문
- [OTel Collector 설정 가이드](https://opentelemetry.io/docs/collector/configuration/) — Tail Sampling Processor, Batch Processor 설정 참고
