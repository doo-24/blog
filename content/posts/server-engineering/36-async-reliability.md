---
title: "[비동기 처리와 이벤트 드리븐] 3편 — 비동기 패턴과 신뢰성: 실패해도 괜찮은 시스템"
date: 2026-03-17T21:05:00+09:00
draft: false
tags: ["DLQ", "Outbox 패턴", "백프레셔", "멱등성", "비동기", "서버"]
series: ["비동기 처리와 이벤트 드리븐"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 5
summary: "Dead Letter Queue 설계와 재처리 전략, 백프레셔(Backpressure) 개념과 구현, Transactional Outbox 패턴으로 DB와 메시지 원자성 보장, 멱등 소비자 구현과 중복 메시지 처리까지"
---

비동기 시스템의 가장 큰 매력은 "실패해도 괜찮다"는 가능성이다. 동기 호출은 실패 순간 사용자에게 에러가 직격하지만, 비동기 시스템은 실패한 작업을 나중에 다시 시도할 수 있다.

그런데 이 가능성이 현실이 되려면 치밀한 설계가 필요하다. 아무 준비 없이 메시지를 큐에 던지기만 하면, 소비자가 죽거나 처리 로직에 버그가 있을 때 메시지는 그냥 사라진다. 혹은 반대로, 계속 재시도하다가 시스템 전체가 마비된다.

이번 글에서는 실패한 메시지를 안전하게 포착하는 Dead Letter Queue, 소비자가 처리 가능한 속도로 트래픽을 조절하는 백프레셔, DB 커밋과 메시지 발행을 원자적으로 묶는 Transactional Outbox 패턴, 그리고 같은 메시지가 두 번 와도 결과가 달라지지 않게 만드는 멱등 소비자 구현을 다룬다.

이 네 가지를 이해하면 비동기 시스템을 진정한 의미에서 신뢰할 수 있게 된다.

---

## 1. Dead Letter Queue: 실패한 메시지의 격리와 재처리

### DLQ가 필요한 이유

메시지 처리가 실패할 수 있는 이유는 다양하다. 역직렬화 오류, 비즈니스 로직 예외, 외부 시스템 장애, 타임아웃.

이런 메시지를 그냥 무한정 재시도하면 문제가 생긴다. 다른 메시지들이 이 메시지에 막혀 처리되지 못하고(head-of-line blocking), 장애 상황에서 시스템은 재시도 폭풍에 시달린다.

Dead Letter Queue(DLQ)는 이 문제를 해결하는 가장 검증된 패턴이다. 일정 횟수 이상 실패한 메시지를 별도 큐로 격리함으로써, 메인 큐의 흐름을 유지하면서 실패 원인을 나중에 분석하고 재처리할 수 있다.

### Kafka에서 DLQ 구현

Kafka는 기본적으로 DLQ를 제공하지 않는다. 직접 구현해야 한다. Spring Kafka의 `DefaultErrorHandler`와 `DeadLetterPublishingRecoverer`를 조합하면 깔끔하게 처리할 수 있다.

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> kafkaTemplate) {
        // 최대 3회 재시도 후 DLQ로 보낸다
        // 재시도 간격: 1초, 2초, 4초 (지수 백오프)
        ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(3);
        backOff.setInitialInterval(1_000L);
        backOff.setMultiplier(2.0);
        backOff.setMaxInterval(10_000L);

        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            // 원본 토픽명에 '.DLT' 접미사를 붙인다
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
        );

        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);

        // 재시도 없이 즉시 DLQ로 보낼 예외 유형
        handler.addNotRetryableExceptions(
            DeserializationException.class,
            IllegalArgumentException.class
        );

        return handler;
    }
}
```

DLT 토픽으로 전송될 때 Kafka는 헤더에 실패 정보를 자동으로 담는다.

```java
// DLQ 메시지를 소비해 분석하는 컨슈머
@KafkaListener(topics = "order-events.DLT", groupId = "dlq-analyzer")
public void consumeDLQ(
    ConsumerRecord<String, String> record,
    @Header(KafkaHeaders.DLT_EXCEPTION_CAUSE_FQCN) String exceptionClass,
    @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) String exceptionMessage,
    @Header(KafkaHeaders.DLT_ORIGINAL_TOPIC) String originalTopic,
    @Header(KafkaHeaders.DLT_ORIGINAL_OFFSET) long originalOffset
) {
    log.error(
        "DLQ 메시지 수신 - 원본 토픽: {}, 오프셋: {}, 예외: {}, 메시지: {}",
        originalTopic, originalOffset, exceptionClass, exceptionMessage
    );

    // DB에 저장해 운영팀이 검토할 수 있게 한다
    dlqRepository.save(DlqRecord.of(record, exceptionClass, exceptionMessage));
}
```

### DLQ 재처리 전략

DLQ에 쌓인 메시지를 어떻게 처리할지가 진짜 핵심이다. 세 가지 전략이 있다.

**1) 수동 재처리 (Manual Reprocessing)**

운영팀이 DLQ 메시지를 검토하고, 원인을 수정한 뒤 원본 토픽으로 다시 발행한다. 가장 안전하지만 사람이 개입해야 한다는 단점이 있다.

```java
@Service
public class DlqReprocessingService {

