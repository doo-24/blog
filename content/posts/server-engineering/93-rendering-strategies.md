---
title: "[Frontend 통합] 1편 — 렌더링 전략: SSR, CSR, SSG를 서버 관점에서 이해하기"
date: 2026-03-17T12:07:00+09:00
draft: false
tags: ["SSR", "CSR", "SSG", "Next.js", "서버"]
series: ["Frontend 통합"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 14
summary: "CSR/SSR/SSG/ISR 각 방식의 서버 부담과 인프라 요구사항, Next.js 서버 사이드 동작 원리와 Node.js 서버 운영, SSR 서버 부하 관리(캐싱, 스트리밍 SSR), 정적 사이트 배포 전략(S3+CloudFront, Vercel)까지"
---

프론트엔드 렌더링 전략은 단순히 화면을 그리는 방식의 차이가 아니다. 서버 관점에서 보면 CPU 소비, 메모리 압력, 네트워크 대역폭, 캐시 가능성이 근본적으로 달라지는 인프라 결정이다.

"어떤 프레임워크를 쓸지"보다 "이 선택이 서버에 무엇을 요구하는지"를 먼저 이해해야 한다.

---

## 1. 렌더링 전략 개요: 네 가지 모델

현대 웹 애플리케이션에서 사용하는 렌더링 전략은 크게 네 가지다.

**CSR (Client-Side Rendering)**: 서버는 빈 HTML 껍데기와 JavaScript 번들만 전송한다. 브라우저가 JS를 실행해 DOM을 직접 구성한다.

**SSR (Server-Side Rendering)**: 요청마다 서버가 완성된 HTML을 생성해 응답한다. 서버 CPU가 렌더링 연산을 담당한다.

**SSG (Static Site Generation)**: 빌드 타임에 HTML을 미리 생성한다. 런타임에 서버 연산이 없고, CDN이 파일을 그대로 서빙한다.

**ISR (Incremental Static Regeneration)**: SSG의 변형이다. 정적 파일을 주기적으로 또는 요청 기반으로 재생성한다. Next.js가 대표적으로 지원한다.

---

## 2. CSR — 서버 부담 최소, 클라이언트 부담 최대

### 서버에서 일어나는 일

CSR에서 서버가 하는 일은 단순하다. HTML 파일 하나와 JS 번들을 전송하는 것이 전부다.

```
GET /  -> index.html (2KB)
GET /static/js/main.12ab34.js  -> 500KB~2MB
```

서버 입장에서 HTML 응답은 정적 파일 서빙과 동일하다. NGINX나 S3가 처리해도 된다.

### 인프라 요구사항

서버 컴퓨팅 자원이 거의 필요 없다. 정적 파일 호스팅으로 충분하다.

API 서버는 별도로 존재하며, 브라우저가 직접 API를 호출한다. API 서버에 인증, 인가, 비즈니스 로직이 집중된다.

```
브라우저 -> CDN (HTML + JS)
브라우저 -> API 서버 (데이터)
```

### 트레이드오프

**장점**: 서버 비용이 낮다. CDN 캐시 히트율이 100%에 가깝다.

**단점**: 초기 로딩이 느리다. JavaScript 번들이 파싱되기 전까지 화면이 비어 있다.

SEO가 문제다. 검색 크롤러가 JS를 실행하지 않으면 빈 페이지를 인덱싱한다. Google은 JS 실행을 지원하지만 크롤링 지연이 발생할 수 있다.

Time to First Byte(TTFB)는 빠르지만, Time to Interactive(TTI)가 길다. 사용자는 화면이 뜨기까지 빈 화면을 본다.

### 적합한 시나리오

로그인이 필요한 SaaS 대시보드, 관리자 패널처럼 SEO가 불필요하고 인증 뒤에 있는 페이지에 적합하다.

---

## 3. SSR — 서버가 렌더링을 책임진다

### 서버에서 일어나는 일

SSR에서는 요청마다 서버가 HTML을 생성한다. 다음 순서로 처리된다.

```
1. 요청 수신
2. 데이터 페칭 (DB 쿼리, 외부 API 호출)
3. React/Vue 컴포넌트 트리를 HTML 문자열로 렌더링
4. HTML 응답 전송
5. 브라우저에서 Hydration (JS가 이벤트 핸들러 부착)
```

3번 단계가 CPU 집약적이다. 컴포넌트 트리가 복잡할수록, 데이터가 많을수록 연산 비용이 올라간다.

### 메모리와 CPU 소비 패턴

Node.js 기반 SSR 서버에서 주의할 점은 요청 당 메모리 할당이다.

요청마다 React 컴포넌트 인스턴스, 렌더링 결과 문자열, 데이터 페칭 결과가 힙에 올라간다. 트래픽이 높을 때 GC(가비지 컬렉션) 압력이 높아진다.

```
요청 100 rps, 페이지 당 평균 힙 사용 2MB
-> 순간 힙 사용량: 200MB 이상
```

Node.js 기본 힙 한도(~1.5GB)에 주의해야 한다. 트래픽 급증 시 OOM(Out of Memory)이 발생할 수 있다.

### 인프라 요구사항

SSR 서버는 상태가 없어야(stateless) 한다. 수평 확장을 위해 각 인스턴스가 독립적으로 요청을 처리해야 한다.

세션 정보나 캐시는 외부 저장소(Redis, Memcached)에 두어야 한다. 인스턴스 로컬 메모리에 저장하면 로드 밸런서가 다른 인스턴스로 라우팅할 때 데이터가 없다.

```yaml
# 최소 권장 구성 (중간 트래픽)
Web Server: 2-4 vCPU, 4-8GB RAM x N 인스턴스
Cache: Redis 1-2GB
Load Balancer: ALB 또는 NGINX
```

### 트레이드오프

**장점**: 초기 HTML이 완성된 상태로 전달된다. SEO에 유리하다. TTFB와 FCP(First Contentful Paint)가 개선된다.

**단점**: 서버 비용이 CSR 대비 높다. 서버 확장이 필요하다. 서버가 다운되면 페이지 자체가 서빙되지 않는다.

데이터 페칭 레이턴시가 TTFB에 직접 영향을 미친다. DB 쿼리가 200ms 걸리면 그만큼 응답이 느려진다.

---

## 4. SSG — 빌드 타임에 결정, 런타임에 제로 연산

### 작동 방식

빌드 파이프라인이 모든 페이지를 HTML 파일로 생성한다.

```bash
# Next.js 빌드 시
next build
# -> .next/static/ 에 모든 페이지 HTML 생성
# -> 총 100개 페이지 = 100개의 .html 파일
```

런타임에 서버는 파일을 그대로 전송한다. 연산이 없다.

### 서버 부담

런타임 서버 부담은 사실상 제로다. CDN이 HTML 파일을 캐시하면 오리진 서버에 요청이 도달하지도 않는다.

빌드 타임에 CPU를 많이 쓴다. 페이지 수가 수천, 수만 개면 빌드 시간이 수십 분에 달할 수 있다.

```
페이지 1,000개 x 평균 빌드 시간 1초/페이지 = 16분 빌드
```

### 인프라 요구사항

CDN + 오리진 스토리지(S3 등)로 충분하다. 웹 서버 인스턴스가 필요 없다.

```
CDN (CloudFront) -> S3 (HTML, JS, CSS, 이미지)
```

운영 비용이 극히 낮다. S3 스토리지 비용과 CloudFront 전송 비용만 발생한다.

### 한계

콘텐츠가 변경되면 재빌드가 필요하다. 실시간 데이터를 보여줄 수 없다.

페이지 수가 매우 많으면 빌드 시간이 문제가 된다. 이 한계를 해결하기 위해 ISR이 등장했다.

### 적합한 시나리오

마케팅 랜딩 페이지, 블로그, 문서 사이트처럼 콘텐츠 변경이 드물고 트래픽이 많은 경우에 최적이다.

---

## 5. ISR — SSG와 SSR의 절충

### 작동 방식

ISR은 정적 페이지를 캐시하되, 주기적으로 백그라운드에서 재생성한다.

```javascript
// Next.js App Router (서버 컴포넌트)
export const revalidate = 60; // 60초마다 재검증

// 또는 Pages Router
export async function getStaticProps() {
  return {
    props: { data },
    revalidate: 60, // 초 단위
  };
}
```

첫 요청에 캐시된 HTML을 반환하면서, 백그라운드에서 새 HTML을 생성한다. 다음 요청부터 새 페이지를 서빙한다. 이를 "stale-while-revalidate" 패턴이라 한다.

### On-Demand ISR

Next.js는 시간 기반 외에 API를 통한 명시적 재검증을 지원한다.

```javascript
// /api/revalidate.js
export default async function handler(req, res) {
  await res.revalidate('/posts/my-post');
  return res.json({ revalidated: true });
}
```

CMS에서 콘텐츠를 수정하면 웹훅으로 이 API를 호출하는 패턴이 일반적이다.

### 서버 부담

첫 요청 이후에는 캐시를 서빙하므로 부담이 낮다. 재검증 시에만 SSR과 유사한 연산이 발생한다.

단, Next.js ISR은 자체 서버(Node.js)가 필요하다. 순수 정적 호스팅으로는 ISR을 사용할 수 없다.

---

## 6. Next.js 서버 사이드 동작 원리

### Node.js 서버로서의 Next.js

`next start`는 Node.js HTTP 서버를 실행한다. 내부적으로 `http.createServer`를 사용하며, Next.js 라우터가 요청을 처리한다.

```bash
next start
# -> Node.js 프로세스 시작
# -> 기본 포트 3000에서 리스닝
# -> 요청당 렌더링 파이프라인 실행
```

프로덕션에서는 보통 NGINX나 ALB를 앞에 두고 Next.js는 upstream으로 운영한다.

```
클라이언트 -> NGINX (SSL termination, 정적 파일) -> Next.js (동적 렌더링)
```

### 요청 처리 파이프라인 (App Router)

Next.js 13+ App Router 기준으로 요청이 처리되는 흐름이다.

```
요청 수신
  -> Middleware 실행 (Edge Runtime)
  -> 라우트 매칭
  -> 레이아웃/페이지 서버 컴포넌트 실행
    -> 데이터 페칭 (fetch, DB 쿼리)
    -> React Server Components 렌더링
  -> HTML 스트리밍 응답
  -> 클라이언트: Hydration
```

Middleware는 Edge Runtime에서 실행된다. Node.js API를 사용할 수 없고, 경량 연산만 가능하다.

### 서버 컴포넌트 vs 클라이언트 컴포넌트

App Router의 핵심 개념이다. 서버 컴포넌트는 서버에서만 실행되고, 번들에 포함되지 않는다.

```javascript
// 서버 컴포넌트 (기본값)
// 'use server' 불필요 — 서버가 기본
async function ProductList() {
  const products = await db.query('SELECT * FROM products');
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}

// 클라이언트 컴포넌트
'use client';
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

서버 컴포넌트는 DB에 직접 접근할 수 있다. 클라이언트로 API 호출 레이어가 필요 없다. 단, 이벤트 핸들러, useState, useEffect는 클라이언트 컴포넌트에서만 쓸 수 있다.

### 메모리 관리 주의사항

Next.js 서버 컴포넌트에서 모듈 레벨 변수에 요청 데이터를 저장하면 안 된다.

```javascript
// 위험: 모듈 레벨 캐시
let cache = {};

export default async function Page() {
  cache[userId] = data; // 메모리 누수, 요청 간 데이터 오염
}

// 안전: React cache() 사용
import { cache } from 'react';

const getData = cache(async (id) => {
  return await fetchData(id);
});
```

`React.cache()`는 단일 요청 범위 내에서만 캐시를 유지한다. 요청 간 상태 오염을 방지한다.

---

## 7. SSR 부하 관리

### 캐싱 전략

SSR 서버의 부하를 줄이는 가장 효과적인 방법은 응답 캐싱이다.

**HTTP 캐시 헤더 활용**

```javascript
// Next.js Route Handler
export async function GET() {
  const data = await fetchData();
  return Response.json(data, {
    headers: {
      'Cache-Control': 'public, s-maxage=60, stale-while-revalidate=300',
    },
  });
}
```

`s-maxage`는 CDN(공유 캐시) TTL을 지정한다. `stale-while-revalidate`는 만료 후에도 캐시를 서빙하면서 백그라운드에서 갱신하는 시간을 지정한다.

**Redis를 활용한 렌더링 캐시**

```javascript
// SSR 응답 캐싱 예시
export async function getServerSideProps({ params }) {
  const cacheKey = `page:${params.id}`;
  const cached = await redis.get(cacheKey);

  if (cached) {
    return { props: JSON.parse(cached) };
  }

  const data = await fetchFromDB(params.id);
  await redis.setex(cacheKey, 300, JSON.stringify(data)); // 5분 TTL
  return { props: data };
}
```

DB 쿼리 비용이 높은 경우 Redis 캐시가 응답 속도를 10배 이상 개선할 수 있다.

**Next.js fetch() 캐싱**

Next.js App Router에서 `fetch()`는 기본적으로 캐시된다.

```javascript
// 기본: 요청 메모이제이션 + HTTP 캐시
const data = await fetch('https://api.example.com/data');

// 캐시 비활성화 (항상 최신 데이터)
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store'
});

