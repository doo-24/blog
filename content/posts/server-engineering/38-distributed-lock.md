---
title: "[비동기 처리와 이벤트 드리븐] 5편 — 분산 락 실전: 여러 서버가 하나의 자원을 다투는 법"
date: 2026-03-17T21:03:00+09:00
draft: false
tags: ["분산 락", "Redis", "Redlock", "Redisson", "동시성", "서버"]
series: ["비동기 처리와 이벤트 드리븐"]
summary: "분산 락이 필요한 상황(스케줄러, 선착순, 재고), Redis 기반 분산 락(SETNX, Redlock 알고리즘과 한계), 분산 락 vs DB 낙관적 락 선택 기준, Redisson 실전 적용 패턴까지"
---

단일 서버에서는 `synchronized` 키워드 하나로 해결되던 문제가 서버가 두 대, 세 대로 늘어나는 순간 전혀 다른 차원의 문제가 된다. 프로세스 내 메모리를 공유하지 않는 분산 환경에서 "지금 이 작업은 나 혼자 한다"는 보장을 얻으려면 프로세스 외부의 누군가가 중재자 역할을 맡아야 한다. 분산 락(Distributed Lock)은 그 중재자를 구현하는 메커니즘이다. Redis, ZooKeeper, etcd 등 다양한 백엔드 위에 구축할 수 있지만, 현업에서 가장 많이 쓰이는 Redis 기반 구현을 중심으로 실제로 어디서 막히는지, 어떻게 설계해야 안전한지를 낱낱이 파헤쳐 본다.

---

## 분산 락이 필요한 상황

분산 락이 필요하다고 느끼는 순간은 대개 비슷하다. "이 코드는 한 번에 하나만 실행돼야 하는데, 서버가 여러 대라 제어할 방법이 없다."

### 스케줄러 중복 실행

배치 작업이나 주기적 정산 잡은 전통적으로 단일 서버에서만 실행해 왔다. 하지만 고가용성을 위해 서버를 이중화하면 Spring `@Scheduled`나 Quartz가 두 인스턴스에서 동시에 실행된다.

```java
// 위험한 패턴: 이중화 환경에서 중복 실행
@Scheduled(cron = "0 0 2 * * *")
public void settlementJob() {
    // 새벽 2시에 두 서버가 동시에 이 코드를 실행한다
    settlementService.processDaily();
}
```

실제로 이런 상황에서 정산 데이터가 두 배로 생성되거나, 이메일 알림이 두 번 발송되는 장애가 발생한다. 분산 락을 잡은 인스턴스만 작업을 수행하고, 락 획득에 실패한 인스턴스는 조용히 건너뛰는 패턴이 필요하다.

### 선착순 이벤트와 쿠폰 발급

"선착순 100명"이라는 조건은 데이터베이스 레벨의 제약만으로는 부족할 때가 많다. 재고 확인과 차감 사이의 gap에서 동시 요청이 몰리면 100개를 초과해 발급된다.

```
시간축:
T1: 서버A - 잔여 쿠폰 조회 -> 1개 남음
T2: 서버B - 잔여 쿠폰 조회 -> 1개 남음  (T1과 거의 동시)
T3: 서버A - 쿠폰 발급, 잔여 0으로 업데이트
T4: 서버B - 쿠폰 발급, 잔여 0으로 업데이트  <- 중복 발급!
```

이 문제는 DB 트랜잭션 격리 수준을 높이거나 낙관적 락으로도 해결할 수 있지만, 트래픽이 폭증하는 이벤트 상황에서는 충돌 빈도가 높아 재시도 비용이 커진다. 분산 락으로 직렬화하는 편이 예측 가능성 면에서 유리하다.

### 재고 관리와 중복 주문 방지

같은 사용자가 네트워크 불안정으로 동일한 주문 요청을 두 번 보내는 상황, 또는 여러 서버에서 동시에 같은 상품의 재고를 감소시키는 상황 모두 분산 락의 적용 대상이다.

