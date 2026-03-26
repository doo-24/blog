---
title: "[성능 최적화와 안정성] 4편 — 부하 테스트: 서버의 한계를 미리 아는 법"
date: 2026-03-17T18:03:00+09:00
draft: false
tags: ["부하 테스트", "k6", "Gatling", "TPS", "성능", "서버"]
series: ["성능 최적화와 안정성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 8
summary: "부하 테스트 종류(Load, Stress, Spike, Soak), k6/nGrinder/Gatling 실전 시나리오, TPS·Latency(p50/p95/p99)·처리량 해석, 병목 지점 찾기와 결과 보고서 작성까지"
---

프로덕션 장애의 상당수는 "예상보다 많은 트래픽"에서 시작된다.

런칭 전날 QA를 통과한 서버가, 오픈 당일 유저가 몰리자 3분 만에 응답을 멈추는 장면은 낯설지 않다.

부하 테스트는 그 장면을 미리 재현해서 서버의 한계를 사전에 파악하는 실험이다.

단순히 "버티는지 확인"하는 게 아니라, 병목이 어디에 있는지, 어느 지점에서 성능이 꺾이는지, 복구는 되는지를 측정하는 과학적 과정이다.

---

## 1. 부하 테스트의 종류

부하 테스트는 목적에 따라 네 가지로 나뉜다.

각각의 테스트가 질문하는 것이 다르기 때문에, 상황에 맞는 테스트를 선택해야 한다.

### Load Test (부하 테스트)

"정상 트래픽 또는 예상 최대 트래픽을 견디는가?"를 확인한다.

예상 동시 접속자 수를 서서히 올리며 TPS, Latency, 에러율을 측정한다. 가장 기본적인 테스트이며, SLA(Service Level Agreement) 기준을 충족하는지 검증하는 데 쓰인다.

```
가상 유저(VU): 0 → 100 → 100 (유지) → 0
시간: ramp-up 2분, 유지 10분, ramp-down 2분
```

### Stress Test (스트레스 테스트)

"한계를 넘어서도 우아하게 실패하는가?"를 확인한다.

부하를 단계적으로 높여가며 서버가 언제, 어떻게 무너지는지 관찰한다. 에러율이 급증하는 지점, 응답시간이 폭발적으로 늘어나는 지점이 서버의 실제 한계 용량이다.

```
가상 유저: 0 → 50 → 100 → 200 → 400 → 800 ...
각 단계 유지: 5분
목표: 에러율 5% 초과 또는 p99 > 5s 도달 시점 확인
```

### Spike Test (스파이크 테스트)

"갑작스러운 폭발적 트래픽을 처리할 수 있는가?"를 확인한다.

티켓팅 오픈, 뉴스 노출, 마케팅 이벤트처럼 트래픽이 순식간에 수십 배로 뛰는 시나리오를 재현한다. Auto Scaling이 제때 동작하는지, 큐(Queue)가 버퍼 역할을 하는지도 함께 검증한다.

```
가상 유저: 10 → 1000 (10초 안에) → 10 (1분 후)
확인 지점: 피크 도달 후 에러율, 피크 이후 복구 시간
```

### Soak Test (지속 테스트, Endurance Test)

"장시간 운영에서 성능이 유지되는가?"를 확인한다.

메모리 누수, 커넥션 풀 고갈, 디스크 공간 부족처럼 시간이 지남에 따라 서서히 발생하는 문제를 탐지한다. 적정 부하를 6시간 ~ 24시간 지속적으로 유지하며 관찰한다.

```
가상 유저: 100 (일정하게 유지)
시간: 6시간 이상
확인 지점: JVM Heap 증가 추이, DB 커넥션 수, 응답시간 드리프트
```

---

## 2. k6 실전 시나리오

k6는 Go로 작성되어 가볍고, JavaScript로 시나리오를 작성한다. CLI 기반이라 CI/CD 파이프라인에 붙이기 쉽다.

### 설치

```bash
# macOS
brew install k6

# Ubuntu
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
echo "deb https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
sudo apt-get update && sudo apt-get install k6

# Docker
docker run --rm -i grafana/k6 run - <script.js
```

### 기본 Load Test 시나리오

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// 커스텀 메트릭 정의
const errorRate = new Rate('error_rate');
const apiLatency = new Trend('api_latency', true);

export const options = {
  stages: [
    { duration: '2m', target: 50 },   // ramp-up: 0 → 50 VU
    { duration: '10m', target: 50 },  // steady state: 50 VU 유지
    { duration: '2m', target: 0 },    // ramp-down: 50 → 0 VU
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // p95 < 500ms, p99 < 1s
    error_rate: ['rate<0.01'],                        // 에러율 1% 미만
    http_req_failed: ['rate<0.01'],
  },
};

// 테스트 데이터 (실제 환경에서는 파일에서 읽어온다)
const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

export default function () {
  // 1. 로그인
  const loginRes = http.post(`${BASE_URL}/api/auth/login`, JSON.stringify({
    email: `user${Math.floor(Math.random() * 1000)}@test.com`,
    password: 'testpass123',
  }), {
    headers: { 'Content-Type': 'application/json' },
    tags: { name: 'login' },
  });

  check(loginRes, {
    'login status 200': (r) => r.status === 200,
    'login has token': (r) => JSON.parse(r.body).token !== undefined,
  });

  errorRate.add(loginRes.status !== 200);
  apiLatency.add(loginRes.timings.duration, { endpoint: 'login' });

  if (loginRes.status !== 200) return;

  const token = JSON.parse(loginRes.body).token;
  const authHeaders = {
    Authorization: `Bearer ${token}`,
    'Content-Type': 'application/json',
  };

  sleep(1); // 실제 유저처럼 잠깐 대기

  // 2. 상품 목록 조회
  const listRes = http.get(`${BASE_URL}/api/products?page=1&size=20`, {
    headers: authHeaders,
    tags: { name: 'product_list' },
  });

  check(listRes, {
    'product list status 200': (r) => r.status === 200,
    'product list not empty': (r) => JSON.parse(r.body).items.length > 0,
  });

  errorRate.add(listRes.status !== 200);
  apiLatency.add(listRes.timings.duration, { endpoint: 'product_list' });

  sleep(2);

  // 3. 상품 상세 조회
  const productId = Math.floor(Math.random() * 100) + 1;
  const detailRes = http.get(`${BASE_URL}/api/products/${productId}`, {
    headers: authHeaders,
    tags: { name: 'product_detail' },
  });

  check(detailRes, {
    'product detail status 200': (r) => r.status === 200,
  });

  errorRate.add(detailRes.status !== 200);
  apiLatency.add(detailRes.timings.duration, { endpoint: 'product_detail' });

  sleep(1);
}
```

### Stress Test 시나리오

```javascript
// stress-test.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '5m', target: 100 },
    { duration: '5m', target: 200 },
    { duration: '5m', target: 400 },
    { duration: '5m', target: 800 },
    { duration: '5m', target: 1200 },
    { duration: '10m', target: 0 },  // 복구 관찰
  ],
  thresholds: {
    http_req_duration: ['p(99)<5000'],  // 스트레스 테스트는 관대하게 설정
    http_req_failed: ['rate<0.1'],       // 에러율 10%까지 허용
  },
};

