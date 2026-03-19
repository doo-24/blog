---
title: "[성능 최적화와 안정성] 2편 — 캐싱 전략: 빠른 서비스의 비밀"
date: 2026-03-17T18:05:00+09:00
draft: false
tags: ["캐시", "Redis", "Caffeine", "캐시 무효화", "서버"]
series: ["성능 최적화와 안정성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 8
summary: "로컬 캐시(Caffeine) vs 분산 캐시(Redis) 선택 기준, Cache-Aside/Write-Through/Write-Behind/Refresh-Ahead 패턴, TTL 설계와 캐시 워밍·핫스팟 문제, 캐시 무효화 전략까지"
---

데이터베이스 쿼리 한 번에 300ms가 걸리는 API가 있다고 가정하자. 동시에 1,000명이 그 API를 호출하면 DB는 즉시 포화 상태에 빠진다. 캐싱은 이 문제를 10ms 이하로 줄이는 가장 직접적인 수단이다. 하지만 캐시를 "그냥 Redis 붙이면 된다"고 생각하는 순간, 데이터 불일치·메모리 폭발·캐시 스탬피드 같은 새로운 악몽이 시작된다. 이 글에서는 캐싱의 핵심 패턴과 실전 함정을 정면으로 다룬다.

---

## 1. 로컬 캐시 vs 분산 캐시: 무엇을 선택할 것인가

캐시를 도입하기 전에 가장 먼저 결정해야 할 질문은 "어디에 저장할 것인가"다.

로컬 캐시는 애플리케이션 프로세스 메모리(JVM heap) 안에 데이터를 저장한다. 네트워크 왕복이 없으므로 응답 속도는 1µs(마이크로초) 수준이다.

분산 캐시(Redis, Memcached)는 별도의 캐시 서버에 데이터를 저장한다. 네트워크를 거치므로 응답 속도는 1~5ms 수준이지만, 여러 서버 인스턴스가 동일한 데이터를 공유할 수 있다.

### 로컬 캐시가 적합한 경우

- 불변(immutable)이거나 변경 빈도가 매우 낮은 데이터 (국가 코드, 환율 테이블 등)
- 단일 서버 인스턴스로 운영하거나, 인스턴스 간 데이터 불일치가 허용되는 경우
- 네트워크 지연을 절대적으로 줄여야 하는 초고성능 경로 (hot path)

### 분산 캐시가 적합한 경우

- 여러 서버 인스턴스가 동일한 캐시를 공유해야 하는 경우
- 사용자 세션, 장바구니처럼 인스턴스 간 일관성이 필요한 데이터
- JVM 힙 메모리가 제한적일 때 (GC 압력을 피하기 위해)
- 서버 재시작 후에도 캐시가 유지되어야 할 때

### 두 계층을 함께 쓰는 L1/L2 패턴

가장 강력한 구성은 로컬 캐시(L1)와 분산 캐시(L2)를 계층적으로 조합하는 것이다.

```
요청 → L1(Caffeine) 조회 → HIT: 즉시 반환
                         → MISS: L2(Redis) 조회 → HIT: L1 채우고 반환
                                                → MISS: DB 조회 → L2, L1 채우고 반환
```

읽기 빈도가 매우 높은 데이터는 L1에서 처리하고, L1 미스가 발생하면 L2로 폴백한다. 이 구조는 DB 부하를 극단적으로 줄이면서 네트워크 비용도 최소화한다.

---

## 2. Java/Spring에서 Caffeine 캐시 구성

Caffeine은 현재 Java 생태계에서 가장 성능이 뛰어난 로컬 캐시 라이브러리다. Guava Cache의 후계자로, W-TinyLFU 알고리즘을 사용해 캐시 히트율을 극대화한다.

### 의존성 추가 (Gradle)

```groovy
implementation 'com.github.ben-manes.caffeine:caffeine:3.1.8'
implementation 'org.springframework.boot:spring-boot-starter-cache'
```

### Spring Cache와 통합

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)           // 최대 10,000개 엔트리
            .expireAfterWrite(5, TimeUnit.MINUTES)  // 쓰기 후 5분 만료
            .recordStats());               // 히트율 통계 수집
        return manager;
    }

    // 캐시별 개별 설정이 필요한 경우
    @Bean
    public CacheManager detailedCacheManager() {
        SimpleCacheManager manager = new SimpleCacheManager();
        manager.setCaches(Arrays.asList(
            buildCache("products", 1000, 10, TimeUnit.MINUTES),
            buildCache("userProfiles", 5000, 30, TimeUnit.MINUTES),
            buildCache("exchangeRates", 100, 1, TimeUnit.HOURS)
        ));
        return manager;
    }

    private CaffeineCache buildCache(String name, long maxSize,
                                      long duration, TimeUnit unit) {
        return new CaffeineCache(name,
            Caffeine.newBuilder()
                .maximumSize(maxSize)
                .expireAfterWrite(duration, unit)
                .recordStats()
                .build());
    }
}
```

### @Cacheable 활용

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#productId")
    public ProductDto getProduct(Long productId) {
        // 캐시 미스 시에만 실행됨
        return productRepository.findById(productId)
            .map(ProductDto::from)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    @CachePut(value = "products", key = "#result.id")
    public ProductDto updateProduct(Long productId, UpdateProductRequest req) {
        Product product = productRepository.findById(productId)
            .orElseThrow();
        product.update(req);
        return ProductDto.from(productRepository.save(product));
    }

    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(Long productId) {
        productRepository.deleteById(productId);
    }
}
```

