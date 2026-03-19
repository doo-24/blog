---
title: "[인증과 보안] 5편 — Rate Limiting과 DDoS 방어: 과부하 공격 대응"
date: 2026-03-17T22:03:00+09:00
draft: false
tags: ["Rate Limiting", "DDoS", "토큰 버킷", "WAF", "보안", "서버"]
series: ["인증과 보안"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 4
summary: "Rate Limiting 알고리즘(토큰 버킷, 리키 버킷, 슬라이딩 윈도우), 분산 Rate Limiting(Redis), API Gateway 스로틀링, DDoS 유형과 CloudFlare/AWS Shield 방어까지"
---

2016년 깃허브 API는 예고 없이 불특정 IP로부터 초당 수천 건의 요청을 받기 시작했다. 일반 사용자 요청과 구분되지 않는 정상처럼 보이는 트래픽이었지만, 결과는 API 서버 전체 응답 지연이었다.

그 이후로 깃허브는 인증 토큰 기반 Rate Limiting을 대폭 강화했다.

현대 서버 엔지니어링에서 Rate Limiting은 "나쁜 사용자를 차단한다"는 단순한 개념이 아니다. 시스템의 처리 한계를 명시적으로 정의하고, 공정하게 자원을 분배하며, 의도적이든 우발적이든 과부하로부터 서비스를 지키는 방어선이다.

이 글에서는 Rate Limiting의 핵심 알고리즘부터 분산 환경 구현, API Gateway 통합, 그리고 DDoS 방어 전략까지 실전 중심으로 다룬다.

---

## Rate Limiting이란 무엇인가

Rate Limiting은 단위 시간 내에 허용되는 요청 수를 제한하는 메커니즘이다. 목적은 크게 세 가지다.

- **서비스 안정성 보장**: 특정 클라이언트의 과도한 요청이 전체 서비스에 영향을 주지 않도록 격리
- **자원 공정 배분**: 무거운 소수 사용자가 경량 다수 사용자의 몫을 빼앗지 못하게 방지
- **악의적 공격 차단**: 브루트포스, 크리덴셜 스터핑, DDoS 공격의 효과를 감소

Rate Limiting을 설계할 때는 두 가지 질문에 먼저 답해야 한다. "**무엇을 기준으로 제한하는가**"(IP, 사용자 ID, API 키, 엔드포인트)와 "**어떤 시간 단위로 카운팅하는가**"(초, 분, 시간)다. 이 두 축이 결정되면 알고리즘 선택이 따라온다.

---

## Rate Limiting 알고리즘

### 고정 윈도우 카운터 (Fixed Window Counter)

가장 단순한 방식이다. 시간을 고정된 창(예: 1분)으로 나누고, 각 창 내의 요청 수를 카운팅한다. 한도를 초과하면 창이 리셋될 때까지 거부한다.

```
[00:00 ~ 00:59] 요청 100개 허용
[01:00 ~ 01:59] 카운터 리셋, 다시 100개 허용
```

**문제점**: 창 경계(boundary) 부근에서 2배의 트래픽이 통과할 수 있다. 00:59에 100개, 01:00에 100개가 연속으로 들어오면 2초 사이에 200개 요청이 처리된다. 이를 "경계 폭발(burst at boundary)" 문제라 한다.

```java
// 고정 윈도우 카운터 - Redis 구현
public boolean isAllowed(String key, int limit, int windowSeconds) {
    String redisKey = "rate:" + key + ":" + (System.currentTimeMillis() / (windowSeconds * 1000L));

    try (Jedis jedis = jedisPool.getResource()) {
        long count = jedis.incr(redisKey);
        if (count == 1) {
            jedis.expire(redisKey, windowSeconds);
        }
        return count <= limit;
    }
}
```

### 슬라이딩 윈도우 로그 (Sliding Window Log)

고정 윈도우의 경계 폭발 문제를 해결한다. 요청마다 타임스탬프를 저장하고, 현재 시각으로부터 윈도우 크기만큼 이전의 타임스탬프만 유효하게 본다.

```java
public boolean isAllowed(String userId, int limit, long windowMs) {
    long now = System.currentTimeMillis();
    long windowStart = now - windowMs;
    String key = "rate:log:" + userId;

    try (Jedis jedis = jedisPool.getResource()) {
        Pipeline pipe = jedis.pipelined();
        // 만료된 엔트리 제거
        pipe.zremrangeByScore(key, 0, windowStart);
        // 현재 요청 추가
        pipe.zadd(key, now, now + ":" + UUID.randomUUID());
        // 현재 윈도우 내 요청 수 확인
        Response<Long> count = pipe.zcard(key);
        pipe.expire(key, (int)(windowMs / 1000) + 1);
        pipe.sync();

        return count.get() <= limit;
    }
}
```

**장점**: 정확한 슬라이딩 윈도우를 제공한다. **단점**: 모든 요청의 타임스탬프를 저장해야 하므로 메모리 사용량이 크다. 트래픽이 많은 서비스에서는 부담이 된다.

### 슬라이딩 윈도우 카운터 (Sliding Window Counter)

고정 윈도우와 슬라이딩 로그의 절충점이다. 이전 창과 현재 창의 요청 수를 가중 평균으로 계산해 슬라이딩 효과를 근사한다.

```
현재 허용 요청 수 = 이전 창 카운트 × (1 - 현재 창에서 경과한 비율) + 현재 창 카운트
```

예를 들어 1분 창에서 현재 시각이 창의 40% 지점이라면:

```
허용 요청 수 = 이전 창(80개) × 0.6 + 현재 창(30개) = 48 + 30 = 78개
```

```java
public boolean isAllowed(String key, int limit, long windowMs) {
    long now = System.currentTimeMillis();
    long currentWindow = now / windowMs;
    long previousWindow = currentWindow - 1;
    double elapsed = (double)(now % windowMs) / windowMs;

    try (Jedis jedis = jedisPool.getResource()) {
        String prevKey = "rate:sw:" + key + ":" + previousWindow;
        String currKey = "rate:sw:" + key + ":" + currentWindow;

        String prevCountStr = jedis.get(prevKey);
        long prevCount = prevCountStr != null ? Long.parseLong(prevCountStr) : 0;

        double weightedCount = prevCount * (1.0 - elapsed);

        long currCount = jedis.incr(currKey);
        jedis.expire(currKey, (int)(windowMs / 1000) * 2);

        return (weightedCount + currCount) <= limit;
    }
}
```

**실무에서 가장 많이 선택되는 방식**이다. 메모리 효율이 좋고, 경계 폭발 문제를 실용적 수준으로 제어한다.

### 토큰 버킷 (Token Bucket)

개념상 가장 직관적인 알고리즘이다. 버킷에는 최대 용량만큼 토큰이 채워지고, 일정 속도로 토큰이 보충된다. 요청이 들어오면 토큰을 하나 소비하고, 토큰이 없으면 거부한다.

- 버킷 용량(capacity): 최대 허용 버스트 크기
- 보충 속도(refill rate): 초당 추가되는 토큰 수

```java
@Component
public class TokenBucketRateLimiter {

    private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();

    public boolean tryAcquire(String key, int capacity, double refillRatePerSecond) {
        TokenBucket bucket = buckets.computeIfAbsent(key,
            k -> new TokenBucket(capacity, refillRatePerSecond));
        return bucket.tryConsume();
    }

    private static class TokenBucket {
        private final int capacity;
        private final double refillRatePerMs;
        private double tokens;
        private long lastRefillTimestamp;

        TokenBucket(int capacity, double refillRatePerSecond) {
            this.capacity = capacity;
            this.refillRatePerMs = refillRatePerSecond / 1000.0;
            this.tokens = capacity;
            this.lastRefillTimestamp = System.currentTimeMillis();
        }

        synchronized boolean tryConsume() {
            refill();
            if (tokens >= 1.0) {
                tokens -= 1.0;
                return true;
            }
            return false;
        }

        private void refill() {
            long now = System.currentTimeMillis();
            double tokensToAdd = (now - lastRefillTimestamp) * refillRatePerMs;
            tokens = Math.min(capacity, tokens + tokensToAdd);
            lastRefillTimestamp = now;
        }
    }
}
```

**장점**: 버스트 트래픽을 자연스럽게 허용하면서도 장기 평균 속도를 제한한다. AWS API Gateway, Stripe 등 대형 서비스가 채택하는 이유다. **단점**: 분산 환경에서 버킷 상태 동기화가 필요하다.

### 리키 버킷 (Leaky Bucket)

큐(queue) 기반 알고리즘이다. 요청이 들어오면 큐에 쌓이고, 큐는 일정한 속도로 처리된다. 큐가 꽉 차면 새 요청을 거부한다.

```
들어오는 요청 (불규칙) -> [큐 버퍼] -> 일정 속도로 처리
```

**토큰 버킷 vs 리키 버킷**:

| 특성 | 토큰 버킷 | 리키 버킷 |
|------|-----------|-----------|
| 버스트 허용 | 예 (버킷 용량만큼) | 아니오 (큐 용량만큼 대기) |
| 출력 속도 | 불규칙 (버스트 가능) | 일정 |
| 적합한 용도 | API 접근 제한 | 네트워크 트래픽 쉐이핑 |

리키 버킷은 출력 속도를 완전히 평탄화해야 하는 네트워크 QoS나 결제 처리 같은 시나리오에 적합하다.

---

## 분산 Rate Limiting

단일 서버에서는 인메모리로 Rate Limiting 상태를 관리할 수 있다. 하지만 서비스가 수평 확장되면 각 서버 인스턴스가 독립적으로 카운트를 관리하므로, 동일 사용자가 여러 서버를 거치면 실제 허용량의 N배 요청이 통과한다.

### Redis를 이용한 분산 Rate Limiting

Redis는 단일 스레드 원자 연산, 만료 시간 지원, 낮은 지연 시간 덕분에 분산 Rate Limiting의 표준 백엔드다.

**Lua 스크립트로 원자적 처리**

Redis에서 INCR + EXPIRE를 별도 명령으로 보내면 두 명령 사이에 다른 요청이 끼어들 수 있다(race condition). Lua 스크립트는 Redis에서 원자적으로 실행된다.

```lua
-- rate_limit.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end

if current > limit then
    return 0
else
    return 1
end
```

```java
@Component
public class RedisRateLimiter {

    private final RedisTemplate<String, String> redisTemplate;
    private final DefaultRedisScript<Long> rateLimitScript;

    public RedisRateLimiter(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
        this.rateLimitScript = new DefaultRedisScript<>();
        this.rateLimitScript.setScriptText(loadLuaScript());
        this.rateLimitScript.setResultType(Long.class);
    }

    public boolean isAllowed(String identifier, int limit, int windowSeconds) {
        String key = "rate_limit:" + identifier;
        Long result = redisTemplate.execute(
            rateLimitScript,
            Collections.singletonList(key),
            String.valueOf(limit),
            String.valueOf(windowSeconds)
        );
        return result != null && result == 1L;
    }

    private String loadLuaScript() {
        return "local key = KEYS[1]\n" +
               "local limit = tonumber(ARGV[1])\n" +
               "local window = tonumber(ARGV[2])\n" +
               "local current = redis.call('INCR', key)\n" +
               "if current == 1 then\n" +
               "    redis.call('EXPIRE', key, window)\n" +
               "end\n" +
               "if current > limit then\n" +
               "    return 0\n" +
               "else\n" +
               "    return 1\n" +
               "end";
    }
}
```

### 사용자별 / IP별 / 엔드포인트별 제한

실무에서는 단일 Rate Limit 정책이 아니라 다층(multi-tier) 정책을 사용한다.

```java
@Component
public class MultiTierRateLimiter {

    private final RedisRateLimiter redisRateLimiter;

    // IP당: 분당 1000 요청 (DDoS 1차 방어)
    // 사용자당: 분당 100 요청 (공정 사용)
    // 엔드포인트당: /login 분당 10 요청 (브루트포스 방어)

    public RateLimitResult check(HttpServletRequest request, String userId) {
        String ip = extractClientIp(request);
        String endpoint = request.getRequestURI();

        // 1단계: IP 레벨 체크
        if (!redisRateLimiter.isAllowed("ip:" + ip, 1000, 60)) {
            return RateLimitResult.denied("IP_RATE_LIMIT", 60);
        }

        // 2단계: 사용자 레벨 체크 (인증된 경우만)
        if (userId != null && !redisRateLimiter.isAllowed("user:" + userId, 100, 60)) {
            return RateLimitResult.denied("USER_RATE_LIMIT", 60);
        }

        // 3단계: 엔드포인트별 체크
        if (endpoint.startsWith("/api/auth/login")) {
            String loginKey = userId != null ? "login:user:" + userId : "login:ip:" + ip;
            if (!redisRateLimiter.isAllowed(loginKey, 10, 60)) {
                return RateLimitResult.denied("LOGIN_RATE_LIMIT", 60);
            }
        }

        return RateLimitResult.allowed();
    }

    private String extractClientIp(HttpServletRequest request) {
        // 프록시/로드 밸런서 헤더 우선 확인
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        String xRealIp = request.getHeader("X-Real-IP");
        if (xRealIp != null && !xRealIp.isEmpty()) {
            return xRealIp;
        }
        return request.getRemoteAddr();
    }
}
```

### Spring Boot 인터셉터로 통합

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    private final MultiTierRateLimiter rateLimiter;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String userId = (String) request.getAttribute("authenticatedUserId");
        RateLimitResult result = rateLimiter.check(request, userId);

        if (!result.isAllowed()) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value()); // 429
            response.setHeader("Retry-After", String.valueOf(result.getRetryAfterSeconds()));
            response.setHeader("X-RateLimit-Reason", result.getReason());
            response.getWriter().write("{\"error\": \"Too Many Requests\"}");
            return false;
        }

        return true;
    }
}
```

**HTTP 429 상태 코드**를 반드시 사용해야 한다. 403(Forbidden)이나 503(Service Unavailable)으로 응답하면 클라이언트가 Rate Limiting으로 인식하지 못하고 재시도 로직을 잘못 구성한다. `Retry-After` 헤더도 함께 내려줘야 클라이언트가 언제 재시도할지 알 수 있다.

---

## API Gateway 레벨 스로틀링

애플리케이션 서버에서 Rate Limiting을 구현하는 것도 좋지만, 요청이 애플리케이션까지 도달하기 전에 차단하는 것이 더 효율적이다. API Gateway가 이 역할을 담당한다.

### AWS API Gateway 스로틀링

AWS API Gateway는 두 단계의 스로틀링을 제공한다.

- **계정 레벨**: 리전당 기본 초당 10,000 요청, 버스트 5,000
- **스테이지/메서드 레벨**: 각 API 스테이지 또는 개별 메서드별 설정

```yaml
# SAM/CloudFormation으로 메서드별 Rate Limit 설정
Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      MethodSettings:
        - ResourcePath: "/users"
          HttpMethod: "POST"
          ThrottlingRateLimit: 100    # 초당 100 요청
          ThrottlingBurstLimit: 200   # 버스트 허용 200
        - ResourcePath: "/login"
          HttpMethod: "POST"
          ThrottlingRateLimit: 10
          ThrottlingBurstLimit: 20
