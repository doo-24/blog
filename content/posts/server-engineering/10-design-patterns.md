---
title: "[서버 애플리케이션 설계] 2편 — 디자인 패턴 실전: 서버 코드에서 자주 만나는 패턴들"
date: 2026-03-18T00:08:00+09:00
draft: false
tags: ["디자인패턴", "객체지향", "설계", "서버", "아키텍처"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "Factory, Builder, Singleton, Strategy, Observer, Decorator 등 서버 코드에서 실제로 마주치는 패턴들을 실전 사례 중심으로 다루고, 패턴 남용의 위험까지 짚는다."
---

## 들어가며

디자인 패턴은 GoF(Gang of Four)의 23개 패턴으로 유명하지만, 실무에서 자주 쓰이는 패턴은 그중 절반도 되지 않는다.

나머지는 특수한 상황에서만 의미 있거나, 언어/프레임워크가 이미 해결해준다.

이번 편에서는 서버 개발에서 **실제로 자주 만나는 패턴**에 집중한다. 각 패턴이 어떤 문제를 해결하는지, 프레임워크 안에서 어떻게 녹아 있는지, 그리고 언제 직접 적용해야 하는지를 다룬다.

---

## 1. 생성 패턴

### Factory Method — 객체 생성의 위임

**문제**: 조건에 따라 다른 타입의 객체를 생성해야 하는데, 생성 로직이 여기저기 흩어져 있다.

```java
// 위반: 생성 로직이 비즈니스 로직에 섞임
public class NotificationService {
    public void notify(User user, String message) {
        Notification notification;
        if (user.prefersEmail()) {
            notification = new EmailNotification(user.getEmail(), message);
        } else if (user.prefersSms()) {
            notification = new SmsNotification(user.getPhone(), message);
        } else {
            notification = new PushNotification(user.getDeviceToken(), message);
        }
        notification.send();
    }
}
```

```java
// Factory Method 적용
public class NotificationFactory {
    public Notification create(User user, String message) {
        if (user.prefersEmail()) {
            return new EmailNotification(user.getEmail(), message);
        } else if (user.prefersSms()) {
            return new SmsNotification(user.getPhone(), message);
        }
        return new PushNotification(user.getDeviceToken(), message);
    }
}

public class NotificationService {
    private final NotificationFactory factory;

    public void notify(User user, String message) {
        factory.create(user, message).send();
    }
}
```

**프레임워크에서의 Factory**:
- Spring의 `BeanFactory` — 빈 생성과 관리의 핵심
- `LoggerFactory.getLogger()` — SLF4J의 로거 생성
- `ConnectionFactory` — JDBC, JMS 등에서 연결 객체 생성

### Builder — 복잡한 객체의 단계적 구성

**문제**: 생성자 매개변수가 많고, 선택적 필드가 있으며, 조합에 따라 유효성이 달라진다.

```java
// Telescoping Constructor 안티패턴
new HttpRequest("GET", "/api/users", null, null, 30000, true, null, "application/json");
// 7번째 null이 뭐지? 읽을 수 없다

// Builder 패턴
HttpRequest request = HttpRequest.builder()
    .method("GET")
    .url("/api/users")
    .timeout(Duration.ofSeconds(30))
    .followRedirects(true)
    .header("Accept", "application/json")
    .build();
```

**실전 Builder 구현:**

```java
public class QueryFilter {
    private final String keyword;
    private final LocalDate from;
    private final LocalDate to;
    private final int page;
    private final int size;
    private final SortOrder sort;

    private QueryFilter(Builder builder) {
        this.keyword = builder.keyword;
        this.from = builder.from;
        this.to = builder.to;
        this.page = builder.page;
        this.size = builder.size;
        this.sort = builder.sort;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private String keyword;
        private LocalDate from;
        private LocalDate to;
        private int page = 0;         // 기본값
        private int size = 20;        // 기본값
        private SortOrder sort = SortOrder.DESC;

        public Builder keyword(String keyword) {
            this.keyword = keyword;
            return this;
        }

        public Builder dateRange(LocalDate from, LocalDate to) {
            if (from.isAfter(to)) throw new IllegalArgumentException("시작일이 종료일보다 늦습니다");
            this.from = from;
            this.to = to;
            return this;
        }

        public Builder page(int page) {
            this.page = page;
            return this;
        }

        public Builder size(int size) {
            if (size > 100) throw new IllegalArgumentException("페이지 크기는 100 이하");
            this.size = size;
            return this;
        }

        public QueryFilter build() {
            return new QueryFilter(this);
        }
    }
}
```

실무에서는 **Lombok의 `@Builder`**를 쓰는 경우가 많다.

하지만 유효성 검증이 복잡한 경우에는 직접 구현하는 것이 낫다.

### Singleton — 전역 인스턴스 (주의 필요)

**문제**: 시스템에서 하나만 존재해야 하는 객체 (커넥션 풀, 설정 매니저 등).

```java
// 고전적 Singleton (문제 많음)
public class ConnectionPool {
    private static ConnectionPool instance;

    private ConnectionPool() { }

    public static synchronized ConnectionPool getInstance() {
        if (instance == null) {
            instance = new ConnectionPool();
        }
        return instance;
    }
}
```

**Singleton의 문제점:**
- **테스트 어려움**: 전역 상태이므로 테스트 격리가 안 됨
- **숨은 의존성**: 코드 어디서든 `getInstance()`로 접근 → 의존 관계가 보이지 않음
- **동시성 문제**: `synchronized`로 인한 성능 저하, 또는 DCL(Double-Checked Locking) 복잡성

이 세 문제 모두 Singleton이 "전역 상태"라는 사실에서 비롯된다. 전역 상태는 코드 어디서든 접근 가능하므로 의존 관계를 추적하기 어렵고, 테스트 환경에서 상태를 초기화하거나 교체하기 어렵다.

**현대적 해법: DI 컨테이너에 위임**

```java
// Spring에서 빈은 기본적으로 싱글턴
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost/mydb");
        ds.setMaximumPoolSize(10);
        return ds;
    }
}

// 필요한 곳에 주입 — 전역 접근이 아닌 명시적 의존
@Service
public class UserService {
    private final DataSource dataSource;  // 테스트 시 목 주입 가능

    public UserService(DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

**규칙**: 직접 Singleton을 구현하지 마라. DI 컨테이너가 싱글턴 스코프를 관리하게 두라.

---

## 2. 구조 패턴

### Adapter — 인터페이스 변환

**문제**: 내 코드가 기대하는 인터페이스와 외부 라이브러리/서비스의 인터페이스가 다르다.

```java
// 우리 시스템의 결제 인터페이스
public interface PaymentGateway {
    PaymentResult charge(Money amount, CardInfo card);
    void refund(String transactionId, Money amount);
}

// 외부 라이브러리(Stripe)의 인터페이스 — 우리와 다르다
public class StripeClient {
    public StripeCharge createCharge(int amountInCents, String currency,
                                     String cardToken) { ... }
    public StripeRefund createRefund(String chargeId, int amountInCents) { ... }
}

// Adapter: 우리 인터페이스에 맞게 변환
public class StripePaymentAdapter implements PaymentGateway {
    private final StripeClient stripe;

    @Override
    public PaymentResult charge(Money amount, CardInfo card) {
        String token = tokenize(card);
        StripeCharge charge = stripe.createCharge(
            amount.toCents(),
            amount.getCurrency().getCode(),
            token
        );
        return new PaymentResult(charge.getId(), charge.getStatus());
    }

    @Override
    public void refund(String transactionId, Money amount) {
        stripe.createRefund(transactionId, amount.toCents());
    }
}
```

나중에 Toss Payments로 전환할 때? `TossPaymentAdapter`만 새로 만들면 된다. 비즈니스 로직은 변경 없음.

### Decorator — 기능의 동적 추가

**문제**: 기존 객체의 기능에 부가 기능을 유연하게 추가하고 싶다. 상속은 정적이고 조합이 폭발한다.

```java
// 기본 인터페이스
public interface HttpClient {
    Response execute(Request request);
}

// 기본 구현
public class DefaultHttpClient implements HttpClient {
    public Response execute(Request request) {
        return doHttp(request);
    }
}

// 로깅 데코레이터
public class LoggingHttpClient implements HttpClient {
    private final HttpClient delegate;

    public LoggingHttpClient(HttpClient delegate) {
        this.delegate = delegate;
    }

    public Response execute(Request request) {
        log.info("요청: {} {}", request.getMethod(), request.getUrl());
        long start = System.currentTimeMillis();
        Response response = delegate.execute(request);
        log.info("응답: {} ({}ms)", response.getStatus(), System.currentTimeMillis() - start);
        return response;
    }
}

// 재시도 데코레이터
public class RetryHttpClient implements HttpClient {
    private final HttpClient delegate;
    private final int maxRetries;

    public Response execute(Request request) {
        for (int i = 0; i <= maxRetries; i++) {
            try {
                return delegate.execute(request);
            } catch (Exception e) {
                if (i == maxRetries) throw e;
                sleep(backoff(i));
            }
        }
        throw new IllegalStateException("unreachable");
    }
}

// 조합: 로깅 + 재시도 + 기본 클라이언트
HttpClient client = new LoggingHttpClient(
    new RetryHttpClient(
        new DefaultHttpClient(), 3
    )
);
```

```text
호출 흐름:
client.execute(request)
    │
    ▼
LoggingHttpClient      → 요청 로그 기록
    │ delegate.execute()
    ▼
RetryHttpClient        → 실패 시 최대 3회 재시도
    │ delegate.execute()
    ▼
DefaultHttpClient      → 실제 HTTP 요청 수행
    │
    ▼ (응답)
RetryHttpClient        → 성공이면 통과, 실패면 재시도
    ▼
LoggingHttpClient      → 응답 시간/상태 로그 기록
    ▼
결과 반환
```

**프레임워크에서의 Decorator**:
- Java I/O: `new BufferedReader(new InputStreamReader(new FileInputStream("f.txt")))`
- Spring: `@Transactional`이 프록시(Decorator의 변형)로 트랜잭션을 감싸줌
- Servlet Filter / Spring Interceptor — 요청 처리 파이프라인

### Proxy — 접근 제어와 지연 로딩

Decorator와 구조는 동일하지만 **의도**가 다르다:
- Decorator: 기능 추가
- Proxy: 접근 제어, 지연 로딩, 캐싱

```java
// 캐싱 프록시
public class CachingUserRepository implements UserRepository {
    private final UserRepository actual;
    private final Cache cache;

    public User findById(Long id) {
        String key = "user:" + id;
        User cached = cache.get(key, User.class);
        if (cached != null) return cached;

        User user = actual.findById(id);
        cache.put(key, user, Duration.ofMinutes(10));
        return user;
    }
}
```

Spring의 `@Cacheable`, `@Transactional`, `@Async`는 모두 동적 프록시로 구현된다.

### Facade — 복잡한 하위 시스템의 단순한 인터페이스

```java
// 주문 처리에 관여하는 여러 서비스
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notification;

    public OrderResult placeOrder(OrderRequest request) {
        // 복잡한 조합을 하나의 메서드로
        inventory.reserve(request.getItems());
        PaymentResult payResult = payment.charge(request.getPaymentInfo());
        ShippingLabel label = shipping.createLabel(request.getAddress());
        notification.sendOrderConfirmation(request.getUserId());
        return new OrderResult(payResult, label);
    }
}