### 캐시 통계 모니터링

```java
@Component
public class CacheStatsLogger {

    private final CacheManager cacheManager;

    @Scheduled(fixedDelay = 60_000)
    public void logStats() {
        if (cacheManager instanceof CaffeineCacheManager caffeineCacheManager) {
            caffeineCacheManager.getCacheNames().forEach(name -> {
                CaffeineCache cache = (CaffeineCache) cacheManager.getCache(name);
                CacheStats stats = cache.getNativeCache().stats();
                log.info("Cache [{}] hit={:.2f}% miss={} eviction={}",
                    name,
                    stats.hitRate() * 100,
                    stats.missCount(),
                    stats.evictionCount());
            });
        }
    }
}
```

---

## 3. 캐싱 패턴: 네 가지 핵심 전략

캐시를 "어떻게" 채우고 갱신하느냐에 따라 데이터 일관성과 성능이 크게 달라진다.

### Cache-Aside (Lazy Loading)

가장 널리 사용되는 패턴이다. 애플리케이션이 직접 캐시를 관리한다.

```
읽기: 캐시 조회 → 미스 → DB 조회 → 캐시 저장 → 반환
쓰기: DB 업데이트 → 캐시 삭제(또는 갱신)
```

```java
public ProductDto getProduct(Long id) {
    // 1. 캐시 조회
    String cacheKey = "product:" + id;
    String cached = redisTemplate.opsForValue().get(cacheKey);
    if (cached != null) {
        return objectMapper.readValue(cached, ProductDto.class);
    }

    // 2. DB 조회
    ProductDto dto = productRepository.findById(id)
        .map(ProductDto::from)
        .orElseThrow();

    // 3. 캐시 저장 (TTL 5분)
    redisTemplate.opsForValue().set(
        cacheKey,
        objectMapper.writeValueAsString(dto),
        Duration.ofMinutes(5)
    );

    return dto;
}
```

**장점:** 실제 읽힌 데이터만 캐싱한다. 캐시 장애 시에도 DB로 폴백하므로 서비스 가용성이 유지된다.

**단점:** 최초 요청 시 항상 캐시 미스가 발생한다. 또한 DB와 캐시 사이의 짧은 시간 동안 데이터 불일치가 발생할 수 있다.

### Write-Through

데이터를 쓸 때 DB와 캐시를 동시에 업데이트한다.

