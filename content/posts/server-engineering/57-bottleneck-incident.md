---
title: "[성능 최적화와 안정성] 5편 — 병목 분석과 장애 대응: 서버는 어떻게 죽고, 어떻게 버티는가"
date: 2026-03-17T18:02:00+09:00
draft: false
tags: ["병목 분석", "서킷 브레이커", "타임아웃", "Resilience4j", "서버"]
series: ["성능 최적화와 안정성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 8
summary: "CPU/IO/메모리/네트워크 병목 진단, 커넥션 풀 고갈·스레드 스타베이션·큐 누적 대응, 타임아웃 계층 설계, 서킷 브레이커(Closed/Open/Half-Open)와 Resilience4j까지"
---

서버는 갑자기 죽지 않는다. 대부분의 장애는 수십 분, 때로는 수 시간 전부터 조용히 경고 신호를 보낸다. CPU 사용률이 서서히 올라가고, 응답 지연이 p99부터 늘어나고, 커넥션 풀이 조금씩 줄어든다. 문제는 그 신호를 읽지 못하거나, 읽었더라도 어디서 터질지 몰라 대응이 늦는다는 것이다. 이 글은 서버가 어떻게 죽는지를 해부하고, 죽기 전에 어떻게 버티게 만드는지를 다룬다.

---

## 1. 병목은 어디서 오는가

병목(bottleneck)이란 시스템에서 처리 속도가 가장 느린 지점이다.

한 지점의 처리량이 전체 처리량을 결정한다. 그 지점을 찾지 못하면, 다른 곳을 아무리 튜닝해도 전체 성능은 개선되지 않는다.

```text
응답 느림 / 에러 증가
        │
        ▼
 CPU 사용률 높음? ──Yes──► top/async-profiler → CPU 병목
        │
        No
        │
        ▼
 GC 빈번 / 힙 높음? ──Yes──► jstat/힙덤프 → 메모리 병목
        │
        No
        │
        ▼
 iostat %util 높음? ──Yes──► 슬로우 쿼리/디스크 → I/O 병목
        │
        No
        │
        ▼
 커넥션 풀 대기? ──Yes──► HikariCP 설정 → 커넥션 풀 고갈
        │
        No
        │
        ▼
 TIME_WAIT 많음? ──Yes──► keep-alive/tcp_tw_reuse → 네트워크 병목
```

### CPU 병목

CPU 병목은 연산 집약적 작업이 많거나, 컨텍스트 스위칭이 과도할 때 발생한다.

`top` 또는 `htop`으로 전체 CPU 사용률을 먼저 확인하고, `us`(user), `sy`(system), `wa`(iowait) 비율을 분석한다.

```bash
# CPU 사용률 상세 확인
top -b -n 1 | head -20

# 프로세스별 CPU 사용 상위 10개
ps aux --sort=-%cpu | head -10

# 스레드별 CPU 사용 확인 (Java 프로세스 PID=12345)
top -H -p 12345

# 특정 스레드 NID를 16진수로 변환하여 스택 추적
printf '%x\n' <TID>
jstack 12345 | grep -A 30 "<NID>"
```

`us`가 높으면 애플리케이션 로직 문제다. 정규식, 암호화, JSON 파싱 같은 CPU 집약 작업을 의심하라.

`sy`가 높으면 시스템 콜이 많은 것이다. 소켓 I/O, 파일 I/O 빈도를 줄이거나 배치로 묶어야 한다.

```bash
# 시스템 콜 추적
strace -p <PID> -c -f 2>&1 | head -30

# perf로 핫스팟 분석 (Linux)
perf top -p <PID>
perf record -g -p <PID> sleep 30
perf report
```

Java 애플리케이션이라면 JFR(Java Flight Recorder)이 가장 정밀하다.

```bash
# JFR 기록 시작
jcmd <PID> JFR.start duration=60s filename=/tmp/app.jfr settings=profile

# 분석은 JDK Mission Control 또는 jfr 커맨드로
jfr print --events CPULoad,ThreadAllocationStatistics /tmp/app.jfr
```

**안티패턴**: CPU가 높다고 무조건 스레드 풀 크기를 늘리는 것. CPU 바운드 작업에 스레드를 추가하면 컨텍스트 스위칭만 증가한다.

### I/O 병목

I/O 병목은 디스크나 네트워크 대기로 인해 스레드가 블로킹되는 상황이다.

`wa` 값이 높으면 I/O 대기가 많다는 의미다. `iostat`으로 디스크 세부 지표를 확인한다.

```bash
# 디스크 I/O 실시간 모니터링 (1초 간격)
iostat -xz 1

# 핵심 지표
# %util: 디스크 포화도 (100%에 가까울수록 병목)
# await: 평균 I/O 대기 시간 (ms)
# r/s, w/s: 초당 읽기/쓰기 요청 수

# 어떤 프로세스가 I/O를 유발하는지
iotop -o -d 2
```

`%util`이 70% 이상이면 디스크가 병목이다. 읽기 캐시를 늘리거나, 핫 데이터를 SSD로 이동하거나, 쓰기를 비동기화해야 한다.

데이터베이스 I/O 병목은 슬로우 쿼리 로그와 `EXPLAIN ANALYZE`로 찾는다.

```sql
-- PostgreSQL 슬로우 쿼리 확인
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- 실행 계획 확인
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 12345;
```

**안티패턴**: N+1 쿼리는 대표적인 I/O 병목이다. 100개 상품 목록을 조회하면서 각 상품의 카테고리를 별도 쿼리로 가져오면 101번의 DB 왕복이 발생한다.

```java
// 나쁜 예: N+1 문제
List<Product> products = productRepo.findAll(); // 1번
for (Product p : products) {
    Category cat = categoryRepo.findById(p.getCategoryId()); // N번
}

// 좋은 예: JOIN FETCH로 한 번에
@Query("SELECT p FROM Product p JOIN FETCH p.category")
List<Product> findAllWithCategory();
```

### 메모리 압박

메모리 병목은 GC 빈도 증가, 스왑 사용, OOM으로 이어진다.

```bash
# 메모리 사용 현황
free -h
vmstat -s

# 스왑 사용 여부 확인 (스왑 사용 = 성능 위험 신호)
swapon --show
cat /proc/swaps

# Java 힙 덤프 분석
jmap -heap <PID>
jmap -histo:live <PID> | head -30

# GC 로그 활성화 (JVM 옵션)
-Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=20m
```

GC 로그에서 Full GC 빈도와 소요 시간을 확인한다. Full GC가 초당 1회 이상 발생하거나, 소요 시간이 1초를 넘으면 심각한 수준이다.

```bash
# GC Easy (온라인 도구) 또는 GCViewer로 분석
# gc.log를 업로드하면 throughput, pause time 시각화 제공

# 힙 덤프 생성 및 MAT(Memory Analyzer Tool)로 분석
jmap -dump:format=b,file=/tmp/heap.hprof <PID>
```

**안티패턴**: 메모리 릭의 흔한 원인은 `static` 컬렉션에 데이터를 계속 추가하거나, 캐시에 만료 정책을 설정하지 않는 것이다.

```java
// 나쁜 예: 무한 증가하는 static 캐시
private static final Map<String, Object> cache = new HashMap<>();

// 좋은 예: 만료 정책이 있는 캐시
private final Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build();
```

### 네트워크 포화

네트워크 병목은 대역폭 포화, 지연 증가, 패킷 드롭으로 나타난다.

```bash
# 네트워크 인터페이스 통계
ifstat -i eth0 1

# 네트워크 연결 상태 요약
ss -s

# TIME_WAIT 상태 연결 수 확인
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

# 패킷 드롭 및 오류 확인
netstat -s | grep -E 'failed|error|drop'

# 대역폭 실시간 모니터링
iftop -i eth0
```

`TIME_WAIT` 연결이 수만 개라면 커넥션 재사용(Keep-Alive)이 안 되는 것이다. 짧은 연결을 반복하는 구조는 포트 고갈로 이어진다.

```bash
# TIME_WAIT 소켓 빠른 재사용 활성화
sysctl -w net.ipv4.tcp_tw_reuse=1

# SYN 백로그 증가
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

---

## 2. 자원 고갈 패턴과 대응

### 커넥션 풀 고갈

커넥션 풀 고갈은 가장 흔한 장애 원인 중 하나다.

DB 커넥션 풀이 소진되면, 이후 요청들은 커넥션 획득을 기다리며 블로킹된다. 대기 요청이 쌓이면 HTTP 스레드도 소진되고, 결국 전체 서비스가 응답 불능이 된다.

```java
// HikariCP 커넥션 풀 설정 (Spring Boot)
spring:
  datasource:
    hikari:
      maximum-pool-size: 20          # 최대 커넥션 수
      minimum-idle: 5                # 최소 유휴 커넥션
      connection-timeout: 3000       # 커넥션 획득 대기 최대 시간 (ms)
      idle-timeout: 600000           # 유휴 커넥션 해제 시간 (10분)
      max-lifetime: 1800000          # 커넥션 최대 수명 (30분)
      leak-detection-threshold: 5000 # 커넥션 릭 감지 임계치 (ms)
```

`leak-detection-threshold`는 반드시 설정해야 한다. 커넥션을 획득하고 반환하지 않는 버그를 조기에 발견할 수 있다.

```java
// 커넥션 풀 메트릭 모니터링 (Micrometer + Prometheus)
@Bean
public DataSource dataSource(HikariConfig config) {
    HikariDataSource ds = new HikariDataSource(config);
    // HikariCP는 자동으로 Micrometer에 메트릭 등록
    // hikaricp_connections_active, hikaricp_connections_pending 등
    return ds;
}
```

Grafana에서 `hikaricp_connections_pending` 지표가 지속적으로 0 이상이면, 풀 크기가 부족하거나 커넥션이 오래 점유되는 것이다.

**커넥션 풀 크기 공식**: 일반적으로 `(core count * 2) + effective_spindle_count`가 시작점이다. I/O 대기 시간이 길수록 적절한 풀 크기는 커진다. CPU가 4코어이고 SSD 1개라면 `4 * 2 + 1 = 9`개가 시작점이다. 이 수치보다 무조건 크게 잡는 것은 오히려 DB 서버 측 컨텍스트 스위칭을 증가시켜 전체 처리량을 낮출 수 있다.

**안티패턴**: 커넥션 풀을 무조건 크게 잡는 것. DB 서버의 `max_connections`는 고정 자원이다. 각 애플리케이션 인스턴스가 100개씩 연결을 잡으면, 10개 인스턴스만 되어도 DB가 1000개 연결을 감당해야 한다.

### 스레드 스타베이션

스레드 스타베이션은 모든 스레드가 특정 작업(주로 I/O 대기)으로 묶여, 새 요청을 처리할 스레드가 없는 상태다.

```bash
# JVM 스레드 덤프로 상태 확인
jstack <PID> > /tmp/thread_dump.txt

# BLOCKED, WAITING 상태 스레드 수 확인
grep -E "java.lang.Thread.State: (BLOCKED|WAITING)" /tmp/thread_dump.txt | wc -l

# 어떤 락/자원을 기다리는지 확인
grep -A 5 "WAITING" /tmp/thread_dump.txt | grep -E "waiting|locked"
```

스레드 덤프에서 모든 스레드가 `WAITING on java.sql.Connection`을 기다린다면, 커넥션 풀 고갈이 스레드 스타베이션의 원인이다.

```java
// Tomcat 스레드 풀 설정
server:
  tomcat:
    threads:
      max: 200          # 최대 스레드 수
      min-spare: 10     # 최소 유지 스레드
    accept-count: 100   # 대기 큐 크기 (이 이상은 거부)
    connection-timeout: 20000
```

`accept-count`를 무한정 늘리는 것은 위험하다. 대기 큐가 차는 것은 시스템이 이미 포화 상태라는 신호다. 대기를 쌓는 것보다 빠른 실패(fast fail)가 낫다.

### 큐 누적

메시지 큐나 작업 큐가 누적되면 처리 지연이 폭증한다.

```java
// Kafka 컨슈머 랙 확인
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group my-consumer-group

# 출력: LAG 컬럼이 지속적으로 증가하면 컨슈머가 프로듀서를 따라잡지 못하는 것
```

컨슈머 랙이 누적되는 원인은 세 가지다: 처리 로직이 느리거나, 파티션 수보다 컨슈머가 적거나, 다운스트림 DB/API가 병목이다.

```java
// Spring Kafka 컨슈머 병렬화
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setConcurrency(3); // 파티션 수와 맞추거나 그 이하로
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);
    return factory;
}
```

**안티패턴**: 큐 처리 중 DB 쓰기를 건당 commit하는 것. 1만 건을 처리할 때 1만 번 커밋하는 대신, 배치로 묶어서 처리하면 처리량이 수십 배 개선된다.

```java
// 나쁜 예: 건당 저장
@KafkaListener(topics = "events")
public void process(String message) {
    repository.save(parse(message)); // 건당 DB 커밋
}