// 컨트롤러는 Facade만 알면 됨
@PostMapping("/orders")
public OrderResult createOrder(@RequestBody OrderRequest request) {
    return orderFacade.placeOrder(request);
}
```

---

## 3. 행위 패턴

### Strategy — 알고리즘 교체

**문제**: 같은 작업을 여러 방식으로 수행해야 하고, 런타임에 방식을 선택한다.

```java
// 할인 전략
public interface DiscountStrategy {
    Money calculate(Order order);
}

public class PercentageDiscount implements DiscountStrategy {
    private final double rate;

    public Money calculate(Order order) {
        return order.getTotal().multiply(rate);
    }
}

public class FixedAmountDiscount implements DiscountStrategy {
    private final Money amount;

    public Money calculate(Order order) {
        return order.getTotal().compareTo(amount) > 0 ? amount : order.getTotal();
    }
}

public class FirstOrderDiscount implements DiscountStrategy {
    public Money calculate(Order order) {
        return order.isFirstOrder() ? order.getTotal().multiply(0.1) : Money.ZERO;
    }
}

// 사용
public class PricingService {
    public Money calculateFinalPrice(Order order, DiscountStrategy discount) {
        Money discountAmount = discount.calculate(order);
        return order.getTotal().subtract(discountAmount);
    }
}
```

사실 Strategy와 OCP에서 다룬 PaymentHandler는 같은 패턴이다.

Strategy는 GoF 패턴 중 실무에서 가장 많이 쓰인다.

**함수형 스타일로 간소화:**

```java
// Java 8+에서는 인터페이스 없이 람다로 충분할 때가 많다
public class PricingService {
    public Money calculateFinalPrice(Order order, Function<Order, Money> discountFn) {
        return order.getTotal().subtract(discountFn.apply(order));
    }
}