특히 재고 관리에서는 "락의 단위"가 중요하다. 전체 상품 테이블을 잠그면 다른 상품 구매까지 막힌다. `product:{productId}` 단위로 세밀하게 락을 잡아야 처리량을 유지할 수 있다.

---

## Redis 기반 분산 락

### SETNX: 가장 단순한 출발점

Redis의 `SETNX`(SET if Not eXists)는 분산 락의 가장 기본적인 구현 재료다. 키가 없을 때만 설정되고 이미 있으면 실패한다는 원자적 동작을 이용한다.

```java
// Spring Data Redis를 이용한 단순 SETNX 구현
@Component
public class SimpleRedisLock {

    private final StringRedisTemplate redisTemplate;

    public SimpleRedisLock(StringRedisTemplate redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public boolean tryLock(String key, String value, long expireSeconds) {
        Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(key, value, Duration.ofSeconds(expireSeconds));
        return Boolean.TRUE.equals(result);
    }

    public void unlock(String key, String value) {
        // 반드시 자신이 설정한 값인지 확인 후 삭제
        String currentValue = redisTemplate.opsForValue().get(key);
        if (value.equals(currentValue)) {
            redisTemplate.delete(key);
        }
    }
}
```

이 코드에는 치명적인 결함이 있다. `unlock`의 조회-삭제 과정이 원자적이지 않다. 조회 후 삭제 전에 TTL이 만료되어 다른 스레드가 락을 획득하면 엉뚱한 락을 삭제하게 된다.

```java
// 원자적 unlock: Lua 스크립트로 조회와 삭제를 단일 연산으로
private static final String UNLOCK_SCRIPT =
    "if redis.call('get', KEYS[1]) == ARGV[1] then " +
    "    return redis.call('del', KEYS[1]) " +
    "else " +
    "    return 0 " +
    "end";

public boolean unlock(String key, String value) {
    DefaultRedisScript<Long> script = new DefaultRedisScript<>();
    script.setScriptText(UNLOCK_SCRIPT);
    script.setResultType(Long.class);

    Long result = redisTemplate.execute(
        script,
        Collections.singletonList(key),
        value
    );
    return Long.valueOf(1L).equals(result);
}
```

Lua 스크립트는 Redis 내에서 원자적으로 실행된다. 조회와 삭제 사이에 다른 명령이 끼어들 수 없다.

### 락 값으로 UUID를 써야 하는 이유

락 값을 단순히 `"locked"`로 설정하면 어느 클라이언트가 잠갔는지 구분할 수 없다. UUID 또는 `{serverId}:{threadId}:{timestamp}` 형태의 고유 식별자를 사용해야 자신이 잡은 락만 해제할 수 있다.

```java
public class DistributedLockClient {

    private final SimpleRedisLock redisLock;

    public <T> T executeWithLock(String lockKey, long expireSeconds,
                                  Supplier<T> task) {
        String lockValue = UUID.randomUUID().toString();
        boolean acquired = redisLock.tryLock(lockKey, lockValue, expireSeconds);

        if (!acquired) {
            throw new LockAcquisitionException("락 획득 실패: " + lockKey);
        }

        try {
            return task.get();
        } finally {
            redisLock.unlock(lockKey, lockValue);
        }
    }
}
```

### Redlock 알고리즘

단일 Redis 노드 기반 락은 Redis 장애 시 락이 사라지거나, 페일오버 과정에서 같은 락이 두 클라이언트에게 동시에 허용되는 문제가 있다. Redis 공식 문서에서 제안하는 **Redlock** 알고리즘은 독립적인 Redis 노드 N개(권장: 5개)를 이용해 이 문제를 해결하려 한다.

알고리즘의 핵심 단계는 다음과 같다.

1. 현재 시각을 밀리초 단위로 기록한다.
2. N개 노드 모두에 동일한 키와 랜덤 값으로 락 획득을 시도한다. 각 노드에 대한 타임아웃은 전체 락 유효 시간보다 훨씬 짧게 설정한다(예: 5~50ms).
3. `N/2 + 1`개 이상의 노드에서 락을 획득하고, 획득에 소요된 총 시간이 락 유효 시간보다 짧으면 락이 유효하다고 판단한다.
4. 실제 락 유효 시간은 `TTL - 경과시간 - 클록 드리프트 보정값`으로 계산한다.
5. 과반수 획득에 실패하면 즉시 모든 노드에서 락을 해제한다.

