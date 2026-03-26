---
title: "[성능 최적화와 안정성] 6편 — Graceful Shutdown과 무중단 운영"
date: 2026-03-17T18:01:00+09:00
draft: false
tags: ["Graceful Shutdown", "Connection Drain", "시그널", "무중단", "서버"]
series: ["성능 최적화와 안정성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 8
summary: "Graceful Shutdown 구현(요청 완료 대기, 리소스 반납), 커넥션 Drain과 인플라이트 요청 처리, 프로세스 시그널 처리와 상태 저장 서버 종료 절차까지"
---

배포 버튼을 누르는 순간, 누군가의 요청은 절반쯤 처리되고 있다. 데이터베이스 트랜잭션이 열려 있고, 파일은 아직 기록 중이며, 외부 API 호출이 응답을 기다리고 있다. 이 순간 서버를 그냥 죽이면 어떻게 될까? 클라이언트는 연결이 끊겼다는 에러를 받고, 트랜잭션은 롤백되거나 커밋되지 않은 채 남고, 결제 시스템이라면 돈이 오가다 멈춰버릴 수도 있다. Graceful Shutdown은 이 문제에 대한 공학적 해답이다. 서버를 "우아하게" 종료한다는 것은 단순히 프로세스를 깔끔하게 끄는 이야기가 아니라, 서비스 연속성과 데이터 무결성을 보장하는 엔지니어링 철학이다.

---

## Graceful Shutdown이란 무엇인가

**Graceful Shutdown**은 서버 프로세스가 종료 신호를 받았을 때, 즉시 죽는 대신 진행 중인 작업을 안전하게 마무리하고 리소스를 정리한 후 종료하는 패턴이다.

반대 개념인 **Hard Kill(Forceful Shutdown)**은 SIGKILL처럼 프로세스를 즉시 강제 종료한다.

Graceful Shutdown이 보장해야 하는 항목은 크게 세 가지다.

1. **인플라이트 요청 완료**: 현재 처리 중인 HTTP 요청, RPC 호출 등이 응답을 반환할 때까지 기다린다.
2. **신규 요청 거부**: 종료 진행 중에는 새로운 커넥션을 받지 않는다.
3. **리소스 정리**: DB 커넥션 풀 반납, 메시지 큐 커밋, 파일 핸들 닫기 등을 수행한다.

### 왜 Hard Kill은 위험한가

SIGKILL은 운영체제 수준에서 프로세스를 즉시 제거한다.

JVM 기반 서버라면 finalizer가 실행되지 않고, DB 커넥션이 풀에 반납되지 않으며, 커넥션 풀 서버(예: PgBouncer, HikariCP)는 연결이 끊어졌는지조차 인지하지 못한 채 invalid 커넥션을 보유하게 된다.

메시지 큐를 소비 중이었다면 메시지가 ack 없이 사라져 중복 처리 혹은 유실로 이어진다.

---

## 프로세스 시그널 처리

### Unix/Linux 시그널 종류

서버 종료는 대부분 시그널로 시작된다.

| 시그널 | 번호 | 의미 | 기본 동작 |
|--------|------|------|-----------|
| SIGTERM | 15 | 정상 종료 요청 | 프로세스 종료 |
| SIGINT | 2 | 인터럽트 (Ctrl+C) | 프로세스 종료 |
| SIGHUP | 1 | 터미널 연결 끊김 / 설정 재로드 | 종료 or 무시 |
| SIGKILL | 9 | 강제 종료 | 즉시 종료 (캐치 불가) |
| SIGUSR1/2 | 30/31 | 사용자 정의 | 무시 |

**SIGTERM이 표준이다.** `systemd`, `Docker`, `Kubernetes` 모두 컨테이너/프로세스 종료 시 SIGTERM을 먼저 보낸다.

일정 시간 안에 종료되지 않으면 SIGKILL을 보낸다(기본값: Docker 10초, K8s 30초).

### 시그널 핸들러 등록 (Shell)

```bash
#!/bin/bash

cleanup() {
    echo "[$(date)] SIGTERM received. Starting graceful shutdown..."
    # 여기에 정리 로직
    exit 0
}

trap cleanup SIGTERM SIGINT

# 메인 프로세스
./my-server &
SERVER_PID=$!

wait $SERVER_PID
```

`trap` 명령으로 시그널을 가로채고 cleanup 함수를 실행한다.

백그라운드 프로세스를 `wait`로 기다리면 시그널이 쉘 스크립트에 전달된다.

### 주의: Docker 컨테이너에서의 PID 1 문제

Docker 컨테이너에서 쉘 스크립트가 PID 1이 되면 시그널 처리가 꼬인다.

```dockerfile
# 잘못된 방식 - 쉘이 PID 1이 되어 SIGTERM을 자식에게 전달하지 않음
CMD ["sh", "-c", "java -jar app.jar"]

# 올바른 방식 - Java 프로세스가 직접 PID 1
CMD ["java", "-jar", "app.jar"]

# 또는 tini 사용
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["java", "-jar", "app.jar"]
```

`exec` 형식(`CMD ["..."]`)을 쓰거나 `tini`같은 init 프로세스를 사용해야 SIGTERM이 올바르게 애플리케이션에 전달된다.

---

## Java/Spring Boot Graceful Shutdown 구현

### Spring Boot 2.3+ 내장 지원

Spring Boot 2.3부터 Graceful Shutdown이 공식 지원된다.

```yaml
# application.yml
server:
  shutdown: graceful  # 기본값은 immediate

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 대기 최대 시간
```

이 설정만으로 SIGTERM 수신 시 다음 동작이 자동으로 수행된다.

1. 새 요청 수신 거부 (HTTP 503 반환)
2. 진행 중인 요청 완료 대기 (최대 30초)
3. ApplicationContext 종료 및 빈 소멸

### ShutdownHook 직접 구현

Spring Boot의 내장 기능만으로 부족할 때는 `@PreDestroy`나 `DisposableBean`을 활용한다.

```java
@Component
@Slf4j
public class ApplicationShutdownHook implements DisposableBean {

    private final ThreadPoolTaskExecutor taskExecutor;
    private final DataSource dataSource;

    public ApplicationShutdownHook(
            ThreadPoolTaskExecutor taskExecutor,
            DataSource dataSource) {
        this.taskExecutor = taskExecutor;
        this.dataSource = dataSource;
    }

    @Override
    public void destroy() throws Exception {
        log.info("Graceful shutdown initiated...");

        // 1. 진행 중인 태스크 완료 대기
        taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        taskExecutor.setAwaitTerminationSeconds(30);
        taskExecutor.shutdown();

        log.info("All tasks completed. Releasing resources...");

        // 2. DB 커넥션 풀 정리
        if (dataSource instanceof HikariDataSource hikariDS) {
            hikariDS.close();
            log.info("HikariCP connection pool closed.");
        }

        log.info("Graceful shutdown complete.");
    }
}
```

### JVM ShutdownHook 등록

Spring 외부에서도 JVM 레벨 ShutdownHook을 직접 등록할 수 있다.

```java
public class Server {

    private static final AtomicBoolean shuttingDown = new AtomicBoolean(false);
    private static final CountDownLatch shutdownLatch = new CountDownLatch(1);

    public static void main(String[] args) throws InterruptedException {
        // ShutdownHook 등록 (SIGTERM, SIGINT 모두 처리)
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            log.info("Shutdown hook triggered.");
            shuttingDown.set(true);

            try {
                // 진행 중인 요청이 끝나기를 최대 30초 대기
                boolean finished = shutdownLatch.await(30, TimeUnit.SECONDS);
                if (!finished) {
                    log.warn("Shutdown timeout. Forcing shutdown.");
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            log.info("Server stopped.");
        }, "shutdown-hook"));

        // 서버 시작
        startServer();

        // 메인 스레드 블로킹 (서버가 종료될 때까지 대기)
        shutdownLatch.await();
    }

    public static boolean isShuttingDown() {
        return shuttingDown.get();
    }
}
```

### HTTP 요청 핸들러에서의 확인

```java
@RestController
public class HealthController {

    @GetMapping("/health")
    public ResponseEntity<String> health() {
        if (Server.isShuttingDown()) {
            // 로드밸런서가 이 서버를 제외하도록 503 반환
            return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                    .body("Shutting down");
        }
        return ResponseEntity.ok("OK");
    }
}
```

로드밸런서는 `/health` 엔드포인트를 주기적으로 체크하므로, 503을 반환하는 순간 트래픽이 다른 인스턴스로 이동한다.

---

## Go 서버 Graceful Shutdown 구현

### net/http 서버의 Shutdown()

Go 1.8부터 `http.Server`에 `Shutdown()` 메서드가 내장되어 있다.

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)

    server := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    // 시그널 채널 설정
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    // 서버를 고루틴으로 시작
    go func() {
        log.Println("Server starting on :8080")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // 시그널 대기
    sig := <-quit
    log.Printf("Received signal: %v. Starting graceful shutdown...", sig)

    // Shutdown: 새 커넥션 거부 + 기존 요청 완료 대기
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Graceful shutdown failed: %v", err)
    }

    log.Println("Server stopped gracefully.")
}
```

`server.Shutdown(ctx)`은 다음을 수행한다.

- 새 커넥션 수락 중단
- Keep-Alive 커넥션 종료 (idle 상태면 즉시, 요청 처리 중이면 완료 후)
- 컨텍스트 타임아웃 초과 시 강제 종료

### 리소스 정리를 포함한 완전한 예시

```go
type App struct {
    server *http.Server
    db     *sql.DB
    cache  *redis.Client
}

