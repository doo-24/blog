---
title: "[비동기 처리와 이벤트 드리븐] 6편 — 이벤트 소싱과 CQRS: 상태 대신 이벤트를 저장하는 설계"
date: 2026-03-17T21:02:00+09:00
draft: false
tags: ["이벤트 소싱", "CQRS", "이벤트 스토어", "도메인 이벤트", "서버"]
series: ["비동기 처리와 이벤트 드리븐"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 5
summary: "이벤트 소싱 원리와 이벤트 스토어 설계, 이벤트 재생(Replay)과 스냅샷 전략, CQRS 개념과 읽기/쓰기 모델 분리, CQRS + 이벤트 소싱 조합과 도입 트레이드오프까지"
---

은행 계좌를 생각해보자. 잔고가 50만 원이라는 사실보다 중요한 것은 *어떻게* 50만 원이 됐는지다.

월급 200만 원이 입금됐고, 카드값 70만 원이 빠졌고, 현금 80만 원을 인출했다. 현재 잔고는 이 이벤트들의 누적 결과다.

전통적인 RDBMS 설계는 최종 상태만 저장한다 — 잔고 컬럼에 50만 원이라고. 이벤트 소싱(Event Sourcing)은 반대로 간다. 상태가 아니라 상태를 만들어낸 모든 이벤트를 저장하고, 필요할 때 재생(Replay)해서 현재 상태를 도출한다.

감사 로그가 공짜로 따라오고, 타임 트래블이 가능해지며, 버그가 발생한 시점으로 돌아가 원인을 추적할 수 있다.

그리고 이 이벤트 중심의 설계는 자연스럽게 CQRS(Command Query Responsibility Segregation)와 결합되며 읽기와 쓰기를 완전히 다른 모델로 분리하는 아키텍처를 완성한다.

---

## 이벤트 소싱의 원리

### 상태 저장 vs 이벤트 저장

기존 방식은 도메인 객체의 현재 상태를 테이블에 덮어쓴다. 주문 상태가 `PENDING`에서 `CONFIRMED`로 바뀌면 `orders` 테이블의 `status` 컬럼을 업데이트한다. 변경 이력은 사라진다. 언제, 왜 바뀌었는지 알려면 별도의 감사 테이블을 만들거나 애플리케이션 로그를 뒤져야 한다.

이벤트 소싱은 이 흐름을 뒤집는다. 주문이 생성되면 `OrderPlaced` 이벤트를, 결제가 완료되면 `OrderConfirmed` 이벤트를, 취소되면 `OrderCancelled` 이벤트를 이벤트 스토어에 추가(append)한다. 현재 주문 상태가 필요하면 해당 주문의 모든 이벤트를 순서대로 재생해서 상태를 계산한다.

```
[OrderPlaced] → [PaymentProcessed] → [OrderConfirmed] → [OrderShipped]
     ↓                  ↓                    ↓                 ↓
  상태: PENDING    상태: PAID          상태: CONFIRMED    상태: SHIPPED
                                         (현재 상태)
```

핵심 불변식은 하나다. **이벤트 스토어는 추가만 가능하다(append-only).** 과거 이벤트는 절대 수정하거나 삭제하지 않는다. 이것이 이벤트 소싱의 신뢰성과 감사 가능성의 근거다.

### 도메인 이벤트 설계

도메인 이벤트는 과거에 발생한 사실을 나타낸다. 이름은 항상 과거형이어야 한다 — `OrderPlaced`(주문됨), `PaymentFailed`(결제 실패), `InventoryReserved`(재고 예약됨). 미래 의도를 나타내는 커맨드(`PlaceOrder`, `ProcessPayment`)와 명확히 구분된다.

```java
// 기본 이벤트 인터페이스
public interface DomainEvent {
    String getAggregateId();
    String getEventType();
    Instant getOccurredAt();
    long getVersion();
}

// 추상 기반 클래스
@Getter
public abstract class AbstractDomainEvent implements DomainEvent {
    private final String aggregateId;
    private final String eventType;
    private final Instant occurredAt;
    private long version;

    protected AbstractDomainEvent(String aggregateId, String eventType) {
        this.aggregateId = aggregateId;
        this.eventType = eventType;
        this.occurredAt = Instant.now();
    }

    public void setVersion(long version) {
        this.version = version;
    }
}

// 구체 이벤트
@Getter
public class OrderPlacedEvent extends AbstractDomainEvent {
    private final String customerId;
    private final List<OrderItem> items;
    private final Money totalAmount;

    public OrderPlacedEvent(String orderId, String customerId,
                            List<OrderItem> items, Money totalAmount) {
        super(orderId, "OrderPlaced");
        this.customerId = customerId;
        this.items = Collections.unmodifiableList(items);
        this.totalAmount = totalAmount;
    }
}

@Getter
public class OrderConfirmedEvent extends AbstractDomainEvent {
    private final String paymentId;
    private final Instant confirmedAt;

    public OrderConfirmedEvent(String orderId, String paymentId) {
        super(orderId, "OrderConfirmed");
        this.paymentId = paymentId;
        this.confirmedAt = Instant.now();
    }
}
```

---

## 이벤트 스토어 설계

### 스키마 설계

이벤트 스토어의 핵심은 단순하다. 이벤트를 순서대로, 중복 없이, 영속적으로 저장하면 된다. 일반 RDBMS 테이블로도 충분히 구현할 수 있으며, 핵심은 `(aggregate_id, event_version)` 유일 제약으로 낙관적 동시성을 보장하는 것이다.

```sql
CREATE TABLE event_store (
    id          BIGSERIAL PRIMARY KEY,
    aggregate_id    VARCHAR(255) NOT NULL,
    aggregate_type  VARCHAR(100) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    event_version   BIGINT NOT NULL,
    event_data      JSONB NOT NULL,
    metadata        JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (aggregate_id, event_version)  -- 낙관적 잠금의 핵심
);

CREATE INDEX idx_event_store_aggregate
    ON event_store (aggregate_id, event_version);

CREATE INDEX idx_event_store_type_time
    ON event_store (aggregate_type, occurred_at);
```

`UNIQUE (aggregate_id, event_version)` 제약이 핵심이다. 같은 애그리게이트에 동시에 같은 버전 이벤트를 저장하려 하면 DB 레벨에서 충돌이 발생한다. 이것이 별도의 분산 락 없이 낙관적 동시성 제어를 구현하는 방법이다.

### EventStore 구현

```java
@Repository
@RequiredArgsConstructor
public class PostgresEventStore implements EventStore {

    private final JdbcTemplate jdbcTemplate;
    private final ObjectMapper objectMapper;
    private final EventRegistry eventRegistry;

    @Override
    @Transactional
    public void appendEvents(String aggregateId, String aggregateType,
                             long expectedVersion, List<DomainEvent> events) {
        // 현재 최대 버전 확인
        Long currentVersion = jdbcTemplate.queryForObject(
            "SELECT MAX(event_version) FROM event_store WHERE aggregate_id = ?",
            Long.class, aggregateId
        );
        long actualVersion = currentVersion != null ? currentVersion : -1L;

        if (actualVersion != expectedVersion) {
            throw new OptimisticConcurrencyException(
                String.format("Aggregate %s: expected version %d but was %d",
                    aggregateId, expectedVersion, actualVersion)
            );
        }

        long nextVersion = actualVersion + 1;
        for (DomainEvent event : events) {
            ((AbstractDomainEvent) event).setVersion(nextVersion);
            String eventData = serialize(event);

            jdbcTemplate.update(
                """
                INSERT INTO event_store
                    (aggregate_id, aggregate_type, event_type, event_version, event_data, occurred_at)
                VALUES (?, ?, ?, ?, ?::jsonb, ?)
                """,
                aggregateId, aggregateType, event.getEventType(),
                nextVersion, eventData, event.getOccurredAt()
            );
            nextVersion++;
        }
    }

    @Override
    public List<DomainEvent> loadEvents(String aggregateId) {
        return loadEvents(aggregateId, 0L, Long.MAX_VALUE);
    }

    @Override
    public List<DomainEvent> loadEvents(String aggregateId, long fromVersion, long toVersion) {
        return jdbcTemplate.query(
            """
            SELECT event_type, event_data, event_version
            FROM event_store
            WHERE aggregate_id = ?
              AND event_version BETWEEN ? AND ?
            ORDER BY event_version ASC
            """,
            (rs, rowNum) -> {
                String eventType = rs.getString("event_type");
                String eventData = rs.getString("event_data");
                Class<? extends DomainEvent> eventClass = eventRegistry.getClass(eventType);
                DomainEvent event = deserialize(eventData, eventClass);
                ((AbstractDomainEvent) event).setVersion(rs.getLong("event_version"));
                return event;
            },
            aggregateId, fromVersion, toVersion
        );
    }

    private String serialize(DomainEvent event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (JsonProcessingException e) {
            throw new EventSerializationException("Failed to serialize event", e);
        }
    }

    private <T extends DomainEvent> T deserialize(String json, Class<T> type) {
        try {
            return objectMapper.readValue(json, type);
        } catch (JsonProcessingException e) {
            throw new EventSerializationException("Failed to deserialize event", e);
        }
    }
}
```

### 애그리게이트 재구성

이벤트 소싱에서 애그리게이트는 이벤트를 적용(apply)하는 방식으로 상태를 변경한다.

```java
public class Order {
    private String orderId;
    private String customerId;
    private OrderStatus status;
    private List<OrderItem> items;
    private Money totalAmount;
    private long version = -1L;

    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();

    // 이벤트 스토어에서 재구성할 때 사용
    public static Order reconstitute(List<DomainEvent> events) {
        Order order = new Order();
        for (DomainEvent event : events) {
            order.applyEvent(event);
            order.version = event.getVersion();
        }
        return order;
    }

    // 새 주문 생성 (커맨드 핸들러에서 호출)
    public static Order place(String orderId, String customerId,
                               List<OrderItem> items, Money totalAmount) {
        Order order = new Order();
        OrderPlacedEvent event = new OrderPlacedEvent(orderId, customerId, items, totalAmount);
        order.applyEvent(event);
        order.uncommittedEvents.add(event);
        return order;
    }

    public void confirm(String paymentId) {
        if (this.status != OrderStatus.PENDING) {
            throw new InvalidOrderStateException("Only PENDING orders can be confirmed");
        }
        OrderConfirmedEvent event = new OrderConfirmedEvent(this.orderId, paymentId);
        applyEvent(event);
        uncommittedEvents.add(event);
    }

    // 이벤트 적용 — 상태 변경만 담당, 검증 없음
    private void applyEvent(DomainEvent event) {
        if (event instanceof OrderPlacedEvent e) {
            this.orderId = e.getAggregateId();
            this.customerId = e.getCustomerId();
            this.items = new ArrayList<>(e.getItems());
            this.totalAmount = e.getTotalAmount();
            this.status = OrderStatus.PENDING;
        } else if (event instanceof OrderConfirmedEvent e) {
            this.status = OrderStatus.CONFIRMED;
        } else if (event instanceof OrderCancelledEvent e) {
            this.status = OrderStatus.CANCELLED;
        }
    }

    public List<DomainEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }

    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }

    public long getVersion() {
        return version;
    }
}
```

---

## 이벤트 재생과 스냅샷 전략

### 이벤트 재생(Replay)

이벤트 재생은 이벤트 소싱의 핵심 기능이다. 특정 시점의 상태를 알고 싶다면 그 시점까지의 이벤트만 재생하면 된다. 버그를 유발한 시점의 데이터를 확인하거나, 새로운 읽기 모델을 처음부터 구축하거나, 프로젝션을 재계산할 때 사용한다.

```java
@Service
@RequiredArgsConstructor
public class OrderRepository {

    private final EventStore eventStore;

    public Order findById(String orderId) {
        List<DomainEvent> events = eventStore.loadEvents(orderId);
        if (events.isEmpty()) {
            throw new OrderNotFoundException(orderId);
        }
        return Order.reconstitute(events);
    }

    @Transactional
    public void save(Order order) {
        List<DomainEvent> uncommittedEvents = order.getUncommittedEvents();
        if (uncommittedEvents.isEmpty()) return;

        eventStore.appendEvents(
            order.getOrderId(),
            "Order",
            order.getVersion(),
            uncommittedEvents
        );
        order.markEventsAsCommitted();

        // 이벤트 발행 (비동기 프로젝션, 사가 등에서 소비)
        uncommittedEvents.forEach(applicationEventPublisher::publishEvent);
    }

    // 특정 시점 상태 조회 — 타임 트래블
    public Order findByIdAt(String orderId, Instant pointInTime) {
        List<DomainEvent> events = eventStore.loadEventsUpTo(orderId, pointInTime);
        return Order.reconstitute(events);
    }
}
```

### 스냅샷 전략

이벤트가 수천 개 쌓이면 매 요청마다 전체를 재생하는 것은 비효율적이다. 스냅샷은 특정 버전에서의 애그리게이트 상태를 직렬화해서 저장하고, 이후에는 스냅샷 + 그 이후 이벤트만 로딩하도록 최적화한다.

```sql
CREATE TABLE aggregate_snapshots (
    aggregate_id    VARCHAR(255) NOT NULL,
    aggregate_type  VARCHAR(100) NOT NULL,
    snapshot_version BIGINT NOT NULL,
    snapshot_data   JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (aggregate_id, snapshot_version)
);
```

```java
@Service
@RequiredArgsConstructor
public class SnapshotEventStore implements EventStore {

    private final PostgresEventStore delegateStore;
    private final SnapshotStore snapshotStore;
    private final int snapshotThreshold = 50; // 50 이벤트마다 스냅샷

    @Override
    public Order loadOrder(String orderId) {
        // 1. 최신 스냅샷 조회
        Optional<Snapshot<Order>> snapshot = snapshotStore.findLatest(orderId);

        Order order;
        long fromVersion;

        if (snapshot.isPresent()) {
            // 2. 스냅샷에서 상태 복원
            order = snapshot.get().getState();
            fromVersion = snapshot.get().getVersion() + 1;
        } else {
            // 스냅샷 없음 — 처음부터 재생
            order = new Order();
            fromVersion = 0L;
        }

        // 3. 스냅샷 이후 이벤트만 로딩
        List<DomainEvent> remainingEvents =
            delegateStore.loadEvents(orderId, fromVersion, Long.MAX_VALUE);

        for (DomainEvent event : remainingEvents) {
            order.applyEvent(event);
        }

        return order;
    }

    @Override
    @Transactional
    public void save(Order order) {
        delegateStore.save(order);

        // 스냅샷 생성 조건 확인
        long currentVersion = order.getVersion();
        long lastSnapshotVersion = snapshotStore.findLatestVersion(order.getOrderId())
            .orElse(-1L);

        if (currentVersion - lastSnapshotVersion >= snapshotThreshold) {
            Snapshot<Order> snapshot = new Snapshot<>(
                order.getOrderId(), "Order", currentVersion, order
            );
            snapshotStore.save(snapshot);
        }
    }
}
```

### 안티패턴: 이벤트 수정과 삭제

이벤트 소싱에서 가장 흔한 실수는 과거 이벤트를 수정하거나 삭제하려는 유혹이다.

```java
// WRONG: 이벤트를 수정하려는 시도
// 절대 하지 말 것 — 이벤트 소싱의 근본을 파괴한다
jdbcTemplate.update(
    "UPDATE event_store SET event_data = ? WHERE aggregate_id = ? AND event_version = ?",
    newData, aggregateId, version
);
```

개인정보 삭제(GDPR Right to be Forgotten) 같은 요구사항은 이벤트 암호화로 대응한다. 각 사용자의 이벤트를 사용자별 키로 암호화하고, 삭제 요청이 들어오면 키만 삭제한다. 이벤트는 남아있지만 복호화할 수 없게 된다. 이를 Crypto-Shredding이라 한다.

---

## CQRS 개념과 읽기/쓰기 모델 분리

### CQRS란 무엇인가

CQRS는 Greg Young이 CQS(Command Query Separation) 원칙을 아키텍처 수준으로 확장한 패턴이다. 핵심은 단순하다. **데이터를 변경하는 커맨드(Command)와 데이터를 조회하는 쿼리(Query)를 완전히 다른 모델로 처리한다.**

전통적인 CRUD 아키텍처에서는 같은 도메인 모델로 읽기와 쓰기를 모두 처리한다. 이 접근은 몇 가지 문제를 낳는다.

- **임피던스 불일치**: 쓰기에 최적화된 정규화 스키마는 복잡한 조회에 불리하다. 반대로 조회에 최적화된 비정규화 구조는 쓰기 성능을 해친다.
- **복잡성 누적**: 하나의 모델이 모든 유스케이스를 담당하다 보면 점점 비대해진다.
- **독립적 확장 불가**: 읽기와 쓰기의 부하 특성이 다른데(대부분의 시스템은 읽기가 훨씬 많다) 같은 서버를 확장해야 한다.

### CQRS 구조

```
[Client]
    │
    ├─── Command ──→ [Command Handler] ──→ [Write Model/Aggregate] ──→ [Event Store]
    │                                                                        │
    │                                                                   [Event Bus]
    │                                                                        │
    └─── Query  ──→ [Query Handler]  ──→ [Read Model/Projection] ←──── [Projector]
```

커맨드 사이드는 도메인 로직의 정확성에 집중하고, 쿼리 사이드는 조회 성능에 집중한다.

### 커맨드 처리

```java
// 커맨드 정의
public record PlaceOrderCommand(
    String customerId,
    List<OrderItemDto> items
) {}

// 커맨드 핸들러
@Service
@RequiredArgsConstructor
@Transactional
public class OrderCommandHandler {

    private final OrderRepository orderRepository;
    private final InventoryService inventoryService;
    private final PriceCalculator priceCalculator;

    public String handle(PlaceOrderCommand command) {
        // 1. 재고 확인 (도메인 서비스)
        command.items().forEach(item ->
            inventoryService.reserve(item.productId(), item.quantity())
        );

        // 2. 가격 계산
        List<OrderItem> orderItems = command.items().stream()
            .map(dto -> new OrderItem(dto.productId(), dto.quantity(),
                priceCalculator.getPrice(dto.productId())))
            .toList();
        Money totalAmount = orderItems.stream()
            .map(item -> item.price().multiply(item.quantity()))
            .reduce(Money.ZERO, Money::add);

        // 3. 애그리게이트 생성 및 저장
        String orderId = UUID.randomUUID().toString();
        Order order = Order.place(orderId, command.customerId(), orderItems, totalAmount);
        orderRepository.save(order);

        return orderId;
    }

    public void handle(ConfirmOrderCommand command) {
        Order order = orderRepository.findById(command.orderId());
        order.confirm(command.paymentId());
        orderRepository.save(order);
    }
}
```

### 읽기 모델과 프로젝션

읽기 모델은 쿼리에 최적화된 비정규화 데이터 구조다. 여러 애그리게이트에 걸친 데이터를 미리 조합해서 저장한다.

```sql
-- 주문 목록 조회에 최적화된 읽기 모델
CREATE TABLE order_summaries (
    order_id        VARCHAR(255) PRIMARY KEY,
    customer_id     VARCHAR(255) NOT NULL,
    customer_name   VARCHAR(255) NOT NULL,  -- 비정규화
    status          VARCHAR(50) NOT NULL,
    item_count      INT NOT NULL,
    total_amount    DECIMAL(19, 4) NOT NULL,
    placed_at       TIMESTAMPTZ NOT NULL,
    confirmed_at    TIMESTAMPTZ,
    shipped_at      TIMESTAMPTZ
);
```

```java
// 프로젝터 — 이벤트를 받아 읽기 모델을 업데이트
@Component
@RequiredArgsConstructor
public class OrderSummaryProjector {

    private final OrderSummaryRepository repository;
    private final CustomerQueryService customerQueryService;

    @EventListener
    @Async
    public void on(OrderPlacedEvent event) {
        String customerName = customerQueryService.getCustomerName(event.getCustomerId());

        OrderSummary summary = OrderSummary.builder()
            .orderId(event.getAggregateId())
            .customerId(event.getCustomerId())
            .customerName(customerName)
            .status("PENDING")
            .itemCount(event.getItems().size())
            .totalAmount(event.getTotalAmount().getAmount())
            .placedAt(event.getOccurredAt())
            .build();

        repository.save(summary);
    }

    @EventListener
    @Async
    public void on(OrderConfirmedEvent event) {
        repository.findById(event.getAggregateId()).ifPresent(summary -> {
            summary.setStatus("CONFIRMED");
            summary.setConfirmedAt(event.getOccurredAt());
            repository.save(summary);
        });
    }
}

// 쿼리 핸들러 — 읽기 모델에서 직접 조회
@Service
@RequiredArgsConstructor
public class OrderQueryHandler {

    private final OrderSummaryRepository orderSummaryRepository;
    private final OrderDetailRepository orderDetailRepository;

    public Page<OrderSummaryDto> getOrdersByCustomer(String customerId, Pageable pageable) {
        return orderSummaryRepository
            .findByCustomerId(customerId, pageable)
            .map(OrderSummaryDto::from);
    }

    public OrderDetailDto getOrderDetail(String orderId) {
        return orderDetailRepository.findById(orderId)
            .map(OrderDetailDto::from)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

---

## CQRS + 이벤트 소싱 조합

### 왜 두 패턴이 잘 맞는가

이벤트 소싱과 CQRS는 서로를 보완한다. 이벤트 소싱은 쓰기 모델에서 발생한 모든 변경을 이벤트로 발행한다. CQRS의 프로젝터는 이 이벤트들을 구독해서 다양한 읽기 모델을 구축한다. 하나의 쓰기 사이드에서 여러 개의 읽기 모델을 독립적으로 운용할 수 있다.

```
[Order Aggregate] ─ 이벤트 발행 ─→ [Event Bus / Event Store]
                                           │
                            ┌──────────────┼──────────────┐
                            ↓              ↓              ↓
                    [OrderSummary    [Analytics      [Search
                     Projector]      Projector]      Projector]
                            │              │              │
                    [MySQL 읽기 DB]  [ClickHouse]  [Elasticsearch]
```

각 읽기 모델은 독립적으로 확장, 재구축할 수 있다. 새로운 기능을 위해 새로운 읽기 모델이 필요하다면 이벤트 스토어의 처음부터 재생해서 프로젝션을 만들면 된다.

### 전체 흐름 구현

```java
// API 레이어 — 커맨드와 쿼리 엔드포인트 분리
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderCommandHandler commandHandler;
    private final OrderQueryHandler queryHandler;

    // 커맨드 엔드포인트 — 201 Created + 위치 반환
    @PostMapping
    public ResponseEntity<Void> placeOrder(@RequestBody @Valid PlaceOrderRequest request) {
        String orderId = commandHandler.handle(request.toCommand());
        URI location = URI.create("/orders/" + orderId);
        return ResponseEntity.created(location).build();
    }

    // 쿼리 엔드포인트 — 읽기 모델에서 직접 조회
    @GetMapping("/{orderId}")
    public ResponseEntity<OrderDetailDto> getOrder(@PathVariable String orderId) {
        return ResponseEntity.ok(queryHandler.getOrderDetail(orderId));
    }

    @GetMapping
    public ResponseEntity<Page<OrderSummaryDto>> getOrders(
            @RequestParam String customerId,
            Pageable pageable) {
        return ResponseEntity.ok(queryHandler.getOrdersByCustomer(customerId, pageable));
    }
}
```

### 결과적 일관성 처리

CQRS에서 커맨드가 성공했더라도 읽기 모델이 즉시 업데이트되지 않을 수 있다. 프로젝터가 비동기로 동작하기 때문이다. 이 결과적 일관성(Eventual Consistency)을 클라이언트가 인지하도록 설계해야 한다.

```java
// 커맨드 응답에 버전 정보 포함
@PostMapping
public ResponseEntity<CommandResponse> placeOrder(@RequestBody @Valid PlaceOrderRequest request) {
    PlaceOrderResult result = commandHandler.handle(request.toCommand());

    CommandResponse response = new CommandResponse(
        result.orderId(),
        result.version(),          // 현재 이벤트 버전
        "/orders/" + result.orderId()
    );

    return ResponseEntity.accepted().body(response);
}

