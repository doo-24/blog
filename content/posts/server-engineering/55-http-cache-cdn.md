---
title: "[성능 최적화와 안정성] 3편 — HTTP 캐싱과 CDN: 네트워크 레벨의 성능"
date: 2026-03-17T18:04:00+09:00
draft: false
tags: ["CDN", "Cache-Control", "ETag", "CloudFront", "서버"]
series: ["성능 최적화와 안정성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 8
summary: "Cache-Control, ETag, Last-Modified, Vary 헤더, CDN 동작 원리와 엣지 캐시 무효화, 브라우저 캐시 전략과 정적 자산 버저닝, CDN 선택 기준(CloudFront, Fastly, Cloudflare)까지"
---

서버 성능 튜닝의 대부분은 "더 빠르게 처리하는 것"에 집중된다. 하지만 가장 빠른 요청은 애초에 서버에 도달하지 않는 요청이다. HTTP 캐싱과 CDN은 바로 이 원칙을 네트워크 레벨에서 구현하는 기술이다. 올바르게 설정된 캐시 전략 하나가 서버 부하를 90% 이상 줄이고, 사용자 응답 시간을 수백 밀리초에서 수십 밀리초로 단축시킨다.

---

## HTTP 캐싱의 핵심 헤더

캐싱을 제대로 이해하려면 관련 헤더들이 각각 어떤 역할을 하는지 정확히 알아야 한다.

헤더를 잘못 설정하면 캐시가 전혀 동작하지 않거나, 반대로 변경된 콘텐츠가 오래된 캐시로 서비스되는 최악의 상황이 발생한다.

### Cache-Control

`Cache-Control`은 HTTP/1.1에서 도입된 캐시 제어의 핵심 헤더다.

여러 디렉티브를 조합해서 사용하며, 각 디렉티브의 의미를 명확히 이해해야 한다.

```http
# 공개 캐시(CDN, 프록시)와 브라우저 모두에서 1시간 캐싱
Cache-Control: public, max-age=3600

# 브라우저에만 캐싱, 중간 프록시는 캐싱 금지
Cache-Control: private, max-age=600

# 캐싱 완전 금지 (민감한 데이터)
Cache-Control: no-store

# 캐시에 저장하되, 사용 전 항상 서버에 유효성 검증
Cache-Control: no-cache

# CDN에는 1일, 브라우저에는 1시간 (s-maxage가 CDN에 적용)
Cache-Control: public, max-age=3600, s-maxage=86400

# 캐시 만료 후 재검증 중에도 오래된 응답 1분간 허용 (stale-while-revalidate)
Cache-Control: public, max-age=3600, stale-while-revalidate=60
```

`no-cache`와 `no-store`를 혼동하는 경우가 많다.

`no-store`는 캐시를 아예 저장하지 말라는 지시다. `no-cache`는 캐시에 저장은 하되, 사용할 때마다 서버에 유효성을 확인하라는 의미다.

`s-maxage`는 공유 캐시(CDN, 프록시)에만 적용되는 TTL이다. 브라우저 캐시와 CDN 캐시를 독립적으로 제어할 때 유용하다.

`stale-while-revalidate`는 백그라운드 재검증 패턴을 구현한다. 캐시가 만료된 후에도 설정한 시간 동안은 오래된 응답을 즉시 반환하면서, 동시에 백그라운드에서 새로운 콘텐츠를 가져온다. 응답 지연 없이 캐시를 갱신할 수 있는 강력한 패턴이다.

### ETag

`ETag`는 리소스의 특정 버전을 식별하는 고유 식별자다.

서버는 응답에 ETag를 포함시키고, 클라이언트는 이후 요청에서 `If-None-Match` 헤더로 ETag를 전송한다.

```http
# 서버 응답
HTTP/1.1 200 OK
ETag: "abc123def456"
Cache-Control: public, max-age=0, must-revalidate
Content-Type: application/json

# 클라이언트 재검증 요청
GET /api/users HTTP/1.1
If-None-Match: "abc123def456"

# 변경 없음 (서버 응답, 본문 없음)
HTTP/1.1 304 Not Modified
ETag: "abc123def456"

# 변경됨 (서버 응답, 새로운 본문 포함)
HTTP/1.1 200 OK
ETag: "xyz789new000"
Content-Type: application/json
```

ETag에는 강한 검증자(strong validator)와 약한 검증자(weak validator)가 있다.

강한 ETag는 바이트 단위로 동일한 경우에만 일치한다. 약한 ETag는 의미상 동등하면 일치한다고 본다(`W/"abc123"` 형식).

Nginx에서 ETag를 활성화하는 설정은 간단하다.

```nginx
# nginx.conf
server {
    # ETag는 Nginx 기본값으로 활성화됨
    etag on;

    location /api/ {
        proxy_pass http://backend;
        # 프록시 응답에서 ETag 헤더 전달
        proxy_pass_header ETag;
    }

    location /static/ {
        root /var/www;
        # 정적 파일은 ETag + Last-Modified 모두 활성화
        etag on;
    }
}
```

### Last-Modified

`Last-Modified`는 ETag의 시간 기반 대안이다.

리소스가 마지막으로 수정된 시각을 표현하며, 클라이언트는 `If-Modified-Since` 헤더로 조건부 요청을 보낸다.

```http
# 서버 응답
HTTP/1.1 200 OK
Last-Modified: Wed, 18 Mar 2026 10:30:00 GMT
Cache-Control: public, max-age=3600

# 클라이언트 재검증 요청
GET /images/logo.png HTTP/1.1
If-Modified-Since: Wed, 18 Mar 2026 10:30:00 GMT

# 변경 없음
HTTP/1.1 304 Not Modified
```

ETag와 Last-Modified를 동시에 사용하는 것이 권장 사항이다.

클라이언트가 두 헤더를 모두 전송하면 서버는 ETag를 우선적으로 처리한다. 시계 동기화 문제나 1초 미만의 수정을 ETag로 보완할 수 있다.

### Vary 헤더

`Vary`는 캐시된 응답이 어떤 요청 헤더에 따라 달라지는지 알리는 헤더다.

CDN과 프록시는 `Vary`에 명시된 헤더 값을 기준으로 별도의 캐시 항목을 유지한다.

```http
# 압축 방식에 따라 다른 캐시 항목 유지
Vary: Accept-Encoding

# 언어별로 다른 캐시 항목 유지
Vary: Accept-Language

# 인증 여부에 따라 캐시 분리
Vary: Authorization

# 여러 헤더 조합
Vary: Accept-Encoding, Accept-Language
```

`Vary: Accept-Encoding`은 거의 모든 콘텐츠에 필수다.

gzip을 지원하는 브라우저와 그렇지 않은 브라우저에게 동일한 캐시 항목이 서비스되면 안 되기 때문이다.

#### Anti-pattern: Vary 남용

`Vary: User-Agent`는 절대 사용해서는 안 된다.

User-Agent 값의 종류가 수천 가지에 달하기 때문에 캐시 히트율이 사실상 0%에 수렴하고, CDN 스토리지를 낭비한다. 브라우저별로 다른 응답이 필요하다면 URL을 분리하거나, 클라이언트 사이드 분기를 활용하는 것이 올바른 접근이다.

---

## CDN 동작 원리

CDN(Content Delivery Network)은 전 세계에 분산된 엣지 서버 네트워크다.

사용자는 오리진 서버 대신 지리적으로 가장 가까운 엣지 서버에서 응답을 받는다. 물리적 거리가 줄어들면 RTT(Round-Trip Time)가 감소하고, 응답 속도가 극적으로 개선된다.

### 요청 흐름

CDN의 기본 요청 처리 흐름을 이해하는 것이 설정의 출발점이다.

```
사용자 요청
    │
    ▼
DNS 해석 (CDN이 가장 가까운 엣지 노드 반환)
    │
    ▼
엣지 서버 (PoP: Point of Presence)
    │
    ├─ Cache HIT → 즉시 응답 반환 (오리진 미접촉)
    │
    └─ Cache MISS
           │
           ▼
       오리진 서버에 요청
           │
           ▼
       응답 수신 → 엣지 캐시 저장 → 사용자에게 반환
```

첫 번째 요청(MISS)은 오리진까지 도달하지만, 이후 동일한 리소스에 대한 요청은 모두 엣지에서 처리된다.

글로벌 서비스에서는 서울 엣지가 캐시를 채우면 서울 사용자는 이후부터 오리진 없이 응답을 받는다.

### 엣지 캐시 무효화 (Cache Invalidation)

캐시 무효화는 CDN 운영에서 가장 어려운 문제 중 하나다.

"컴퓨터 과학에서 어려운 문제는 두 가지다. 캐시 무효화와 네이밍"이라는 말이 있을 정도다.

#### CloudFront에서의 무효화

```bash
# AWS CLI로 특정 경로 무효화
aws cloudfront create-invalidation \
    --distribution-id E1234567890 \
    --paths "/images/*" "/api/products*"

# 전체 무효화 (비용 발생 주의)
aws cloudfront create-invalidation \
    --distribution-id E1234567890 \
    --paths "/*"
```

CloudFront는 매월 1,000건의 무효화 경로를 무료로 제공하고, 초과분은 요금이 부과된다.

전체 무효화(`/*`)는 하나의 경로로 카운트된다. 배포 시 전체 무효화가 필요한 경우 비용 관점에서 유리하다.

#### 무효화 대신 버저닝

무효화보다 효율적인 전략은 애초에 무효화가 필요 없는 구조를 만드는 것이다.

정적 자산에 콘텐츠 해시를 포함시키면 파일이 변경될 때마다 URL이 바뀌므로, 이전 URL의 캐시는 자연스럽게 소멸되고 새 URL은 항상 최신 파일을 가리킨다.

```
# 변경 전 (버저닝 없음)
/static/main.css → 무효화 필요

# 변경 후 (콘텐츠 해시 포함)
/static/main.abc123.css → 무효화 불필요
/static/main.xyz789.css → 새 파일, 새 URL
```

#### CloudFront 배포 설정 예시

```json
{
  "Origins": {
    "Items": [
      {
        "DomainName": "api.example.com",
        "Id": "ApiOrigin",
        "CustomOriginConfig": {
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "https-only"
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "CacheBehaviors": {
    "Items": [
      {
        "PathPattern": "/api/*",
        "TargetOriginId": "ApiOrigin",
        "ViewerProtocolPolicy": "https-only",
        "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
        "TTL": {
          "DefaultTTL": 0,
          "MaxTTL": 0
        }
      },
      {
        "PathPattern": "/static/*",
        "TargetOriginId": "S3Origin",
        "ViewerProtocolPolicy": "redirect-to-https",
        "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
        "TTL": {
          "DefaultTTL": 86400,
          "MaxTTL": 31536000
        }
      }
    ]
  }
}
```

API 엔드포인트(`/api/*`)는 TTL을 0으로 설정해 캐싱을 비활성화하고, 정적 자산(`/static/*`)은 최대 TTL로 설정해 엣지에 오래 유지한다.

---

## 브라우저 캐시 전략

브라우저 캐시는 사용자 디바이스에서 동작하는 가장 가까운 캐시다.

올바른 전략을 사용하면 네트워크 요청 자체를 없앨 수 있다. 리소스 종류별로 최적의 캐시 전략이 다르다.

### 리소스 유형별 전략

```
리소스 유형         | 캐시 전략                              | 이유
--------------------|----------------------------------------|-------------------------
HTML 문서           | no-cache 또는 max-age=0               | 항상 최신 HTML 필요
CSS/JS (해시 포함)  | max-age=31536000, immutable           | 해시가 변경 감지 역할
이미지 (해시 포함)  | max-age=31536000, immutable           | 동일
API 응답            | no-store 또는 short max-age           | 데이터 신선도 중요
Service Worker      | no-cache                              | 즉각 업데이트 필요
```

`immutable` 디렉티브는 캐시 유효 기간 내에는 재검증 요청도 보내지 않도록 지시한다.

콘텐츠 해시를 포함한 파일에만 사용해야 한다. 해시가 없는 파일에 사용하면 변경된 파일이 절대 갱신되지 않는다.

### Nginx에서의 캐시 헤더 설정

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    # HTML 파일: 항상 재검증
    location ~* \.html$ {
        add_header Cache-Control "public, no-cache";
        add_header X-Content-Type-Options "nosniff";
    }

    # 해시가 포함된 정적 자산: 1년 캐싱
    location ~* \.(css|js)$ {
        # 파일명에 해시 패턴이 있는 경우
        if ($uri ~* "\.[a-f0-9]{8,}\.(css|js)$") {
            add_header Cache-Control "public, max-age=31536000, immutable";
        }
        # 해시 없는 파일
        if ($uri !~* "\.[a-f0-9]{8,}\.(css|js)$") {
            add_header Cache-Control "public, max-age=3600";
        }
    }

    # 이미지 파일
    location ~* \.(jpg|jpeg|png|gif|webp|svg|ico)$ {
        add_header Cache-Control "public, max-age=86400, stale-while-revalidate=3600";
    }

    # 폰트 파일 (불변)
    location ~* \.(woff|woff2|ttf|eot)$ {
        add_header Cache-Control "public, max-age=31536000, immutable";
        add_header Access-Control-Allow-Origin "*";
    }

    # API
    location /api/ {
        proxy_pass http://backend;
        add_header Cache-Control "private, no-cache";
    }
}
```

`location` 블록 내에서 `if`를 사용하는 것은 Nginx에서 권장되지 않는 패턴이다.

실무에서는 `map` 디렉티브나 별도 location 블록으로 분리하는 것이 더 안전하다.

```nginx
# 권장 패턴: map으로 분리
map $uri $cache_control {
    ~*\.[a-f0-9]{8,}\.(css|js)$    "public, max-age=31536000, immutable";
    ~*\.(css|js)$                   "public, max-age=3600";
    ~*\.(jpg|jpeg|png|gif|webp)$    "public, max-age=86400";
    ~*\.(woff|woff2)$               "public, max-age=31536000, immutable";
    default                         "public, no-cache";
}

