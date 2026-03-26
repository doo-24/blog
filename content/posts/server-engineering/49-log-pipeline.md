---
title: "[로그, 모니터링, 관측 가능성] 2편 — 로그 수집과 저장: 분산 환경의 로그 파이프라인"
date: 2026-03-17T19:04:00+09:00
draft: false
tags: ["ELK", "Loki", "Fluentd", "로그 파이프라인", "서버"]
series: ["로그, 모니터링, 관측 가능성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 7
summary: "ELK 스택(Elasticsearch, Logstash, Kibana) 구조, Loki+Grafana 경량 대안, Fluentd/Fluent Bit 로그 수집기 설정, 로그 보존 정책과 인덱싱 전략, 비용 최적화까지"
---

서비스 규모가 커질수록 로그는 단순한 디버그 도구를 넘어 운영의 핵심 자산이 된다.

인스턴스 10대가 동시에 로그를 쏟아낼 때, 어느 서버의 어느 프로세스가 문제를 일으켰는지 파악하려면 로그를 한곳에 모아 검색하고 분석할 수 있는 파이프라인이 반드시 필요하다.

이 글에서는 ELK 스택부터 Loki, Fluentd까지 실무에서 사용하는 로그 파이프라인 구성을 구체적인 설정 파일과 함께 살펴본다.

---

## ELK 스택: 분산 로그의 표준 아키텍처

ELK는 **Elasticsearch**, **Logstash**, **Kibana** 세 컴포넌트의 머릿글자다.

Elasticsearch는 로그를 저장하고 검색하는 분산 검색 엔진이다. Logstash는 다양한 소스에서 로그를 수집해 변환한 뒤 Elasticsearch로 전달한다. Kibana는 저장된 로그를 시각화하고 대시보드로 만드는 UI 레이어다.

현재는 Beats(경량 수집기)가 추가되면서 **Elastic Stack**이라고도 불린다.

### Elasticsearch 클러스터 구조

Elasticsearch는 노드들의 클러스터로 동작한다. 노드 유형은 역할에 따라 분리한다.

```yaml
# elasticsearch.yml
cluster.name: production-logs
node.name: es-node-01

# 역할 분리 (프로덕션 권장)
node.roles: [ data_hot, ingest ]

# 데이터 경로
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# 네트워크
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

# 클러스터 초기화 (최초 1회)
cluster.initial_master_nodes: ["es-master-01", "es-master-02", "es-master-03"]

# JVM 힙 설정은 jvm.options에서
# -Xms4g
# -Xmx4g
```

프로덕션에서는 마스터 노드 3대, 데이터 노드 3대 이상으로 역할을 분리한다.

마스터 노드가 데이터까지 처리하면 클러스터 상태 관리가 불안정해진다.

### ILM: 인덱스 생명주기 관리

로그 인덱스는 시간이 지날수록 쌓인다. ILM(Index Lifecycle Management)으로 Hot-Warm-Cold-Delete 정책을 자동화한다.

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_primary_shard_size": "50gb"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

Hot 단계에서는 빠른 읽기/쓰기를 위해 SSD 노드에 배치한다. Warm 단계부터는 HDD 노드로 이동시켜 비용을 낮춘다.

### 인덱스 템플릿과 매핑

로그 필드 타입을 미리 정의해두지 않으면 Elasticsearch가 동적으로 추론한다. 이는 예상치 못한 매핑 충돌을 일으킨다.

```json
PUT _index_template/app-logs-template
{
  "index_patterns": ["app-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "app-logs"
    },
    "mappings": {
      "properties": {
        "@timestamp":   { "type": "date" },
        "level":        { "type": "keyword" },
        "service":      { "type": "keyword" },
        "trace_id":     { "type": "keyword" },
        "message":      { "type": "text", "analyzer": "standard" },
        "duration_ms":  { "type": "long" },
        "status_code":  { "type": "short" },
        "host": {
          "properties": {
            "name":     { "type": "keyword" },
            "ip":       { "type": "ip" }
          }
        }
      }
    }
  }
}
```

`message` 필드는 `text`로, 검색 필터링에 사용하는 필드는 `keyword`로 정의한다.

`keyword` 타입은 전체 값으로만 매칭되어 집계에 적합하다.

---

## Logstash: 변환의 중심

Logstash는 Input → Filter → Output 파이프라인으로 동작한다.

### 기본 파이프라인 설정

```ruby
# /etc/logstash/conf.d/app-logs.conf

input {
  beats {
    port => 5044
    ssl  => true
    ssl_certificate_authorities => ["/etc/logstash/certs/ca.crt"]
    ssl_certificate             => "/etc/logstash/certs/logstash.crt"
    ssl_key                     => "/etc/logstash/certs/logstash.key"
  }

  kafka {
    bootstrap_servers => "kafka-01:9092,kafka-02:9092,kafka-03:9092"
    topics            => ["app-logs", "access-logs"]
    group_id          => "logstash-consumers"
    codec             => "json"
    consumer_threads  => 4
  }
}

filter {
  # JSON 파싱 (이미 구조화된 로그)
  if [message] =~ /^\{/ {
    json {
      source => "message"
      target => "parsed"
    }
    mutate {
      rename => { "[parsed][level]"    => "level"    }
      rename => { "[parsed][service]"  => "service"  }
      rename => { "[parsed][trace_id]" => "trace_id" }
      rename => { "[parsed][msg]"      => "message"  }
      remove_field => ["parsed"]
    }
  }

  # 날짜 파싱
  date {
    match   => ["timestamp", "ISO8601"]
    target  => "@timestamp"
    remove_field => ["timestamp"]
  }

  # 레벨 정규화
  mutate {
    uppercase => ["level"]
  }

  # 민감 정보 마스킹
  mutate {
    gsub => [
      "message", "password=[^\s&]+",    "password=***",
      "message", "token=[A-Za-z0-9._-]+", "token=***"
    ]
  }

  # 느린 요청 태깅
  if [duration_ms] and [duration_ms] > 1000 {
    mutate {
      add_tag => ["slow_request"]
    }
  }
}

output {
  elasticsearch {
    hosts           => ["https://es-01:9200", "https://es-02:9200"]
    index           => "app-logs-%{+YYYY.MM.dd}"
    user            => "${ES_USER}"
    password        => "${ES_PASSWORD}"
    ssl             => true
    cacert          => "/etc/logstash/certs/ca.crt"
    ilm_enabled     => true
    ilm_rollover_alias => "app-logs"
    ilm_policy      => "logs-policy"
  }

  # 에러 레벨은 별도 인덱스로
  if [level] == "ERROR" {
    elasticsearch {
      hosts => ["https://es-01:9200"]
      index => "error-logs-%{+YYYY.MM.dd}"
      user  => "${ES_USER}"
      password => "${ES_PASSWORD}"
    }
  }
}
```

### Logstash의 약점

Logstash는 JVM 기반이라 메모리를 많이 쓴다. 최소 2GB 힙을 요구하며, 파이프라인 복잡도가 올라가면 4GB도 부족하다.

경량 환경에서는 Logstash 대신 Fluentd나 Fluent Bit이 더 적합하다.

---

## Loki + Grafana: 경량 대안

Grafana Labs가 만든 Loki는 Prometheus의 로그 버전이라고 볼 수 있다.

Elasticsearch처럼 로그 내용을 전부 인덱싱하지 않는다. 레이블(label)만 인덱싱하고, 로그 본문은 압축해서 객체 스토리지에 저장한다.

이 구조 덕분에 저장 비용이 Elasticsearch 대비 10분의 1 수준이다.

### Loki 설계 철학

Loki는 **"가능하면 인덱싱하지 마라"** 는 원칙을 따른다.

레이블은 낮은 카디널리티(cardinality) 값만 사용한다. `user_id`나 `request_id` 같은 고유값을 레이블로 쓰면 인덱스가 폭발적으로 커진다.

올바른 레이블: `service`, `env`, `level`, `region` (값의 종류가 수십 개 이하)

### Loki 설정

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: memberlist
      replication_factor: 1
  chunk_idle_period:   5m
  chunk_retain_period: 30s
  wal:
    enabled: true
    dir:     /loki/wal

schema_config:
  configs:
    - from: 2024-01-01
      store:      boltdb-shipper
      object_store: s3
      schema:     v12
      index:
        prefix: loki_index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location:         /loki/cache
    shared_store:           s3
  aws:
    s3:            s3://ap-northeast-2/my-loki-logs
    region:        ap-northeast-2
    access_key_id:     ${AWS_ACCESS_KEY_ID}
    secret_access_key: ${AWS_SECRET_ACCESS_KEY}

compactor:
  working_directory:   /loki/compactor
  shared_store:        s3
  compaction_interval: 10m
  retention_enabled:   true
  retention_delete_delay:  2h

limits_config:
  retention_period:         720h  # 30일
  ingestion_rate_mb:        64
  ingestion_burst_size_mb:  128
  max_query_series:         500

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled:   true
        max_size_mb: 100
```

### LogQL: Loki 쿼리 언어

LogQL은 Prometheus의 PromQL과 유사한 구조다.

```logql
# 기본 로그 검색: 레이블 필터 + 텍스트 필터
{service="api-server", env="production"} |= "ERROR"

# JSON 파싱 후 필드 필터링
{service="api-server"}
  | json
  | duration_ms > 500
  | line_format "{{.level}} {{.message}} ({{.duration_ms}}ms)"

# 에러 비율 계산 (메트릭 쿼리)
sum(rate({service="api-server"} |= "ERROR" [5m])) by (service)
/
sum(rate({service="api-server"} [5m])) by (service)

# 레이턴시 p99 (파싱 필요)
quantile_over_time(0.99,
  {service="api-server"}
  | json
  | unwrap duration_ms [5m]
) by (service)
```

Grafana에서 Loki 데이터소스를 추가하면 위 쿼리를 대시보드와 알람에 바로 사용할 수 있다.

### Loki의 한계

전문 검색(full-text search)이 Elasticsearch보다 느리다. 레이블 없이 텍스트만으로 검색하면 전체 청크를 스캔한다.

대규모 텍스트 분석이 필요한 경우는 Elasticsearch가 더 적합하다. 저장 비용이 중요하고 레이블 기반 필터링이 충분하다면 Loki가 더 좋은 선택이다.

---

## Fluentd / Fluent Bit: 로그 수집기

### Fluentd vs Fluent Bit

| 항목 | Fluentd | Fluent Bit |
|------|---------|------------|
| 언어 | Ruby + C | C |
| 메모리 | ~40MB | ~1MB |
| 플러그인 | 1000+ | 제한적 |
| 용도 | 중앙 집계 레이어 | 엣지 수집기 |

일반적인 아키텍처는 **Fluent Bit(각 노드)** → **Fluentd(중앙)** → **스토리지** 구조다.

Kubernetes 환경에서는 각 노드에 Fluent Bit DaemonSet을 배포하고, 중앙 Fluentd 집계 레이어를 거쳐 Elasticsearch나 Loki로 보낸다.

### Fluent Bit 설정 (Kubernetes DaemonSet)

```ini
# fluent-bit.conf
[SERVICE]
    Flush         5
    Daemon        Off
    Log_Level     info
    Parsers_File  parsers.conf
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020

[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    Parser            cri
    DB                /var/log/flb_kube.db
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
    Refresh_Interval  10

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Kube_Tag_Prefix     kube.var.log.containers.
    Merge_Log           On
    Merge_Log_Key       log_processed
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[FILTER]
    Name    modify
    Match   kube.*
    Add     cluster production-k8s
    Add     region ap-northeast-2

[FILTER]
    Name   grep
    Match  kube.*
    # 시스템 네임스페이스 제외
    Exclude $kubernetes['namespace_name'] ^(kube-system|monitoring)$

[OUTPUT]
    Name          forward
    Match         *
    Host          fluentd-aggregator.logging.svc.cluster.local
    Port          24224
    Retry_Limit   False
    tls           On
    tls.verify    On
    tls.ca_file   /fluent-bit/certs/ca.crt
    tls.crt_file  /fluent-bit/certs/client.crt
    tls.key_file  /fluent-bit/certs/client.key
```

```ini
# parsers.conf
[PARSER]
    Name        cri
    Format      regex
    Regex       ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<log>.*)$
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L%z

[PARSER]
    Name        json
    Format      json
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%LZ
```

### Fluentd 집계 레이어 설정

```ruby
# fluentd-aggregator.conf

<source>
  @type forward
  port 24224
  bind 0.0.0.0
  <transport tls>
    ca_path        /fluentd/certs/ca.crt
    cert_path      /fluentd/certs/server.crt
    private_key_path /fluentd/certs/server.key
    client_cert_auth true
  </transport>
</source>

# 버퍼 큐를 통한 안정적인 전달
<match kube.**>
  @type copy

  <store>
    @type elasticsearch
    host        es-01.logging.svc.cluster.local
    port        9200
    scheme      https
    user        "#{ENV['ES_USER']}"
    password    "#{ENV['ES_PASSWORD']}"
    index_name  app-logs
    logstash_format true
    logstash_prefix app-logs

    <buffer tag,time>
      @type           file
      path            /fluentd/buffer/es
      timekey         1h
      timekey_wait    10m
      chunk_limit_size 8MB
      total_limit_size 2GB
      overflow_action  block
      retry_max_times  10
      retry_type       exponential_backoff
    </buffer>
  </store>

  # 에러 레벨 별도 처리
  <store>
    @type           rewrite_tag_filter
    <rule>
      key     level
      pattern /^(ERROR|FATAL)$/
      tag     error.${tag}
    </rule>
  </store>
</match>

<match error.**>
  @type slack
  webhook_url "#{ENV['SLACK_WEBHOOK_URL']}"
  channel     "#alerts"
  username    "FluentD"
  icon_emoji  ":rotating_light:"
  message     "[%s] %s: %s"
  message_keys level,kubernetes.pod_name,message
  flush_interval 30s
</match>
```

### 안티패턴: 버퍼 없이 직접 전달

```ruby
# 나쁜 예 - 버퍼 없이 직접 전달
<match **>
  @type elasticsearch
  host es-01
  port 9200
  # buffer 설정 없음!
</match>
```

Elasticsearch가 잠시 다운되면 이 구간의 로그가 유실된다.

항상 `<buffer>` 블록을 설정하고 `overflow_action block`으로 백프레셔를 활성화해야 한다.

---

## 로그 보존 정책과 비용 최적화

### 보존 정책 설계

로그를 무한히 저장하면 비용이 선형으로 늘어난다. 데이터 성격에 따라 보존 기간을 차등화한다.

| 로그 유형 | Hot | Warm | Cold | 삭제 |
|-----------|-----|------|------|------|
| 애플리케이션 로그 | 3일 | 14일 | 30일 | 90일 |
| 보안/감사 로그 | 7일 | 30일 | 180일 | 1년+ |
| 액세스 로그 | 1일 | 7일 | 14일 | 30일 |
| 디버그 로그 | 1일 | 3일 | — | 7일 |

보안 규정(PCI DSS, HIPAA 등)이 있는 서비스는 감사 로그 보존 기간이 법적으로 정해져 있다.

### 샘플링으로 볼륨 줄이기

모든 로그를 저장할 필요는 없다. 정상 요청의 DEBUG 로그는 1%만 샘플링해도 충분하다.

```python
# 애플리케이션 레벨 샘플링 (Python)
import random
import logging

class SamplingHandler(logging.Handler):
    def __init__(self, handler, sample_rate=0.01):
        super().__init__()
        self.handler  = handler
        self.sample_rate = sample_rate

    def emit(self, record):
        # ERROR 이상은 항상 기록
        if record.levelno >= logging.ERROR:
            self.handler.emit(record)
            return
        # DEBUG/INFO는 sample_rate 확률로만 기록
        if random.random() < self.sample_rate:
            self.handler.emit(record)

# 설정
logger  = logging.getLogger("app")
handler = logging.StreamHandler()
logger.addHandler(SamplingHandler(handler, sample_rate=0.01))
```

Fluentd에서도 샘플링 플러그인을 사용할 수 있다.

```ruby
<filter kube.**>
  @type sampling
  sample_unit minute
  sample_threshold 100  # 분당 100건 초과 시
  sample_rate 10        # 10%만 통과
</filter>
```

### 압축과 티어링

Elasticsearch의 Cold 단계에서는 `_freeze` API로 인덱스를 읽기 전용으로 전환한다.

S3나 GCS 기반 Searchable Snapshot을 활용하면 로컬 디스크 없이 객체 스토리지에서 직접 검색할 수 있다. 비용은 로컬 저장의 10% 수준이다.

```json
POST app-logs-2024.01.01/_snapshot/s3-repository/snapshot-2024-01-01
{
  "indices":             "app-logs-2024.01.01",
  "include_global_state": false
}

# Searchable Snapshot으로 마운트
POST /_snapshot/s3-repository/snapshot-2024-01-01/_mount
{
  "index":         "app-logs-2024.01.01",
  "renamed_index": "restored-app-logs-2024.01.01",
  "storage":       "shared_cache"
}
```

### 필드 최적화: 저장 크기 줄이기

불필요한 필드를 제거하면 저장 공간을 크게 줄일 수 있다.

```ruby
# Fluentd: 불필요 필드 제거
<filter kube.**>
  @type record_transformer
  remove_keys docker,stream,logtag,kubernetes.labels.pod-template-hash
  enable_ruby true
  <record>
    # 긴 스택 트레이스는 처음 2000자만 유지
    message ${record["message"].to_s[0, 2000]}
  </record>
</filter>
```

Elasticsearch에서는 `_source` 압축을 활성화한다.

```json
PUT app-logs-2024.01.01/_settings
{
  "index.codec": "best_compression"
}
```

`best_compression`은 기본 LZ4 대신 DEFLATE를 사용해 저장 공간을 20-30% 더 절약한다.

---

## Kibana로 대시보드 구성

### KQL 기본 쿼리

Kibana Query Language는 Elasticsearch를 직접 쿼리하는 것보다 훨씬 간결하다.

```kql
# 에러 레벨 로그
level: ERROR

# 특정 서비스의 느린 요청
service: "api-server" and duration_ms > 1000

# 지난 1시간 내 특정 사용자 관련 로그
@timestamp > now-1h and message: "user_id=12345"

# 와일드카드 검색
message: "connection*refused"

# NOT 제외
level: * and not level: DEBUG
```

### Discover에서 인덱스 패턴 생성

인덱스 패턴 `app-logs-*`을 만들면 날짜별로 분산된 인덱스를 하나의 뷰로 통합할 수 있다.

`@timestamp`를 기본 시간 필드로 설정하면 시간 범위 필터가 자동 적용된다.

---

## 모니터링: 파이프라인 자체를 관찰하기

로그 파이프라인이 정상 동작하는지도 모니터링해야 한다.

Fluent Bit은 `/api/v1/metrics/prometheus` 엔드포인트로 Prometheus 메트릭을 노출한다.

```yaml
# Prometheus scrape 설정
scrape_configs:
  - job_name: fluent-bit
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        action:        keep
        regex:         fluent-bit
    metrics_path: /api/v1/metrics/prometheus
    scrape_interval: 30s
```

주요 확인 메트릭:

- `fluentbit_input_records_total`: 수집된 레코드 수
- `fluentbit_output_retries_total`: 재시도 횟수 (급증하면 백엔드 문제)
- `fluentbit_output_errors_total`: 전달 실패 (알람 설정 필수)

Elasticsearch 클러스터 상태는 `/_cluster/health` API로 확인한다.

```bash
curl -s https://es-01:9200/_cluster/health | jq '{
  status,
  number_of_nodes,
  active_shards,
  unassigned_shards,
  active_primary_shards
}'
```

`unassigned_shards`가 0이 아니면 클러스터가 정상이 아닌 것이다.

---

## 안티패턴 정리

**1. 모든 것을 ERROR로 로깅하기**

의미 없는 WARN/ERROR가 넘쳐나면 실제 에러를 놓친다. 레벨 기준을 팀 내에서 명확히 정의해야 한다.

**2. 구조화되지 않은 로그**

```
# 나쁜 예
log.error("User 12345 failed to login from 192.168.1.1 at 2024-01-15")