// 시간 기반 재검증
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 3600 } // 1시간
});
```

### 스트리밍 SSR

React 18의 Suspense와 Next.js의 스트리밍을 활용하면 TTFB를 개선하면서 SSR의 데이터 의존성 문제를 완화할 수 있다.

```javascript
// app/page.tsx
import { Suspense } from 'react';

export default function Page() {
  return (
    <div>
      <h1>빠르게 렌더링</h1>
      <Suspense fallback={<div>로딩 중...</div>}>
        <SlowComponent /> {/* 데이터 페칭이 느린 컴포넌트 */}
      </Suspense>
    </div>
  );
}
```

스트리밍 SSR은 HTML을 청크 단위로 전송한다. 페이지 전체가 완성될 때까지 기다리지 않고, 준비된 부분부터 브라우저에 전송한다.

```
서버 -> 브라우저
[청크 1: <html><head>...</head><body><h1>빠르게 렌더링</h1>]
[청크 2: <div>로딩 중...</div>]  <- Suspense fallback
[청크 3: <실제 컴포넌트 HTML>]  <- 데이터 준비 완료 후
```

브라우저는 첫 청크를 받자마자 화면을 그리기 시작한다. 사용자 인지 성능이 향상된다.

### 스케일링 전략

**수평 확장**: SSR 서버는 stateless이어야 한다. 상태를 외부에 두면 인스턴스를 자유롭게 추가할 수 있다.

```yaml
# Kubernetes HPA 예시
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

