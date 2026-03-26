---
title: "[데이터베이스] 7편 — 특수 목적 DB: 검색, 시계열, 벡터"
date: 2026-03-17T23:03:00+09:00
draft: false
tags: ["Elasticsearch", "시계열 DB", "벡터 DB", "검색", "데이터베이스", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "Elasticsearch 아키텍처와 역인덱스 원리, 검색 쿼리 설계(형태소 분석, 스코어링, 자동완성), 시계열 DB(InfluxDB, TimescaleDB), 벡터 DB(Pinecone, pgvector), 용도별 DB 선택 기준까지"
---

MySQL이나 PostgreSQL로 모든 문제를 해결하려는 시도는, 마치 스크루드라이버로 망치질을 하는 것과 같다.

관계형 DB는 탁월한 범용 도구지만, "이 상품명에 '삼성 갤럭시'가 포함된 문서를 관련도 순으로 보여줘", "지난 1시간 동안 CPU 사용률의 평균을 1분 단위로 집계해줘", "이 문장과 의미적으로 가장 유사한 문서 10개를 찾아줘" 같은 요구사항 앞에서는 맥없이 무너진다.

검색, 시계열, 벡터 — 이 세 가지 도메인은 각각 수십 년의 연구와 수억 달러의 엔지니어링이 녹아든 전용 DB가 존재한다. 이 글에서는 그 원리를 깊게 파고들어, 언제 어떤 도구를 선택해야 하는지 확실한 기준을 세운다.

---

## 1. Elasticsearch 아키텍처: 분산 검색 엔진의 내부

### 1.1 클러스터, 노드, 인덱스, 샤드

Elasticsearch(이하 ES)는 Apache Lucene 위에 분산 레이어를 얹은 검색 엔진이다. 가장 먼저 이해해야 할 것은 데이터가 어떻게 물리적으로 배치되는가다.

```
[Cluster: my-search-cluster]
├── Node 1 (master-eligible + data)
│   ├── Index: products  Shard 0 (Primary)
│   └── Index: products  Shard 2 (Replica of Node 2's Primary)
├── Node 2 (data)
│   ├── Index: products  Shard 1 (Primary)
│   └── Index: products  Shard 0 (Replica of Node 1's Primary)
└── Node 3 (data)
    ├── Index: products  Shard 2 (Primary)
    └── Index: products  Shard 1 (Replica of Node 2's Primary)
```

**인덱스(Index)** 는 관계형 DB의 테이블에 해당하지만, 내부적으로는 하나 이상의 **샤드(Shard)** 로 나뉜다. 각 샤드는 독립적인 Lucene 인스턴스다.

샤드 수는 인덱스 생성 시점에 결정되며, 이후 변경이 매우 어렵다(reindex 작업 필요). 따라서 초기 설계가 중요하다.

**레플리카 샤드(Replica Shard)** 는 가용성과 읽기 성능을 위해 존재한다. 프라이머리 샤드가 죽으면 레플리카가 프라이머리로 승격된다. 검색 요청은 프라이머리와 레플리카 모두에 분산되므로, 레플리카를 늘리면 읽기 처리량이 향상된다.

**마스터 노드** 는 클러스터 상태(어떤 노드가 어떤 샤드를 가지는지)를 관리한다. 프로덕션에서는 전용 마스터 노드를 별도로 두는 것이 안정적이다. 마스터, 데이터, 코디네이터 역할을 한 노드에 몰아넣으면 GC pause 동안 클러스터 전체가 흔들린다.

### 1.2 역인덱스(Inverted Index)의 원리

검색 엔진의 핵심은 역인덱스다. 일반 인덱스(forward index)가 "문서 → 단어 목록"을 저장한다면, 역인덱스는 "단어 → 문서 목록"을 저장한다.

**예시 문서들:**
```
Doc 1: "삼성 갤럭시 스마트폰 최신형"
Doc 2: "삼성 냉장고 에너지 효율"
Doc 3: "애플 아이폰 스마트폰 비교"
```

**역인덱스 구조:**
```
Term          | Doc IDs  | Positions
-------------|----------|----------
삼성          | [1, 2]   | [1:0, 2:0]
갤럭시        | [1]      | [1:1]
스마트폰      | [1, 3]   | [1:2, 3:1]
최신형        | [1]      | [1:3]
냉장고        | [2]      | [2:1]
에너지        | [2]      | [2:2]
효율          | [2]      | [2:3]
애플          | [3]      | [3:0]
아이폰        | [3]      | [3:1]
비교          | [3]      | [3:2]
```

"스마트폰 삼성"으로 검색하면 스마트폰=[1,3], 삼성=[1,2]의 교집합인 Doc 1이 가장 관련도 높은 결과로 나온다. LIKE '%스마트폰%' 방식의 풀스캔과는 차원이 다른 속도다.

### 1.3 세그먼트(Segment)와 Lucene의 쓰기 모델

Lucene은 **세그먼트(Segment)** 라는 불변(immutable) 단위로 데이터를 저장한다. 문서가 인덱싱되면 메모리 버퍼에 쌓이다가 주기적으로 새 세그먼트로 플러시된다. 삭제된 문서는 실제로 지워지지 않고 `.del` 파일에 마킹만 된다.

```
[메모리 버퍼] → (refresh, 기본 1초) → [세그먼트 N] (새로 검색 가능)
                                      [세그먼트 N-1]
                                      [세그먼트 N-2]
                                          ↓
                               (merge) [세그먼트 통합]  ← 삭제된 문서 실제 제거
```

`refresh_interval`이 1초인 이유가 여기 있다. 문서를 색인한 뒤 즉시 검색되지 않는 이유는 아직 세그먼트로 플러시되지 않았기 때문이다. 실시간성이 중요한 경우 `?refresh=true` 옵션을 쓸 수 있지만 성능 비용이 크다.

세그먼트가 너무 많아지면 검색 성능이 떨어진다(모든 세그먼트를 병렬 검색 후 머지). 백그라운드 **merge** 프로세스가 세그먼트를 주기적으로 통합한다. `force_merge` API로 수동으로 세그먼트를 1개로 통합하면 읽기 전용 인덱스(과거 로그 등)에서 검색 성능을 극대화할 수 있다.

### 1.4 문서 인덱싱 흐름

```
Client → [Coordinating Node]
             │
             ├── 라우팅 계산: shard = hash(routing) % num_shards
             │
             ↓
         [Primary Shard Node]
             │
             ├── 트랜스로그(translog)에 기록 (durability)
             ├── 메모리 버퍼에 추가
             ├── 레플리카로 복제 전송
             │
             ↓
         [Replica Shard Nodes]
             │
             └── 복제 완료 시 클라이언트에 응답
```

기본 `wait_for_active_shards=1`은 프라이머리만 쓰면 성공 응답을 준다. `all`로 설정하면 모든 레플리카 복제 완료 후 응답하므로 일관성은 높아지나 쓰기 레이턴시가 증가한다.

---

## 2. 검색 쿼리 설계

### 2.1 형태소 분석(Analysis)과 Analyzer

검색의 품질은 분석기(Analyzer) 설계에서 결정된다. 분석기는 세 단계로 구성된다:

```
원문 텍스트
    ↓
[Character Filter]  - HTML 태그 제거, 특수문자 처리
    ↓
[Tokenizer]         - 텍스트를 토큰으로 분리
    ↓
[Token Filter]      - 소문자 변환, 불용어 제거, 동의어 처리, 형태소 분석
    ↓
토큰 스트림 → 역인덱스에 저장
```

한국어 검색에서는 **nori** 분석기가 사실상 표준이다:

```json
PUT /products
{
  "settings": {
    "analysis": {
      "analyzer": {
        "korean_analyzer": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": [
            "nori_part_of_speech",
            "lowercase",
            "nori_readingform"
          ]
        },
        "korean_search_analyzer": {
          "type": "custom",
          "tokenizer": "nori_tokenizer",
          "filter": [
            "nori_part_of_speech",
            "lowercase",
            "synonym_filter"
          ]
        }
      },
      "filter": {
        "nori_part_of_speech": {
          "type": "nori_part_of_speech",
          "stoptags": ["E", "IC", "J", "MAG", "MAJ", "MM", "SP", "SSC", "SSO", "SC", "SE", "XPN", "XSA", "XSN", "XSV", "UNA", "NA", "VSV"]
        },
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "갤럭시, galaxy, 삼성폰",
            "아이폰, iphone, 애플폰"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "korean_analyzer",
        "search_analyzer": "korean_search_analyzer"
      }
    }
  }
}
```

**인덱스 시 분석기와 검색 시 분석기를 분리하는 것**이 핵심 패턴이다. 인덱스할 때는 더 공격적으로 분해하고(색인 품질), 검색할 때는 동의어를 확장해서 재현율을 높인다. 예를 들어 "삼성 갤럭시"로 검색할 때 "galaxy"나 "삼성폰"도 함께 매칭되게 하려면 검색 분석기에 동의어 필터를 추가해야 한다.

### 2.2 쿼리 DSL과 스코어링

ES의 쿼리는 크게 두 가지로 나뉜다:
- **Filter context**: 예/아니오 판단, 스코어 계산 없음, 캐시됨 → `filter`, `must_not`
- **Query context**: 관련도 스코어 계산, 캐시 안 됨 → `must`, `should`

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "삼성 갤럭시 스마트폰",
            "fields": ["name^3", "description^1", "brand^2"],
            "type": "best_fields",
            "operator": "or",
            "minimum_should_match": "75%"
          }
        }
      ],
      "filter": [
        { "term": { "category": "electronics" } },
        { "range": { "price": { "gte": 500000, "lte": 2000000 } } },
        { "term": { "in_stock": true } }
      ],
      "should": [
        { "term": { "is_promoted": true } },
        { "range": { "rating": { "gte": 4.5 } } }
      ],
      "boost": 1.0
    }
  },
  "sort": [
    { "_score": { "order": "desc" } },
    { "sales_count": { "order": "desc" } }
  ]
}
```

**BM25 스코어링**: ES 5.0 이후 기본 스코어링 알고리즘은 TF-IDF를 개선한 BM25다.

```
BM25 Score = IDF(t) × (TF(t,d) × (k1 + 1)) / (TF(t,d) + k1 × (1 - b + b × |d|/avgdl))
```

- `IDF(t)`: 검색어가 전체 문서에서 얼마나 희귀한가 (흔한 단어일수록 낮은 가중치)
- `TF(t,d)`: 문서 내 검색어 빈도 (많을수록 높은 가중치, 하지만 포화 효과 있음)
- `k1`, `b`: 튜닝 파라미터 (k1=1.2, b=0.75가 기본값)

실무에서 스코어가 예상대로 안 나온다면 `"explain": true`를 쿼리에 추가해서 스코어 계산 근거를 확인한다.

### 2.3 자동완성(Autocomplete) 구현

자동완성은 세 가지 방식이 있다. 각각 트레이드오프가 다르다.

**방법 1: Edge N-gram (가장 범용적)**

```json
PUT /autocomplete
{
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete_index": {
          "tokenizer": "standard",
          "filter": ["lowercase", "edge_ngram_filter"]
        },
        "autocomplete_search": {
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      },
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20
        }
      }
    }
  }
}
```

"갤럭시"를 색인하면 "갤", "갤럭", "갤럭시"가 모두 토큰으로 저장된다. 사용자가 "갤럭"을 입력하면 매칭된다. 단점은 인덱스 크기가 커진다.

**방법 2: Completion Suggester (성능 최고)**

```json
PUT /products
{
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "analyzer": "korean_analyzer"
      }
    }
  }
}