export default function () {
  const res = http.get('http://localhost:8080/api/health');
  check(res, { 'status 200': (r) => r.status === 200 });
}
```

### 실행 및 결과 출력

```bash
# 기본 실행
k6 run load-test.js

# 환경변수 주입
k6 run -e BASE_URL=https://api.example.com load-test.js

# JSON 결과 저장
k6 run --out json=results.json load-test.js

# InfluxDB + Grafana 연동 (실시간 대시보드)
k6 run --out influxdb=http://localhost:8086/k6 load-test.js

# HTML 리포트 생성 (k6-reporter 사용 시)
k6 run --out json=results.json load-test.js
node reporter.js results.json
```

---

## 3. Gatling 실전 시나리오

Gatling은 Scala DSL로 시나리오를 작성하며, 상세한 HTML 리포트가 강점이다. 복잡한 비즈니스 플로우를 표현하기에 적합하다.

### 프로젝트 설정 (build.sbt)

```scala
// build.sbt
enablePlugins(GatlingPlugin)

scalaVersion := "2.13.10"

libraryDependencies ++= Seq(
  "io.gatling.highcharts" % "gatling-charts-highcharts" % "3.9.5" % Test,
  "io.gatling"            % "gatling-test-framework"    % "3.9.5" % Test
)
```

### 기본 시뮬레이션

```scala
// src/test/scala/simulations/ProductSimulation.scala
package simulations

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class ProductSimulation extends Simulation {

  // HTTP 프로토콜 설정
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .userAgentHeader("Gatling/3.9")

  // 피더: CSV에서 테스트 데이터 읽기
  val userFeeder = csv("users.csv").random

  // 로그인 → 상품 조회 → 주문 시나리오
  val productScenario = scenario("상품 주문 플로우")
    .feed(userFeeder)
    .exec(
      http("로그인")
        .post("/api/auth/login")
        .body(StringBody("""{"email":"${email}","password":"${password}"}"""))
        .check(status.is(200))
        .check(jsonPath("$.token").saveAs("authToken"))
    )
    .pause(1.second, 3.seconds)
    .exec(
      http("상품 목록 조회")
        .get("/api/products?page=1&size=20")
        .header("Authorization", "Bearer ${authToken}")
        .check(status.is(200))
        .check(jsonPath("$.items[0].id").saveAs("productId"))
    )
    .pause(2.seconds)
    .exec(
      http("상품 상세 조회")
        .get("/api/products/${productId}")
        .header("Authorization", "Bearer ${authToken}")
        .check(status.is(200))
    )
    .pause(1.second)
    .exec(
      http("주문 생성")
        .post("/api/orders")
        .header("Authorization", "Bearer ${authToken}")
        .body(StringBody("""{"productId":"${productId}","quantity":1}"""))
        .check(status.is(201))
        .check(jsonPath("$.orderId").exists)
    )

  // 부하 설정: Ramp-up 후 유지
  setUp(
    productScenario.inject(
      rampUsersPerSec(1).to(50).during(2.minutes),
      constantUsersPerSec(50).during(10.minutes),
      rampUsersPerSec(50).to(0).during(2.minutes)
    )
  )
    .protocols(httpProtocol)
    .assertions(
      global.responseTime.percentile3.lt(500),   // p95 < 500ms
      global.responseTime.percentile4.lt(1000),  // p99 < 1000ms
      global.successfulRequests.percent.gt(99)   // 성공률 99% 이상
    )
}
```

### 실행

```bash
# Maven
mvn gatling:test -Dgatling.simulationClass=simulations.ProductSimulation