CPU 사용률 70%를 기준으로 자동 스케일아웃한다.

**Node.js 클러스터**: 단일 서버에서 멀티코어를 활용하려면 PM2 cluster 모드를 사용한다.

```bash
pm2 start npm --name "nextjs" -i max -- start
# -i max: CPU 코어 수만큼 프로세스 생성
```

**연결 풀링**: SSR에서 DB에 직접 접근한다면 연결 풀이 필수다. 요청마다 새 DB 연결을 열면 성능이 급격히 저하된다.

```javascript
// Prisma 연결 풀 설정
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL + '?connection_limit=10'
    }
  }
});

// 싱글톤 패턴 (개발 환경 핫 리로드 시 연결 누수 방지)
const globalForPrisma = globalThis;
export const db = globalForPrisma.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db;
```

---

## 8. 정적 사이트 배포 전략

### S3 + CloudFront

AWS에서 정적 사이트를 배포하는 표준 패턴이다.

**S3 설정**

```bash
# 빌드 결과물 업로드
aws s3 sync ./out s3://my-blog-bucket --delete

# 정적 웹사이트 호스팅 활성화
aws s3 website s3://my-blog-bucket \
  --index-document index.html \
  --error-document 404.html
```

**CloudFront 배포 설정 핵심**

```json
{
  "DefaultCacheBehavior": {
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "managed-caching-optimized",
    "Compress": true
  },
  "CustomErrorResponses": [
    {
      "ErrorCode": 404,
      "ResponsePagePath": "/404.html",
      "ResponseCode": 404
    }
  ]
}
```

