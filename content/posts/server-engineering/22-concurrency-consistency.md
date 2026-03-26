---
title: "[데이터베이스] 5편 — 동시성과 데이터 정합성: 실전 시나리오"
date: 2026-03-17T23:05:00+09:00
draft: false
tags: ["동시성", "정합성", "멱등성", "Saga", "데이터베이스", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "재고 차감 동시성 문제와 해결 패턴, 중복 결제 방지와 멱등성 키 설계, 분산 환경 정합성(2PC vs Saga), 이벤트 기반 정합성, 유니크 제약 vs 애플리케이션 레벨 중복 체크까지"
---

운영 환경에서 트래픽이 몰릴 때 가장 먼저 터지는 곳이 어디냐고 묻는다면, 십중팔구 재고 차감이나 결제 로직이다.

이론상 완벽해 보이는 코드가 동시 요청 앞에서 속절없이 무너지는 장면을 경험해본 엔지니어라면 알 것이다. 동시성 버그는 재현이 어렵고, 로그에 증거가 남지 않으며, 문제가 터진 뒤에야 원인을 역추적해야 한다.

이 글은 그 경험을 미리 해보는 글이다. 재고 차감부터 중복 결제 방지, 분산 트랜잭션, 이벤트 기반 정합성까지 — 실제로 마주치는 시나리오를 코드 수준에서 파헤친다.

---

## 1. 재고 차감 동시성 문제와 해결 패턴

### 문제의 구조: Lost Update

가장 고전적인 동시성 버그는 `Lost Update`다. 두 트랜잭션이 같은 행을 읽고, 각자 계산한 값을 쓴다. 한 쪽의 업데이트가 다른 쪽에 의해 덮어써진다.

```
T1: SELECT stock FROM products WHERE id = 1;  -- stock = 10
T2: SELECT stock FROM products WHERE id = 1;  -- stock = 10
T1: UPDATE products SET stock = 9 WHERE id = 1;
T2: UPDATE products SET stock = 9 WHERE id = 1;  -- T1의 차감이 사라짐
```

두 요청이 각각 1개씩 차감했는데 결과는 9다. 실제 재고는 8이어야 한다. Read Committed 격리 수준에서는 이 문제가 기본적으로 발생한다.

### 해결책 1: 비관적 락 (Pessimistic Lock)

`SELECT ... FOR UPDATE`는 행에 배타적 락을 건다. 락을 획득한 트랜잭션이 커밋하거나 롤백할 때까지 다른 트랜잭션은 대기한다.

```sql
-- MySQL / PostgreSQL 공통
BEGIN;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;
-- 여기서 다른 트랜잭션은 동일 행에 대해 대기
UPDATE products SET stock = stock - 1 WHERE id = 1 AND stock > 0;
COMMIT;
```

Spring Data JPA에서는 `@Lock` 어노테이션으로 처리한다.

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithLock(@Param("id") Long id);
}

@Service
@Transactional
public class StockService {

    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findByIdWithLock(productId)
            .orElseThrow(() -> new IllegalArgumentException("상품 없음"));

        if (product.getStock() < quantity) {
            throw new InsufficientStockException("재고 부족");
        }

        product.decreaseStock(quantity);
    }
}
```

비관적 락의 장점은 구현이 단순하고 정합성이 확실하다는 점이다. 단점은 락 경합이 심할 때 처리량이 급격히 떨어진다는 것이다. 락을 기다리는 트랜잭션이 쌓이고, 타임아웃이 발생하고, 데드락 위험도 있다.

### 해결책 2: 낙관적 락 (Optimistic Lock)

낙관적 락은 "충돌이 드물 것"이라고 가정한다. 버전 번호를 이용해 내가 읽은 시점과 쓰는 시점 사이에 다른 트랜잭션이 수정했는지 확인한다.

```sql
-- version 컬럼 추가
ALTER TABLE products ADD COLUMN version INT NOT NULL DEFAULT 0;