    public void reprocess(String dlqRecordId) {
        DlqRecord record = dlqRepository.findById(dlqRecordId)
            .orElseThrow(() -> new NotFoundException("DLQ 레코드를 찾을 수 없습니다: " + dlqRecordId));

        // 상태를 REPROCESSING으로 변경 (중복 재처리 방지)
        record.markAsReprocessing();
        dlqRepository.save(record);

        try {
            kafkaTemplate.send(record.getOriginalTopic(), record.getKey(), record.getValue());
            record.markAsReprocessed();
        } catch (Exception e) {
            record.markAsReprocessingFailed(e.getMessage());
            throw e;
        } finally {
            dlqRepository.save(record);
        }
    }
}
```

**2) 자동 재처리 (Scheduled Reprocessing)**

일정 시간이 지난 후 자동으로 재시도한다. 외부 시스템 일시 장애처럼 시간이 지나면 해결될 문제에 적합하다.

```java
@Scheduled(fixedDelay = 300_000) // 5분마다
public void autoReprocess() {
    List<DlqRecord> candidates = dlqRepository.findReprocessCandidates(
        LocalDateTime.now().minusHours(1), // 1시간 이상 경과
        DlqStatus.PENDING,
        5 // 최대 재처리 횟수
    );

    for (DlqRecord record : candidates) {
        if (record.getReprocessCount() >= 5) {
            record.markAsAbandoned();
            alertService.notify("재처리 한계 초과: " + record.getId());
            dlqRepository.save(record);
            continue;
        }

        reprocessingService.reprocess(record.getId());
    }
}
```

**3) 회로 차단기와 연동**

외부 시스템 장애 시 DLQ 재처리를 중단하고, 장애 복구 후 재개한다.

```java
@Component
public class CircuitBreakerAwareDlqProcessor {

    private final CircuitBreaker paymentCircuitBreaker;

    public void process(DlqRecord record) {
        if (paymentCircuitBreaker.getState() == CircuitBreaker.State.OPEN) {
            log.info("결제 서비스 서킷 브레이커 OPEN 상태. DLQ 재처리 건너뜀.");
            return;
        }

        reprocessingService.reprocess(record.getId());
    }
}
```

### DLQ 안티패턴

**DLQ를 영구 보관소로 쓰는 것**: DLQ는 임시 격리 공간이다. 메시지가 영원히 쌓이면 근본 문제를 해결하지 않고 있다는 신호다. DLQ 크기에 알림을 설정하고, 쌓이는 속도가 빨라지면 즉시 조사해야 한다.

**재처리 없이 DLQ를 버리는 것**: DLQ를 만들었지만 아무도 안 본다면 없는 것과 같다. DLQ 모니터링과 재처리 프로세스를 시스템 설계의 일부로 포함시켜야 한다.

---

## 2. 백프레셔: 소비자가 감당할 수 있는 속도로

### 백프레셔가 없으면 무슨 일이 생기나

생산자가 소비자보다 훨씬 빠르게 메시지를 쏟아낼 때, 큐는 계속 늘어난다. 메모리가 고갈되거나, 처리 지연이 수십 분에 달하거나, 결국 소비자가 OOM으로 죽는다. 이것이 백프레셔가 없는 시스템의 전형적인 장애 패턴이다.

백프레셔(Backpressure)는 소비자가 자신의 처리 능력을 생산자에게 전달해 속도를 조절하는 메커니즘이다. 호스로 물을 받는데, 수압이 너무 세면 호스를 반만 열듯이.

### Kafka 컨슈머의 백프레셔 구현

Kafka 컨슈머는 `max.poll.records`와 `fetch.max.bytes`로 폴링 단위를 제어할 수 있다.

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processor");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        // 한 번에 가져올 최대 레코드 수 (기본값: 500)
        // 처리 시간이 긴 작업이라면 낮추는 것이 좋다
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 50);

        // 세션 타임아웃: 이 시간 안에 heartbeat가 없으면 리밸런싱
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30_000);

        // max.poll.interval.ms: poll 호출 간격 최대값
        // 처리 시간이 길다면 이 값을 늘려야 한다
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300_000);

        return new DefaultKafkaConsumerFactory<>(props);
    }
}
```