SPA처럼 클라이언트 라우팅을 사용하는 경우 404를 index.html로 리다이렉트해야 한다. SSG는 각 경로마다 HTML이 있으므로 이 처리가 불필요하다.

**캐시 무효화**

새 배포 시 CloudFront 캐시를 무효화해야 최신 파일이 서빙된다.

```bash
# 배포 스크립트
aws s3 sync ./out s3://my-blog-bucket --delete
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*"
```

파일명에 콘텐츠 해시를 포함하면(Next.js 기본 동작) 캐시 무효화 없이 버전 관리가 된다. HTML 파일만 무효화하면 충분하다.

```bash
# HTML만 무효화 (JS/CSS는 해시로 관리)
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/*.html" "/index.html"
```

### Vercel

Next.js 개발사가 운영하는 배포 플랫폼이다. SSR, SSG, ISR, Edge Functions를 통합 지원한다.

**동작 원리**

Vercel은 Next.js 빌드 결과를 분석해 자동으로 인프라를 구성한다.

```
정적 파일 -> Vercel Edge Network (글로벌 CDN)
SSR 페이지 -> Serverless Function (요청 시 실행)
Edge Middleware -> Edge Runtime (전 세계 PoP에서 실행)
```

서버를 직접 관리할 필요가 없다. 인스턴스 스케일링, 로드 밸런싱이 자동으로 처리된다.