// 사용
Money price = pricingService.calculateFinalPrice(order,
    o -> o.getTotal().multiply(0.15));  // 15% 할인
```

Strategy 인터페이스를 만들지 말고 `Function`/`Predicate` 같은 기존 함수형 인터페이스로 충분한지 먼저 판단하자.

### Template Method — 알고리즘의 골격

**문제**: 처리 흐름의 순서는 동일한데, 각 단계의 구현이 다르다.

```java
// 데이터 내보내기: 공통 흐름 + 형식별 차이
public abstract class DataExporter {
    // 템플릿 메서드 — 골격 정의 (final로 오버라이드 방지)
    public final void export(Query query, OutputStream out) {
        List<Record> data = fetchData(query);
        List<Record> filtered = applyFilters(data);
        writeHeader(out);
        for (Record record : filtered) {
            writeRecord(out, record);
        }
        writeFooter(out);
    }

    // 공통 구현
    private List<Record> fetchData(Query query) {
        return repository.findAll(query);
    }

    // 서브클래스가 구현하는 부분
    protected abstract void writeHeader(OutputStream out);
    protected abstract void writeRecord(OutputStream out, Record record);
    protected abstract void writeFooter(OutputStream out);
}

public class CsvExporter extends DataExporter {
    protected void writeHeader(OutputStream out) { write(out, "id,name,email\n"); }
    protected void writeRecord(OutputStream out, Record r) {
        write(out, r.getId() + "," + r.getName() + "," + r.getEmail() + "\n");
    }
    protected void writeFooter(OutputStream out) { /* CSV는 푸터 없음 */ }
}