```java
public ProductDto updateProduct(Long id, UpdateProductRequest req) {
    // 1. DB 업데이트
    Product product = productRepository.findById(id).orElseThrow();
    product.update(req);
    productRepository.save(product);

    // 2. 캐시 즉시 갱신
    ProductDto dto = ProductDto.from(product);
    String cacheKey = "product:" + id;
    redisTemplate.opsForValue().set(
        cacheKey,
        objectMapper.writeValueAsString(dto),
        Duration.ofMinutes(5)
    );

    return dto;
}
```

**장점:** 캐시가 항상 최신 상태를 유지한다. 읽기 캐시 미스가 거의 발생하지 않는다.

**단점:** 쓰기 지연이 증가한다. 실제로 읽히지 않는 데이터도 캐시에 저장되어 메모리를 낭비할 수 있다.

### Write-Behind (Write-Back)

캐시에만 먼저 쓰고, DB 업데이트는 비동기적으로 나중에 처리하는 방식이다.

```java
@Service
public class WriteBackCacheService {

    private final RedisTemplate<String, String> redisTemplate;
    private final BlockingQueue<WriteTask> writeQueue = new LinkedBlockingQueue<>();

    public void updateProduct(Long id, UpdateProductRequest req) {
        // 1. 캐시만 즉시 업데이트
        String cacheKey = "product:" + id;
        ProductDto dto = buildUpdatedDto(id, req);
        redisTemplate.opsForValue().set(cacheKey,
            objectMapper.writeValueAsString(dto),
            Duration.ofMinutes(30));

        // 2. 비동기 DB 쓰기 큐에 등록
        writeQueue.offer(new WriteTask(id, req));
    }

    @Scheduled(fixedDelay = 1000)
    public void flushWriteQueue() {
        List<WriteTask> tasks = new ArrayList<>();
        writeQueue.drainTo(tasks, 100);  // 최대 100건씩 배치 처리
        if (!tasks.isEmpty()) {
            productRepository.batchUpdate(tasks);
        }
    }
}
```

**장점:** 쓰기 응답 속도가 극단적으로 빠르다. 배치 처리로 DB 부하를 줄일 수 있다.

**단점:** 캐시 서버 장애 시 데이터 유실 위험이 있다. 구현 복잡도가 높다. 주로 로그 집계, 조회수 같이 유실이 허용되는 데이터에 적합하다.

### Refresh-Ahead (프리페치)

TTL 만료 전에 미리 캐시를 갱신하는 전략이다.

```java
@Component
public class RefreshAheadCache {

    // 남은 TTL이 전체 TTL의 20% 이하일 때 비동기 갱신
    private static final double REFRESH_THRESHOLD = 0.2;

    public ProductDto getProduct(Long id) {
        String cacheKey = "product:" + id;
        Duration ttl = redisTemplate.getExpire(cacheKey, TimeUnit.SECONDS) != null
            ? Duration.ofSeconds(redisTemplate.getExpire(cacheKey, TimeUnit.SECONDS))
            : Duration.ZERO;

        String cached = redisTemplate.opsForValue().get(cacheKey);

        if (cached != null) {
            // 남은 TTL이 임계값 이하이면 비동기 갱신 트리거
            long remainingSeconds = ttl.getSeconds();
            if (remainingSeconds < TOTAL_TTL_SECONDS * REFRESH_THRESHOLD) {
                CompletableFuture.runAsync(() -> refreshCache(id, cacheKey));
            }
            return objectMapper.readValue(cached, ProductDto.class);
        }

        return loadAndCache(id, cacheKey);
    }

    private void refreshCache(Long id, String cacheKey) {
        ProductDto fresh = productRepository.findById(id)
            .map(ProductDto::from)
            .orElse(null);
        if (fresh != null) {
            redisTemplate.opsForValue().set(cacheKey,
                objectMapper.writeValueAsString(fresh),
                Duration.ofMinutes(5));
        }
    }
}
```