server {
    location /static/ {
        add_header Cache-Control $cache_control;
    }
}
```

### 정적 자산 버저닝

콘텐츠 해시 기반 버저닝은 현대 프론트엔드 빌드 도구의 표준 기능이다.

Webpack, Vite, Parcel 모두 기본적으로 지원한다.

```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        // JS: [name]-[hash].js 형태로 출력
        entryFileNames: 'assets/[name]-[hash].js',
        chunkFileNames: 'assets/[name]-[hash].js',
        // CSS: [name]-[hash].css 형태로 출력
        assetFileNames: 'assets/[name]-[hash][extname]',
      },
    },
    // 청크 분리 전략
    chunkSizeWarningLimit: 500,
  },
}
```

```javascript
// webpack.config.js
module.exports = {
  output: {
    filename: '[name].[contenthash:8].js',
    chunkFilename: '[name].[contenthash:8].chunk.js',
    assetModuleFilename: 'images/[name].[contenthash:8][ext]',
  },
};
```

빌드 결과물은 다음과 같은 형태가 된다.

```
dist/
├── index.html              ← no-cache (매번 최신 버전 확인)
├── assets/
│   ├── main-a8f3c2d1.js   ← 1년 캐싱 (해시 변경 시 자동 갱신)
│   ├── vendor-b4e7f9a2.js ← 1년 캐싱
│   └── main-c9d2e1f4.css  ← 1년 캐싱
```

`index.html`은 절대 캐싱하지 않는다.

HTML이 최신 해시 파일명을 참조하기 때문에, HTML만 새로 받으면 브라우저는 자동으로 변경된 자산을 요청한다.

### Service Worker와 캐시 전략

Service Worker는 브라우저와 네트워크 사이의 프록시로 동작한다.

정교한 캐시 전략을 JavaScript로 구현할 수 있다.

```javascript
// service-worker.js
const CACHE_VERSION = 'v2';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;