# 좋은 예
log.error("login_failed", user_id=12345, ip="192.168.1.1", reason="invalid_password")
```

구조화된 JSON 로그는 파싱 없이 바로 필드로 인덱싱된다.

**3. 로그에 민감 정보 포함**

비밀번호, 토큰, 개인정보가 로그에 들어가는 순간 보안 사고로 이어질 수 있다. Logstash/Fluentd 필터에서 마스킹하는 것은 최후 방어선이고, 애플리케이션 코드에서 처음부터 제외해야 한다.

**4. 단일 Elasticsearch 노드**

노드가 하나면 재시작 시 서비스가 중단된다. 최소 3노드 클러스터를 구성해야 한다.

**5. 인덱스 샤드 과다 생성**

날짜별 인덱스를 수백 개 생성하면 클러스터 오버헤드가 커진다. ILM의 rollover 정책으로 샤드 크기를 기준으로 인덱스를 나눠야 한다.

---

## 참고 자료

- [Elasticsearch 공식 문서 — ILM: Manage the index lifecycle](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [Grafana Loki 공식 문서 — Best practices](https://grafana.com/docs/loki/latest/best-practices/)
- [Fluent Bit 공식 문서 — Kubernetes Filter](https://docs.fluentbit.io/manual/pipeline/filters/kubernetes)
- [Fluentd 공식 문서 — Buffer Section](https://docs.fluentd.org/configuration/buffer-section)
- [Elastic Blog — Searchable Snapshots](https://www.elastic.co/blog/introducing-elasticsearch-searchable-snapshots)
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