func (a *App) Shutdown(ctx context.Context) error {
    // 1. HTTP 서버 종료 (인플라이트 요청 대기)
    log.Println("Stopping HTTP server...")
    if err := a.server.Shutdown(ctx); err != nil {
        return fmt.Errorf("http server shutdown: %w", err)
    }

    // 2. DB 커넥션 풀 닫기
    log.Println("Closing database connections...")
    if err := a.db.Close(); err != nil {
        log.Printf("DB close error: %v", err)
    }

    // 3. Redis 커넥션 닫기
    log.Println("Closing cache connections...")
    if err := a.cache.Close(); err != nil {
        log.Printf("Cache close error: %v", err)
    }

    log.Println("All resources released.")
    return nil
}

func main() {
    app := &App{
        // 초기화...
    }

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    go app.server.ListenAndServe()

    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := app.Shutdown(ctx); err != nil {
        log.Fatalf("Shutdown error: %v", err)
    }
}
```

### 고루틴 추적 패턴

장시간 실행되는 백그라운드 고루틴이 있는 경우 `sync.WaitGroup`으로 추적한다.

```go
type WorkerPool struct {
    wg   sync.WaitGroup
    done chan struct{}
}

func (wp *WorkerPool) Start(n int) {
    for i := 0; i < n; i++ {
        wp.wg.Add(1)
        go func(id int) {
            defer wp.wg.Done()
            for {
                select {
                case <-wp.done:
                    log.Printf("Worker %d stopping", id)
                    return
                default:
                    // 작업 처리
                    wp.processJob()
                }
            }
        }(i)
    }
}