처리 로직에서 세마포어로 동시 처리 수를 제한하는 방법도 있다.

```java
@Service
public class OrderEventConsumer {

    // 동시에 처리할 수 있는 최대 작업 수
    private final Semaphore semaphore = new Semaphore(10);
    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    @KafkaListener(topics = "order-events", groupId = "order-processor")
    public void consume(List<ConsumerRecord<String, String>> records,
                        Acknowledgment ack) {
        List<CompletableFuture<Void>> futures = new ArrayList<>();

        for (ConsumerRecord<String, String> record : records) {
            try {
                // 슬롯이 없으면 여기서 블로킹 (백프레셔!)
                semaphore.acquire();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }

            CompletableFuture<Void> future = CompletableFuture
                .runAsync(() -> {
                    try {
                        processOrder(record.value());
                    } finally {
                        semaphore.release();
                    }
                }, executor);

            futures.add(future);
        }

        // 모든 처리가 완료된 후 커밋
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        ack.acknowledge();
    }

    private void processOrder(String orderJson) {
        // 시간이 걸리는 주문 처리 로직
    }
}
```

### Project Reactor(WebFlux)에서의 백프레셔

리액티브 스트림은 백프레셔를 프레임워크 수준에서 지원한다. `limitRate()`로 업스트림에서 요청하는 양을 제한할 수 있다.

```java
@Service
public class ReactiveOrderProcessor {

    public Flux<ProcessResult> processOrders(Flux<Order> orderStream) {
        return orderStream
            // 한 번에 최대 20개 요청 (업스트림에 백프레셔 신호 전송)
            .limitRate(20)
            // 병렬 처리하되 동시에 5개로 제한
            .flatMap(
                order -> processOrderAsync(order),
                5 // maxConcurrency
            )
            // 처리 중 에러가 나도 스트림이 죽지 않게
            .onErrorContinue((throwable, order) -> {
                log.error("주문 처리 실패: {}", order, throwable);
                deadLetterService.send((Order) order, throwable);
            });
    }

    private Mono<ProcessResult> processOrderAsync(Order order) {
        return Mono.fromCallable(() -> {
            // 실제 처리 로직
            return processOrder(order);
        })
        .subscribeOn(Schedulers.boundedElastic())
        .timeout(Duration.ofSeconds(30));
    }
}
```

### 적응형 백프레셔 (Adaptive Backpressure)

고정된 concurrency 대신, 시스템 부하에 따라 동적으로 조정하는 방식이다.

```java
@Component
public class AdaptiveRateLimiter {

    private final AtomicInteger currentConcurrency = new AtomicInteger(10);
    private final AtomicLong successCount = new AtomicLong(0);
    private final AtomicLong failureCount = new AtomicLong(0);

    @Scheduled(fixedRate = 10_000) // 10초마다 조정
    public void adjustConcurrency() {
        long success = successCount.getAndSet(0);
        long failure = failureCount.getAndSet(0);
        long total = success + failure;

        if (total == 0) return;

        double errorRate = (double) failure / total;
        int current = currentConcurrency.get();

        if (errorRate > 0.1) {
            // 에러율 10% 초과: 동시성 25% 감소
            int newValue = Math.max(1, (int) (current * 0.75));
            currentConcurrency.set(newValue);
            log.warn("에러율 {}%. 동시성 {} -> {}로 감소", String.format("%.1f", errorRate * 100), current, newValue);
        } else if (errorRate < 0.01) {
            // 에러율 1% 미만: 동시성 10% 증가
            int newValue = Math.min(100, (int) (current * 1.1));
            currentConcurrency.set(newValue);
            log.info("에러율 낮음. 동시성 {} -> {}로 증가", current, newValue);
        }
    }

    public int getConcurrency() {
        return currentConcurrency.get();
    }

    public void recordSuccess() { successCount.incrementAndGet(); }
    public void recordFailure() { failureCount.incrementAndGet(); }
}
```