// 설치 시 핵심 자산 사전 캐싱
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) =>
      cache.addAll([
        '/',
        '/offline.html',
        '/assets/main-a8f3c2d1.js',
        '/assets/main-c9d2e1f4.css',
      ])
    )
  );
  self.skipWaiting();
});

// 요청 처리 전략
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);

  // API 요청: Network First (네트워크 우선, 실패 시 캐시)
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(request));
    return;
  }

  // 정적 자산: Cache First (캐시 우선, 미스 시 네트워크)
  if (url.pathname.startsWith('/assets/')) {
    event.respondWith(cacheFirst(request));
    return;
  }

  // HTML: Stale While Revalidate
  event.respondWith(staleWhileRevalidate(request));
});

async function networkFirst(request) {
  try {
    const response = await fetch(request);
    const cache = await caches.open(DYNAMIC_CACHE);
    cache.put(request, response.clone());
    return response;
  } catch {
    return caches.match(request);
  }
}

async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  const response = await fetch(request);
  const cache = await caches.open(STATIC_CACHE);
  cache.put(request, response.clone());
  return response;
}

async function staleWhileRevalidate(request) {
  const cache = await caches.open(DYNAMIC_CACHE);
  const cached = await cache.match(request);

  const fetchPromise = fetch(request).then((response) => {
    cache.put(request, response.clone());
    return response;
  });

  return cached || fetchPromise;
}
```

#### Anti-pattern: Service Worker 없는 오프라인 처리

Service Worker 없이 `Cache-Control`만으로 오프라인 경험을 구현하려는 시도는 동작하지 않는다.

`Cache-Control`은 네트워크가 연결된 상태에서의 캐시 동작을 제어한다. 오프라인 폴백, 백그라운드 동기화, 푸시 알림은 반드시 Service Worker를 통해 구현해야 한다.

---

## CDN 선택 기준

CDN 선택은 단순히 가격의 문제가 아니다.

지연 시간, 글로벌 PoP 분포, 엣지 컴퓨팅 기능, 보안, 운영 편의성을 종합적으로 고려해야 한다.

### Amazon CloudFront

AWS 생태계와의 통합이 핵심 강점이다.

S3, ALB, API Gateway와 원클릭으로 연결되며, WAF, Shield, ACM과 자연스럽게 통합된다.

**강점:**
- AWS 서비스와의 깊은 통합
- Lambda@Edge / CloudFront Functions로 엣지 컴퓨팅
- 글로벌 PoP 450개 이상
- AWS 내부 트래픽 비용 절감

**약점:**
- 캐시 무효화 지연 (수분 소요 가능)
- 설정 복잡도가 높음
- 실시간 로그 분석 기능 제한적

```python
# boto3로 CloudFront 배포 자동화
import boto3

