---
title: "[AWS / 클라우드] 4편 — 서버리스: Lambda와 이벤트 드리븐 아키텍처"
date: 2026-03-20T14:05:00+09:00
draft: false
tags: ["Lambda", "서버리스", "Step Functions", "EventBridge", "AWS", "서버"]
series: ["AWS / 클라우드"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "Lambda 실행 모델(콜드 스타트, 동시성, 메모리/타임아웃), 이벤트 소스 연동(API Gateway, SQS, S3, EventBridge), Step Functions 워크플로 오케스트레이션, 서버리스 안티패턴과 컨테이너 비용 교차점까지"
---

## 목차

1. 서버리스란 무엇인가
2. Lambda 실행 모델
3. 콜드 스타트와 웜 스타트
4. 동시성과 스케일링
5. 메모리, CPU, 타임아웃
6. 이벤트 소스 연동
7. Step Functions 워크플로 오케스트레이션
8. 서버리스 안티패턴
9. 서버리스 vs 컨테이너 비용 교차점
10. 참고 자료

---

## 1. 서버리스란 무엇인가

서버리스(Serverless)는 서버가 없다는 뜻이 아니다. 개발자가 서버를 직접 프로비저닝하거나 관리하지 않아도 된다는 의미다.

AWS Lambda는 대표적인 FaaS(Function as a Service) 플랫폼이다. 코드를 업로드하면 AWS가 실행 환경을 자동으로 관리해 준다.

서버리스의 핵심 가치는 세 가지다: **이벤트 기반 실행**, **자동 스케일링**, **사용한 만큼만 과금**.

---

## 2. Lambda 실행 모델

Lambda 함수는 이벤트(Event)를 받아 핸들러(Handler)를 실행하고 결과를 반환한다.

```python
import json

def lambda_handler(event, context):
    """
    event: 트리거에서 전달된 데이터 (dict)
    context: 실행 환경 메타데이터 (남은 시간, 요청 ID 등)
    """
    print(f"Request ID: {context.aws_request_id}")
    print(f"Remaining time: {context.get_remaining_time_in_millis()}ms")

    name = event.get("name", "World")
    return {
        "statusCode": 200,
        "body": json.dumps({"message": f"Hello, {name}!"})
    }
```

Lambda 실행 수명 주기는 Init → Invoke → Shutdown 세 단계로 구성된다.

- **Init**: 런타임 초기화, 코드 다운로드, 핸들러 외부 코드 실행
- **Invoke**: 핸들러 함수 실행
- **Shutdown**: 런타임 종료 (SIGTERM 신호 처리 가능)

핸들러 바깥에 선언된 코드는 Init 단계에서 한 번만 실행된다. DB 연결이나 설정 로드는 이 영역에 두어야 효율적이다.

```python
import boto3

# Init 단계에서 한 번만 실행됨
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("Users")

def lambda_handler(event, context):
    # Invoke 단계마다 실행됨
    user_id = event["userId"]
    response = table.get_item(Key={"userId": user_id})
    return response.get("Item")
```

---

## 3. 콜드 스타트와 웜 스타트

### 콜드 스타트란

Lambda는 요청이 들어오면 새 실행 환경(Execution Environment)을 생성한다. 이 초기화 과정을 **콜드 스타트**라고 한다.

콜드 스타트 시간은 런타임, 함수 크기, VPC 설정에 따라 달라진다. 일반적으로 수백 ms에서 수 초까지 발생할 수 있다.

| 요인 | 영향 |
|------|------|
| 런타임 (Python/Node.js) | 빠름 (수십~수백 ms) |
| 런타임 (Java/C#) | 느림 (1~3초) |
| 함수 패키지 크기 | 클수록 느림 |
| VPC 설정 | 추가 지연 발생 |
| 메모리 크기 | 높을수록 빠름 |

### 웜 스타트

이미 초기화된 실행 환경이 재사용되는 경우를 **웜 스타트**라고 한다. Init 단계를 건너뛰므로 훨씬 빠르다.

Lambda는 실행 환경을 일정 시간 동안 유지하지만, 유지 기간은 보장되지 않는다. 트래픽이 없으면 환경이 제거된다.

### 콜드 스타트 완화 전략

**Provisioned Concurrency**를 사용하면 미리 초기화된 실행 환경을 지정한 수만큼 유지할 수 있다.

```yaml
# serverless.yml (Serverless Framework)
functions:
  api:
    handler: handler.main
    provisionedConcurrency: 5  # 항상 5개의 환경을 초기화 상태로 유지
```

```bash
# AWS CLI로 Provisioned Concurrency 설정
aws lambda put-provisioned-concurrency-config \
  --function-name my-function \
  --qualifier prod \
  --provisioned-concurrent-executions 10
```

**Lambda SnapStart** (Java 전용)는 init 단계 스냅샷을 캐시해 콜드 스타트를 90% 이상 줄인다.

패키지 크기를 줄이는 것도 중요하다. 불필요한 의존성을 제거하고 Lambda Layer로 공통 라이브러리를 분리한다.

---

## 4. 동시성과 스케일링

### 동시성 모델

Lambda는 요청마다 별도의 실행 환경에서 처리된다. 동시 요청 10개라면 10개의 환경이 병렬로 동작한다.

계정 기본 동시성 한도는 리전당 1,000개다. 이 한도를 초과하면 `TooManyRequestsException`이 발생한다.

### 동시성 제어

**Reserved Concurrency**는 특정 함수에 동시성을 예약한다. 다른 함수가 이 예약분을 사용할 수 없다.

```bash
# 함수에 최대 동시성 100 설정
aws lambda put-function-concurrency \
  --function-name critical-function \
  --reserved-concurrent-executions 100
```

Reserved Concurrency를 0으로 설정하면 함수 실행을 완전히 차단할 수 있다. 긴급 배포 롤백 시 유용하다.

**Provisioned Concurrency**와 Reserved Concurrency는 다르다. Provisioned는 웜 상태 환경 수를 보장하고, Reserved는 최대 동시성 상한을 설정한다.

### 스케일링 속도

Lambda는 초당 1,000개 실행 환경까지 즉시 확장할 수 있다. 그 이상은 분당 500개씩 증가한다.

SQS 트리거의 경우 배치 크기와 동시성 설정이 복잡하게 상호작용한다. 큐 처리량을 설계할 때 이 점을 고려해야 한다.

---

## 5. 메모리, CPU, 타임아웃

### 메모리와 CPU

Lambda에서 CPU는 메모리에 비례해 할당된다. 메모리를 늘리면 자동으로 CPU도 증가한다.

1,769 MB 기준으로 vCPU 1개가 온전히 할당된다. 3,008 MB 이상부터는 vCPU 2개가 주어진다.

| 메모리 | CPU | 적합한 워크로드 |
|--------|-----|-----------------|
| 128 MB | 0.07 vCPU | 단순 I/O 작업 |
| 512 MB | 0.28 vCPU | 일반 API 처리 |
| 1769 MB | 1 vCPU | CPU 바운드 작업 |
| 3008 MB | 2 vCPU | 병렬 처리 |
| 10240 MB | 6 vCPU | 이미지/영상 처리 |

CPU 집약적인 작업은 메모리를 늘려 실행 시간을 단축하면 전체 비용이 오히려 줄어드는 경우가 많다.

### 타임아웃

Lambda의 최대 실행 시간은 **15분**이다. 이 시간을 초과하면 강제로 종료된다.

```python
import json

def lambda_handler(event, context):
    # 남은 시간이 1초 미만이면 조기 종료
    if context.get_remaining_time_in_millis() < 1000:
        return {
            "statusCode": 503,
            "body": json.dumps({"error": "Function timeout imminent"})
        }

    # 실제 처리 로직
    result = process_data(event)
    return {"statusCode": 200, "body": json.dumps(result)}
```

15분을 초과하는 작업은 Step Functions나 ECS Fargate로 이관해야 한다.

### AWS Lambda Power Tuning

오픈소스 도구 [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)으로 메모리 설정 최적화를 자동화할 수 있다.

다양한 메모리 설정으로 함수를 실행하고 비용과 성능 그래프를 생성해 준다.

---

## 6. 이벤트 소스 연동

Lambda는 다양한 AWS 서비스와 직접 연동된다. 이벤트 소스에 따라 처리 방식이 다르다.

### API Gateway

HTTP 요청을 Lambda로 라우팅하는 가장 일반적인 패턴이다.

```python
def lambda_handler(event, context):
    """
    API Gateway (HTTP API) 이벤트 구조
    """
    method = event["requestContext"]["http"]["method"]
    path = event["rawPath"]
    body = event.get("body", "{}")
    headers = event.get("headers", {})

    if method == "GET":
        return handle_get(path, event.get("queryStringParameters", {}))
    elif method == "POST":
        import json
        return handle_post(path, json.loads(body))

    return {"statusCode": 405, "body": "Method Not Allowed"}
```

API Gateway REST API와 HTTP API는 다르다. HTTP API가 더 빠르고 저렴하지만 기능이 제한적이다.

### SQS 트리거

SQS 큐에 메시지가 쌓이면 Lambda가 자동으로 폴링해 처리한다.

```python
def lambda_handler(event, context):
    """
    SQS 이벤트: 배치로 레코드가 전달됨
    """
    failed_items = []

    for record in event["Records"]:
        message_id = record["messageId"]
        body = record["body"]

        try:
            process_message(body)
        except Exception as e:
            print(f"Failed to process {message_id}: {e}")
            # 실패한 항목만 재처리 (Report Batch Item Failures)
            failed_items.append({"itemIdentifier": message_id})

    # 실패 항목 반환 시 해당 메시지만 큐로 돌아감
    return {"batchItemFailures": failed_items}
```

`ReportBatchItemFailures` 기능을 사용하면 배치 중 일부 실패 시 성공한 메시지는 삭제하고 실패한 것만 재처리할 수 있다.

SQS와 Lambda의 동시성 관계에 유의해야 한다. 큐의 파티션 수가 Lambda 동시성에 영향을 준다.

### S3 이벤트

파일 업로드 시 Lambda를 트리거하는 패턴이다.

```python
import boto3
import urllib.parse

s3 = boto3.client("s3")

def lambda_handler(event, context):
    """
    S3 이벤트: 파일 업로드/삭제 등
    """
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key = urllib.parse.unquote_plus(
            record["s3"]["object"]["key"],
            encoding="utf-8"
        )
        size = record["s3"]["object"].get("size", 0)
        event_name = record["eventName"]

        print(f"Event: {event_name}, Bucket: {bucket}, Key: {key}, Size: {size}")

        if event_name.startswith("ObjectCreated"):
            process_new_file(bucket, key)
```

S3 이벤트는 최소 1회 전달을 보장한다. 중복 처리에 대한 멱등성(Idempotency) 설계가 필요하다.

### EventBridge

이벤트 버스 방식으로 느슨하게 결합된 이벤트 드리븐 아키텍처를 구성할 수 있다.

```python
import boto3
import json
from datetime import datetime

events_client = boto3.client("events")

def publish_order_event(order_id: str, status: str):
    """
    EventBridge에 커스텀 이벤트 발행
    """
    response = events_client.put_events(
        Entries=[
            {
                "Time": datetime.utcnow(),
                "Source": "com.myapp.orders",
                "DetailType": "OrderStatusChanged",
                "Detail": json.dumps({
                    "orderId": order_id,
                    "status": status,
                    "timestamp": datetime.utcnow().isoformat()
                }),
                "EventBusName": "default"
            }
        ]
    )
    return response
```

```yaml
# EventBridge 룰: 주문 완료 이벤트를 Lambda로 라우팅
Resources:
  OrderCompletedRule:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: default
      EventPattern:
        source:
          - "com.myapp.orders"
        detail-type:
          - "OrderStatusChanged"
        detail:
          status:
            - "COMPLETED"
      Targets:
        - Arn: !GetAtt NotificationLambda.Arn
          Id: "NotificationTarget"
```

EventBridge Schedule을 사용하면 Cron 기반 스케줄 실행도 Lambda로 처리할 수 있다.

---

## 7. Step Functions 워크플로 오케스트레이션

Lambda는 단일 함수다. 여러 단계로 구성된 복잡한 비즈니스 로직은 **Step Functions**로 오케스트레이션한다.

### Step Functions 기본 개념

Step Functions는 ASL(Amazon States Language)로 상태 머신을 정의한다. 각 상태(State)는 Lambda 호출, 조건 분기, 병렬 실행, 대기 등의 작업을 수행한다.

```json
{
  "Comment": "주문 처리 워크플로",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-northeast-2:123456789:function:validate-order",
      "Next": "CheckInventory",
      "Catch": [
        {
          "ErrorEquals": ["ValidationError"],
          "Next": "OrderFailed"
        }
      ]
    },
    "CheckInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-northeast-2:123456789:function:check-inventory",
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-northeast-2:123456789:function:process-payment",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Next": "OrderCompleted"
    },
    "OrderCompleted": {
      "Type": "Succeed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "Order validation or payment failed"
    }
  }
}
```

### 병렬 실행

`Parallel` 상태를 사용하면 여러 Lambda를 동시에 실행할 수 있다.

```json
{
  "NotifyAll": {
    "Type": "Parallel",
    "Branches": [
      {
        "StartAt": "SendEmail",
        "States": {
          "SendEmail": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:::function:send-email",
            "End": true
          }
        }
      },
      {
        "StartAt": "SendSMS",
        "States": {
          "SendSMS": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:::function:send-sms",
            "End": true
          }
        }
      }
    ],
    "Next": "WorkflowComplete"
  }
}
```

### Express vs Standard 워크플로

| 구분 | Standard | Express |
|------|----------|---------|
| 최대 실행 시간 | 1년 | 5분 |
| 실행 보장 | Exactly-once | At-least-once |
| 감사 로그 | 90일 보관 | CloudWatch Logs |
| 비용 | 상태 전환당 | 실행 횟수 + 기간 |
| 적합한 용도 | 주문 처리, 승인 플로우 | 고처리량 이벤트 처리 |

짧고 빈번한 워크플로는 Express를 사용해야 비용이 합리적이다.

---

## 8. 서버리스 안티패턴

### 1. 거대한 단일 Lambda (Monolithic Lambda)

모든 라우팅과 비즈니스 로직을 하나의 Lambda에 몰아넣는 패턴이다.

배포 단위가 커지고, 불필요한 코드가 항상 메모리에 올라가며, 독립적인 스케일링이 불가능해진다.

함수를 도메인 또는 기능별로 분리하는 것이 올바른 방향이다.

### 2. 동기 체인 (Lambda Calling Lambda)

Lambda가 다른 Lambda를 직접 동기 호출하는 패턴이다.

```python
# 나쁜 패턴
import boto3

lambda_client = boto3.client("lambda")

def lambda_handler(event, context):
    # Lambda A가 Lambda B를 동기 호출
    response = lambda_client.invoke(
        FunctionName="function-b",
        InvocationType="RequestResponse",  # 동기 호출
        Payload=json.dumps(event)
    )
    # 두 함수의 실행 시간이 합산됨
    # 에러 전파가 복잡해짐
    # 비용이 두 배로 과금됨
```

이벤트 방식으로 전환하거나, Step Functions로 오케스트레이션하는 것이 낫다.

### 3. 영구 연결이 필요한 워크로드

RDS 같은 전통적인 관계형 데이터베이스에 Lambda가 직접 연결되면 문제가 생긴다.

Lambda의 동시성이 증가할수록 DB 연결 수가 폭발적으로 증가한다. RDS Proxy를 반드시 사용해야 한다.

```yaml
# RDS Proxy를 통한 Lambda 연결
Resources:
  MyFunction:
    Type: AWS::Lambda::Function
    Properties:
      Environment:
        Variables:
          DB_HOST: !GetAtt RDSProxy.Endpoint  # RDS 직접 연결 대신 Proxy 사용
          DB_PORT: "5432"
```

### 4. 상태 저장

Lambda 실행 환경은 재사용될 수도, 새로 생성될 수도 있다. 실행 환경에 상태를 저장하면 안 된다.

영구 상태는 DynamoDB, ElastiCache, S3 같은 외부 저장소를 사용해야 한다.

### 5. 15분 초과 작업

Lambda의 최대 실행 시간은 15분이다. 영상 인코딩, 대용량 데이터 처리 같은 장시간 작업은 적합하지 않다.

이런 워크로드는 ECS Fargate, AWS Batch, EC2로 처리해야 한다.

### 6. VPC 남용

Lambda를 VPC에 연결하면 ENI(Elastic Network Interface) 생성으로 콜드 스타트가 늘어난다.

VPC는 실제로 필요한 경우(RDS, ElastiCache 접근)에만 설정해야 한다. 외부 API 호출만 하는 Lambda는 VPC가 필요 없다.

---

## 9. 서버리스 vs 컨테이너 비용 교차점

서버리스가 항상 저렴한 것은 아니다. 트래픽 패턴과 실행 특성에 따라 컨테이너가 유리할 수 있다.

### Lambda 비용 구조

Lambda는 **호출 횟수**와 **실행 시간(GB-초)** 기준으로 과금된다.

```
비용 = (호출 횟수 × $0.0000002) + (GB-초 × $0.0000166667)

예시: 512 MB Lambda, 100ms 실행, 월 1억 회 호출
- 호출 비용: 1억 × $0.0000002 = $20
- 실행 비용: 1억 × 0.1초 × 0.5GB × $0.0000166667 ≈ $83
- 총계: 약 $103/월 (프리 티어 제외)
```

### ECS Fargate 비용 구조

Fargate는 **vCPU 시간**과 **메모리 시간** 기준으로 과금된다. 요청이 없어도 컨테이너가 실행 중이면 비용이 발생한다.

```
비용 = (vCPU 시간 × $0.04048) + (GB 시간 × $0.004445)

예시: 0.5 vCPU, 1 GB, 30일 상시 실행
- vCPU: 720시간 × 0.5 × $0.04048 ≈ $14.6
- 메모리: 720시간 × 1 × $0.004445 ≈ $3.2
- 총계: 약 $17.8/월
```

### 교차점 분석

트래픽이 **산발적이고 예측 불가능**한 경우: Lambda가 유리하다.

트래픽이 **지속적이고 높은 처리량**인 경우: Fargate가 유리하다.

실행 시간이 **수십 밀리초** 수준이고 호출이 잦다면 Lambda 비용이 빠르게 증가한다.

```
교차점 계산 예시 (512 MB, 100ms 함수):
Lambda 월간 비용 = Fargate 고정 비용
약 월 200만 회 이상 호출 시 Fargate가 더 저렴해질 수 있음
(실제 수치는 메모리, 실행시간, 리전에 따라 다름)
```

### 하이브리드 전략

실제 프로덕션 환경에서는 두 가지를 혼합하는 경우가 많다.

- 산발적 이벤트 처리 → Lambda
- 상시 실행 API 서버 → Fargate 또는 EKS
- 배치 작업 → AWS Batch 또는 Fargate
- 장시간 작업 → EC2 또는 Fargate

비용 최적화를 위해 AWS Compute Optimizer와 Cost Explorer를 정기적으로 검토해야 한다.

---

## 핵심 포인트 요약

> **콜드 스타트**: Init 단계 비용. Provisioned Concurrency로 완화. Java는 SnapStart 사용.

> **동시성**: Reserved로 상한 설정, Provisioned로 웜 환경 보장. 계정 한도 1,000개 주의.

> **메모리 = CPU**: 메모리를 높이면 CPU도 늘어남. CPU 집약 작업은 메모리 증가가 비용 절감으로 이어질 수 있음.

> **이벤트 소스**: API Gateway, SQS, S3, EventBridge 각각 처리 방식이 다름. SQS는 Report Batch Item Failures 활용.

> **Step Functions**: Lambda 체인을 대체. 재시도, 병렬 실행, 오류 처리를 선언적으로 관리.

> **비용 교차점**: 트래픽이 지속적이고 높으면 Fargate가 유리. 산발적이면 Lambda가 유리.

---

## 참고 자료

- [AWS Lambda 개발자 가이드](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/welcome.html)
- [AWS Lambda 실행 환경 수명 주기](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/lambda-runtime-environment.html)
- [Lambda Provisioned Concurrency](https://docs.aws.amazon.com/ko_kr/lambda/latest/dg/provisioned-concurrency.html)
- [AWS Step Functions 개발자 가이드](https://docs.aws.amazon.com/ko_kr/step-functions/latest/dg/welcome.html)
- [Amazon EventBridge 사용 설명서](https://docs.aws.amazon.com/ko_kr/eventbridge/latest/userguide/eb-what-is.html)
- [AWS Lambda Power Tuning (GitHub)](https://github.com/alexcasalboni/aws-lambda-power-tuning)
- [Serverless Land — AWS 서버리스 패턴 모음](https://serverlessland.com/)
- [AWS Compute Optimizer](https://docs.aws.amazon.com/ko_kr/compute-optimizer/latest/ug/what-is-compute-optimizer.html)