public class ExcelExporter extends DataExporter {
    protected void writeHeader(OutputStream out) { /* 엑셀 헤더 행 */ }
    protected void writeRecord(OutputStream out, Record r) { /* 엑셀 행 */ }
    protected void writeFooter(OutputStream out) { /* 시트 마무리 */ }
}
```

**주의**: Template Method는 상속 기반이라 결합도가 높다.

가능하면 Strategy (조합) 방식을 먼저 고려하고, 흐름 제어가 복잡할 때만 Template Method를 사용한다.

### Observer — 이벤트 기반 통신

**문제**: 한 객체의 상태 변경을 여러 객체에 알려야 하는데, 직접 호출하면 결합도가 높아진다.

```java
// 직접 호출 — 강한 결합
public class OrderService {
    private final InventoryService inventory;
    private final NotificationService notification;
    private final PointService point;
    private final AnalyticsService analytics;

    public void completeOrder(Order order) {
        order.complete();
        orderRepository.save(order);

        inventory.decrease(order.getItems());        // 주문 완료할 때마다
        notification.sendReceipt(order);              // 새 기능 추가 =
        point.accumulate(order.getUserId(), order);   // OrderService 수정
        analytics.trackPurchase(order);               // 점점 비대해짐
    }
}
```

```java
// Observer/Event 기반 — 느슨한 결합
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public void completeOrder(Order order) {
        order.complete();
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCompletedEvent(order));
        // 이벤트 발행만 하면 끝. 누가 수신하는지 모름
    }
}

// 각 리스너는 독립적
@Component
public class InventoryListener {
    @EventListener
    public void onOrderCompleted(OrderCompletedEvent event) {
        inventoryService.decrease(event.getOrder().getItems());
    }
}

@Component
public class PointListener {
    @EventListener
    public void onOrderCompleted(OrderCompletedEvent event) {
        pointService.accumulate(event.getOrder().getUserId(), event.getOrder());
    }
}

// 새 기능 추가 = 새 리스너 클래스 추가 (OrderService 수정 없음)
@Component
public class AnalyticsListener {
    @EventListener
    public void onOrderCompleted(OrderCompletedEvent event) {
        analyticsService.trackPurchase(event.getOrder());
    }
}
```

**주의점:**
- 동기 이벤트: 같은 트랜잭션 안에서 실행. 리스너가 실패하면 주문도 롤백
- `@TransactionalEventListener(phase = AFTER_COMMIT)`: 트랜잭션 커밋 후 실행. 이메일 발송 같은 부작용에 적합
- 비동기 이벤트: `@Async` + `@EventListener`. 리스너 실패가 원본에 영향 없음. 하지만 순서 보장 안 됨

### Chain of Responsibility — 처리 파이프라인

```java
// Servlet Filter Chain — 가장 대표적인 구현
public class AuthFilter implements Filter {
    public void doFilter(Request req, Response res, FilterChain chain) {
        if (!isAuthenticated(req)) {
            res.sendError(401);
            return;  // 체인 중단
        }
        chain.doFilter(req, res);  // 다음 필터로
    }
}