func (wp *WorkerPool) Shutdown(timeout time.Duration) {
    // done 채널 닫아서 모든 고루틴에게 종료 신호 전달
    close(wp.done)

    // 타임아웃과 함께 완료 대기
    done := make(chan struct{})
    go func() {
        wp.wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        log.Println("All workers stopped.")
    case <-time.After(timeout):
        log.Println("Worker shutdown timeout. Forcing stop.")
    }
}
```

---

## 커넥션 Drain

### Drain이란

**Connection Drain**은 로드밸런서가 특정 서버를 제거할 때, 기존에 그 서버로 향하던 커넥션이 자연스럽게 완료될 때까지 기다리는 메커니즘이다.

새 요청은 다른 서버로 라우팅되고, 기존 커넥션만 완료될 때까지 유지된다. Drain 없이 서버를 즉시 제거하면 처리 중이던 요청이 강제로 끊기고 클라이언트는 연결 오류를 받는다. 배포 중에 "간헐적 502 오류"가 발생한다면 Drain 설정이 누락됐을 가능성이 높다.

### 로드밸런서 레벨 Drain

**AWS ALB/ELB**는 `Deregistration Delay`(기본 300초)로 이를 구현한다.

```bash
# AWS CLI로 Deregistration Delay 설정
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:elasticloadbalancing:... \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

**NGINX** 업스트림 서버 제거 시 drain 모드:

```nginx
# NGINX Plus (상용)
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080 drain;  # 새 요청 안 보내고 기존 커넥션만 처리
}
```

오픈소스 NGINX에서는 `down` 플래그와 별도 스크립트로 유사하게 구현한다.

### 애플리케이션 레벨 Drain

로드밸런서에만 의존하지 않고 애플리케이션이 직접 Drain을 구현할 수 있다.

