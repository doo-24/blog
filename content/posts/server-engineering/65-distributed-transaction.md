---
title: "[분산 시스템과 MSA] 7편 — 분산 트랜잭션과 멱등성: 여러 서비스에 걸친 일관성"
date: 2026-03-17T17:01:00+09:00
draft: false
tags: ["Saga", "2PC", "멱등성", "보상 트랜잭션", "서버"]
series: ["분산 시스템과 MSA"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 9
summary: "2PC(2-Phase Commit) 원리와 한계, Saga 패턴(Choreography vs Orchestration) 실전 설계, 결과적 일관성과 보상 트랜잭션, 멱등성 설계 심화(멱등키 저장소, TTL, 클라이언트 재시도 계약)까지"
---

주문을 처리하는 MSA 시스템을 떠올려 보자. 고객이 결제를 완료했는데 재고 서비스가 응답하지 않는다면 어떻게 해야 할까? 결제는 됐는데 상품이 출고되지 않거나, 반대로 재고는 줄었는데 결제가 취소되는 상황이 발생한다. 단일 데이터베이스라면 롤백 한 줄로 해결되지만, 서비스가 분리된 순간 그 단순함은 사라진다. 분산 트랜잭션은 MSA의 가장 어려운 문제 중 하나이고, 이를 올바르게 설계하는 것이 시스템 신뢰성의 핵심이다.

---

## 1. 2PC (2-Phase Commit): 강한 일관성의 대가

### 2PC의 작동 원리

2PC는 분산 환경에서 원자성(Atomicity)을 보장하는 고전적인 프로토콜이다.

이름 그대로 두 단계로 구성된다.

**Phase 1 — Prepare (준비 단계)**

코디네이터(Coordinator)가 모든 참여자(Participant)에게 "커밋할 준비가 됐는가?"를 묻는다.

각 참여자는 트랜잭션을 실행하고 결과를 로컬에 임시 저장한 뒤 `VOTE_COMMIT` 또는 `VOTE_ABORT`로 응답한다.

**Phase 2 — Commit (확정 단계)**

모든 참여자가 `VOTE_COMMIT`이면 코디네이터가 `GLOBAL_COMMIT`을 전송한다.

하나라도 `VOTE_ABORT`이거나 타임아웃이 발생하면 `GLOBAL_ABORT`를 전송한다.

```text
Coordinator         Participant A        Participant B
     │                   │                    │
     │── PREPARE ────────►│                    │
     │── PREPARE ─────────────────────────────►│
     │                   │                    │
     │◄── VOTE_COMMIT ───│                    │
     │◄── VOTE_COMMIT ────────────────────────│
     │                   │                    │
     │  (모두 OK → COMMIT 결정)                │
     │                   │                    │
     │── COMMIT ─────────►│                    │
     │── COMMIT ───────────────────────────────►│
     │                   │                    │
     │◄── ACK ───────────│                    │
     │◄── ACK ────────────────────────────────│
```

### 2PC의 심각한 한계

2PC는 이론적으로 완벽해 보이지만 실전에서는 치명적인 문제를 가진다.

**블로킹(Blocking) 문제**

코디네이터가 Phase 2 도중 장애가 나면, 참여자들은 `VOTE_COMMIT`을 보낸 상태로 락(lock)을 잡은 채 무한 대기한다.

코디네이터가 복구될 때까지 해당 리소스는 완전히 차단된다. 이것이 2PC를 "블로킹 프로토콜"이라 부르는 이유다.

**성능 문제**

참여자 수가 늘어날수록 네트워크 왕복이 선형으로 증가한다.

모든 참여자가 응답할 때까지 기다리므로, 가장 느린 참여자가 전체 트랜잭션의 지연을 결정한다.

**마이크로서비스와의 부적합성**

2PC는 참여자들이 동일한 트랜잭션 매니저를 공유한다고 가정한다.

MSA에서 각 서비스는 독립적인 데이터베이스를 가지며, 서비스 간 직접 트랜잭션 조정은 결합도를 극단적으로 높인다.

### 안티패턴: MSA에서 2PC 강행

```java
// 절대 이렇게 하지 말 것 — 서비스 간 XA 트랜잭션
@Transactional
public void placeOrder(OrderRequest request) {
    // 주문 DB와 결제 DB, 재고 DB를 하나의 XA 트랜잭션으로 묶으려는 시도
    orderRepository.save(order);                    // DB 1
    paymentServiceClient.charge(request.getPayment()); // DB 2 — 외부 서비스!
    inventoryServiceClient.reserve(request.getItems()); // DB 3 — 외부 서비스!
}
```

이 코드는 서비스 경계를 무시한다. 네트워크 장애 시 XA 코디네이터가 락을 점유한 채 멈추며, 서비스 독립 배포가 불가능해진다.

---

## 2. Saga 패턴: 결과적 일관성으로의 전환

### Saga의 핵심 아이디어

Saga는 긴 트랜잭션을 여러 개의 짧은 로컬 트랜잭션 시퀀스로 분해한다.

각 단계가 성공하면 다음 단계로 이벤트를 전달하고, 실패하면 이미 완료된 단계를 되돌리는 **보상 트랜잭션(Compensating Transaction)**을 실행한다.

```text
[정상 흐름]
주문 서비스        결제 서비스        재고 서비스        배송 서비스
    │                  │                  │                  │
    │── 주문 생성 ─────►│                  │                  │
    │                  │── 결제 처리 ─────►│                  │
    │                  │                  │── 재고 차감 ─────►│
    │                  │                  │                  │── 배송 등록
    │                  │                  │                  │   완료

[실패 시 보상 트랜잭션]
주문 서비스        결제 서비스        재고 서비스        배송 서비스
    │                  │                  │                  │
    │── 주문 생성 ─────►│                  │                  │
    │                  │── 결제 처리 ─────►│                  │
    │                  │                  │── 재고 차감 ─────►│
    │                  │                  │                  X (배송 실패)
    │                  │                  │◄── 재고 복원 ─────│
    │                  │◄── 결제 취소 ─────│                  │
    │◄── 주문 취소 ─────│                  │                  │
```

즉각적인 일관성(Immediate Consistency) 대신 **결과적 일관성(Eventual Consistency)**을 추구한다.

### Choreography Saga: 이벤트 기반 협력

각 서비스가 이벤트를 발행하고 구독하는 방식으로 Saga를 조율한다.

중앙 조율자가 없으며 서비스 간 직접 호출도 없다.

```java
// 주문 서비스 — Choreography 방식
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public Order createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);
        order.markPending();
        orderRepository.save(order);

        // 로컬 트랜잭션 완료 후 이벤트 발행
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId(), order.getItems(), order.getTotal()));
        return order;
    }

    // 결제 완료 이벤트 수신
    @EventListener
    @Transactional
    public void on(PaymentCompletedEvent event) {
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.markPaymentConfirmed();
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderConfirmedEvent(order.getId()));
    }

    // 결제 실패 이벤트 수신 — 보상 트랜잭션
    @EventListener
    @Transactional
    public void on(PaymentFailedEvent event) {
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.cancel("결제 실패");
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCancelledEvent(order.getId()));
    }
}
```

```java
// 결제 서비스 — 주문 생성 이벤트 처리
@Service
public class PaymentService {

    private final PaymentRepository paymentRepository;
    private final ApplicationEventPublisher eventPublisher;

    @EventListener
    @Transactional
    public void on(OrderCreatedEvent event) {
        try {
            Payment payment = processPayment(event.getOrderId(), event.getTotal());
            paymentRepository.save(payment);
            eventPublisher.publishEvent(new PaymentCompletedEvent(event.getOrderId(), payment.getId()));
        } catch (InsufficientFundsException e) {
            eventPublisher.publishEvent(new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
        }
    }

    // 주문 취소 이벤트 수신 — 결제 환불 보상 트랜잭션
    @EventListener
    @Transactional
    public void on(OrderCancelledEvent event) {
        paymentRepository.findByOrderId(event.getOrderId())
            .ifPresent(payment -> {
                payment.refund();
                paymentRepository.save(payment);
            });
    }
}
```

**Choreography의 장단점**

장점: 서비스 간 결합도가 낮고 새로운 서비스를 추가하기 쉽다.

단점: 전체 흐름을 한눈에 파악하기 어렵고, 순환 이벤트(Cyclic Event) 문제가 발생할 수 있다.

### Orchestration Saga: 중앙 조율자

Saga Orchestrator가 각 서비스에 명시적으로 명령을 내리고 응답을 처리한다.

전체 비즈니스 흐름이 한 곳에 집중되어 추적과 디버깅이 용이하다.

```java
// Saga Orchestrator 구현 (Eventuate Tram 스타일)
@Component
public class OrderSagaOrchestrator {

    public SagaDefinition<OrderSagaData> sagaDefinition() {
        return step()
            .invokeParticipant(this::reserveInventory)
            .withCompensation(this::cancelInventoryReservation)
        .step()
            .invokeParticipant(this::processPayment)
            .withCompensation(this::refundPayment)
        .step()
            .invokeParticipant(this::confirmOrder)
        .build();
    }

    private CommandWithDestination reserveInventory(OrderSagaData data) {
        return send(new ReserveInventoryCommand(data.getOrderId(), data.getItems()))
            .to("inventory-service");
    }

    private CommandWithDestination cancelInventoryReservation(OrderSagaData data) {
        return send(new CancelInventoryReservationCommand(data.getOrderId()))
            .to("inventory-service");
    }

    private CommandWithDestination processPayment(OrderSagaData data) {
        return send(new ProcessPaymentCommand(data.getOrderId(), data.getTotal()))
            .to("payment-service");
    }

    private CommandWithDestination refundPayment(OrderSagaData data) {
        return send(new RefundPaymentCommand(data.getOrderId()))
            .to("payment-service");
    }

    private CommandWithDestination confirmOrder(OrderSagaData data) {
        return send(new ConfirmOrderCommand(data.getOrderId()))
            .to("order-service");
    }
}
```

**Orchestration의 장단점**

장점: 흐름이 명확하고, 실패 시나리오 관리가 용이하며, 상태 추적이 쉽다.

단점: Orchestrator 자체가 복잡해지고 단일 장애점이 될 수 있다. 서비스 간 결합도가 다소 높아진다.

### Choreography vs Orchestration 선택 기준

| 기준 | Choreography | Orchestration |
|------|-------------|---------------|
| 서비스 수 | 소규모 (3개 이하) | 중대규모 |
| 흐름 복잡도 | 단순 | 복잡한 분기/반복 |
| 디버깅 용이성 | 어려움 | 쉬움 |
| 결합도 | 낮음 | 상대적으로 높음 |
| 신규 참여자 추가 | 쉬움 | Orchestrator 수정 필요 |

---

## 3. 결과적 일관성과 보상 트랜잭션 설계

### 결과적 일관성의 현실

결과적 일관성은 "언젠가는 일관된 상태가 된다"는 보장이다.

그 "언젠가" 사이의 불일치 상태를 시스템이 얼마나 잘 처리하느냐가 설계의 핵심이다. 예를 들어 주문이 생성됐지만 결제 처리가 아직 완료되지 않은 수 초 동안, 사용자는 주문 내역에서 "처리 중" 상태를 볼 수 있다. 이 불일치는 정상이며 허용 가능하다. 문제는 이 기간이 지나치게 길어지거나, 중간 실패 시 불일치가 영구적으로 남는 경우다.

### 보상 트랜잭션의 설계 원칙

보상 트랜잭션은 단순한 롤백이 아니다.

이미 외부에 노출된 부작용(환불, 알림, 외부 API 호출 등)을 되돌리는 **의미론적 되돌리기(Semantic Undo)**다.

```java
// 올바른 보상 트랜잭션 — 단순 DELETE가 아닌 상태 변경
@Service
public class InventoryService {

    @Transactional
    public void reserveStock(String orderId, List<OrderItem> items) {
        for (OrderItem item : items) {
            Stock stock = stockRepository.findByProductId(item.getProductId())
                .orElseThrow(() -> new StockNotFoundException(item.getProductId()));

            if (stock.getAvailable() < item.getQuantity()) {
                throw new InsufficientStockException(item.getProductId());
            }

            stock.reserve(item.getQuantity()); // available 감소, reserved 증가
            stockRepository.save(stock);
        }

        // 예약 이력 저장 — 보상 트랜잭션에서 참조용
        StockReservation reservation = StockReservation.create(orderId, items);
        reservationRepository.save(reservation);
    }

    // 보상 트랜잭션 — 예약 취소
    @Transactional
    public void cancelReservation(String orderId) {
        StockReservation reservation = reservationRepository.findByOrderId(orderId)
            .orElse(null);

        // 멱등성: 이미 취소됐거나 예약이 없으면 조용히 무시
        if (reservation == null || reservation.isCancelled()) {
            return;
        }

        for (ReservationItem item : reservation.getItems()) {
            Stock stock = stockRepository.findByProductId(item.getProductId())
                .orElseThrow();
            stock.releaseReservation(item.getQuantity()); // reserved 감소, available 증가
            stockRepository.save(stock);
        }

        reservation.cancel();
        reservationRepository.save(reservation);
    }
}
```

### 보상 불가능한 트랜잭션 처리

일부 작업은 보상 트랜잭션으로 완전히 되돌릴 수 없다.

예를 들어 이미 발송한 문자 메시지나 외부 결제 승인은 취소 API가 있더라도 고객에게 혼란을 줄 수 있다.

이런 경우에는 **피봇 트랜잭션(Pivot Transaction)** 개념을 적용한다.

Saga의 단계를 보상 가능한 단계(Compensatable)와 재시도 가능한 단계(Retriable)로 구분하고, 피봇 포인트 이후 단계는 반드시 성공하도록 설계한다.

```java
// 피봇 트랜잭션 패턴: 외부 결제 처리 이후는 반드시 완료
@Service
public class ShipmentService {

    // Saga의 마지막 단계 — 반드시 성공해야 하는 Retriable 트랜잭션
    @Transactional
    @Retryable(maxAttempts = 5, backoff = @Backoff(delay = 1000, multiplier = 2))
    public void scheduleShipment(String orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        // 이미 처리된 경우 멱등하게 처리
        if (order.isShipmentScheduled()) {
            return;
        }

        Shipment shipment = Shipment.create(order);
        shipmentRepository.save(shipment);
        order.markShipmentScheduled(shipment.getId());
        orderRepository.save(order);
    }
}
```

### Outbox 패턴으로 이벤트 발행 보장

로컬 트랜잭션과 이벤트 발행의 원자성을 보장하는 핵심 패턴이다.

데이터베이스 트랜잭션 내에서 이벤트를 Outbox 테이블에 함께 저장하고, 별도 프로세스가 이를 읽어 메시지 브로커로 전송한다.

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxEventRepository outboxRepository;

    @Transactional // 주문 저장과 이벤트 저장이 하나의 트랜잭션
    public Order createOrder(CreateOrderCommand command) {
        Order order = Order.create(command);
        orderRepository.save(order);

        // 이벤트를 메시지 브로커가 아닌 DB에 저장
        OutboxEvent event = OutboxEvent.builder()
            .aggregateId(order.getId())
            .aggregateType("Order")
            .eventType("OrderCreated")
            .payload(serialize(new OrderCreatedEvent(order)))
            .build();
        outboxRepository.save(event); // 같은 트랜잭션으로 저장

        return order;
    }
}