-- 업데이트 시 version 조건 포함
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;

-- 영향받은 행이 0이면 → 다른 트랜잭션이 먼저 수정한 것
```

JPA에서는 `@Version` 하나면 된다.

```java
@Entity
public class Product {

    @Id
    private Long id;

    private int stock;

    @Version
    private int version;

    public void decreaseStock(int quantity) {
        if (this.stock < quantity) {
            throw new InsufficientStockException("재고 부족");
        }
        this.stock -= quantity;
    }
}
```

충돌 시 JPA가 `OptimisticLockException`을 던진다. 이를 잡아서 재시도하는 로직을 작성해야 한다.

```java
@Service
public class StockService {

    @Retryable(
        value = OptimisticLockException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100)
    )
    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow();
        product.decreaseStock(quantity);
    }

    @Recover
    public void recover(OptimisticLockException e, Long productId, int quantity) {
        throw new StockConflictException("재고 처리 실패, 재시도 횟수 초과");
    }
}
```

낙관적 락은 경합이 낮을 때 매우 효과적이다. 락 대기가 없기 때문에 동시 처리량이 높다.

반면 경합이 높으면 재시도가 폭발적으로 늘어나 오히려 성능이 나빠진다. 플래시 세일처럼 특정 상품에 요청이 집중되는 상황에서는 비관적 락이 낫다.

### 해결책 3: 원자적 UPDATE

사실 가장 깔끔한 해결책은 애플리케이션에서 재고를 읽지 않는 것이다. 조건을 포함한 UPDATE 하나로 처리한다.

```sql
UPDATE products
SET stock = stock - :quantity
WHERE id = :id AND stock >= :quantity;
```

`affected rows`가 0이면 재고 부족이다. 이 방식은 단일 DB 연산이 원자적임을 이용한다. 추가 락이나 버전 컬럼 없이도 정합성이 보장된다.

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Modifying
    @Query("UPDATE Product p SET p.stock = p.stock - :quantity " +
           "WHERE p.id = :id AND p.stock >= :quantity")
    int decreaseStock(@Param("id") Long id, @Param("quantity") int quantity);
}

@Service
@Transactional
public class StockService {

    public void decreaseStock(Long productId, int quantity) {
        int updated = productRepository.decreaseStock(productId, quantity);
        if (updated == 0) {
            throw new InsufficientStockException("재고 부족 또는 상품 없음");
        }
    }
}
```

### 해결책 4: Redis를 이용한 분산 락

여러 서버 인스턴스가 동시에 동작하는 환경에서는 DB 락만으로 부족할 때가 있다. 특히 재고 확인이 여러 단계에 걸쳐 있고 각 단계 사이에 외부 API 호출이 있는 경우다.

Redisson의 분산 락을 사용하면 JVM 경계를 넘어 락을 제어할 수 있다.