```

API Gateway 스로틀링이 발동하면 503이 아닌 **429 Too Many Requests**를 반환한다(2023년 이후 기본). 이전에는 503을 반환했는데, 많은 클라이언트가 서버 오류로 오인해 즉시 재시도를 반복하는 문제가 있었다.

### Kong Gateway Rate Limiting 플러그인

오픈소스 API Gateway인 Kong은 Rate Limiting 플러그인을 내장한다.

```yaml
plugins:
  - name: rate-limiting
    config:
      minute: 100          # 분당 100 요청
      hour: 1000           # 시간당 1000 요청
      policy: redis        # Redis 백엔드 사용
      redis_host: redis
      redis_port: 6379
      limit_by: consumer   # 소비자(인증 사용자) 기준
      hide_client_headers: false  # X-RateLimit-* 헤더 노출
```

Kong은 `limit_by`로 `consumer`, `credential`, `ip`, `service`, `header` 등 다양한 기준을 지원한다. 분산 환경에서 `policy: redis`를 지정하면 모든 Kong 노드가 동일한 Redis를 공유한다.

---

## WAF (Web Application Firewall)

Rate Limiting이 "얼마나 자주"를 제한한다면, WAF는 "어떤 내용"을 차단한다. 둘은 상호 보완적이다.

### WAF의 핵심 기능

WAF는 HTTP/HTTPS 트래픽을 검사해 알려진 공격 패턴을 차단한다.

- **SQL Injection 탐지**: `' OR 1=1 --` 같은 패턴 필터링
- **XSS 차단**: `<script>alert()</script>` 패턴 탐지
- **OWASP Top 10 방어**: 주요 웹 취약점 규칙셋 적용
- **IP 평판 기반 차단**: 알려진 악성 IP, 토르(Tor) 출구 노드 차단
- **지리적 차단(Geo-blocking)**: 특정 국가/지역 트래픽 제한