// 색인
POST /products/_doc
{
  "name": "삼성 갤럭시 S24",
  "suggest": {
    "input": ["삼성 갤럭시 S24", "갤럭시 S24", "Galaxy S24"],
    "weight": 100
  }
}

// 검색
GET /products/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "갤럭시",
      "completion": {
        "field": "suggest",
        "size": 10,
        "fuzzy": {
          "fuzziness": 1
        }
      }
    }
  }
}
```

Completion Suggester는 FST(Finite State Transducer) 자료구조를 메모리에 올려 처리한다. 레이턴시가 ms 이하다. 단, prefix 검색만 지원하고 중간 검색은 안 된다.

**방법 3: Search-as-you-type 필드 타입**

```json
"name": {
  "type": "search_as_you_type"
}
```

`search_as_you_type`은 내부적으로 2-gram, 3-gram, prefix 서브필드를 자동 생성한다. 간단하게 자동완성을 구현할 때 편리하다.

### 2.4 집계(Aggregation)로 패싯 검색 구현

이커머스의 "브랜드별 상품 수", "가격대별 필터링" 같은 패싯 검색은 집계(Aggregation)로 구현한다:

```json
GET /products/_search
{
  "query": {
    "match": { "name": "스마트폰" }
  },
  "aggs": {
    "brands": {
      "terms": {
        "field": "brand.keyword",
        "size": 10
      }
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 500000 },
          { "from": 500000, "to": 1000000 },
          { "from": 1000000, "to": 2000000 },
          { "from": 2000000 }
        ]
      }
    },
    "avg_rating": {
      "avg": { "field": "rating" }
    }
  },
  "size": 20
}
```

집계는 필터 컨텍스트에서 수행되므로 쿼리 필터와 독립적으로 전체 통계를 낼 수 있다. `post_filter`를 활용하면 집계 결과에는 영향을 주지 않으면서 결과만 필터링하는 패턴도 가능하다.

### 2.5 실무 안티패턴

**안티패턴 1: wildcard/regexp 남용**
```json
// 절대 하지 말 것 — 앞에 와일드카드는 전체 인덱스 스캔
{ "wildcard": { "name": "*갤럭시*" } }

