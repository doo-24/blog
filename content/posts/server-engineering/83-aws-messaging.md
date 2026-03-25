---
title: "[AWS / 클라우드] 6편 — 메시징과 통합 서비스: SQS, SNS, EventBridge"
date: 2026-03-20T14:03:00+09:00
draft: false
tags: ["SQS", "SNS", "EventBridge", "MSK", "AWS", "서버"]
series: ["AWS / 클라우드"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "SQS 표준 vs FIFO 큐와 DLQ·가시성 타임아웃, SNS 팬아웃 패턴(SNS→SQS), EventBridge 이벤트 버스와 규칙 기반 라우팅·스키마 레지스트리, MSK vs SQS/SNS 선택 기준까지"
---

서비스 간 직접 호출은 강한 결합을 만든다. 메시징 서비스를 사이에 두면 느슨한 결합과 비동기 처리가 가능해진다.

이 글은 SQS, SNS, EventBridge의 동작 원리와 선택 기준을 다룬다.

---

## 목차

1. [메시징 서비스가 필요한 이유](#1-메시징-서비스가-필요한-이유)
2. [SQS — Simple Queue Service](#2-sqs--simple-queue-service)
   - [표준 큐 vs FIFO 큐](#표준-큐-vs-fifo-큐)
   - [가시성 타임아웃](#가시성-타임아웃-visibility-timeout)
   - [Dead Letter Queue (DLQ)](#dead-letter-queue-dlq)
3. [SNS — Simple Notification Service](#3-sns--simple-notification-service)
   - [팬아웃 패턴 (SNS → SQS)](#팬아웃-패턴-sns--sqs)
4. [EventBridge — 이벤트 기반 통합](#4-eventbridge--이벤트-기반-통합)
   - [이벤트 버스](#이벤트-버스)
   - [규칙 기반 라우팅](#규칙-기반-라우팅)
   - [스키마 레지스트리](#스키마-레지스트리)
5. [MSK — Managed Streaming for Apache Kafka](#5-msk--managed-streaming-for-apache-kafka)
6. [MSK vs SQS/SNS 선택 기준](#6-msk-vs-sqssns-선택-기준)
7. [핵심 포인트 정리](#7-핵심-포인트-정리)
8. [참고 자료](#참고-자료)

---

## 1. 메시징 서비스가 필요한 이유

마이크로서비스 환경에서 서비스 간 직접 HTTP 호출은 강한 결합(tight coupling)을 만든다. 하나의 서비스가 느려지거나 다운되면 호출자도 함께 영향을 받는다.

메시징 미들웨어를 사이에 두면 생산자(Producer)와 소비자(Consumer)가 서로를 몰라도 된다. 생산자는 메시지를 큐 또는 토픽에 던지고 바로 다음 작업으로 넘어갈 수 있다.

AWS는 이 역할을 담당하는 서비스를 세 가지 축으로 제공한다.

| 서비스 | 모델 | 대표 용도 |
|---|---|---|
| SQS | 큐(Point-to-Point) | 비동기 작업 처리 |
| SNS | 발행/구독(Pub/Sub) | 다중 구독자 알림 |
| EventBridge | 이벤트 버스 | 서비스 간 이벤트 라우팅 |
| MSK | 스트리밍 | 고처리량 실시간 스트림 |

---

## 2. SQS — Simple Queue Service

SQS는 AWS에서 가장 오래된 서비스 중 하나다. 완전 관리형 메시지 큐로, 서버를 직접 운영할 필요가 없다.

메시지는 큐에 들어간 뒤 소비자가 폴링(poll)해서 가져간다. 소비자가 처리를 완료하면 메시지를 명시적으로 삭제해야 한다.

### 표준 큐 vs FIFO 큐

**표준 큐(Standard Queue)**는 초당 거의 무제한의 처리량을 제공한다. 단, 메시지 순서가 보장되지 않고 동일한 메시지가 두 번 이상 전달될 수 있다.

**FIFO 큐**는 이름 그대로 선입선출 순서를 보장한다. 중복 메시지도 제거되지만, 처리량이 초당 최대 3,000 TPS(배치 사용 시)로 제한된다.

```bash
# 표준 큐 생성
aws sqs create-queue \
  --queue-name my-standard-queue

# FIFO 큐 생성 (이름이 반드시 .fifo로 끝나야 함)
aws sqs create-queue \
  --queue-name my-order-queue.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true
```

FIFO 큐에 메시지를 보낼 때는 `MessageGroupId`를 지정해야 한다. 같은 그룹 ID 내에서만 순서가 보장된다.

```bash
aws sqs send-message \
  --queue-url https://sqs.ap-northeast-2.amazonaws.com/123456789012/my-order-queue.fifo \
  --message-body '{"orderId": "O-1001", "status": "CREATED"}' \
  --message-group-id "order-group-1" \
  --message-deduplication-id "O-1001-created"
```

### 가시성 타임아웃 (Visibility Timeout)

소비자가 메시지를 꺼내가는 순간, 해당 메시지는 다른 소비자에게 보이지 않는 상태가 된다. 이 숨김 시간이 **가시성 타임아웃**이다.

소비자가 처리를 완료하지 못하고 타임아웃이 지나면, 메시지는 다시 큐에 나타나 다른 소비자가 가져갈 수 있다. 처리 시간이 긴 작업은 이 값을 여유 있게 설정해야 한다.

```bash
# 가시성 타임아웃을 300초(5분)로 설정
aws sqs set-queue-attributes \
  --queue-url https://sqs.ap-northeast-2.amazonaws.com/123456789012/my-standard-queue \
  --attributes VisibilityTimeout=300
```

처리 중에 타임아웃 연장이 필요하다면 `ChangeMessageVisibility` API를 호출할 수 있다.

```bash
aws sqs change-message-visibility \
  --queue-url https://sqs.ap-northeast-2.amazonaws.com/123456789012/my-standard-queue \
  --receipt-handle "AQEBwJnKyrHigUMZj..." \
  --visibility-timeout 60
```

> **핵심 포인트**: 가시성 타임아웃은 메시지 처리 예상 시간보다 충분히 크게 설정한다. 너무 짧으면 동일 메시지가 여러 소비자에게 중복 처리될 수 있다.

### Dead Letter Queue (DLQ)

메시지 처리가 반복적으로 실패하면 해당 메시지는 큐에 계속 남아 다른 메시지 처리를 방해할 수 있다. **DLQ(Dead Letter Queue)**는 이런 "독소 메시지"를 격리하는 별도의 큐다.

원본 큐에서 지정한 횟수(`maxReceiveCount`)만큼 처리에 실패하면, 메시지가 자동으로 DLQ로 이동한다.

```bash
# 1단계: DLQ 먼저 생성
aws sqs create-queue --queue-name my-standard-queue-dlq

# DLQ의 ARN 확인
aws sqs get-queue-attributes \
  --queue-url https://sqs.ap-northeast-2.amazonaws.com/123456789012/my-standard-queue-dlq \
  --attribute-names QueueArn
```

```bash
# 2단계: 원본 큐에 Redrive Policy 설정
aws sqs set-queue-attributes \
  --queue-url https://sqs.ap-northeast-2.amazonaws.com/123456789012/my-standard-queue \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:ap-northeast-2:123456789012:my-standard-queue-dlq\",\"maxReceiveCount\":\"3\"}"
  }'
```

`maxReceiveCount`를 3으로 설정하면, 세 번 이상 처리 실패한 메시지가 DLQ로 넘어간다. DLQ를 모니터링하면 처리 오류를 빠르게 파악할 수 있다.

---

## 3. SNS — Simple Notification Service

SNS는 하나의 메시지를 여러 구독자에게 동시에 전달하는 **발행/구독(Pub/Sub)** 모델이다. 생산자는 토픽(Topic)에 메시지를 발행하고, 구독자들은 각자의 방식으로 메시지를 받는다.

지원하는 구독 방식은 다양하다: SQS, Lambda, HTTP/HTTPS 엔드포인트, 이메일, SMS, 모바일 푸시 등.

```bash
# SNS 토픽 생성
aws sns create-topic --name order-events

# SQS 큐를 토픽에 구독
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-northeast-2:123456789012:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:ap-northeast-2:123456789012:order-processing-queue

# 메시지 발행
aws sns publish \
  --topic-arn arn:aws:sns:ap-northeast-2:123456789012:order-events \
  --message '{"orderId": "O-1001", "event": "ORDER_PLACED"}' \
  --subject "New Order"
```

### 팬아웃 패턴 (SNS → SQS)

**팬아웃(Fan-out)**은 하나의 이벤트를 여러 시스템에 동시에 전달해야 할 때 사용하는 패턴이다.

예를 들어, 주문 생성 이벤트 하나가 발생했을 때 재고 서비스, 배송 서비스, 알림 서비스가 각각 독립적으로 처리해야 하는 상황이다.

```
[Order Service]
      |
      v
[SNS: order-events]
      |
  ----|----
  |   |   |
  v   v   v
[SQS: inventory] [SQS: shipping] [SQS: notification]
```

각 SQS 큐는 독립적으로 소비자를 운영한다. 한 서비스가 느려지거나 다운되어도 다른 서비스의 처리에 영향을 주지 않는다.

```bash
# 팬아웃을 위한 세 개 큐 생성
aws sqs create-queue --queue-name inventory-queue
aws sqs create-queue --queue-name shipping-queue
aws sqs create-queue --queue-name notification-queue

# 각 큐를 SNS 토픽에 구독
for queue in inventory-queue shipping-queue notification-queue; do
  QUEUE_ARN=$(aws sqs get-queue-attributes \
    --queue-url https://sqs.ap-northeast-2.amazonaws.com/123456789012/$queue \
    --attribute-names QueueArn \
    --query 'Attributes.QueueArn' --output text)

  aws sns subscribe \
    --topic-arn arn:aws:sns:ap-northeast-2:123456789012:order-events \
    --protocol sqs \
    --notification-endpoint $QUEUE_ARN
done
```

SNS에서 SQS로 메시지가 전달되려면 SQS 큐의 액세스 정책(Access Policy)에서 SNS의 `SendMessage` 권한을 허용해야 한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "sns.amazonaws.com"},
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:ap-northeast-2:123456789012:inventory-queue",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:sns:ap-northeast-2:123456789012:order-events"
        }
      }
    }
  ]
}
```

SNS는 **메시지 필터링(Message Filtering)**도 지원한다. 구독자별로 수신할 메시지를 속성 기준으로 필터링할 수 있다.

```bash
# 배송 큐는 "ORDER_PLACED" 이벤트만 수신
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:ap-northeast-2:123456789012:order-events:abc123 \
  --attribute-name FilterPolicy \
  --attribute-value '{"eventType": ["ORDER_PLACED"]}'
```

---

## 4. EventBridge — 이벤트 기반 통합

EventBridge는 SQS/SNS보다 한 단계 더 진화한 이벤트 라우팅 서비스다. 단순히 메시지를 전달하는 것을 넘어, **이벤트를 기반으로 복잡한 라우팅 로직**을 구성할 수 있다.

AWS 서비스 이벤트(EC2 상태 변경, S3 업로드 등)부터 서드파티 SaaS 이벤트, 직접 만든 커스텀 이벤트까지 모두 처리할 수 있다.

### 이벤트 버스

EventBridge의 중심 개념은 **이벤트 버스(Event Bus)**다. 이벤트가 흘러가는 통로라고 생각하면 된다.

| 버스 종류 | 설명 |
|---|---|
| default | AWS 서비스 이벤트가 자동으로 전송되는 기본 버스 |
| Custom | 사용자가 직접 만드는 애플리케이션 이벤트 버스 |
| Partner | Salesforce, Zendesk 등 SaaS 파트너 이벤트 버스 |

```bash
# 커스텀 이벤트 버스 생성
aws events create-event-bus --name my-app-event-bus

# 이벤트 전송
aws events put-events \
  --entries '[
    {
      "Source": "com.myapp.orders",
      "DetailType": "OrderPlaced",
      "Detail": "{\"orderId\": \"O-1001\", \"customerId\": \"C-500\", \"amount\": 59000}",
      "EventBusName": "my-app-event-bus"
    }
  ]'
```

### 규칙 기반 라우팅

EventBridge의 핵심은 **규칙(Rule)**이다. 이벤트 패턴을 정의하고, 조건에 맞는 이벤트를 특정 대상(Target)으로 전달한다.

대상으로는 Lambda 함수, SQS 큐, SNS 토픽, Step Functions, Kinesis, 다른 이벤트 버스 등을 지원한다.

```bash
# 규칙 생성: 주문 금액이 50,000원 이상인 이벤트만 처리
aws events put-rule \
  --name "high-value-order-rule" \
  --event-bus-name my-app-event-bus \
  --event-pattern '{
    "source": ["com.myapp.orders"],
    "detail-type": ["OrderPlaced"],
    "detail": {
      "amount": [{"numeric": [">=", 50000]}]
    }
  }' \
  --state ENABLED
```

규칙에 대상(Target)을 연결한다.

```bash
# Lambda 함수를 대상으로 지정
aws events put-targets \
  --rule "high-value-order-rule" \
  --event-bus-name my-app-event-bus \
  --targets '[
    {
      "Id": "vip-order-handler",
      "Arn": "arn:aws:lambda:ap-northeast-2:123456789012:function:VIPOrderProcessor"
    }
  ]'
```

하나의 규칙에 최대 5개의 대상을 지정할 수 있다. 동일한 이벤트를 여러 서비스에서 동시에 처리하는 팬아웃과 유사한 효과를 낸다.

**입력 변환(Input Transformer)**을 사용하면 이벤트 구조를 변환해서 대상에 전달할 수도 있다.

```bash
aws events put-targets \
  --rule "high-value-order-rule" \
  --event-bus-name my-app-event-bus \
  --targets '[
    {
      "Id": "sqs-target",
      "Arn": "arn:aws:sqs:ap-northeast-2:123456789012:vip-order-queue",
      "InputTransformer": {
        "InputPathsMap": {
          "orderId": "$.detail.orderId",
          "amount": "$.detail.amount"
        },
        "InputTemplate": "{\"order\": \"<orderId>\", \"price\": <amount>, \"tier\": \"VIP\"}"
      }
    }
  ]'
```

### 스키마 레지스트리

EventBridge에는 이벤트 구조를 문서화하고 공유하는 **스키마 레지스트리(Schema Registry)**가 내장되어 있다.

이벤트 버스에서 흘러가는 이벤트 구조를 자동으로 감지하고 스키마를 추론한다. 팀 간에 이벤트 계약(contract)을 명확하게 공유할 수 있다.

```bash
# 스키마 레지스트리 생성
aws schemas create-registry --registry-name my-app-schemas

# 특정 이벤트 버스의 스키마 자동 감지 활성화
aws schemas create-discoverer \
  --source-arn arn:aws:events:ap-northeast-2:123456789012:event-bus/my-app-event-bus \
  --description "Auto-discover schemas from my-app-event-bus"
```

스키마를 등록하면 AWS 콘솔에서 코드 바인딩(Code Binding)을 자동 생성할 수 있다. Java, Python, TypeScript용 데이터 클래스를 자동으로 만들어준다.

> **핵심 포인트**: 스키마 레지스트리는 이벤트 기반 시스템에서 서비스 간 인터페이스를 명확히 하는 데 매우 유용하다. 이벤트 구조가 변경될 때 어떤 소비자가 영향을 받는지 파악하는 데 도움된다.

---

## 5. MSK — Managed Streaming for Apache Kafka

**MSK(Amazon Managed Streaming for Apache Kafka)**는 Apache Kafka를 완전 관리형으로 제공하는 서비스다. Kafka 브로커, ZooKeeper, 패치, 확장을 AWS가 대신 관리한다.

Kafka는 SQS/SNS와 달리 **로그 기반 스트리밍** 모델을 사용한다. 메시지를 소비해도 삭제되지 않고 설정된 보존 기간 동안 유지된다. 여러 소비자 그룹이 독립적으로 같은 데이터를 읽을 수 있다.

```bash
# MSK 클러스터 생성 (설정 파일 사용)
aws kafka create-cluster \
  --cluster-name my-kafka-cluster \
  --broker-node-group-info '{
    "InstanceType": "kafka.m5.large",
    "ClientSubnets": [
      "subnet-0a1b2c3d",
      "subnet-1a2b3c4d",
      "subnet-2a3b4c5d"
    ],
    "StorageInfo": {
      "EbsStorageInfo": {"VolumeSize": 100}
    }
  }' \
  --kafka-version "3.5.1" \
  --number-of-broker-nodes 3
```

MSK는 Kafka 클라이언트 라이브러리를 그대로 사용할 수 있다. 기존 Kafka 기반 애플리케이션을 거의 수정 없이 마이그레이션할 수 있다.

```python
# Python에서 MSK 토픽에 메시지 발행 (kafka-python 라이브러리)
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['b-1.my-kafka-cluster.xxx.kafka.ap-northeast-2.amazonaws.com:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

producer.send('order-events', {
    'orderId': 'O-1001',
    'event': 'ORDER_PLACED',
    'timestamp': '2026-03-20T14:00:00+09:00'
})
producer.flush()
```

MSK Serverless 옵션도 있다. 용량을 미리 프로비저닝하지 않고 사용량에 따라 자동 확장된다.

```bash
# MSK Serverless 클러스터 생성
aws kafka create-cluster-v2 \
  --cluster-name my-serverless-cluster \
  --serverless '{
    "VpcConfigs": [
      {
        "SubnetIds": ["subnet-0a1b2c3d", "subnet-1a2b3c4d"],
        "SecurityGroupIds": ["sg-0a1b2c3d"]
      }
    ]
  }'
```

---

## 6. MSK vs SQS/SNS 선택 기준

가장 자주 받는 질문이 "언제 Kafka(MSK)를 쓰고 언제 SQS/SNS를 써야 하나요?"다. 각각 다른 요구사항에 맞게 설계되어 있다.

| 기준 | SQS/SNS | MSK (Kafka) |
|---|---|---|
| 처리량 | 중간 (수천 TPS) | 매우 높음 (수백만 TPS) |
| 메시지 보존 | 최대 14일, 소비 후 삭제 | 설정 기간 동안 유지 |
| 순서 보장 | FIFO 큐만 보장 | 파티션 내 보장 |
| 재처리 | 별도 구현 필요 | 오프셋으로 언제든 재처리 |
| 운영 복잡도 | 낮음 (완전 서버리스) | 중간 (브로커 관리 필요) |
| 학습 곡선 | 낮음 | 높음 (Kafka 개념 필요) |
| 비용 | 메시지 수 기반 | 클러스터 시간 기반 |

**SQS를 선택하는 경우:**

- 단순한 비동기 작업 큐가 필요할 때 (이미지 처리, 이메일 발송 등)
- 팀이 Kafka에 익숙하지 않을 때
- 빠른 도입과 낮은 운영 부담이 우선순위일 때
- 메시지 처리량이 초당 수천 건 이하일 때

**SNS를 선택하는 경우:**

- 동일한 이벤트를 여러 시스템에 동시에 전달해야 할 때
- 이메일, SMS, 모바일 푸시 알림이 필요할 때
- SQS 팬아웃 패턴을 구성해야 할 때

**EventBridge를 선택하는 경우:**

- AWS 서비스 이벤트에 반응하는 자동화가 필요할 때
- 여러 SaaS 서비스의 이벤트를 통합해야 할 때
- 이벤트 패턴 기반의 복잡한 라우팅 로직이 필요할 때
- 서비스 간 이벤트 계약을 스키마로 관리하고 싶을 때

**MSK를 선택하는 경우:**

- 초당 수십만 건 이상의 고처리량 스트리밍이 필요할 때
- 이벤트 소싱(Event Sourcing)이나 CQRS 패턴을 구현할 때
- 과거 데이터를 다시 읽어야 하는 재처리(replay) 요구사항이 있을 때
- 이미 Kafka 기반 시스템이 있어 마이그레이션하는 경우

> **현실적인 조언**: 대부분의 스타트업과 중소 서비스는 SQS + SNS 조합으로 충분하다. MSK는 데이터 파이프라인이나 실시간 분석처럼 명확한 필요가 생겼을 때 도입하는 것이 좋다. 운영 복잡도를 과소평가하지 말자.

---

## 7. 핵심 포인트 정리

> **SQS 표준 vs FIFO**
> - 표준 큐: 높은 처리량, 순서/중복 보장 없음
> - FIFO 큐: 순서와 정확히 1회 처리 보장, 처리량 제한 있음

> **가시성 타임아웃**
> - 소비자가 메시지를 꺼내간 동안 다른 소비자에게 숨겨지는 시간
> - 처리 예상 시간보다 충분히 크게 설정할 것

> **DLQ**
> - 반복 실패 메시지를 격리하는 별도 큐
> - maxReceiveCount 초과 시 자동 이동
> - 반드시 알람을 설정해 모니터링할 것

> **SNS 팬아웃**
> - 하나의 이벤트를 여러 SQS 큐에 동시 전달
> - 구독자별 메시지 필터링 지원
> - SQS 정책에 SNS SendMessage 권한 추가 필요

> **EventBridge**
> - 이벤트 패턴 기반 라우팅으로 복잡한 조건 처리 가능
> - 스키마 레지스트리로 이벤트 계약 문서화
> - AWS 서비스 이벤트 자동화에 최적

> **MSK vs SQS/SNS**
> - 재처리(replay), 고처리량, 이벤트 소싱 → MSK
> - 단순 비동기 처리, 알림, 낮은 운영 부담 → SQS/SNS

---

## 참고 자료

- [Amazon SQS 개발자 가이드](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
- [Amazon SQS FIFO 큐](https://docs.aws.amazon.com/ko_kr/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)
- [Amazon SNS 개발자 가이드](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/welcome.html)
- [SNS 메시지 필터링](https://docs.aws.amazon.com/ko_kr/sns/latest/dg/sns-message-filtering.html)
- [Amazon EventBridge 사용 설명서](https://docs.aws.amazon.com/ko_kr/eventbridge/latest/userguide/eb-what-is.html)
- [EventBridge 스키마 레지스트리](https://docs.aws.amazon.com/ko_kr/eventbridge/latest/userguide/eb-schema.html)
- [Amazon MSK 개발자 가이드](https://docs.aws.amazon.com/ko_kr/msk/latest/developerguide/what-is-msk.html)
- [AWS 메시징 서비스 선택 가이드](https://aws.amazon.com/ko/messaging/)