**장점:** TTL 만료로 인한 갑작스러운 캐시 미스(cold miss)를 방지한다. 사용자 응답 속도가 항상 일정하게 유지된다.

**단점:** 갱신 중 잠깐 동안 오래된 데이터가 반환될 수 있다. 실제로 필요하지 않은 데이터를 미리 로드하는 비용이 발생한다.

---

## 4. TTL 설계: 단순한 숫자가 아닌 전략

TTL을 "5분으로 설정"하는 것은 시작일 뿐이다. 잘못된 TTL 설계는 캐시 스탬피드나 오래된 데이터 노출로 이어진다.

### TTL 값 결정 원칙

**데이터 변경 빈도에 따른 TTL 기준:**

| 데이터 유형 | 변경 빈도 | 권장 TTL |
|---|---|---|
| 국가/통화 코드 | 거의 없음 | 24시간 ~ 7일 |
| 상품 정보 | 낮음 | 10분 ~ 1시간 |
| 사용자 프로필 | 보통 | 5분 ~ 30분 |
| 재고/가격 | 높음 | 30초 ~ 2분 |
| 실시간 데이터 | 매우 높음 | 5초 ~ 30초 |

### TTL 지터(Jitter) 적용

모든 캐시가 동시에 만료되면 대규모 캐시 미스가 한꺼번에 발생한다. 이를 **Thunder Herd** 또는 **캐시 스탬피드**라고 한다.

```java
public Duration calculateTtlWithJitter(Duration baseTtl) {
    // 기본 TTL의 ±10% 범위에서 무작위로 지터 추가
    long baseSeconds = baseTtl.getSeconds();
    long jitterRange = baseSeconds / 10;  // 10% 범위
    long jitter = ThreadLocalRandom.current().nextLong(-jitterRange, jitterRange);
    return Duration.ofSeconds(baseSeconds + jitter);
}

// 사용 예시
public void cacheProduct(Long id, ProductDto dto) {
    Duration ttl = calculateTtlWithJitter(Duration.ofMinutes(5));
    redisTemplate.opsForValue().set(
        "product:" + id,
        objectMapper.writeValueAsString(dto),
        ttl
    );
}
```

### 캐시 워밍(Cache Warming)

서버 재시작 후 캐시가 비어있는 상태에서 모든 요청이 DB로 직접 향하는 현상을 콜드 스타트 문제라고 한다.

```java
@Component
public class CacheWarmer implements ApplicationListener<ApplicationReadyEvent> {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        log.info("캐시 워밍 시작...");
        warmTopProducts();
        log.info("캐시 워밍 완료");
    }

    private void warmTopProducts() {
        // 최근 7일간 조회 상위 1,000개 상품을 미리 캐싱
        List<Long> topProductIds = analyticsRepository.getTopViewedProductIds(1000, 7);

        // 배치로 DB 조회 후 Redis에 파이프라인으로 적재
        List<Product> products = productRepository.findAllById(topProductIds);

        redisTemplate.executePipelined((RedisCallback<?>) connection -> {
            products.forEach(product -> {
                String key = "product:" + product.getId();
                String value = objectMapper.writeValueAsString(ProductDto.from(product));
                Duration ttl = calculateTtlWithJitter(Duration.ofMinutes(10));
                connection.setEx(key.getBytes(), ttl.getSeconds(),
                    value.getBytes());
            });
            return null;
        });
    }
}
```

---

## 5. 핫스팟 문제와 캐시 스탬피드 방어

### 핫스팟 키 문제

특정 키에 요청이 집중되는 현상이다. 예를 들어 인기 상품 ID `12345`에 초당 10만 건의 요청이 몰리면, Redis 클러스터의 해당 슬롯이 병목이 된다.

**해결책 1: 로컬 캐시로 분산**