// Outbox Poller — 별도 스레드/서비스
@Component
public class OutboxEventPublisher {

    @Scheduled(fixedDelay = 100)
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository.findPendingEvents(100);
        for (OutboxEvent event : pending) {
            try {
                kafkaTemplate.send(event.getTopic(), event.getAggregateId(), event.getPayload());
                event.markPublished();
                outboxRepository.save(event);
            } catch (Exception e) {
                log.error("이벤트 발행 실패: {}", event.getId(), e);
                // 재시도를 위해 상태 변경 없이 통과
            }
        }
    }
}
```

---

## 4. 멱등성 설계 심화

### 멱등성이란 무엇인가

멱등성(Idempotency)은 동일한 요청을 여러 번 실행해도 결과가 같음을 보장하는 성질이다.

분산 시스템에서 네트워크 재시도는 불가피하다. 클라이언트가 타임아웃으로 인해 요청을 재전송하면, 서버는 해당 요청이 처음인지 재시도인지 알 수 없다.

멱등성이 없으면 재시도마다 결제가 중복 청구되거나, 재고가 이중으로 차감된다.

### 멱등키 저장소 설계

멱등성 보장의 핵심은 **클라이언트가 생성한 고유 키(Idempotency Key)**를 서버가 추적하는 것이다.

```java
// 멱등키 저장소 엔티티
@Entity
@Table(name = "idempotency_keys")
public class IdempotencyRecord {

