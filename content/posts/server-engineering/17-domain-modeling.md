---
title: "[서버 애플리케이션 설계] 9편 — 도메인 모델링: 비즈니스를 코드로 표현하는 법"
date: 2026-03-18T00:01:00+09:00
draft: false
tags: ["DDD", "도메인 모델링", "Value Object", "Entity", "Aggregate", "서버"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "도메인 모델 vs 데이터 모델, 풍부한 도메인 모델, Value Object·Entity·Aggregate 개념, DDD 전술 패턴 핵심, 유비쿼터스 언어, 도메인 이벤트, Application Service vs Domain Service까지"
---

코드를 오래 짜다 보면 어느 순간 이런 함수를 마주치게 된다. `updateUser(Long id, String name, String email, int status, BigDecimal balance, boolean verified)`. 파라미터가 6개인데, 그 중 어떤 것이 null이어도 되는지, `status`는 어떤 값이 유효한지, `balance`가 음수이면 어떻게 되는지 — 이런 것들이 코드 어디에도 명확하게 드러나지 않는다. 비즈니스 규칙은 서비스 레이어 여기저기에 흩어져 있고, 새로운 팀원은 도메인을 이해하려면 코드베이스 전체를 읽어야 한다. 이것이 **빈약한 도메인 모델(Anemic Domain Model)**이 만들어내는 현실이다. 도메인 모델링은 이 문제를 정면으로 해결하는 기술이다. 비즈니스의 언어를 코드로 직접 표현하고, 규칙을 데이터 근처에 두며, 잘못된 상태를 컴파일 타임 혹은 도메인 객체 생성 시점에 차단한다.

---

## 1. 도메인 모델 vs 데이터 모델

### 두 모델은 목적 자체가 다르다

데이터 모델(Data Model)은 데이터를 어떻게 **저장**하느냐에 대한 것이다. 정규화, 인덱스, 조인 성능, 스토리지 효율이 핵심 관심사다. 반면 도메인 모델(Domain Model)은 비즈니스를 어떻게 **표현**하느냐에 대한 것이다. 어떤 개념이 존재하는지, 개념들 간의 관계가 무엇인지, 어떤 행위가 허용되는지를 코드로 담는다.

흔히 하는 실수는 이 둘을 같다고 취급하는 것이다. JPA 엔티티를 그대로 도메인 모델로 사용하면 데이터베이스 스키마가 도메인 설계를 지배하게 된다. 테이블에 컬럼이 하나 추가되면 도메인 객체가 바뀌고, ORM의 제약(지연 로딩, cascade, 양방향 연관관계)이 비즈니스 로직 작성 방식을 제한한다.

```
데이터 모델 (DB 관점)          도메인 모델 (비즈니스 관점)
┌─────────────────────┐        ┌─────────────────────────┐
│ orders              │        │ Order                   │
│ ──────────────────  │        │ ─────────────────────── │
│ id          BIGINT  │        │ orderId: OrderId         │
│ user_id     BIGINT  │        │ customer: Customer       │
│ status      INT     │   vs   │ items: List<OrderItem>   │
│ total_price DECIMAL │        │ status: OrderStatus      │
│ created_at  DATETIME│        │                         │
│ updated_at  DATETIME│        │ + place()               │
│                     │        │ + cancel()              │
│                     │        │ + calculateTotal()      │
└─────────────────────┘        └─────────────────────────┘
```

데이터 모델에서 `status INT`는 그냥 숫자다. 도메인 모델에서 `OrderStatus`는 유효한 상태 전이를 강제하는 타입이다.

### 빈약한 도메인 모델의 징후

Martin Fowler가 명명한 **Anemic Domain Model**은 안티패턴이다. 객체지향의 형태를 빌리지만 실질은 절차적 코드다.

```java
// 빈약한 도메인 모델 — 데이터 홀더에 불과
public class Order {
    private Long id;
    private int status;
    private BigDecimal totalPrice;
    private List<OrderItem> items;

    // getter/setter 뿐
    public int getStatus() { return status; }
    public void setStatus(int status) { this.status = status; }
    public BigDecimal getTotalPrice() { return totalPrice; }
    // ...
}

// 모든 비즈니스 로직이 서비스에 집중
public class OrderService {
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        if (order.getStatus() == 1) {
            order.setStatus(3); // 1이 뭐고 3이 뭔지 알 수 없음
            // 환불 처리, 재고 복구, 알림 발송...
        }
    }
}
```

이 코드의 문제는 명백하다. `status == 1`이 '결제완료'인지 '배송중'인지 코드만 봐서는 알 수 없다. `setStatus(3)`이 어떤 비즈니스 의미인지 알려면 다른 곳을 찾아봐야 한다. 잘못된 상태 전이를 막는 로직이 없어서 어디서든 `setStatus()`를 호출할 수 있다.

### 풍부한 도메인 모델

**Rich Domain Model**은 비즈니스 규칙과 행위를 도메인 객체 안에 캡슐화한다.

```java
public class Order {
    private final OrderId id;
    private OrderStatus status;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private Money totalAmount;

    // 팩토리 메서드 — 유효한 초기 상태만 허용
    public static Order place(CustomerId customerId, List<OrderItem> items) {
        if (items == null || items.isEmpty()) {
            throw new DomainException("주문 항목이 비어있습니다");
        }
        Order order = new Order(OrderId.generate(), customerId, items);
        order.status = OrderStatus.PLACED;
        order.totalAmount = order.calculateTotal();
        return order;
    }

    public void cancel() {
        if (!status.isCancellable()) {
            throw new DomainException(
                "상태 " + status + "인 주문은 취소할 수 없습니다"
            );
        }
        this.status = OrderStatus.CANCELLED;
    }

    public void confirmPayment(PaymentId paymentId) {
        if (status != OrderStatus.PLACED) {
            throw new DomainException("결제 확인은 PLACED 상태에서만 가능합니다");
        }
        this.status = OrderStatus.PAYMENT_CONFIRMED;
    }

    private Money calculateTotal() {
        return items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

이제 잘못된 상태 전이는 `Order` 객체 수준에서 차단된다. 서비스가 아무리 `cancel()`을 호출해도 이미 배송된 주문은 취소되지 않는다.

---

## 2. Value Object, Entity, Aggregate

DDD(Domain-Driven Design)의 전술 패턴 중 가장 핵심적인 세 가지 빌딩 블록이다. 이것들을 정확히 이해하면 도메인 모델의 품질이 근본적으로 달라진다.

### Value Object — 값으로 정의되는 개념

**Value Object(값 객체)**는 식별자가 없고, 속성의 조합이 곧 그 객체의 정체성이 되는 개념을 표현한다. 두 Value Object가 모든 속성이 같다면 같은 것이다.

`Money`가 대표적인 예다. 5000원짜리 지폐 두 장은 '같은 돈'이다. 어떤 특정 지폐인지는 중요하지 않다. 반면 특정 고객은 식별자(회원번호)로 구분된다.

**Value Object의 핵심 특성:**

1. **불변성(Immutability)**: 생성 후 상태가 변하지 않는다. 변경이 필요하면 새 인스턴스를 만든다.
2. **동등성(Equality by value)**: `equals()`가 식별자가 아닌 속성 값으로 판단된다.
3. **자가 검증(Self-validation)**: 생성 시점에 유효성을 스스로 검증한다.

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount == null || currency == null) {
            throw new IllegalArgumentException("금액과 통화는 필수입니다");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new DomainException("금액은 음수일 수 없습니다");
        }
        // 소수점 2자리로 정규화
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new DomainException("다른 통화끼리는 더할 수 없습니다");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)), this.currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Money)) return false;
        Money other = (Money) o;
        return amount.equals(other.amount) && currency.equals(other.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}
```

이제 `Money`는 단순한 `BigDecimal`이 아니다. 통화가 다른 두 금액을 더하려고 하면 도메인 규칙이 이를 막는다. `BigDecimal`을 직접 쓸 때는 이런 실수를 컴파일러가 잡아주지 않는다.

**Value Object로 표현할 수 있는 것들:**
- `Email`, `PhoneNumber`, `Address`
- `DateRange`, `TimeSlot`
- `Percentage`, `Quantity`
- `OrderId`, `CustomerId` (식별자도 Value Object로 만들면 타입 안전성이 생긴다)

```java
// 타입으로 구분되는 식별자
public final class OrderId {
    private final UUID value;

    private OrderId(UUID value) { this.value = value; }

    public static OrderId generate() {
        return new OrderId(UUID.randomUUID());
    }

    public static OrderId of(String value) {
        return new OrderId(UUID.fromString(value));
    }
}

// 이제 컴파일러가 실수를 잡아준다
void process(OrderId orderId, CustomerId customerId) { ... }

// process(customerId, orderId) — 컴파일 에러! Long이었다면 조용히 버그가 됐을 것
```

### Entity — 식별자로 구분되는 개념

**Entity**는 식별자(Identity)를 가지며, 시간이 지남에 따라 상태가 변해도 동일한 개념으로 취급된다. 고객은 이름이 바뀌고 주소가 바뀌어도 여전히 같은 고객이다.

```java
public class Customer {
    private final CustomerId id;  // 불변 식별자
    private CustomerName name;    // 변경 가능한 속성
    private Email email;
    private Address shippingAddress;
    private CustomerGrade grade;

    // Entity equality는 id로만 판단
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Customer)) return false;
        Customer other = (Customer) o;
        return id.equals(other.id);
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }

    // 비즈니스 행위를 메서드로 표현
    public void changeShippingAddress(Address newAddress) {
        if (newAddress == null) {
            throw new DomainException("배송지 주소는 필수입니다");
        }
        this.shippingAddress = newAddress;
    }

    public void upgradeGrade(CustomerGrade newGrade) {
        if (newGrade.level() <= this.grade.level()) {
            throw new DomainException("등급은 상향만 가능합니다");
        }
        this.grade = newGrade;
    }
}
```

**Entity vs Value Object 판단 기준:**

| 질문 | Entity | Value Object |
|------|--------|-------------|
| 이 개념을 추적해야 하는가? | 예 | 아니오 |
| 두 인스턴스가 같은 속성이면 같은가? | 아니오 | 예 |
| 시간에 따라 상태가 변하는가? | 예 | 아니오 (교체) |
| 예시 | 고객, 주문, 상품 | 금액, 주소, 이름 |

주소(Address)는 Value Object인가 Entity인가? 상황에 따라 다르다. 단순히 배송지를 표현할 때는 Value Object로 충분하다. 하지만 고객이 여러 배송지를 관리하고 이름을 붙여 저장한다면(`집`, `회사`) Entity가 된다. **도메인 맥락이 모델의 성격을 결정한다.**

### Aggregate — 일관성의 경계

**Aggregate**는 도메인 모델에서 가장 중요하면서도 가장 잘못 이해되는 개념이다. Aggregate는 여러 Entity와 Value Object를 묶어 하나의 **일관성 경계(Consistency Boundary)**를 형성한다.

```
Order Aggregate
┌─────────────────────────────────────────┐
│  Order (Aggregate Root)                 │
│  ┌───────────────────────────────────┐  │
│  │  OrderId: OrderId                 │  │
│  │  status: OrderStatus              │  │
│  │  totalAmount: Money               │  │
│  └───────────────────────────────────┘  │
│                                         │
│  List<OrderItem> (Entity)               │
│  ┌───────────────────────────────────┐  │
│  │  OrderItemId: OrderItemId         │  │
│  │  productId: ProductId             │  │
│  │  quantity: Quantity               │  │
│  │  unitPrice: Money                 │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ShippingInfo (Value Object)            │
│  ┌───────────────────────────────────┐  │
│  │  address: Address                 │  │
│  │  receiverName: String             │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
         외부에서는 반드시 Order를 통해서만 접근
```

**Aggregate의 세 가지 규칙:**

**규칙 1: 오직 Aggregate Root를 통해서만 내부에 접근한다.**

외부에서 `OrderItem`을 직접 수정하는 코드는 허용하지 않는다. 항상 `Order`를 통해야 한다.

```java
// 잘못된 방식
OrderItem item = orderItemRepository.findById(itemId); // 직접 접근
item.setQuantity(3); // Aggregate 규칙 위반

// 올바른 방식
Order order = orderRepository.findById(orderId);
order.changeItemQuantity(itemId, 3); // Aggregate Root를 통해
orderRepository.save(order);
```

**규칙 2: Aggregate 내부의 일관성은 트랜잭션 하나로 보장한다.**

주문 항목 수량을 변경하면 총액이 자동으로 재계산된다. 이 두 가지가 별도 트랜잭션에 걸쳐서는 안 된다.

```java
public class Order {
    public void changeItemQuantity(OrderItemId itemId, int newQuantity) {
        OrderItem item = findItemOrThrow(itemId);
        item.changeQuantity(newQuantity);
        this.totalAmount = recalculateTotal(); // 일관성 유지
    }
}
```

**규칙 3: 다른 Aggregate는 ID로만 참조한다.**

```java
public class Order {
    private final CustomerId customerId; // Customer 객체를 직접 들고 있지 않음
    private final List<OrderItem> items; // OrderItem은 내부 Entity
}
```

`Order`가 `Customer` 객체를 직접 참조하면 두 Aggregate의 경계가 무너진다. `Customer` 조회는 `CustomerRepository`를 통해 별도로 하면 된다.

**Aggregate는 작게 설계하라**

흔한 실수는 연관된 모든 것을 하나의 Aggregate에 넣는 것이다. `User` Aggregate에 `Post`, `Comment`, `Follow`, `Notification`을 모두 넣으면 한 사용자를 로딩할 때 수백 개의 객체가 메모리에 올라온다. 트랜잭션 충돌도 빈번해진다. 두 사람이 같은 게시물에 댓글을 달면 같은 `User` Aggregate를 수정하게 되어 충돌이 발생한다.

**Aggregate 설계의 실용적 지침:**

1. 먼저 모든 것을 별개의 Aggregate로 출발한다.
2. 불변식(Invariant)을 지키기 위해 반드시 동일 트랜잭션에서 함께 변경되어야 하는 것들만 묶는다.
3. 의심스러우면 작게 유지한다.

---

## 3. DDD 전술 패턴 핵심과 유비쿼터스 언어

### 유비쿼터스 언어 — 코드가 곧 도메인 문서

**유비쿼터스 언어(Ubiquitous Language)**는 DDD에서 전략적 패턴에 속하지만, 전술 패턴의 품질을 결정하는 토대다. 개발자와 도메인 전문가(기획자, 비즈니스 팀)가 같은 언어를 공유하는 것이다. "주문을 확정한다"는 개념이 코드에서 `confirmOrder()`로 표현되고, 문서에서도, 회의에서도 같은 단어를 쓴다.

**왜 중요한가?** 번역 비용이 없어진다. "기획서에서 '주문 확정'이라고 했는데 코드에서는 어떤 함수를 건드려야 하죠?"라는 질문이 사라진다.

```java
// 유비쿼터스 언어가 반영된 코드
public class Order {
    // 나쁜 예: 기술적 언어
    public void updateStatusToConfirmed() { ... }

    // 좋은 예: 도메인 언어
    public void confirm() { ... }
    public void ship() { ... }
    public void deliver() { ... }
    public void cancel(CancellationReason reason) { ... }
}

public enum OrderStatus {
    PLACED,           // 주문 접수
    PAYMENT_CONFIRMED, // 결제 확인
    PREPARING,        // 준비 중
    SHIPPED,          // 배송 중
    DELIVERED,        // 배송 완료
    CANCELLED         // 취소됨
}
```

**유비쿼터스 언어를 코드에 녹이는 방법:**

1. 도메인 전문가와 화이트보드 앞에서 설계한다. 그들이 쓰는 단어를 그대로 클래스명, 메서드명으로 사용한다.
2. 기술 용어와 비즈니스 용어를 혼용하지 않는다. `User`를 어디서는 `Member`, 어디서는 `Customer`, 어디서는 `Account`로 부르면 개념이 흐려진다.
3. 코드 리뷰 때 "이 메서드 이름이 기획서의 어떤 행위와 대응하나요?"를 질문한다.

### Repository 패턴

Repository는 Aggregate의 영속성을 추상화한다. 도메인 레이어는 데이터베이스를 몰라야 한다.

```java
// 도메인 레이어에 정의
public interface OrderRepository {
    Order findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    List<Order> findPendingOrders(LocalDate since);
    void save(Order order);
    void delete(Order order);
}

// 인프라 레이어에 구현
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpaRepo;
    private final OrderMapper mapper;

    @Override
    public Order findById(OrderId id) {
        return jpaRepo.findById(id.getValue())
            .map(mapper::toDomain)
            .orElseThrow(() -> new OrderNotFoundException(id));
    }

    @Override
    public void save(Order order) {
        OrderEntity entity = mapper.toEntity(order);
        jpaRepo.save(entity);
    }
}
```

Repository는 컬렉션처럼 동작해야 한다. `findById`, `save`, `delete`가 있고, 복잡한 쿼리도 의미있는 이름의 메서드로 표현한다. `findByStatusAndCreatedAtBetweenAndCustomerIdOrderByCreatedAtDesc`는 유비쿼터스 언어가 아니다. `findRecentCancelledOrdersByCustomer`가 낫다.

### Factory 패턴

복잡한 Aggregate 생성 로직은 Factory로 분리한다.

```java
public class OrderFactory {
    private final ProductRepository productRepository;
    private final PricingPolicy pricingPolicy;

    public Order createOrder(CustomerId customerId, List<OrderRequest> requests) {
        List<OrderItem> items = requests.stream()
            .map(req -> {
                Product product = productRepository.findById(req.productId());
                if (!product.isAvailable()) {
                    throw new DomainException(product.getName() + "은 현재 구매 불가 상태입니다");
                }
                Money price = pricingPolicy.priceFor(product, customerId);
                return new OrderItem(product.getId(), req.quantity(), price);
            })
            .collect(toList());

        return Order.place(customerId, items);
    }
}
```

생성 로직이 여러 레포지토리와 서비스에 의존할 때 Factory는 이를 응집도 있게 담아낸다.

### Domain Service — 어디에도 속하지 않는 도메인 로직

특정 Entity나 Value Object에 자연스럽게 속하지 않는 도메인 로직은 **Domain Service**로 표현한다.

**전형적인 예: 두 Aggregate가 협력해야 하는 로직**

```java
// 계좌 이체 로직 — Account에도, Bank에도 속하지 않는다
public class TransferService {
    public void transfer(Account from, Account to, Money amount) {
        if (!from.hasSufficientBalance(amount)) {
            throw new DomainException("잔액이 부족합니다");
        }
        from.withdraw(amount);
        to.deposit(amount);
        // 이 두 동작은 원자적으로 실행되어야 한다
    }
}
```

**Domain Service와 Application Service를 혼동하지 말 것.** Domain Service는 도메인 언어로 표현되며 순수 비즈니스 로직만 담는다. 인프라(DB, 외부 API)에 의존하지 않는다.

---

## 4. 도메인 이벤트와 Application Service vs Domain Service

### 도메인 이벤트 — 비즈니스에서 중요한 일이 발생했다

**도메인 이벤트(Domain Event)**는 도메인에서 발생한 중요한 사실을 나타낸다. "주문이 취소되었다", "결제가 완료되었다", "재고가 소진되었다" 같은 것들이다. 이벤트는 과거형으로 이름을 짓는다.

```java
// 도메인 이벤트 베이스
public abstract class DomainEvent {
    private final UUID eventId;
    private final Instant occurredAt;

    protected DomainEvent() {
        this.eventId = UUID.randomUUID();
        this.occurredAt = Instant.now();
    }
}

// 구체적인 이벤트
public class OrderCancelledEvent extends DomainEvent {
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money refundAmount;
    private final CancellationReason reason;

    public OrderCancelledEvent(Order order) {
        super();
        this.orderId = order.getId();
        this.customerId = order.getCustomerId();
        this.refundAmount = order.getTotalAmount();
        this.reason = order.getCancellationReason();
    }
}
```

**Aggregate가 이벤트를 발행하는 방식:**

```java
public class Order {
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public void cancel(CancellationReason reason) {
        if (!status.isCancellable()) {
            throw new DomainException("취소할 수 없는 상태입니다: " + status);
        }
        this.status = OrderStatus.CANCELLED;
        this.cancellationReason = reason;

        // 이벤트 등록 (아직 발행하지 않음)
        domainEvents.add(new OrderCancelledEvent(this));
    }

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearDomainEvents() {
        domainEvents.clear();
    }
}
```

이벤트를 Aggregate 내부에 모아두고, 저장 시점에 이벤트를 발행한다. 이를 **Transactional Outbox Pattern**으로 확장하면 이벤트 발행의 원자성을 보장할 수 있다.

**Spring에서의 이벤트 처리:**

```java
@Service
@Transactional
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId());
        order.cancel(command.reason());
        orderRepository.save(order);

        // 트랜잭션 커밋 후 이벤트 발행
        order.getDomainEvents().forEach(eventPublisher::publishEvent);
        order.clearDomainEvents();
    }
}

@Component
public class OrderCancelledEventHandler {
    @EventListener
    @Async
    public void handle(OrderCancelledEvent event) {
        // 환불 처리, 재고 복구, 이메일 발송 — 별도 트랜잭션
        refundService.processRefund(event.getOrderId(), event.getRefundAmount());
        inventoryService.restoreStock(event.getOrderId());
        notificationService.sendCancellationNotice(event.getCustomerId());
    }
}
```

### 도메인 이벤트의 실질적 가치

도메인 이벤트 없이 위 로직을 구현하면 `cancelOrder()` 메서드 하나가 환불 처리, 재고 복구, 이메일 발송을 모두 직접 호출하게 된다. 의존성이 폭발한다. 나중에 "취소 시 포인트도 복구해야 한다"는 요구사항이 생기면 `cancelOrder()`를 수정해야 한다. 이벤트 기반으로 설계하면 새로운 핸들러를 추가하기만 하면 된다. **Open/Closed Principle**을 도메인 레이어에서 실현하는 방법이다.

### Application Service vs Domain Service — 명확한 구분

이 둘을 혼동하면 레이어 간 경계가 무너진다.

```
┌─────────────────────────────────────────────────────────┐
│                   Application Layer                      │
│  Application Service                                     │
│  - 유스케이스 오케스트레이션                              │
│  - 트랜잭션 경계 관리                                    │
│  - 인프라 의존성 조율 (Repository, EventPublisher...)    │
│  - 도메인 객체를 조합해 흐름 제어                        │
│  - 비즈니스 로직 없음 (결정하지 않음)                    │
└──────────────────────────┬──────────────────────────────┘
                           │ 호출
┌──────────────────────────▼──────────────────────────────┐
│                    Domain Layer                          │
│  Domain Service                                         │
│  - 여러 Aggregate에 걸친 도메인 로직                     │
│  - 인프라 의존성 없음                                    │
│  - 순수 비즈니스 규칙                                    │
│                                                         │
│  Entity / Value Object / Aggregate Root                 │
│  - 단일 Aggregate 내 비즈니스 로직                       │
└─────────────────────────────────────────────────────────┘
```

**Application Service 예시:**

```java
@Service
@Transactional
public class PlaceOrderApplicationService {
    private final CustomerRepository customerRepository;
    private final OrderRepository orderRepository;
    private final OrderFactory orderFactory;
    private final ApplicationEventPublisher eventPublisher;

    // 유스케이스: 주문 생성
    public OrderId placeOrder(PlaceOrderCommand command) {
        // 1. 필요한 객체 로딩
        Customer customer = customerRepository.findById(command.customerId());

        // 2. 도메인 로직 위임 (Application Service는 결정하지 않는다)
        Order order = orderFactory.createOrder(customer.getId(), command.items());

        // 3. 영속성 저장
        orderRepository.save(order);

        // 4. 이벤트 발행
        order.getDomainEvents().forEach(eventPublisher::publishEvent);

        return order.getId();
    }
}
```

Application Service는 "어떻게 실행하는가"를 담당하고, 도메인 모델은 "무엇이 올바른가"를 담당한다.

**Application Service에서 비즈니스 로직이 새어나오는 안티패턴:**

```java
// 잘못된 Application Service
@Service
public class OrderApplicationService {
    public void placeOrder(PlaceOrderCommand command) {
        // 비즈니스 규칙이 Application Service에 있다 — 잘못됨
        if (command.items().size() > 50) {
            throw new Exception("주문 항목은 50개를 초과할 수 없습니다");
        }
        if (command.items().stream().anyMatch(i -> i.quantity() <= 0)) {
            throw new Exception("수량은 양수여야 합니다");
        }
        // ... 이런 규칙들은 Order나 OrderItem 내부에 있어야 한다
    }
}
```

이런 코드는 단위 테스트에서도 서비스를 통해야 비즈니스 규칙을 테스트할 수 있게 되어, 테스트가 복잡해지고 느려진다.

### Command 객체 패턴

Application Service의 파라미터는 Command 객체로 표현하면 좋다. 메서드 시그니처가 간결해지고, 유효성 검사를 Command 레이어에서 처리할 수 있다.

```java
public record PlaceOrderCommand(
    CustomerId customerId,
    List<OrderItemRequest> items,
    Address shippingAddress
) {
    // Command 레벨의 기본 유효성 검사 (형식, null 체크)
    public PlaceOrderCommand {
        Objects.requireNonNull(customerId, "customerId는 필수입니다");
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("주문 항목은 필수입니다");
        }
    }
}
```

Command의 유효성 검사(null, 형식)와 도메인 규칙("재고가 충분한가")은 다르다. 전자는 Command에서, 후자는 도메인 모델에서 처리한다.

---

## 5. 실전에서 마주치는 문제들

### Aggregate 경계를 잘못 설계하면 생기는 일

**너무 큰 Aggregate:**

```java
// 잘못된 설계 — User에 모든 것이 들어있다
public class User {
    private List<Post> posts;         // 수백 개
    private List<Comment> comments;   // 수천 개
    private List<Order> orders;       // 수십 개
    private List<Notification> notifications; // 수천 개
}
```

이렇게 되면 `User`를 로딩할 때마다 수천 개의 객체가 메모리에 올라온다. 두 사람이 게시물에 댓글을 달면 둘 다 같은 `User` Aggregate를 수정하므로 트랜잭션 충돌이 발생한다.

**올바른 설계:**

```java
// User는 핵심 식별 정보만
public class User { /* userId, name, email, grade */ }

// Post는 별개 Aggregate
public class Post {
    private final PostId id;
    private final UserId authorId; // User ID만 참조
    private List<Comment> comments; // Comment는 Post Aggregate 내부
}
```

### 도메인 이벤트의 순서와 중복

이벤트 기반 아키텍처에서 이벤트가 중복 전달될 수 있다(at-least-once delivery). 이벤트 핸들러는 **멱등성(idempotency)**을 갖춰야 한다.

```java
@Component
public class RefundEventHandler {
    private final RefundRepository refundRepository;

    @EventListener
    public void handle(OrderCancelledEvent event) {
        // 이미 처리된 이벤트는 무시
        if (refundRepository.existsByOrderId(event.getOrderId())) {
            return;
        }
        // 환불 처리
        refundRepository.save(new Refund(event.getOrderId(), event.getRefundAmount()));
    }
}
```

### Value Object의 데이터베이스 매핑

JPA에서 Value Object는 `@Embeddable`로 매핑한다.

```java
@Embeddable
public class Money {
    @Column(name = "amount")
    private BigDecimal amount;

    @Column(name = "currency")
    @Enumerated(EnumType.STRING)
    private Currency currency;
}

@Entity
@Table(name = "orders")
public class OrderEntity {
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "total_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "total_currency"))
    })
    private Money totalAmount;
}
```

하지만 앞서 말했듯이 JPA Entity를 도메인 모델과 분리하는 게 더 깔끔하다. 매퍼를 통해 변환하면 JPA의 제약이 도메인 모델을 오염시키지 않는다. 성능이나 복잡도 면에서 작은 프로젝트는 같이 써도 되지만, 도메인이 복잡해질수록 분리의 이득이 커진다.

### 테스트 용이성

도메인 모델이 제대로 설계되면 인프라 없이 단위 테스트가 가능해진다.

```java
class OrderTest {
    @Test
    void 결제완료된_주문은_취소할_수_없다() {
        Order order = Order.place(
            CustomerId.of("cust-1"),
            List.of(new OrderItem(ProductId.of("prod-1"), 2, Money.of(1000, KRW)))
        );
        order.confirmPayment(PaymentId.of("pay-1"));
        order.ship(ShippingId.of("ship-1"));

        assertThatThrownBy(() -> order.cancel(CancellationReason.CUSTOMER_REQUEST))
            .isInstanceOf(DomainException.class)
            .hasMessageContaining("취소할 수 없는 상태");
    }

    @Test
    void 주문_취소_시_도메인_이벤트가_등록된다() {
        Order order = Order.place(
            CustomerId.of("cust-1"),
            List.of(new OrderItem(ProductId.of("prod-1"), 1, Money.of(5000, KRW)))
        );

        order.cancel(CancellationReason.CUSTOMER_REQUEST);

        assertThat(order.getDomainEvents())
            .hasSize(1)
            .first()
            .isInstanceOf(OrderCancelledEvent.class);
    }
}
```

DB 없이, 스프링 컨텍스트 없이, 수 밀리초 안에 비즈니스 규칙을 검증한다. 이것이 도메인 모델링의 실질적인 이점이다.

---

## 정리 — 무엇을 가져갈 것인가

도메인 모델링은 "코드를 예쁘게 짜는 기술"이 아니다. 비즈니스 복잡도를 관리하는 방법이다. 잘못된 상태를 타입 시스템과 도메인 규칙으로 차단하고, 비즈니스 변화가 있을 때 어디를 수정해야 하는지 명확하게 만든다.

핵심을 정리하면:

- **Value Object**: 불변, 값으로 동등성 판단, 생성 시 자가 검증
- **Entity**: 식별자로 동등성 판단, 비즈니스 행위를 메서드로 표현
- **Aggregate**: 일관성 경계, 작게 유지, Root를 통해서만 접근
- **Domain Event**: 비즈니스 사실의 과거형 표현, 결합도 낮추기
- **Application Service**: 유스케이스 오케스트레이션, 비즈니스 결정 없음
- **Domain Service**: 여러 Aggregate에 걸친 순수 도메인 로직
- **유비쿼터스 언어**: 코드 = 도메인 문서

처음부터 완벽한 도메인 모델을 설계할 필요는 없다. 빈약한 도메인 모델로 시작해서 비즈니스 규칙이 서비스 레이어에 쌓이기 시작할 때, 같은 조건 검사가 여러 곳에 중복될 때, 잘못된 상태 전이가 버그를 만들 때 — 이때가 도메인 모델을 강화할 시점이다.

---

## 참고 자료

- **Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003)** — DDD의 원전. 전술 패턴의 근거와 철학이 담겨 있다.
- **Vaughn Vernon, *Implementing Domain-Driven Design* (2013)** — Evans의 개념을 실전 코드로 구현하는 방법을 상세히 다룬다.
- **Vaughn Vernon, *Domain-Driven Design Distilled* (2016)** — DDD를 빠르게 실용적으로 적용하고 싶을 때 입문서로 좋다.
- **Martin Fowler, *Patterns of Enterprise Application Architecture* (2002)** — Domain Model, Repository, Service Layer 등 패턴의 원형 정의가 담겨 있다.
- **Scott Millett & Nick Tune, *Patterns, Principles, and Practices of Domain-Driven Design* (2015)** — 현대적 관점에서 DDD를 다루며 CQRS, Event Sourcing까지 포괄한다.
- **Martin Fowler, "Anemic Domain Model" (bliki)** — https://martinfowler.com/bliki/AnemicDomainModel.html — 빈약한 도메인 모델이 왜 안티패턴인지를 명확하게 설명하는 글.