// 좋은 예: 배치 저장
@KafkaListener(topics = "events")
public void processBatch(List<ConsumerRecord<String, String>> records) {
    List<Event> events = records.stream()
        .map(r -> parse(r.value()))
        .collect(Collectors.toList());
    repository.saveAll(events); // 한 번에 저장
}
```

---

## 3. 타임아웃 계층 설계

타임아웃은 시스템이 무한정 대기하지 않도록 하는 안전장치다.

타임아웃이 없으면, 느린 다운스트림 하나가 전체 스레드 풀을 잠식할 수 있다. 모든 I/O 지점에는 반드시 타임아웃을 설정해야 한다.

### 타임아웃의 4계층

```
클라이언트
    │ ← 전체 요청 타임아웃 (Overall / Total)
    │
[로드 밸런서]
    │ ← 유휴 연결 타임아웃 (Idle Connection)
    │
[애플리케이션 서버]
    │ ← 연결 타임아웃 (Connect Timeout)
    │   읽기 타임아웃 (Read Timeout)
    │   쓰기 타임아웃 (Write Timeout)
    │
[DB / 외부 API]
```

각 계층의 타임아웃은 상위 계층보다 짧아야 한다. 하위 계층이 먼저 실패해야 상위 계층이 의미 있는 재시도를 할 수 있다.

### HTTP 클라이언트 타임아웃 (RestClient / WebClient)

```java
// Spring Boot 3.x RestClient 타임아웃 설정
@Bean
public RestClient restClient() {
    HttpComponentsClientHttpRequestFactory factory =
        new HttpComponentsClientHttpRequestFactory();

    factory.setConnectTimeout(Duration.ofSeconds(2));    // TCP 연결 수립
    factory.setConnectionRequestTimeout(Duration.ofSeconds(1)); // 풀에서 커넥션 획득
    factory.setReadTimeout(Duration.ofSeconds(5));       // 응답 데이터 수신

    return RestClient.builder()
        .requestFactory(factory)
        .build();
}