### AWS WAF 규칙 예시

```json
{
  "Rules": [
    {
      "Name": "AWSManagedRulesCommonRuleSet",
      "Priority": 1,
      "Statement": {
        "ManagedRuleGroupStatement": {
          "VendorName": "AWS",
          "Name": "AWSManagedRulesCommonRuleSet"
        }
      },
      "Action": { "Block": {} }
    },
    {
      "Name": "LoginRateLimit",
      "Priority": 2,
      "Statement": {
        "RateBasedStatement": {
          "Limit": 100,
          "AggregateKeyType": "IP",
          "ScopeDownStatement": {
            "ByteMatchStatement": {
              "FieldToMatch": { "UriPath": {} },
              "PositionalConstraint": "STARTS_WITH",
              "SearchString": "/api/auth/login"
            }
          }
        }
      },
      "Action": { "Block": {} }
    }
  ]
}
```

WAF는 Rate Limiting보다 상위 레이어에서 동작한다. 요청이 WAF에서 차단되면 애플리케이션 서버는 물론 API Gateway에도 도달하지 않는다.

---

## DDoS 공격의 유형과 방어

### Volumetric Attack (대용량 공격)

네트워크 대역폭을 포화시켜 정상 트래픽이 진입하지 못하게 한다.

- **UDP Flood**: 대량의 UDP 패킷으로 대역폭 고갈
- **ICMP Flood (Ping Flood)**: ICMP 에코 요청으로 대역폭 소진
- **DNS Amplification**: 작은 DNS 요청으로 수백 배 큰 응답을 피해자 IP로 유도