### 백프레셔 안티패턴

**큐를 무한정 성장시키는 것**: `LinkedBlockingQueue`를 unbounded로 쓰거나, Kafka의 consumer lag를 모니터링하지 않으면 문제가 숨어있다가 한꺼번에 터진다. 항상 큐의 깊이(depth)에 알림을 설정하라.

**단순히 스레드 수만 늘리는 것**: 스레드를 늘리면 처리량이 늘어나지만, 다운스트림 시스템(DB, 외부 API)이 견디지 못하면 그쪽이 터진다. 백프레셔는 시스템 전체를 고려해야 한다.

---

## 3. Transactional Outbox 패턴: DB와 메시지의 원자성

### 이중 쓰기 문제

주문이 생성될 때 두 가지 일이 필요하다. DB에 주문을 저장하고, 메시지 큐에 이벤트를 발행한다. 이 두 작업은 서로 다른 시스템에 속하기 때문에 하나의 트랜잭션으로 묶을 수 없다. 이 두 가지를 원자적으로 처리하지 않으면 심각한 불일치가 생긴다.

```java
// 위험한 코드: 원자성 보장 없음
@Transactional
public void createOrder(OrderRequest request) {
    Order order = orderRepository.save(Order.from(request)); // DB 저장 성공
    kafkaTemplate.send("order-events", order.toEvent()); // 여기서 실패하면?
    // DB에는 주문이 있지만 이벤트는 발행되지 않음!
}
```

반대 순서도 문제다. 이벤트를 먼저 발행하면 DB 저장이 실패할 경우 없는 주문에 대한 이벤트가 퍼진다.

### Outbox 패턴의 핵심 아이디어

해결책은 간단하다. 이벤트를 같은 DB 트랜잭션 안에서 `outbox` 테이블에 저장한다. 그리고 별도 프로세스가 outbox 테이블을 읽어 메시지 큐에 발행한다.

```sql
CREATE TABLE outbox_events (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id   VARCHAR(100) NOT NULL,
    event_type     VARCHAR(100) NOT NULL,
    payload        JSONB       NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at   TIMESTAMPTZ,
    status         VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    retry_count    INT         NOT NULL DEFAULT 0
);

CREATE INDEX idx_outbox_pending ON outbox_events(created_at)
    WHERE status = 'PENDING';
```

```java
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {

    @Id
    @GeneratedValue
    private UUID id;

    private String aggregateType;
    private String aggregateId;
    private String eventType;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> payload;

    @CreationTimestamp
    private LocalDateTime createdAt;

    private LocalDateTime publishedAt;

    @Enumerated(EnumType.STRING)
    private OutboxStatus status = OutboxStatus.PENDING;

    private int retryCount = 0;

    public static OutboxEvent from(Order order) {
        OutboxEvent event = new OutboxEvent();
        event.aggregateType = "Order";
        event.aggregateId = order.getId().toString();
        event.eventType = "OrderCreated";
        event.payload = Map.of(
            "orderId", order.getId(),
            "customerId", order.getCustomerId(),
            "totalAmount", order.getTotalAmount(),
            "items", order.getItems()
        );
        return event;
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxEventRepository outboxEventRepository;

    @Transactional // 하나의 트랜잭션으로 묶임
    public Order createOrder(OrderRequest request) {
        Order order = orderRepository.save(Order.from(request));

        // 이벤트를 같은 트랜잭션에서 outbox 테이블에 저장
        outboxEventRepository.save(OutboxEvent.from(order));

        return order;
        // 여기서 커밋되거나, 둘 다 롤백됨. 완전한 원자성 보장.
    }
}
```

### Outbox Relay: Polling 방식

별도 스케줄러가 outbox를 폴링하고 Kafka에 발행한다.