// WebClient (Reactive) 타임아웃 설정
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
        .responseTimeout(Duration.ofSeconds(5))
        .doOnConnected(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(5))
            .addHandlerLast(new WriteTimeoutHandler(2)));

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
```

### DB 타임아웃

```java
// HikariCP + JPA 타임아웃 설정
spring:
  datasource:
    hikari:
      connection-timeout: 3000      # 커넥션 획득 대기
      validation-timeout: 1000      # 커넥션 유효성 검사
  jpa:
    properties:
      hibernate:
        jdbc:
          fetch_size: 100
        # 쿼리 실행 타임아웃 (초)
        javax.persistence.query.timeout: 5000
```

```java
// 개별 쿼리 타임아웃 (Spring Data JPA)
@QueryHints(value = @QueryHint(
    name = "javax.persistence.query.timeout",
    value = "3000"  // 3초
))
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);
```

### 타임아웃 값 설정 가이드

타임아웃은 SLO(Service Level Objective)에서 역산한다.

예를 들어 p99 응답시간이 200ms인 서비스라면, 개별 외부 API 호출 타임아웃은 300-500ms가 합리적이다. 단, 전체 요청 타임아웃보다는 짧아야 한다.

```
전체 요청 타임아웃:  10s
  └─ 외부 API 호출:  3s
  └─ DB 쿼리:        2s
  └─ 캐시 조회:      100ms