# SBT
sbt "Gatling/testOnly simulations.ProductSimulation"

# 리포트는 target/gatling/ 디렉토리에 생성됨
```

---

## 4. nGrinder 간략 설명

nGrinder는 네이버가 만든 오픈소스 분산 부하 테스트 플랫폼이다.

웹 UI에서 Groovy/Jython으로 스크립트를 작성하고, Controller-Agent 구조로 분산 부하를 생성한다. 대규모 테스트 인프라가 필요한 조직에 적합하며, 결과 히스토리를 DB에 저장해서 회귀 비교가 쉽다.

```groovy
// nGrinder Groovy 스크립트 예시
import static net.grinder.script.Grinder.grinder
import net.grinder.script.Test
import org.ngrinder.groovy.util.Http

class TestRunner {
    def static http = new Http("http://localhost:8080")

    @Test
    public void test() {
        def response = http.GET("/api/products")
        assert response.statusCode == 200
    }
}
```

---

## 5. TPS, Latency, 처리량 해석

숫자를 읽는 것과 숫자를 이해하는 것은 다르다.

### TPS (Transactions Per Second)

TPS는 초당 처리한 요청 수다. 높을수록 좋지만, 에러율과 함께 봐야 한다.

에러 응답도 TPS에 포함되기 때문에, 에러가 터지면서 TPS가 높아 보이는 함정이 있다. 반드시 `successful TPS`를 기준으로 판단해야 한다.

```
TPS = 완료된 요청 수 / 측정 시간(초)
예: 10분(600초) 동안 180,000 요청 완료 → TPS = 300
```

### Latency 퍼센타일

평균(average)은 아웃라이어에 왜곡된다. 퍼센타일이 진실을 말한다.

- **p50 (Median)**: 절반의 요청이 이 시간 이내에 처리된다. "보통의 경험"
- **p95**: 95%의 요청이 이 시간 이내에 처리된다. "거의 모든 사용자의 경험"
- **p99**: 99%의 요청이 이 시간 이내에 처리된다. "극단적인 느린 경험 탐지"

```
좋은 지표 예시:
  p50:  45ms
  p95: 120ms
  p99: 380ms