```java
@Component
public class ConnectionDrainFilter implements Filter {

    private final AtomicInteger activeRequests = new AtomicInteger(0);
    private volatile boolean draining = false;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        if (draining) {
            ((HttpServletResponse) response)
                .sendError(HttpServletResponse.SC_SERVICE_UNAVAILABLE,
                           "Server is draining");
            return;
        }

        activeRequests.incrementAndGet();
        try {
            chain.doFilter(request, response);
        } finally {
            activeRequests.decrementAndGet();
        }
    }

    public void startDrain() {
        draining = true;
        log.info("Draining started. Active requests: {}", activeRequests.get());
    }

    public int getActiveRequests() {
        return activeRequests.get();
    }
}
```

```java
@Component
public class GracefulShutdownCoordinator {

    private final ConnectionDrainFilter drainFilter;

    @EventListener(ContextClosingEvent.class)
    public void onShutdown() throws InterruptedException {
        // 1. Drain 시작 (새 요청 거부)
        drainFilter.startDrain();

        // 2. 기존 요청 완료 대기 (최대 30초)
        int maxWaitSeconds = 30;
        for (int i = 0; i < maxWaitSeconds; i++) {
            int active = drainFilter.getActiveRequests();
            if (active == 0) {
                log.info("All requests drained.");
                return;
            }
            log.info("Waiting for {} active requests...", active);
            Thread.sleep(1000);
        }

        log.warn("Drain timeout. Proceeding with shutdown.");
    }
}
```

### Keep-Alive 커넥션 처리

HTTP Keep-Alive 커넥션은 요청이 완료된 후에도 커넥션이 열려 있다.

서버 종료 시 응답 헤더에 `Connection: close`를 추가하면 클라이언트가 재사용하지 않고 커넥션을 닫는다.

```java
@Component
public class ShutdownConnectionCloseFilter implements Filter {

    private volatile boolean shuttingDown = false;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        chain.doFilter(request, response);

        if (shuttingDown) {
            ((HttpServletResponse) response)
                .setHeader("Connection", "close");
        }
    }
}
```

---

## 인플라이트 요청 처리 전략

### 타임아웃 설정의 중요성

Graceful Shutdown에는 반드시 최대 대기 시간(타임아웃)을 설정해야 한다.

타임아웃 없이 무한 대기하면 배포가 영원히 완료되지 않거나, 느린 요청 하나가 전체 종료를 막는다.

```java
// Spring Boot application.yml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

// 또는 프로그래밍 방식
@Bean
public GracefulShutdown gracefulShutdown() {
    return new GracefulShutdown();
}

@Bean
public ConfigurableServletWebServerFactory webServerFactory(
        GracefulShutdown gracefulShutdown) {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    factory.addConnectorCustomizers(gracefulShutdown);
    return factory;
}
```

### 장시간 실행 요청 처리

스트리밍, 파일 업로드, 폴링 등 장시간 요청은 별도로 처리해야 한다.

```go
func longRunningHandler(w http.ResponseWriter, r *http.Request) {
    // 요청 컨텍스트로 취소 전파 수신
    ctx := r.Context()

    for {
        select {
        case <-ctx.Done():
            // 클라이언트가 연결을 끊었거나 서버가 Shutdown() 중
            log.Println("Request context cancelled:", ctx.Err())
            return
        default:
            // 작업 처리
            if err := doWork(ctx); err != nil {
                return
            }
        }
    }
}
```

`http.Server.Shutdown()`을 호출하면 진행 중인 요청의 컨텍스트가 취소된다.

핸들러가 `r.Context()`를 잘 전파하고 있다면 자동으로 정리된다.

### 메시지 큐 컨슈머의 Graceful Shutdown

Kafka 컨슈머 예시:

```java
@Component
public class OrderEventConsumer {

    private final KafkaConsumer<String, OrderEvent> consumer;
    private volatile boolean running = true;

    @Async
    public void consume() {
        consumer.subscribe(List.of("order-events"));

        while (running) {
            ConsumerRecords<String, OrderEvent> records =
                consumer.poll(Duration.ofMillis(100));

            for (ConsumerRecord<String, OrderEvent> record : records) {
                try {
                    processOrder(record.value());
                } catch (Exception e) {
                    log.error("Failed to process record: {}", record.offset(), e);
                    // DLQ 전송 또는 재시도 로직
                }
            }

            // 처리 완료 후 커밋 (at-least-once 보장)
            consumer.commitSync();
        }

        log.info("Consumer stopped. Closing...");
        consumer.close();
    }

    @PreDestroy
    public void shutdown() {
        log.info("Stopping Kafka consumer...");
        running = false;
        consumer.wakeup();  // poll() 블로킹 해제
    }
}
```