cloudfront = boto3.client('cloudfront')

def invalidate_paths(distribution_id: str, paths: list[str]) -> str:
    response = cloudfront.create_invalidation(
        DistributionId=distribution_id,
        InvalidationBatch={
            'Paths': {
                'Quantity': len(paths),
                'Items': paths,
            },
            'CallerReference': str(time.time()),
        }
    )
    return response['Invalidation']['Id']

# 배포 시 HTML과 동적 경로만 무효화
invalidate_paths('E1234567890', ['/index.html', '/api/sitemap.xml'])
```

### Fastly

실시간 캐시 퍼지(purge)가 Fastly의 최대 강점이다.

수백 밀리초 내에 전 세계 엣지에서 캐시를 무효화할 수 있어, 뉴스 미디어나 전자상거래처럼 콘텐츠 변경이 빈번한 서비스에 적합하다.

**강점:**
- 실시간 캐시 퍼지 (150ms 이내)
- VCL(Varnish Configuration Language) 기반 세밀한 제어
- 뛰어난 실시간 분석 대시보드
- Compute@Edge (WebAssembly 기반 엣지 컴퓨팅)

**약점:**
- 높은 단가
- VCL 학습 곡선
- 아시아 PoP가 상대적으로 적음

```vcl
# Fastly VCL: 커스텀 캐시 키 설정
sub vcl_hash {
    # 기본 URL 해시
    set req.hash += req.url;
    set req.hash += req.http.host;

    # Accept-Encoding에 따라 별도 캐시
    if (req.http.Accept-Encoding ~ "gzip") {
        set req.hash += "gzip";
    }

    # 모바일/데스크톱 분리
    if (req.http.User-Agent ~ "(?i)mobile") {
        set req.hash += "mobile";
    }

    #FASTLY hash
    return(hash);
}
```

### Cloudflare

가성비와 보안의 균형이 뛰어나다.

무료 플랜도 무제한 대역폭을 제공하며, DDoS 방어와 WAF가 기본 포함된다.

**강점:**
- 전 세계 300개 이상 PoP (국내 PoP 포함)
- 무제한 대역폭 (플랜 무관)
- 통합 DDoS 방어 및 WAF
- Workers(V8 기반 엣지 컴퓨팅)
- 무료 SSL/TLS 인증서

**약점:**
- 캐시 정책이 상대적으로 단순
- 캐시 무효화 속도가 Fastly보다 느림
- 엔터프라이즈 기능은 고가

```javascript
// Cloudflare Worker: 동적 캐시 제어
export default {
  async fetch(request, env) {
    const url = new URL(request.url);

    // API 요청은 캐시 우회
    if (url.pathname.startsWith('/api/')) {
      return fetch(request, {
        cf: { cacheTtl: 0, cacheEverything: false }
      });
    }

    // 정적 자산은 엣지에서 1년 캐싱
    if (url.pathname.startsWith('/static/')) {
      return fetch(request, {
        cf: {
          cacheTtl: 31536000,
          cacheEverything: true,
          cacheKey: request.url,  // 쿼리 파라미터 포함
        }
      });
    }

    // 기본: 오리진의 Cache-Control 헤더 준수
    return fetch(request);
  }
};
```

### CDN 선택 의사결정 매트릭스

```
상황                                          | 추천
----------------------------------------------|------------------
AWS 중심 인프라, 비용 최적화 중요              | CloudFront
뉴스, 이커머스, 잦은 콘텐츠 변경             | Fastly
소규모 서비스, 보안 우선, 비용 민감           | Cloudflare (무료/Pro)
WebAssembly 엣지 컴퓨팅 필요                 | Fastly Compute@Edge
V8/JS 엣지 컴퓨팅, 간단한 로직              | Cloudflare Workers
국내 트래픽 비중 높음                        | Cloudflare (서울 PoP)
멀티 CDN 전략 (고가용성)                     | CloudFront + Cloudflare
```

### Anti-pattern: 모든 경로를 CDN으로 라우팅

인증이 필요한 API, 개인화된 응답, 실시간 데이터를 CDN을 통해 캐싱하는 실수는 실제로 자주 발생한다.

`Authorization` 헤더가 있는 요청은 반드시 `Cache-Control: private` 또는 `no-store`로 설정해야 한다. CDN이 A 사용자의 인증된 응답을 캐싱하고, B 사용자에게 동일한 응답을 반환하는 심각한 보안 사고가 발생할 수 있다.

```nginx
# 인증 API 보안 캐시 설정
location /api/user/ {
    proxy_pass http://backend;
    # 절대 공개 캐시하지 않음
    add_header Cache-Control "private, no-store, no-cache";
    add_header Pragma "no-cache";
}
```

---

## 캐시 모니터링과 튜닝

캐시를 설정한 이후에는 실제 히트율을 측정하고 지속적으로 튜닝해야 한다.

히트율이 낮은 구간을 찾아 원인을 분석하는 것이 성능 개선의 핵심이다.

### 캐시 히트율 측정

CDN에서 반환하는 `X-Cache` 헤더로 히트 여부를 확인할 수 있다.

```bash
# CloudFront 캐시 상태 확인
curl -I https://example.com/static/main.abc123.js