// 올바른 방법 — 분석된 full-text 쿼리 사용
{ "match": { "name": "갤럭시" } }
```

**안티패턴 2: _all 필드 또는 너무 많은 필드**
모든 필드에 multi_match를 날리면 스코어링이 희석된다. 검색 대상 필드를 명시하고 부스팅으로 중요도를 표현한다.

**안티패턴 3: 너무 작은 shard 크기**
ES 공식 권장은 샤드 하나당 10GB~50GB. 수백 MB짜리 샤드가 수백 개 있으면 오버헤드가 극심하다.

**안티패턴 4: 동적 매핑에 의존**
ES는 처음 본 필드를 자동으로 매핑하는데, 날짜 형식 오해석, text/keyword 혼용 등으로 예상치 못한 동작이 발생한다. 명시적 매핑을 항상 정의한다.

---

## 3. 시계열 DB: 메트릭과 로그를 위한 전용 엔진

### 3.1 시계열 데이터의 특성

시계열 데이터는 일반 관계형 데이터와 근본적으로 다른 특성을 가진다:

- **단조 증가 타임스탬프**: 새 데이터는 항상 최신 시간에 삽입된다
- **높은 쓰기 처리량**: 수천 개의 서버에서 초당 수만 건의 메트릭이 들어온다
- **시간 범위 쿼리 집중**: "지난 1시간 평균", "최근 7일 트렌드" 형태의 쿼리가 대부분
- **데이터 TTL**: 1년 전 로그는 보통 필요 없다 — 자동 만료가 필수
- **고압축률**: 시간 순서로 정렬된 유사한 값들은 델타 인코딩으로 극도로 압축된다

PostgreSQL에 `CREATE TABLE metrics (time TIMESTAMPTZ, host VARCHAR, metric VARCHAR, value FLOAT)` 로 만들고 운영하면 어떻게 될까?

초당 10만 건 삽입 시 인덱스 유지 비용이 폭발하고, 시간 범위 쿼리는 전체 테이블 스캔에 가까워진다. 6개월 후에는 DELETE로 오래된 데이터를 지워야 하는데 이것도 엄청난 I/O다.

### 3.2 InfluxDB: 목적 특화 시계열 DB

InfluxDB의 데이터 모델은 관계형 DB와 다르다:

```
Measurement: cpu_usage
Tags (indexed, string): host=web-01, region=ap-northeast-2, env=prod
Fields (not indexed, typed): value=73.5, user=60.2, system=13.3
Timestamp: 2026-03-17T23:03:00Z
```

Tags는 그루핑/필터링에 쓰이는 카디널리티가 낮은 값, Fields는 실제 측정값이다. Tags는 자동으로 인덱싱되지만 Fields는 안 된다.

**Tags에 카디널리티가 높은 값(user_id, trace_id 등)을 넣으면 "High Series Cardinality" 문제로 메모리가 폭발한다.** InfluxDB는 각 고유한 태그 조합(Series)을 별도로 인덱싱하기 때문에, 태그 값의 종류가 수백만 개가 되면 인덱스 자체가 메모리를 고갈시킨다. 이것이 InfluxDB의 가장 흔한 운영 실수다.

**InfluxQL 쿼리 예시:**
```sql
-- 최근 1시간 동안 host별 평균 CPU 사용률을 5분 간격으로
SELECT mean("value") AS avg_cpu
FROM "cpu_usage"
WHERE time >= now() - 1h
  AND "env" = 'prod'