`consumer.wakeup()`은 블로킹 중인 `poll()`을 즉시 `WakeupException`으로 중단시킨다.

`running = false`로 루프를 탈출하게 하고, `consumer.close()`로 오프셋을 커밋하고 그룹에서 탈퇴한다.

---

## 상태 저장 서버의 종료 절차

### 상태를 가진 서버란

로드밸런서 뒤에서 세션 상태를 로컬 메모리에 저장하거나, 인메모리 캐시를 운영하거나, 소켓 연결 상태를 유지하는 서버가 여기 해당한다.

단순히 요청을 완료하고 종료하는 것만으로는 부족하다.

### 세션 상태 이전

```java
@Component
public class SessionStatefulShutdown {

    private final SessionRepository sessionRepository;
    private final DistributedCache distributedCache;

    @PreDestroy
    public void migrateState() {
        log.info("Starting session state migration...");

        // 로컬 메모리의 세션 데이터를 분산 캐시(Redis)로 플러시
        Map<String, Session> localSessions = getLocalSessions();
        localSessions.forEach((id, session) -> {
            distributedCache.set("session:" + id, session, Duration.ofHours(1));
            log.debug("Migrated session: {}", id);
        });

        log.info("Migrated {} sessions to distributed cache.", localSessions.size());
    }
}
```

### WebSocket 서버 종료

WebSocket은 HTTP보다 더 복잡하다. 연결이 장시간 유지되고, 클라이언트가 재연결 로직을 가져야 한다.

```java
@Component
public class WebSocketShutdownHandler {

    private final Set<WebSocketSession> activeSessions =
        ConcurrentHashMap.newKeySet();

    @PreDestroy
    public void closeAllSessions() throws Exception {
        log.info("Closing {} WebSocket sessions...", activeSessions.size());

        // 모든 클라이언트에게 서버 재시작 메시지 전송
        CloseStatus restarting = new CloseStatus(1012, "Server restarting");

        List<CompletableFuture<Void>> closeFutures = activeSessions.stream()
            .map(session -> CompletableFuture.runAsync(() -> {
                try {
                    session.close(restarting);
                } catch (IOException e) {
                    log.warn("Failed to close session: {}", session.getId());
                }
            }))
            .toList();

        // 모든 세션 닫힘 대기 (최대 10초)
        CompletableFuture.allOf(closeFutures.toArray(new CompletableFuture[0]))
            .orTimeout(10, TimeUnit.SECONDS)
            .exceptionally(ex -> {
                log.warn("Some sessions did not close cleanly: {}", ex.getMessage());
                return null;
            })
            .join();

        log.info("All WebSocket sessions closed.");
    }
}
```

WebSocket 종료 코드 1012는 "Service Restart"를 의미한다.

클라이언트는 이 코드를 받으면 일정 시간 후 자동 재연결을 시도해야 한다.

### 분산 락 해제

서버가 분산 락(Redis SETNX, ZooKeeper 등)을 보유 중이라면 종료 전에 반드시 해제해야 한다.

```java
@Component
public class DistributedLockAwareShutdown {

    private final RedissonClient redisson;
    private final Set<String> acquiredLocks = ConcurrentHashMap.newKeySet();

    public void acquireLock(String lockKey) {
        RLock lock = redisson.getLock(lockKey);
        lock.lock(30, TimeUnit.SECONDS);
        acquiredLocks.add(lockKey);
    }

    public void releaseLock(String lockKey) {
        RLock lock = redisson.getLock(lockKey);
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
            acquiredLocks.remove(lockKey);
        }
    }

    @PreDestroy
    public void releaseAllLocks() {
        log.info("Releasing {} distributed locks...", acquiredLocks.size());
        acquiredLocks.forEach(this::releaseLock);
        log.info("All distributed locks released.");
    }
}
```

---

## 안티패턴

### 안티패턴 1: 타임아웃 없는 무한 대기

```java
// 잘못된 예
@PreDestroy
public void shutdown() {
    while (activeRequests.get() > 0) {
        Thread.sleep(100); // 무한 루프 가능성
    }
}

// 올바른 예
@PreDestroy
public void shutdown() throws InterruptedException {
    long deadline = System.currentTimeMillis() + 30_000; // 30초
    while (activeRequests.get() > 0) {
        if (System.currentTimeMillis() > deadline) {
            log.warn("Shutdown timeout. Active: {}", activeRequests.get());
            break;
        }
        Thread.sleep(100);
    }
}
```