나쁜 지표 예시 (롱테일 문제):
  p50:  60ms
  p95: 800ms    ← p50 대비 급격한 증가
  p99: 4500ms   ← 심각한 롱테일
```

p95와 p99의 차이가 크면 특정 요청 유형이나 특정 데이터 조건에서 느린 쿼리가 존재한다는 신호다.

### 처리량 (Throughput)

처리량은 단위 시간당 전송된 데이터 양이다. 네트워크 대역폭 병목을 탐지하는 데 유용하다.

```
예: TPS 500, 평균 응답 크기 10KB
→ 처리량 = 500 × 10KB = 5MB/s
→ 서버 네트워크가 100Mbps라면 충분
→ 서버 네트워크가 10Mbps라면 → 네트워크가 병목
```

### k6 출력 읽기

```
scenarios: (100.00%) 1 scenario, 50 max VUs, 14m30s max duration
default: Up to 50 looping VUs for 14m0s (gracefulStop: 30s)

✓ login status 200
✓ product list status 200

checks.........................: 99.82% ✓ 71850 ✗ 128
data_received..................: 234 MB 278 kB/s
data_sent......................: 45 MB  53 kB/s
http_req_blocked...............: avg=12µs    min=1µs    med=4µs    max=45ms   p(90)=7µs    p(95)=9µs
http_req_duration..............: avg=145ms   min=12ms   med=98ms   max=3.2s   p(90)=310ms  p(95)=480ms p(99)=920ms
http_req_failed................: 0.17%  ✓ 0    ✗ 128
http_reqs......................: 71978  84.9/s
iteration_duration.............: avg=4.1s    min=3.1s   med=3.9s   max=8.4s   p(90)=5.2s   p(95)=5.9s
iterations.....................: 23993  28.3/s
vus............................: 50     min=0 max=50
vus_max........................: 50     min=50 max=50

✓ http_req_duration............: p(95)=480ms < 500ms  ← threshold 통과
✗ http_req_failed...............: 0.17% >= 0.01%       ← threshold 실패
```

`http_req_blocked`는 TCP 연결을 기다린 시간이다. 이 값이 크면 커넥션 풀이 부족하다는 신호다.

---

## 6. 병목 지점 찾기

부하 테스트에서 성능이 나쁘게 나왔을 때, 원인을 찾는 체계적 접근이 필요하다.

### CPU 병목

```bash
# 테스트 중 CPU 확인
top -p $(pgrep -f java)

# 또는 htop
htop

# 스레드별 CPU 확인
ps -eLo pid,tid,pcpu,comm | grep java | sort -k3 -rn | head -20
```

CPU가 지속적으로 80% 이상이라면 애플리케이션 로직 최적화, 알고리즘 개선, 스케일 아웃이 필요하다.

### 메모리 및 GC 병목

```bash
# JVM GC 모니터링
jstat -gcutil <pid> 1000

# 출력 예시:
#  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
#   0.00  12.47  65.32  88.91  94.23  91.45    142    3.421     8    4.892    8.313
# O(Old Gen)가 지속적으로 높으면 메모리 누수 의심
# FGC(Full GC) 빈도가 높으면 Heap 크기 조정 필요
```

Full GC가 잦으면 응답시간이 갑자기 수 초씩 튀는 현상이 발생한다. 이는 p99 급등의 주요 원인이다.

### DB 병목

```sql
-- PostgreSQL: 느린 쿼리 확인
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- 현재 실행 중인 쿼리
SELECT pid, query, query_start, state
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;