public class RateLimitFilter implements Filter {
    public void doFilter(Request req, Response res, FilterChain chain) {
        if (isRateLimited(req)) {
            res.sendError(429);
            return;
        }
        chain.doFilter(req, res);
    }
}

public class LoggingFilter implements Filter {
    public void doFilter(Request req, Response res, FilterChain chain) {
        log.info("요청: {} {}", req.getMethod(), req.getPath());
        chain.doFilter(req, res);
        log.info("응답: {}", res.getStatus());
    }
}

// 실행 순서: Logging → Auth → RateLimit → Controller
```

Spring Security의 필터 체인, Netty의 Channel Pipeline, Express.js의 미들웨어가 모두 이 패턴이다.

---

## 4. 패턴 남용의 위험

### "패턴을 위한 패턴" 안티패턴

```java
// 파일 하나 읽는 기능에 이 모든 것이 필요한가?
public interface FileReader { ... }
public interface FileReaderFactory { ... }
public class DefaultFileReader implements FileReader { ... }
public class DefaultFileReaderFactory implements FileReaderFactory { ... }
public class FileReaderDecorator implements FileReader { ... }
public class CachingFileReaderDecorator extends FileReaderDecorator { ... }
public class FileReaderStrategyContext { ... }
public class FileReaderBuilder { ... }

// vs

// 이것으로 충분하다
String content = Files.readString(Path.of("config.json"));
```

### 패턴 선택 가이드

```
"이 조건문이 반복된다"           → Strategy
"생성자가 너무 복잡하다"         → Builder
"외부 API 인터페이스가 안 맞다"  → Adapter
"부가 기능을 유연하게 끼우고 싶다" → Decorator
"상태 변경을 여러 곳에 알려야 한다" → Observer (Event)
"처리 단계를 파이프라인으로 구성"  → Chain of Responsibility
"처리 흐름은 같고 세부만 다르다"  → Template Method (또는 Strategy)
"복잡한 서브시스템을 감추고 싶다"  → Facade
```

### 패턴을 적용하기 전 질문

1. **문제가 진짜 있는가?** 패턴은 문제를 해결하기 위한 것이다. 문제가 없는데 패턴을 적용하면 불필요한 복잡도만 추가된다.
2. **언어/프레임워크가 이미 해결해주지 않는가?** Java의 람다, Spring의 DI, Kotlin의 확장 함수 등이 패턴을 대체하는 경우가 많다.
3. **팀원이 이해할 수 있는가?** 아무리 우아한 패턴이라도 팀이 이해 못하면 유지보수 부채다.

---

## 마치며

디자인 패턴은 **어휘**다. 문제를 설명하고 해법을 소통하는 공통 언어다.

하지만 패턴 자체가 목적이 되면 안 된다.

- **생성 패턴**: Factory는 조건별 생성 분리에, Builder는 복잡한 객체 구성에 사용한다. Singleton은 DI 컨테이너에 맡겨라
- **구조 패턴**: Adapter는 외부 연동의 필수 도구이고, Decorator는 기능의 유연한 합성을 가능하게 한다
- **행위 패턴**: Strategy는 가장 자주 쓰이는 패턴이며, Observer(Event)는 시스템의 결합도를 극적으로 낮춘다
- **패턴 적용의 원칙**: 문제가 먼저, 패턴은 그다음. 단순한 코드가 항상 최선의 출발점이다

---

## 참고 자료

- *Design Patterns: Elements of Reusable Object-Oriented Software* — Gamma, Helm, Johnson, Vlissides (GoF)
- *Head First Design Patterns, 2nd Edition* — Eric Freeman, Elisabeth Robson (O'Reilly)
- Refactoring Guru — [https://refactoring.guru/design-patterns](https://refactoring.guru/design-patterns)
- *Effective Java, 3rd Edition* — Joshua Bloch, Item 1-2 (Static Factory, Builder)
- Spring Framework Reference — Event handling, AOP Proxies