```
Redis 노드 5개에서 Redlock 획득:
노드1: 성공 (5ms 소요)
노드2: 성공 (3ms 소요)
노드3: 실패 (타임아웃)
노드4: 성공 (4ms 소요)
노드5: 성공 (6ms 소요)

총 소요 시간: 18ms
설정한 TTL: 10,000ms
유효한 락 유효 시간: 10,000 - 18 - 보정값(약 200ms) = 9,782ms
과반수(3개) 이상 성공 -> 락 획득 성공
```

### Redlock의 한계와 논쟁

마틴 클레프만(Martin Kleppmann)은 2016년 Redlock에 대한 유명한 비판 글을 발표했다. 핵심 논점은 두 가지다.

**클록 드리프트 문제**: Redis 노드들의 시스템 클록이 다를 수 있고, NTP 서버의 시각 보정으로 시계가 갑자기 앞으로 당겨지면 TTL이 예상보다 일찍 만료될 수 있다.

**GC 중단(Stop-the-World) 문제**: JVM에서 Full GC가 발생하면 프로세스가 수십 초 멈출 수 있다. 이 경우 락의 TTL이 만료되어 다른 클라이언트가 락을 획득했는데, GC에서 깨어난 클라이언트는 자신이 여전히 락을 보유하고 있다고 착각하고 작업을 계속한다.

```
T1: 클라이언트A가 락 획득 (TTL: 30초)
T2: 클라이언트A에서 Full GC 시작 (40초간 중단)
T3: 락의 TTL 만료
T4: 클라이언트B가 동일한 락 획득
T5: 클라이언트A의 GC 종료, 자신이 락을 보유하고 있다고 믿고 작업 수행
T6: 클라이언트A와 클라이언트B가 동시에 보호 구역에서 작업 중 <- 위험!
```

이를 방지하기 위한 패턴이 **펜싱 토큰(Fencing Token)**이다. 락 획득 시 단조 증가하는 토큰을 발급받고, 보호 대상 자원(DB, 파일 등)에 접근할 때 이 토큰을 함께 전달한다. 자원은 이전에 처리한 것보다 낮은 토큰 번호를 가진 요청을 거부한다.

```java
public class FencedLockRequest {
    private final long fencingToken; // 락 획득 시 증가하는 토큰
    private final String data;

    // DB에 저장 시 last_token 컬럼과 비교
    // INSERT INTO orders (...) WHERE last_token < ?
}
```

현실에서 Redlock이 완전히 잘못된 것은 아니다. 락이 "작업 중복 방지"를 위한 best-effort 메커니즘이고, 최악의 경우 중복 실행이 허용되는 시스템(멱등성이 보장된 작업)이라면 Redlock은 실용적이다. 하지만 "이 코드는 절대로 동시에 실행되면 안 된다"는 강한 상호 배제가 필요하다면 ZooKeeper나 etcd 기반의 구현이 더 적합하다.

---

## 분산 락 vs DB 낙관적 락 선택 기준

### DB 낙관적 락의 특성

JPA의 `@Version`을 이용한 낙관적 락은 별도의 인프라 없이 데이터베이스만으로 동시성을 제어한다.

```java
@Entity
public class Product {

    @Id
    private Long id;

    private int stock;

    @Version
    private Long version;
}

// 서비스 레이어
@Transactional
public void decreaseStock(Long productId, int quantity) {
    Product product = productRepository.findById(productId)
        .orElseThrow();

    if (product.getStock() < quantity) {
        throw new InsufficientStockException();
    }

    product.decreaseStock(quantity);
    // version이 변경되었으면 OptimisticLockException 발생
    // -> 상위에서 재시도 처리 필요
}
```