```java
@Service
public class StockService {

    private final RedissonClient redissonClient;
    private final ProductRepository productRepository;

    public void decreaseStock(Long productId, int quantity) {
        String lockKey = "stock:lock:" + productId;
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // 최대 3초 대기, 락 유지 10초
            boolean acquired = lock.tryLock(3, 10, TimeUnit.SECONDS);
            if (!acquired) {
                throw new LockAcquisitionException("락 획득 실패");
            }

            // 이제 이 블록은 한 번에 하나의 인스턴스만 실행
            Product product = productRepository.findById(productId).orElseThrow();
            product.decreaseStock(quantity);
            productRepository.save(product);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

Redis 분산 락은 강력하지만 Redis 장애 시 전체 재고 차감이 불가능해지는 단일 장애 지점이 될 수 있다. 락 유지 시간(leaseTime)을 너무 짧게 설정하면 비즈니스 로직 실행 중에 락이 풀리는 문제가 생기므로 충분히 여유 있게 설정해야 한다.

---

## 2. 중복 결제/요청 방지와 멱등성 키 설계

### 왜 멱등성이 필요한가

HTTP는 신뢰할 수 없는 네트워크 위에서 동작한다. 클라이언트가 결제 요청을 보냈는데 응답이 오지 않으면? 타임아웃인지, 서버가 처리했는지 알 수 없다.

클라이언트는 재시도한다. 서버가 실제로 처리했다면 결제가 두 번 된다.

멱등성(Idempotency)은 같은 요청을 여러 번 보내도 결과가 한 번 보낸 것과 동일하다는 성질이다. 엘리베이터 버튼을 열 번 눌러도 한 번만 도착하는 것처럼, 재시도가 가능한 연산으로 만드는 개념이다. 결제, 포인트 적립, 쿠폰 사용 — 한 번만 실행되어야 하는 모든 연산에 멱등성이 필요하다.

### 멱등성 키 구조 설계

멱등성 키는 클라이언트가 요청마다 고유한 값을 생성해서 헤더에 포함한다.

```
POST /payments HTTP/1.1
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
  "orderId": 12345,
  "amount": 50000
}
```

서버는 이 키를 기준으로 요청이 이미 처리됐는지 확인한다.

```sql
CREATE TABLE idempotency_keys (
    idempotency_key VARCHAR(100) PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,  -- 'payment', 'coupon', etc.
    request_hash    VARCHAR(64) NOT NULL,   -- 요청 본문의 SHA-256
    response_status INT,
    response_body   TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at      TIMESTAMP NOT NULL,
    INDEX idx_user_resource (user_id, resource_type)
);
```

`request_hash`를 저장하는 이유가 중요하다. 같은 키로 다른 본문을 보내는 경우를 탐지하기 위해서다.

예를 들어 키는 같지만 금액이 다른 두 요청이 들어오면 이는 클라이언트 버그다. 요청 본문이 다르면 에러를 반환해야 한다.

### Spring에서 멱등성 처리 구현

```java
@Component
public class IdempotencyFilter implements Filter {

    private final IdempotencyKeyRepository repository;
    private final ObjectMapper objectMapper;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpReq = (HttpServletRequest) request;
        String idempotencyKey = httpReq.getHeader("Idempotency-Key");

        if (idempotencyKey == null) {
            chain.doFilter(request, response);
            return;
        }

        // 캐싱된 응답 조회
        Optional<IdempotencyKey> existing = repository.findById(idempotencyKey);
        if (existing.isPresent()) {
            IdempotencyKey record = existing.get();

            // 요청 본문 해시 비교
            String currentHash = computeHash(httpReq);
            if (!record.getRequestHash().equals(currentHash)) {
                ((HttpServletResponse) response).setStatus(422);
                return;
            }

            // 이전 응답 반환
            HttpServletResponse httpResp = (HttpServletResponse) response;
            httpResp.setStatus(record.getResponseStatus());
            httpResp.getWriter().write(record.getResponseBody());
            return;
        }

        // 처음 요청 — 처리 후 저장
        CachedBodyHttpServletRequest wrappedReq =
            new CachedBodyHttpServletRequest(httpReq);
        CachedBodyHttpServletResponse wrappedResp =
            new CachedBodyHttpServletResponse((HttpServletResponse) response);

        chain.doFilter(wrappedReq, wrappedResp);