GROUP BY time(5m), "host"
FILL(null)
```

**Flux 쿼리 (InfluxDB 2.x):**
```flux
from(bucket: "metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu_usage" and r.env == "prod")
  |> filter(fn: (r) => r._field == "value")
  |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)
  |> group(columns: ["host"])
```

**TSM(Time-Structured Merge Tree) 스토리지 엔진**: InfluxDB는 LSM Tree 변형인 TSM을 사용한다. 데이터는 태그+필드+시간 순으로 정렬되어 저장되므로 시간 범위 쿼리와 집계가 극도로 빠르다. 2시간 이전 데이터는 디스크로 플러시되고, 같은 시리즈의 데이터가 연속 저장되어 압축률이 10:1을 넘는 경우도 흔하다.

**Retention Policy와 Continuous Query:**
```sql
-- 30일 보관 정책 생성
CREATE RETENTION POLICY "30d" ON "mydb" DURATION 30d REPLICATION 1 DEFAULT

-- 1시간 집계를 다른 측정값에 저장 (다운샘플링)
CREATE CONTINUOUS QUERY "cq_hourly_cpu" ON "mydb"
BEGIN
  SELECT mean("value") INTO "cpu_usage_1h" FROM "cpu_usage"
  GROUP BY time(1h), *
END
```

원본 데이터(1초 간격)는 7일, 1분 집계는 30일, 1시간 집계는 1년 보관하는 다운샘플링 전략이 표준이다.

### 3.3 TimescaleDB: PostgreSQL 위의 시계열

TimescaleDB는 PostgreSQL 확장으로, SQL을 그대로 쓰면서 시계열 최적화를 얻을 수 있다. 기존 PostgreSQL 인프라와 통합이 필요하거나, 복잡한 조인/윈도우 함수가 필요할 때 선택한다.

**하이퍼테이블(Hypertable)**: TimescaleDB의 핵심 개념이다. 하이퍼테이블은 내부적으로 시간 기반으로 파티셔닝된 **청크(Chunk)** 들의 집합이다.

```sql
-- 일반 PostgreSQL 테이블로 시작
CREATE TABLE metrics (
  time        TIMESTAMPTZ NOT NULL,
  host        TEXT NOT NULL,
  metric_name TEXT NOT NULL,
  value       DOUBLE PRECISION,
  tags        JSONB
);

-- 하이퍼테이블로 변환 (7일 단위 청크)
SELECT create_hypertable('metrics', 'time', chunk_time_interval => INTERVAL '7 days');