```java
@Component
@RequiredArgsConstructor
public class OutboxRelay {

    private final OutboxEventRepository outboxEventRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Scheduled(fixedDelay = 1_000) // 1초마다
    @Transactional
    public void relay() {
        // 배타적 잠금으로 중복 처리 방지 (다중 인스턴스 환경)
        List<OutboxEvent> events = outboxEventRepository
            .findPendingEventsWithLock(100);

        for (OutboxEvent event : events) {
            try {
                String topic = resolveTopicName(event.getAggregateType(), event.getEventType());

                kafkaTemplate.send(
                    topic,
                    event.getAggregateId(), // Kafka 파티셔닝 키
                    event.getPayload()
                ).get(5, TimeUnit.SECONDS); // 발행 확인 대기

                event.markAsPublished();
            } catch (Exception e) {
                event.incrementRetryCount();
                if (event.getRetryCount() >= 5) {
                    event.markAsFailed();
                    alertService.notify("Outbox 발행 최대 재시도 초과: " + event.getId());
                }
                log.error("Outbox 이벤트 발행 실패: {}", event.getId(), e);
            }

            outboxEventRepository.save(event);
        }
    }

    private String resolveTopicName(String aggregateType, String eventType) {
        return aggregateType.toLowerCase() + "-" + eventType.toLowerCase() + "s";
    }
}
```

```java
// Repository에서 SELECT ... FOR UPDATE SKIP LOCKED 사용
public interface OutboxEventRepository extends JpaRepository<OutboxEvent, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2")) // SKIP LOCKED
    @Query("""
        SELECT e FROM OutboxEvent e
        WHERE e.status = 'PENDING'
          AND e.retryCount < 5
        ORDER BY e.createdAt ASC
        LIMIT :limit
        """)
    List<OutboxEvent> findPendingEventsWithLock(@Param("limit") int limit);
}
```

### Outbox Relay: CDC(Change Data Capture) 방식

폴링 방식은 DB 부하와 지연이 있다. Debezium 같은 CDC 도구를 쓰면 PostgreSQL WAL 로그를 실시간으로 읽어 Kafka에 발행할 수 있다.

```yaml
# Debezium PostgreSQL 커넥터 설정
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "localhost",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "orders",
    "table.include.list": "public.outbox_events",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.key": "aggregate_id",
    "transforms.outbox.route.by.field": "aggregate_type",
    "transforms.outbox.route.topic.replacement": "outbox.event.${routedByValue}"
  }
}
```

CDC 방식은 DB에 폴링 부하 없이 실시간에 가깝게 이벤트를 발행할 수 있다. 단, Debezium 인프라를 운영해야 하는 복잡성이 있다.

### Outbox 안티패턴

**outbox 테이블을 지우지 않는 것**: 발행 완료된 이벤트를 계속 쌓아두면 테이블이 무한정 커진다. 발행 후 일정 기간(예: 7일) 경과한 레코드는 배치로 정리해야 한다.

**outbox 지연을 무시하는 것**: 폴링 간격이 1초라면 최악의 경우 1초 지연이 생긴다. 이벤트 소비자가 이 지연을 허용할 수 있는지 확인해야 한다. 실시간성이 중요하다면 CDC를 고려하라.

---

## 4. 멱등 소비자: 중복 메시지를 안전하게 처리하기

### 중복 메시지는 왜 발생하나

비동기 메시징에서 "정확히 한 번(Exactly-once)" 전달은 이론적으로 가능하지만 실제로는 매우 어렵다. 현실에서는 "최소 한 번(At-least-once)"이 기본값이다.

소비자가 메시지를 처리하고 커밋하기 직전에 죽으면, 재시작 후 같은 메시지를 다시 처리한다. Kafka의 경우 리밸런싱 과정에서도 중복이 생길 수 있다. Outbox 릴레이도 네트워크 오류로 발행 확인을 못 받으면 같은 이벤트를 재발행한다.

멱등(Idempotent) 소비자는 같은 메시지를 여러 번 받아도 결과가 한 번 처리한 것과 동일하다. 중복을 막는 게 아니라, 중복이 와도 안전한 것이다.

### 중복 감지: processed_messages 테이블

가장 신뢰할 수 있는 방법이다. 처리한 메시지 ID를 DB에 기록하고, 다음 메시지가 오면 먼저 확인한다.

```sql
CREATE TABLE processed_messages (
    message_id   VARCHAR(255) PRIMARY KEY,
    consumer_group VARCHAR(100) NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    topic        VARCHAR(255),
    partition_id INT,
    offset_value BIGINT
);

-- 오래된 레코드 자동 삭제를 위한 인덱스
CREATE INDEX idx_processed_at ON processed_messages(processed_at);
```