        // 결과 저장 (2xx 응답만)
        if (wrappedResp.getStatus() >= 200 && wrappedResp.getStatus() < 300) {
            IdempotencyKey newRecord = IdempotencyKey.builder()
                .key(idempotencyKey)
                .requestHash(computeHash(httpReq))
                .responseStatus(wrappedResp.getStatus())
                .responseBody(wrappedResp.getCapturedBody())
                .expiresAt(LocalDateTime.now().plusHours(24))
                .build();
            repository.save(newRecord);
        }
    }
}
```

### 동시 중복 요청 처리: Race Condition

멱등성 구현에서 간과하기 쉬운 부분이 있다. 네트워크 지연으로 인해 동일한 키의 두 요청이 거의 동시에 도착할 수 있다는 것이다. 두 요청 모두 "처음 요청"으로 판단하고 실제 결제가 두 번 실행될 수 있다.

해결책은 DB 유니크 제약을 이용하는 것이다.

```sql
-- idempotency_keys 테이블의 PRIMARY KEY가 자연스럽게 유니크 제약을 강제한다.
-- 두 번째 INSERT는 DuplicateKeyException을 발생시킨다.
INSERT INTO idempotency_keys (idempotency_key, ...) VALUES (...);
```

또는 Redis를 이용한 분산 락으로 처리한다.

```java
public PaymentResponse processPayment(String idempotencyKey, PaymentRequest request) {
    String lockKey = "idempotency:lock:" + idempotencyKey;
    RLock lock = redissonClient.getLock(lockKey);

    try {
        lock.lock(5, TimeUnit.SECONDS);

        // 락 획득 후 다시 확인
        Optional<IdempotencyKey> existing = repository.findById(idempotencyKey);
        if (existing.isPresent()) {
            return deserialize(existing.get().getResponseBody());
        }

        PaymentResponse result = paymentGateway.charge(request);
        saveIdempotencyKey(idempotencyKey, request, result);
        return result;

    } finally {
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
}
```

### 멱등성 키 만료 전략

멱등성 키를 영구히 저장하면 테이블이 비대해진다. 적절한 만료 전략이 필요하다.

- **결제**: 24~48시간. 클라이언트 재시도 창이 보통 이 범위 안에 있다.
- **쿠폰/포인트**: 7일. 배치 재처리 가능성을 고려한다.
- **주문 생성**: 1시간. 재시도 창이 짧다.

만료된 키는 배치 잡으로 정기적으로 삭제하면 된다.

---

## 3. 분산 환경 정합성: 2PC vs Saga

### 분산 트랜잭션의 근본 문제

단일 DB에서 트랜잭션은 ACID를 보장한다. 하지만 마이크로서비스 아키텍처에서는 주문 서비스, 결제 서비스, 재고 서비스가 각자의 DB를 가진다. 주문 생성, 결제 처리, 재고 차감이 모두 성공하거나 모두 실패해야 한다. 이를 어떻게 보장할까?

```
[주문 서비스 DB]   [결제 서비스 DB]   [재고 서비스 DB]
   orders table      payments table    inventory table
       ↕                  ↕                  ↕
   주문 서비스         결제 서비스          재고 서비스
```

### 2PC (Two-Phase Commit)

2PC는 코디네이터와 참여자 간의 2단계 프로토콜이다.

```
코디네이터                참여자 (주문, 결제, 재고)
     |
     |--- PREPARE ------→  각 서비스가 트랜잭션 준비
     |←-- YES/NO ---------  준비 완료 또는 실패 응답
     |
     | (모두 YES면)
     |--- COMMIT -------→  각 서비스가 커밋
     |
     | (하나라도 NO면)
     |--- ROLLBACK -----→  각 서비스가 롤백
```

2PC의 치명적인 약점은 **블로킹 프로토콜**이라는 점이다. PREPARE 완료 후 코디네이터가 다운되면, 참여자들은 락을 쥔 채로 코디네이터가 복구될 때까지 기다린다. 이 시간 동안 해당 리소스는 완전히 잠긴다.

또한 마이크로서비스 환경에서 각 서비스의 DB에 분산 트랜잭션 프로토콜을 강제하려면 XA 트랜잭션이 필요한데, 이는 성능 오버헤드가 크고 NoSQL DB나 메시지 브로커는 지원조차 하지 않는 경우가 많다.

### Saga 패턴

Saga는 긴 트랜잭션을 연속된 로컬 트랜잭션으로 분리한다. 각 로컬 트랜잭션이 성공하면 다음 단계를 트리거하고, 실패하면 **보상 트랜잭션(Compensating Transaction)**으로 이전 단계를 되돌린다.

```
주문 생성 → 결제 처리 → 재고 차감 → 배송 요청
   ↓실패        ↓실패       ↓실패
주문 취소    결제 환불    재고 복구
```

Saga에는 두 가지 구현 방식이 있다.

#### Choreography (안무) 방식

각 서비스가 이벤트를 구독하고 발행한다. 중앙 코디네이터가 없다.

```
주문서비스 → [OrderCreated 이벤트] → 결제서비스
결제서비스 → [PaymentCompleted 이벤트] → 재고서비스
재고서비스 → [StockReserved 이벤트] → 배송서비스

결제 실패 시:
결제서비스 → [PaymentFailed 이벤트] → 주문서비스
주문서비스 → 주문 취소 처리
```

```java
// 결제 서비스
@KafkaListener(topics = "order.created")
public void handleOrderCreated(OrderCreatedEvent event) {
    try {
        Payment payment = paymentService.charge(event.getOrderId(), event.getAmount());
        eventPublisher.publish(new PaymentCompletedEvent(event.getOrderId(), payment.getId()));
    } catch (PaymentException e) {
        eventPublisher.publish(new PaymentFailedEvent(event.getOrderId(), e.getMessage()));
    }
}

// 주문 서비스 (보상)
@KafkaListener(topics = "payment.failed")
public void handlePaymentFailed(PaymentFailedEvent event) {
    orderService.cancelOrder(event.getOrderId(), "결제 실패: " + event.getReason());
}
```

Choreography 방식은 느슨한 결합이 장점이지만, 전체 흐름을 추적하기 어렵고 디버깅이 복잡해진다.

#### Orchestration (오케스트레이션) 방식

중앙 Saga 오케스트레이터가 각 서비스에 명령을 내리고 응답을 처리한다.

```java
@Component
public class OrderSaga {

    public void execute(CreateOrderCommand command) {
        SagaContext ctx = new SagaContext(command.getOrderId());

        try {
            // Step 1: 주문 생성
            orderService.createOrder(command);
            ctx.addCompensation(() -> orderService.cancelOrder(command.getOrderId()));

            // Step 2: 결제
            paymentService.charge(command.getOrderId(), command.getAmount());
            ctx.addCompensation(() -> paymentService.refund(command.getOrderId()));

            // Step 3: 재고 차감
            inventoryService.reserve(command.getOrderId(), command.getItems());
            ctx.addCompensation(() -> inventoryService.release(command.getOrderId()));

            // Step 4: 배송 요청
            shippingService.schedule(command.getOrderId());

        } catch (Exception e) {
            // 실패 지점까지의 보상 트랜잭션 역순 실행
            ctx.compensate();
            throw new SagaExecutionException("Saga 실패", e);
        }
    }
}

public class SagaContext {
    private final Deque<Runnable> compensations = new ArrayDeque<>();

    public void addCompensation(Runnable compensation) {
        compensations.push(compensation);
    }

    public void compensate() {
        while (!compensations.isEmpty()) {
            try {
                compensations.pop().run();
            } catch (Exception e) {
                // 보상 트랜잭션 실패 로깅 — 수동 개입 필요
                log.error("보상 트랜잭션 실패", e);
            }
        }
    }
}
```

### Saga의 약점: 보상 트랜잭션이 실패하면?

보상 트랜잭션도 실패할 수 있다. 결제 환불 API가 네트워크 오류로 실패하면? 이 경우는 **자동 복구가 불가능**하다. Dead Letter Queue에 이벤트를 보내고, 알림을 발송하고, 수동 개입이 필요하다. 이것이 Saga를 사용할 때 반드시 고려해야 할 현실이다.

보상이 실패해도 재시도가 가능하도록 **보상 트랜잭션도 멱등하게** 설계해야 한다.

---

## 4. 이벤트 기반 정합성과 결과적 일관성 허용 범위

### 결과적 일관성(Eventual Consistency)이란

분산 시스템에서 모든 연산을 강한 일관성으로 처리하려면 성능을 심각하게 희생해야 한다. 결과적 일관성은 타협점이다. "지금 당장은 다를 수 있지만, 충분한 시간이 지나면 일치한다."

문제는 "어느 정도의 불일치"를 허용할 것인지를 비즈니스 요구사항에서 결정해야 한다는 점이다.

- **절대 허용 불가**: 이중 결제, 재고 초과 판매
- **허용 가능**: 포인트 잔액이 1~2초 늦게 반영됨, 상품 조회수가 실시간이 아님
- **정책적 허용**: 쿠폰 중복 사용 시 사후 보정

### Transactional Outbox 패턴

이벤트 기반 시스템에서 가장 흔한 버그는 "DB는 업데이트했는데 이벤트는 발행이 안 된 경우"다. DB 트랜잭션과 메시지 발행이 원자적이지 않기 때문이다.

```java
// 잘못된 구현 — 이 둘은 원자적이지 않다
@Transactional
public void createOrder(OrderRequest request) {
    orderRepository.save(order);
    // 여기서 서버가 죽으면? DB에는 저장됐지만 이벤트는 발행되지 않음
    kafkaTemplate.send("order.created", new OrderCreatedEvent(order.getId()));
}
```

Transactional Outbox 패턴은 이 문제를 해결한다. 이벤트를 외부 브로커가 아닌 같은 DB 트랜잭션에 함께 저장한다.

```sql
CREATE TABLE outbox_events (
    id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    aggregate_id BIGINT NOT NULL,
    event_type   VARCHAR(100) NOT NULL,
    payload      JSON NOT NULL,
    status       ENUM('PENDING', 'PUBLISHED', 'FAILED') DEFAULT 'PENDING',
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMP,
    INDEX idx_status_created (status, created_at)
);
```

```java
@Transactional
public void createOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));

    // 같은 트랜잭션 안에서 outbox에 이벤트 저장
    OutboxEvent event = OutboxEvent.builder()
        .aggregateId(order.getId())
        .eventType("ORDER_CREATED")
        .payload(objectMapper.writeValueAsString(new OrderCreatedEvent(order)))
        .build();
    outboxRepository.save(event);
    // 트랜잭션 커밋 — 두 저장이 원자적으로 처리됨
}
```

별도의 Outbox Relay가 PENDING 이벤트를 폴링해서 Kafka에 발행하고 PUBLISHED로 업데이트한다.

```java
@Scheduled(fixedDelay = 1000)
@Transactional
public void relayOutboxEvents() {
    List<OutboxEvent> pendingEvents = outboxRepository
        .findByStatusOrderByCreatedAtAsc(EventStatus.PENDING, PageRequest.of(0, 100));

    for (OutboxEvent event : pendingEvents) {
        try {
            kafkaTemplate.send(topicFor(event.getEventType()), event.getPayload());
            event.markPublished();
        } catch (Exception e) {
            event.markFailed();
            log.error("이벤트 발행 실패: {}", event.getId(), e);
        }
    }
}
```

Debezium 같은 CDC(Change Data Capture) 도구를 사용하면 폴링 대신 DB 변경 로그(binlog)를 실시간으로 읽어서 이벤트를 발행할 수 있다. 폴링 방식보다 지연이 훨씬 짧다.

### 이벤트 소비의 멱등성

이벤트가 적어도 한 번(at-least-once) 배달되는 메시지 브로커를 사용할 때, 소비자는 중복 이벤트를 처리할 수 있어야 한다.

```java
@KafkaListener(topics = "order.created")
public void handleOrderCreated(OrderCreatedEvent event) {
    // 이미 처리된 이벤트인지 확인
    if (processedEventRepository.existsById(event.getEventId())) {
        log.info("중복 이벤트 무시: {}", event.getEventId());
        return;
    }

    // 처리 + 처리 완료 기록을 같은 트랜잭션으로
    processOrderCreatedEvent(event);
    processedEventRepository.save(new ProcessedEvent(event.getEventId()));
}
```

### 결과적 일관성 허용 범위 정의 기준

```
                    일관성 강도
         강함 ◄──────────────────► 약함
          |                          |
         2PC          Saga        Eventually
          |             |          Consistent
          |             |              |
        낮은 성능    중간 성능       높은 성능
        높은 결합    느슨한 결합    매우 느슨한 결합