Volumetric 공격의 규모는 수백 Gbps에서 Tbps 수준이다. 단일 서비스 운영자가 자체적으로 방어하기 불가능한 규모이며, 업스트림(ISP 또는 CDN)에서 흡수해야 한다.

### Protocol Attack (프로토콜 공격)

네트워크 프로토콜 취약점을 이용해 서버 리소스를 고갈시킨다.

- **SYN Flood**: TCP 3-way handshake의 SYN만 보내고 ACK를 보내지 않아 half-open 연결을 쌓음
- **Smurf Attack**: 브로드캐스트 주소로 ICMP 요청을 보내 응답이 피해자에게 집중
- **Fragmentation Attack**: 비정상 IP 단편화로 재조합 처리를 과부하

SYN Flood 대응으로 운영 체제는 **SYN Cookies**를 지원한다. half-open 연결 테이블을 쌓지 않고 암호화 쿠키로 상태를 관리해 메모리 고갈을 방지한다.

```bash
# Linux에서 SYN Cookies 활성화
sysctl -w net.ipv4.tcp_syncookies=1
# 영구 설정
echo "net.ipv4.tcp_syncookies = 1" >> /etc/sysctl.conf
```

### Application Layer Attack (애플리케이션 레이어 공격, L7 DDoS)

HTTP/HTTPS 레이어에서 발생하는 공격으로, 정상 트래픽과 구분이 어렵다.