    @Id
    private String idempotencyKey;

    private String requestHash;      // 요청 본문의 해시 — 동일 키로 다른 요청 방지
    private String responseBody;     // 성공 응답 캐시
    private int responseStatus;
    private String status;           // PROCESSING, COMPLETED, FAILED

    @Column(nullable = false)
    private LocalDateTime expiresAt; // TTL

    private LocalDateTime createdAt;
    private LocalDateTime completedAt;
}
```

```java
// 멱등성 필터 — Spring AOP 또는 Filter로 구현
@Component
public class IdempotencyFilter extends OncePerRequestFilter {

    private final IdempotencyRecordRepository repository;
    private final ObjectMapper objectMapper;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws IOException, ServletException {

        String idempotencyKey = request.getHeader("Idempotency-Key");

        // 멱등키가 없으면 멱등성 검사 없이 통과 (GET 등 멱등 메서드)
        if (idempotencyKey == null || !requiresIdempotency(request)) {
            chain.doFilter(request, response);
            return;
        }

        // 캐시된 응답 확인
        Optional<IdempotencyRecord> existing = repository.findById(idempotencyKey);

        if (existing.isPresent()) {
            IdempotencyRecord record = existing.get();

            if ("PROCESSING".equals(record.getStatus())) {
                // 동일 요청이 처리 중 — 409 Conflict 반환
                response.setStatus(409);
                response.getWriter().write("{\"error\": \"요청이 처리 중입니다. 잠시 후 다시 시도하세요.\"}");
                return;
            }

            if ("COMPLETED".equals(record.getStatus())) {
                // 이미 완료된 요청 — 캐시된 응답 반환
                response.setStatus(record.getResponseStatus());
                response.setContentType("application/json");
                response.getWriter().write(record.getResponseBody());
                return;
            }
        }

        // 새 요청 — 처리 시작
        IdempotencyRecord record = IdempotencyRecord.builder()
            .idempotencyKey(idempotencyKey)
            .requestHash(hashRequest(request))
            .status("PROCESSING")
            .expiresAt(LocalDateTime.now().plusHours(24))
            .createdAt(LocalDateTime.now())
            .build();
        repository.save(record);

        // 응답 래핑으로 결과 캡처
        CachedBodyResponseWrapper wrappedResponse = new CachedBodyResponseWrapper(response);
        try {
            chain.doFilter(request, wrappedResponse);

            // 성공 응답 저장
            record.complete(wrappedResponse.getStatus(), wrappedResponse.getCachedBody());
            repository.save(record);

        } catch (Exception e) {
            record.fail();
            repository.save(record);
            throw e;
        }
    }
}
```

### TTL 설계 전략

멱등키 저장소는 영구적으로 유지할 필요가 없다.

적절한 TTL 설정은 저장소 공간을 절약하면서도 합리적인 재시도 창을 제공한다.

```java
@Service
public class IdempotencyKeyService {