// 클라이언트가 특정 버전이 프로젝션에 반영될 때까지 폴링
@GetMapping("/{orderId}")
public ResponseEntity<OrderDetailDto> getOrder(
        @PathVariable String orderId,
        @RequestParam(required = false) Long minVersion) {

    if (minVersion != null) {
        // 읽기 모델이 해당 버전을 따라잡을 때까지 대기 (짧은 폴링)
        orderReadModelWaiter.waitForVersion(orderId, minVersion, Duration.ofSeconds(5));
    }

    return ResponseEntity.ok(queryHandler.getOrderDetail(orderId));
}
```

### 프로젝션 재구축

이벤트 소싱의 강력한 기능 중 하나는 언제든 프로젝션을 처음부터 다시 만들 수 있다는 점이다.

```java
@Service
@RequiredArgsConstructor
public class ProjectionRebuilder {

    private final EventStore eventStore;
    private final OrderSummaryProjector projector;
    private final OrderSummaryRepository summaryRepository;

    @Transactional
    public void rebuildOrderSummaries() {
        log.info("Starting OrderSummary projection rebuild");

        // 기존 읽기 모델 초기화
        summaryRepository.deleteAll();

        // 모든 Order 이벤트를 처음부터 재생
        eventStore.loadAllEventsByType("Order")
            .forEach(event -> {
                if (event instanceof OrderPlacedEvent e) {
                    projector.on(e);
                } else if (event instanceof OrderConfirmedEvent e) {
                    projector.on(e);
                }
                // ... 모든 이벤트 타입 처리
            });

        log.info("OrderSummary projection rebuild completed");
    }
}
```

---

## 도입 트레이드오프

### 이점

**감사 및 추적 가능성.** 모든 변경의 이력이 자동으로 기록된다. 누가 언제 무엇을 했는지 언제나 확인 가능하다.

**타임 트래블 디버깅.** 특정 시점의 상태로 돌아가 버그를 재현할 수 있다. 프로덕션 버그 추적이 극적으로 쉬워진다.

**읽기 모델 유연성.** 새로운 비즈니스 요구가 생기면 이벤트를 재생해서 새로운 읽기 모델을 만들면 된다. 기존 데이터를 마이그레이션할 필요가 없다.

**이벤트 드리븐 통합.** 마이크로서비스 간 통합이 자연스러워진다. 이벤트 스토어가 통신의 원천이 된다.

### 비용과 한계

**쿼리 복잡성.** 현재 상태를 직접 조회할 수 없다. 항상 프로젝션이나 재생을 거쳐야 한다. 단순한 CRUD 시스템에는 과도한 복잡성이다.

**결과적 일관성.** 커맨드 성공 후 읽기 모델이 즉시 반영되지 않는다. 애플리케이션과 UI가 이를 처리해야 한다.

**이벤트 스키마 진화.** 이벤트 구조를 변경하기 어렵다. 과거 이벤트와의 하위 호환성을 항상 유지해야 한다. 이를 위해 이벤트 업캐스팅(Upcasting) 전략이 필요하다.

```java
// 이벤트 업캐스터 — 구버전 이벤트를 신버전으로 변환
@Component
public class OrderPlacedEventUpcaster implements EventUpcaster {