```

허용 범위를 결정하는 질문들:
1. 불일치가 발생했을 때 비즈니스 피해는 얼마나 큰가?
2. 사후 보정이 가능한가?
3. 고객이 즉시 인지하는 데이터인가?
4. 법적/금융적 요구사항이 있는가?

---

## 5. 유니크 제약 vs 애플리케이션 레벨 중복 체크

### 두 방법의 차이

같은 이메일로 두 계정이 만들어지는 것을 막고 싶다. 방법은 두 가지다.

**방법 A: 애플리케이션 레벨 체크**
```java
// SELECT → INSERT 사이에 레이스 컨디션 존재
public void register(String email, String password) {
    if (userRepository.existsByEmail(email)) {
        throw new DuplicateEmailException();
    }
    userRepository.save(new User(email, password));
}
```

**방법 B: DB 유니크 제약**
```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

```java
public void register(String email, String password) {
    try {
        userRepository.save(new User(email, password));
    } catch (DataIntegrityViolationException e) {
        throw new DuplicateEmailException();
    }
}
```

방법 A는 간단해 보이지만 동시에 두 요청이 들어오면 둘 다 `existsByEmail`에서 false를 받고 모두 INSERT를 시도한다. DB 유니크 제약이 없다면 중복 계정이 만들어진다.