**Serverless 모델의 특성**

Vercel SSR은 Serverless Function으로 실행된다. 콜드 스타트가 발생할 수 있다.

트래픽이 없다가 갑자기 요청이 오면 함수 초기화 시간(100ms~수초)이 TTFB에 추가된다. 지속적 트래픽이 있는 서비스라면 문제가 적지만, 간헐적 트래픽이라면 체감할 수 있다.

함수 메모리와 실행 시간 제한이 있다. Vercel 기본값은 1GB RAM, 10초 타임아웃(Pro 플랜 60초)이다.

**비용 모델**

SSG 페이지는 CDN 서빙이므로 호출 비용이 발생하지 않는다. SSR 페이지는 함수 실행 시간에 따라 과금된다.

트래픽이 많은 SSR 페이지는 캐시를 적극 활용해야 비용을 통제할 수 있다.

```javascript
// Vercel에서 SSR 응답 캐싱
export const dynamic = 'force-dynamic'; // 캐시 비활성화
// 또는
export const revalidate = 3600; // 1시간 캐시
```

### Netlify

Vercel과 유사한 JAMstack 플랫폼이다. Next.js를 포함한 다양한 프레임워크를 지원한다.

정적 파일은 Netlify CDN에서 서빙하고, 서버리스 기능은 Netlify Functions(AWS Lambda 기반)으로 처리한다.

Next.js SSR을 Netlify에서 사용하려면 `@netlify/plugin-nextjs` 플러그인이 필요하다.

```toml
# netlify.toml
[[plugins]]
package = "@netlify/plugin-nextjs"

[build]
  command = "next build"
  publish = ".next"
```