```

**안티패턴**: 타임아웃을 너무 길게 설정하는 것. "혹시 느릴 수도 있으니" 30초, 60초로 설정하면, 장애 전파 속도가 느려질 뿐 아니라 스레드가 수십 초간 묶인다.

**안티패턴**: 타임아웃 없이 기본값 사용. Java `HttpURLConnection`의 기본 연결 타임아웃은 무한대다.

---

## 4. 서킷 브레이커

서킷 브레이커는 전기 회로 차단기에서 이름을 따왔다. 장애가 연쇄 전파되는 것을 막는 패턴이다.

느린 또는 실패하는 다운스트림 서비스를 계속 호출하면, 호출자의 스레드와 커넥션이 낭비된다. 서킷 브레이커는 일정 시간 동안 호출을 차단하여 다운스트림이 회복할 시간을 준다.

### 세 가지 상태

**Closed (정상)**: 모든 요청을 통과시킨다. 실패율이 임계치를 넘으면 Open으로 전환.

**Open (차단)**: 모든 요청을 즉시 실패시킨다 (fallback 실행). 대기 시간 후 Half-Open으로 전환.

**Half-Open (탐색)**: 제한된 수의 요청만 통과. 성공하면 Closed로, 실패하면 다시 Open으로 전환. Half-Open은 다운스트림이 실제로 회복됐는지 조심스럽게 확인하는 단계다. Open 상태에서 바로 Closed로 가면 아직 불안정한 서비스에 전체 트래픽이 몰려 재차 장애를 일으킬 수 있다.

```text
                    실패율 초과 (예: 50%)
         ┌─────────────────────────────────┐
         ▼                                 │
    [CLOSED]                           [OPEN]
    (정상 통과)  ──실패율 초과──►  (즉시 실패/폴백)
         ▲                                 │
         │                          대기 시간 경과
         │                          (예: 10초)
         │                                 ▼
         │                         [HALF-OPEN]
         │                      (제한된 요청만 통과)
         │                                 │
         └──────── 성공 ──────────────────┘
                                           │
                         실패 ─────────►[OPEN]