    // TTL은 비즈니스 맥락에 따라 다르게 설정
    private static final Map<String, Duration> TTL_POLICY = Map.of(
        "payment",        Duration.ofHours(24),   // 결제: 하루 이내 재시도 허용
        "order",          Duration.ofHours(12),   // 주문: 반일
        "notification",   Duration.ofMinutes(30), // 알림: 짧은 창
        "inventory",      Duration.ofHours(6)     // 재고: 6시간
    );

    public Duration getTtl(String operationType) {
        return TTL_POLICY.getOrDefault(operationType, Duration.ofHours(1));
    }

    // TTL 만료된 키 정리 — 배치 작업
    @Scheduled(cron = "0 0 2 * * *") // 매일 새벽 2시
    @Transactional
    public void cleanupExpiredKeys() {
        int deleted = repository.deleteByExpiresAtBefore(LocalDateTime.now());
        log.info("만료된 멱등키 {}개 삭제 완료", deleted);
    }
}
```

### Redis를 활용한 고성능 멱등키 저장소

DB 기반 멱등키 저장소는 모든 요청에 추가 DB I/O를 유발한다.

Redis를 활용하면 성능을 크게 향상시킬 수 있다.

```java
@Service
public class RedisIdempotencyService {

    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    private static final String KEY_PREFIX = "idem:";
    private static final Duration DEFAULT_TTL = Duration.ofHours(24);