- **HTTP Flood**: 정상적인 GET/POST 요청을 대량으로 전송
- **Slowloris**: HTTP 헤더를 느리게 전송해 연결을 오래 점유, 서버의 연결 한도 소진
- **ReDoS**: 정규식 처리에 취약한 입력으로 CPU 고갈
- **Cache Busting**: 캐시 불가능한 파라미터를 붙여 모든 요청이 원본 서버에 도달하게 유도

L7 공격은 개별 요청이 합법적으로 보이기 때문에 **행동 패턴 분석**이 필요하다. 동일 User-Agent에서 비정상적인 빈도, 비정상적인 요청 경로 패턴, 단일 IP에서 여러 계정 접근 시도 등을 탐지한다.

---

## CloudFlare를 이용한 DDoS 방어

CloudFlare는 전 세계 200개 이상의 PoP(Point of Presence)를 통해 트래픽을 필터링한다. 공격 트래픽이 오리진 서버에 도달하기 전에 CloudFlare 네트워크에서 흡수한다.

### CloudFlare Rate Limiting 설정

```
Rules > Rate Limiting Rules

조건:
  - URI Path: /api/auth/*
  - Request Method: POST

특성(Characteristics):
  - IP 주소

임계값: 10 요청 / 60초

액션:
  - Block (403) 또는 Challenge (CAPTCHA)
  - 차단 지속 시간: 10분
```