DB 유니크 제약은 DB가 직접 중복을 막아주므로 레이스 컨디션이 없다. 그러나 예외를 애플리케이션에서 잡아서 적절한 비즈니스 예외로 변환해야 한다.

### 복합 유니크 제약의 함정

쿠폰 발행 테이블에서 사용자당 쿠폰 1개 발행을 보장하고 싶다면:

```sql
CREATE UNIQUE INDEX idx_coupons_user_type
ON user_coupons(user_id, coupon_type);
```

그런데 소프트 삭제(soft delete)를 사용한다면?

```sql
-- deleted_at이 NULL인 것만 유니크하길 원하지만...
CREATE UNIQUE INDEX idx_coupons_user_type
ON user_coupons(user_id, coupon_type);

-- 삭제 후 재발급하면 UNIQUE 제약 위반 발생
UPDATE user_coupons SET deleted_at = NOW() WHERE id = 1;
INSERT INTO user_coupons (user_id, coupon_type) VALUES (1, 'WELCOME');
-- ERROR: Duplicate entry
```

해결책은 삭제 여부를 유니크 인덱스에 포함하는 것이다. MySQL에서는 NULL 값이 유니크 제약에서 중복으로 간주되지 않는 특성을 활용할 수 있다.

```sql
-- MySQL: deleted_at이 NULL인 경우만 유니크 보장
CREATE UNIQUE INDEX idx_coupons_active
ON user_coupons(user_id, coupon_type, deleted_at);
-- deleted_at이 NULL이면 (user_id, coupon_type, NULL)로 비교 → NULL은 유니크

-- PostgreSQL: 부분 인덱스 사용
CREATE UNIQUE INDEX idx_coupons_active
ON user_coupons(user_id, coupon_type)
WHERE deleted_at IS NULL;
```