### 안티패턴 2: ShutdownHook에서 블로킹 I/O

JVM ShutdownHook은 제한된 환경에서 실행된다.

네트워크 호출이나 무거운 DB 작업은 ShutdownHook이 아닌 `@PreDestroy`나 `LifecycleProcessor`에서 처리해야 한다.

```java
// 잘못된 예
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // ShutdownHook에서 원격 API 호출 - 위험
    externalService.deregister();
}));

// 올바른 예
@PreDestroy
public void deregister() {
    // Spring 컨텍스트가 살아있는 동안 실행
    externalService.deregister();
}
```

### 안티패턴 3: Drain 순서 역전

```text
잘못된 순서:
1. DB 커넥션 풀 닫기
2. HTTP 서버 종료 (요청 대기)  <- DB가 이미 닫혀 있어 요청 실패

올바른 순서:
1. 새 요청 거부 (Drain 시작)
2. 인플라이트 요청 완료 대기
3. 애플리케이션 레이어 정리 (캐시, 큐)
4. 인프라 레이어 정리 (DB, 파일)
5. 프로세스 종료
```

### 안티패턴 4: 시그널 무시

```go
// 잘못된 예 - 시그널 처리 없이 무한 루프
func main() {
    http.ListenAndServe(":8080", handler)
}

// 올바른 예
func main() {
    srv := &http.Server{Addr: ":8080"}
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    go srv.ListenAndServe()
    <-quit
    srv.Shutdown(context.Background())
}
```

---

## 종료 절차 체크리스트

서버 종료 시 올바른 순서를 요약하면 다음과 같다.

```
[1] 시그널 수신 (SIGTERM)
 ↓
[2] 헬스체크 엔드포인트 → 503 반환
    (로드밸런서가 이 서버를 제외하도록 유도)
 ↓
[3] 새 커넥션/요청 수락 중단
 ↓
[4] 인플라이트 요청 완료 대기 (타임아웃 포함)
 ↓
[5] 백그라운드 작업/워커 종료
 ↓
[6] 메시지 큐 커밋/언서브스크라이브
 ↓
[7] 분산 락 해제
 ↓
[8] 캐시/세션 상태 플러시
 ↓
[9] DB 커넥션 풀 반납
 ↓
[10] 파일 핸들 닫기, 임시 파일 정리
 ↓
[11] 프로세스 종료 (exit 0)
```

각 단계마다 로그를 남기면 종료 과정에서 무엇이 오래 걸리는지 프로파일링할 수 있다.

---

## K8s 환경의 Graceful Shutdown

Kubernetes 환경에서는 Graceful Shutdown이 더 복잡하다.

Pod 종료 흐름, `preStop` 훅, `terminationGracePeriodSeconds` 설정 등이 맞물려 동작하며, 로드밸런서(Service)와의 타이밍 문제도 다뤄야 한다.

이 주제는 **[11] Docker & 컨테이너** 편에서 K8s 배포 전략과 함께 상세히 다룬다.

---

## 마무리

Graceful Shutdown은 "있으면 좋은 기능"이 아니다.

무중단 배포가 요구되는 프로덕션 환경에서는 필수 요소다.

시그널 핸들러를 올바르게 등록하고, 인플라이트 요청을 추적하고, 리소스를 정해진 순서대로 정리하는 것. 이 세 가지만 제대로 구현해도 배포 중 에러율을 0에 가깝게 유지할 수 있다.

복잡한 분산 시스템일수록 종료 절차가 복잡해진다. 그러나 원칙은 동일하다. **새 요청을 막고, 기존 요청을 완료하고, 리소스를 정리하고, 그 다음에 종료한다.**

---

## 참고 자료

1. [Spring Boot - Graceful Shutdown](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.graceful-shutdown) — Spring 공식 문서
2. [Go net/http - Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown) — Go 표준 라이브러리 문서
3. [Kubernetes - Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) — K8s Pod 종료 흐름 공식 문서
4. [AWS ELB - Deregistration Delay](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#deregistration-delay) — ALB 드레인 설정
5. [The Twelve-Factor App - Disposability](https://12factor.net/disposability) — 빠른 시작과 우아한 종료 원칙
6. [Linux Signal(7) man page](https://man7.org/linux/man-pages/man7/signal.7.html) — Unix 시그널 레퍼런스