    public IdempotencyResult checkAndSet(String idempotencyKey, Supplier<ResponseEntity<?>> operation) {
        String redisKey = KEY_PREFIX + idempotencyKey;

        // SET NX (Not eXists) — 원자적으로 키 존재 확인 및 PROCESSING 상태 설정
        Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent(redisKey, "PROCESSING", DEFAULT_TTL);

        if (Boolean.FALSE.equals(isNew)) {
            // 이미 존재하는 키
            String value = redisTemplate.opsForValue().get(redisKey);

            if ("PROCESSING".equals(value)) {
                return IdempotencyResult.processing();
            }

            // COMPLETED 상태 — 캐시된 응답 반환
            try {
                CachedResponse cached = objectMapper.readValue(value, CachedResponse.class);
                return IdempotencyResult.cached(cached);
            } catch (Exception e) {
                log.error("멱등키 응답 역직렬화 실패: {}", idempotencyKey, e);
            }
        }

        // 새 요청 처리
        try {
            ResponseEntity<?> result = operation.get();

            // 응답 캐싱
            CachedResponse cached = CachedResponse.of(result);
            redisTemplate.opsForValue().set(redisKey,
                objectMapper.writeValueAsString(cached),
                DEFAULT_TTL);

            return IdempotencyResult.fresh(result);

        } catch (Exception e) {
            // 실패 시 키 삭제 — 클라이언트가 재시도 가능하도록
            redisTemplate.delete(redisKey);
            throw e;
        }
    }
}
```

### 클라이언트 재시도 계약 설계

멱등성은 서버만의 문제가 아니다.

클라이언트가 올바르게 재시도하지 않으면 멱등성 보장이 의미 없어진다.

**클라이언트-서버 재시도 계약**

```java
// 클라이언트 측 재시도 로직
public class IdempotentHttpClient {