# 응답 헤더에서 캐시 상태 확인
# X-Cache: Hit from cloudfront   → 캐시 히트
# X-Cache: Miss from cloudfront  → 캐시 미스
# X-Cache: RefreshHit from cloudfront → 재검증 후 캐시 사용
```

### Nginx 캐시 히트율 로깅

```nginx
# 캐시 상태를 로그에 기록
log_format cache_log '$remote_addr - [$time_local] '
                     '"$request" $status '
                     'cache:$upstream_cache_status '
                     'rt:$request_time';

access_log /var/log/nginx/access.log cache_log;

# upstream_cache_status 값:
# HIT      → 캐시 히트
# MISS     → 캐시 미스 (오리진 접촉)
# EXPIRED  → 만료된 캐시 (재검증)
# BYPASS   → proxy_cache_bypass 조건 충족
# STALE    → 만료됐지만 stale 허용으로 반환
```

히트율 목표는 리소스 유형에 따라 다르다.

정적 자산은 95% 이상을 목표로 한다. API 응답은 캐시 가능한 엔드포인트에 한해 60~80%를 목표로 한다. HTML 문서는 캐싱하지 않거나 매우 짧은 TTL을 설정한다.

---

HTTP 캐싱과 CDN은 단독으로도 강력하지만, 두 기술을 함께 계층화했을 때 진정한 효과가 발휘된다.

브라우저 캐시가 첫 번째 방어선, CDN 엣지가 두 번째 방어선, 오리진 서버가 최후의 방어선이 되는 구조를 설계하라. 캐시 전략은 한 번 설정하고 잊는 것이 아니라, 히트율 데이터를 기반으로 지속적으로 튜닝해야 하는 살아있는 설정이다.

---

## 참고 자료

- [MDN Web Docs: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching) — HTTP 캐시 헤더 완전 참고서
- [Amazon CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/) — CloudFront 설정 공식 문서
- [Fastly Documentation: Caching](https://developer.fastly.com/learning/concepts/caching/) — Fastly VCL 및 캐시 전략
- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/) — Workers 엣지 컴퓨팅 가이드
- [web.dev: Love your cache](https://web.dev/articles/love-your-cache) — Google의 웹 캐시 전략 가이드
- [Nginx caching guide](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) — Nginx 프록시 캐시 설정 레퍼런스
