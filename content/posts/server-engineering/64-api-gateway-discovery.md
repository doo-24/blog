---
title: "[분산 시스템과 MSA] 6편 — API Gateway와 서비스 디스커버리"
date: 2026-03-17T17:02:00+09:00
draft: false
tags: ["API Gateway", "BFF", "서비스 디스커버리", "Consul", "서버"]
series: ["분산 시스템과 MSA"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 9
summary: "API Gateway 역할(라우팅, 인증, Rate Limiting, 집계), BFF 패턴, 서비스 디스커버리(Client-side vs Server-side, Consul, Eureka), 쿠버네티스 내장 디스커버리 vs 외부 레지스트리까지"
---

수십 개의 마이크로서비스가 존재하는 시스템에서 클라이언트가 각 서비스의 주소를 직접 알아야 한다면 어떨까. 서비스가 배포될 때마다 IP가 바뀌고, 새 서비스가 추가될 때마다 클라이언트 코드를 수정해야 한다면 MSA는 오히려 모놀리스보다 더 복잡한 시스템이 되어버린다. API Gateway와 서비스 디스커버리는 이 문제를 정면으로 해결하는 두 가지 핵심 인프라 패턴이다.

---

## API Gateway란 무엇인가

API Gateway는 마이크로서비스 아키텍처에서 단일 진입점(Single Entry Point) 역할을 한다.

클라이언트는 API Gateway 하나와만 통신하고, Gateway가 내부 서비스들로 요청을 라우팅한다.

이 단순한 구조가 가져오는 이점은 생각보다 크다.

서비스 내부 구조의 변경이 클라이언트에게 완전히 투명해지고, 횡단 관심사(Cross-cutting Concerns)를 한 곳에서 처리할 수 있게 된다.

### API Gateway의 핵심 역할

**1. 라우팅 (Routing)**

요청 경로, 헤더, 메서드에 따라 적절한 서비스로 요청을 전달한다.

단순한 path prefix 매칭부터 헤더 기반 A/B 테스트, 가중치 기반 카나리 배포까지 다양한 라우팅 전략을 지원한다.

**2. 인증/인가 (Authentication & Authorization)**

각 서비스가 개별적으로 인증 로직을 구현하는 대신, Gateway에서 JWT 검증이나 OAuth 토큰 확인을 일괄 처리한다.

인증이 통과된 요청에만 내부 서비스 호출이 허용되어 서비스 자체는 비즈니스 로직에만 집중할 수 있다.

**3. Rate Limiting**

특정 클라이언트나 IP가 서비스를 과도하게 호출하는 것을 막는다.

Redis 기반 슬라이딩 윈도우나 토큰 버킷 알고리즘으로 구현하는 것이 일반적이다.

**4. 집계 (Aggregation)**

모바일 클라이언트가 페이지 하나를 그리기 위해 사용자 서비스, 주문 서비스, 상품 서비스를 각각 호출해야 한다면 레이턴시가 쌓인다.

Gateway에서 여러 서비스 호출을 병렬로 처리하고 결과를 합쳐서 클라이언트에 단일 응답으로 내려줄 수 있다.

---

## Spring Cloud Gateway 실전 구성

Spring Cloud Gateway는 Reactive 기반으로 동작하는 고성능 API Gateway다.

Zuul 1.x의 블로킹 아키텍처 한계를 넘어 WebFlux와 Project Reactor를 사용한다.

### 기본 라우팅 설정

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service          # 로드밸런서를 통한 서비스 디스커버리
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1               # /api 접두사 제거
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                key-resolver: "#{@userKeyResolver}"

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Header=X-Request-Source, mobile  # 모바일 요청만 라우팅
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, api-gateway

        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: product-cb
                fallbackUri: forward:/fallback/products
```

### JWT 인증 필터 구현

```java
@Component
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {

    private final JwtTokenProvider jwtTokenProvider;

    // 인증이 필요 없는 경로 목록
    private static final List<String> WHITELIST = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/api/products"  // 상품 목록은 비인증 접근 허용
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();

        // 화이트리스트 경로는 바로 통과
        if (WHITELIST.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest()
            .getHeaders()
            .getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String token = authHeader.substring(7);

        try {
            Claims claims = jwtTokenProvider.validateToken(token);

            // 검증된 사용자 정보를 헤더에 추가해 내부 서비스로 전달
            ServerHttpRequest mutatedRequest = exchange.getRequest()
                .mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Role", claims.get("role", String.class))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (JwtException e) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
    }

    @Override
    public int getOrder() {
        return -1;  // 가장 먼저 실행
    }
}
```

### Rate Limiting Key Resolver

```java
@Bean
public KeyResolver userKeyResolver() {
    return exchange -> {
        // 인증된 사용자는 userId 기준, 미인증은 IP 기준으로 제한
        String userId = exchange.getRequest()
            .getHeaders()
            .getFirst("X-User-Id");

        if (userId != null) {
            return Mono.just("user:" + userId);
        }

        return Mono.just("ip:" + Objects.requireNonNull(
            exchange.getRequest().getRemoteAddress()
        ).getAddress().getHostAddress());
    };
}
```

### 집계 엔드포인트 구현

```java
@RestController
@RequestMapping("/api/aggregate")
public class AggregationController {

    private final WebClient userClient;
    private final WebClient orderClient;
    private final WebClient productClient;

    // 대시보드 데이터를 한 번의 요청으로 제공
    @GetMapping("/dashboard/{userId}")
    public Mono<DashboardResponse> getDashboard(@PathVariable String userId) {
        Mono<UserProfile> userMono = userClient.get()
            .uri("/users/{id}", userId)
            .retrieve()
            .bodyToMono(UserProfile.class)
            .onErrorReturn(UserProfile.empty());

        Mono<List<Order>> ordersMono = orderClient.get()
            .uri("/orders/recent?userId={id}&limit=5", userId)
            .retrieve()
            .bodyToFlux(Order.class)
            .collectList()
            .onErrorReturn(List.of());

        // zip으로 두 결과를 병렬 처리 후 합침
        return Mono.zip(userMono, ordersMono)
            .map(tuple -> DashboardResponse.builder()
                .user(tuple.getT1())
                .recentOrders(tuple.getT2())
                .build());
    }
}
```

---

## BFF (Backend for Frontend) 패턴

하나의 API Gateway가 웹, 모바일, TV 앱을 모두 서빙하려 하면 문제가 생긴다.

모바일은 작은 화면에 맞는 최소한의 데이터를, 웹은 풍부한 상세 데이터를 원하는데 단일 API로 모든 요구사항을 만족시키는 것은 불가능에 가깝다. 모바일에 최적화하면 웹이 여러 번 호출해야 하고, 웹에 맞추면 모바일이 불필요한 데이터를 과다하게 받아 배터리와 데이터를 낭비한다. 이 모순을 해결하기 위해 등장한 것이 BFF 패턴이다.

### BFF 구조

BFF 패턴은 클라이언트 유형별로 전용 Gateway를 두는 방식이다.

```
[Web Browser]   --> [Web BFF]    --+
[iOS App]       --> [Mobile BFF] --+--> [내부 마이크로서비스들]
[Android App]   --> [Mobile BFF] --+
[Admin Panel]   --> [Admin BFF]  --+
```

각 BFF는 해당 클라이언트의 요구사항에 최적화된 API를 제공한다.

모바일 BFF는 응답을 압축하고 필드를 최소화하며, 웹 BFF는 더 풍부한 데이터와 실시간 기능을 지원한다.

### BFF 구현 예시

```java
// Mobile BFF: 필드 최소화 + 압축 응답
@RestController
@RequestMapping("/mobile/v1")
public class MobileBffController {

    @GetMapping("/products/{id}")
    public Mono<MobileProductResponse> getProduct(@PathVariable Long id) {
        return productService.getProduct(id)
            .map(product -> MobileProductResponse.builder()
                .id(product.getId())
                .name(product.getName())
                .price(product.getPrice())
                .thumbnailUrl(product.getThumbnailUrl())  // 작은 이미지만
                // 상세 설명, 리뷰 등은 제외
                .build());
    }
}

// Web BFF: 풍부한 데이터 + 실시간 재고 정보 포함
@RestController
@RequestMapping("/web/v1")
public class WebBffController {

    @GetMapping("/products/{id}")
    public Mono<WebProductResponse> getProduct(@PathVariable Long id) {
        Mono<Product> productMono = productService.getProduct(id);
        Mono<StockInfo> stockMono = inventoryService.getStock(id);
        Mono<List<Review>> reviewsMono = reviewService.getTopReviews(id, 10);

        return Mono.zip(productMono, stockMono, reviewsMono)
            .map(tuple -> WebProductResponse.builder()
                .product(tuple.getT1())
                .stock(tuple.getT2())
                .topReviews(tuple.getT3())
                .build());
    }
}
```

### BFF 안티패턴

**안티패턴 1: BFF에 비즈니스 로직 집어넣기**

BFF는 데이터 집계와 포맷 변환에만 집중해야 한다.

할인 계산, 재고 차감 같은 비즈니스 로직이 BFF에 들어가면 동일한 로직이 여러 BFF에 중복되고, 변경 시 모두 수정해야 하는 악몽이 시작된다.

**안티패턴 2: 너무 많은 BFF**

플랫폼별, 팀별로 BFF를 무분별하게 늘리면 운영 부담이 증가한다.

Web/Mobile/Admin 정도의 구분이 적절하며, 그 이상은 신중하게 판단해야 한다.

---

## 서비스 디스커버리

마이크로서비스 환경에서 서비스의 IP와 포트는 고정되지 않는다.

컨테이너 재시작, 오토스케일링, 롤링 배포 등으로 인스턴스가 수시로 교체된다.

서비스 디스커버리는 이런 동적 환경에서 서비스들이 서로를 찾을 수 있게 해주는 메커니즘이다.

### Client-side vs Server-side Discovery

두 가지 근본적으로 다른 접근 방식이 존재한다.

**Client-side Discovery**

클라이언트가 서비스 레지스트리에서 직접 인스턴스 목록을 가져와 로드밸런싱을 직접 수행한다.

```
[Client] --조회--> [Service Registry]
[Client] --선택--> [Service Instance A or B or C]
```

Netflix Ribbon이 대표적인 구현체다.

클라이언트에게 더 많은 제어권을 주지만, 각 클라이언트가 로드밸런싱 로직을 가져야 한다는 부담이 있다.

**Server-side Discovery**

클라이언트는 로드밸런서에게 요청을 보내고, 로드밸런서가 레지스트리를 조회해 적절한 인스턴스로 라우팅한다.

```
[Client] --> [Load Balancer] --조회--> [Service Registry]
                             --라우팅--> [Service Instance]
```

쿠버네티스의 kube-proxy + Service 리소스가 이 방식으로 동작한다.

클라이언트는 단순해지지만, 로드밸런서가 SPOF(Single Point of Failure)가 될 수 있다.

| 구분 | Client-side | Server-side |
|------|-------------|-------------|
| 구현 복잡도 | 클라이언트에 부담 | 클라이언트 단순 |
| 언어 의존성 | 언어별 클라이언트 필요 | 언어 독립적 |
| 지연 시간 | 직접 호출로 낮음 | 중간 홉 추가 |
| 장애 지점 | 분산됨 | 로드밸런서 집중 |

---

## Consul 실전 구성

HashiCorp Consul은 서비스 디스커버리, 헬스체크, KV 스토어, 서비스 메시를 통합 제공하는 도구다.

단순한 레지스트리를 넘어 서비스 세그멘테이션과 mTLS까지 지원한다.

### Consul 서버 설정

```hcl
# consul-server.hcl
datacenter = "dc1"
data_dir   = "/opt/consul"
log_level  = "INFO"

server           = true
bootstrap_expect = 3  # 3개 서버로 Raft 쿼럼 구성

ui_config {
  enabled = true
}

# 클러스터 내부 통신 암호화
encrypt = "생성된-gossip-키"

# TLS 설정
tls {
  defaults {
    ca_file   = "/etc/consul/certs/ca.pem"
    cert_file = "/etc/consul/certs/server.pem"
    key_file  = "/etc/consul/certs/server-key.pem"
    verify_incoming = true
    verify_outgoing = true
  }
}

# 퍼포먼스 튜닝
performance {
  raft_multiplier = 1  # 프로덕션에서는 1 (기본값 5보다 빠른 리더 선출)
}
```

### 서비스 등록 설정

```hcl
# service-registration.hcl
service {
  name = "order-service"
  id   = "order-service-1"
  port = 8080

  tags = ["v2", "production"]

  # 헬스체크 설정
  check {
    id       = "order-service-health"
    name     = "Order Service HTTP Health Check"
    http     = "http://localhost:8080/actuator/health"
    interval = "10s"
    timeout  = "3s"

    # 연속 실패 2회 시 critical 상태로 전환
    deregister_critical_service_after = "30s"
  }

  # 메타데이터
  meta {
    version     = "2.1.0"
    environment = "production"
    region      = "ap-northeast-2"
  }
}
```

### Spring Boot와 Consul 연동

```yaml
# application.yml
spring:
  cloud:
    consul:
      host: consul.internal
      port: 8500
      discovery:
        enabled: true
        register: true
        service-name: order-service
        health-check-path: /actuator/health
        health-check-interval: 10s
        instance-id: ${spring.application.name}:${random.value}
        # 태그로 서비스 메타데이터 전달
        tags:
          - version=2.1.0
          - environment=production
      config:
        enabled: true
        format: YAML
        # Consul KV에서 설정 로드
        prefix: config
        default-context: application
```

```java
// Consul을 통한 서비스 조회
@Service
public class OrderServiceClient {

    private final DiscoveryClient discoveryClient;
    private final WebClient.Builder webClientBuilder;

    public Mono<Order> getOrder(Long orderId) {
        // Consul에서 서비스 인스턴스 목록 조회
        List<ServiceInstance> instances = discoveryClient
            .getInstances("order-service");

        if (instances.isEmpty()) {
            return Mono.error(new ServiceUnavailableException("order-service"));
        }

        // 랜덤 선택 (실제로는 Ribbon/LoadBalancer가 처리)
        ServiceInstance instance = instances.get(
            ThreadLocalRandom.current().nextInt(instances.size())
        );

        String url = instance.getUri() + "/orders/" + orderId;

        return webClientBuilder.build()
            .get()
            .uri(url)
            .retrieve()
            .bodyToMono(Order.class);
    }
}
```

### Consul DNS를 통한 서비스 조회

Consul은 DNS 인터페이스도 제공하므로 코드 변경 없이 서비스를 찾을 수 있다.

```bash
# Consul DNS로 서비스 조회 (기본 포트 8600)
dig @127.0.0.1 -p 8600 order-service.service.consul

# SRV 레코드로 IP + 포트 함께 조회
dig @127.0.0.1 -p 8600 order-service.service.consul SRV

# 특정 태그가 달린 인스턴스만 조회
dig @127.0.0.1 -p 8600 v2.order-service.service.consul
```

---

## Netflix Eureka와의 비교

Netflix Eureka는 Spring 생태계와 깊이 통합된 서비스 레지스트리다.

Consul과 달리 AP(가용성 우선) 방식으로 설계되어 네트워크 파티션 상황에서도 레지스트리 서비스가 계속 응답한다.

### Eureka 특성

```yaml
# Eureka 서버 설정
eureka:
  server:
    enable-self-preservation: true   # 자기보호 모드 (프로덕션에서 필수)
    eviction-interval-timer-in-ms: 5000
  instance:
    hostname: eureka-server
  client:
    register-with-eureka: false  # 서버 자신은 등록하지 않음
    fetch-registry: false
```

```yaml
# Eureka 클라이언트 설정
eureka:
  client:
    service-url:
      defaultZone: http://eureka1:8761/eureka/,http://eureka2:8762/eureka/
    register-with-eureka: true
    fetch-registry: true
    registry-fetch-interval-seconds: 30  # 캐시 갱신 주기
  instance:
    lease-renewal-interval-in-seconds: 30   # 하트비트 간격
    lease-expiration-duration-in-seconds: 90  # 만료 시간
    prefer-ip-address: true
```

### Consul vs Eureka 선택 기준

| 기준 | Consul | Eureka |
|------|--------|--------|
| 일관성 모델 | CP (Raft) | AP (자기보호) |
| Spring 통합 | 좋음 | 최고 |
| 다중 데이터센터 | 기본 지원 | 별도 구성 필요 |
| KV 스토어 | 내장 | 없음 |
| 서비스 메시 | 지원 (Envoy) | 미지원 |
| 운영 복잡도 | 높음 | 낮음 |

Spring 환경만 쓴다면 Eureka가 쉽다.

다중 언어 환경이거나 서비스 메시가 필요하다면 Consul을 선택한다.

---

## 쿠버네티스 내장 서비스 디스커버리

쿠버네티스는 자체적인 서비스 디스커버리 메커니즘을 내장하고 있다.

별도의 Consul이나 Eureka 없이도 쿠버네티스 Service 리소스와 CoreDNS만으로 서비스 디스커버리가 동작한다.

### 쿠버네티스 Service 리소스

```yaml
# ClusterIP Service: 클러스터 내부에서만 접근
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
spec:
  selector:
    app: order-service  # 이 레이블을 가진 Pod에게 라우팅
  ports:
    - name: http
      port: 80
      targetPort: 8080
  type: ClusterIP

---
# 배포된 Pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:2.1.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

쿠버네티스에서 `order-service`를 호출하려면 DNS 이름 `order-service.ecommerce.svc.cluster.local`을 사용한다.

같은 네임스페이스라면 `order-service`만으로도 해석된다.

### Headless Service와 StatefulSet

데이터베이스처럼 각 Pod에 직접 접근해야 할 때는 Headless Service를 사용한다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None  # Headless: VIP 없이 Pod IP 직접 반환
  selector:
    app: postgres
  ports:
    - port: 5432
```

`postgres-headless`를 DNS 조회하면 VIP 대신 각 Pod의 실제 IP 목록이 반환된다.

StatefulSet과 함께 쓰면 `postgres-0.postgres-headless`, `postgres-1.postgres-headless` 처럼 예측 가능한 이름으로 개별 Pod에 접근할 수 있다.

### Ingress와 API Gateway

쿠버네티스 Ingress는 외부 트래픽을 내부 Service로 라우팅하는 L7 라우터다.

하지만 Ingress는 기능이 제한적이고 구현체(Nginx, Traefik 등)마다 어노테이션이 달라 이식성이 낮다.

```yaml
# Nginx Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/users(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 80
          - path: /api/orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
```

### 쿠버네티스 내장 vs 외부 레지스트리

프로젝트를 쿠버네티스 위에서만 운영한다면 내장 디스커버리로 충분한 경우가 많다.

그러나 아래 상황에서는 외부 레지스트리가 필요하다.

**외부 레지스트리가 필요한 경우**

1. 멀티 클러스터 환경: 여러 K8s 클러스터의 서비스가 서로를 찾아야 할 때 K8s 내장 DNS만으로는 불가능하다.

2. 하이브리드 클라우드: K8s 밖에 있는 VM이나 베어메탈 서버도 레지스트리에 참여해야 할 때.

3. 세밀한 헬스체크: K8s의 readinessProbe보다 더 복잡한 헬스체크 로직이 필요할 때.

4. 서비스 세그멘테이션: 어떤 서비스가 어떤 서비스를 호출할 수 있는지 정책을 중앙에서 관리해야 할 때.

```yaml
# Consul + 쿠버네티스 통합: consul-k8s 사용
# Consul이 K8s 서비스를 자동으로 동기화
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceDefaults
metadata:
  name: order-service
spec:
  protocol: http
  # Consul에서 서킷브레이커 정책 정의
  upstreamConfig:
    defaults:
      limitsMaxConnections: 512
      limitsMaxPendingRequests: 512
      limitsMaxConcurrentRequests: 512

---
# 서비스 인텐션: product-service만 order-service 호출 허용
apiVersion: consul.hashicorp.com/v1alpha1
kind: ServiceIntentions
metadata:
  name: order-service
spec:
  destination:
    name: order-service
  sources:
    - name: api-gateway
      action: allow
    - name: product-service
      action: allow
    - name: "*"
      action: deny
```

---

## 운영 시 주의사항과 안티패턴

### Gateway 안티패턴

**안티패턴: Gateway에 도메인 로직 집중**

API Gateway가 "스마트" 해질수록 병목이 된다.

라우팅, 인증, Rate Limiting은 Gateway의 역할이지만 재고 계산, 가격 책정, 할인 로직은 절대 Gateway에 두지 않는다.

**안티패턴: 동기 집계 과도 사용**

10개 서비스를 순차적으로 호출하는 집계 API는 최악의 레이턴시가 10개 서비스 레이턴시의 합이 된다.

반드시 병렬 처리하고, 실패 허용 응답(Partial Response)을 설계해야 한다.

**안티패턴: Gateway 없는 서비스 간 직접 호출**

내부 서비스가 외부 클라이언트처럼 Gateway를 거쳐 다른 서비스를 호출하면 불필요한 홉이 추가된다.

서비스 간 통신은 서비스 메시(Istio, Linkerd) 또는 직접 서비스 디스커버리를 사용한다.

### 서비스 디스커버리 안티패턴

**안티패턴: 레지스트리 캐시 없이 매 요청마다 조회**

서비스 레지스트리는 높은 가용성을 목표로 하지만 그렇다고 매 요청마다 조회하면 레지스트리가 병목이 된다.

클라이언트 측에서 일정 시간(30초~1분) 캐싱하고, 백그라운드 갱신을 수행한다.

**안티패턴: 헬스체크 없는 서비스 등록**

헬스체크 없이 등록된 서비스는 죽어도 레지스트리에서 제거되지 않는다.

클라이언트는 죽은 인스턴스에 계속 요청을 보내고 오류가 발생한다.

모든 서비스 등록에는 반드시 헬스체크를 함께 구성한다.

---

## 정리

API Gateway는 단순한 리버스 프록시가 아니라 횡단 관심사의 집약점이다.

인증, Rate Limiting, 집계를 Gateway에서 처리함으로써 개별 서비스는 비즈니스 로직에만 집중할 수 있다.

BFF 패턴은 다양한 클라이언트의 요구사항을 각각 최적화된 방식으로 만족시키는 우아한 해법이다.

서비스 디스커버리는 동적 환경에서 서비스들이 서로를 찾는 인프라 기반이다.

쿠버네티스만 사용한다면 내장 디스커버리로 시작하고, 멀티 클러스터나 하이브리드 환경이 되면 Consul 같은 외부 레지스트리를 도입하는 것이 현실적인 진화 경로다.

---

## 참고 자료

- [Spring Cloud Gateway 공식 문서](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/)
- [HashiCorp Consul 아키텍처 가이드](https://developer.hashicorp.com/consul/docs/architecture)
- [Netflix Tech Blog: Eureka! Why You Shouldn't Use ZooKeeper for Service Discovery](https://netflixtechblog.com/eureka-the-netflix-service-registry-790a4f8f09aa)
- [Sam Newman, "Building Microservices" 2nd Ed. — Chapters 9-10](https://samnewman.io/books/building_microservices_2nd_edition/)
- [쿠버네티스 Service 공식 문서](https://kubernetes.io/docs/concepts/services-networking/service/)
- [BFF Pattern — Phil Calçado](https://philcalcado.com/2015/09/18/the_back_end_for_front_end_pattern_bff.html)