    private final HttpClient httpClient;
    private final RetryPolicy retryPolicy;

    public <T> T post(String url, Object body, Class<T> responseType) {
        // 멱등키는 요청 단위로 생성하고 재시도 간에 재사용
        String idempotencyKey = UUID.randomUUID().toString();

        return retryPolicy.execute(() -> {
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(url))
                .header("Content-Type", "application/json")
                .header("Idempotency-Key", idempotencyKey) // 동일 키 재사용
                .POST(HttpRequest.BodyPublishers.ofString(serialize(body)))
                .build();

            HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());

            if (response.statusCode() == 409) {
                // 처리 중 — 짧은 대기 후 재시도
                throw new RetryableException("요청 처리 중");
            }

            if (response.statusCode() >= 500) {
                // 서버 오류 — 재시도 가능
                throw new RetryableException("서버 오류: " + response.statusCode());
            }

            if (response.statusCode() == 200 || response.statusCode() == 201) {
                return deserialize(response.body(), responseType);
            }

            // 4xx (400, 422 등) — 재시도 불가, 즉시 실패
            throw new NonRetryableException("클라이언트 오류: " + response.statusCode());
        });
    }
}

// Exponential Backoff with Jitter 재시도 정책
public class RetryPolicy {

    private static final int MAX_ATTEMPTS = 3;
    private static final Duration BASE_DELAY = Duration.ofMillis(500);
    private static final Duration MAX_DELAY = Duration.ofSeconds(10);
    private final Random random = new Random();

    public <T> T execute(CheckedSupplier<T> operation) {
        int attempt = 0;
        while (attempt < MAX_ATTEMPTS) {
            try {
                return operation.get();
            } catch (RetryableException e) {
                attempt++;
                if (attempt >= MAX_ATTEMPTS) throw e;

                // Exponential Backoff + Jitter — 동시 재시도로 인한 thundering herd 방지
                long delay = Math.min(
                    (long) (BASE_DELAY.toMillis() * Math.pow(2, attempt)),
                    MAX_DELAY.toMillis()
                );
                long jitter = (long) (delay * 0.3 * random.nextDouble());
                sleep(delay + jitter);
            } catch (NonRetryableException e) {
                throw e;
            }
        }
        throw new MaxRetriesExceededException();
    }
}
```

**API 문서에 명시해야 할 재시도 계약**

서버 개발자는 어떤 상황에서 재시도가 안전한지를 API 문서에 명확히 기술해야 한다.

```
POST /api/v1/orders
Headers:
  Idempotency-Key: <클라이언트가 생성한 UUID> (필수)

