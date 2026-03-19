---
title: "[로그, 모니터링, 관측 가능성] 3편 — 메트릭과 모니터링: 숫자로 서버 상태 읽기"
date: 2026-03-17T19:03:00+09:00
draft: false
tags: ["Prometheus", "Grafana", "메트릭", "모니터링", "서버"]
series: ["로그, 모니터링, 관측 가능성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 7
summary: "메트릭 유형(Counter, Gauge, Histogram, Summary), RED/USE 방법론, Prometheus 스크레이핑과 PromQL 쿼리, Grafana 대시보드 설계와 알림 규칙 작성까지"
---

서버가 정상인지, 언제 무너질지를 알려면 로그만으로는 부족하다. 로그는 개별 사건을 기록하지만, **메트릭**은 시스템 전체의 건강 상태를 숫자로 요약한다. 초당 요청 수, CPU 사용률, 응답 시간의 99번째 백분위수 — 이 숫자들을 올바르게 수집하고 해석할 수 있을 때 비로소 "장애가 왜 났는가"가 아니라 "장애가 나기 전에 무엇을 해야 했는가"라는 질문에 답할 수 있다.

---

## 메트릭이란 무엇인가

메트릭(metric)은 시간에 따라 변하는 숫자 측정값이다.

단일 이벤트를 기록하는 로그와 달리, 메트릭은 **집계된 상태**를 표현한다. "요청 1개가 실패했다"는 로그지만, "최근 1분간 요청의 2.3%가 실패했다"는 메트릭이다.

메트릭의 핵심 구성 요소는 세 가지다: **이름(name)**, **레이블(label)**, **값(value)**.

```
http_requests_total{method="POST", status="200", handler="/api/users"} 4327
```

이름은 측정 대상을 나타내고, 레이블은 다차원 필터링을 가능하게 하며, 값은 실제 수치다.

---

## 메트릭 유형: 무엇을 어떻게 측정하는가

### Counter — 단조 증가 카운터

Counter는 **오직 증가만 하는** 누적 값이다. 재시작 시에만 0으로 리셋된다.

HTTP 요청 수, 에러 발생 수, 처리한 바이트 수처럼 "지금까지 몇 번 발생했는가"를 측정할 때 사용한다.

```java
// Spring Boot + Micrometer
@Service
public class OrderService {
    private final Counter orderCounter;
    private final Counter orderFailCounter;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("order.created.total")
            .description("Total number of orders created")
            .tag("region", "seoul")
            .register(registry);

        this.orderFailCounter = Counter.builder("order.failed.total")
            .description("Total number of failed orders")
            .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        try {
            Order order = processOrder(request);
            orderCounter.increment();
            return order;
        } catch (Exception e) {
            orderFailCounter.increment();
            throw e;
        }
    }
}
```

Counter 자체는 큰 의미가 없다. `rate()` 함수와 결합해야 "초당 몇 개"라는 유용한 정보가 된다.

**안티패턴**: Counter로 재고 수량이나 현재 접속자 수를 측정하는 것. 감소할 수 있는 값은 Counter가 아니라 Gauge를 써야 한다.

---

### Gauge — 현재 상태값

Gauge는 **임의로 오르내릴 수 있는** 현재 상태 값이다.

현재 활성 연결 수, 큐 대기열 크기, 메모리 사용량, JVM 힙 사용률처럼 "지금 얼마나"를 측정할 때 사용한다.

```java
// 큐 크기를 Gauge로 측정
@Service
public class QueueMonitor {
    private final BlockingQueue<Task> taskQueue;

    public QueueMonitor(MeterRegistry registry, BlockingQueue<Task> taskQueue) {
        this.taskQueue = taskQueue;

        Gauge.builder("queue.size", taskQueue, Queue::size)
            .description("Current number of tasks in queue")
            .tag("queue_name", "order-processing")
            .register(registry);
    }
}
```

Gauge는 람다나 함수 참조로 현재 값을 주기적으로 읽어 등록하는 방식이 권장된다.

**안티패턴**: Gauge 값을 `rate()`로 분석하는 것. Gauge는 현재 상태를 그대로 사용하면 된다.

---

### Histogram — 분포와 백분위수

Histogram은 값의 **분포**를 미리 정의된 버킷(bucket)으로 나눠 기록한다.

응답 시간, 요청 크기, 큐 대기 시간처럼 "얼마나 자주 이 범위에 들어오는가"를 측정할 때 쓴다.

```java
// 응답 시간 Histogram
@Component
public class ApiLatencyRecorder {
    private final Timer requestTimer;

    public ApiLatencyRecorder(MeterRegistry registry) {
        this.requestTimer = Timer.builder("http.server.requests.duration")
            .description("HTTP request processing time")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .sla(
                Duration.ofMillis(100),
                Duration.ofMillis(500),
                Duration.ofSeconds(1)
            )
            .register(registry);
    }

    public <T> T record(Supplier<T> supplier) {
        return requestTimer.record(supplier);
    }
}
```

Prometheus에서 Histogram은 세 가지 시계열을 생성한다.

- `http_request_duration_seconds_bucket{le="0.1"}` — 0.1초 이하 요청 수
- `http_request_duration_seconds_sum` — 전체 응답 시간 합계
- `http_request_duration_seconds_count` — 전체 요청 수

**안티패턴**: 버킷 범위를 너무 넓거나 좁게 설정하는 것. 실제 트래픽 분포를 먼저 파악하고 버킷을 설계해야 한다.

---

### Summary — 클라이언트 사이드 백분위수

Summary는 Histogram과 비슷하지만, **클라이언트에서 직접 백분위수를 계산**해서 내보낸다.

집계 없이 단일 인스턴스의 정밀한 백분위수가 필요할 때 사용한다.

```java
DistributionSummary summary = DistributionSummary.builder("response.body.size")
    .description("Response body size distribution")
    .baseUnit("bytes")
    .percentilePrecision(2)
    .publishPercentiles(0.5, 0.95, 0.99)
    .register(registry);
```

**Histogram vs Summary 선택 기준**:

| 기준 | Histogram | Summary |
|------|-----------|---------|
| 집계 가능 여부 | 가능 (여러 인스턴스) | 불가 (단일 인스턴스) |
| 백분위수 정확도 | 버킷 범위에 따라 근사 | 정확 |
| 서버 부하 | 낮음 | 높음 |
| 권장 상황 | 분산 시스템, 다중 인스턴스 | 단일 인스턴스, 정밀도 중요 |

분산 환경에서는 **Histogram이 기본값**이다. 여러 인스턴스의 응답 시간을 합산해 전체 99번째 백분위수를 구하려면 Histogram만 가능하다.

---

## 무엇을 측정할 것인가: RED와 USE 방법론

### RED 방법론 — 서비스 관점

RED는 **사용자가 경험하는 서비스 품질**을 세 가지 숫자로 요약한다.

- **Rate**: 초당 요청 수 (처리량)
- **Error**: 실패율 (품질)
- **Duration**: 응답 시간 (속도)

마이크로서비스, API 서버, 사용자 대면 서비스에 적합하다.

```promql
# Rate: 초당 요청 수
rate(http_requests_total[5m])

# Error Rate: 에러 비율
rate(http_requests_total{status=~"5.."}[5m])
  / rate(http_requests_total[5m])

# Duration: p99 응답 시간
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket[5m])
)
```

RED 하나만 이상해도 사용자는 이미 문제를 느끼고 있다. 가장 먼저 대시보드에 올려야 할 지표다.

---

### USE 방법론 — 리소스 관점

USE는 **시스템 리소스의 포화도와 오류**를 진단한다.

- **Utilization**: 리소스 사용률 (CPU, 메모리, 디스크 I/O)
- **Saturation**: 대기열이나 처리 지연 (큐 깊이, 컨텍스트 스위치)
- **Errors**: 하드웨어/OS 수준 오류

인프라 레이어, 데이터베이스, 네트워크 장비 모니터링에 적합하다.

```promql
# Utilization: CPU 사용률
100 - (avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
) * 100)

# Saturation: 실행 큐 대기 프로세스 수
node_load1

# Errors: 디스크 I/O 에러
rate(node_disk_io_time_seconds_total[5m])
```

USE와 RED는 상호 보완적이다. RED로 증상을 발견하고, USE로 원인을 찾는다.

---

## Prometheus: 스크레이핑 기반 메트릭 수집

### 아키텍처 이해

Prometheus는 **풀(pull) 방식**으로 메트릭을 수집한다.

애플리케이션은 `/metrics` 엔드포인트를 열어두고, Prometheus가 주기적으로 가져간다. 이 방식을 **스크레이핑(scraping)**이라 한다.

```
[App :8080/metrics] <-- scrape -- [Prometheus] --> [Grafana]
[App :8081/metrics] <--/
[Node Exporter]     <--/
```

푸시 방식과 달리 풀 방식은 애플리케이션이 Prometheus 주소를 알 필요 없다. 서비스 디스커버리와 결합하면 새 인스턴스가 자동으로 수집 대상에 포함된다.

---

### prometheus.yml 설정

```yaml
# prometheus.yml
global:
  scrape_interval: 15s       # 기본 스크레이핑 주기
  evaluation_interval: 15s   # 알림 규칙 평가 주기
  scrape_timeout: 10s

# 알림 매니저 연결
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# 알림 규칙 파일 경로
rule_files:
  - "rules/*.yml"

# 스크레이핑 대상 설정
scrape_configs:
  # Prometheus 자신 모니터링
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Spring Boot 애플리케이션
  - job_name: "spring-api"
    metrics_path: "/actuator/prometheus"
    scrape_interval: 10s
    static_configs:
      - targets:
          - "api-server-1:8080"
          - "api-server-2:8080"
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: "([^:]+)(:[0-9]+)?"
        replacement: "$1"

  # Node Exporter (서버 시스템 메트릭)
  - job_name: "node"
    static_configs:
      - targets:
          - "server-1:9100"
          - "server-2:9100"
    relabel_configs:
      - source_labels: [__address__]
        regex: "([^:]+):.*"
        target_label: hostname
        replacement: "$1"

  # Kubernetes 서비스 디스커버리 예시
  - job_name: "kubernetes-pods"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: "(.+)"
```

`relabel_configs`는 강력하지만 복잡해질 수 있다. 처음에는 `static_configs`로 시작해서 필요할 때 동적 디스커버리로 전환하는 것이 좋다.

---

### Spring Boot Actuator로 메트릭 노출

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  endpoint:
    prometheus:
      enabled: true
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${DEPLOY_ENV:local}
    distribution:
      percentiles-histogram:
        http.server.requests: true
      slo:
        http.server.requests: 100ms, 500ms, 1000ms
```

`management.metrics.tags`에 `application`과 `environment`를 추가하면 모든 메트릭에 공통 레이블이 붙는다. 여러 서비스를 한 Prometheus에서 수집할 때 필수다.

---

## PromQL: 메트릭 쿼리 언어

### 기본 문법

PromQL의 핵심은 **시계열 선택자**와 **함수 조합**이다.

```promql
# 메트릭 이름으로 선택
http_requests_total

# 레이블 필터링
http_requests_total{job="api", status="200"}

# 정규식 매칭 (=~: 매칭, !~: 불일치)
http_requests_total{status=~"2.."}
http_requests_total{status!~"4..|5.."}

# 범위 벡터 (5분간의 데이터)
http_requests_total[5m]
```

---

### 자주 쓰는 PromQL 패턴

**요청 처리량 (초당 요청 수)**

```promql
# 전체 초당 요청 수
sum(rate(http_requests_total[5m]))

# 서비스별, 엔드포인트별 분리
sum by (handler, method) (
  rate(http_requests_total[5m])
)
```

**에러율**

```promql
# 5xx 에러 비율 (%)
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
) * 100