낙관적 락은 충돌이 드문 경우에 효율적이다. 락 자체의 오버헤드가 없고 데드락이 발생하지 않는다. 하지만 충돌이 빈번하면 재시도 폭풍(retry storm)이 발생하고, 재시도 로직을 직접 구현해야 한다.

### 선택 기준표

| 기준 | 분산 락 (Redis) | DB 낙관적 락 |
|------|----------------|-------------|
| 충돌 빈도 | 높음에 적합 | 낮음에 적합 |
| 인프라 의존성 | Redis 필요 | DB만으로 충분 |
| 성능 특성 | 직렬화, 예측 가능 | 낮은 충돌시 고성능 |
| 구현 복잡도 | 중간 | 낮음 |
| 교차 서비스 보호 | 가능 | DB 트랜잭션 범위 내만 |
| 비-DB 자원 보호 | 가능 (파일, API 등) | 불가능 |

**분산 락을 선택해야 하는 상황:**
- 스케줄러 중복 방지처럼 DB 트랜잭션 밖에서의 제어가 필요한 경우
- 외부 API 호출, 파일 생성 등 DB 트랜잭션으로 감쌀 수 없는 자원을 보호하는 경우
- 선착순 이벤트처럼 충돌이 높고 재시도가 비효율적인 경우
- 여러 마이크로서비스가 공유 자원에 동시에 접근하는 경우

**DB 낙관적 락을 선택해야 하는 상황:**
- Redis 인프라를 추가하기 어려운 경우
- 충돌이 드물고 대부분의 트랜잭션이 성공하는 경우
- 이미 JPA를 사용하고 있어 추가 학습 비용을 최소화하려는 경우

### 락 타임아웃 설계

TTL 설정은 두 가지 상충하는 요구사항의 균형이다.

- **너무 짧으면**: 작업이 완료되기 전에 락이 만료되어 다른 프로세스가 동일 작업을 시작한다.
- **너무 길면**: 락을 보유한 프로세스가 비정상 종료될 때 다음 실행이 지연된다.

일반적인 설계 원칙:

```
TTL = 예상 최대 실행 시간 × 안전 계수(1.5~2.0) + 네트워크 지연 보정값

예시:
- 정산 배치: 예상 5분, TTL = 15분 (충분한 여유를 주되 장애 시 15분 후 복구)
- 재고 차감: 예상 100ms, TTL = 5초 (빠른 복구 우선)
- 쿠폰 발급: 예상 200ms, TTL = 10초
```

작업 진행 중 TTL 갱신(락 연장)이 필요한 경우 Redisson의 Watchdog 메커니즘이 이를 자동으로 처리한다.

---

## Spring Integration과 Redisson 실전 적용

### Redisson 선택 이유

Spring Data Redis의 `RedisTemplate`으로 직접 분산 락을 구현할 수 있지만, Redisson은 다음을 제공해 실무에서 훨씬 편리하다.

- `RLock` 인터페이스: `java.util.concurrent.locks.Lock` 호환
- Watchdog: 락 보유 중 TTL 자동 연장 (기본 30초, 1/3 주기로 연장)
- `tryLock(waitTime, leaseTime, unit)`: 대기 시간과 만료 시간 분리
- Fair Lock, Read/Write Lock, MultiLock 지원

```xml
<!-- Maven 의존성 -->
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.27.2</version>
</dependency>
```

```yaml
# application.yml
spring:
  redis:
    host: localhost
    port: 6379

# Redisson 설정 (redisson.yaml 또는 RedissonClient 빈 직접 설정)
```

### RedissonClient 빈 설정

```java
@Configuration
public class RedissonConfig {

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer()
              .setAddress("redis://" + redisHost + ":" + redisPort)
              .setConnectionMinimumIdleSize(5)
              .setConnectionPoolSize(20)
              .setConnectTimeout(3000)
              .setTimeout(3000)
              .setRetryAttempts(3)
              .setRetryInterval(1500);
        return Redisson.create(config);
    }
}
```

### 기본 분산 락 사용 패턴

