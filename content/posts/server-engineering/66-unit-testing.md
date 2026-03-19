---
title: "[테스트 전략] 1편 — 단위 테스트: 신뢰할 수 있는 코드"
date: 2026-03-17T16:04:00+09:00
draft: false
tags: ["단위 테스트", "JUnit", "Mockito", "TDD", "서버"]
series: ["테스트 전략"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 10
summary: "좋은 단위 테스트의 조건(FIRST 원칙), JUnit 5 완전 활용과 Parameterized Test, Mock vs Stub vs Spy vs Fake 정확한 구분, Mockito 실전과 과도한 Mocking의 위험까지"
---

코드를 배포할 때마다 손이 떨린다면, 그건 테스트를 신뢰하지 못한다는 신호다.

단위 테스트는 단순히 버그를 잡는 도구가 아니다. 잘 작성된 단위 테스트는 코드가 "무엇을 해야 하는지"를 문서화하고, 리팩터링을 안전하게 만들며, 설계의 품질을 실시간으로 피드백한다.

이 글에서는 단위 테스트를 단순히 "작성하는 법"이 아니라 **신뢰할 수 있는 테스트**를 만드는 원칙과 실전 기술을 다룬다.

---

## 좋은 단위 테스트의 조건: FIRST 원칙

FIRST는 단위 테스트가 갖춰야 할 다섯 가지 속성의 첫 글자를 모은 약어다.

이 원칙을 위반하는 테스트는 시간이 지날수록 팀의 발목을 잡는 기술 부채가 된다.

### Fast: 빨라야 한다

단위 테스트는 밀리초 단위로 실행돼야 한다.

테스트 하나가 1초가 걸린다면, 1000개의 테스트를 돌리는 데 17분이 소요된다. 그 누구도 커밋할 때마다 17분을 기다리지 않는다. 결국 테스트를 건너뛰게 된다.

**안티패턴: 느린 테스트**
```java
// 나쁜 예시 - 실제 Thread.sleep()을 사용하는 테스트
@Test
void shouldRetryAfterDelay() throws InterruptedException {
    service.sendRequest(); // 내부적으로 Thread.sleep(1000) 호출
    Thread.sleep(1500);
    verify(mockClient, times(2)).call();
}
```

`Thread.sleep()`이 보이면 설계를 다시 의심해야 한다. 시간 의존성은 추상화해야 한다.

```java
// 좋은 예시 - 시간을 추상화
@Test
void shouldRetryAfterDelay() {
    Clock fakeClock = Clock.fixed(Instant.now(), ZoneId.systemDefault());
    RetryService service = new RetryService(fakeClock);
    service.sendRequest();
    verify(mockClient, times(2)).call(); // 즉시 실행
}
```

### Independent: 독립적이어야 한다

테스트는 실행 순서에 무관하게 항상 동일한 결과를 내야 한다.

테스트 간 공유 상태, 전역 변수, 싱글턴 남용은 독립성을 파괴하는 주범이다.

**안티패턴: 순서 의존적 테스트**
```java
// 나쁜 예시 - 정적 상태 공유
class OrderServiceTest {
    static List<Order> orders = new ArrayList<>(); // 공유 상태!

    @Test
    void shouldAddOrder() {
        orders.add(new Order("item1"));
        assertEquals(1, orders.size());
    }

    @Test
    void shouldRemoveOrder() {
        // shouldAddOrder()가 먼저 실행됐다고 가정
        orders.remove(0);
        assertEquals(0, orders.size()); // 순서가 바뀌면 실패
    }
}
```

각 테스트는 자신이 필요한 상태를 직접 생성해야 한다.

```java
// 좋은 예시 - 독립적인 테스트
class OrderServiceTest {
    private OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService(); // 매 테스트마다 새로 생성
    }

    @Test
    void shouldAddOrder() {
        orderService.add(new Order("item1"));
        assertEquals(1, orderService.count());
    }
}
```

### Repeatable: 반복 가능해야 한다

어떤 환경에서 실행해도 동일한 결과가 나와야 한다.

로컬에서는 통과하고 CI에서는 실패하는 테스트는 테스트가 아니다. 불안감을 주는 노이즈일 뿐이다.

반복 불가능한 테스트의 원인은 대부분 세 가지다: 현재 시간, 랜덤 값, 외부 시스템 의존성.

```java
// 나쁜 예시 - 현재 시간에 의존
@Test
void shouldExpireAfter24Hours() {
    Token token = new Token();
    assertTrue(token.isExpired()); // 테스트 실행 시점에 따라 결과가 달라짐
}

// 좋은 예시 - 시간을 주입
@Test
void shouldExpireAfter24Hours() {
    Instant issuedAt = Instant.parse("2026-01-01T00:00:00Z");
    Instant now = issuedAt.plusHours(25);
    Token token = new Token(issuedAt);
    assertTrue(token.isExpiredAt(now));
}
```

### Self-validating: 자기 검증이 가능해야 한다

테스트 결과가 Pass/Fail로 명확하게 표현돼야 한다.

로그를 눈으로 확인해야 결과를 알 수 있는 테스트는 자동화된 테스트가 아니다. `System.out.println()`으로 결과를 확인하는 순간 그 테스트는 의미를 잃는다.

```java
// 나쁜 예시 - 눈으로 확인해야 하는 테스트
@Test
void shouldCalculateTotal() {
    Order order = new Order();
    order.addItem(new Item("book", 15000));
    System.out.println("Total: " + order.getTotal()); // 사람이 확인해야 함
}

// 좋은 예시 - 명시적 단언
@Test
void shouldCalculateTotal() {
    Order order = new Order();
    order.addItem(new Item("book", 15000));
    assertEquals(15000, order.getTotal());
}
```

### Timely: 적시에 작성돼야 한다

단위 테스트는 프로덕션 코드를 작성하기 직전이나 동시에 작성해야 한다.

수백 줄의 코드를 먼저 작성하고 나중에 테스트를 추가하려 하면, 테스트하기 어려운 구조가 이미 굳어져 있다. TDD(Test-Driven Development)가 강력한 이유는 여기에 있다.

---

## JUnit 5 완전 활용

JUnit 5는 JUnit 4와 아키텍처가 완전히 다르다. JUnit Platform, JUnit Jupiter, JUnit Vintage 세 모듈로 구성된다.

실무에서 JUnit 5를 제대로 활용하면 테스트 코드의 표현력이 크게 높아진다.

### 핵심 어노테이션

```java
import org.junit.jupiter.api.*;

class UserServiceTest {

    @BeforeAll
    static void initAll() {
        // 클래스 전체에서 한 번만 실행 (static)
        // 데이터베이스 컨테이너 시작 등 비용이 큰 초기화에 사용
    }

    @BeforeEach
    void init() {
        // 각 테스트 메서드 실행 전 호출
        // 테스트 픽스처 초기화
    }

    @AfterEach
    void tearDown() {
        // 각 테스트 메서드 실행 후 호출
        // 리소스 정리
    }

    @AfterAll
    static void tearDownAll() {
        // 클래스 전체에서 한 번만 실행 (static)
    }

    @Test
    @DisplayName("활성 사용자는 로그인할 수 있다")
    void activeUserCanLogin() {
        // 테스트 내용
    }

    @Test
    @Disabled("JIRA-1234: 결제 모듈 수정 후 활성화")
    void disabledTest() {
        // 임시로 비활성화
    }
}
```

### 예외 테스트

JUnit 4의 `@Test(expected = ...)` 방식은 예외 메시지를 검증할 수 없었다. JUnit 5의 `assertThrows`는 예외 객체 자체를 반환한다.

```java
@Test
@DisplayName("존재하지 않는 사용자 조회 시 예외 발생")
void shouldThrowWhenUserNotFound() {
    // given
    long nonExistentId = 999L;

    // when & then
    UserNotFoundException exception = assertThrows(
        UserNotFoundException.class,
        () -> userService.findById(nonExistentId)
    );

    assertEquals("User not found: 999", exception.getMessage());
    assertEquals(nonExistentId, exception.getUserId());
}
```

예외 발생 여부만 확인하고 메시지를 검증하지 않으면, 다른 이유로 같은 타입의 예외가 발생해도 테스트가 통과한다는 점을 기억하자.

### Assertions 고급 활용

```java
@Test
void shouldReturnCorrectUserDetails() {
    User user = userService.findById(1L);

    assertAll("사용자 정보 검증",
        () -> assertEquals("홍길동", user.getName()),
        () -> assertEquals("hong@example.com", user.getEmail()),
        () -> assertTrue(user.isActive()),
        () -> assertNotNull(user.getCreatedAt())
    );
}
```

`assertAll`은 모든 단언을 실행하고 실패한 것을 한 번에 보고한다. 일반 `assertEquals`를 여러 개 나열하면 첫 번째 실패에서 멈춰 디버깅 정보가 줄어든다.

### Parameterized Test

같은 로직을 다양한 입력값으로 검증할 때 코드 중복 없이 표현할 수 있는 JUnit 5의 핵심 기능이다.

```java
@ParameterizedTest
@ValueSource(strings = {"hong@example.com", "user.name+tag@domain.co.kr", "test@sub.domain.com"})
@DisplayName("유효한 이메일 형식 검증")
void shouldValidateCorrectEmails(String email) {
    assertTrue(EmailValidator.isValid(email));
}

@ParameterizedTest
@ValueSource(strings = {"notanemail", "@domain.com", "user@", ""})
@DisplayName("유효하지 않은 이메일 형식 거부")
void shouldRejectInvalidEmails(String email) {
    assertFalse(EmailValidator.isValid(email));
}
```

`@CsvSource`를 사용하면 입력과 기대값을 함께 지정할 수 있다.

```java
@ParameterizedTest
@CsvSource({
    "10000, 0.1, 9000",
    "50000, 0.2, 40000",
    "100000, 0.3, 70000"
})
@DisplayName("할인율 적용 후 최종 가격 계산")
void shouldApplyDiscount(int originalPrice, double discountRate, int expectedPrice) {
    assertEquals(expectedPrice, priceCalculator.calculate(originalPrice, discountRate));
}
```

대규모 데이터셋은 `@MethodSource`로 외부 메서드에서 공급받는다.

```java
@ParameterizedTest
@MethodSource("provideInvalidOrders")
@DisplayName("유효하지 않은 주문은 거부된다")
void shouldRejectInvalidOrders(Order order, String expectedError) {
    ValidationException ex = assertThrows(ValidationException.class,
        () -> orderService.validate(order));
    assertEquals(expectedError, ex.getMessage());
}

static Stream<Arguments> provideInvalidOrders() {
    return Stream.of(
        Arguments.of(new Order(null, 1), "상품이 없습니다"),
        Arguments.of(new Order("item", 0), "수량은 1 이상이어야 합니다"),
        Arguments.of(new Order("item", -1), "수량은 1 이상이어야 합니다")
    );
}
```

### @Nested로 테스트 구조화

연관된 테스트를 계층적으로 조직화하면 가독성이 크게 높아진다.

```java
@DisplayName("OrderService 테스트")
class OrderServiceTest {

    @Nested
    @DisplayName("주문 생성")
    class CreateOrder {

        @Test
        @DisplayName("정상적인 주문이 생성된다")
        void shouldCreateOrderSuccessfully() { ... }

        @Test
        @DisplayName("재고 부족 시 예외가 발생한다")
        void shouldThrowWhenOutOfStock() { ... }
    }

    @Nested
    @DisplayName("주문 취소")
    class CancelOrder {

        @Test
        @DisplayName("24시간 이내 주문은 취소 가능하다")
        void shouldCancelWithinTimeLimit() { ... }

        @Test
        @DisplayName("이미 배송된 주문은 취소 불가하다")
        void shouldNotCancelShippedOrder() { ... }
    }
}
```

---

## Mock vs Stub vs Spy vs Fake: 정확한 구분

테스트 더블(Test Double)의 종류를 혼용해서 사용하는 것은 의도를 불명확하게 만든다.

각각의 목적과 사용 시점을 명확히 이해해야 한다. Gerard Meszaros의 분류를 기반으로 설명한다.

### Stub: 정해진 값을 반환하는 대역

Stub은 호출에 대해 미리 정해진 응답을 반환한다. **상태 검증**에 사용한다. 호출 여부나 횟수를 검증하지 않는다.

```java
// Stub 예시 - 미리 정해진 값을 반환
@Test
void shouldCalculateTotalWithTax() {
    // TaxCalculator가 항상 10%를 반환하도록 스텁
    TaxCalculator stubCalculator = price -> price * 0.1;

    PriceService service = new PriceService(stubCalculator);
    assertEquals(11000, service.getPriceWithTax(10000));
}
```

Stub의 핵심: "이 협력자가 이 값을 반환하면, 내 코드는 올바르게 동작하는가?"를 검증한다.

### Mock: 호출을 검증하는 대역

Mock은 기대하는 호출이 실제로 발생했는지를 검증한다. **행동 검증**에 사용한다.

```java
// Mock 예시 - 메서드 호출 검증
@Test
void shouldSendEmailWhenOrderCreated() {
    EmailService mockEmailService = mock(EmailService.class);
    OrderService orderService = new OrderService(mockEmailService);

    orderService.createOrder(new Order("item1"));

    // "이 메서드가 이 인자로 정확히 한 번 호출됐는가?"를 검증
    verify(mockEmailService, times(1)).sendOrderConfirmation(any(Order.class));
}
```

Mock은 행동에 대한 기대를 설정하고, 테스트 후 그 기대가 충족됐는지 확인한다.

### Spy: 실제 객체를 감싸는 대역

Spy는 실제 객체를 감싸서 일부 동작만 오버라이드하거나 호출을 기록한다.

```java
// Spy 예시 - 실제 객체를 감싸서 호출 추적
@Test
void shouldCallSuperClassMethod() {
    List<String> realList = new ArrayList<>();
    List<String> spyList = spy(realList);

    spyList.add("item1");
    spyList.add("item2");

    // 실제 메서드가 호출되고, 그 호출도 기록됨
    verify(spyList, times(2)).add(anyString());
    assertEquals(2, spyList.size()); // 실제 데이터도 있음
}
```

Spy는 레거시 코드나 일부만 대체해야 할 때 유용하지만, 남용하면 테스트가 구현 세부사항에 과도하게 결합된다.

### Fake: 동작하는 단순 구현체

Fake는 실제로 동작하지만 프로덕션에서는 사용하기 부적합한 단순화된 구현체다.

인메모리 데이터베이스, 가짜 캐시, 단순화된 메시지 큐가 대표적인 예다.

```java
// Fake 예시 - 동작하는 인메모리 구현
public class FakeUserRepository implements UserRepository {
    private final Map<Long, User> store = new HashMap<>();
    private long nextId = 1;

    @Override
    public User save(User user) {
        user.setId(nextId++);
        store.put(user.getId(), user);
        return user;
    }

    @Override
    public Optional<User> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public void deleteAll() {
        store.clear();
    }
}

// 테스트에서 Fake 사용
@Test
void shouldFindUserAfterSave() {
    UserRepository fakeRepo = new FakeUserRepository();
    UserService service = new UserService(fakeRepo);

    User saved = service.createUser("홍길동", "hong@example.com");
    Optional<User> found = service.findById(saved.getId());

    assertTrue(found.isPresent());
    assertEquals("홍길동", found.get().getName());
}
```

Fake는 Mock보다 테스트를 더 현실적으로 만들고, 구현 세부사항에 덜 결합된다.

### 언제 무엇을 쓸 것인가

| 종류 | 목적 | 검증 방식 | 사용 시점 |
|------|------|-----------|-----------|
| Stub | 미리 정해진 값 반환 | 상태 검증 | 의존성이 특정 값을 반환해야 할 때 |
| Mock | 호출 여부 검증 | 행동 검증 | 부수 효과(이메일, 이벤트)를 검증할 때 |
| Spy | 실제 + 호출 추적 | 상태 + 행동 | 레거시 코드, 부분 대체 |
| Fake | 단순 동작 구현 | 상태 검증 | 복잡한 의존성을 단순화할 때 |

---

## Mockito 실전

Mockito는 Java 생태계에서 가장 널리 사용되는 Mock 프레임워크다.

올바르게 사용하면 강력하지만, 잘못 사용하면 테스트를 오히려 취약하게 만든다.

### 기본 사용법

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    private PaymentGateway paymentGateway;

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private PaymentService paymentService;

    @Test
    @DisplayName("결제 성공 시 주문 상태가 PAID로 변경된다")
    void shouldUpdateOrderStatusOnSuccessfulPayment() {
        // given
        Order order = new Order(1L, 50000);
        PaymentResult successResult = PaymentResult.success("TXN-001");

        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
        when(paymentGateway.charge(50000)).thenReturn(successResult);

        // when
        paymentService.pay(1L);

        // then
        assertEquals(OrderStatus.PAID, order.getStatus());
        verify(orderRepository).save(order);
    }
}
```

### ArgumentCaptor: 인자를 캡처해서 검증

단순히 메서드가 호출됐는지 외에, **어떤 인자로** 호출됐는지 검증해야 할 때 사용한다.

```java
@Test
@DisplayName("주문 생성 시 올바른 감사 로그가 기록된다")
void shouldRecordAuditLogOnOrderCreation() {
    // given
    ArgumentCaptor<AuditLog> logCaptor = ArgumentCaptor.forClass(AuditLog.class);
    Order order = new Order(1L, "item1");

    // when
    orderService.createOrder(order);

    // then
    verify(auditLogger).log(logCaptor.capture());
    AuditLog capturedLog = logCaptor.getValue();
    assertEquals("ORDER_CREATED", capturedLog.getEventType());
    assertEquals(1L, capturedLog.getResourceId());
    assertNotNull(capturedLog.getTimestamp());
}
```

### 예외 stubbing

```java
@Test
@DisplayName("결제 게이트웨이 오류 시 PaymentException이 전파된다")
void shouldPropagatePaymentGatewayException() {
    // given
    when(paymentGateway.charge(anyInt()))
        .thenThrow(new GatewayTimeoutException("결제 게이트웨이 타임아웃"));

    // when & then
    assertThrows(PaymentException.class, () -> paymentService.pay(1L));
}
```

### doReturn vs thenReturn

`thenReturn`은 Stub 설정 시 실제 메서드를 한 번 호출한다. Spy와 함께 쓸 때 예상치 못한 부수 효과가 발생할 수 있다.

```java
// Spy와 함께 사용할 때는 doReturn을 사용
@Test
void shouldUseDoReturnWithSpy() {
    List<String> spyList = spy(new ArrayList<>());

    // 나쁜 예 - IndexOutOfBoundsException 발생 가능
    // when(spyList.get(0)).thenReturn("first");

    // 좋은 예 - 실제 메서드를 호출하지 않음
    doReturn("first").when(spyList).get(0);

    assertEquals("first", spyList.get(0));
}
```

---

## 과도한 Mocking의 위험

Mockito를 사용할 수 있다고 해서 모든 의존성을 Mock으로 대체하는 건 잘못된 방향이다.

과도한 Mocking은 테스트를 구현 세부사항에 강하게 결합시켜, 리팩터링 시 테스트가 먼저 깨지는 역설적인 상황을 만든다.

### 안티패턴 1: 모든 것을 Mock으로 대체

```java
// 나쁜 예시 - 과도한 Mocking
@Test
void shouldCalculateOrderTotalWithExcessiveMocking() {
    // 단순한 계산 로직인데 모든 것을 Mock으로 대체
    Item mockItem = mock(Item.class);
    when(mockItem.getPrice()).thenReturn(10000);
    when(mockItem.getQuantity()).thenReturn(3);

    PriceCalculator mockCalculator = mock(PriceCalculator.class);
    when(mockCalculator.multiply(10000, 3)).thenReturn(30000);

    OrderSummary summary = new OrderSummary(mockItem, mockCalculator);
    assertEquals(30000, summary.getTotal());

    // 이 테스트는 실제로 아무것도 검증하지 않는다
    // mockCalculator.multiply()의 반환값을 그냥 확인하는 것
}