```java
@Service
public class HotspotAwareCacheService {

    private final Cache<String, ProductDto> localCache = Caffeine.newBuilder()
        .maximumSize(100)                         // 핫스팟 상위 100개만
        .expireAfterWrite(30, TimeUnit.SECONDS)   // 짧은 TTL로 신선도 유지
        .build();

    private final RedisTemplate<String, String> redisTemplate;

    public ProductDto getProduct(Long id) {
        String key = "product:" + id;

        // L1: 로컬 캐시 우선 조회 (네트워크 없음)
        ProductDto local = localCache.getIfPresent(key);
        if (local != null) return local;

        // L2: Redis 조회
        String cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            ProductDto dto = objectMapper.readValue(cached, ProductDto.class);
            localCache.put(key, dto);  // L1에도 저장
            return dto;
        }

        // DB 폴백
        return loadFromDb(id, key);
    }
}
```

**해결책 2: 키 샤딩(Key Sharding)**

```java
// 하나의 핫스팟 키를 여러 복제본으로 분산
public String getHotspotKey(String baseKey, int shardCount) {
    int shard = ThreadLocalRandom.current().nextInt(shardCount);
    return baseKey + ":shard:" + shard;
}

public void setWithSharding(String baseKey, String value,
                             Duration ttl, int shardCount) {
    // 모든 샤드에 동일한 값 저장
    for (int i = 0; i < shardCount; i++) {
        redisTemplate.opsForValue().set(
            baseKey + ":shard:" + i, value, ttl);
    }
}
```

### 캐시 스탬피드(Stampede) 방어

동시에 여러 요청이 캐시 미스를 감지하고 모두 DB로 향하는 현상이다.

**해결책: 분산 락(Distributed Lock)**

```java
@Service
public class StampedeGuardService {

    private static final String LOCK_PREFIX = "lock:";
    private static final Duration LOCK_TTL = Duration.ofSeconds(10);

    public ProductDto getProductWithLock(Long id) {
        String cacheKey = "product:" + id;
        String lockKey = LOCK_PREFIX + cacheKey;

        // 1. 캐시 조회
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return objectMapper.readValue(cached, ProductDto.class);
        }

        // 2. 락 획득 시도 (SET NX EX)
        Boolean locked = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "1", LOCK_TTL);

        if (Boolean.TRUE.equals(locked)) {
            try {
                // 락 획득 성공: DB 조회 후 캐시 저장
                return loadAndCache(id, cacheKey);
            } finally {
                redisTemplate.delete(lockKey);
            }
        } else {
            // 락 획득 실패: 짧게 대기 후 캐시 재조회
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            String retried = redisTemplate.opsForValue().get(cacheKey);
            if (retried != null) {
                return objectMapper.readValue(retried, ProductDto.class);
            }
            // 여전히 없으면 직접 DB 조회 (최후 수단)
            return loadFromDb(id);
        }
    }
}
```

**더 간단한 해결책: Probabilistic Early Expiration**

```java
// 만료 직전의 일부 요청만 미리 갱신하도록 확률적으로 처리
public ProductDto getWithProbabilisticRefresh(Long id) {
    String key = "product:" + id;
    Long ttlSeconds = redisTemplate.getExpire(key, TimeUnit.SECONDS);
    String cached = redisTemplate.opsForValue().get(key);

    if (cached == null) {
        return loadAndCache(id, key);
    }

    // 남은 TTL이 30초 이하이고, 10% 확률로 미리 갱신
    if (ttlSeconds != null && ttlSeconds < 30
            && Math.random() < 0.1) {
        CompletableFuture.runAsync(() -> loadAndCache(id, key));
    }

    return objectMapper.readValue(cached, ProductDto.class);
}
```

---

## 6. 캐시 무효화 전략

Phil Karlton의 유명한 말이 있다: "컴퓨터 과학에서 가장 어려운 두 가지는 캐시 무효화와 이름 짓기다."

캐시 무효화를 잘못 설계하면 데이터 불일치나 불필요한 전체 캐시 초기화로 이어진다.