---

## 9. 전략 선택 가이드

어떤 렌더링 전략을 선택할지는 콘텐츠 성격, 트래픽, 팀 운영 능력에 따라 결정한다.

| 기준 | CSR | SSR | SSG | ISR |
|------|-----|-----|-----|-----|
| 서버 비용 | 낮음 | 높음 | 매우 낮음 | 낮음 |
| 실시간 데이터 | 가능 | 가능 | 불가 | 제한적 |
| SEO | 제한적 | 우수 | 우수 | 우수 |
| 빌드 시간 | 빠름 | 빠름 | 느릴 수 있음 | 빠름 |
| 운영 복잡도 | 낮음 | 높음 | 낮음 | 중간 |

**의사결정 흐름**

콘텐츠가 자주 바뀌는가? -> 예 -> SSR 또는 ISR
SEO가 중요한가? -> 아니오 -> CSR
빌드 시 데이터가 확정되는가? -> 예 -> SSG
콘텐츠가 수시간/일 단위로 갱신되는가? -> ISR

하나의 애플리케이션에서 여러 전략을 혼합할 수 있다. 홈 페이지와 마케팅 페이지는 SSG, 사용자 대시보드는 CSR, 상품 목록은 ISR, 개인화 페이지는 SSR로 구성하는 방식이다.

---

## 10. 모니터링과 성능 지표

렌더링 전략을 선택한 뒤에는 실제 서버 지표를 모니터링해야 한다.

**SSR 서버 핵심 지표**

- TTFB (Time to First Byte): 목표 200ms 이하
- 서버 CPU 사용률: 70% 이하 유지
- 메모리 사용량 추이: 메모리 누수 여부 확인
- 오류율: 5xx 응답 비율

**Next.js 내장 측정 도구**

```javascript
// next.config.js
module.exports = {
  experimental: {
    instrumentationHook: true,
  },
};

// instrumentation.js
export function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    // OpenTelemetry 초기화
  }
}
```

Next.js 15부터 OpenTelemetry 통합이 안정화되어 있다. Datadog, Grafana Cloud, Honeycomb과 연동해 분산 추적을 할 수 있다.

**Web Vitals 서버 사이드 연관**

Core Web Vitals는 브라우저에서 측정되지만, 서버 성능이 직접 영향을 미친다. TTFB가 높으면 LCP(Largest Contentful Paint)가 나빠진다. SSR 서버 응답 시간을 최적화하면 LCP 점수가 개선된다.

---

## 마무리

렌더링 전략은 프론트엔드 선택이 아니라 시스템 전체 설계 결정이다. CSR은 서버 부담을 클라이언트로 이전하고, SSR은 서버가 모든 렌더링 비용을 부담하며, SSG는 빌드 타임으로 비용을 앞당기고, ISR은 그 사이를 절충한다.

서버 엔지니어 관점에서 핵심은 "어떤 연산이 어디에서 실행되는가"다. 이를 명확히 파악하면 인프라 설계, 캐시 전략, 스케일링 계획을 올바르게 세울 수 있다.

다음 편에서는 Next.js API Routes와 BFF(Backend For Frontend) 패턴을 서버 구조 관점에서 살펴본다.

---

## 참고 자료

- [Next.js 공식 문서 - Rendering](https://nextjs.org/docs/app/building-your-application/rendering)
- [Next.js 공식 문서 - Caching](https://nextjs.org/docs/app/building-your-application/caching)
- [Vercel - Edge Network 문서](https://vercel.com/docs/edge-network/overview)
- [React 공식 문서 - Server Components](https://react.dev/reference/rsc/server-components)
- [Web.dev - Rendering on the Web](https://web.dev/articles/rendering-on-the-web)
- [AWS - CloudFront + S3 정적 호스팅](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [Node.js - Cluster 모듈](https://nodejs.org/api/cluster.html)