-- 압축 설정 (90일 이상 된 청크 압축)
ALTER TABLE metrics SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'host, metric_name',
  timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('metrics', INTERVAL '90 days');

-- 데이터 보관 정책 (1년 이후 삭제)
SELECT add_retention_policy('metrics', INTERVAL '1 year');
```

**TimescaleDB의 시간 집계 함수:**
```sql
-- 5분 버킷으로 평균 집계 (빈 버킷은 NULL로 채움)
SELECT
  time_bucket('5 minutes', time) AS bucket,
  host,
  avg(value) AS avg_cpu,
  max(value) AS max_cpu
FROM metrics
WHERE time > NOW() - INTERVAL '1 hour'
  AND metric_name = 'cpu_usage'
GROUP BY bucket, host
ORDER BY bucket DESC;

-- 연속 집계(Continuous Aggregate) — 마테리얼라이즈드 뷰의 시계열 버전
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS bucket,
  host,
  metric_name,
  avg(value) AS avg_value,
  max(value) AS max_value
FROM metrics
GROUP BY bucket, host, metric_name;

-- 자동 갱신 정책
SELECT add_continuous_aggregate_policy('metrics_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset   => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour'
);
```

### 3.4 InfluxDB vs TimescaleDB 선택 기준

| 기준 | InfluxDB | TimescaleDB |
|------|----------|-------------|
| 순수 시계열 성능 | 높음 (전용 엔진) | 준수 (PostgreSQL 기반) |
| SQL 지원 | 제한적 (InfluxQL/Flux) | 완전한 SQL |
| 기존 PostgreSQL 생태계 통합 | 불가 | 가능 |
| 복잡한 조인 쿼리 | 어려움 | 용이 |
| 운영 복잡도 | 낮음 (단순) | PostgreSQL 수준 |
| 클러스터링 | OSS 버전 제한 | 별도 설정 필요 |

메트릭/모니터링 전용 스택(Grafana + InfluxDB)이라면 InfluxDB, 기존 서비스 DB가 PostgreSQL이고 시계열 기능을 추가하고 싶다면 TimescaleDB가 자연스럽다.

---

## 4. 벡터 DB: AI 시대의 유사도 검색

### 4.1 벡터 임베딩이란

Large Language Model(LLM)이나 임베딩 모델은 텍스트, 이미지, 음성을 수백~수천 차원의 실수 벡터로 변환한다. 이 벡터 공간에서 의미적으로 유사한 것들은 가까이 위치한다.

```
"갤럭시 스마트폰" → [0.23, -0.45, 0.11, ..., 0.89]  (1536차원)
"삼성 핸드폰"     → [0.25, -0.43, 0.09, ..., 0.87]  (의미적으로 유사 → 코사인 유사도 높음)
"냉장고 구매"     → [-0.31, 0.67, -0.55, ..., 0.12] (의미적으로 다름 → 유사도 낮음)
```

"갤럭시 스마트폰"과 "삼성 핸드폰"은 키워드가 전혀 다르지만 임베딩 공간에서 가깝다. 역인덱스 기반 검색은 이 두 문서를 연결하지 못하지만, 벡터 검색은 가능하다. 이것이 **시맨틱 검색(Semantic Search)** 이다.

### 4.2 ANN 알고리즘: HNSW와 IVF

1536차원 벡터 수백만 개 중에서 가장 가까운 k개를 찾는 것(kNN)을 브루트포스로 하면 O(n×d) 연산이 필요하다. 실시간 서비스에서는 불가능하므로 **근사 최근접 이웃(ANN, Approximate Nearest Neighbor)** 알고리즘을 사용한다.

**HNSW (Hierarchical Navigable Small World)**

```
Layer 2 (희소): ●-----------●-----------●
                    \                  /
Layer 1 (중간): ●--●--●------●--●--●--●
                 \  |   \   /   |  |
Layer 0 (밀집): ●-●-●-●-●-●-●-●-●-●-●-●
```

멀티레이어 그래프 구조. 상위 레이어에서 빠르게 근방을 찾고, 하위 레이어에서 정밀하게 좁힌다. 정확도와 속도의 균형이 좋아 대부분의 벡터 DB에서 기본 알고리즘으로 사용한다.

- `ef_construction`: 색인 시 탐색 폭 (클수록 정확, 느린 색인)
- `M`: 각 노드의 최대 연결 수 (클수록 정확, 많은 메모리)
- `ef_search`: 검색 시 탐색 폭 (클수록 정확, 느린 검색)

**IVF (Inverted File Index)**

벡터 공간을 클러스터로 나누고(K-Means), 쿼리 벡터와 가까운 클러스터만 탐색한다. 대용량(수억 건)에서 메모리 효율적이나, 클러스터 경계 근처의 벡터는 누락될 수 있다.

### 4.3 pgvector: PostgreSQL에 벡터 검색 추가

가장 진입 장벽이 낮은 선택지다. 기존 PostgreSQL 데이터와 벡터를 같은 테이블에 저장할 수 있다.

```sql
-- 확장 활성화
CREATE EXTENSION vector;