```java
@Service
@RequiredArgsConstructor
public class IdempotentOrderConsumer {

    private final ProcessedMessageRepository processedMessageRepository;
    private final OrderProcessingService orderProcessingService;

    @KafkaListener(topics = "order-events", groupId = "order-processor")
    @Transactional
    public void consume(ConsumerRecord<String, OrderEvent> record) {
        String messageId = extractMessageId(record);

        // 이미 처리한 메시지인지 확인
        if (processedMessageRepository.existsByMessageIdAndConsumerGroup(
                messageId, "order-processor")) {
            log.info("중복 메시지 무시: messageId={}", messageId);
            return;
        }

        // 비즈니스 로직 실행
        orderProcessingService.process(record.value());

        // 처리 완료 기록 (같은 트랜잭션에서)
        processedMessageRepository.save(ProcessedMessage.builder()
            .messageId(messageId)
            .consumerGroup("order-processor")
            .topic(record.topic())
            .partitionId(record.partition())
            .offsetValue(record.offset())
            .build());
    }

    private String extractMessageId(ConsumerRecord<String, OrderEvent> record) {
        // 메시지 헤더에서 고유 ID 추출 (없으면 토픽+파티션+오프셋으로 생성)
        Header idHeader = record.headers().lastHeader("X-Message-Id");
        if (idHeader != null) {
            return new String(idHeader.value());
        }
        return record.topic() + "-" + record.partition() + "-" + record.offset();
    }
}
```

### 비즈니스 로직 레벨의 멱등성

DB 조회를 최소화하려면 비즈니스 로직 자체를 멱등적으로 설계할 수 있다.

**Upsert 패턴**: INSERT 대신 INSERT ... ON CONFLICT UPDATE를 사용한다.

```java
@Repository
public class OrderRepository {

    @Modifying
    @Query(value = """
        INSERT INTO orders (id, customer_id, status, total_amount, created_at)
        VALUES (:#{#order.id}, :#{#order.customerId}, :#{#order.status},
                :#{#order.totalAmount}, :#{#order.createdAt})
        ON CONFLICT (id) DO NOTHING
        """, nativeQuery = true)
    void upsert(@Param("order") Order order);
}
```

**상태 기계 기반 멱등성**: 이미 완료된 상태의 주문에 동일 이벤트가 오면 무시한다.

```java
@Service
public class OrderStateMachine {

    @Transactional
    public void handlePaymentConfirmed(String orderId) {
        Order order = orderRepository.findByIdWithLock(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        // 이미 PAID 상태라면 멱등적으로 성공 처리
        if (order.getStatus() == OrderStatus.PAID) {
            log.info("주문 {} 이미 결제 완료 상태. 중복 이벤트 무시.", orderId);
            return;
        }

        // PENDING 상태에서만 전이 가능
        if (order.getStatus() != OrderStatus.PENDING) {
            throw new InvalidStateTransitionException(
                "결제 확인 이벤트를 처리할 수 없는 상태: " + order.getStatus()
            );
        }

        order.setStatus(OrderStatus.PAID);
        order.setPaidAt(LocalDateTime.now());
        orderRepository.save(order);
    }
}
```

### 낙관적 잠금으로 경쟁 조건 방지

동시에 같은 메시지를 처리하는 복수의 인스턴스가 있을 때, 낙관적 잠금이 두 번째 처리를 막는다.

```java
@Entity
public class Order {

    @Id
    private UUID id;

    @Version // 낙관적 잠금 버전 필드
    private Long version;

    private OrderStatus status;

    // ...
}

@Service
public class OrderService {

    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100)
    )
    @Transactional
    public void processPayment(String orderId) {
        Order order = orderRepository.findById(UUID.fromString(orderId))
            .orElseThrow();

        if (order.getStatus() == OrderStatus.PAID) {
            return; // 멱등: 이미 처리됨
        }

        order.setStatus(OrderStatus.PAID);
        orderRepository.save(order); // 동시 수정 시 OptimisticLockException 발생
    }
}
```

### Redis를 활용한 고성능 중복 감지

DB 기록 방식은 안전하지만 DB 부하가 있다. 처리량이 많을 때는 Redis의 `SET NX`(Not eXists)를 쓰면 더 빠르다.