### 전략 1: TTL 기반 만료 (가장 단순)

데이터 일관성보다 성능이 중요하고, 약간의 지연 반영이 허용될 때 사용한다.

```bash
# Redis에서 TTL 설정
SET product:1234 '{"id":1234,"name":"노트북"}' EX 300

# 남은 TTL 확인
TTL product:1234
```

장점은 구현이 매우 단순하다는 것이다. 단점은 TTL이 만료되기 전까지 오래된 데이터가 반환될 수 있다는 것이다.

### 전략 2: 이벤트 기반 무효화

데이터가 변경될 때 이벤트를 발행하고, 해당 이벤트를 구독하는 컴포넌트가 캐시를 삭제한다.

```java
// 이벤트 발행
@Service
public class ProductCommandService {

    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public ProductDto updateProduct(Long id, UpdateProductRequest req) {
        Product product = productRepository.findById(id).orElseThrow();
        product.update(req);
        productRepository.save(product);

        // 트랜잭션 커밋 후 이벤트 발행 (데이터 불일치 방지)
        eventPublisher.publishEvent(new ProductUpdatedEvent(id));
        return ProductDto.from(product);
    }
}

// 이벤트 구독 및 캐시 무효화
@Component
public class ProductCacheInvalidator {

    private final RedisTemplate<String, String> redisTemplate;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onProductUpdated(ProductUpdatedEvent event) {
        String cacheKey = "product:" + event.getProductId();
        redisTemplate.delete(cacheKey);
        log.info("캐시 무효화 완료: {}", cacheKey);
    }
}
```

`@TransactionalEventListener(phase = AFTER_COMMIT)`을 반드시 사용해야 한다. 트랜잭션이 롤백될 경우 캐시를 삭제하지 않아야 하기 때문이다.

### 전략 3: 버전 키(Versioned Key) 기반

캐시 키에 버전 번호를 포함시켜, 버전이 바뀌면 자동으로 새로운 키로 전환한다.

```java
@Service
public class VersionedCacheService {

    private static final String VERSION_KEY_PREFIX = "version:product:";
    private static final String CACHE_KEY_TEMPLATE = "product:v%d:%d";

    public ProductDto getProduct(Long id) {
        // 현재 버전 조회
        Long version = getCurrentVersion(id);
        String cacheKey = String.format(CACHE_KEY_TEMPLATE, version, id);

        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return objectMapper.readValue(cached, ProductDto.class);
        }

        ProductDto dto = productRepository.findById(id)
            .map(ProductDto::from).orElseThrow();
        redisTemplate.opsForValue().set(cacheKey,
            objectMapper.writeValueAsString(dto),
            Duration.ofMinutes(30));
        return dto;
    }

    public void invalidateProduct(Long id) {
        // 버전 증가만으로 자동 무효화 (삭제 불필요)
        String versionKey = VERSION_KEY_PREFIX + id;
        redisTemplate.opsForValue().increment(versionKey);
    }

    private Long getCurrentVersion(Long id) {
        String versionKey = VERSION_KEY_PREFIX + id;
        String version = redisTemplate.opsForValue().get(versionKey);
        return version != null ? Long.parseLong(version) : 1L;
    }
}
```

이 방식은 실제 캐시 데이터를 삭제하지 않아도 된다. 버전 번호가 바뀌면 새 키가 생성되어 자연스럽게 이전 캐시를 무시한다. 단, 오래된 버전의 캐시가 TTL이 만료될 때까지 메모리에 남아있는다.

### 전략 4: 태그 기반 무효화

관련된 여러 캐시를 그룹으로 묶어 한 번에 무효화한다.