```java
@Service
@RequiredArgsConstructor
public class CouponService {

    private final RedissonClient redissonClient;
    private final CouponRepository couponRepository;

    private static final String COUPON_LOCK_PREFIX = "lock:coupon:";
    private static final long WAIT_TIME = 3L;
    private static final long LEASE_TIME = 10L;

    public CouponIssueResult issueCoupon(Long couponId, Long userId) {
        String lockKey = COUPON_LOCK_PREFIX + couponId;
        RLock lock = redissonClient.getLock(lockKey);

        boolean acquired = false;
        try {
            // WAIT_TIME 동안 락 획득 시도, 획득 후 LEASE_TIME 초 유지
            acquired = lock.tryLock(WAIT_TIME, LEASE_TIME, TimeUnit.SECONDS);

            if (!acquired) {
                return CouponIssueResult.failure("락 획득 실패 - 잠시 후 다시 시도하세요");
            }

            // 임계 구역
            Coupon coupon = couponRepository.findById(couponId)
                .orElseThrow(() -> new CouponNotFoundException(couponId));

            if (!coupon.hasRemaining()) {
                return CouponIssueResult.failure("쿠폰이 모두 소진되었습니다");
            }

            coupon.issue(userId);
            couponRepository.save(coupon);

            return CouponIssueResult.success(coupon.getCode());

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CouponIssueResult.failure("락 획득 중 인터럽트 발생");
        } finally {
            // 자신이 보유한 락만 해제 (isHeldByCurrentThread 확인)
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### AOP 기반 분산 락 추상화

동일한 try-catch-finally 패턴이 여러 서비스에 반복된다면 AOP로 추상화할 수 있다.

```java
// 커스텀 애노테이션 정의
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DistributedLock {
    String key();                            // SpEL 표현식 지원
    long waitTime() default 5L;
    long leaseTime() default 10L;
    TimeUnit timeUnit() default TimeUnit.SECONDS;
}
```

```java
// AOP Aspect 구현
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class DistributedLockAspect {

    private final RedissonClient redissonClient;
    private final AopForTransaction aopForTransaction;

    @Around("@annotation(distributedLock)")
    public Object around(ProceedingJoinPoint joinPoint,
                         DistributedLock distributedLock) throws Throwable {
        String lockKey = parseLockKey(joinPoint, distributedLock.key());
        RLock lock = redissonClient.getLock(lockKey);

        boolean acquired = false;
        try {
            acquired = lock.tryLock(
                distributedLock.waitTime(),
                distributedLock.leaseTime(),
                distributedLock.timeUnit()
            );

            if (!acquired) {
                throw new LockAcquisitionException("분산 락 획득 실패: " + lockKey);
            }

            log.debug("분산 락 획득: {}", lockKey);
            // 트랜잭션을 새로 시작하여 락 해제 전에 커밋되도록 보장
            return aopForTransaction.proceed(joinPoint);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockAcquisitionException("락 획득 중 인터럽트", e);
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
                log.debug("분산 락 해제: {}", lockKey);
            }
        }
    }

    private String parseLockKey(ProceedingJoinPoint joinPoint, String keyExpression) {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        Object[] args = joinPoint.getArgs();
        String[] paramNames = signature.getParameterNames();

        StandardEvaluationContext context = new StandardEvaluationContext();
        for (int i = 0; i < paramNames.length; i++) {
            context.setVariable(paramNames[i], args[i]);
        }

        ExpressionParser parser = new SpelExpressionParser();
        return "lock:" + parser.parseExpression(keyExpression).getValue(context, String.class);
    }
}
```

```java
// 트랜잭션을 별도 빈에서 실행 (REQUIRES_NEW로 락 내에서 커밋 보장)
@Component
public class AopForTransaction {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Object proceed(ProceedingJoinPoint joinPoint) throws Throwable {
        return joinPoint.proceed();
    }
}
```

```java
// 사용 예시 - 깔끔해진 서비스 코드
@Service
@RequiredArgsConstructor
public class StockService {

    private final ProductRepository productRepository;

    @DistributedLock(key = "'product:' + #productId", waitTime = 3, leaseTime = 5)
    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow();