// 좋은 예시 - 실제 객체 사용
@Test
void shouldCalculateOrderTotal() {
    Item item = new Item("book", 10000, 3);
    OrderSummary summary = new OrderSummary(item);
    assertEquals(30000, summary.getTotal());
}
```

### 안티패턴 2: 구현을 테스트하는 Mock

```java
// 나쁜 예시 - 내부 구현을 검증
@Test
void shouldCallRepositoryMethods() {
    userService.updateEmail(1L, "new@example.com");

    // 내부 구현 세부사항을 검증
    verify(userRepository).findById(1L);
    verify(userRepository).save(any(User.class));
    // 리팩터링 시 이 테스트는 깨진다
}

// 좋은 예시 - 결과(행동)를 검증
@Test
void shouldUpdateUserEmail() {
    User user = new User(1L, "old@example.com");
    when(userRepository.findById(1L)).thenReturn(Optional.of(user));

    userService.updateEmail(1L, "new@example.com");

    // 저장된 사용자의 이메일이 변경됐는지 확인
    ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
    verify(userRepository).save(captor.capture());
    assertEquals("new@example.com", captor.getValue().getEmail());
}
```

### 언제 Mock을 사용하지 말아야 하는가

**Mock을 쓰지 말아야 할 때:**
- 단순한 값 객체(Value Object) — 실제 객체를 사용하라
- 변환/계산 로직 — 실제 로직을 테스트하라
- 이미 Fake나 인메모리 구현이 있을 때 — Fake를 사용하라

**Mock을 써야 할 때:**
- 외부 HTTP API 호출 — 실제 호출은 느리고 불안정하다
- 이메일, SMS 발송 — 테스트에서 실제로 발송하면 안 된다
- 비용이 발생하는 서드파티 서비스 — 과금이 발생할 수 있다
- 제어할 수 없는 외부 시스템 — 재현 불가능한 상태를 만든다

### 테스트 설계 원칙: 런던 학파 vs 디트로이트 학파

Mock 사용에 대한 두 가지 철학이 있다.

**런던 학파(Mockist):** 협력자를 모두 Mock으로 대체하고, 단일 클래스의 행동을 고립해서 검증한다. 세밀한 격리가 가능하지만 구현에 결합된다.

**디트로이트 학파(Classicist):** 가능하면 실제 객체를 사용하고, 외부 시스템만 Mock으로 대체한다. 리팩터링에 강하지만 실패 원인 파악이 어려울 수 있다.

현실적인 조언은 **디트로이트 학파를 기본으로 하되**, 외부 시스템 호출에만 Mock을 사용하는 것이다. 그리고 Mock보다는 Fake를 우선적으로 고려하라.

---

## 실전 패턴: Given-When-Then

테스트의 가독성을 높이는 가장 효과적인 구조화 방식이다.

```java
@Test
@DisplayName("VIP 고객은 추가 5% 할인을 받는다")
void shouldApplyVipDiscountOnTopOfRegularDiscount() {
    // given - 테스트에 필요한 상태와 협력자를 설정
    Customer vipCustomer = Customer.builder()
        .id(1L)
        .grade(CustomerGrade.VIP)
        .build();

    Product product = Product.builder()
        .id(10L)
        .price(100000)
        .discountRate(0.1) // 기본 10% 할인
        .build();

    // when - 테스트하려는 행동을 실행
    int finalPrice = pricingService.calculateFinalPrice(vipCustomer, product);

    // then - 기대하는 결과를 단언
    // 기본 할인 10% + VIP 추가 5% = 85000
    assertEquals(85000, finalPrice);
}
```

Given-When-Then의 핵심은 각 섹션이 **하나의 책임**만 갖는 것이다. When 섹션은 한 줄이어야 한다. 여러 줄이 필요하다면 테스트 대상이 너무 크다는 신호다.

### 테스트 이름 지어주기

좋은 테스트 이름은 문서화다.

```
// 나쁜 이름
void test1()
void testLogin()
void shouldWork()