```java
@Service
public class TagBasedCacheService {

    // 태그와 캐시 키의 매핑을 Set으로 관리
    private static final String TAG_PREFIX = "tag:";

    public void cacheWithTags(String cacheKey, Object value,
                               Duration ttl, String... tags) {
        // 1. 실제 데이터 저장
        redisTemplate.opsForValue().set(cacheKey,
            objectMapper.writeValueAsString(value), ttl);

        // 2. 각 태그에 이 캐시 키 등록
        for (String tag : tags) {
            redisTemplate.opsForSet().add(TAG_PREFIX + tag, cacheKey);
        }
    }

    public void invalidateByTag(String tag) {
        String tagKey = TAG_PREFIX + tag;

        // 태그에 등록된 모든 캐시 키 삭제
        Set<String> keys = redisTemplate.opsForSet().members(tagKey);
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
        redisTemplate.delete(tagKey);  // 태그 자체도 삭제
    }
}

// 사용 예시: 특정 카테고리의 모든 상품 캐시를 한 번에 무효화
@Service
public class ProductService {

    public ProductDto getProductWithTag(Long id) {
        Product product = productRepository.findById(id).orElseThrow();
        String cacheKey = "product:" + id;
        String categoryTag = "category:" + product.getCategoryId();

        tagBasedCacheService.cacheWithTags(cacheKey,
            ProductDto.from(product),
            Duration.ofMinutes(10),
            categoryTag, "all-products");

        return ProductDto.from(product);
    }

    public void updateCategory(Long categoryId) {
        // 해당 카테고리의 모든 상품 캐시 일괄 무효화
        tagBasedCacheService.invalidateByTag("category:" + categoryId);
    }
}
```

---

## 7. 피해야 할 안티패턴

### 안티패턴 1: 캐시 키 충돌

```java
// 나쁜 예: 다른 엔티티에 같은 키 형식 사용
redisTemplate.set("1234", productJson);
redisTemplate.set("1234", userJson);  // 같은 키로 덮어씀!

// 좋은 예: 네임스페이스 포함
redisTemplate.set("product:1234", productJson);
redisTemplate.set("user:1234", userJson);
```

### 안티패턴 2: 거대한 캐시 값

Redis는 단일 값이 512MB까지 저장 가능하지만, 10KB를 넘는 값은 네트워크 전송과 역직렬화 비용이 크다.

```java
// 나쁜 예: 관계된 모든 데이터를 한 키에 저장
ProductDetailDto detail = new ProductDetailDto(
    product, reviews, images, relatedProducts, ...  // 수백KB
);
redisTemplate.set("product:detail:" + id, serialize(detail));

// 좋은 예: 조각별로 분리하여 필요한 것만 로드
redisTemplate.set("product:basic:" + id, serialize(basicInfo));
redisTemplate.set("product:reviews:" + id, serialize(reviews));
redisTemplate.set("product:images:" + id, serialize(images));
```

### 안티패턴 3: DB와 캐시의 비원자적 업데이트

```java
// 나쁜 예: DB 업데이트와 캐시 삭제가 비원자적
public void updateProduct(Long id, UpdateProductRequest req) {
    productRepository.save(updated);    // 성공
    // 여기서 서버가 죽으면?
    redisCache.delete("product:" + id); // 실행 안 됨 → 캐시 오염!
}

// 좋은 예: 트랜잭션 커밋 후 이벤트로 캐시 무효화
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onProductUpdated(ProductUpdatedEvent event) {
    redisCache.delete("product:" + event.getProductId());
}
```

### 안티패턴 4: Null 값 캐싱 누락 (Cache Penetration)

존재하지 않는 키로 반복 요청이 오면 매번 DB로 쿼리가 발생한다.

