---
title: "[비동기 처리와 이벤트 드리븐] 2편 — Kafka 심화: 대용량 이벤트 스트리밍"
date: 2026-03-17T21:06:00+09:00
draft: false
tags: ["Kafka", "이벤트 스트리밍", "파티션", "Consumer Group", "서버"]
series: ["비동기 처리와 이벤트 드리븐"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 5
summary: "Kafka 아키텍처(Broker, Topic, Partition, Consumer Group), 오프셋 관리와 At-Least-Once/Exactly-Once 처리, 파티션 설계 전략과 컨슈머 리밸런싱, Kafka vs RabbitMQ 선택 기준과 스키마 레지스트리까지"
---

하루에 수천억 건의 이벤트를 처리한다고 상상해 보자.

LinkedIn은 2011년 내부 메시지 큐 시스템의 한계에 부딪혔다. 데이터베이스 폴링, 커스텀 파이프라인, 파편화된 로그 수집 — 모든 것이 뒤엉켜 있었다. 그들이 만든 해답이 Kafka다.

오늘날 Netflix, Uber, Airbnb, 그리고 수만 개의 회사가 Kafka 위에서 실시간 데이터 파이프라인을 돌린다.

단순한 메시지 큐가 아니다. Kafka는 **분산 커밋 로그**라는 근본적으로 다른 추상화를 제공하며, 이 차이가 대용량 이벤트 스트리밍의 판도를 바꿨다.

이번 편에서는 Kafka의 내부 동작 원리부터 실전 운영 전략까지, 표면 아래를 깊게 파고든다.

---

## Kafka 아키텍처: 분산 커밋 로그의 해부

### Broker, Cluster, ZooKeeper/KRaft

Kafka 클러스터는 여러 **Broker**로 구성된다. 각 Broker는 독립적인 서버로, 프로듀서로부터 메시지를 받아 디스크에 저장하고 컨슈머에게 전달한다. 브로커는 상태를 공유하지 않는다 — 이것이 수평 확장을 가능하게 하는 핵심이다.

전통적으로 Kafka는 클러스터 메타데이터 관리를 **ZooKeeper**에 위임했다. 브로커 등록, 리더 선출, 설정 동기화 모두 ZooKeeper가 담당했다.

그러나 ZooKeeper는 별도 운영 복잡성을 유발했고, Kafka 3.0부터는 **KRaft(Kafka Raft Metadata)** 모드가 GA로 전환되었다. KRaft는 Kafka 내부에 Raft 합의 알고리즘을 구현해 ZooKeeper 의존성을 완전히 제거한다.

```yaml
# KRaft 모드 server.properties 핵심 설정
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@broker1:9093,2@broker2:9093,3@broker3:9093
listeners=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
inter.broker.listener.name=PLAINTEXT
controller.listener.names=CONTROLLER
log.dirs=/var/kafka/data
```

### Topic과 Partition: 병렬성의 단위

**Topic**은 논리적 채널이다. "주문 이벤트", "사용자 활동 로그"처럼 비즈니스 도메인 단위로 구분한다. 그리고 각 Topic은 하나 이상의 **Partition**으로 분할된다.

Partition이 Kafka의 병렬성과 확장성의 근간이다. 각 Partition은:
- 순서가 보장된 불변의 레코드 시퀀스 (커밋 로그)
- 단일 브로커에 **리더**가 존재하고, 다른 브로커들이 **팔로워**로 복제
- 독립적으로 읽고 쓸 수 있는 단위

```
Topic: order-events (파티션 4개, 복제 인수 3)

Broker 1: Partition 0 [Leader], Partition 2 [Follower]
Broker 2: Partition 1 [Leader], Partition 0 [Follower]
Broker 3: Partition 2 [Leader], Partition 1 [Follower], Partition 3 [Follower]
Broker 4: Partition 3 [Leader]
```

프로듀서가 메시지를 보낼 때 파티션을 선택하는 기준은 세 가지다:
1. **키 기반**: 동일 키는 동일 파티션 → 순서 보장 (`hash(key) % numPartitions`)
2. **라운드 로빈**: 키가 없을 때 기본 전략 (Kafka 2.4+는 sticky partitioning 사용)
3. **커스텀 Partitioner**: 비즈니스 로직으로 직접 결정

```java
// 커스텀 Partitioner 구현
public class OrderTypePartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        if (key == null) {
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }

        String orderType = key.toString();
        // VIP 주문은 파티션 0번으로 고정 → 우선순위 처리
        if ("VIP".equals(orderType)) {
            return 0;
        }
        return Math.abs(key.hashCode()) % (numPartitions - 1) + 1;
    }

    @Override
    public void configure(Map<String, ?> configs) {}

    @Override
    public void close() {}
}
```

### Consumer Group: 경쟁 소비와 팬아웃

**Consumer Group**은 Kafka의 소비 모델에서 가장 우아한 개념이다. 같은 그룹 내 컨슈머들은 파티션을 나눠 가져 **경쟁 소비(competing consumer)** 패턴을 구현하고, 다른 그룹은 동일 토픽을 독립적으로 소비하는 **팬아웃(fan-out)** 패턴을 구현한다.

```
Topic: order-events (파티션 4개)

Consumer Group A (주문 처리 서비스):
  Consumer A-1 → Partition 0, Partition 1
  Consumer A-2 → Partition 2, Partition 3

Consumer Group B (분석 서비스):
  Consumer B-1 → Partition 0
  Consumer B-2 → Partition 1
  Consumer B-3 → Partition 2
  Consumer B-4 → Partition 3
```

핵심 제약: **파티션 하나는 동일 그룹 내 컨슈머 하나에만 할당된다**. 따라서 컨슈머 수가 파티션 수를 초과하면 일부 컨슈머는 유휴 상태가 된다. 최대 병렬 처리 수준은 파티션 수에 의해 결정된다.

---

## 오프셋 관리: 내가 어디까지 읽었는가

### 오프셋의 본질

Partition 내 각 레코드는 고유한 **오프셋(offset)**을 가진다. 0부터 시작하는 단조 증가 정수다. 컨슈머는 자신이 마지막으로 처리한 오프셋을 커밋함으로써 "여기까지 읽었다"는 상태를 저장한다.

오프셋은 `__consumer_offsets`라는 내부 토픽에 저장된다 (Kafka 0.9 이전에는 ZooKeeper에 저장). 이로 인해 컨슈머가 재시작되거나 리밸런싱이 발생해도 중단점부터 재개할 수 있다.

```java
// Spring Kafka 컨슈머 기본 설정
@Configuration
@EnableKafka
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, OrderEvent> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka1:9092,kafka2:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processing-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        // 오프셋 자동 커밋 비활성화 (수동 제어)
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        // 그룹에 오프셋이 없을 때 처음부터 읽기
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        // 수동 오프셋 커밋 모드
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        factory.setConcurrency(4); // 파티션 수와 동일하게 설정
        return factory;
    }
}
```

### At-Least-Once 처리

가장 일반적인 처리 보장 수준이다. 메시지는 **최소 한 번 처리된다** — 실패 시 재처리되므로 중복 가능성이 있다. 오프셋 커밋을 처리 완료 후에 하는 것이 핵심이다.

```java
@Service
public class OrderConsumer {

    private final OrderService orderService;

    @KafkaListener(topics = "order-events", groupId = "order-processing-group")
    public void consume(ConsumerRecord<String, OrderEvent> record,
                        Acknowledgment acknowledgment) {
        try {
            // 1. 메시지 처리
            orderService.processOrder(record.value());

            // 2. 처리 완료 후 오프셋 커밋 → At-Least-Once
            acknowledgment.acknowledge();

            log.info("Processed order: {}, offset: {}",
                record.value().getOrderId(), record.offset());

        } catch (Exception e) {
            // 커밋 안 함 → 재처리됨 (중복 발생 가능)
            log.error("Failed to process order at offset {}: {}",
                record.offset(), e.getMessage());
            // 적절한 에러 처리 (DLQ 전송 등)
        }
    }
}
```

**안티패턴**: 오프셋을 처리 전에 커밋하면 At-Most-Once가 된다. 처리 실패 시 메시지가 유실된다.

```java
// 안티패턴: 선커밋 후처리
acknowledgment.acknowledge(); // 먼저 커밋
orderService.processOrder(record.value()); // 여기서 실패하면 메시지 유실!
```

### Exactly-Once 처리: 트랜잭션 프로듀서

Kafka 0.11부터 **트랜잭션 API**가 도입되어 Exactly-Once Semantics(EOS)가 가능해졌다. 프로듀서는 여러 파티션에 원자적으로 쓸 수 있고, 컨슈머-프로듀서 파이프라인에서 "읽기-처리-쓰기"를 하나의 트랜잭션으로 묶을 수 있다.

```java
// 트랜잭션 프로듀서 설정
@Bean
public ProducerFactory<String, OrderEvent> producerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka1:9092,kafka2:9092");
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
    // Exactly-Once 설정
    props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-processor-tx-1");
    props.put(ProducerConfig.ACKS_CONFIG, "all");
    props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
    return new DefaultKafkaProducerFactory<>(props);
}

// 트랜잭션을 이용한 Exactly-Once 처리
@Service
public class ExactlyOnceOrderProcessor {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void processWithTransaction(ConsumerRecord<String, OrderEvent> record,
                                       Consumer<String, OrderEvent> consumer) {
        kafkaTemplate.executeInTransaction(operations -> {
            // 1. 처리 결과를 다른 토픽에 쓰기
            OrderProcessedEvent processed = processOrder(record.value());
            operations.send("order-processed", record.key(), processed);

            // 2. 오프셋을 트랜잭션에 포함하여 커밋
            Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
            offsets.put(
                new TopicPartition(record.topic(), record.partition()),
                new OffsetAndMetadata(record.offset() + 1)
            );
            operations.sendOffsetsToTransaction(offsets, consumer.groupMetadata());

            return true;
        });
    }
}
```

EOS는 성능 트레이드오프가 있다. 트랜잭션 코디네이터와의 통신, 쓰기 지연 증가(약 20-30ms 추가), 컨슈머 측 `isolation.level=read_committed` 필요 등이 따라온다.

**모든 경우에 EOS가 필요한 것은 아니다**. 금융 결제, 재고 차감처럼 중복이 절대 허용되지 않는 경우에만 도입을 고려하라.

---

## 파티션 설계 전략

### 파티션 수는 얼마로 설정해야 하는가

파티션 수는 나중에 늘릴 수 있지만 줄일 수 없다. 초기 설계가 중요하다.

**고려 요소:**
- **목표 처리량**: 단일 파티션의 처리량(P)과 목표 처리량(T)으로 `T/P` 개 이상 필요
- **컨슈머 병렬성**: 최대 병렬 컨슈머 수 = 파티션 수
- **브로커 수**: 파티션 수는 브로커 수의 배수로 설정하면 균등 분배

```
실용적 공식:
파티션 수 = max(목표_TPS / 단일파티션_TPS, 최대_컨슈머_수)

예시:
- 목표: 100,000 msg/sec
- 단일 파티션 처리량 (디스크 I/O 기준): ~10,000 msg/sec
- 최대 컨슈머: 20개

파티션 수 = max(100,000 / 10,000, 20) = max(10, 20) = 20
```

**안티패턴**: 파티션을 과도하게 많이 만드는 것. 각 파티션은 브로커에서 파일 핸들과 메모리를 사용한다. 파티션이 너무 많으면 리더 선출 시간이 길어지고 GC 압력이 증가한다. LinkedIn의 벤치마크에서는 브로커당 4,000개 파티션이 한계치였다.

### 파티션 키 설계: 순서와 분산의 균형

파티션 키 선택은 **순서 보장의 범위**를 결정한다. 같은 키를 가진 메시지는 항상 같은 파티션으로 가므로 순서가 보장된다.

```java
// 주문 서비스 - 파티션 키 전략
@Service
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderCreated(Order order) {
        OrderEvent event = OrderEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .status(OrderStatus.CREATED)
            .timestamp(Instant.now())
            .build();

        // userId를 키로 사용 → 같은 유저의 주문은 순서 보장
        // 단, 특정 유저에 이벤트가 몰리면 핫 파티션 문제 발생
        kafkaTemplate.send("order-events", order.getUserId().toString(), event);
    }

    // 핫 파티션 방지를 위한 복합 키 전략
    public void publishWithSalt(Order order) {
        // 소금(salt) 추가로 분산, 단 순서 보장 범위는 좁아짐
        String key = order.getUserId() + "-" + (order.getOrderId() % 10);
        kafkaTemplate.send("order-events", key, event);
    }
}
```

**핫 파티션 탐지**: 특정 파티션의 lag이 다른 파티션보다 현저히 높다면 핫 파티션을 의심하라.

```bash
# 파티션별 오프셋 및 lag 확인
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --group order-processing-group \
  --describe

# 출력 예시
TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
order-events    0          15234           15234           0
order-events    1          89234           92000           2766   ← 핫 파티션
order-events    2          14500           14500           0
order-events    3          13800           13800           0
```

---

## 컨슈머 리밸런싱: 그룹의 재조정

### 리밸런싱이 발생하는 시점

**리밸런싱(Rebalancing)**은 Consumer Group 내 파티션 할당을 재조정하는 과정이다. 다음 상황에서 트리거된다:
- 컨슈머가 그룹에 참여하거나 이탈할 때
- 컨슈머가 Heartbeat 타임아웃(`session.timeout.ms`)을 초과할 때
- `max.poll.interval.ms`를 초과해 poll()을 호출하지 않을 때
- 구독 토픽의 파티션 수가 변경될 때

리밸런싱 중에는 **모든 컨슈머가 일시 정지**된다 (Stop-The-World). 이것이 핵심 성능 문제다.

### Cooperative Sticky Assignor로 STW 최소화

Kafka 2.4+에서 **Incremental Cooperative Rebalancing**이 도입되었다. 기존 Eager 방식은 모든 파티션을 반납하고 재할당하지만, Cooperative 방식은 변경이 필요한 파티션만 이동한다.

```java
// Cooperative Sticky Assignor 설정
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    CooperativeStickyAssignor.class.getName());
```

기존 Eager 리밸런싱:
```
1단계: 모든 컨슈머가 파티션 반납 (처리 중단)
2단계: 새로운 할당 계산
3단계: 파티션 재할당 (처리 재개)

STW 시간 = 전체 리밸런싱 완료 시간 (수 초 ~ 수십 초)
```

Cooperative 리밸런싱:
```
1단계: 이동이 필요한 파티션만 반납
      (유지되는 파티션은 계속 처리)
2단계: 이동된 파티션만 재할당

STW 시간 = 거의 0 (변경 파티션만 잠깐 중단)
```

### 리밸런싱 최적화 설정

```java
// 리밸런싱 관련 핵심 설정값
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 45000);    // 45초 (기본 10초보다 크게)
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 15000); // session.timeout의 1/3
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 처리 시간 고려 (5분)
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);        // 한번에 처리할 레코드 수
```

**안티패턴**: `max.poll.interval.ms`를 무한정 늘리는 것. 컨슈머가 실제로 죽었을 때 그룹 재조정이 지연된다. 대신 `max.poll.records`를 줄여 처리 시간 자체를 단축하라.

```java
// 올바른 접근: 배치 크기 조정으로 처리 시간 단축
@KafkaListener(topics = "order-events",
               groupId = "order-processing-group",
               containerFactory = "batchKafkaListenerContainerFactory")
public void consumeBatch(List<ConsumerRecord<String, OrderEvent>> records,
                         Acknowledgment acknowledgment) {
    // 배치 처리로 처리량 향상
    List<OrderEvent> events = records.stream()
        .map(ConsumerRecord::getValue)
        .collect(Collectors.toList());

    orderService.processBatch(events); // 벌크 처리
    acknowledgment.acknowledge();

    log.info("Processed batch of {} records", records.size());
}
```

### Static Group Membership으로 불필요한 리밸런싱 방지

배포나 재시작 시 발생하는 불필요한 리밸런싱을 막으려면 **Static Membership**을 사용하라.

```java
// 각 컨슈머 인스턴스에 고정 ID 부여
props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG,
    "order-consumer-" + podName); // 쿠버네티스 Pod 이름 활용

// session.timeout.ms 이내에 재시작하면 리밸런싱 없이 복귀
```

---

## Kafka vs RabbitMQ: 무엇을 선택할 것인가

두 시스템은 근본적으로 다른 철학을 가진다. "어느 것이 더 좋은가"가 아닌 "어느 것이 내 문제에 맞는가"를 물어야 한다.

### 핵심 차이점

| 기준 | Kafka | RabbitMQ |
|------|-------|----------|
| **처리 모델** | Pull (컨슈머가 가져감) | Push (브로커가 밀어줌) |
| **메시지 보존** | 설정된 기간 보존 (기본 7일) | 소비 후 삭제 |
| **순서 보장** | 파티션 내 보장 | 큐 내 보장 (단일 컨슈머) |
| **최대 처리량** | 수백만 msg/sec | 수만 ~ 수십만 msg/sec |
| **재처리** | 오프셋 리셋으로 가능 | 기본 불가능 (DLQ 필요) |
| **라우팅** | 토픽 기반 (단순) | Exchange/Routing Key (복잡) |
| **지연 시간** | 수 ms ~ 수십 ms | 1ms 이하 가능 |
| **운영 복잡성** | 높음 | 낮음 |

### Kafka를 선택해야 할 때

- **대용량 이벤트 스트리밍**: 초당 수만 건 이상의 이벤트 처리
- **이벤트 소싱/CQRS**: 이벤트 히스토리 재처리가 필요한 경우
- **다수의 독립적 컨슈머**: 동일 이벤트를 여러 서비스가 다른 방식으로 소비
- **실시간 분석 파이프라인**: Kafka Streams, ksqlDB와 연계
- **로그 집계**: 높은 처리량과 내구성이 핵심

### RabbitMQ를 선택해야 할 때

- **복잡한 라우팅 로직**: Direct, Topic, Fanout, Headers Exchange 조합
- **낮은 지연 시간이 최우선**: 금융 거래, 실시간 게임 등
- **작업 큐 패턴**: 단일 소비 후 삭제가 자연스러운 워크플로우
- **소규모 시스템**: 운영 복잡성을 줄이고 싶을 때
- **RPC 패턴**: 요청-응답 패턴이 필요한 경우

```
실무 판단 기준:
"이벤트를 나중에 다시 재처리해야 하나?" → Yes → Kafka
"초당 10만 건 이상 처리해야 하나?" → Yes → Kafka
"복잡한 라우팅이 필요한가?" → Yes → RabbitMQ
"지연이 1ms 미만이어야 하나?" → Yes → RabbitMQ 또는 Redis Streams
```

---

## 스키마 레지스트리: 데이터 계약 관리

### 왜 스키마가 필요한가

Kafka는 바이트 배열을 저장한다. 직렬화 형식에 대한 합의가 없으면 프로듀서가 스키마를 변경할 때 컨슈머가 깨진다. 예를 들어 `amount` 필드를 `int`에서 `double`로 바꾸면, 이전 컨슈머가 해당 메시지를 역직렬화하는 순간 예외가 발생한다. **스키마 레지스트리**는 이 계약을 중앙에서 관리한다.

Confluent Schema Registry가 사실상 표준이며, **Apache Avro**, Protobuf, JSON Schema를 지원한다.

### Avro + Schema Registry 실전 구현

```avro
// order-event.avsc
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.example.orders",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "userId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "status", "type": {
      "type": "enum",
      "name": "OrderStatus",
      "symbols": ["CREATED", "PAID", "SHIPPED", "DELIVERED", "CANCELLED"]
    }},
    {"name": "timestamp", "type": "long", "logicalType": "timestamp-millis"},
    {"name": "metadata", "type": ["null", {"type": "map", "values": "string"}],
     "default": null}
  ]
}
```

```java
// Schema Registry를 사용하는 프로듀서 설정
@Bean
public ProducerFactory<String, OrderEvent> avroProducerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
        KafkaAvroSerializer.class);
    // Schema Registry 주소
    props.put("schema.registry.url", "http://schema-registry:8081");
    // 자동으로 스키마 등록 허용 여부 (프로덕션에서는 false 권장)
    props.put("auto.register.schemas", false);
    props.put("use.latest.version", true);
    return new DefaultKafkaProducerFactory<>(props);
}

// 컨슈머 설정
@Bean
public ConsumerFactory<String, OrderEvent> avroConsumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
        KafkaAvroDeserializer.class);
    props.put("schema.registry.url", "http://schema-registry:8081");
    props.put("specific.avro.reader", true); // 생성된 클래스 사용
    return new DefaultKafkaConsumerFactory<>(props);
}
```

### 스키마 진화와 호환성 전략

스키마는 시간이 지나면 변한다. Schema Registry는 호환성 규칙으로 이 진화를 안전하게 관리한다.

```bash
# 토픽별 호환성 수준 설정
curl -X PUT \
  http://schema-registry:8081/config/order-events-value \
  -H "Content-Type: application/json" \
  -d '{"compatibility": "BACKWARD"}'
```

| 호환성 수준 | 의미 | 사용 시나리오 |
|------------|------|--------------|
| **BACKWARD** | 새 스키마로 이전 데이터 읽기 가능 | 컨슈머 먼저 배포 |
| **FORWARD** | 이전 스키마로 새 데이터 읽기 가능 | 프로듀서 먼저 배포 |
| **FULL** | 양방향 호환 | 롤링 배포, 가장 안전 |
| **NONE** | 호환성 검사 안 함 | 개발 환경에서만 |

```avro
// BACKWARD 호환 스키마 변경 예시
// 새 필드에 기본값을 부여하면 이전 데이터도 읽기 가능
{
  "type": "record",
  "name": "OrderEvent",
  "fields": [
    // 기존 필드...
    // 새 필드 추가: 반드시 default 값 필요
    {"name": "couponCode", "type": ["null", "string"], "default": null},
    {"name": "deliveryAddress", "type": ["null", "string"], "default": null}
  ]
}
```

**안티패턴**: 기존 필드의 타입을 변경하거나 삭제하는 것. 이는 모든 호환성 수준을 위반하며 컨슈머를 깨뜨린다.

---

## 실전 운영: 모니터링과 튜닝

### 핵심 메트릭

```yaml
# Prometheus + Grafana로 수집할 핵심 Kafka 메트릭

프로듀서:
  - kafka_producer_record_send_rate: 초당 전송 레코드 수
  - kafka_producer_request_latency_avg: 평균 요청 지연
  - kafka_producer_record_error_rate: 에러율

컨슈머:
  - kafka_consumer_records_consumed_rate: 초당 소비 레코드 수
  - kafka_consumer_fetch_latency_avg: 평균 Fetch 지연
  - records_lag_max: 컨슈머 그룹 최대 Lag ← 가장 중요

브로커:
  - kafka_server_bytes_in_per_sec: 수신 처리량
  - kafka_server_bytes_out_per_sec: 송신 처리량
  - kafka_controller_active_controller_count: 활성 컨트롤러 수 (항상 1이어야 함)
  - kafka_log_log_flush_rate: 디스크 플러시 빈도
```

Consumer Lag은 가장 중요한 건강 지표다. Lag이 지속적으로 증가한다면 컨슈머가 프로듀서 속도를 따라가지 못하는 것이다.

해결책은 다음 순서로 시도한다:

1. 컨슈머 처리 로직 최적화 (DB 쿼리 개선, 배치 처리 도입)
2. 컨슈머 인스턴스 수 증가 (파티션 수 범위 내에서)
3. 파티션 수 증가 후 컨슈머 추가

### 프로듀서 튜닝

```java
// 처리량 최대화를 위한 프로듀서 배치 설정
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);          // 64KB 배치
props.put(ProducerConfig.LINGER_MS_CONFIG, 10);              // 10ms 대기 후 배치 전송
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy"); // 압축으로 처리량 향상
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864);    // 64MB 버퍼
props.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 60000);        // 버퍼 가득 찼을 때 최대 대기

// 내구성 vs 성능 트레이드오프
props.put(ProducerConfig.ACKS_CONFIG, "all");         // 가장 안전, 가장 느림
// props.put(ProducerConfig.ACKS_CONFIG, "1");        // 리더만 응답, 중간
// props.put(ProducerConfig.ACKS_CONFIG, "0");        // 응답 안 받음, 가장 빠름 (유실 가능)
```

---

## 정리

Kafka를 단순한 메시지 큐로 다루면 그 진가를 반도 못 끌어낸다.

Broker-Partition-Consumer Group의 구조를 이해하고, 오프셋 관리로 처리 보장 수준을 제어하며, 파티션 키 설계로 순서와 분산을 균형 있게 맞추는 것 — 이것이 Kafka를 제대로 다루는 방법이다.

RabbitMQ와의 선택은 처리량, 재처리 필요성, 라우팅 복잡성이라는 세 축에서 결정된다. 그리고 스키마 레지스트리는 분산 시스템에서 데이터 계약을 명시적으로 관리하는 가장 효과적인 방법이다.

리밸런싱 최소화를 위해 Cooperative Sticky Assignor와 Static Membership을 적용하고, Consumer Lag을 주요 건강 지표로 모니터링하라. 대용량 이벤트 스트리밍은 기술보다 설계가 먼저다 — Kafka는 그 설계를 실현하는 강력한 도구다.

---

## 참고 자료

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/) — 공식 문서, 설정값 레퍼런스 필수
- [Confluent Schema Registry Documentation](https://docs.confluent.io/platform/current/schema-registry/index.html) — Avro/Protobuf 스키마 관리 상세 가이드
- [Designing Data-Intensive Applications](https://dataintensive.net/) — Martin Kleppmann, 분산 시스템 이론의 교과서
- [Kafka: The Definitive Guide, 2nd Edition](https://www.confluent.io/resources/kafka-the-definitive-guide-v2/) — Confluent 공식 가이드북
- [KIP-429: Kafka Consumer Incremental Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-429%3A+Kafka+Consumer+Incremental+Rebalance+Protocol) — Cooperative 리밸런싱 설계 문서
- [LinkedIn Engineering Blog: Kafka at Scale](https://engineering.linkedin.com/kafka/kafka-linkedin-current-and-future) — Kafka 탄생 배경과 대규모 운영 사례