-- 임베딩 컬럼이 있는 테이블
CREATE TABLE documents (
  id          BIGSERIAL PRIMARY KEY,
  title       TEXT NOT NULL,
  content     TEXT NOT NULL,
  embedding   vector(1536),  -- OpenAI text-embedding-3-small 차원
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW 인덱스 생성
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 또는 IVFFlat 인덱스
CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

**쿼리 예시 — 의미적으로 유사한 문서 찾기:**
```sql
-- 쿼리 벡터 (실제로는 애플리케이션에서 임베딩 모델로 생성)
SELECT
  id,
  title,
  content,
  1 - (embedding <=> '[0.23, -0.45, 0.11, ...]'::vector) AS similarity
FROM documents
WHERE created_at > NOW() - INTERVAL '1 year'  -- 필터 조건
ORDER BY embedding <=> '[0.23, -0.45, 0.11, ...]'::vector
LIMIT 10;
```

연산자 의미:
- `<=>`: 코사인 거리 (방향 유사도, 텍스트에 적합)
- `<->`: L2 거리 (유클리드 거리, 이미지 임베딩에 적합)
- `<#>`: 음의 내적 (정규화된 벡터에서 코사인과 동일)

**pgvector의 한계**: HNSW 인덱스는 메모리에 올라가야 한다. 수억 건의 벡터는 현실적으로 어렵다. 수백만 건 이하, 기존 PostgreSQL 스택과 통합이 필요한 경우에 적합하다.

### 4.4 Pinecone: 완전관리형 벡터 DB

Pinecone은 벡터 DB 전용 클라우드 서비스다. 인프라 관리 없이 REST API로 벡터를 저장하고 검색할 수 있다.

```python
import pinecone
from openai import OpenAI

# 초기화
pc = pinecone.Pinecone(api_key="YOUR_API_KEY")
openai_client = OpenAI(api_key="YOUR_OPENAI_KEY")

# 인덱스 생성 (처음 한 번만)
pc.create_index(
    name="product-search",
    dimension=1536,
    metric="cosine",
    spec=pinecone.ServerlessSpec(cloud="aws", region="ap-northeast-1")
)

index = pc.Index("product-search")

# 벡터 업서트
def embed_text(text: str) -> list[float]:
    response = openai_client.embeddings.create(
        input=text,
        model="text-embedding-3-small"
    )
    return response.data[0].embedding

# 상품 색인
products = [
    {"id": "prod-001", "name": "삼성 갤럭시 S24", "category": "smartphone"},
    {"id": "prod-002", "name": "삼성 냉장고 비스포크", "category": "appliance"},
]

vectors = [
    {
        "id": p["id"],
        "values": embed_text(p["name"]),
        "metadata": {"name": p["name"], "category": p["category"]}
    }
    for p in products
]

index.upsert(vectors=vectors, namespace="products")

# 유사도 검색
query_embedding = embed_text("삼성 스마트폰 추천")

results = index.query(
    vector=query_embedding,
    top_k=5,
    include_metadata=True,
    filter={"category": {"$eq": "smartphone"}},  # 메타데이터 필터
    namespace="products"
)

for match in results.matches:
    print(f"Score: {match.score:.4f} | {match.metadata['name']}")
```

Pinecone의 **메타데이터 필터링**은 벡터 유사도 검색과 스칼라 필터를 결합한다. 이를 **하이브리드 검색**의 일종으로 볼 수 있다.

### 4.5 Milvus: 오픈소스 고성능 벡터 DB

대용량(수억 건 이상), 온프레미스 배포, 복잡한 인덱스 전략이 필요한 경우 Milvus를 선택한다.

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType, utility

# 연결
connections.connect(host="localhost", port="19530")

# 스키마 정의
fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="product_id", dtype=DataType.VARCHAR, max_length=100),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=1536),
    FieldSchema(name="category", dtype=DataType.VARCHAR, max_length=50),
    FieldSchema(name="price", dtype=DataType.FLOAT),
]
schema = CollectionSchema(fields, description="Product embeddings")
collection = Collection("products", schema)

# HNSW 인덱스 생성
index_params = {
    "metric_type": "COSINE",
    "index_type": "HNSW",
    "params": {"M": 16, "efConstruction": 200}
}
collection.create_index(field_name="embedding", index_params=index_params)
collection.load()

# 검색
search_params = {"metric_type": "COSINE", "params": {"ef": 64}}
results = collection.search(
    data=[query_embedding],
    anns_field="embedding",
    param=search_params,
    limit=10,
    expr="category == 'smartphone' and price < 1500000",  # 스칼라 필터
    output_fields=["product_id", "category", "price"]
)
```

### 4.6 RAG (Retrieval-Augmented Generation) 패턴

벡터 DB의 가장 중요한 실용 사례는 RAG다. LLM의 hallucination을 줄이고 최신 정보를 제공하기 위해 관련 컨텍스트를 DB에서 검색해 프롬프트에 주입한다.

```
사용자 질문
    ↓