-- 잠금 대기 확인
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

DB 커넥션 수도 확인해야 한다.

```sql
-- PostgreSQL 커넥션 현황
SELECT count(*), state
FROM pg_stat_activity
GROUP BY state;
```

커넥션 수가 max_connections에 근접하면 HikariCP 풀 크기 조정 또는 PgBouncer 같은 커넥션 풀러 도입이 필요하다.

### 네트워크 병목

```bash
# 네트워크 처리량 확인
iftop -i eth0

# 또는 nethogs (프로세스별)
nethogs

# TCP 연결 상태
ss -s
# ESTABLISHED: 현재 연결된 수
# TIME_WAIT: 연결 종료 후 대기 (많으면 SO_REUSEADDR, tcp_tw_reuse 설정 고려)
```

### 스레드 병목

```bash
# Java 스레드 덤프
kill -3 <pid>          # 표준 출력으로 출력
jstack <pid>           # 파일로 저장

# 스레드 상태 요약
jstack <pid> | grep "java.lang.Thread.State" | sort | uniq -c | sort -rn
# WAITING, BLOCKED 상태 스레드가 많으면 동기화 문제 의심
```

---

## 7. 안티패턴

### 안티패턴 1: Think Time 미포함

실제 유저는 페이지를 읽고, 클릭하고, 입력하는 시간이 있다. Think Time 없이 요청을 연속으로 보내면 실제보다 훨씬 가혹한 테스트가 된다.

```javascript
// 나쁜 예: think time 없음
export default function () {
  http.get('/api/products');
  http.get('/api/cart');
  http.post('/api/orders');
}

// 좋은 예: 실제 사용자 행동 반영
export default function () {
  http.get('/api/products');
  sleep(2);  // 유저가 목록을 보는 시간
  http.get('/api/cart');
  sleep(1);
  http.post('/api/orders');
}
```

### 안티패턴 2: 단일 엔드포인트만 테스트

`/api/health`만 100 TPS 나왔다고 만족하면 안 된다. 실제 트래픽 비율을 반영한 혼합 시나리오가 필요하다.

```javascript
// 실제 트래픽 비율 반영 예시
// 상품 조회 70%, 검색 20%, 주문 10%
export const options = {
  scenarios: {
    browse: {
      executor: 'constant-vus',
      vus: 70,
      duration: '10m',
      exec: 'browseProducts',
    },
    search: {
      executor: 'constant-vus',
      vus: 20,
      duration: '10m',
      exec: 'searchProducts',
    },
    order: {
      executor: 'constant-vus',
      vus: 10,
      duration: '10m',
      exec: 'createOrder',
    },
  },
};
```

### 안티패턴 3: 캐시 워밍 상태로 테스트

처음 부하를 걸 때와 캐시가 충분히 찬 후의 성능이 다르다. Cold Start와 Warm Start 모두 측정해야 한다.

### 안티패턴 4: 프로덕션과 다른 환경에서만 테스트

테스트 환경이 프로덕션의 1/10 스펙이라면, 결과를 단순 환산하면 오차가 크다. 가능하면 프로덕션과 동일한 스펙의 스테이징 환경에서 테스트해야 한다.

---

## 8. 부하 테스트 결과 보고서 작성

결과 보고서는 개발자뿐 아니라 PO, 경영진도 읽는다. 숫자만 나열하지 말고, 의미와 조치 방안을 함께 제시해야 한다.

### 보고서 구조

