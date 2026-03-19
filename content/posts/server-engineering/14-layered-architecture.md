---
title: "[서버 애플리케이션 설계] 6편 — 계층 구조와 의존성: 코드를 어떻게 나누는가"
date: 2026-03-18T00:04:00+09:00
draft: false
tags: ["아키텍처", "DI", "클린아키텍처", "헥사고날", "서버"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "Controller-Service-Repository 각 책임, DI/IoC 원리, 의존 방향 규칙, 순환 의존 탐지, 클린 아키텍처, 헥사고날 아키텍처(Port & Adapter)까지."
---

## 들어가며

코드가 수천 줄을 넘어가면 "어디에 무엇을 넣어야 하는가"가 가장 중요한 문제가 된다. 계층 구조는 이 문제에 대한 답이다 — 코드를 역할별로 나누고, 계층 간 의존 방향을 통제하는 것.

이번 편에서는 가장 보편적인 3계층 구조부터 시작해서, 클린 아키텍처와 헥사고날 아키텍처까지 단계적으로 깊이를 더한다.

---

## 1. Controller-Service-Repository 패턴

### 세 계층의 책임

```
┌──────────────────────────────────────────┐
│  Controller (프레젠테이션 계층)             │
│  - HTTP 요청/응답 처리                    │
│  - 요청 파라미터 바인딩, 유효성 검증         │
│  - 응답 형식 변환 (DTO → JSON)            │
│  - 인증/인가 처리 (또는 필터에 위임)        │
├──────────────────────────────────────────┤
│  Service (비즈니스 계층)                   │
│  - 비즈니스 로직                          │
│  - 트랜잭션 관리                          │
│  - 여러 Repository/외부 서비스 조합        │
│  - 도메인 규칙 적용                       │
├──────────────────────────────────────────┤
│  Repository (데이터 접근 계층)              │
│  - 데이터베이스 CRUD                      │
│  - 쿼리 작성 (SQL, JPA, QueryDSL)        │
│  - 데이터 매핑 (DB 레코드 ↔ 엔티티)       │
└──────────────────────────────────────────┘

의존 방향: Controller → Service → Repository
```

### 각 계층의 코드 예시

```java
// Controller: HTTP 관심사만
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request,
            @AuthenticationPrincipal UserDetails user) {

        Order order = orderService.createOrder(user.getId(), request.toCommand());
        return ResponseEntity
                .created(URI.create("/api/orders/" + order.getId()))
                .body(OrderResponse.from(order));
    }
}

// Service: 비즈니스 로직
@Service
@Transactional
public class OrderService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final PaymentService paymentService;

    public Order createOrder(Long userId, CreateOrderCommand command) {
        // 재고 확인
        List<Product> products = productRepository.findAllById(command.getProductIds());
        products.forEach(Product::validateStock);

        // 주문 생성
        Order order = Order.create(userId, products, command.getQuantities());

        // 결제 처리
        paymentService.charge(order.getTotalAmount(), command.getPaymentInfo());

        // 재고 차감
        products.forEach(p -> p.decreaseStock(command.getQuantityOf(p.getId())));

        return orderRepository.save(order);
    }
}

// Repository: 데이터 접근만
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByUserIdOrderByCreatedAtDesc(Long userId);

    @Query("SELECT o FROM Order o WHERE o.status = :status AND o.createdAt < :before")
    List<Order> findStaleOrders(@Param("status") OrderStatus status,
                                @Param("before") LocalDateTime before);
}
```

### 흔한 실수

**1. 서비스에 HTTP 관심사가 침투**
```java
// ✗ 잘못된 예
@Service
public class UserService {
    public ResponseEntity<User> getUser(HttpServletRequest request) {
        String token = request.getHeader("Authorization");  // HTTP!
        // ...
    }
}

// ✓ 올바른 예
@Service
public class UserService {
    public User getUser(Long userId) {  // 순수 비즈니스 파라미터
        return userRepository.findById(userId)
                .orElseThrow(() -> new UserNotFoundException(userId));
    }
}
```

**2. 컨트롤러에 비즈니스 로직**
```java
// ✗ 잘못된 예
@PostMapping("/api/orders")
public ResponseEntity<?> createOrder(@RequestBody CreateOrderRequest req) {
    if (productRepository.findById(req.getProductId()).getStock() < req.getQuantity()) {
        return ResponseEntity.badRequest().body("재고 부족");  // 비즈니스 로직!
    }
    // ...
}
```

**3. 서비스가 다른 서비스의 Repository를 직접 사용**
```java
// ✗ 경계 침범
@Service
public class OrderService {
    private final UserRepository userRepository;  // User 도메인의 Repository를 직접 사용

// ✓ 서비스를 통해 접근
@Service
public class OrderService {
    private final UserService userService;  // User 도메인의 Service를 통해 접근
```

---

## 2. DTO와 데이터 흐름

### 계층 간 데이터 변환

```
HTTP 요청 → [Request DTO] → Controller → [Command/Query] → Service
                                                              │
                                                         [Entity]
                                                              │
Service → [Entity] → Controller → [Response DTO] → HTTP 응답
```

```java
// Request DTO (입력)
public record CreateOrderRequest(
    Long productId,
    int quantity,
    PaymentInfo paymentInfo
) {
    public CreateOrderCommand toCommand() {
        return new CreateOrderCommand(productId, quantity, paymentInfo);
    }
}

// Response DTO (출력)
public record OrderResponse(
    Long id,
    String status,
    int totalAmount,
    LocalDateTime createdAt
) {
    public static OrderResponse from(Order order) {
        return new OrderResponse(
            order.getId(),
            order.getStatus().name(),
            order.getTotalAmount(),
            order.getCreatedAt()
        );
    }
}
```

### 왜 Entity를 직접 반환하면 안 되는가

```java
// ✗ Entity를 직접 JSON으로 반환
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userService.getUser(id);  // User 엔티티 그대로 반환
}

문제점:
1. password, salt 같은 민감 정보가 노출될 수 있음
2. JPA lazy loading에 의한 N+1 쿼리
3. Entity 변경이 API 응답을 깨뜨림 (내부 변경 → 외부 영향)
4. 순환 참조 (User → Orders → User → ...)로 JSON 직렬화 실패
```

---

## 3. DI/IoC — 의존성 주입의 원리

### IoC (Inversion of Control)

```
제어의 역전: 객체의 생성과 생명주기를 내가 관리하지 않고 프레임워크에 위임

전통적 방식 (제어가 내 코드에):
UserService service = new UserService(new UserRepository(dataSource));

IoC (제어가 프레임워크에):
프레임워크가 UserRepository를 만들고 → UserService에 주입
```

### DI (Dependency Injection)

```java
// 1. 생성자 주입 (권장)
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    // Spring이 자동으로 적합한 빈을 주입
    public OrderService(OrderRepository orderRepository,
                        PaymentService paymentService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
    }
}

// 2. 필드 주입 (비권장)
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // 테스트 시 주입 어려움
}

// 3. Setter 주입 (선택적 의존성에만)
@Service
public class NotificationService {
    private EmailSender emailSender;

    @Autowired(required = false)
    public void setEmailSender(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
}
```

**생성자 주입이 권장되는 이유:**
```
1. 불변성: final 필드 사용 가능 → 객체 생성 후 변경 불가
2. 필수 의존성 보장: 생성자 파라미터 = 필수 → null 불가
3. 테스트 용이: new OrderService(mockRepo, mockPayment) 가능
4. 순환 의존성 감지: 생성자 주입은 순환 참조 시 즉시 실패
```

### 스프링 빈 생명주기

```
1. BeanDefinition 로드 (설정/어노테이션 스캔)
2. 빈 인스턴스 생성 (생성자 호출)
3. 의존성 주입 (DI)
4. @PostConstruct 호출
5. InitializingBean.afterPropertiesSet()
6. ─── 사용 ───
7. @PreDestroy 호출
8. DisposableBean.destroy()
9. 빈 소멸

스코프:
- singleton (기본): 컨테이너에 하나. 대부분의 서비스.
- prototype: 요청마다 새 인스턴스.
- request: HTTP 요청마다.
- session: HTTP 세션마다.
```

---

## 4. 의존 방향 규칙과 순환 의존

### 의존 방향 원칙

```
안쪽(고수준)은 바깥쪽(저수준)을 모른다:

Controller → Service → Repository
    │            │          │
  HTTP        비즈니스     데이터
  관심사       로직        접근

✓ Controller가 Service를 알음
✓ Service가 Repository를 알음
✗ Repository가 Service를 알면 안 됨
✗ Service가 Controller를 알면 안 됨
```

### 순환 의존의 문제

```java
// 순환 의존
@Service
public class OrderService {
    private final UserService userService;  // Order → User
}

@Service
public class UserService {
    private final OrderService orderService;  // User → Order
}

// Spring: 생성자 주입 시 BeanCurrentlyInCreationException
// 필드 주입은 에러가 나지 않지만, 설계적으로 문제임
```

### 순환 의존 해결

**방법 1: 공통 의존성 추출**
```
Before:  OrderService ↔ UserService
After:   OrderService → UserQueryService ← UserService
         (조회 전용 서비스를 분리)
```

**방법 2: 이벤트 기반 분리**
```java
// OrderService는 UserService를 직접 호출하지 않음
@Service
public class OrderService {
    private final ApplicationEventPublisher events;

    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.cancel();
        events.publishEvent(new OrderCancelledEvent(order));
    }
}

// UserService가 이벤트를 구독
@Component
public class UserOrderListener {
    private final UserService userService;

    @EventListener
    public void onOrderCancelled(OrderCancelledEvent event) {
        userService.refundPoints(event.getUserId(), event.getPointsUsed());
    }
}
```

**방법 3: 인터페이스로 의존 방향 역전**
```
Before:  OrderService → UserService → OrderService (순환)

After:
domain/
  OrderService → UserInfoProvider (인터페이스)
infrastructure/
  UserServiceAdapter implements UserInfoProvider
    → UserService
```

---

## 5. 클린 아키텍처

### 핵심 아이디어

로버트 마틴의 클린 아키텍처는 **의존 방향을 항상 안쪽으로** 향하게 한다:

```
┌─────────────────────────────────────────────────────┐
│  Frameworks & Drivers (가장 바깥)                     │
│  - Spring, JPA, MySQL, Redis, HTTP                  │
│  ┌─────────────────────────────────────────────┐    │
│  │  Interface Adapters                          │    │
│  │  - Controller, Repository 구현, DTO 변환     │    │
│  │  ┌─────────────────────────────────────┐    │    │
│  │  │  Application (Use Cases)             │    │    │
│  │  │  - 비즈니스 유스케이스                 │    │    │
│  │  │  ┌─────────────────────────────┐    │    │    │
│  │  │  │  Entities (Domain)           │    │    │    │
│  │  │  │  - 핵심 비즈니스 규칙          │    │    │    │
│  │  │  │  - 프레임워크 의존 없음        │    │    │    │
│  │  │  └─────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘

의존 방향: 항상 바깥 → 안쪽
안쪽 원은 바깥쪽 원의 존재를 모른다
```

### 클린 아키텍처의 실제 구현

```
src/
├── domain/                     # 엔티티, 비즈니스 규칙 (프레임워크 의존 없음)
│   ├── Order.java
│   ├── OrderStatus.java
│   └── Money.java
│
├── application/                # 유스케이스 (도메인 조합)
│   ├── port/
│   │   ├── in/
│   │   │   └── CreateOrderUseCase.java       # 인바운드 포트 (인터페이스)
│   │   └── out/
│   │       ├── OrderRepository.java          # 아웃바운드 포트 (인터페이스)
│   │       └── PaymentPort.java
│   └── service/
│       └── OrderApplicationService.java      # 유스케이스 구현
│
├── adapter/                    # 인터페이스 어댑터
│   ├── in/
│   │   └── web/
│   │       ├── OrderController.java          # HTTP 어댑터
│   │       └── OrderResponse.java
│   └── out/
│       ├── persistence/
│       │   └── JpaOrderRepository.java       # DB 어댑터
│       └── payment/
│           └── StripePaymentAdapter.java     # 결제 어댑터
│
└── config/                     # 프레임워크 설정 (Spring 등)
    └── BeanConfig.java
```

---

## 6. 헥사고날 아키텍처 (Port & Adapter)

### 클린 아키텍처와의 관계

헥사고날 아키텍처(Alistair Cockburn)는 클린 아키텍처의 구체적 구현 전략이다. 핵심 아이디어: **애플리케이션은 포트를 통해 외부와 소통하고, 어댑터가 포트를 구현한다.**

```
              ┌───────────────┐
   HTTP ──→  │ in:            │
   gRPC ──→  │ Port          │
              │  (인터페이스)   │     ┌──────────────────────┐
              │               │────→│                      │
              │               │     │  Application Core    │
              │               │     │  (Domain + UseCase)  │
              │               │←────│                      │
              │ out:          │     └──────────────────────┘
   DB    ←── │ Port          │
   API   ←── │  (인터페이스)   │
   Queue ←── │               │
              └───────────────┘

Inbound Port:  외부 → 애플리케이션 (드라이빙 포트)
Outbound Port: 애플리케이션 → 외부 (드리븐 포트)
Adapter:       포트의 실제 구현 (HTTP 어댑터, DB 어댑터 등)
```

### 포트와 어댑터 구현

```java
// === 인바운드 포트 (유스케이스 인터페이스) ===
public interface CreateOrderUseCase {
    Order execute(CreateOrderCommand command);
}

// === 아웃바운드 포트 (외부 시스템 인터페이스) ===
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(Long id);
}

public interface PaymentPort {
    PaymentResult charge(Money amount, PaymentInfo info);
}

// === 애플리케이션 코어 (포트 사용) ===
public class OrderApplicationService implements CreateOrderUseCase {
    private final OrderRepository orderRepository;  // 아웃바운드 포트
    private final PaymentPort paymentPort;           // 아웃바운드 포트

    @Override
    public Order execute(CreateOrderCommand command) {
        Order order = Order.create(command);
        paymentPort.charge(order.getTotalAmount(), command.getPaymentInfo());
        return orderRepository.save(order);
    }
}

// === 인바운드 어댑터 (HTTP) ===
@RestController
public class OrderController {
    private final CreateOrderUseCase createOrder;  // 인바운드 포트

    @PostMapping("/api/orders")
    public ResponseEntity<?> create(@RequestBody CreateOrderRequest req) {
        Order order = createOrder.execute(req.toCommand());
        return ResponseEntity.created(...).body(OrderResponse.from(order));
    }
}

// === 아웃바운드 어댑터 (JPA) ===
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final JpaOrderEntityRepository jpaRepo;

    @Override
    public Order save(Order order) {
        OrderEntity entity = OrderEntity.from(order);
        return jpaRepo.save(entity).toDomain();
    }
}

// === 아웃바운드 어댑터 (Stripe) ===
public class StripePaymentAdapter implements PaymentPort {
    private final StripeClient stripe;

    @Override
    public PaymentResult charge(Money amount, PaymentInfo info) {
        StripeCharge charge = stripe.createCharge(amount.toCents(), ...);
        return new PaymentResult(charge.getId(), charge.getStatus());
    }
}
```

### 테스트가 쉬워지는 이유

```java
// 인프라 없이 비즈니스 로직만 테스트
@Test
void 재고_부족시_주문_실패() {
    // 아웃바운드 포트를 Mock으로 대체
    OrderRepository mockRepo = mock(OrderRepository.class);
    PaymentPort mockPayment = mock(PaymentPort.class);

    CreateOrderUseCase useCase = new OrderApplicationService(mockRepo, mockPayment);

    assertThrows(InsufficientStockException.class, () ->
        useCase.execute(new CreateOrderCommand(productId, 999))
    );
}
```

---

## 7. 아키텍처 선택 가이드

```
3계층 (Controller-Service-Repository):
- 소~중규모 프로젝트
- 팀이 아키텍처에 익숙하지 않을 때
- 빠른 개발이 우선
- 대부분의 CRUD 서비스에 충분

클린/헥사고날 아키텍처:
- 복잡한 비즈니스 로직
- 외부 시스템이 자주 교체될 가능성
- 테스트 커버리지가 중요
- 도메인 전문가와 협업
- 장기 운영 서비스

주의:
- 아키텍처는 코드에 맞게 진화해야 함
- 처음부터 클린 아키텍처를 도입하면 CRUD에 과도한 보일러플레이트
- 3계층으로 시작하고, 복잡도가 높아지면 점진적으로 전환
```

---

## 마치며

계층 구조는 코드를 이해하고 변경하기 쉽게 만드는 핵심 도구다.

- **Controller-Service-Repository**는 가장 보편적인 구조이며, 각 계층의 책임을 명확히 지키는 것이 핵심이다
- **DTO**로 계층 간 데이터를 변환하면 내부 변경이 외부에 영향을 주지 않는다
- **DI**는 의존성을 외부에서 주입해서 결합도를 낮추고 테스트를 쉽게 만든다. 생성자 주입이 표준이다
- **순환 의존**은 설계 문제의 신호다. 이벤트, 인터페이스 분리, 공통 의존성 추출로 해결한다
- **클린/헥사고날 아키텍처**는 도메인을 중심에 놓고 인프라를 교체 가능하게 만든다. 복잡한 도메인에서 빛을 발한다

---

## 참고 자료

- *Clean Architecture* — Robert C. Martin (Prentice Hall)
- Alistair Cockburn, "Hexagonal Architecture" — [https://alistair.cockburn.us/hexagonal-architecture/](https://alistair.cockburn.us/hexagonal-architecture/)
- *Get Your Hands Dirty on Clean Architecture* — Tom Hombergs (Packt)
- Spring Framework Reference: IoC Container — [https://docs.spring.io/spring-framework/reference/core/beans.html](https://docs.spring.io/spring-framework/reference/core/beans.html)
- Martin Fowler, "Inversion of Control Containers and the Dependency Injection pattern" — [https://martinfowler.com/articles/injection.html](https://martinfowler.com/articles/injection.html)
