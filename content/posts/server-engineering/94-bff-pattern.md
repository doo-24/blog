---
title: "[Frontend 통합] 2편 — BFF 패턴 실전: 프론트엔드를 위한 백엔드 계층"
date: 2026-03-17T12:06:00+09:00
draft: false
tags: ["BFF", "GraphQL", "API Gateway", "오버페칭", "서버"]
series: ["Frontend 통합"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 14
summary: "BFF가 필요한 상황(모바일/웹/서드파티 API 분화), BFF 구현 방식(독립 서비스, API Gateway 확장, GraphQL as BFF), 데이터 집계와 오버페칭/언더페칭 해결, BFF 테스트 전략까지"
---

## 왜 BFF가 필요한가

초기 서비스는 단일 API 서버로 출발하는 경우가 많다.

웹 프론트엔드, 모바일 앱, 외부 파트너 시스템이 모두 같은 엔드포인트를 바라본다.

처음에는 그게 단순해서 좋다. 그런데 서비스가 성장하면 균열이 생긴다.

모바일 앱은 네트워크가 불안정하고 배터리 제약이 있어서 최소한의 데이터만 원한다. 웹 앱은 풍부한 화면을 위해 여러 리소스를 한 번에 받고 싶어한다. 서드파티 파트너는 자신들 도메인에 맞는 인터페이스가 필요하다.

결국 하나의 API가 이 세 가지 요구를 동시에 만족시키려다 복잡해진다.

응답 필드가 늘어나고, 파라미터 조합이 폭발하고, "이 필드는 모바일용이고 저 필드는 웹용"이라는 주석이 코드에 쌓인다.

**BFF(Backend For Frontend)** 는 이 문제를 해결하는 아키텍처 패턴이다.

클라이언트 종류마다 전용 백엔드 계층을 두는 것이다. 각 BFF는 해당 클라이언트가 필요한 형태로 데이터를 가공해서 제공한다.

---

## BFF가 필요한 상황 — 분화의 징조

### 클라이언트별 응답 형태가 달라지기 시작할 때

모바일 앱이 `/products/{id}` 를 호출하는데, 반환되는 JSON에 필드가 40개라면 문제다.

모바일이 실제로 사용하는 필드는 8개인데 나머지 32개가 네트워크를 낭비한다.

웹은 반대로 40개 필드로도 부족해서 상품 리뷰, 추천 상품, 재고 현황을 별도로 3번 더 호출한다.

### 한 화면에 여러 서비스 데이터가 필요할 때

대시보드 화면을 예로 들면, 사용자 정보, 최근 주문, 알림 목록, 포인트 잔액이 한 화면에 들어간다.

각각 다른 마이크로서비스에서 온다면 프론트엔드가 4번 API를 호출해야 한다.

병렬 호출을 해도 각각의 응답을 기다리고 조합하는 로직이 프론트엔드에 생긴다.

### 클라이언트가 직접 마이크로서비스를 알면 안 될 때

프론트엔드가 `user-service.internal`, `order-service.internal`, `notification-service.internal` 를 직접 호출하면 내부 구조가 노출된다.

서비스를 분리하거나 합칠 때마다 프론트엔드 코드를 바꿔야 한다.

BFF가 그 사이에서 추상화 계층 역할을 한다.

---

## BFF 구현 방식 세 가지

### 방식 1 — 독립 서비스 BFF

가장 전통적인 방식이다. 클라이언트별로 BFF 서비스를 따로 배포한다.

```
[모바일 앱] → [Mobile BFF] → [User Service]
                           → [Order Service]
                           → [Product Service]

[웹 앱]     → [Web BFF]    → [User Service]
                           → [Order Service]
                           → [Product Service]
                           → [Recommendation Service]
```

Node.js로 Mobile BFF를 만드는 예시를 보자.

```typescript
// mobile-bff/src/routes/home.ts
import express from 'express';
import { UserService } from '../clients/user-service';
import { OrderService } from '../clients/order-service';

const router = express.Router();

router.get('/home', async (req, res) => {
  const userId = req.user.id;

  // 병렬로 여러 서비스 호출
  const [user, recentOrders] = await Promise.all([
    UserService.getProfile(userId),
    OrderService.getRecent(userId, { limit: 3 }),
  ]);

  // 모바일에 필요한 필드만 추려서 반환
  res.json({
    name: user.name,
    avatar: user.avatarUrl,
    orders: recentOrders.map(order => ({
      id: order.id,
      status: order.status,
      totalPrice: order.totalPrice,
    })),
  });
});

export default router;
```

웹 BFF는 같은 서비스를 호출하지만 다른 필드를 내려준다.

```typescript
// web-bff/src/routes/home.ts
router.get('/home', async (req, res) => {
  const userId = req.user.id;

  const [user, recentOrders, notifications, recommendations] = await Promise.all([
    UserService.getProfile(userId),
    OrderService.getRecent(userId, { limit: 10 }),
    NotificationService.getUnread(userId),
    RecommendationService.getForUser(userId),
  ]);

  // 웹은 더 많은 데이터가 필요
  res.json({
    user: {
      id: user.id,
      name: user.name,
      email: user.email,
      avatar: user.avatarUrl,
      membershipLevel: user.membershipLevel,
      points: user.points,
    },
    orders: recentOrders,
    unreadNotifications: notifications.count,
    recommendations: recommendations.items,
  });
});
```

**장점:** 각 BFF 팀이 자율적으로 배포하고 변경할 수 있다.

**단점:** BFF가 늘어날수록 공통 로직(인증, 로깅, 에러 처리)이 중복된다.

### 방식 2 — API Gateway 확장

API Gateway에 변환 로직을 얹는 방식이다.

AWS API Gateway, Kong, NGINX를 쓰거나, Spring Cloud Gateway에 커스텀 필터를 추가한다.

Spring Cloud Gateway 예시를 보자.

```java
// GatewayConfig.java
@Configuration
public class GatewayConfig {

    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            // 모바일 트래픽 라우팅
            .route("mobile-product", r -> r
                .path("/mobile/products/**")
                .filters(f -> f
                    .rewritePath("/mobile/products/(?<id>.*)", "/internal/products/${id}")
                    .filter(new MobileResponseTransformFilter())
                )
                .uri("lb://product-service"))
            // 웹 트래픽 라우팅
            .route("web-product", r -> r
                .path("/web/products/**")
                .filters(f -> f
                    .rewritePath("/web/products/(?<id>.*)", "/internal/products/${id}")
                    .filter(new WebResponseTransformFilter())
                )
                .uri("lb://product-service"))
            .build();
    }
}
```

```java
// MobileResponseTransformFilter.java
@Component
public class MobileResponseTransformFilter implements GatewayFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        return chain.filter(exchange.mutate()
            .response(new MobileResponseDecorator(exchange.getResponse()))
            .build());
    }
}
```

**장점:** 인프라 수준에서 처리하므로 별도 서비스를 운영할 필요가 없다.

**단점:** 복잡한 비즈니스 로직을 Gateway 필터에 넣으면 유지보수가 어려워진다.

### 방식 3 — GraphQL as BFF

GraphQL 자체가 BFF 역할을 한다. 가장 유연한 방식이다.

클라이언트가 필요한 필드를 쿼리로 직접 지정하므로, BFF가 클라이언트별로 분리되지 않아도 된다.

```typescript
// schema.ts
const typeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
    avatar: String
    membershipLevel: String
    points: Int
  }

  type Order {
    id: ID!
    status: String!
    totalPrice: Float!
    createdAt: String!
    items: [OrderItem!]!
  }

  type OrderItem {
    productName: String!
    quantity: Int!
    price: Float!
  }

  type HomeData {
    user: User!
    orders: [Order!]!
    unreadNotifications: Int!
  }

  type Query {
    home: HomeData!
  }
`;
```

```typescript
// resolvers.ts
const resolvers = {
  Query: {
    home: async (_, __, { userId, dataSources }) => {
      const [user, orders, notifications] = await Promise.all([
        dataSources.userAPI.getProfile(userId),
        dataSources.orderAPI.getRecent(userId),
        dataSources.notificationAPI.getUnreadCount(userId),
      ]);

      return { user, orders, unreadNotifications: notifications };
    },
  },
};
```

모바일은 꼭 필요한 필드만 쿼리한다.

```graphql
# 모바일 쿼리
query MobileHome {
  home {
    user {
      name
      avatar
    }
    orders {
      id
      status
      totalPrice
    }
    unreadNotifications
  }
}
```

웹은 더 많은 필드를 요청한다.

```graphql
# 웹 쿼리
query WebHome {
  home {
    user {
      id
      name
      email
      avatar
      membershipLevel
      points
    }
    orders {
      id
      status
      totalPrice
      createdAt
      items {
        productName
        quantity
        price
      }
    }
    unreadNotifications
  }
}
```

**장점:** 단일 엔드포인트로 다양한 클라이언트를 유연하게 지원한다. 오버페칭과 언더페칭 문제를 구조적으로 해결한다.

**단점:** GraphQL 인프라 운영 비용이 있다. N+1 쿼리 문제를 DataLoader로 직접 해결해야 한다.

---

## 데이터 집계와 변환 — 오버페칭/언더페칭 해결

### 오버페칭(Overfetching) 이란

필요한 데이터보다 더 많은 데이터를 받는 상황이다.

`/users/{id}` 가 50개 필드를 반환하는데 화면에서 이름과 프로필 이미지만 필요한 경우다.

불필요한 JSON을 직렬화하고 전송하고 파싱하는 비용이 발생한다.

BFF에서 응답을 필터링해 필요한 필드만 내려주면 해결된다.

```typescript
// 백엔드 서비스는 전체 유저 객체를 반환
const fullUser = await userService.getById(userId);
// {id, name, email, phone, address, birthDate, createdAt, updatedAt,
//  lastLoginAt, role, permissions, preferences, metadata, ...}

// BFF는 모바일에 필요한 것만 추린다
const mobileUser = {
  name: fullUser.name,
  avatar: fullUser.profileImageUrl,
};
```

### 언더페칭(Underfetching) 이란

한 번의 호출로 필요한 데이터를 다 받지 못해서 추가 호출이 필요한 상황이다.

주문 목록을 받은 뒤, 각 주문의 상품 상세를 개별적으로 N번 더 호출하는 패턴이 대표적이다.

BFF가 필요한 데이터를 사전에 집계해서 한 번의 응답에 담아준다.

```typescript
// 언더페칭 문제: 프론트가 직접 해결
const orders = await fetch('/orders');
const ordersWithDetails = await Promise.all(
  orders.map(order => fetch(`/orders/${order.id}/items`))
);
// 프론트엔드가 집계 로직을 직접 가짐 — 좋지 않다

// BFF에서 해결
router.get('/orders-summary', async (req, res) => {
  const orders = await orderService.getByUser(req.user.id);

  const ordersWithItems = await Promise.all(
    orders.map(async order => {
      const items = await orderService.getItems(order.id);
      return { ...order, items };
    })
  );

  res.json(ordersWithItems);
});
```

### 데이터 변환 — 도메인 모델 차이 극복

백엔드 도메인 모델과 프론트엔드가 원하는 구조가 다를 때 BFF가 변환을 담당한다.

```typescript
// 백엔드 응답
const backendOrder = {
  order_id: "ORD-20240315-001",
  order_status_code: "03",
  total_amount_krw: 59000,
  ordered_at_utc: "2024-03-15T04:30:00Z",
};

// BFF에서 프론트엔드 친화적으로 변환
const frontendOrder = {
  id: backendOrder.order_id,
  status: resolveStatusLabel(backendOrder.order_status_code), // "배송중"
  totalPrice: backendOrder.total_amount_krw,
  orderedAt: toKSTString(backendOrder.ordered_at_utc), // "2024-03-15 13:30"
};

function resolveStatusLabel(code: string): string {
  const labels: Record<string, string> = {
    "01": "결제 완료",
    "02": "상품 준비중",
    "03": "배송중",
    "04": "배송 완료",
    "05": "취소됨",
  };
  return labels[code] ?? "알 수 없음";
}
```

날짜 변환, 상태 코드 라벨 변환, 통화 단위 처리 등이 BFF에 집중된다.

프론트엔드 코드에서 이 변환 로직이 사라진다.

---

## BFF 테스트 전략

### 단위 테스트 — 변환 로직 검증

BFF의 핵심 가치는 데이터 변환과 집계다. 이 로직을 단위 테스트로 먼저 검증한다.

```typescript
// order-transformer.test.ts
import { transformOrder } from '../transformers/order';

describe('transformOrder', () => {
  it('상태 코드를 한국어 라벨로 변환한다', () => {
    const raw = {
      order_id: 'ORD-001',
      order_status_code: '03',
      total_amount_krw: 59000,
      ordered_at_utc: '2024-03-15T04:30:00Z',
    };

    const result = transformOrder(raw);

    expect(result.status).toBe('배송중');
  });

  it('UTC 시각을 KST로 변환한다', () => {
    const raw = {
      order_id: 'ORD-001',
      order_status_code: '01',
      total_amount_krw: 10000,
      ordered_at_utc: '2024-03-15T00:00:00Z',
    };

    const result = transformOrder(raw);

    expect(result.orderedAt).toBe('2024-03-15 09:00');
  });

  it('알 수 없는 상태 코드는 기본값을 반환한다', () => {
    const raw = {
      order_id: 'ORD-001',
      order_status_code: '99',
      total_amount_krw: 10000,
      ordered_at_utc: '2024-03-15T00:00:00Z',
    };

    const result = transformOrder(raw);

    expect(result.status).toBe('알 수 없음');
  });
});
```

### 통합 테스트 — 서비스 클라이언트 Mock

BFF가 여러 서비스를 호출해서 집계하는 동작을 테스트한다. 실제 서비스 대신 Mock을 쓴다.

```typescript
// home-route.test.ts
import request from 'supertest';
import app from '../app';
import { UserService } from '../clients/user-service';
import { OrderService } from '../clients/order-service';

jest.mock('../clients/user-service');
jest.mock('../clients/order-service');

describe('GET /home', () => {
  it('유저와 주문 정보를 집계해서 반환한다', async () => {
    (UserService.getProfile as jest.Mock).mockResolvedValue({
      id: 'user-1',
      name: '홍길동',
      avatarUrl: 'https://cdn.example.com/avatar.jpg',
    });

    (OrderService.getRecent as jest.Mock).mockResolvedValue([
      { id: 'ORD-001', status: '배송중', totalPrice: 59000 },
    ]);

    const res = await request(app)
      .get('/home')
      .set('Authorization', 'Bearer test-token');

    expect(res.status).toBe(200);
    expect(res.body.name).toBe('홍길동');
    expect(res.body.orders).toHaveLength(1);
    expect(res.body.orders[0].id).toBe('ORD-001');
  });

  it('UserService 실패 시 503을 반환한다', async () => {
    (UserService.getProfile as jest.Mock).mockRejectedValue(
      new Error('Connection refused')
    );

    const res = await request(app)
      .get('/home')
      .set('Authorization', 'Bearer test-token');

    expect(res.status).toBe(503);
  });
});
```

### Contract Test — 하위 서비스와의 계약 검증

BFF가 의존하는 서비스의 응답 형태가 바뀌면 BFF가 조용히 깨진다.

Pact 같은 Consumer Driven Contract Test 도구로 계약을 명시적으로 관리한다.

```typescript
// pact/user-service.pact.ts
const provider = new PactV3({
  consumer: 'mobile-bff',
  provider: 'user-service',
});

describe('UserService Pact', () => {
  it('getProfile 응답에 name과 avatarUrl이 포함된다', async () => {
    provider
      .addInteraction()
      .given('user user-1 exists')
      .uponReceiving('a request for user profile')
      .withRequest({
        method: 'GET',
        path: '/internal/users/user-1',
      })
      .willRespondWith({
        status: 200,
        body: {
          id: 'user-1',
          name: like('홍길동'),
          avatarUrl: like('https://cdn.example.com/avatar.jpg'),
        },
      });

    await provider.executeTest(async mockProvider => {
      const client = new UserServiceClient(mockProvider.url);
      const user = await client.getProfile('user-1');

      expect(user.name).toBeDefined();
      expect(user.avatarUrl).toBeDefined();
    });
  });
});
```

이 계약 파일을 User Service 팀에게 공유하면, User Service가 변경될 때 계약 위반을 사전에 감지할 수 있다.

---

## BFF 장애가 프론트엔드에 미치는 영향

### BFF는 단일 장애점이 될 수 있다

BFF가 죽으면 해당 클라이언트의 모든 기능이 멈춘다.

모바일 BFF가 다운되면 모바일 앱 전체가 동작하지 않는다.

이 리스크를 줄이기 위해 BFF 자체를 수평 확장하고, 헬스 체크를 통해 로드밸런서가 정상 인스턴스로만 트래픽을 보내게 한다.

### 의존 서비스 장애를 BFF가 격리해야 한다

Recommendation Service 가 느려졌다고 해서 홈 화면 전체가 느려지면 안 된다.

BFF에서 타임아웃과 Fallback을 설정한다.

```typescript
// 타임아웃과 Fallback 처리
async function getHomeData(userId: string) {
  const [user, orders, recommendations] = await Promise.allSettled([
    withTimeout(UserService.getProfile(userId), 1000),
    withTimeout(OrderService.getRecent(userId), 1000),
    withTimeout(RecommendationService.getForUser(userId), 500), // 더 짧은 타임아웃
  ]);

  return {
    user: user.status === 'fulfilled' ? user.value : null,
    orders: orders.status === 'fulfilled' ? orders.value : [],
    recommendations: recommendations.status === 'fulfilled'
      ? recommendations.value
      : [], // Recommendation 실패해도 빈 배열로 처리
  };
}

function withTimeout<T>(promise: Promise<T>, ms: number): Promise<T> {
  return Promise.race([
    promise,
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error(`Timeout after ${ms}ms`)), ms)
    ),
  ]);
}
```

`Promise.allSettled` 는 하나가 실패해도 나머지 결과를 유지한다.

중요하지 않은 데이터(추천 상품 등)는 실패해도 빈 배열로 Fallback 처리한다.

### 서킷 브레이커 적용

반복 실패하는 서비스를 빠르게 차단해서 BFF가 계속 대기하지 않도록 한다.

```typescript
import CircuitBreaker from 'opossum';