```java
@Component
@RequiredArgsConstructor
public class RedisIdempotencyChecker {

    private final RedisTemplate<String, String> redisTemplate;
    private static final Duration TTL = Duration.ofDays(7);

    /**
     * 메시지를 처음 처리하면 true, 중복이면 false를 반환한다.
     */
    public boolean tryProcess(String messageId, String consumerGroup) {
        String key = "idempotency:" + consumerGroup + ":" + messageId;

        // SET key value NX EX seconds: 키가 없을 때만 설정
        Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent(key, "processed", TTL);

        return Boolean.TRUE.equals(isNew);
    }
}

@Service
@RequiredArgsConstructor
public class HighThroughputOrderConsumer {

    private final RedisIdempotencyChecker idempotencyChecker;
    private final OrderProcessingService orderProcessingService;

    @KafkaListener(topics = "order-events", groupId = "order-processor",
                   concurrency = "5")
    public void consume(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
        String messageId = buildMessageId(record);

        if (!idempotencyChecker.tryProcess(messageId, "order-processor")) {
            log.debug("중복 메시지 스킵: {}", messageId);
            ack.acknowledge();
            return;
        }

        try {
            orderProcessingService.process(record.value());
            ack.acknowledge();
        } catch (Exception e) {
            // 처리 실패 시 Redis 키 삭제해 재처리 가능하게
            redisTemplate.delete("idempotency:order-processor:" + messageId);
            log.error("주문 처리 실패. 재처리를 위해 중복 체크 해제: {}", messageId, e);
            // ack하지 않으므로 Kafka가 재전달함
        }
    }

    private String buildMessageId(ConsumerRecord<String, OrderEvent> record) {
        // 이벤트 자체의 비즈니스 ID를 우선 사용
        String eventId = record.value().getEventId();
        if (eventId != null) return eventId;

        return record.topic() + ":" + record.partition() + ":" + record.offset();
    }
}
```

### 멱등성 안티패턴

**타임스탬프로만 중복을 감지하는 것**: 같은 시간에 같은 이벤트가 두 번 올 수 있다. 항상 고유 ID 기반으로 중복을 감지해야 한다.

**멱등성 체크와 비즈니스 로직을 다른 트랜잭션으로 분리하는 것**: 체크는 성공했지만 비즈니스 로직 실패 후 재시도할 때, 멱등성 체크가 이미 처리됨으로 막으면 안 된다. 체크와 처리는 같은 트랜잭션에 있거나, 실패 시 롤백 가능해야 한다.

---

## 5. 패턴들의 조합: 완전한 신뢰성 레이어

네 가지 패턴은 독립적이지 않다. 함께 사용할 때 진정한 신뢰성이 만들어진다.

```
[생산자]
  주문 서비스 → (같은 트랜잭션) → [orders 테이블 + outbox_events 테이블]
                                          ↓
                                   [Outbox Relay]
                                          ↓ (at-least-once 발행)
                                   [Kafka 토픽]
                                          ↓
[소비자]                           백프레셔 제어 (limitRate/semaphore)
  멱등성 체크 (Redis/DB)                  ↓
        ↓ 중복이면 스킵           [소비자 처리 로직]
        ↓ 신규면 처리                     ↓ 실패 시
                                   [재시도 + DLQ]
```

Outbox 패턴이 "메시지가 반드시 발행된다"를 보장하고, 멱등 소비자가 "중복 발행이 와도 안전하다"를 보장한다. 백프레셔가 "소비자가 감당 가능한 속도로 처리한다"를 보장하고, DLQ가 "실패한 메시지를 잃지 않는다"를 보장한다.

이 조합 위에서 비로소 "실패해도 괜찮은 시스템"이 현실이 된다.

---

## 참고 자료

- [Kafka Documentation — Error Handling](https://docs.spring.io/spring-kafka/docs/current/reference/html/#error-handling) — Spring Kafka의 DefaultErrorHandler와 DLQ 공식 문서
- [Microservices Patterns — Chris Richardson](https://microservices.io/patterns/data/transactional-outbox.html) — Transactional Outbox 패턴 원문
- [Debezium Documentation](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html) — CDC 기반 Outbox Event Router 설정
- [Reactive Streams Specification](https://www.reactive-streams.org/) — 백프레셔를 포함한 리액티브 스트림 표준 명세
- [Building Event-Driven Microservices — Adam Bellemare](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/) — 이벤트 드리븐 시스템에서의 신뢰성 패턴 심층 분석
- [Idempotent Consumer Pattern — Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/IdempotentReceiver.html) — Gregor Hohpe의 멱등 수신자 패턴 원문