# 특정 서비스의 에러율
sum(rate(http_requests_total{job="order-api", status=~"5.."}[5m]))
/ sum(rate(http_requests_total{job="order-api"}[5m]))
```

**응답 시간 백분위수**

```promql
# p99 응답 시간 (Histogram 사용)
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{job="api"}[5m])
  )
)

# 서비스별 p95
histogram_quantile(0.95,
  sum by (le, job) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

**JVM 메트릭**

```promql
# Heap 사용률
jvm_memory_used_bytes{area="heap"}
/ jvm_memory_max_bytes{area="heap"} * 100

# GC 일시 정지 시간 (초당)
rate(jvm_gc_pause_seconds_sum[5m])

# 스레드 풀 사용률
executor_active_threads / executor_pool_max_threads
```

**인프라 메트릭**

```promql
# CPU 사용률 (인스턴스별)
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)

# 메모리 사용률
(
  1 - (
    node_memory_MemAvailable_bytes
    / node_memory_MemTotal_bytes
  )
) * 100

# 디스크 사용률
(
  1 - (
    node_filesystem_avail_bytes{mountpoint="/"}
    / node_filesystem_size_bytes{mountpoint="/"}
  )
) * 100
```

---

### PromQL 안티패턴

**안티패턴 1: `rate()` 없이 Counter 사용**

```promql
# 잘못된 예시 — Counter 값 자체는 의미 없음
http_requests_total

# 올바른 예시
rate(http_requests_total[5m])
```

**안티패턴 2: 고카디널리티 레이블**

```promql
# 잘못된 예시 — user_id를 레이블로 사용하면 시계열 폭발
http_requests_total{user_id="12345678"}

# 올바른 예시 — 집계 단위 레이블만 사용
http_requests_total{region="seoul", tier="premium"}
```

레이블 값의 종류가 수천 개 이상이 되면 Prometheus 메모리가 폭발한다. URL 경로, 사용자 ID, 요청 ID는 레이블에 절대 넣으면 안 된다.

**안티패턴 3: 너무 짧은 `rate()` 범위**

```promql
# 잘못된 예시 — 스크레이핑 주기(15s)보다 짧은 범위
rate(http_requests_total[10s])

# 올바른 예시 — 스크레이핑 주기의 4배 이상
rate(http_requests_total[1m])
```

---

## Grafana 대시보드 설계

### 대시보드 계층 구조

좋은 대시보드는 **하향식(top-down)**으로 설계된다.

1. **Overview 대시보드**: 전체 서비스 RED 지표를 한 화면에 표시
2. **Service 대시보드**: 특정 서비스의 상세 지표
3. **Instance 대시보드**: 특정 인스턴스의 시스템 리소스

처음부터 모든 것을 한 대시보드에 넣으면 노이즈가 너무 많다. 계층을 나눠서 드릴다운하는 구조가 효과적이다.

---

### Grafana 대시보드 JSON 구조

```json
{
  "title": "API Service Overview",
  "uid": "api-overview",
  "tags": ["api", "overview"],
  "time": { "from": "now-1h", "to": "now" },
  "refresh": "30s",
  "templating": {
    "list": [
      {
        "name": "job",
        "type": "query",
        "query": "label_values(http_requests_total, job)",
        "multi": false
      },
      {
        "name": "instance",
        "type": "query",
        "query": "label_values(http_requests_total{job=\"$job\"}, instance)",
        "multi": true
      }
    ]
  },
  "panels": [
    {
      "title": "Request Rate",
      "type": "stat",
      "gridPos": { "x": 0, "y": 0, "w": 6, "h": 4 },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{job=\"$job\"}[5m]))",
          "legendFormat": "req/s"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "gridPos": { "x": 6, "y": 0, "w": 6, "h": 4 },
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "thresholds": {
            "steps": [
              { "value": 0, "color": "green" },
              { "value": 0.01, "color": "yellow" },
              { "value": 0.05, "color": "red" }
            ]
          }
        }
      },
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{job=\"$job\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{job=\"$job\"}[5m]))",
          "legendFormat": "error rate"
        }
      ]
    }
  ]
}
```

`templating`의 변수 기능을 적극 활용하자. 대시보드 상단에서 `job`이나 `instance`를 선택하면 모든 패널이 연동되는 동적 대시보드를 만들 수 있다.

---

### 시각화 유형 선택 가이드

| 측정 대상 | 권장 패널 유형 |
|-----------|---------------|
| 현재 값 (에러율, CPU) | Stat, Gauge |
| 시간에 따른 변화 | Time series |
| 여러 항목 비교 | Bar chart |
| 응답 시간 분포 | Heatmap |
| 상태 개요 | State timeline |
| SLA 달성 여부 | Stat + thresholds |

Heatmap은 Histogram 메트릭과 완벽하게 맞아떨어진다. 시간축 X, 응답 시간 버킷 Y, 요청 수 색상으로 응답 시간 분포의 변화를 직관적으로 파악할 수 있다.

---

## 알림(Alert) 규칙 설계

### Prometheus 알림 규칙

```yaml
# rules/api-alerts.yml
groups:
  - name: api.rules
    interval: 30s
    rules:
      # 높은 에러율
      - alert: HighErrorRate
        expr: |
          (
            sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
            / sum by (job) (rate(http_requests_total[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: |
            Error rate is {{ $value | humanizePercentage }}
            for job {{ $labels.job }}.
            (threshold: 5%)
          runbook_url: "https://wiki/runbooks/high-error-rate"

      # 느린 응답 시간
      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.99,
            sum by (le, job) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          ) > 1.0
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "Slow p99 response time on {{ $labels.job }}"
          description: |
            p99 latency is {{ $value | humanizeDuration }}
            for job {{ $labels.job }}.

      # 서비스 다운
      - alert: ServiceDown
        expr: up{job=~"api.*"} == 0
        for: 1m
        labels:
          severity: critical
          team: ops
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Instance {{ $labels.instance }} has been down for 1 minute."

      # CPU 과부하
      - alert: HighCpuUsage
        expr: |
          100 - (
            avg by (instance) (
              rate(node_cpu_seconds_total{mode="idle"}[5m])
            ) * 100
          ) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}% for 10+ minutes."

      # 디스크 공간 부족
      - alert: DiskSpaceLow
        expr: |
          (
            1 - (
              node_filesystem_avail_bytes{mountpoint="/"}
              / node_filesystem_size_bytes{mountpoint="/"}
            )
          ) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk usage is {{ $value | humanize }}% on {{ $labels.mountpoint }}."
```

`for` 필드가 핵심이다. 조건이 `for` 시간 동안 지속될 때만 알림이 발화된다. 순간적인 스파이크로 인한 오알림을 방지한다.

---

### Alertmanager 설정

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

route:
  receiver: "default"
  group_by: ["alertname", "job"]
  group_wait: 30s       # 첫 알림 전 대기 (그룹화)
  group_interval: 5m    # 같은 그룹 알림 간격
  repeat_interval: 4h   # 동일 알림 반복 간격

  routes:
    # Critical 알림은 PagerDuty
    - match:
        severity: critical
      receiver: "pagerduty-critical"
      continue: true

    # 특정 팀 알림은 해당 Slack 채널로
    - match:
        team: backend
      receiver: "slack-backend"

receivers:
  - name: "default"
    slack_configs:
      - channel: "#alerts-general"
        title: "[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}"
        text: |
          {{ range .Alerts }}
          *Summary:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook_url }}
          {{ end }}

  - name: "pagerduty-critical"
    pagerduty_configs:
      - service_key: "YOUR_PAGERDUTY_KEY"

  - name: "slack-backend"
    slack_configs:
      - channel: "#alerts-backend"
        send_resolved: true