```java
// 나쁜 예: null 결과를 캐싱하지 않음
public ProductDto getProduct(Long id) {
    String cached = redisTemplate.get("product:" + id);
    if (cached != null) return deserialize(cached);

    Optional<Product> product = productRepository.findById(id);
    if (product.isEmpty()) return null;  // DB 미스를 캐싱하지 않음!

    // ...
}

// 좋은 예: null 센티넬 값을 캐싱
public Optional<ProductDto> getProduct(Long id) {
    String cacheKey = "product:" + id;
    String cached = redisTemplate.opsForValue().get(cacheKey);

    if ("NULL".equals(cached)) {
        return Optional.empty();  // 존재하지 않음이 캐싱됨
    }
    if (cached != null) {
        return Optional.of(deserialize(cached, ProductDto.class));
    }

    Optional<Product> product = productRepository.findById(id);
    if (product.isEmpty()) {
        // 짧은 TTL로 null 센티넬 저장 (5분)
        redisTemplate.opsForValue().set(cacheKey, "NULL", Duration.ofMinutes(5));
        return Optional.empty();
    }

    ProductDto dto = ProductDto.from(product.get());
    redisTemplate.opsForValue().set(cacheKey, serialize(dto), Duration.ofMinutes(10));
    return Optional.of(dto);
}
```

---

## 8. 실전 운영: 모니터링과 메트릭

캐시를 도입한 후 실제로 잘 작동하는지 확인하는 지표가 필요하다.

### 핵심 메트릭

**캐시 히트율(Hit Rate):** 가장 중요한 지표다. 80% 이상을 목표로 설정한다.

```java
// Micrometer를 통한 메트릭 수집
@Component
public class CacheMetricsCollector {

    private final MeterRegistry meterRegistry;
    private final Counter hitCounter;
    private final Counter missCounter;

    public CacheMetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.hitCounter = Counter.builder("cache.requests")
            .tag("result", "hit")
            .register(meterRegistry);
        this.missCounter = Counter.builder("cache.requests")
            .tag("result", "miss")
            .register(meterRegistry);
    }

    public void recordHit(String cacheName) {
        meterRegistry.counter("cache.requests",
            "cache", cacheName, "result", "hit").increment();
    }

    public void recordMiss(String cacheName) {
        meterRegistry.counter("cache.requests",
            "cache", cacheName, "result", "miss").increment();
    }
}
```

### Redis INFO 명령어로 상태 확인

```bash
# 전체 통계
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# 결과 예시:
keyspace_hits:15234789
keyspace_misses:342156
# 히트율 = 15234789 / (15234789 + 342156) = 97.8%

# 메모리 사용량
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human"

# 연결 수
redis-cli INFO clients
```

### 알람 설정 기준

| 메트릭 | 경고(Warning) | 위험(Critical) |
|---|---|---|
| 캐시 히트율 | < 80% | < 60% |
| Redis 메모리 사용률 | > 70% | > 85% |
| 응답 지연 (P99) | > 10ms | > 50ms |
| 연결 수 | > 1,000 | > 5,000 |

---

## 정리

캐싱은 단순히 "빠르게 만드는 기술"이 아니다.

데이터의 특성(변경 빈도, 일관성 요구사항, 크기)에 맞는 패턴을 선택하고, TTL 지터로 스탬피드를 방지하며, 이벤트 기반 무효화로 데이터 불일치를 최소화하는 것이 핵심이다.

로컬 캐시와 분산 캐시를 L1/L2로 계층화하면 성능과 일관성을 동시에 얻을 수 있다. 그리고 무엇보다 캐시 히트율, 메모리 사용량, 응답 지연을 꾸준히 관찰해야 한다. 관찰하지 않는 캐시는 언제든 시한폭탄이 될 수 있다.

---

## 참고 자료

1. [Caffeine Cache GitHub - High Performance Caching Library for Java](https://github.com/ben-manes/caffeine)
2. [Redis Documentation - Caching Patterns](https://redis.io/docs/manual/patterns/)
3. [Martin Fowler - Cache-Aside Pattern](https://martinfowler.com/bliki/TwoHardThings.html)
4. [AWS Architecture Blog - Caching Best Practices](https://aws.amazon.com/caching/best-practices/)
5. [Google Cloud - Designing Distributed Systems: Cache Stampede Prevention](https://cloud.google.com/architecture/best-practices-for-running-cost-effective-kubernetes-applications-on-gke)
6. [Spring Framework Documentation - Cache Abstraction](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache)