    @Override
    public boolean canUpcast(String eventType, int fromVersion) {
        return "OrderPlaced".equals(eventType) && fromVersion == 1;
    }

    @Override
    public DomainEvent upcast(String eventData, ObjectMapper mapper) {
        // v1 이벤트에는 없던 필드를 기본값으로 채워 v2로 변환
        JsonNode node = mapper.readTree(eventData);
        ((ObjectNode) node).put("currency", "KRW"); // 신규 필드 기본값
        return mapper.treeToValue(node, OrderPlacedEventV2.class);
    }
}
```

**운영 복잡성.** 이벤트 스토어, 프로젝터, 읽기 모델 DB를 별도로 운영해야 한다. 모니터링과 장애 대응 포인트가 늘어난다.

### 언제 도입해야 하는가

이벤트 소싱과 CQRS는 강력하지만 모든 시스템에 적합하지 않다. 다음 조건 중 여러 개가 해당될 때 도입을 고려하라.

- 감사 이력이 필수인 도메인 (금융, 의료, 법무)
- 읽기와 쓰기 부하 특성이 극단적으로 다른 시스템
- 다양한 클라이언트가 같은 데이터를 다른 형태로 필요로 하는 경우
- 도메인이 복잡하고 비즈니스 규칙이 자주 바뀌는 경우
- 마이크로서비스 간 이벤트 드리븐 통합이 필요한 경우

반대로 단순한 관리자 도구, 프로토타입, 강한 일관성이 필수인 시스템에는 도입을 피하는 것이 낫다.

---

## 참고 자료

- Greg Young, *CQRS Documents* (2010) — CQRS와 이벤트 소싱의 개념적 원형을 정립한 문서 모음
- Vaughn Vernon, *Implementing Domain-Driven Design* (2013) — 이벤트 소싱과 CQRS의 DDD 맥락에서의 구현 가이드
- Martin Fowler, *Event Sourcing* — https://martinfowler.com/eaaDev/EventSourcing.html
- Axon Framework 공식 문서 — https://docs.axoniq.io — Java 기반 이벤트 소싱/CQRS 프레임워크 레퍼런스
- Microsoft Azure Architecture Center, *CQRS Pattern* — https://learn.microsoft.com/azure/architecture/patterns/cqrs
- Eventuate 플랫폼 문서 — https://eventuate.io/docs — 마이크로서비스 환경에서의 이벤트 소싱 실전 구현
