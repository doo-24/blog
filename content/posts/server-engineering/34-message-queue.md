---
title: "[비동기 처리와 이벤트 드리븐] 1편 — 메시지 큐 기초: 요청을 기다리지 않는 서버"
date: 2026-03-17T21:07:00+09:00
draft: false
tags: ["메시지 큐", "RabbitMQ", "Pub/Sub", "비동기", "서버"]
series: ["비동기 처리와 이벤트 드리븐"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 5
summary: "동기 호출의 한계와 메시지 큐의 해결, Point-to-Point vs Pub/Sub 모델, RabbitMQ 구조(Exchange, Queue, Binding, Routing Key), 메시지 지속성과 소비자 확인(ACK), 메시지 순서 보장까지"
---

전자상거래 서비스에서 주문이 들어오면 어떤 일이 벌어질까?

재고 차감, 결제 승인, 이메일 발송, SMS 알림, 포인트 적립, 배송 시스템 연동... 이 모든 작업을 하나의 HTTP 요청 안에서 순서대로 처리한다면, 사용자는 버튼을 누른 뒤 몇 초를 기다려야 한다.

그리고 SMS 게이트웨이가 일시적으로 느려지는 순간, 전체 주문 처리가 멈춰버린다. 동기적 설계의 함정이다.

메시지 큐는 이 문제를 근본에서 해결한다. 호출자가 응답을 기다리지 않고 메시지를 던져두면, 수신자가 자신의 속도로 처리한다.

서비스들은 서로를 직접 알 필요가 없고, 하나가 느려져도 다른 하나가 멈추지 않는다.

이 편에서는 메시지 큐가 왜 필요한지, 어떻게 동작하는지, 그리고 RabbitMQ를 중심으로 실제 코드 수준의 깊이까지 파고든다.

---

## 동기 호출의 한계

### 강결합(Tight Coupling)이 만드는 취약성

동기 HTTP 호출로 시스템을 연결하면 발신자와 수신자는 강하게 결합된다. 수신자가 응답하지 않으면 발신자는 타임아웃까지 블로킹되고, 수신자가 배포 중이거나 장애 상태라면 발신자의 기능도 함께 중단된다.

```java
// 안티패턴: 동기 호출로 연결된 주문 처리
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
    // 1. 주문 생성
    Order order = orderService.create(request);

    // 2. 재고 서비스 동기 호출 — 재고 서비스가 느리면 여기서 멈춤
    inventoryClient.deduct(order.getItems());

    // 3. 결제 서비스 동기 호출 — 결제 서비스 장애 = 주문 전체 실패
    paymentClient.charge(order.getPaymentInfo());

    // 4. 알림 서비스 동기 호출 — SMS 게이트웨이 지연 = 사용자 대기
    notificationClient.sendOrderConfirmation(order);

    // 5. 포인트 서비스 동기 호출
    pointClient.accrue(order.getUserId(), order.getAmount());

    return ResponseEntity.ok(OrderResponse.from(order));
}
```

이 코드의 전체 응답 시간은 각 서비스 응답 시간의 합이다. 재고(50ms) + 결제(200ms) + 알림(150ms) + 포인트(30ms) = 430ms.

그리고 어느 하나라도 실패하면 트랜잭션 롤백을 어떻게 처리할 것인가? 분산 트랜잭션의 악몽이 시작된다.

### 확장성의 벽

동기 호출 구조에서 트래픽이 몰리면 어떻게 될까?

주문 서비스에 초당 1,000건의 요청이 들어오면, 알림 서비스도 초당 1,000건의 요청을 처리할 수 있어야 한다. 모든 하위 서비스가 동일한 피크 부하를 감당할 수 있도록 스케일 아웃해야 한다.

이 중 알림 서비스는 평소에 초당 10건만 처리하면 충분한데도 피크 시간을 위해 100배의 자원을 상시 보유해야 한다.

### 메시지 큐가 해결하는 세 가지 문제

**1. 시간적 결합 제거(Temporal Decoupling)**: 발신자와 수신자가 동시에 온라인 상태일 필요가 없다. 수신자가 배포 중이어도 메시지는 큐에 쌓이고, 재시작 후 처리된다.

**2. 속도 차이 완충(Rate Buffering)**: 발신자가 초당 1,000건을 생산해도 수신자는 초당 100건씩 안정적으로 처리할 수 있다. 큐가 버퍼 역할을 한다.

**3. 부하 분산(Load Leveling)**: 피크 트래픽이 즉각적으로 하위 서비스에 전달되지 않는다. 큐에 쌓아두고 소비자가 처리 가능한 속도로 꺼내간다.

---

## Point-to-Point vs Pub/Sub 모델

메시지 큐의 두 가지 핵심 패턴을 제대로 이해해야 적재적소에 사용할 수 있다.

### Point-to-Point (큐 모델)

```
Producer → [Queue] → Consumer A
                  → (Consumer B는 받지 못함)
```

메시지는 정확히 하나의 소비자에게만 전달된다. 여러 소비자가 같은 큐를 구독해도 각 메시지는 그 중 하나만 처리한다. 이 특성이 핵심이다.

**사용 사례**: 작업 분배(Task Distribution). 이미지 리사이징, 이메일 발송, 결제 처리처럼 각 작업이 정확히 한 번만 실행되어야 할 때 사용한다.

```
주문 서비스 → [order-processing-queue] → Worker 1 (주문 #1001 처리)
                                       → Worker 2 (주문 #1002 처리)
                                       → Worker 3 (주문 #1003 처리)
```

여러 Worker가 큐를 공유하면 자동으로 부하가 분산된다. Worker를 늘리면 처리 속도가 올라간다. 이것이 컴피티티브 컨슈머(Competing Consumers) 패턴이다.

### Pub/Sub (토픽 모델)

```
Publisher → [Topic/Exchange] → Queue A → Consumer Group A (이메일 서비스)
                             → Queue B → Consumer Group B (SMS 서비스)
                             → Queue C → Consumer Group C (푸시 알림 서비스)
```

하나의 메시지가 여러 소비자 그룹에 모두 전달된다. 발행자는 누가 구독하는지 알 필요가 없다. 새로운 구독자가 추가되어도 발행자 코드는 변경되지 않는다.

**사용 사례**: 이벤트 브로드캐스팅. "주문 완료" 이벤트가 발생했을 때, 이메일 서비스, SMS 서비스, 분석 서비스, 재고 서비스가 각자 독립적으로 반응해야 할 때.

### 두 모델의 선택 기준

| 기준 | Point-to-Point | Pub/Sub |
|------|---------------|---------|
| 메시지 처리 | 정확히 1회 | 구독자 수만큼 |
| 소비자 추가 | 처리 능력 향상 | 독립적 반응자 추가 |
| 발행자가 소비자를 알아야 하는가 | 큐 이름만 알면 됨 | 전혀 불필요 |
| 주요 목적 | 작업 분배 | 이벤트 전파 |

---

## RabbitMQ 구조 깊이 이해하기

RabbitMQ는 AMQP(Advanced Message Queuing Protocol)를 구현한 메시지 브로커다. 구조가 Kafka보다 복잡하지만, 그 복잡성이 강력한 라우팅 유연성을 제공한다.

### 핵심 개념: Exchange, Queue, Binding

```
Producer → Exchange → (Binding + Routing Rules) → Queue → Consumer
```

많은 개발자들이 "메시지를 큐에 직접 보낸다"고 오해한다. RabbitMQ에서 Producer는 메시지를 **Exchange**에 보낸다. Exchange는 라우팅 규칙에 따라 하나 이상의 Queue에 메시지를 전달한다.

### Exchange 유형

#### 1. Direct Exchange

Routing Key가 Queue의 Binding Key와 정확히 일치할 때 전달한다.

```java
// Spring AMQP 설정
@Configuration
public class RabbitMQConfig {

    public static final String DIRECT_EXCHANGE = "order.direct";
    public static final String PAYMENT_QUEUE = "order.payment";
    public static final String INVENTORY_QUEUE = "order.inventory";

    @Bean
    public DirectExchange orderDirectExchange() {
        return new DirectExchange(DIRECT_EXCHANGE, true, false);
        // durable=true: 브로커 재시작 후에도 Exchange 유지
        // autoDelete=false: 소비자가 없어도 Exchange 삭제 안 함
    }

    @Bean
    public Queue paymentQueue() {
        return QueueBuilder.durable(PAYMENT_QUEUE).build();
    }

    @Bean
    public Queue inventoryQueue() {
        return QueueBuilder.durable(INVENTORY_QUEUE).build();
    }

    @Bean
    public Binding paymentBinding() {
        // "payment" routing key로 들어온 메시지 → paymentQueue
        return BindingBuilder.bind(paymentQueue())
                .to(orderDirectExchange())
                .with("payment");
    }

    @Bean
    public Binding inventoryBinding() {
        // "inventory" routing key로 들어온 메시지 → inventoryQueue
        return BindingBuilder.bind(inventoryQueue())
                .to(orderDirectExchange())
                .with("inventory");
    }
}

// 메시지 발행
@Service
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishPaymentRequest(PaymentRequest request) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.DIRECT_EXCHANGE,
            "payment",           // routing key
            request
        );
    }

    public void publishInventoryDeduction(InventoryRequest request) {
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.DIRECT_EXCHANGE,
            "inventory",         // routing key
            request
        );
    }
}
```

#### 2. Fanout Exchange

Routing Key를 무시하고 바인딩된 모든 Queue에 메시지를 전달한다. 순수한 Pub/Sub.

```java
@Configuration
public class FanoutExchangeConfig {

    public static final String ORDER_FANOUT_EXCHANGE = "order.completed.fanout";

    @Bean
    public FanoutExchange orderCompletedFanoutExchange() {
        return new FanoutExchange(ORDER_FANOUT_EXCHANGE, true, false);
    }

    @Bean
    public Queue emailNotificationQueue() {
        return QueueBuilder.durable("order.notification.email").build();
    }

    @Bean
    public Queue smsNotificationQueue() {
        return QueueBuilder.durable("order.notification.sms").build();
    }

    @Bean
    public Queue analyticsQueue() {
        return QueueBuilder.durable("order.analytics").build();
    }

    // Fanout은 routing key가 의미 없으므로 with("") 또는 with(AmqpAdmin.DEFAULT_DEFAULT_BINDING_ROUTING_KEY)
    @Bean
    public Binding emailBinding() {
        return BindingBuilder.bind(emailNotificationQueue())
                .to(orderCompletedFanoutExchange());
    }

    @Bean
    public Binding smsBinding() {
        return BindingBuilder.bind(smsNotificationQueue())
                .to(orderCompletedFanoutExchange());
    }

    @Bean
    public Binding analyticsBinding() {
        return BindingBuilder.bind(analyticsQueue())
                .to(orderCompletedFanoutExchange());
    }
}

// "주문 완료" 이벤트 하나로 이메일, SMS, 분석 세 큐에 모두 전달
@Service
public class OrderCompletionService {

    public void completeOrder(Order order) {
        orderRepository.save(order.complete());

        OrderCompletedEvent event = OrderCompletedEvent.from(order);
        // routing key는 무시됨 — 모든 바인딩된 큐에 전달
        rabbitTemplate.convertAndSend(
            FanoutExchangeConfig.ORDER_FANOUT_EXCHANGE,
            "",
            event
        );
    }
}
```

#### 3. Topic Exchange

Routing Key의 패턴 매칭으로 전달할 큐를 선택한다. `*`는 단어 하나, `#`는 0개 이상의 단어를 의미한다.

```java
@Configuration
public class TopicExchangeConfig {

    public static final String LOG_TOPIC_EXCHANGE = "application.logs";

    @Bean
    public TopicExchange logTopicExchange() {
        return new TopicExchange(LOG_TOPIC_EXCHANGE, true, false);
    }

    @Bean
    public Queue criticalErrorQueue() {
        return QueueBuilder.durable("logs.critical").build();
    }

    @Bean
    public Queue orderServiceAllLogsQueue() {
        return QueueBuilder.durable("logs.order-service.all").build();
    }

    @Bean
    public Queue allErrorsQueue() {
        return QueueBuilder.durable("logs.all-errors").build();
    }

    @Bean
    public Binding criticalErrorBinding() {
        // "*.*.critical" — 예: "order-service.payment.critical"
        return BindingBuilder.bind(criticalErrorQueue())
                .to(logTopicExchange())
                .with("*.*.critical");
    }

    @Bean
    public Binding orderServiceBinding() {
        // "order-service.#" — order-service로 시작하는 모든 키
        return BindingBuilder.bind(orderServiceAllLogsQueue())
                .to(logTopicExchange())
                .with("order-service.#");
    }

    @Bean
    public Binding allErrorsBinding() {
        // "#.error" — error로 끝나는 모든 키
        return BindingBuilder.bind(allErrorsQueue())
                .to(logTopicExchange())
                .with("#.error");
    }
}

// 로그 발행
// routing key: "order-service.payment.error"
// → orderServiceAllLogsQueue(order-service.#), allErrorsQueue(#.error) 두 곳에 전달
// routing key: "order-service.checkout.critical"
// → criticalErrorQueue(*.*.critical), orderServiceAllLogsQueue(order-service.#) 두 곳에 전달
```

#### 4. Headers Exchange

Routing Key 대신 메시지 헤더의 속성으로 라우팅한다. 복잡한 라우팅 조건에 유용하지만 성능이 가장 낮아 자주 쓰이지는 않는다.

### Dead Letter Exchange (DLX)

처리에 실패한 메시지를 버리지 않고 별도 큐로 보내는 패턴이다. 장애 분석과 재처리에 필수적이다.

```java
@Configuration
public class DeadLetterConfig {

    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange("order.dlx", true, false);
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable("order.dlq").build();
    }

    @Bean
    public Binding deadLetterBinding() {
        return BindingBuilder.bind(deadLetterQueue())
                .to(deadLetterExchange())
                .with("order.dead");
    }

    @Bean
    public Queue orderMainQueue() {
        return QueueBuilder.durable("order.main")
                // DLX 설정: 이 큐에서 거절된 메시지는 order.dlx로 전달
                .withArgument("x-dead-letter-exchange", "order.dlx")
                .withArgument("x-dead-letter-routing-key", "order.dead")
                // TTL: 30분 내 처리 안 되면 DLX로
                .withArgument("x-message-ttl", 1800000)
                .build();
    }
}
```

---

## 메시지 지속성(Durability)

메시지 큐의 가장 중요한 속성 중 하나는 브로커가 재시작되어도 메시지가 유실되지 않는 것이다.

### 지속성의 세 계층

메시지가 유실되지 않으려면 세 곳 모두 지속성이 설정되어야 한다.

**1. Exchange Durability**: `durable=true`로 선언하면 브로커 재시작 후에도 Exchange가 재생성된다.

**2. Queue Durability**: `durable=true`로 선언하면 브로커 재시작 후에도 큐가 재생성된다.

**3. Message Persistence**: 개별 메시지에 `delivery_mode=2`를 설정해야 디스크에 기록된다.

```java
@Service
public class PersistentMessagePublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishPersistent(String exchange, String routingKey, Object payload) {
        rabbitTemplate.convertAndSend(exchange, routingKey, payload, message -> {
            // delivery mode 2 = persistent (디스크에 저장)
            message.getMessageProperties().setDeliveryMode(
                MessageDeliveryMode.PERSISTENT
            );
            return message;
        });
    }
}

// RabbitTemplate 기본 설정으로 영속성 지정
@Bean
public MessageConverter jsonMessageConverter() {
    return new Jackson2JsonMessageConverter();
}

@Bean
public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    RabbitTemplate template = new RabbitTemplate(connectionFactory);
    template.setMessageConverter(jsonMessageConverter());
    // 기본적으로 모든 메시지를 persistent로 발행
    template.setMandatory(true);
    return template;
}
```

### 안티패턴: 잘못된 지속성 설정

```java
// 안티패턴 1: Exchange는 durable이지만 Queue가 durable이 아님
// 브로커 재시작 시 Queue가 사라지고 메시지 유실
Queue transientQueue = new Queue("important.queue", false); // durable=false

// 안티패턴 2: Queue는 durable이지만 메시지 persistence가 없음
// 메시지가 메모리에만 있어서 브로커 재시작 시 유실
rabbitTemplate.convertAndSend(exchange, routingKey, message);
// MessageDeliveryMode.PERSISTENT를 설정하지 않으면 기본값은 NON_PERSISTENT

// 안티패턴 3: Publisher Confirm 없이 "발송 완료"로 간주
// 실제로 브로커에 도달했는지 확인 없이 성공 처리
```

### Publisher Confirms

메시지가 브로커에 안전하게 도달했는지 확인하는 메커니즘이다.

```java
@Configuration
public class RabbitMQPublisherConfirmConfig {

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory("localhost");
        factory.setPublisherConfirmType(
            CachingConnectionFactory.ConfirmType.CORRELATED
        );
        factory.setPublisherReturns(true);
        return factory;
    }

    @Bean
    public RabbitTemplate confirmingRabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMandatory(true);

        // ACK 수신 시 콜백
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.info("메시지 브로커 도달 확인: {}", correlationData.getId());
                // DB에서 '전송 완료'로 업데이트
            } else {
                log.error("메시지 전송 실패: {}, 원인: {}", correlationData.getId(), cause);
                // 재전송 로직 또는 알림
            }
        });

        // 라우팅 실패 시 콜백 (어떤 큐에도 전달되지 못한 경우)
        template.setReturnsCallback(returned -> {
            log.error("메시지 반환됨 — Exchange: {}, RoutingKey: {}, 응답코드: {}",
                returned.getExchange(),
                returned.getRoutingKey(),
                returned.getReplyCode()
            );
        });

        return template;
    }
}

// Outbox 패턴과 Publisher Confirm 결합
@Service
@Transactional
public class ReliableOrderService {

    public void createOrder(OrderRequest request) {
        Order order = orderRepository.save(Order.from(request));

        // Outbox 테이블에 메시지 저장 (DB 트랜잭션 내)
        OutboxMessage outbox = outboxRepository.save(
            OutboxMessage.builder()
                .aggregateId(order.getId())
                .eventType("ORDER_CREATED")
                .payload(serialize(OrderCreatedEvent.from(order)))
                .status(OutboxStatus.PENDING)
                .build()
        );

        // 트랜잭션 커밋 후 비동기 발행
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronizationAdapter() {
                @Override
                public void afterCommit() {
                    publishWithConfirm(outbox);
                }
            }
        );
    }
}
```

---

## 소비자 확인(ACK)과 메시지 안전 처리

메시지를 받은 소비자가 처리에 성공했는지 실패했는지를 브로커에 알려주는 메커니즘이 ACK(Acknowledge)다.

### ACK 모드의 종류

**Auto ACK**: 메시지를 소비자에게 전달하자마자 큐에서 삭제한다. 처리 중 소비자가 죽으면 메시지는 유실된다.

**Manual ACK**: 소비자가 명시적으로 `basicAck`를 호출해야 큐에서 삭제된다. 처리 실패 시 `basicNack` 또는 `basicReject`로 재처리를 요청할 수 있다.

```java
@Component
public class OrderPaymentConsumer {

    @RabbitListener(
        queues = RabbitMQConfig.PAYMENT_QUEUE,
        ackMode = "MANUAL"  // 반드시 Manual ACK 사용
    )
    public void processPayment(
        PaymentRequest request,
        Channel channel,
        @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag
    ) {
        try {
            paymentService.process(request);

            // 처리 성공 — 큐에서 메시지 삭제
            channel.basicAck(deliveryTag, false);
            // false: 이 메시지 하나만 ACK (true면 이전 미확인 메시지 모두 ACK)

        } catch (PaymentRetryableException e) {
            // 일시적 오류 — 재처리 가능
            log.warn("결제 처리 일시 실패, 재큐잉: {}", e.getMessage());
            channel.basicNack(
                deliveryTag,
                false,   // multiple: 이 메시지만
                true     // requeue: 다시 큐의 앞으로 돌려보냄
            );

        } catch (PaymentPermanentException e) {
            // 영구 오류 — 재처리 불가능 (데이터 오류 등)
            log.error("결제 처리 영구 실패, DLQ로 이동: {}", e.getMessage());
            channel.basicNack(
                deliveryTag,
                false,
                false    // requeue=false: DLX로 전달
            );
        }
    }
}
```

### Prefetch Count 설정

소비자가 한 번에 가져갈 수 있는 미확인(unacked) 메시지의 최대 수를 설정한다. 기본값은 무제한이어서, 처리가 느린 소비자가 메시지를 모두 가져가 다른 소비자가 놀게 되는 문제가 생긴다.

```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
    ConnectionFactory connectionFactory
) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);

    // 소비자당 최대 미확인 메시지 수
    // 처리 시간이 짧으면 높게, 길면 낮게 설정
    factory.setPrefetchCount(10);

    // 동시 소비자 수
    factory.setConcurrentConsumers(3);
    factory.setMaxConcurrentConsumers(10);

    return factory;
}
```

### 안티패턴: Auto ACK의 함정

```java
// 안티패턴: Auto ACK (기본값)
@RabbitListener(queues = "critical.queue")
// ackMode를 지정하지 않으면 AUTO가 기본값
public void processMessage(ImportantPayload payload) {
    // 메시지를 받자마자 ACK 발송 → 큐에서 삭제
    // 이 라인에서 예외가 발생하면? 메시지는 이미 사라짐
    externalPaymentGateway.charge(payload.getAmount()); // 실패 가능
    // 결제도 안 됐는데 메시지는 유실됨
}
```

---

## 메시지 순서 보장

"메시지 큐는 FIFO 순서를 보장한다"는 흔한 오해다. 실제로는 훨씬 복잡하다.

### RabbitMQ의 순서 보장 범위

단일 큐(Single Queue) + 단일 소비자(Single Consumer) 환경에서는 FIFO 순서가 보장된다. 그러나 현실에서는 이 조건이 잘 성립되지 않는다.

**순서가 깨지는 상황**:

1. **여러 소비자**: 소비자 A가 메시지 1을 받고, 소비자 B가 메시지 2를 받는다. 소비자 A의 처리가 더 오래 걸리면 메시지 2가 먼저 완료된다.

2. **nack + requeue**: 메시지 1이 처리 실패로 재큐잉되면, 이미 가져간 메시지 2가 먼저 처리될 수 있다.

3. **Priority Queue**: `x-max-priority`가 설정된 큐는 우선순위에 따라 순서가 바뀐다.

### 순서가 중요한 경우의 전략

**전략 1: 파티셔닝(Consistent Hashing)**

같은 엔티티에 대한 메시지는 항상 같은 큐로 보낸다. 동일 큐 내에서는 순서가 보장된다.

```java
@Service
public class OrderEventRouter {

    private static final int QUEUE_COUNT = 5;
    private static final String[] QUEUES = {
        "order.partition.0",
        "order.partition.1",
        "order.partition.2",
        "order.partition.3",
        "order.partition.4"
    };

    public void publishOrderEvent(OrderEvent event) {
        // 같은 orderId는 항상 같은 큐로 라우팅
        int partition = Math.abs(event.getOrderId().hashCode()) % QUEUE_COUNT;
        String targetQueue = QUEUES[partition];

        rabbitTemplate.convertAndSend(
            "",           // default exchange
            targetQueue,  // routing key = queue name
            event
        );
    }
}
```

**전략 2: 시퀀스 번호 + 멱등성(Idempotency)**

순서를 강제하기보다 순서에 무관하게 처리될 수 있도록 설계한다. 각 메시지에 시퀀스 번호를 붙이고, 소비자가 이미 처리한 시퀀스보다 낮은 메시지는 무시한다.

```java
@Component
public class IdempotentOrderConsumer {

    private final ProcessedEventRepository processedEventRepository;

    @RabbitListener(queues = "order.events")
    public void consume(OrderEvent event, Channel channel,
                        @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        String eventId = event.getEventId();

        // 이미 처리된 이벤트인지 확인 (DB 조회)
        if (processedEventRepository.existsById(eventId)) {
            log.info("중복 이벤트 무시: {}", eventId);
            channel.basicAck(deliveryTag, false);
            return;
        }

        try {
            orderEventProcessor.process(event);

            // 처리 완료 기록 (멱등성 보장)
            processedEventRepository.save(
                ProcessedEvent.builder()
                    .eventId(eventId)
                    .processedAt(Instant.now())
                    .build()
            );

            channel.basicAck(deliveryTag, false);

        } catch (Exception e) {
            channel.basicNack(deliveryTag, false, true);
        }
    }
}
```

**전략 3: 순서 보장이 필요하면 Kafka를 고려하라**

RabbitMQ는 유연한 라우팅과 복잡한 토폴로지에 강점이 있다. 엄격한 순서 보장과 대용량 스트리밍이 핵심 요건이라면 Kafka의 파티션 내 순서 보장이 더 적합하다. 두 시스템의 차이를 이해하고 선택해야 한다.

| 특성 | RabbitMQ | Kafka |
|------|----------|-------|
| 순서 보장 | 단일 큐 + 단일 소비자 | 파티션 내 보장 |
| 메시지 보관 | 소비 후 삭제 | 기간 기반 보관 |
| 라우팅 | Exchange 기반 유연한 라우팅 | 토픽 + 파티션 |
| 재처리 | DLQ + 재발행 | 오프셋 리셋 |
| 주요 용도 | 태스크 큐, 복잡한 라우팅 | 이벤트 스트리밍, 로그 집계 |

---

## 실전 적용: 주문 처리 시스템 재설계

지금까지 배운 내용을 통합해서 처음에 보여준 안티패턴을 개선해보자.

```java
// 개선된 주문 처리: 핵심 작업만 동기, 나머지는 비동기
@PostMapping("/orders")
@Transactional
public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
    // 1. 주문 생성 및 재고 차감 — 비즈니스 일관성이 필요한 작업만 동기
    Order order = orderService.create(request);
    inventoryService.deductWithPessimisticLock(order.getItems());

    // 2. Outbox에 이벤트 저장 (같은 트랜잭션)
    outboxRepository.save(OrderCreatedEvent.from(order));

    // 3. 즉시 응답 반환 — 결제/알림/포인트는 비동기 처리
    return ResponseEntity.ok(OrderResponse.from(order));
}

// Outbox Poller: 주기적으로 미발행 이벤트를 큐에 발행
@Scheduled(fixedDelay = 1000)
public void publishPendingEvents() {
    List<OutboxMessage> pending = outboxRepository.findPending(100);
    for (OutboxMessage msg : pending) {
        try {
            rabbitTemplate.convertAndSend(
                FanoutExchangeConfig.ORDER_FANOUT_EXCHANGE,
                "",
                deserialize(msg.getPayload())
            );
            outboxRepository.markPublished(msg.getId());
        } catch (Exception e) {
            outboxRepository.incrementRetryCount(msg.getId());
        }
    }
}

// 결제 처리 소비자
@Component
public class PaymentConsumer {
    @RabbitListener(queues = "order.notification.payment", ackMode = "MANUAL")
    public void processPayment(OrderCreatedEvent event, Channel channel,
                               @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        try {
            paymentService.asyncCharge(event.getOrderId(), event.getAmount());
            channel.basicAck(tag, false);
        } catch (Exception e) {
            channel.basicNack(tag, false, false); // DLQ로
        }
    }
}
```

이 구조에서 주문 API의 응답 시간은 주문 생성 + 재고 차감(~100ms)만으로 결정된다.

결제, 알림, 포인트 적립은 각자의 속도로 독립적으로 처리된다. SMS 게이트웨이가 느려도 주문 API는 영향받지 않는다.

---

## 마무리

메시지 큐는 "빠르게 만드는 기술"이 아니라 "격리하는 기술"이다.

동기 호출의 강결합을 끊어내고, 서비스들이 서로를 모른 채 협력하게 만든다.

RabbitMQ의 Exchange, Queue, Binding은 단순한 설정이 아니라 메시지 라우팅 전략이다. 지속성과 ACK는 옵션이 아니라 신뢰성의 기반이다. 순서 보장은 자동으로 주어지는 것이 아니라 설계로 확보해야 한다.

다음 편에서는 Apache Kafka를 다룬다. RabbitMQ가 "누가 이 메시지를 처리할 것인가"를 중심으로 설계됐다면, Kafka는 "이 이벤트의 스트림을 어떻게 영속화하고 재처리할 것인가"를 중심으로 설계됐다. 두 시스템의 철학적 차이와 적절한 선택 기준을 깊이 살펴본다.

---

## 참고 자료

1. **RabbitMQ Official Documentation** — AMQP Concepts, Reliability Guide, Consumer Acknowledgements and Publisher Confirms. https://www.rabbitmq.com/documentation.html

2. **Gregor Hohpe, Bobby Woolf — Enterprise Integration Patterns** (2003) — 메시지 큐, 라우팅, 변환 패턴의 교과서적 레퍼런스.

3. **Spring AMQP Reference Documentation** — RabbitTemplate, @RabbitListener, Connection and Channel Management. https://docs.spring.io/spring-amqp/docs/current/reference/html/

4. **Chris Richardson — Microservices Patterns** (2018) — Transactional Outbox, Saga 패턴, 이벤트 기반 아키텍처에서의 메시지 활용.

5. **Martin Fowler — "What do you mean by Event-Driven?"** https://martinfowler.com/articles/201701-event-driven.html — 이벤트 알림, 이벤트-운반 상태 전송, 이벤트 소싱, CQRS의 차이를 명확히 정리.

6. **Clemens Vasters — "Lessons Learned from Building Distributed Systems"** — 메시지 전달 보장(at-most-once, at-least-once, exactly-once)의 실제 구현 트레이드오프.