[임베딩 모델] → 질문 벡터
    ↓
[벡터 DB 검색] → 관련 문서 Top-K
    ↓
[프롬프트 조합]
  "다음 문서를 참고하여 답변하세요:
   {관련 문서 1}
   {관련 문서 2}
   ...
   질문: {사용자 질문}"
    ↓
[LLM] → 최종 답변
```

실무에서 RAG 품질을 올리는 핵심은:
1. **청킹 전략**: 문서를 어떻게 나누는가 (문장/문단/고정 길이/의미 단위)
2. **임베딩 모델 선택**: 도메인 특화 모델이 범용 모델보다 성능 좋음
3. **하이브리드 검색**: 벡터 유사도 + BM25 키워드 검색 결합 → Reciprocal Rank Fusion

---

## 5. 폴리글랏 퍼시스턴스: 용도별 DB 선택과 통합 설계

### 5.1 용도별 DB 선택 기준표

한 시스템에서 여러 DB를 혼용하는 것을 **폴리글랏 퍼시스턴스(Polyglot Persistence)** 라고 한다. 각 데이터 특성에 맞는 저장소를 선택하는 전략이다.

| 데이터 특성 | 추천 DB | 이유 |
|-------------|---------|------|
| 정형 트랜잭션 데이터 (주문, 결제) | PostgreSQL, MySQL | ACID, 복잡한 조인 |
| 전문 검색, 패싯 검색 | Elasticsearch | 역인덱스, BM25 스코어링 |
| 세션, 캐시, 리더보드 | Redis | 인메모리, O(1) 접근 |
| 메트릭, 모니터링 | InfluxDB, TimescaleDB | 시계열 최적화 |
| 문서, 이벤트 로그 | MongoDB, DynamoDB | 스키마 유연성 |
| 그래프 관계 (추천, SNS) | Neo4j, Amazon Neptune | 관계 탐색 최적화 |
| 의미적 유사도 검색 | pgvector, Pinecone, Milvus | ANN 알고리즘 |
| 대용량 배치 분석 | BigQuery, Redshift | 컬럼 스토어 |

### 5.2 이커머스 시스템의 폴리글랏 설계 예시

```
[이커머스 플랫폼]

┌─────────────────────────────────────────────────────┐
│                   API Gateway                        │
└──────┬────────────┬─────────────┬────────────────────┘
       │            │             │
  [상품 API]   [검색 API]   [추천 API]
       │            │             │
  PostgreSQL   Elasticsearch  Redis (세션/캐시)
  (상품 정보,   (전문 검색,     + pgvector
   주문, 재고)   자동완성,       (유사 상품 추천)
                패싯 필터)

[이벤트 스트림 - Kafka]
       │
  ┌────┴────────────────┐
  │                     │
 InfluxDB            Elasticsearch
(클릭/구매 메트릭,   (사용자 행동 로그,
 서버 성능 지표)      오류 추적)
```

**동기화 전략**: 상품 정보는 PostgreSQL이 원본(source of truth)이다. 상품이 생성/수정되면 Kafka 이벤트를 통해 Elasticsearch와 pgvector를 비동기로 업데이트한다.

이 구조에서 검색 DB가 잠깐 오래된 데이터를 보여줄 수 있지만(최종 일관성), 구매/결제는 항상 PostgreSQL에서 최신 데이터를 참조한다.

### 5.3 검색과 벡터의 하이브리드 전략

순수 키워드 검색(BM25)과 순수 벡터 검색 모두 한계가 있다:
- 키워드 검색: "아이폰"으로 "iPhone"을 못 찾음, 동의어/오타에 약함
- 벡터 검색: "Samsung Galaxy S24 Ultra 256GB" 같은 정확한 모델명 매칭에 약함

실무 해결책은 **하이브리드 검색 + 재순위화(Reranking)**:

```python
from elasticsearch import Elasticsearch
import numpy as np

def hybrid_search(query: str, top_k: int = 20) -> list:
    es = Elasticsearch()

    # 1. BM25 키워드 검색
    bm25_results = es.search(
        index="products",
        body={
            "query": {"match": {"name": query}},
            "size": top_k
        }
    )

    # 2. 벡터 유사도 검색
    query_vector = embed_text(query)
    vector_results = es.search(
        index="products",
        body={
            "knn": {
                "field": "embedding",
                "query_vector": query_vector,
                "k": top_k,
                "num_candidates": top_k * 5
            }
        }
    )

    # 3. Reciprocal Rank Fusion (RRF)
    # 각 결과의 순위를 역수로 변환해서 합산
    scores = {}
    k_const = 60  # RRF 상수

    for rank, hit in enumerate(bm25_results["hits"]["hits"]):
        doc_id = hit["_id"]
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k_const + rank + 1)

    for rank, hit in enumerate(vector_results["hits"]["hits"]):
        doc_id = hit["_id"]
        scores[doc_id] = scores.get(doc_id, 0) + 1 / (k_const + rank + 1)

    # 스코어 내림차순 정렬
    ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return ranked[:top_k]