        product.decrease(quantity);
        productRepository.save(product);
    }
}
```

### 스케줄러 중복 실행 방지 패턴

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class DailySettlementJob {

    private final RedissonClient redissonClient;
    private final SettlementService settlementService;

    @Scheduled(cron = "0 0 2 * * *")
    public void run() {
        String lockKey = "scheduler:settlement:daily";
        RLock lock = redissonClient.getLock(lockKey);

        // tryLock(0, leaseTime): 대기 없이 즉시 시도
        // leaseTime은 작업 예상 시간의 2배 이상으로 설정
        boolean acquired = false;
        try {
            acquired = lock.tryLock(0, 30, TimeUnit.MINUTES);

            if (!acquired) {
                log.info("이미 다른 인스턴스에서 정산 작업 실행 중. 건너뜀.");
                return;
            }

            log.info("정산 작업 시작");
            settlementService.processDaily();
            log.info("정산 작업 완료");

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("정산 작업 중 인터럽트 발생", e);
        } finally {
            if (acquired && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### Watchdog 동작 이해

`leaseTime`을 `-1`로 설정하면 Redisson의 Watchdog이 활성화된다. 기본 TTL은 30초이고, 락을 보유한 동안 10초마다 자동으로 연장한다.

```java
// Watchdog 활성화 (leaseTime = -1 또는 생략)
RLock lock = redissonClient.getLock("long-running-task");

try {
    lock.lock(); // leaseTime 없이 호출하면 Watchdog 활성화
    // 또는
    lock.lock(TimeUnit.SECONDS); // 이것도 동일

    // 오래 걸리는 작업... Watchdog이 TTL을 자동 갱신
    performLongRunningTask();

} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock(); // unlock 시 Watchdog도 중단됨
    }
}
```

Watchdog은 편리하지만 주의가 필요하다. 프로세스가 비정상 종료되면 Watchdog도 멈추고 TTL이 자연 만료되어 락이 해제된다. 하지만 GC 중단처럼 프로세스는 살아있지만 응답이 없는 경우 Watchdog이 계속 갱신을 시도한다. 이 경우 의도치 않게 락을 무한히 보유할 수 있다.

실무에서는 작업 유형에 따라 선택한다.
- 예측 가능한 짧은 작업: 명시적 `leaseTime` 설정
- 실행 시간이 가변적인 배치 작업: Watchdog 활용

---

## 자주 발생하는 안티패턴

### 안티패턴 1: 트랜잭션과 락의 순서 오류

```java
// 잘못된 패턴: 트랜잭션이 먼저 시작되고 락이 나중에 획득됨
@Transactional
public void badPattern(Long productId) {
    RLock lock = redissonClient.getLock("product:" + productId);
    lock.lock();
    try {
        // 작업 수행
    } finally {
        lock.unlock(); // 락은 해제됐지만 트랜잭션은 아직 커밋 전
        // 다른 스레드가 락을 획득하고 DB를 읽으면 아직 커밋 전 데이터를 볼 수 없음
    }
}
```

락 범위 안에서 트랜잭션을 완전히 포함시켜야 한다. 앞서 보인 `AopForTransaction`처럼 `REQUIRES_NEW` 전파 수준을 이용하거나, 서비스를 분리하여 락 획득 → 트랜잭션 시작 → 작업 → 트랜잭션 커밋 → 락 해제 순서를 보장한다.

### 안티패턴 2: 예외 시 락 미해제

```java
// 잘못된 패턴: 예외 발생 시 락이 해제되지 않음
public void badPattern(Long productId) {
    RLock lock = redissonClient.getLock("product:" + productId);
    lock.lock();

    // 여기서 예외가 발생하면 lock.unlock()이 호출되지 않음
    performRiskyOperation();

    lock.unlock();
}
```

항상 `try-finally`로 감싸야 한다. TTL이 설정되어 있으면 결국 만료되지만, 그전까지 해당 자원은 잠긴 상태다.

### 안티패턴 3: 너무 넓은 락 범위

```java
// 잘못된 패턴: 상품 ID와 무관하게 전체 재고를 하나의 락으로 보호
public void badPattern(Long productId, int quantity) {
    RLock lock = redissonClient.getLock("lock:all-products"); // 너무 넓음!
    // 상품A 재고 차감 중에 상품B 구매도 막힘
}

