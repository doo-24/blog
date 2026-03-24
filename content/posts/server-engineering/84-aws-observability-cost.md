---
title: "[AWS / 클라우드] 7편 — 관측과 비용: CloudWatch, X-Ray, 비용 최적화"
date: 2026-03-20T14:02:00+09:00
draft: false
tags: ["CloudWatch", "X-Ray", "FinOps", "비용 최적화", "AWS", "서버"]
series: ["AWS / 클라우드"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 12
summary: "CloudWatch Metrics·Logs·Alarms 설계, X-Ray 분산 트레이싱과 Insights 쿼리, 비용 최적화 전략(Reserved, Savings Plans, Spot, Right Sizing), FinOps 실천과 비용 가시화까지"
---

## 목차

1. [관측 가능성(Observability)이란](#1-관측-가능성observability이란)
2. [CloudWatch Metrics](#2-cloudwatch-metrics)
3. [CloudWatch Logs와 Insights](#3-cloudwatch-logs와-insights)
4. [CloudWatch Alarms와 대시보드](#4-cloudwatch-alarms와-대시보드)
5. [X-Ray 분산 트레이싱](#5-x-ray-분산-트레이싱)
6. [X-Ray Insights와 ServiceLens](#6-x-ray-insights와-servicelens)
7. [AWS 비용 구조 이해](#7-aws-비용-구조-이해)
8. [Reserved Instances와 Savings Plans](#8-reserved-instances와-savings-plans)
9. [Spot Instance 활용 전략](#9-spot-instance-활용-전략)
10. [Right Sizing과 Compute Optimizer](#10-right-sizing과-compute-optimizer)
11. [FinOps 실천과 비용 가시화](#11-finops-실천과-비용-가시화)
12. [참고 자료](#참고-자료)

---

## 1. 관측 가능성(Observability)이란

시스템이 복잡해질수록 "지금 무슨 일이 일어나고 있는가"를 파악하는 능력이 핵심이 된다.

관측 가능성(Observability)은 외부 출력만으로 시스템 내부 상태를 추론할 수 있는 정도를 뜻한다. 세 가지 기둥인 **메트릭(Metrics)**, **로그(Logs)**, **트레이스(Traces)**가 그 기반이다.

| 구분 | 특징 | AWS 서비스 |
|------|------|-----------|
| Metrics | 수치 기반 시계열 데이터 | CloudWatch Metrics |
| Logs | 이벤트 텍스트 기록 | CloudWatch Logs |
| Traces | 요청 흐름 추적 | AWS X-Ray |

AWS는 이 세 가지를 CloudWatch와 X-Ray로 통합 제공한다.

---

## 2. CloudWatch Metrics

### 메트릭의 기본 개념

CloudWatch Metrics는 AWS 리소스와 애플리케이션의 수치 데이터를 시계열로 수집한다. EC2 CPU 사용률, RDS 연결 수, Lambda 호출 횟수 등이 자동으로 수집된다.

메트릭은 **네임스페이스(Namespace)**, **차원(Dimension)**, **이름(MetricName)**의 조합으로 식별된다.

```
Namespace: AWS/EC2
Dimension: InstanceId=i-0abc12345
MetricName: CPUUtilization
```

### 기본 모니터링 vs 상세 모니터링

EC2 기본 모니터링은 5분 간격으로 수집된다. 상세 모니터링(Detailed Monitoring)을 활성화하면 1분 간격으로 수집되며, 추가 비용이 발생한다.

```bash
# 상세 모니터링 활성화
aws ec2 monitor-instances --instance-ids i-0abc12345

# 메트릭 조회 (최근 1시간, 5분 평균)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc12345 \
  --start-time 2026-03-20T05:00:00Z \
  --end-time 2026-03-20T06:00:00Z \
  --period 300 \
  --statistics Average
```

### 커스텀 메트릭

애플리케이션 레벨의 지표(활성 사용자 수, 주문 처리 시간 등)는 커스텀 메트릭으로 직접 발행한다.

```bash
# 커스텀 메트릭 발행
aws cloudwatch put-metric-data \
  --namespace "MyApp/Orders" \
  --metric-data '[
    {
      "MetricName": "OrderProcessingTime",
      "Dimensions": [
        { "Name": "Environment", "Value": "production" },
        { "Name": "Region", "Value": "ap-northeast-2" }
      ],
      "Value": 142.5,
      "Unit": "Milliseconds",
      "Timestamp": "2026-03-20T06:00:00Z"
    }
  ]'
```

SDK를 통해 코드 내에서 직접 발행하는 방식이 일반적이다. 커스텀 메트릭은 표준 해상도(60초)와 고해상도(1초 단위)를 선택할 수 있다.

### 메트릭 수학(Metric Math)

여러 메트릭을 조합해 새로운 지표를 계산할 수 있다. 예를 들어 요청 성공률을 계산하려면 다음과 같이 수식을 작성한다.

```json
{
  "Metrics": [
    { "Id": "e1", "Expression": "m1/(m1+m2)*100", "Label": "SuccessRate" },
    { "Id": "m1", "MetricStat": { "Metric": { "Namespace": "MyApp", "MetricName": "SuccessCount" }, "Period": 60, "Stat": "Sum" } },
    { "Id": "m2", "MetricStat": { "Metric": { "Namespace": "MyApp", "MetricName": "ErrorCount"   }, "Period": 60, "Stat": "Sum" } }
  ]
}
```

---

## 3. CloudWatch Logs와 Insights

### CloudWatch Logs 구조

CloudWatch Logs는 로그 데이터를 **로그 그룹(Log Group)** > **로그 스트림(Log Stream)** 계층으로 관리한다. 애플리케이션 하나당 로그 그룹 하나를 만드는 것이 일반적인 관례다.

```bash
# 로그 그룹 생성 및 보존 기간 설정
aws logs create-log-group --log-group-name /myapp/production

aws logs put-retention-policy \
  --log-group-name /myapp/production \
  --retention-in-days 30
```

보존 기간을 설정하지 않으면 로그가 영구 저장되어 비용이 계속 증가한다. 반드시 보존 기간을 지정하는 것을 권장한다.

### CloudWatch Logs Insights 쿼리

Logs Insights는 대용량 로그를 SQL과 유사한 쿼리 언어로 빠르게 분석한다. 수 GB 로그도 수 초 내에 결과를 반환한다.

```
# 에러 로그 상위 10건 조회
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 10
```

```
# HTTP 상태 코드별 요청 수 집계
fields @timestamp, statusCode
| stats count(*) as requestCount by statusCode
| sort requestCount desc
```

```
# 응답 시간 p50/p90/p99 분석
fields @timestamp, duration
| filter ispresent(duration)
| stats
    percentile(duration, 50) as p50,
    percentile(duration, 90) as p90,
    percentile(duration, 99) as p99
  by bin(5m)
```

### 메트릭 필터(Metric Filter)

로그에서 특정 패턴을 감지해 자동으로 메트릭을 생성할 수 있다. 애플리케이션 에러 발생 횟수를 실시간 메트릭으로 만드는 예시다.

```bash
aws logs put-metric-filter \
  --log-group-name /myapp/production \
  --filter-name ErrorCount \
  --filter-pattern "ERROR" \
  --metric-transformations '[{
    "metricName": "AppErrorCount",
    "metricNamespace": "MyApp",
    "metricValue": "1",
    "unit": "Count"
  }]'
```

---

## 4. CloudWatch Alarms와 대시보드

### Alarm 설정

Alarm은 메트릭 값이 임계치를 초과하거나 미달할 때 자동 알림 또는 액션을 트리거한다. 상태는 `OK`, `ALARM`, `INSUFFICIENT_DATA` 세 가지다.

```bash
# CPU 사용률 70% 초과 시 알람
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-i-0abc12345" \
  --alarm-description "EC2 CPU usage over 70%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0abc12345 \
  --alarm-actions arn:aws:sns:ap-northeast-2:123456789012:ops-alert \
  --ok-actions arn:aws:sns:ap-northeast-2:123456789012:ops-alert
```

`evaluation-periods 2`는 2번 연속 임계치를 넘어야 알람이 발생함을 의미한다. 단발성 스파이크에 의한 오알람을 방지한다.

### Composite Alarm

여러 Alarm을 AND/OR 조합해 복합 조건 알람을 구성할 수 있다. 예를 들어 CPU가 높으면서 동시에 에러율도 높은 경우에만 페이지를 보내도록 설정한다.

```bash
aws cloudwatch put-composite-alarm \
  --alarm-name "CriticalServiceDegradation" \
  --alarm-rule "ALARM(HighCPU-i-0abc12345) AND ALARM(HighErrorRate-myapp)" \
  --alarm-actions arn:aws:sns:ap-northeast-2:123456789012:pagerduty
```

### CloudWatch 대시보드

대시보드는 여러 메트릭과 알람 상태를 한 화면에 시각화한다.

```bash
aws cloudwatch put-dashboard \
  --dashboard-name "ProductionOverview" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "properties": {
          "title": "API Latency (p99)",
          "metrics": [
            [ "MyApp", "ResponseTime", { "stat": "p99" } ]
          ],
          "period": 60,
          "view": "timeSeries"
        }
      },
      {
        "type": "alarm",
        "properties": {
          "title": "Active Alarms",
          "alarms": [
            "arn:aws:cloudwatch:ap-northeast-2:123456789012:alarm:HighCPU-i-0abc12345"
          ]
        }
      }
    ]
  }'
```

---

## 5. X-Ray 분산 트레이싱

### X-Ray란

마이크로서비스 환경에서 하나의 요청이 여러 서비스를 거칠 때, 각 구간의 처리 시간과 오류를 추적하는 것이 X-Ray다.

X-Ray는 각 요청에 **트레이스 ID**를 부여하고, 서비스 간 이 ID를 HTTP 헤더로 전파한다. 각 서비스의 처리 구간을 **세그먼트(Segment)**, 세그먼트 안의 하위 단위를 **서브세그먼트(Subsegment)**라고 부른다.

### X-Ray SDK 통합 (Python 예시)

```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# AWS SDK, requests, SQLAlchemy 등 자동 계측
patch_all()

@xray_recorder.capture("process_order")
def process_order(order_id: str):
    # 커스텀 어노테이션 (검색 가능)
    xray_recorder.current_subsegment().put_annotation("order_id", order_id)

    # 커스텀 메타데이터 (검색 불가, 디버그용)
    xray_recorder.current_subsegment().put_metadata("payload", {"id": order_id})

    with xray_recorder.in_subsegment("db_query"):
        result = db.query(f"SELECT * FROM orders WHERE id = '{order_id}'")

    return result
```

### X-Ray 샘플링 규칙

모든 요청을 트레이싱하면 비용이 급증한다. 샘플링 규칙으로 트레이싱 비율을 조절한다.

```json
{
  "version": 2,
  "rules": [
    {
      "description": "Health check - low sampling",
      "host": "*",
      "http_method": "GET",
      "url_path": "/health",
      "fixed_target": 0,
      "rate": 0.001
    },
    {
      "description": "API - high sampling",
      "host": "*",
      "http_method": "*",
      "url_path": "/api/*",
      "fixed_target": 1,
      "rate": 0.05
    }
  ],
  "default": {
    "fixed_target": 1,
    "rate": 0.05
  }
}
```

`fixed_target`은 초당 고정 트레이스 수, `rate`는 그 이후의 샘플링 비율이다. 초당 최소 1개는 항상 추적하고, 나머지는 5%만 추적하는 설정이다.

### X-Ray 데몬

EC2나 ECS 환경에서는 X-Ray 데몬이 SDK의 트레이스 데이터를 받아 X-Ray API로 전송한다.

```bash
# X-Ray 데몬 설치 및 실행 (Amazon Linux)
curl -o xray-daemon.rpm \
  https://s3.amazonaws.com/aws-xray-assets.us-east-1/xray-daemon/aws-xray-daemon-3.x.rpm
rpm -U xray-daemon.rpm

/usr/bin/xray --bind 127.0.0.1:2000 &
```

Lambda와 Fargate는 데몬 없이 관리형 트레이싱을 기본 지원한다.

---

## 6. X-Ray Insights와 ServiceLens

### X-Ray Service Map

Service Map은 서비스 간 의존 관계와 각 엣지의 레이턴시·에러율을 시각화한다. 장애 발생 시 어느 서비스가 병목인지 한눈에 파악할 수 있다.

```bash
# 서비스 그래프 조회 (최근 1시간)
aws xray get-service-graph \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s)
```

### X-Ray Insights

X-Ray Insights는 비정상적인 레이턴시 증가나 에러율 급증을 자동으로 감지해 알림을 보낸다. 직접 임계치를 설정하지 않아도 머신러닝 기반으로 이상을 탐지한다.

활성화는 X-Ray 콘솔 > Insights 탭에서 그룹별로 설정한다. SNS 토픽과 연결하면 이상 감지 즉시 알림을 받을 수 있다.

### CloudWatch ServiceLens

ServiceLens는 CloudWatch, X-Ray, CloudWatch Logs를 통합한 단일 뷰다. 서비스 상태를 메트릭·트레이스·로그를 함께 보면서 문제를 분석할 수 있다.

트레이스 상세 화면에서 특정 요청의 로그를 바로 조회하거나, 에러 트레이스에서 연관 CloudWatch 알람으로 이동하는 식의 연계가 가능하다.

---

## 7. AWS 비용 구조 이해

### 주요 비용 항목

AWS 비용은 크게 세 가지로 구성된다.

| 항목 | 예시 |
|------|------|
| 컴퓨팅 | EC2, Lambda, Fargate |
| 스토리지 | S3, EBS, EFS |
| 네트워크 | 데이터 전송(Egress), NAT Gateway |

특히 **네트워크 비용**은 예상치 못하게 큰 비율을 차지하는 경우가 많다. 리전 간 또는 인터넷으로 나가는 트래픽(Egress)에는 GB당 요금이 부과된다.

### 비용 분석 도구

```bash
# 이번 달 서비스별 비용 조회
aws ce get-cost-and-usage \
  --time-period Start=2026-03-01,End=2026-03-31 \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE

# 일별 비용 추이 조회
aws ce get-cost-and-usage \
  --time-period Start=2026-03-01,End=2026-03-20 \
  --granularity DAILY \
  --metrics "UnblendedCost"
```

AWS Cost Explorer 콘솔에서는 필터와 그룹핑을 통해 서비스·태그·계정 단위로 비용을 시각화할 수 있다.

### 태그 기반 비용 배분

리소스에 태그를 붙이면 프로젝트·팀·환경 단위로 비용을 분리할 수 있다.

```bash
# EC2 인스턴스에 비용 배분 태그 부착
aws ec2 create-tags \
  --resources i-0abc12345 \
  --tags \
    Key=Project,Value=myapp \
    Key=Team,Value=backend \
    Key=Environment,Value=production
```

Cost Explorer에서 비용 배분 태그를 활성화하면 해당 태그 기준으로 비용을 필터링할 수 있다. 태그 전략을 초기부터 설계하는 것이 중요하다.

---

## 8. Reserved Instances와 Savings Plans

### 온디맨드 vs 예약 요금

온디맨드는 사용한 만큼 시간 단위로 지불하며 가장 유연하지만 가장 비싸다. 장기 실행이 확실한 워크로드라면 예약을 통해 최대 72%까지 절감할 수 있다.

### Reserved Instances (RI)

특정 인스턴스 타입·리전·OS를 1년 또는 3년 약정으로 예약한다.

| 결제 방식 | 할인율 |
|-----------|--------|
| 전액 선불 (All Upfront) | 최대 |
| 부분 선불 (Partial Upfront) | 중간 |
| 선불 없음 (No Upfront) | 최소 |

```bash
# 사용 가능한 RI 오퍼링 조회
aws ec2 describe-reserved-instances-offerings \
  --instance-type m5.large \
  --product-description "Linux/UNIX" \
  --offering-class standard \
  --offering-type "All Upfront" \
  --filters Name=duration,Values=31536000  # 1년
```

RI는 인스턴스 타입에 묶이기 때문에 유연성이 낮다. 인스턴스 타입 변경이 필요하면 RI 마켓플레이스에서 매도할 수 있지만 손실이 발생할 수 있다.

### Savings Plans

Savings Plans는 특정 인스턴스가 아닌 **시간당 사용 금액**을 약정하는 방식이다. RI보다 유연하며 비슷한 수준의 할인을 제공한다.

세 가지 유형이 있다.

| 유형 | 범위 | 유연성 |
|------|------|--------|
| Compute Savings Plans | EC2, Lambda, Fargate | 최대 (인스턴스 타입·리전 자유) |
| EC2 Instance Savings Plans | 특정 EC2 패밀리 | 중간 |
| SageMaker Savings Plans | SageMaker 전용 | SageMaker 한정 |

```bash
# Savings Plans 권고 조회 (7일 사용량 기준)
aws savingsplans describe-savings-plans-offering-rates \
  --savings-plan-offering-filters \
    filter=region,values=ap-northeast-2 \
    filter=instanceType,values=m5.large
```

일반적으로 **Compute Savings Plans**가 가장 권장된다. 인스턴스 패밀리나 리전을 바꿔도 자동으로 할인이 적용된다.

---

## 9. Spot Instance 활용 전략

### Spot Instance 개념

Spot Instance는 AWS의 유휴 컴퓨팅 자원을 경매 방식으로 구매하는 방식이다. 온디맨드 대비 최대 90%까지 저렴하지만, AWS가 자원을 회수할 때 2분 전 통지 후 종료된다.

Spot에 적합한 워크로드는 다음과 같다.

- 배치 처리, 데이터 변환
- CI/CD 빌드 파이프라인
- 내결함성이 있는 분산 처리 (Spark, EMR)
- 상태 비저장(Stateless) 웹 서버 (Auto Scaling 그룹과 조합)

### Spot Fleet과 혼합 인스턴스 그룹

단일 인스턴스 타입만 사용하면 Spot 가용성이 낮아질 수 있다. 여러 인스턴스 타입과 가용 영역을 조합해 가용성을 높인다.

```json
{
  "MixedInstancesPolicy": {
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateId": "lt-0abc12345",
        "Version": "$Latest"
      },
      "Overrides": [
        { "InstanceType": "m5.large" },
        { "InstanceType": "m5a.large" },
        { "InstanceType": "m4.large" },
        { "InstanceType": "t3.large" }
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 20,
      "SpotAllocationStrategy": "capacity-optimized"
    }
  }
}
```

`OnDemandBaseCapacity 2`는 최소 2개는 온디맨드로 유지하고, 그 이상은 80%를 Spot으로 채우는 설정이다. `capacity-optimized` 전략은 가용 용량이 가장 많은 풀에서 Spot을 선택해 중단 가능성을 낮춘다.

### Spot 중단 처리

2분 중단 통지가 오면 진행 중인 작업을 중단하고 체크포인트를 저장해야 한다. EC2 메타데이터 엔드포인트를 폴링하는 방식으로 중단 신호를 감지한다.

```bash
# Spot 중단 통지 감지 스크립트
while true; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
    http://169.254.169.254/latest/meta-data/spot/termination-time)
  if [ "$STATUS" = "200" ]; then
    echo "Spot termination notice received. Saving checkpoint..."
    # 체크포인트 저장 로직
    break
  fi
  sleep 5
done
```

---

## 10. Right Sizing과 Compute Optimizer

### Right Sizing이란

Right Sizing은 실제 사용량에 맞게 인스턴스 크기를 조정하는 작업이다. 과도하게 프로비저닝된 리소스를 줄여 비용을 절감한다.

많은 경우 처음 인스턴스를 띄울 때 여유 있게 설정하고 이후 재검토하지 않는다. CPU 사용률이 5% 미만인 인스턴스가 수십 개 실행되는 것이 흔한 사례다.

### AWS Compute Optimizer

Compute Optimizer는 CloudWatch 메트릭(기본 2주, 확장 3개월)을 분석해 EC2, EBS, Lambda, ECS 등에 대한 최적 크기를 추천한다.

```bash
# EC2 권고 사항 조회
aws compute-optimizer get-ec2-instance-recommendations \
  --instance-arns arn:aws:ec2:ap-northeast-2:123456789012:instance/i-0abc12345

# 권고 결과 예시
{
  "instanceRecommendations": [
    {
      "currentInstanceType": "m5.2xlarge",
      "finding": "OVER_PROVISIONED",
      "recommendationOptions": [
        {
          "instanceType": "m5.large",
          "projectedUtilizationMetrics": [
            { "name": "CPU", "statistic": "MAXIMUM", "value": 45.2 }
          ],
          "estimatedMonthlySavings": {
            "currency": "USD",
            "value": 87.3
          }
        }
      ]
    }
  ]
}
```

권고를 무조건 따르기보다 최대 부하 시점과 성장 예측을 감안해 결정하는 것이 중요하다.

### 미사용 리소스 정리

```bash
# 연결되지 않은 EBS 볼륨 조회 (비용 발생 중)
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].{ID:VolumeId, Size:Size, Type:VolumeType}'

# 사용하지 않는 Elastic IP 조회 (연결 안 된 EIP는 요금 부과)
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==`null`].PublicIp'

# 오래된 스냅샷 조회 (30일 이상)
aws ec2 describe-snapshots \
  --owner-ids self \
  --query 'Snapshots[?StartTime<=`2026-02-20`].{ID:SnapshotId, Size:VolumeSize}'
```

연결되지 않은 EBS 볼륨과 사용하지 않는 Elastic IP는 요금이 계속 발생하므로 정기적으로 정리한다.

---

## 11. FinOps 실천과 비용 가시화

### FinOps란

FinOps(Financial Operations)는 클라우드 비용을 기술·재무·비즈니스 팀이 공동으로 관리하는 문화와 실천 체계다. 비용 절감만이 목표가 아니라 비용 대비 가치를 극대화하는 것이 핵심이다.

FinOps 성숙도는 세 단계로 구분된다.

| 단계 | 특징 |
|------|------|
| Crawl (정보 수집) | 비용 가시화, 기본 할당 |
| Walk (최적화) | 예약 구매, Right Sizing 실행 |
| Run (자동화) | 자동 알림, 정책 기반 거버넌스 |

### 예산 알림 설정

AWS Budgets를 사용하면 비용이 임계치에 가까워질 때 자동 알림을 받는다.

```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "MonthlySpend",
    "BudgetLimit": { "Amount": "1000", "Unit": "USD" },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        { "SubscriptionType": "EMAIL", "Address": "devops@company.com" }
      ]
    },
    {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [
        { "SubscriptionType": "EMAIL", "Address": "devops@company.com" }
      ]
    }
  ]'
```

실제 지출이 80%를 넘으면 알림을 보내고, 예측 지출이 100%를 초과할 것으로 보이면 선제 알림을 보내는 설정이다.

### Cost Anomaly Detection

Cost Anomaly Detection은 머신러닝으로 비정상적인 비용 증가를 자동 감지한다. 실수로 인스턴스를 대량 생성하거나 NAT Gateway 비용이 급증하는 상황을 빠르게 탐지할 수 있다.

```bash
# 이상 탐지 모니터 생성 (서비스 단위)
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "ServiceMonitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

# 알림 구독 생성 (일 $20 초과 이상 탐지 시)
aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "DailyAnomalyAlert",
    "MonitorArnList": ["arn:aws:ce::123456789012:anomalymonitor/abc123"],
    "Subscribers": [
      { "Address": "devops@company.com", "Type": "EMAIL" }
    ],
    "Threshold": 20,
    "Frequency": "DAILY"
  }'
```

### 비용 최적화 체크리스트

정기적으로 점검할 항목을 정리하면 다음과 같다.

**월별 점검**
- Compute Optimizer 권고 사항 검토
- 미사용 리소스(EBS, EIP, 스냅샷) 정리
- RI/Savings Plans 커버리지 확인 (목표: 70% 이상)

**분기별 점검**
- 새로운 Savings Plans 구매 검토
- 아키텍처 레벨 비용 최적화 기회 탐색 (예: EC2 → Lambda 전환)
- 팀별 비용 배분 정확도 검토 및 태그 전략 보완

**자동화**

```bash
# Lambda + EventBridge로 매일 미사용 리소스 리포트 자동화
aws events put-rule \
  --name "DailyUnusedResourceReport" \
  --schedule-expression "cron(0 9 * * ? *)" \
  --state ENABLED

aws events put-targets \
  --rule "DailyUnusedResourceReport" \
  --targets '[{
    "Id": "ResourceReportFunction",
    "Arn": "arn:aws:lambda:ap-northeast-2:123456789012:function:unused-resource-reporter"
  }]'
```

### 관측성과 비용의 균형

관측 도구 자체도 비용이 발생한다. CloudWatch Logs 수집량, X-Ray 트레이스 수, 커스텀 메트릭 수가 많아질수록 관측 비용도 증가한다.

균형을 잡기 위한 실천 방법은 다음과 같다.

- 로그 보존 기간을 필요에 맞게 설정한다 (전체 30일, 에러만 90일 등)
- X-Ray 샘플링 비율을 환경별로 다르게 설정한다 (개발 100%, 운영 5%)
- 불필요한 커스텀 메트릭과 대시보드를 주기적으로 정리한다
- Logs Insights 쿼리는 비용이 발생하므로 자주 실행되는 쿼리는 메트릭 필터로 대체한다

---

## 핵심 포인트

> **관측 가능성은 비용이다.** 모든 것을 다 수집하기보다 "무엇을 알아야 하는가"를 먼저 정의하고 필요한 데이터만 수집하라.

> **비용 최적화는 일회성 작업이 아니다.** 월별 점검 루틴을 팀 문화로 만들어야 지속적인 절감이 가능하다.

> **Savings Plans가 RI보다 낫다.** 동일한 할인율이라면 인스턴스 타입 유연성이 높은 Compute Savings Plans를 선택하라.

> **태그 전략은 처음부터.** 나중에 태그를 소급 적용하는 것은 매우 어렵다. 리소스 생성 정책에 필수 태그를 강제하라.

---

## 참고 자료

- [Amazon CloudWatch 공식 문서](https://docs.aws.amazon.com/cloudwatch/)
- [AWS X-Ray 개발자 가이드](https://docs.aws.amazon.com/xray/latest/devguide/)
- [AWS Cost Optimization Hub](https://aws.amazon.com/aws-cost-management/cost-optimization-hub/)
- [AWS Compute Optimizer 문서](https://docs.aws.amazon.com/compute-optimizer/)
- [AWS Savings Plans 문서](https://docs.aws.amazon.com/savingsplans/)
- [FinOps Foundation](https://www.finops.org/)
- [CloudWatch Logs Insights 쿼리 구문](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [EC2 Spot Instance 모범 사례](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)