// 좋은 이름 - "상황에서 행동을 하면 결과가 나온다"
void shouldReturnUserWhenValidCredentialsProvided()
void shouldThrowExceptionWhenUserIsInactive()
void shouldSendEmailWhenOrderStatusChangedToPaid()
```

`@DisplayName`을 활용하면 한글로 더 명확하게 표현할 수 있다.

```java
@Test
@DisplayName("비활성 사용자가 로그인 시도 시 UserInactiveException이 발생한다")
void shouldThrowWhenInactiveUserAttemptLogin() { ... }
```

---

## 테스트 커버리지의 오해

커버리지 100%가 목표가 되는 순간, 테스트는 수단이 아니라 목적이 된다.

커버리지는 테스트가 얼마나 많은 코드를 **실행**했는지를 보여주지, 얼마나 잘 **검증**했는지는 보여주지 않는다.

```java
// 커버리지는 100%지만 아무것도 검증하지 않는 테스트
@Test
void coverageTest() {
    userService.createUser("test", "test@example.com"); // 예외 없이 실행됨
    // 아무런 단언 없음
}
```

실무에서 의미 있는 지표는:
- **라인 커버리지 80% 이상** — 실용적인 최소 기준
- **브랜치 커버리지** — if/else, switch 분기를 모두 테스트했는가
- **Mutation Testing** — 테스트가 실제로 결함을 잡는가 (PIT 사용)

---

## 마무리: 단위 테스트는 설계 피드백이다

단위 테스트가 작성하기 어렵다면, 그건 테스트의 문제가 아니라 **설계의 문제**다.

의존성이 너무 많은 클래스, 비공개 메서드가 너무 많은 설계, 생성자에서 비즈니스 로직을 실행하는 코드는 모두 테스트가 어렵다. 테스트가 어렵다는 신호를 무시하고 Reflection이나 PowerMock으로 억지로 테스트하는 건 근본 문제를 방치하는 것이다.

좋은 단위 테스트는 빠르게 실행되고, 실패 원인을 명확하게 알려주며, 리팩터링 후에도 살아남는다.

다음 편에서는 단위 테스트의 한계를 넘어, 실제 컴포넌트 간 상호작용을 검증하는 통합 테스트를 다룬다.

---

## 참고 자료

1. **Vladimir Khorikov, "Unit Testing: Principles, Practices, and Patterns"** (Manning, 2020) — 런던 학파와 디트로이트 학파의 차이, Mock 남용의 위험성에 대한 가장 체계적인 분석
2. **Martin Fowler, "Mocks Aren't Stubs"** — https://martinfowler.com/articles/mocksArentStubs.html — 테스트 더블의 종류와 차이에 대한 원본 논문
3. **JUnit 5 공식 문서** — https://junit.org/junit5/docs/current/user-guide/ — Parameterized Test, Extension Model 등 공식 레퍼런스
4. **Mockito 공식 문서** — https://site.mockito.org/ — ArgumentCaptor, doReturn/thenReturn 차이 등 상세 가이드
5. **Gerard Meszaros, "xUnit Test Patterns"** (Addison-Wesley, 2007) — 테스트 더블 분류의 원전, Stub/Mock/Spy/Fake/Dummy 개념 정의
6. **Kent Beck, "Test-Driven Development: By Example"** (Addison-Wesley, 2002) — TDD의 원형, Red-Green-Refactor 사이클의 철학적 배경