### CloudFlare Under Attack Mode

초당 수만 건 이상의 트래픽이 감지되면 "I'm Under Attack Mode"를 활성화할 수 있다. 모든 방문자에게 JavaScript 챌린지를 제시하고, 브라우저 환경이 없는 봇은 통과하지 못한다.

```bash
# CloudFlare API로 Under Attack Mode 활성화
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/{zone_id}/settings/security_level" \
  -H "Authorization: Bearer {api_token}" \
  -H "Content-Type: application/json" \
  --data '{"value":"under_attack"}'
```

공격이 끝나면 즉시 `"high"` 또는 `"medium"`으로 복원해야 한다. `under_attack` 모드는 정상 사용자에게도 불편을 주기 때문이다.

---

## AWS Shield를 이용한 DDoS 방어

### AWS Shield Standard

모든 AWS 계정에 무료로 제공된다. L3/L4 공격(SYN Flood, UDP Flood, Reflection 공격)을 자동으로 감지하고 완화한다.

### AWS Shield Advanced

유료(월 3,000 USD + 데이터 전송 비용)이지만 엔터프라이즈 수준의 보호를 제공한다.

- **L7 공격 완화**: HTTP Flood 등 애플리케이션 레이어 공격 대응
- **DRT(DDoS Response Team) 접근**: AWS 전문가 팀에 24/7 지원 요청 가능
- **비용 보호**: DDoS로 인한 AWS 비용 급증을 크레딧으로 보상
- **상세 가시성**: 실시간 공격 진단 및 사후 분석 리포트

```hcl
# Terraform으로 Shield Advanced 보호 리소스 등록
resource "aws_shield_protection" "alb_protection" {
  name         = "alb-ddos-protection"
  resource_arn = aws_lb.main.arn
}

resource "aws_shield_protection" "cloudfront_protection" {
  name         = "cloudfront-ddos-protection"
  resource_arn = aws_cloudfront_distribution.main.arn
}
```

Shield Advanced는 CloudFront, ALB, Route 53, Elastic IP, Global Accelerator에 연동된다. 최전방(CloudFront)부터 오리진(ALB)까지 계층적 방어가 가능하다.

---

## Anti-pattern: 흔한 실수들

**1. X-Forwarded-For를 무조건 신뢰**

로드 밸런서나 프록시 뒤에 있는 서버는 `X-Forwarded-For` 헤더로 클라이언트 IP를 확인한다. 그런데 이 헤더는 클라이언트가 위조할 수 있다. `X-Forwarded-For: 1.2.3.4, 5.6.7.8`에서 첫 번째 IP(클라이언트 제공)를 그대로 쓰는 대신, 신뢰할 수 있는 프록시가 추가한 마지막 IP를 사용해야 한다.