PostgreSQL의 부분 인덱스(Partial Index)가 의미적으로 더 명확하다. "삭제되지 않은 행들 사이에서만 유니크"를 정확히 표현한다.

### 언제 애플리케이션 레벨 체크를 써야 하나

DB 유니크 제약이 항상 답은 아니다.

**DB 유니크 제약이 부적합한 경우:**
- 조건이 복잡해서 인덱스로 표현하기 어려울 때
- 외부 시스템 데이터와의 중복 체크가 필요할 때
- 중복 판단 로직이 시간에 따라 바뀔 때 (예: "7일 이내 동일 주소로 배송 불가")

```java
// 7일 이내 동일 주소 배송 제한 — DB 유니크 제약으로 표현 불가
public void createShipping(Long orderId, String address) {
    LocalDateTime cutoff = LocalDateTime.now().minusDays(7);
    boolean recentExists = shippingRepository.existsByAddressAndCreatedAtAfter(address, cutoff);

    if (recentExists) {
        throw new DuplicateShippingException("7일 이내 동일 주소 배송 내역 존재");
    }

    shippingRepository.save(new Shipping(orderId, address));
}
```

이 경우에도 레이스 컨디션은 존재한다. 빈도가 낮고, 중복 발생 시 사후 처리가 가능하다면 애플리케이션 레벨 체크로 충분하다. 중복이 심각한 비즈니스 피해를 유발한다면 별도의 분산 락이나 DB 레벨 직렬화가 필요하다.