// 올바른 패턴: 상품별로 독립적인 락
public void goodPattern(Long productId, int quantity) {
    RLock lock = redissonClient.getLock("lock:product:" + productId);
    // 상품A와 상품B의 재고 차감은 독립적으로 진행
}
```

### 안티패턴 4: 락 내부에서 외부 I/O 과다

락을 보유한 상태에서 느린 외부 API 호출이나 대용량 파일 읽기를 수행하면 락 보유 시간이 길어져 처리량이 저하된다. 필요한 데이터를 락 획득 전에 미리 준비하거나, 락 내부 로직을 최소화한다.

---

## 분산 락 모니터링

프로덕션에서 분산 락을 운영할 때 다음 지표를 모니터링해야 한다.

```java
@Aspect
@Component
@RequiredArgsConstructor
public class LockMetricsAspect {

    private final MeterRegistry meterRegistry;

    @Around("@annotation(distributedLock)")
    public Object recordMetrics(ProceedingJoinPoint joinPoint,
                                DistributedLock distributedLock) throws Throwable {
        String lockKey = distributedLock.key();
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            Object result = joinPoint.proceed();
            meterRegistry.counter("distributed.lock.acquired",
                "key", lockKey, "result", "success").increment();
            return result;

        } catch (LockAcquisitionException e) {
            meterRegistry.counter("distributed.lock.acquired",
                "key", lockKey, "result", "failure").increment();
            throw e;

        } finally {
            sample.stop(Timer.builder("distributed.lock.held.duration")
                .tag("key", lockKey)
                .register(meterRegistry));
        }
    }
}
```

**주요 모니터링 지표:**
- 락 획득 실패율: 높으면 병목 또는 TTL 설정 문제
- 락 보유 시간: 예상보다 길면 로직 최적화 또는 TTL 조정 필요
- Redis 연결 오류율: 분산 락 전체 가용성과 직결

---

분산 락은 강력하지만 복잡성을 수반한다. "락이 필요한가?"를 먼저 물어보고, DB 트랜잭션이나 낙관적 락으로 해결 가능한 문제에 불필요하게 Redis 락을 도입하는 것은 오버엔지니어링이다. 하지만 스케줄러 중복 방지나 외부 시스템 호출 직렬화처럼 DB 트랜잭션 밖에 있는 문제라면 Redisson과 잘 설계된 TTL 전략이 가장 실용적인 해답이다. 락의 범위를 최소화하고, 트랜잭션과의 순서를 명확히 하고, 항상 `finally`에서 해제한다는 세 가지 원칙만 지켜도 대부분의 함정은 피할 수 있다.

---

## 참고 자료

1. [Martin Kleppmann - How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) - Redlock의 안전성 문제를 심층 분석한 필독 글
2. [Redis 공식 문서 - Distributed Locks with Redis](https://redis.io/docs/manual/patterns/distributed-locks/) - Redlock 알고리즘 원문과 Antirez의 반박 포함
3. [Redisson 공식 문서 - Distributed Lock](https://redisson.org/docs/data-and-services/locks-and-synchronizers/) - RLock, Fair Lock, MultiLock 등 상세 API 문서
4. [Designing Data-Intensive Applications, Martin Kleppmann](https://dataintensive.net/) - 8장, 9장에서 분산 시스템에서의 타이밍과 클록 문제 심층 설명
5. [Baeldung - Introduction to Redisson](https://www.baeldung.com/redisson) - Spring Boot와의 통합 예제 및 실전 패턴
6. [Spring Integration - Redis Lock Registry](https://docs.spring.io/spring-integration/docs/current/reference/html/redis.html#redis-lock-registry) - Spring 생태계 내 Redis 분산 락 공식 구현체