```

ES 8.x부터는 `knn`과 `query`를 동시에 사용하는 네이티브 하이브리드 검색을 지원한다.

### 5.4 운영 복잡도 관리: 언제 단순함을 선택할 것인가

폴리글랏 퍼시스턴스는 강력하지만 비용이 따른다:
- 각 DB의 운영 지식이 필요하다
- 데이터 동기화 로직의 버그는 찾기 어렵다
- 장애 시나리오가 기하급수적으로 늘어난다

**현실적인 도입 기준:**

초기에는 PostgreSQL로 시작하라. TimescaleDB 확장으로 시계열을, pgvector 확장으로 벡터 검색을 추가하면 단일 DB로 상당히 많은 것을 해결할 수 있다.

PostgreSQL 한계에 실제로 부딪히는 시점 — 검색 결과 품질이 불만족스럽고 월 검색량이 수천만을 넘거나, 초당 수만 건의 메트릭 인입이 시작되거나 — 그 시점에 전용 DB로 마이그레이션한다.

"트래픽이 늘면 쓰려고 미리 준비"는 함정이다. 실제로 그 트래픽이 오기 전에 팀이 복잡도에 압도될 수 있다.

### 5.5 실전 장애 시나리오와 대응

**시나리오 1: Elasticsearch 색인 지연**
- 현상: 상품 등록 후 검색에 안 보임
- 원인: refresh_interval (기본 1초) 대기 중
- 대응: 관리자 도구에서만 `?refresh=true` 사용, 일반 사용자에게는 "잠시 후 검색 가능" 안내

**시나리오 2: InfluxDB High Series Cardinality**
- 현상: OOM으로 InfluxDB 반복 크래시
- 원인: tags에 user_id나 trace_id 같은 무제한 카디널리티 값 삽입
- 대응: `SHOW SERIES CARDINALITY ON mydb` 로 확인, 문제 태그를 fields로 이동

**시나리오 3: pgvector 인덱스 미활용**
- 현상: 벡터 쿼리가 느림 (수초 소요)
- 원인: WHERE 조건이 있을 때 PostgreSQL이 인덱스 대신 시퀀셜 스캔을 선택
- 대응: `EXPLAIN ANALYZE`로 확인, `SET enable_seqscan = off`로 디버그, `ivfflat.probes` 또는 `hnsw.ef_search` 파라미터 조정

**시나리오 4: ES 클러스터 yellow/red 상태**
- 현상: 레플리카 샤드 미할당으로 클러스터 yellow
- 원인: 노드 수보다 레플리카 설정이 높거나, 노드 장애 후 미복구
- 대응: `GET /_cluster/allocation/explain` → `POST /_cluster/reroute`

---

## 마무리: 도구보다 문제를 먼저 정의하라

특수 목적 DB의 세계는 넓고 깊다. Elasticsearch의 역인덱스, 시계열 DB의 TSM 스토리지, 벡터 DB의 HNSW — 이 모든 것은 특정 문제를 극한까지 최적화한 결과물이다.

그 도구를 올바르게 활용하려면, 먼저 자신이 어떤 문제를 풀려는지 명확해야 한다.

"어떤 텍스트 필드에서 관련도 순 검색이 필요한가?" → Elasticsearch.
"수백만 개의 시계열 포인트를 초당 수만 건씩 쓰고, 시간 범위 집계를 한다?" → InfluxDB 또는 TimescaleDB.
"의미적 유사도로 문서/상품/이미지를 찾는다?" → pgvector 또는 전용 벡터 DB.

도구를 먼저 선택하고 문제를 끼워 맞추지 않는 것. 그것이 폴리글랏 퍼시스턴스의 진짜 원칙이다.

---

## 참고 자료

1. **"Elasticsearch: The Definitive Guide"** — Clinton Gormley, Zachary Tong (O'Reilly) — ES의 내부 동작과 검색 쿼리 설계의 기본서
2. **"Designing Data-Intensive Applications"** — Martin Kleppmann (O'Reilly) — 분산 시스템과 다양한 DB 패러다임을 아우르는 필독서
3. **Elasticsearch 공식 문서** — https://www.elastic.co/guide/en/elasticsearch/reference/current/ — 최신 쿼리 DSL, 매핑, 집계 레퍼런스
4. **TimescaleDB 공식 문서** — https://docs.timescale.com/ — 하이퍼테이블, 연속 집계, 압축 정책의 구체적 설명
5. **"Vector Databases" by Pinecone** — https://www.pinecone.io/learn/vector-database/ — ANN 알고리즘과 벡터 DB 아키텍처에 대한 무료 심층 가이드
6. **pgvector GitHub** — https://github.com/pgvector/pgvector — HNSW/IVFFlat 인덱스 파라미터 튜닝 가이드와 성능 벤치마크