```java
// 잘못된 방식
String clientIp = request.getHeader("X-Forwarded-For").split(",")[0]; // 위조 가능

// 올바른 방식: 알려진 프록시 수만큼 뒤에서 세기
String[] ips = request.getHeader("X-Forwarded-For").split(",");
// 신뢰하는 프록시 수가 1개라면:
String clientIp = ips[Math.max(0, ips.length - 2)].trim();
```

**2. Rate Limit 거부를 503으로 응답**

`503 Service Unavailable`은 서버가 일시적으로 처리 불가 상태임을 의미한다. Rate Limit으로 인한 거부는 `429 Too Many Requests`가 맞다. 클라이언트의 재시도 로직이 다르게 동작한다.

**3. Retry-After 없이 429만 반환**

클라이언트가 언제 재시도해야 할지 모르면 즉각적이고 반복적인 재시도가 발생한다. 이는 서버 부하를 오히려 악화시킨다.

```java
response.setHeader("Retry-After", "60"); // 초 단위
response.setHeader("X-RateLimit-Limit", "100");
response.setHeader("X-RateLimit-Remaining", "0");
response.setHeader("X-RateLimit-Reset", String.valueOf(resetTimestampEpoch));
```

**4. 인증 엔드포인트에 Rate Limit 미적용**

`/login`, `/register`, `/forgot-password`는 브루트포스 공격의 최우선 타깃이다. 이 엔드포인트는 성공/실패 여부와 관계없이 IP + 계정 식별자 조합으로 엄격하게 Rate Limit을 적용해야 한다.

**5. Rate Limiting 없이 WAF만 의존**

WAF는 알려진 패턴 기반으로 동작하고, Rate Limiting은 요청 빈도 기반으로 동작한다. 둘은 역할이 다르다. WAF 규칙에 없는 새로운 공격 패턴은 WAF를 통과하지만 Rate Limiting으로 잡힌다. 반드시 함께 운영해야 한다.

---

## 실전 아키텍처: 계층적 방어

현대 서비스의 DDoS 방어 레이어는 다음과 같이 구성된다.

```
인터넷
  │
  ▼
CloudFlare / AWS Shield
  │  - L3/L4 공격 흡수 (Volumetric, Protocol)
  │  - IP 평판 기반 차단
  │  - 글로벌 Anycast 네트워크로 트래픽 분산
  ▼
CDN (CloudFront)
  │  - 정적 콘텐츠 캐싱으로 오리진 부하 감소
  │  - 지리적 차단
  ▼
WAF (AWS WAF / Cloudflare WAF)
  │  - OWASP 규칙셋 적용
  │  - 커스텀 규칙 (SQL injection, XSS 등)
  │  - IP/User-Agent 기반 차단
  ▼
API Gateway
  │  - API 키 인증
  │  - 스로틀링 (메서드별 Rate Limit)
  │  - 요청/응답 변환
  ▼
Load Balancer (ALB)
  │  - Health Check 기반 라우팅
  │  - SSL Termination
  ▼
Application Server
  │  - 사용자/IP별 Rate Limiting (Redis)
  │  - 비즈니스 로직 레벨 검증
  ▼
Redis (분산 Rate Limit 상태 관리)
```

각 레이어는 독립적으로 방어 역할을 가진다. 한 레이어가 뚫려도 다음 레이어가 방어한다. 이를 "심층 방어(Defense in Depth)"라 한다.

---

## 참고 자료

- [Cloudflare Learning: What is rate limiting?](https://www.cloudflare.com/learning/bots/what-is-rate-limiting/) — Rate Limiting 개념과 알고리즘 설명
- [AWS Shield Developer Guide](https://docs.aws.amazon.com/waf/latest/developerguide/shield-chapter.html) — Shield Standard/Advanced 구성 가이드
- [Redis Documentation: Rate Limiting Pattern](https://redis.io/docs/manual/patterns/rate-limiting/) — Redis 기반 Rate Limiting 패턴
- [IETF RFC 6585: Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585) — HTTP 429 Too Many Requests 스펙
- [OWASP: Blocking Brute Force Attacks](https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks) — 브루트포스 방어 가이드
- [Kong Rate Limiting Plugin Documentation](https://docs.konghq.com/hub/kong-inc/rate-limiting/) — Kong Gateway Rate Limiting 상세 설정