inhibit_rules:
  # ServiceDown이 발화 중이면 다른 알림 억제
  - source_match:
      alertname: "ServiceDown"
    target_match_re:
      alertname: "HighErrorRate|SlowResponseTime"
    equal: ["job"]
```

`inhibit_rules`는 자주 간과되는 기능이다. 서비스가 완전히 다운된 상황에서 에러율 알림이 동시에 쏟아지면 노이즈가 너무 많아진다. 상위 원인 알림이 하위 증상 알림을 억제하도록 설정하자.

---

### 알림 설계 안티패턴

**안티패턴 1: 임계값만으로 알림 설정**

CPU 85% 이상이면 무조건 알림을 보내는 규칙은 새벽 2시에 배치 작업이 돌 때마다 울린다. **기준선(baseline)** 대비 이상 탐지로 보완해야 한다.

```promql
# 단순 임계값 — 오알림 많음
node_cpu_usage > 0.85

# 평균 대비 3시그마 이상 이탈 — 더 정확
node_cpu_usage > (
  avg_over_time(node_cpu_usage[1w]) + 3 * stddev_over_time(node_cpu_usage[1w])
)
```

**안티패턴 2: `for` 없이 순간 조건으로 알림**

네트워크 지연이나 일시적 스파이크로 인한 오알림이 반복되면 팀이 알림 피로(alert fatigue)에 빠진다. 최소 2분 이상의 `for` 조건을 걸자.

**안티패턴 3: runbook_url 없는 알림**

알림을 받은 사람이 무엇을 해야 하는지 모르는 알림은 쓸모없다. 모든 알림에는 `runbook_url` 어노테이션을 달아야 한다.

---

## 운영에서 얻은 교훈

메트릭 시스템을 구축하면서 가장 많이 저지르는 실수는 **측정 자체가 목표**가 되는 것이다.

100개의 대시보드보다 10개의 행동 가능한 알림이 낫다. "이 알림을 받으면 무엇을 해야 하는가"라는 질문에 답할 수 없다면, 그 알림은 노이즈다.

SLO(Service Level Objective)를 먼저 정의하고, 그것을 측정하는 메트릭을 설계하는 순서가 맞다. 메트릭을 먼저 수집하고 나중에 의미를 찾으려 하면 중요한 것을 놓친다.

---

## 참고 자료

- [Prometheus Documentation — Data Model & Query Language](https://prometheus.io/docs/concepts/data_model/)
- [Grafana Documentation — Dashboard Best Practices](https://grafana.com/docs/grafana/latest/dashboards/best-practices/)
- [Tom Wilkie — The RED Method: How to Instrument Your Services](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [Brendan Gregg — USE Method](https://www.brendangregg.com/usemethod.html)
- [Micrometer Documentation — Spring Boot Integration](https://micrometer.io/docs/ref/spring/boot2)
- [Rob Ewaschuk — My Philosophy on Alerting (Google SRE)](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q)