```markdown
# 부하 테스트 결과 보고서

## 1. 테스트 개요
- 테스트 일시: 2026-03-17
- 테스트 환경: AWS EC2 t3.xlarge × 3 (API), RDS db.r6g.large
- 테스트 도구: k6 v0.45
- 목표: 동시 접속자 500명, TPS 300 이상, p95 < 500ms

## 2. 시나리오
| 시나리오 | 비율 | VU 수 |
|---------|------|-------|
| 상품 조회 | 70% | 350 |
| 검색 | 20% | 100 |
| 주문 생성 | 10% | 50 |

## 3. 결과 요약
| 지표 | 목표 | 실제 | 판정 |
|------|------|------|------|
| TPS | 300 | 287 | ✗ 미달 |
| p95 응답시간 | < 500ms | 620ms | ✗ 초과 |
| p99 응답시간 | < 1000ms | 1850ms | ✗ 초과 |
| 에러율 | < 1% | 0.3% | ✓ 통과 |

## 4. 병목 분석
- DB 커넥션 풀 고갈 (HikariCP pool timeout 다수 발생)
- 주문 생성 API의 재고 확인 쿼리에서 테이블 풀스캔 발생
- p99 급등 원인: Full GC 발생 시 2~3초 응답 지연

## 5. 조치 계획
| 우선순위 | 조치 사항 | 담당자 | 기한 |
|---------|----------|-------|------|
| High | 재고 확인 쿼리 인덱스 추가 | 김개발 | 3/20 |
| High | HikariCP 풀 크기 10 → 30 조정 | 이인프라 | 3/18 |
| Medium | JVM Heap 4GB → 8GB 조정, GC 튜닝 | 박백엔드 | 3/22 |
| Low | 읽기 쿼리 Read Replica 분리 | 김개발 | 3/30 |

## 6. 재테스트 일정
조치 완료 후 2026-03-25 동일 조건으로 재측정 예정
```

### Grafana 대시보드 연동

k6와 InfluxDB, Grafana를 연동하면 실시간 시각화가 가능하다.

```yaml
# docker-compose.yml
version: '3'
services:
  influxdb:
    image: influxdb:1.8
    ports:
      - "8086:8086"
    environment:
      INFLUXDB_DB: k6
      INFLUXDB_HTTP_AUTH_ENABLED: "false"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

```bash
# k6 실행 시 InfluxDB로 전송
k6 run \
  --out influxdb=http://localhost:8086/k6 \
  -e BASE_URL=https://api.example.com \
  load-test.js
```

Grafana에서 k6 공식 대시보드(ID: 2587)를 Import하면 TPS, Latency 퍼센타일, VU 수를 실시간으로 확인할 수 있다.

---

## 9. CI/CD 파이프라인 통합

부하 테스트를 배포 파이프라인에 포함하면 성능 회귀를 자동으로 탐지할 수 있다.

```yaml
# .github/workflows/load-test.yml
name: Load Test

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: tests/load-test.js
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}

      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: k6-results
          path: results.json
```

Threshold를 초과하면 k6가 exit code 99를 반환하므로, 파이프라인이 자동으로 실패한다.

```javascript
// thresholds 위반 시 자동 배포 차단
export const options = {
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 이 조건 실패 → exit code 99
    http_req_failed: ['rate<0.01'],
  },
};
```

---

부하 테스트는 한 번 하고 끝나는 작업이 아니다.

코드가 바뀔 때마다, 인프라가 바뀔 때마다 반복해야 비로소 성능 회귀를 조기에 잡을 수 있다. 지금 당장 로컬에서 k6 한 줄로 시작해보자. 처음 테스트 결과를 보는 순간, 예상과 다른 병목에 놀라게 될 것이다.

---

## 참고 자료

1. [k6 공식 문서](https://k6.io/docs/) — 시나리오, 메트릭, 임계값 설정 레퍼런스
2. [Gatling 공식 문서](https://gatling.io/docs/gatling/) — Scala DSL 시뮬레이션 작성 가이드
3. [Google SRE Book — Chapter 20: Load Testing](https://sre.google/sre-book/load-testing/) — SRE 관점의 부하 테스트 철학
4. [nGrinder GitHub](https://github.com/naver/ngrinder) — 네이버 오픈소스 분산 부하 테스트 플랫폼
5. [HdrHistogram — The Coordinated Omission Problem](https://psy-lob-saw.blogspot.com/2015/02/hdrhistogram-better-latency-capture.html) — Latency 측정의 함정과 해결책
6. [Grafana k6 Dashboard](https://grafana.com/grafana/dashboards/2587) — k6 + InfluxDB 실시간 대시보드 템플릿