```

### Resilience4j 설정

Resilience4j는 Java 생태계의 표준 resilience 라이브러리다.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        # 슬라이딩 윈도우 설정
        sliding-window-type: COUNT_BASED     # COUNT_BASED or TIME_BASED
        sliding-window-size: 10              # 최근 10개 요청 기준
        minimum-number-of-calls: 5           # 최소 5개 이후 통계 계산

        # Open 전환 조건
        failure-rate-threshold: 50           # 실패율 50% 이상이면 Open
        slow-call-rate-threshold: 80         # 느린 호출 80% 이상도 Open
        slow-call-duration-threshold: 2000   # 2초 이상이면 느린 호출

        # 상태 전환 설정
        wait-duration-in-open-state: 10s     # Open 유지 시간
        permitted-number-of-calls-in-half-open-state: 3  # Half-Open 허용 요청 수
        automatic-transition-from-open-to-half-open-enabled: true

        # 예외 처리
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.BusinessException  # 비즈니스 예외는 실패로 카운트 안 함
```

```java
// 서킷 브레이커 적용
@Service
public class PaymentService {

    private final RestClient restClient;

    // 어노테이션 방식
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResult charge(PaymentRequest request) {
        return restClient.post()
            .uri("/charge")
            .body(request)
            .retrieve()
            .body(PaymentResult.class);
    }

    // 폴백: 서킷이 Open이거나 예외 발생 시 호출
    public PaymentResult paymentFallback(PaymentRequest request, Exception ex) {
        log.warn("Payment service unavailable, using fallback. reason={}", ex.getMessage());
        // 큐에 저장하거나 임시 응답 반환
        return PaymentResult.pending(request.getOrderId());
    }
}
```

```java
// 프로그래매틱 방식 (세밀한 제어 필요 시)
@Service
public class PaymentService {

    private final CircuitBreakerRegistry registry;
    private final CircuitBreaker circuitBreaker;

    public PaymentService(CircuitBreakerRegistry registry) {
        this.registry = registry;
        this.circuitBreaker = registry.circuitBreaker("paymentService");

        // 상태 변경 이벤트 구독
        circuitBreaker.getEventPublisher()
            .onStateTransition(event ->
                log.warn("Circuit breaker state changed: {} -> {}",
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState())
            );
    }

    public PaymentResult charge(PaymentRequest request) {
        return circuitBreaker.executeSupplier(() -> callPaymentApi(request));
    }
}
```

### Retry와 함께 사용하기

서킷 브레이커만으로는 부족하다. 일시적 오류는 재시도로 해결해야 한다.

```yaml
resilience4j:
  retry:
    instances:
      paymentService:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
        ignore-exceptions:
          - com.example.PaymentDeclinedException  # 결제 거절은 재시도 불필요

  circuitbreaker:
    instances:
      paymentService:
        # ... 위 설정과 동일
```

```java
// Retry + CircuitBreaker 조합 (순서 중요: Retry가 바깥, CB가 안)
@Retry(name = "paymentService")
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResult charge(PaymentRequest request) {
    return callPaymentApi(request);
}
```

순서는 중요하다. Retry가 CircuitBreaker를 감싸야 한다. CircuitBreaker가 Open 상태일 때 Retry가 계속 시도하는 것을 막기 위해서다.

### Bulkhead 패턴

서킷 브레이커와 함께 자주 사용되는 Bulkhead(격벽) 패턴은, 특정 서비스 호출에 사용할 수 있는 동시 실행 수를 제한한다.

```yaml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 10      # 최대 동시 호출 수
        max-wait-duration: 500ms      # 슬롯 대기 최대 시간
```

```java
@Bulkhead(name = "paymentService", type = Bulkhead.Type.SEMAPHORE)
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResult charge(PaymentRequest request) {
    return callPaymentApi(request);
}
```