### INSERT ... ON DUPLICATE KEY 패턴

중복 시 무시하거나 업데이트하는 패턴도 유용하다.

```sql
-- MySQL: 중복 시 업데이트
INSERT INTO user_coupons (user_id, coupon_type, issued_at)
VALUES (1, 'WELCOME', NOW())
ON DUPLICATE KEY UPDATE issued_at = issued_at;  -- 아무것도 바꾸지 않음 (멱등)

-- PostgreSQL: ON CONFLICT
INSERT INTO user_coupons (user_id, coupon_type, issued_at)
VALUES (1, 'WELCOME', NOW())
ON CONFLICT (user_id, coupon_type) DO NOTHING;
```

이 패턴은 분산 환경에서도 안전하게 "처음이면 삽입, 이미 있으면 무시"를 DB 레벨에서 원자적으로 처리한다. 멱등 삽입이 필요한 모든 곳에서 유용하다.

---

## 정합성 전략 선택 가이드

```
요청 특성                   권장 전략
───────────────────────────────────────────────────────────
단일 DB, 낮은 경합           낙관적 락 + 재시도
단일 DB, 높은 경합           비관적 락 또는 원자적 UPDATE
분산 DB, 강한 일관성 필요    Saga (Orchestration) + 보상
분산 DB, 결과적 일관성 허용  Choreography + Outbox 패턴
멱등 보장 필요               멱등성 키 + DB 유니크 제약
중복 삽입 방지               INSERT ON CONFLICT + 유니크 인덱스
```

동시성과 정합성은 단 하나의 "올바른" 해법이 없다. 비즈니스 요구사항, 트래픽 특성, 허용 가능한 복잡도에 따라 가장 단순하면서도 충분한 방법을 선택하는 것이 엔지니어링의 핵심이다.

과한 보장을 위해 성능을 버리지 말고, 성능을 위해 비즈니스 정합성을 희생하지도 말라. 그 균형점을 찾는 것이 이 분야에서 경험이 만드는 차이다.

---

## 참고 자료

1. **Designing Data-Intensive Applications** — Martin Kleppmann. O'Reilly, 2017. 분산 시스템 정합성과 트랜잭션의 바이블.
2. **Database Internals** — Alex Petrov. O'Reilly, 2019. 락, MVCC, 분산 트랜잭션의 내부 구조.
3. **Microservices Patterns** — Chris Richardson. Manning, 2018. Saga 패턴과 Transactional Outbox의 레퍼런스 구현.
4. **Release It!** — Michael T. Nygard. Pragmatic Bookshelf, 2018. 운영 환경에서의 안정성 패턴.
5. **Stripe API 문서 — Idempotency Keys**: https://stripe.com/docs/idempotency. 멱등성 키 설계의 실전 레퍼런스.
6. **Google Cloud Spanner 문서 — Distributed Transactions**: https://cloud.google.com/spanner/docs/transactions. 분산 트랜잭션 구현 사례.
