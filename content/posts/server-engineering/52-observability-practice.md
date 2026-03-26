---
title: "[로그, 모니터링, 관측 가능성] 5편 — 관측 가능성 실전: 장애를 빠르게 찾는 법"
date: 2026-03-17T19:01:00+09:00
draft: false
tags: ["SLI", "SLO", "관측 가능성", "포스트모템", "장애 대응", "서버"]
series: ["로그, 모니터링, 관측 가능성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 7
summary: "로그+메트릭+트레이스 연계 분석, 장애 진단 플로우(알림→대시보드→트레이스→로그), SLI/SLO/SLA 정의와 Error Budget, 포스트모템 작성법과 5 Whys까지"
---

새벽 3시, PagerDuty 알림이 울린다. 결제 API의 에러율이 15%를 넘었다. 대시보드를 열어보니 레이턴시 P99가 8초를 찍고 있다.

로그를 뒤지지만 수백만 줄 사이에서 원인을 찾기란 모래사장에서 바늘 찾기다.

관측 가능성(Observability)이 제대로 갖춰진 시스템이라면 이 상황을 10분 안에 해결할 수 있다. 그렇지 않다면 몇 시간이 걸릴 수도 있다.

---

## 관측 가능성의 세 기둥을 연계한다

로그, 메트릭, 트레이스는 각자 독립적인 데이터 소스가 아니다.

세 기둥이 서로 연결될 때 비로소 "무슨 일이 벌어지고 있는지"를 넘어 "왜 벌어지는지"까지 파악할 수 있다.

```text
[관측 가능성 세 기둥의 역할과 연계]

          메트릭 (Prometheus/Grafana)
          "무엇이 문제인가?"
          에러율 ↑, 레이턴시 ↑
                    │
                    │ traceId로 드릴다운
                    ↓
          트레이스 (Jaeger/Zipkin)
          "어디서 문제인가?"
          Inventory Service 900ms 지연 발견
                    │
                    │ traceId로 로그 필터링
                    ↓
           로그 (ELK/Loki)
           "왜 문제인가?"
           "connection pool exhausted"
```

### 메트릭은 "무엇이"를 알려준다

메트릭은 시스템의 집계된 상태를 숫자로 표현한다.

에러율이 올라갔다, 레이턴시가 늘어났다, CPU가 치솟았다는 사실을 알 수 있다.

```promql
# 결제 서비스 에러율 (5분 이동 평균)
rate(http_requests_total{service="payment", status=~"5.."}[5m])
/
rate(http_requests_total{service="payment"}[5m])
```

하지만 메트릭만으로는 "어느 특정 요청에서" 문제가 생겼는지 알기 어렵다.

### 트레이스는 "어디서"를 알려준다

분산 트레이싱은 하나의 요청이 여러 서비스를 거치는 경로를 시각화한다.

레이턴시 문제라면 어느 서비스, 어느 스팬(span)에서 시간이 가장 많이 소비되는지 즉시 파악된다.

```json
{
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "operationName": "payment.process",
  "startTime": 1742000000000000,
  "duration": 7843000,
  "tags": {
    "http.status_code": 500,
    "db.statement": "SELECT * FROM orders WHERE user_id = ?",
    "error": true
  }
}
```

7.8초짜리 스팬이 있다면, 그 안의 자식 스팬을 파고들면 DB 쿼리 하나가 7초를 먹고 있다는 사실을 발견할 수 있다.

### 로그는 "왜"를 알려준다

트레이스로 의심 지점을 좁혔다면, 해당 `traceId`로 로그를 필터링한다.

```bash
# Loki 쿼리: 특정 트레이스의 로그만 추출
{service="payment"} | json | traceId="4bf92f3577b34da6a3ce929d0e0e4736"
```

이렇게 하면 수백만 줄의 로그 중 해당 요청에 관련된 로그만 추려낼 수 있다.

`"connection pool exhausted: max=10, in-use=10"` 같은 메시지가 등장한다면, DB 커넥션 풀이 한계에 달했다는 근본 원인을 찾은 것이다.

### 연계를 위한 공통 식별자

세 기둥을 연결하는 열쇠는 **공통 식별자**다.

모든 로그, 메트릭의 레이블, 트레이스 스팬에 동일한 `traceId`와 `service` 이름을 심어야 한다.

```python
import logging
from opentelemetry import trace

logger = logging.getLogger(__name__)

def process_payment(order_id: str):
    span = trace.get_current_span()
    trace_id = format(span.get_span_context().trace_id, '032x')

    # 로그에 traceId 자동 삽입
    logger.info("Processing payment", extra={
        "traceId": trace_id,
        "orderId": order_id,
        "service": "payment"
    })
```

OpenTelemetry를 사용하면 이 작업이 자동화된다.

미들웨어 한 줄로 모든 요청에 `traceId`가 전파된다.

---

## 장애 진단 플로우: 알림에서 근본 원인까지

세 기둥의 연계 방법을 알았다면, 이제 실제 장애 상황에서 어떤 순서로 이 도구들을 활용하는지 살펴볼 차례다.

체계적인 진단 플로우 없이 로그를 무작정 뒤지는 것은 시간 낭비다.

효율적인 진단은 넓은 범위에서 좁은 범위로, 빠르게 가설을 세우고 기각하는 과정이다.

### 1단계: 알림 수신 — "무슨 문제인가"

PagerDuty나 OpsGenie 알림이 왔을 때 가장 먼저 확인해야 할 것은 **알림의 맥락**이다.

단순히 "에러율 5% 초과"가 아니라, 어느 서비스의, 어느 엔드포인트에서, 언제부터 문제가 시작됐는지를 알림 자체에 담아야 한다.

```yaml
# Alertmanager 규칙 예시
groups:
  - name: payment_alerts
    rules:
      - alert: PaymentHighErrorRate
        expr: |
          rate(http_requests_total{service="payment", status=~"5.."}[5m])
          /
          rate(http_requests_total{service="payment"}[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
          team: payment
        annotations:
          summary: "결제 서비스 에러율 {{ $value | humanizePercentage }} 초과"
          description: "서비스: {{ $labels.service }}, 인스턴스: {{ $labels.instance }}"
          runbook_url: "https://wiki.internal/runbooks/payment-errors"
          dashboard_url: "https://grafana.internal/d/payment-overview"
```

`runbook_url`과 `dashboard_url`을 알림에 포함하면 온콜 엔지니어가 첫 30초를 허비하지 않는다.

### 2단계: 대시보드 — "어디서 시작됐나"

대시보드를 열면 전체 그림을 봐야 한다.

에러가 특정 인스턴스에서만 발생하는가, 아니면 전체적으로 발생하는가?

```promql
# 인스턴스별 에러율 히트맵
sum by (instance) (
  rate(http_requests_total{service="payment", status=~"5.."}[5m])
)
/
sum by (instance) (
  rate(http_requests_total{service="payment"}[5m])
)
```

특정 인스턴스 하나에서만 에러가 나온다면 배포 문제나 하드웨어 문제일 가능성이 높다.

전체 인스턴스에서 동시에 에러가 난다면 DB, 외부 API, 네트워크 등 공유 의존성을 먼저 의심해야 한다.

**Grafana에서 시간 범위 조정이 핵심이다.**

문제 발생 시각 기준으로 배포 이벤트(Deployment annotation)가 겹치는지 확인한다.

```json
{
  "annotations": {
    "list": [
      {
        "datasource": "-- Grafana --",
        "enable": true,
        "name": "Deployments",
        "tags": ["deployment"],
        "type": "tags"
      }
    ]
  }
}
```

배포 직후 에러율이 상승했다면, 새 버전의 코드 변경이 원인일 가능성이 매우 높다.

### 3단계: 트레이스 — "어떤 요청이 느린가"

메트릭에서 의심 지점을 파악했다면, Jaeger나 Tempo에서 느린 트레이스를 탐색한다.

```bash
# Tempo HTTP API로 고레이턴시 트레이스 검색
curl "http://tempo:3200/api/search?service.name=payment&minDuration=5s&limit=20"
```

상위 20개의 느린 요청을 열어보면 패턴이 보인다.

특정 `user_id`나 특정 `region`에서만 느린 트레이스가 발생한다면, 데이터 편향이나 지역 특화 문제를 의심할 수 있다.

### 4단계: 로그 — "정확히 무슨 에러인가"

트레이스에서 문제 있는 `traceId`를 찾았다면, 로그에서 해당 ID로 직접 필터링한다.

```logql
# Grafana Loki: traceId 기반 로그 검색
{service="payment"}
| json
| traceId = "4bf92f3577b34da6a3ce929d0e0e4736"
| line_format "{{.level}} {{.msg}} {{.error}}"
```

스택 트레이스, 에러 메시지, SQL 쿼리 등 구체적인 증거가 나온다.

이 단계에서 근본 원인을 확정하고, 즉각 대응책(롤백, 쿼리 최적화, 스케일 아웃 등)을 결정한다.

### 안티패턴: 로그부터 뒤지는 실수

많은 엔지니어가 알림을 받자마자 로그를 뒤지기 시작한다.

맥락 없이 로그를 검색하면 관련 없는 에러 메시지에 속아 엉뚱한 방향으로 달려가기 쉽다.

반드시 메트릭 → 트레이스 → 로그 순서로 범위를 좁혀가야 한다.

---

## SLI, SLO, SLA: 장애 기준을 수치로 정의한다

관측 가능성의 궁극적인 목적은 서비스 품질을 측정하고 관리하는 것이다.

SLI, SLO, SLA는 그 측정 기준을 체계화한 프레임워크다.

### SLI: 무엇을 측정할 것인가

**SLI(Service Level Indicator)**는 서비스 품질을 나타내는 정량적 측정값이다.

가장 중요한 SLI는 사용자가 직접 체감하는 것들이다.

| SLI 종류 | 측정 방법 | 예시 |
|---------|---------|-----|
| 가용성 | 성공 요청 / 전체 요청 | HTTP 2xx 비율 |
| 레이턴시 | 특정 퍼센타일 응답 시간 | P99 < 500ms |
| 처리량 | 단위 시간당 처리 요청 수 | 1000 RPS |
| 에러율 | 실패 요청 / 전체 요청 | 5xx 비율 < 0.1% |

```promql
# 가용성 SLI: 지난 30일간 성공률
(
  sum(increase(http_requests_total{service="payment", status!~"5.."}[30d]))
  /
  sum(increase(http_requests_total{service="payment"}[30d]))
) * 100
```

### SLO: 목표치를 설정한다

**SLO(Service Level Objective)**는 SLI의 목표 범위다.

"결제 API의 가용성은 30일 롤링 윈도우 기준 99.9% 이상이어야 한다"처럼 구체적으로 정의한다.

SLO 설정의 핵심은 **달성 가능한 수준을 현실적으로 정의하는 것**이다.

99.999%를 목표로 삼으면 Error Budget이 너무 작아서 배포 자체가 불가능해진다.

```yaml
# SLO 정의 예시 (OpenSLO 형식)
apiVersion: openslo/v1
kind: SLO
metadata:
  name: payment-availability
spec:
  service: payment-service
  sli:
    metricSource:
      type: Prometheus
      spec:
        totalMetric: rate(http_requests_total{service="payment"}[5m])
        goodMetric: rate(http_requests_total{service="payment", status!~"5.."}[5m])
  objectives:
    - displayName: "99.9% 가용성"
      target: 0.999
      timeWindow:
        duration: 30d
        isRolling: true
```

### Error Budget: 실패할 수 있는 여유

**Error Budget**은 SLO를 위반하지 않는 범위에서 허용되는 실패의 총량이다.

99.9% SLO라면 30일 기준 약 43.8분의 다운타임이 허용된다.

```
Error Budget = (1 - SLO 목표) × 기간
             = (1 - 0.999) × 30일 × 24시간 × 60분
             = 43.2분
```

Error Budget이 소진되면 새로운 기능 배포를 중단하고 안정성 개선에 집중해야 한다.

```promql
# Error Budget 소진율 계산
1 - (
  sum(increase(http_requests_total{service="payment", status!~"5.."}[30d]))
  /
  sum(increase(http_requests_total{service="payment"}[30d]))
) / (1 - 0.999)
```

이 값이 1을 초과하면 Error Budget이 소진된 것이다.

### SLA: 외부 약속

**SLA(Service Level Agreement)**는 고객과의 계약이다.

SLO보다 느슨하게 설정하는 것이 일반적이다.

내부 SLO는 99.9%이지만, 외부 SLA는 99.5%로 설정하는 식이다.

SLO 위반이 SLA 위반으로 이어지지 않도록 완충 구간을 두는 것이 핵심이다.

### Grafana에서 Error Budget 대시보드 구성

```json
{
  "panels": [
    {
      "title": "Error Budget 잔여량",
      "type": "gauge",
      "targets": [
        {
          "expr": "1 - ((1 - sum(rate(http_requests_total{service='payment',status!~'5..'}[30d])) / sum(rate(http_requests_total{service='payment'}[30d]))) / (1 - 0.999))",
          "legendFormat": "Budget 잔여율"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "thresholds": {
            "steps": [
              { "value": 0, "color": "red" },
              { "value": 0.25, "color": "yellow" },
              { "value": 0.5, "color": "green" }
            ]
          },
          "unit": "percentunit",
          "min": 0,
          "max": 1
        }
      }
    }
  ]
}
```

---

## 포스트모템: 장애를 학습 기회로 바꾼다

장애 자체보다 더 중요한 것은 **장애에서 무엇을 배웠는가**다.

포스트모템(Post-Mortem)은 장애를 분석하고 재발 방지책을 도출하는 체계적인 프로세스다.

### 포스트모템의 기본 원칙

**비처벌 문화(Blameless Culture)**가 전제되어야 한다.

"누가 잘못했는가"를 찾는 것이 목적이 아니라, "시스템이 왜 실패했는가"를 이해하는 것이 목적이다.

개인을 탓하면 엔지니어들이 사실을 숨기거나 위험을 과소보고하게 된다.

### 포스트모템 문서 구조

좋은 포스트모템 문서는 누가 읽어도 사건을 재구성할 수 있어야 한다.

```markdown
# 포스트모템: 결제 서비스 장애 (2026-03-15)

## 요약
2026년 3월 15일 02:15~03:42 (87분간) 결제 API 에러율이 18%까지 상승.
약 2,400건의 결제 실패 발생. 원인: DB 커넥션 풀 설정 오류.

## 영향 범위
- 영향 시작: 2026-03-15 02:15 KST
- 영향 종료: 2026-03-15 03:42 KST
- 총 다운타임: 87분
- 영향받은 사용자: 약 12,000명
- 실패 거래: 2,400건 (금액: 약 4,800만원)
- SLO 위반: 월간 Error Budget 43.2분 중 87분 소비 (201% 초과)

## 타임라인
| 시각 | 사건 |
|------|------|
| 02:15 | 결제 에러율 5% 초과, Alertmanager 알림 발송 |
| 02:18 | 온콜 엔지니어 김철수 응답 |
| 02:25 | Grafana에서 특정 2개 인스턴스만 에러 발생 확인 |
| 02:31 | Tempo에서 DB 스팬 레이턴시 7초 이상 확인 |
| 02:38 | Loki에서 "connection pool exhausted" 로그 발견 |
| 02:45 | DB 커넥션 풀 설정 확인, max_connections=5로 설정된 것 발견 |
| 03:10 | 긴급 패치 배포 (max_connections=20으로 수정) |
| 03:42 | 에러율 정상화 확인, 장애 종료 선언 |

## 근본 원인
신규 배포(v2.3.1) 과정에서 DB 커넥션 풀 설정이
기본값(5)으로 리셋되었음.
ConfigMap 분리 작업 중 환경변수 오버라이드가 누락됨.

## 5 Whys 분석

**Why 1**: 왜 결제 API 에러가 발생했는가?
→ DB 커넥션 풀이 고갈되어 새 연결을 맺을 수 없었다.

**Why 2**: 왜 커넥션 풀이 고갈됐는가?
→ 풀의 최대 연결 수가 5로 설정되어 있었고, 동시 요청이 이를 초과했다.

**Why 3**: 왜 최대 연결 수가 5였는가?
→ 배포 과정에서 환경변수 오버라이드가 적용되지 않아 기본값이 사용됐다.

**Why 4**: 왜 환경변수 오버라이드가 적용되지 않았는가?
→ ConfigMap 분리 작업 중 Helm chart의 values.yaml 우선순위 설정에 오류가 있었다.

**Why 5**: 왜 배포 전에 이를 발견하지 못했는가?
→ 스테이징 환경에서 DB 커넥션 설정값을 검증하는 테스트가 없었다.

## 재발 방지 액션 아이템

| 우선순위 | 액션 | 담당자 | 기한 |
|---------|------|--------|------|
| P0 | 배포 파이프라인에 커넥션 풀 설정값 검증 단계 추가 | 박영희 | 2026-03-22 |
| P0 | 스테이징 환경에서 DB 연결 수 모니터링 활성화 | 이민준 | 2026-03-22 |
| P1 | ConfigMap 변경 사항 코드 리뷰 체크리스트 추가 | 김철수 | 2026-03-29 |
| P1 | 커넥션 풀 고갈 알림 추가 (pool_usage > 80%) | 박영희 | 2026-03-29 |
| P2 | DB 설정값 변경 감지 자동 테스트 작성 | 이민준 | 2026-04-05 |
```

### 5 Whys 분석법의 실전 적용

5 Whys는 표면적인 원인이 아니라 시스템적 원인을 찾기 위한 도구다.

첫 번째 "왜"는 보통 증상을 설명한다.

진짜 근본 원인은 보통 네 번째, 다섯 번째 "왜"에서 등장한다.

**중요한 원칙**: "사람이 실수했다"는 절대 최종 답이 될 수 없다.

사람은 항상 실수한다. 그 실수가 장애로 이어지지 않도록 시스템이 어떻게 설계됐어야 하는지를 묻는 것이 5 Whys의 본질이다.

### 장애 원인 분류 체계

장애를 체계적으로 분류하면 패턴이 보인다.

```
장애 원인 분류
├── 코드 변경 (배포)
│   ├── 기능 버그
│   ├── 설정 오류
│   └── 의존성 충돌
├── 인프라
│   ├── 용량 부족 (CPU, 메모리, 디스크)
│   ├── 네트워크 장애
│   └── 하드웨어 장애
├── 외부 의존성
│   ├── 서드파티 API 장애
│   ├── DB 장애
│   └── 메시지 큐 장애
└── 운영 실수
    ├── 잘못된 설정 변경
    ├── 데이터 삭제/변조
    └── 점검 누락
```

분류 결과를 통계로 추적하면 "우리 팀의 장애 중 60%가 배포 관련"이라는 인사이트를 얻을 수 있다.

그 인사이트가 투자 우선순위를 결정한다.

### 포스트모템의 안티패턴

**액션 아이템 없는 포스트모템**은 시간 낭비다.

원인을 파악했다면 반드시 구체적인 액션 아이템이 나와야 한다.

"모니터링을 강화한다"는 액션 아이템이 아니다. "결제 서비스의 DB 커넥션 풀 사용률이 80%를 초과하면 P2 알림을 발송하는 Alertmanager 규칙을 2주 내에 추가한다"가 진짜 액션 아이템이다.

**기한 없는 액션 아이템**도 마찬가지다.

담당자와 기한이 없으면 아무도 실행하지 않는다.

---

## 실전 관측 가능성 체크리스트

장애 대응 능력을 높이기 위해 아래 항목들을 정기적으로 점검하자.

### 계측(Instrumentation) 체크리스트

- [ ] 모든 외부 요청(HTTP, DB, 캐시)에 레이턴시 메트릭이 있는가
- [ ] 에러 로그에 `traceId`가 포함되는가
- [ ] 모든 서비스에 `service.name`이 일관되게 설정됐는가
- [ ] 주요 비즈니스 이벤트(결제 완료, 주문 생성)가 로깅되는가
- [ ] 구조화 로그(JSON)를 사용하는가

### 알림 체크리스트

- [ ] 알림에 `runbook_url`이 포함됐는가
- [ ] 알림 노이즈가 과도하지 않은가 (주당 50건 이하)
- [ ] 알림이 실제 사용자 영향을 반영하는가 (SLI 기반)
- [ ] 알림 해결 시간(MTTR)을 추적하는가

### 대시보드 체크리스트

- [ ] RED 메트릭(Rate, Errors, Duration)이 모두 있는가
- [ ] 배포 이벤트 어노테이션이 표시되는가
- [ ] 인스턴스별 분리 가능한가
- [ ] 트레이스와 로그로 드릴다운 가능한가

---

## 실전 장애 시나리오: DB 슬로우 쿼리 탐지

실제 장애 상황을 단계별로 따라가보자.

**상황**: 오후 2시, 주문 조회 API의 P99 레이턴시가 갑자기 3초에서 12초로 상승.

**1단계 (대시보드)**: 특정 인스턴스가 아니라 전체 인스턴스에서 레이턴시가 상승함을 확인.

공유 의존성 문제로 가설을 좁힌다.

```promql
# DB 쿼리 레이턴시 분포 확인
histogram_quantile(0.99,
  rate(db_query_duration_seconds_bucket{service="order"}[5m])
)
```

DB 쿼리 P99가 11초임을 발견한다.

**2단계 (트레이스)**: Tempo에서 느린 트레이스를 샘플링한다.

```
order-service [12.3s]
  └── order.getOrders [12.2s]
        └── db.query [11.8s]
              operation: SELECT * FROM orders WHERE ...
```

DB 스팬이 11.8초를 차지하고 있다.

**3단계 (로그)**: 해당 `traceId`로 로그를 필터링한다.

```
ERROR db query timeout after 12000ms
  query: SELECT o.*, u.name, u.email, p.amount
         FROM orders o
         JOIN users u ON o.user_id = u.id
         JOIN payments p ON o.id = p.order_id
         WHERE o.created_at > '2026-01-01'
  rows_examined: 15234891
```

`rows_examined`가 1500만 건이다.

인덱스를 타지 않는 풀 스캔이 발생하고 있다.

**4단계 (조치)**: `created_at` 컬럼에 인덱스를 추가하거나, 쿼리 조건을 수정한다.

```sql
-- 즉각 조치: 인덱스 추가
CREATE INDEX CONCURRENTLY idx_orders_created_at
ON orders (created_at)
WHERE created_at > '2026-01-01';
```

이 과정이 10분 안에 이루어졌다면 관측 가능성이 제대로 작동한 것이다.

관측 가능성은 장애를 예방하는 것이 아니다.

장애는 반드시 발생한다. 관측 가능성은 장애가 발생했을 때 **빠르게 찾고, 빠르게 고치고, 다시는 같은 이유로 고통받지 않도록** 하는 시스템이다.

로그, 메트릭, 트레이스를 연계하고, 체계적인 진단 플로우를 만들고, SLO로 기준을 명확히 하고, 포스트모템으로 학습하는 사이클이 반복될수록 시스템은 더 강해진다.

관측 가능성에 투자하는 것은 기술 부채를 갚는 것이 아니라, 엔지니어링 팀의 자신감에 투자하는 것이다.

---

## 참고 자료

1. [Google SRE Book - Chapter 6: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) — SLI/SLO/SLA와 Error Budget의 원전. Google의 실전 경험이 담겨 있다.

2. [OpenTelemetry 공식 문서](https://opentelemetry.io/docs/) — 로그, 메트릭, 트레이스 계측의 표준 구현 가이드.

3. [Grafana Loki Best Practices](https://grafana.com/docs/loki/latest/best-practices/) — 구조화 로그와 LogQL 쿼리 최적화 실전 가이드.

4. [The Postmortem Template by PagerDuty](https://postmortems.pagerduty.com/) — 포스트모템 문서 작성을 위한 실용적인 템플릿과 가이드.

5. [Brendan Gregg - Systems Performance](https://www.brendangregg.com/systems-performance-2nd-edition-book.html) — 시스템 병목 분석과 USE 메서드(Utilization, Saturation, Errors)의 바이블.

6. [SLO Adoption and Usage in Practice (USENIX SREcon)](https://www.usenix.org/conference/srecon19asia/presentation/hidalgo) — 실제 팀에서 SLO를 도입하고 운영하면서 겪은 교훈과 사례.