const recommendationBreaker = new CircuitBreaker(
  RecommendationService.getForUser,
  {
    timeout: 500,
    errorThresholdPercentage: 50, // 실패율 50% 초과 시 오픈
    resetTimeout: 10000,          // 10초 후 재시도
  }
);

recommendationBreaker.fallback(() => []); // 오픈 상태에서 빈 배열 반환

// 이벤트 로깅
recommendationBreaker.on('open', () => {
  logger.warn('RecommendationService circuit breaker opened');
});

recommendationBreaker.on('close', () => {
  logger.info('RecommendationService circuit breaker closed');
});
```

서킷 브레이커가 열리면 Recommendation Service로 요청을 보내지 않는다. 바로 Fallback을 반환해서 BFF 응답이 빠르게 유지된다.

### BFF 캐싱으로 부하 완화

BFF 계층에서 캐싱을 적용하면 하위 서비스 부하를 줄이고 응답 속도를 높일 수 있다.

```typescript
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });

async function getCachedProfile(userId: string) {
  const cacheKey = `profile:${userId}`;
  const cached = await redis.get(cacheKey);

  if (cached) {
    return JSON.parse(cached);
  }

  const profile = await UserService.getProfile(userId);
  await redis.setEx(cacheKey, 60, JSON.stringify(profile)); // 60초 캐시

  return profile;
}
```

사용자 프로필처럼 자주 변하지 않는 데이터는 BFF 레벨에서 캐시하면 효과적이다.

캐시 무효화 전략은 데이터 특성에 따라 TTL 기반 또는 이벤트 기반으로 선택한다.

---

## 언제 BFF를 도입할 것인가

BFF는 복잡성을 도입한다. 관리해야 할 서비스가 늘어난다.

다음 상황 중 하나라도 해당하면 BFF 도입을 고려할 시점이다.

- 프론트엔드팀이 "데이터 조합 로직을 왜 우리가 직접 해야 하나요?"라고 묻기 시작할 때
- 모바일과 웹이 같은 API를 다르게 써서 변경이 두려워질 때
- 한 화면을 위해 5번 이상 API를 호출해야 할 때
- 내부 마이크로서비스 구조가 외부로 노출되는 게 부담스러울 때

반대로 단일 클라이언트만 있는 서비스나 팀 규모가 작아서 BFF를 별도로 운영하기 어렵다면 굳이 도입할 필요 없다.

GraphQL을 쓰고 있다면 이미 BFF 역할의 상당 부분을 GraphQL 레이어가 담당한다. 중복 도입을 피하는 게 좋다.

---

## 정리

BFF는 클라이언트와 백엔드 사이의 번역 계층이다.

클라이언트 종류별로 맞춤 응답을 만들고, 여러 서비스 데이터를 집계하고, 오버페칭과 언더페칭을 서버 쪽에서 해결한다.

독립 서비스로 만들거나 API Gateway를 확장하거나 GraphQL로 구현하는 세 가지 방법이 있다. 팀 구조와 기술 스택에 맞는 방식을 선택한다.

BFF 자체의 장애 전파를 막으려면 타임아웃, Fallback, 서킷 브레이커를 반드시 적용해야 한다.

테스트는 변환 로직 단위 테스트, 집계 동작 통합 테스트, 하위 서비스와의 Contract Test 세 층으로 구성한다.

---

## 참고 자료

- Sam Newman, "Building Microservices" (2nd Edition) — BFF 패턴 원론
- Phil Calçado, "Pattern: Backends For Frontends" (soundcloud.com/backstage)
- Apollo GraphQL, "GraphQL as a BFF" (apollographql.com/docs)
- Netflix Tech Blog, "GraphQL: A data query language" — Netflix의 BFF 전환 경험
- Microsoft Azure Architecture Center, "Backends for Frontends pattern"
- Opossum 라이브러리 (node-circuit-breaker): github.com/nodeshift/opossum
- Pact Contract Testing: docs.pact.io