재시도 가능 상황:
  - 네트워크 타임아웃 (동일 Idempotency-Key로 재시도)
  - HTTP 500, 502, 503, 504

재시도 불가 상황:
  - HTTP 400 (잘못된 요청)
  - HTTP 402 (결제 실패)
  - HTTP 409 (처리 중 — 잠시 후 재시도)
  - HTTP 422 (검증 실패)

멱등키 유효 기간: 24시간
동일 키로 다른 본문 요청 시: HTTP 422 반환
```

### 안티패턴: 멱등성 없는 재시도

```java
// 잘못된 패턴 — 재시도마다 새 UUID 생성
@Retryable(maxAttempts = 3)
public OrderResponse createOrder(CreateOrderRequest request) {
    request.setIdempotencyKey(UUID.randomUUID().toString()); // 매번 새 키!
    return orderServiceClient.createOrder(request);
    // 재시도마다 새로운 주문이 생성됨 — 중복 주문 발생!
}

// 올바른 패턴 — 키를 외부에서 생성하여 전달
public OrderResponse createOrderWithRetry(CreateOrderRequest request) {
    String idempotencyKey = UUID.randomUUID().toString(); // 한 번만 생성
    request.setIdempotencyKey(idempotencyKey);
    return retryTemplate.execute(ctx -> orderServiceClient.createOrder(request));
}
```

---

## 5. 실전 운영: 분산 트랜잭션 모니터링

### Saga 상태 추적

Saga는 여러 서비스에 걸쳐 진행되므로 현재 상태를 추적하기 어렵다.

각 Saga 인스턴스의 상태를 저장하고, 비정상 상태를 감지하는 메커니즘이 필수다.

```java
// Saga 상태 테이블
@Entity
public class SagaInstance {
    @Id
    private String sagaId;
    private String sagaType;    // "order-saga", "refund-saga" 등
    private String currentStep;
    private String status;      // STARTED, COMPENSATING, COMPLETED, FAILED
    private String failureReason;
    private int stepCount;
    private LocalDateTime startedAt;
    private LocalDateTime updatedAt;
    private LocalDateTime completedAt;
}

// 장시간 미완료 Saga 감지 및 알림
@Scheduled(fixedDelay = 60_000) // 1분마다
public void detectStaleSagas() {
    LocalDateTime threshold = LocalDateTime.now().minusMinutes(10);
    List<SagaInstance> staleSagas = sagaRepository
        .findByStatusInAndUpdatedAtBefore(
            List.of("STARTED", "COMPENSATING"), threshold);

    if (!staleSagas.isEmpty()) {
        log.warn("장시간 미완료 Saga 감지: {}건", staleSagas.size());
        alertService.notify("STALE_SAGA", staleSagas);
    }
}
```

---

분산 트랜잭션은 은총알이 없다.

2PC는 강한 일관성을 제공하지만 MSA의 자율성과 충돌한다. Saga는 결과적 일관성이라는 트레이드오프를 감수하되, 각 서비스가 독립적으로 동작할 수 있게 한다. 멱등성은 네트워크 불안정성을 시스템 수준에서 흡수하는 방어 전략이다.

결국 분산 시스템 설계의 핵심은 **장애를 예방하는 것이 아니라, 장애가 발생했을 때 시스템이 스스로 회복하도록 만드는 것**이다. 보상 트랜잭션과 멱등성은 그 회복력의 두 축이다.

---

## 참고 자료

1. **Chris Richardson - "Microservices Patterns"** (Manning, 2018) — Saga 패턴과 Outbox 패턴의 원전 교과서
2. **Gregor Hohpe, Bobby Woolf - "Enterprise Integration Patterns"** — 메시지 기반 통합 패턴의 바이블
3. **Bernd Ruecker - "Practical Process Automation"** (O'Reilly, 2021) — Orchestration Saga와 워크플로우 엔진 실전 적용
4. **Stripe Engineering Blog - "Idempotency"** (https://stripe.com/blog/idempotency) — 결제 시스템에서 멱등성을 구현한 실전 사례
5. **Eventuate Tram 공식 문서** (https://eventuate.io/docs/manual/eventuate-tram/latest) — Spring 기반 Saga 구현 레퍼런스
6. **Martin Kleppmann - "Designing Data-Intensive Applications"** (O'Reilly, 2017) — 결과적 일관성과 분산 트랜잭션의 이론적 배경