Bulkhead가 없으면, 결제 API가 느려질 때 모든 스레드가 결제 호출에 묶여 다른 기능까지 영향받는다.

### 메트릭 모니터링

Resilience4j는 Micrometer와 통합되어 자동으로 메트릭을 노출한다.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  health:
    circuitbreakers:
      enabled: true
```

```
# Prometheus 쿼리 예시
# 서킷 브레이커 상태 (0=CLOSED, 1=OPEN, 2=HALF_OPEN)
resilience4j_circuitbreaker_state{name="paymentService"}

# 실패율
resilience4j_circuitbreaker_failure_rate{name="paymentService"}

# 느린 호출 비율
resilience4j_circuitbreaker_slow_call_rate{name="paymentService"}
```

Grafana 알림을 설정하여 서킷이 Open 상태로 전환되면 즉시 알림을 받도록 한다.

**안티패턴**: 폴백에서 또 다른 느린 외부 호출을 하는 것. 폴백은 빠르고 단순해야 한다. 캐시 반환, 기본값 반환, 큐 저장 정도가 적절하다.

**안티패턴**: 모든 예외를 실패로 카운트하는 것. `404 Not Found`나 `400 Bad Request`는 서비스 장애가 아니라 클라이언트 오류다. `ignore-exceptions`로 비즈니스 예외를 제외해야 한다.

---

## 5. 장애 대응 체크리스트

장애가 발생했을 때 침착하게 대응하려면 사전에 절차를 정해두어야 한다.

### 즉각 대응 (0-5분)

1. 영향 범위 파악: 어떤 서비스가 영향받는가?
2. 증상 수집: 에러율, 응답 지연, 로그 확인
3. 최근 변경 사항 확인: 배포, 설정 변경, 인프라 작업

```bash
# 에러율 급증 확인 (nginx 로그 기준)
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 애플리케이션 에러 로그 실시간 모니터링
tail -f /var/log/app/error.log | grep -E "ERROR|WARN"

# 최근 배포 확인
kubectl rollout history deployment/my-app
```

### 격리 및 완화 (5-30분)

4. 장애 서비스 격리: 로드 밸런서에서 제외 또는 서킷 브레이커 강제 Open
5. 롤백 검토: 최근 배포가 원인이면 즉시 롤백
6. 트래픽 제한: Rate Limiting 강화, 불필요한 기능 비활성화

```bash
# Kubernetes 롤백
kubectl rollout undo deployment/my-app

# 특정 파드 트래픽 제외 (레이블 제거)
kubectl label pod <pod-name> app-

# 서킷 브레이커 강제 Open (Actuator)
curl -X POST http://localhost:8080/actuator/circuitbreakers/paymentService/state \
  -H "Content-Type: application/json" \
  -d '{"state": "FORCED_OPEN"}'
```

### 근본 원인 분석 (장애 후)

7. 타임라인 재구성: 언제 무엇이 어떻게 변했는가
8. 5 Whys 분석: 증상에서 시작하여 근본 원인까지
9. 재발 방지: 모니터링 추가, 코드 수정, 프로세스 개선

---

## 마치며

병목 분석과 장애 대응은 사후 작업이 아니다.

CPU/IO/메모리/네트워크 지표를 평소에 기준선(baseline)으로 파악해두고, 이상 징후를 자동으로 감지할 수 있어야 한다. 타임아웃과 서킷 브레이커는 코드를 작성할 때부터 설계에 포함시켜야 한다.

"이 서비스가 죽으면 내 서비스는 어떻게 동작하는가?"라는 질문을 항상 먼저 던져야 한다. 그 답이 "함께 죽는다"라면, 아직 할 일이 남아 있다.

---

## 참고 자료

1. **Resilience4j 공식 문서** — https://resilience4j.readme.io/docs/getting-started
2. **HikariCP — About Pool Sizing** — https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
3. **Netflix Tech Blog: Fault Tolerance in a High Volume, Distributed System** — https://netflixtechblog.com/fault-tolerance-in-a-high-volume-distributed-system-91ab4faae74a
4. **USE Method: Linux Performance Checklist** — https://www.brendangregg.com/USEmethod/use-linux.html
5. **Martin Fowler: CircuitBreaker** — https://martinfowler.com/bliki/CircuitBreaker.html
6. **Google SRE Book: Chapter 22 — Addressing Cascading Failures** — https://sre.google/sre-book/addressing-cascading-failures/
