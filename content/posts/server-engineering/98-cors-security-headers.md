---
title: "[Frontend 통합] 6편 — CORS 실전과 보안 헤더: 브라우저와 서버의 협상"
date: 2026-03-17T12:02:00+09:00
draft: false
tags: ["CORS", "CSP", "보안 헤더", "Preflight", "서버"]
series: ["Frontend 통합"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 14
summary: "CORS Preflight 동작 원리(Simple vs Preflighted Request), CORS 설정 실전(Origin 관리, 와일드카드 위험, credentials), 보안 헤더 종합(CSP, X-Frame-Options, Referrer-Policy), 로컬 개발 CORS 우회까지"
---

브라우저는 다른 출처(Origin)에 요청을 보낼 때 서버의 허락을 먼저 구한다.
이 협상 프로토콜이 CORS(Cross-Origin Resource Sharing)다.

서버 엔지니어 입장에서 CORS는 "어떻게 허용할 것인가"의 문제다.
잘못 설정하면 프론트엔드가 막히고, 과도하게 열어두면 보안 구멍이 생긴다.

이번 글에서는 CORS 동작 원리부터 실전 설정, 그리고 함께 다뤄야 할 보안 헤더까지 서버 관점에서 정리한다.

---

## 1. CORS Preflight — 브라우저의 사전 협상

### Same-Origin Policy가 출발점이다

브라우저는 기본적으로 **Same-Origin Policy**를 강제한다.
`https://app.example.com`에서 로드된 JavaScript가 `https://api.example.com`에 요청을 보내면 출처가 다르므로 차단된다.

출처(Origin)는 **프로토콜 + 호스트 + 포트**의 조합이다.
`https://example.com:443`과 `http://example.com:80`은 다른 출처다.

CORS는 이 제약을 서버가 명시적으로 완화할 수 있게 해주는 메커니즘이다.
서버가 특정 출처를 허용하겠다고 응답 헤더로 선언하면, 브라우저가 요청을 통과시킨다.

### Simple Request — 사전 협상 없이 바로 전송

모든 크로스-오리진 요청이 Preflight를 거치는 것은 아니다.
다음 조건을 모두 만족하면 **Simple Request**로 처리된다.

**메서드 조건:**
- `GET`, `POST`, `HEAD` 중 하나

**헤더 조건:**
- `Accept`, `Accept-Language`, `Content-Language`, `Content-Type`만 사용
- `Content-Type`은 `application/x-www-form-urlencoded`, `multipart/form-data`, `text/plain`만 허용

Simple Request는 브라우저가 요청을 바로 보내고, 응답에 CORS 헤더가 있는지 확인한다.
CORS 헤더가 없거나 허용된 Origin이 아니면 응답을 JavaScript에 노출하지 않는다.

```
GET /api/data HTTP/1.1
Origin: https://app.example.com
Host: api.example.com
```

서버 응답:
```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
Content-Type: application/json
```

브라우저는 `Access-Control-Allow-Origin` 값이 요청한 Origin과 일치하면 응답을 JavaScript에 넘긴다.

### Preflighted Request — 실제 요청 전 OPTIONS 먼저

Simple Request 조건을 벗어나면 브라우저는 **Preflight** 요청을 먼저 보낸다.
실제 요청을 보내도 되는지 서버에 묻는 것이다.

다음 상황에서 Preflight가 발생한다:
- `PUT`, `DELETE`, `PATCH` 등의 메서드 사용
- `Authorization`, `Content-Type: application/json` 등의 커스텀 헤더 사용
- `credentials: 'include'` 모드 사용

Preflight 요청은 `OPTIONS` 메서드로 전송된다:

```
OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Authorization, Content-Type
Host: api.example.com
```

서버는 이 OPTIONS 요청에 허용 여부를 응답해야 한다:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

브라우저는 Preflight 응답이 OK이면 실제 요청을 전송한다.
`Access-Control-Max-Age`를 설정하면 지정된 시간(초) 동안 Preflight 결과를 캐시해서 반복 요청을 줄인다.

### Preflight 흐름 전체 그림

```
브라우저                              서버
  |                                    |
  |-- OPTIONS /api/users ------------>|
  |   Origin: https://app.example.com |
  |   Access-Control-Request-Method:  |
  |     DELETE                        |
  |                                    |
  |<-- 204 No Content ----------------|
  |    Access-Control-Allow-Origin:   |
  |      https://app.example.com      |
  |    Access-Control-Allow-Methods:  |
  |      GET, POST, DELETE            |
  |                                    |
  |-- DELETE /api/users/123 -------->|
  |   Origin: https://app.example.com |
  |   Authorization: Bearer ...       |
  |                                    |
  |<-- 200 OK ------------------------|
  |    Access-Control-Allow-Origin:   |
  |      https://app.example.com      |
```

서버 엔지니어가 놓치기 쉬운 포인트: **OPTIONS 요청도 인증 미들웨어를 통과해야** 한다.
OPTIONS 요청에 Authorization 헤더가 없다는 이유로 401을 반환하면 Preflight가 실패한다. Preflight는 브라우저가 자동으로 보내는 사전 탐색 요청이지 실제 데이터 요청이 아니므로, 인증 검사 없이 CORS 정책만 응답하면 된다. 실제 인증은 뒤따르는 본 요청(GET/POST/DELETE 등)에서 검사한다.

---

## 2. CORS 설정 실전

### Origin 관리 — 동적 허용 목록

프로덕션 환경에서는 허용할 Origin을 명시적으로 관리해야 한다.
환경별로 다른 Origin이 필요하므로 설정 파일이나 환경변수로 관리하는 게 현실적이다.

**Spring Boot 예시 — 전역 CORS 설정:**

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Value("${cors.allowed-origins}")
    private List<String> allowedOrigins;

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(allowedOrigins.toArray(new String[0]))
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .allowedHeaders("Authorization", "Content-Type", "X-Requested-With")
            .exposedHeaders("X-Total-Count", "Link")
            .allowCredentials(true)
            .maxAge(86400);
    }
}
```

`application.yml`:
```yaml
cors:
  allowed-origins:
    - https://app.example.com
    - https://admin.example.com
```

개발 환경에서는 `application-dev.yml`로 오버라이드:
```yaml
cors:
  allowed-origins:
    - http://localhost:3000
    - http://localhost:5173
```

**Express(Node.js) 예시 — 동적 Origin 검증:**

```javascript
const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') ?? [];

app.use(cors({
  origin: (origin, callback) => {
    // 서버 간 통신(origin 없음)은 허용
    if (!origin) return callback(null, true);

    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`CORS: Origin '${origin}' is not allowed`));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
  allowedHeaders: ['Authorization', 'Content-Type'],
  exposedHeaders: ['X-Total-Count'],
  credentials: true,
  maxAge: 86400,
}));
```

**Nginx 레벨 CORS 처리:**

```nginx
map $http_origin $cors_origin {
    default "";
    "https://app.example.com"   $http_origin;
    "https://admin.example.com" $http_origin;
}

server {
    location /api/ {
        # OPTIONS Preflight 처리
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin'  $cors_origin always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
            add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
            add_header 'Access-Control-Max-Age'       86400 always;
            add_header 'Content-Length' 0;
            return 204;
        }

        add_header 'Access-Control-Allow-Origin'  $cors_origin always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;

        proxy_pass http://backend;
    }
}
```

`map` 블록에 없는 Origin은 `$cors_origin`이 빈 문자열이 되어 헤더가 추가되지 않는다.

### 와일드카드(`*`)의 위험

`Access-Control-Allow-Origin: *`은 모든 출처를 허용한다.
공개 API라면 적절하지만, 인증이 필요한 API에는 절대 사용해서는 안 된다.

와일드카드 사용 시 제약사항:
- `Access-Control-Allow-Credentials: true`와 함께 사용할 수 없다.
- 브라우저가 이 조합을 명시적으로 차단한다.

와일드카드가 위험한 실제 시나리오:
```
1. 사용자가 악성 사이트 https://evil.com 방문
2. 악성 스크립트가 https://api.example.com/user/profile 요청
3. 서버가 Access-Control-Allow-Origin: * 반환
4. 브라우저가 응답을 악성 스크립트에 노출
5. 사용자 데이터 탈취
```

인증이 없어도, 사용자별 데이터를 반환하는 API라면 와일드카드를 피해야 한다.

### credentials 설정 — 쿠키와 인증 정보

브라우저가 쿠키, HTTP 인증, TLS 클라이언트 인증서를 크로스-오리진 요청에 포함하려면 `credentials` 설정이 필요하다.

프론트엔드에서:
```javascript
fetch('https://api.example.com/user', {
  credentials: 'include',  // 쿠키 포함
});
```

서버에서 이를 허용하려면:
```
Access-Control-Allow-Origin: https://app.example.com  // 와일드카드 불가
Access-Control-Allow-Credentials: true
```

credentials를 허용할 때 Origin을 명확히 지정하지 않으면 보안 취약점이 된다.
Origin 검증 로직에 버그가 있으면 CSRF 공격의 표면이 된다.

### 노출 헤더 — `exposedHeaders`

브라우저는 기본적으로 몇 가지 안전한 응답 헤더만 JavaScript에 노출한다.
`Content-Type`, `Content-Length` 등 표준 헤더 외에는 접근이 차단된다.

페이지네이션 정보를 응답 헤더로 전달하는 경우:
```
X-Total-Count: 1000
X-Page-Size: 20
Link: <...>; rel="next"
```

이 헤더를 JavaScript가 읽으려면 서버에서 명시적으로 노출해야 한다:
```
Access-Control-Expose-Headers: X-Total-Count, X-Page-Size, Link
```

Spring Boot:
```java
registry.addMapping("/api/**")
    .exposedHeaders("X-Total-Count", "X-Page-Size", "Link");
```

---

## 3. 보안 헤더 종합

CORS만으로는 브라우저 보안의 일부만 커버된다.
서버는 추가 보안 헤더를 통해 브라우저의 보안 동작을 강화할 수 있다.

### Content-Security-Policy (CSP)

CSP는 브라우저에게 어떤 리소스를 어디서 로드할 수 있는지 알려준다.
XSS 공격의 피해를 크게 줄이는 핵심 보안 헤더다.

공격자가 스크립트를 주입해도 CSP가 허용하지 않은 출처의 스크립트는 실행되지 않는다.

**기본 CSP 구조:**

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
```

각 디렉티브 설명:
- `default-src 'self'`: 기본적으로 같은 출처만 허용
- `script-src`: JavaScript 로드 허용 출처
- `style-src`: CSS 로드 허용 출처
- `img-src`: 이미지 로드 허용 출처
- `connect-src`: fetch, XHR, WebSocket 허용 출처
- `frame-ancestors 'none'`: iframe으로 삽입 차단 (Clickjacking 방어)

**Nginx에서 CSP 설정:**

```nginx
add_header Content-Security-Policy "
  default-src 'self';
  script-src 'self' 'nonce-$request_id';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com wss://ws.example.com;
  frame-ancestors 'none';
  report-uri /csp-report;
" always;
```

`'unsafe-inline'`은 인라인 스크립트를 허용하는데, 보안을 상당히 약화시킨다.
가능하면 nonce나 hash 방식으로 교체하는 것이 낫다.

**CSP Report-Only 모드 — 점진적 도입:**

기존 서비스에 CSP를 바로 적용하면 레거시 코드가 깨질 수 있다.
`Content-Security-Policy-Report-Only`로 위반 사항을 먼저 수집하자:

```nginx
add_header Content-Security-Policy-Report-Only "
  default-src 'self';
  script-src 'self';
  report-uri /csp-report;
" always;
```

위반 보고 엔드포인트:
```javascript
app.post('/csp-report', express.json({ type: 'application/csp-report' }), (req, res) => {
  console.log('CSP violation:', req.body['csp-report']);
  // 로그 집계 서비스로 전송
  res.sendStatus(204);
});
```

Report-Only로 2~4주 운영하면서 위반 패턴을 파악한 후 실제 정책을 수립한다.

### X-Frame-Options

페이지가 iframe 안에 삽입되는 것을 제어한다.
Clickjacking 공격을 막는 기본 방어선이다.

```
X-Frame-Options: DENY          # 어떤 iframe도 차단
X-Frame-Options: SAMEORIGIN    # 같은 출처 iframe만 허용
```

CSP의 `frame-ancestors`와 기능이 겹치는데, 오래된 브라우저를 위해 둘 다 설정하는 게 안전하다:

```nginx
add_header X-Frame-Options "DENY" always;
add_header Content-Security-Policy "frame-ancestors 'none';" always;
```

관리 콘솔처럼 외부에 절대 노출되면 안 되는 페이지는 반드시 `DENY`를 설정한다.

### Referrer-Policy

브라우저는 링크를 클릭하거나 리소스를 요청할 때 `Referer` 헤더를 전송한다.
이 헤더에 URL이 포함되면 사용자의 탐색 경로가 외부에 노출될 수 있다.

예시: `https://app.example.com/user/profile?token=abc123`에서 외부 이미지를 로드하면, 이미지 서버에 `Referer: https://app.example.com/user/profile?token=abc123`이 전송된다.

```
Referrer-Policy: strict-origin-when-cross-origin
```

주요 정책 값:
- `no-referrer`: Referer 헤더 전혀 전송 안 함
- `origin`: 출처만 전송 (`https://example.com`)
- `strict-origin-when-cross-origin`: 같은 출처는 전체 URL, 다른 출처는 origin만, HTTP→HTTPS는 전송 안 함 (권장)
- `unsafe-url`: 항상 전체 URL 전송 (비권장)

대부분의 서비스에 `strict-origin-when-cross-origin`이 적절한 기본값이다.

### Permissions-Policy (Feature-Policy 후속)

카메라, 마이크, 위치정보, 결제 API 등 강력한 브라우저 기능을 제어한다.
사용하지 않는 기능을 비활성화해서 XSS 등 공격 시 피해 범위를 줄인다.

```
Permissions-Policy:
  camera=(),
  microphone=(),
  geolocation=(),
  payment=(),
  usb=()
```

빈 괄호 `()`는 해당 기능을 완전히 비활성화한다.
특정 출처에만 허용하려면:

```
Permissions-Policy: geolocation=(self "https://maps.example.com")
```

### X-Content-Type-Options

브라우저가 응답의 Content-Type을 무시하고 내용을 추측하는 MIME 스니핑을 차단한다.

```
X-Content-Type-Options: nosniff
```

공격자가 이미지 업로드 폼에 JavaScript를 담아 올렸을 때, 브라우저가 Content-Type을 무시하고 스크립트로 실행하는 것을 막는다.

### Strict-Transport-Security (HSTS)

HTTPS를 사용하는 서비스에서 브라우저가 HTTP 요청을 자동으로 HTTPS로 업그레이드하도록 강제한다.

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

- `max-age`: 정책 유효 기간(초). 1년(31536000)이 일반적.
- `includeSubDomains`: 서브도메인에도 적용
- `preload`: 브라우저 HSTS 사전 목록에 등재 신청 가능

주의: `preload`를 설정하고 HSTS Preload 목록에 등재되면 나중에 해제가 매우 어렵다.
HTTPS 운영에 확신이 있을 때만 사용한다.

### 보안 헤더 일괄 적용 — Spring Boot

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .frameOptions(frame -> frame.deny())
                .contentTypeOptions(Customizer.withDefaults())
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                )
                .referrerPolicy(referrer -> referrer
                    .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
                )
                .permissionsPolicy(permissions -> permissions
                    .policy("camera=(), microphone=(), geolocation=()")
                )
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'; script-src 'self'; frame-ancestors 'none'")
                )
            );
        return http.build();
    }
}
```

### 보안 헤더 일괄 적용 — Nginx

```nginx
# 별도 파일로 분리: /etc/nginx/conf.d/security-headers.conf
add_header X-Content-Type-Options      "nosniff" always;
add_header X-Frame-Options             "DENY" always;
add_header Referrer-Policy             "strict-origin-when-cross-origin" always;
add_header Permissions-Policy          "camera=(), microphone=(), geolocation=()" always;
add_header Strict-Transport-Security   "max-age=31536000; includeSubDomains" always;
add_header Content-Security-Policy     "default-src 'self'; script-src 'self'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none';" always;
```

`server` 블록에서 include:
```nginx
server {
    listen 443 ssl;
    include /etc/nginx/conf.d/security-headers.conf;
    ...
}
```

`always` 키워드는 오류 응답(4xx, 5xx)에도 헤더를 추가한다.
이를 빠뜨리면 에러 페이지에서 보안 헤더가 누락된다.

---

## 4. 로컬 개발 CORS 우회

개발 환경에서 프론트엔드(`localhost:3000`)와 백엔드(`localhost:8080`)를 따로 실행하면 CORS가 발생한다.
매번 서버에 개발 Origin을 추가하기보다 프론트엔드 개발 서버의 프록시 기능을 활용한다.

### Vite 프록시 설정

```javascript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        // rewrite: (path) => path.replace(/^\/api/, ''),  // /api prefix 제거 시
      },
    },
  },
});
```

브라우저는 `localhost:3000/api/users`로 요청하고, Vite가 `localhost:8080/api/users`로 프록시한다.
브라우저 관점에서는 Same-Origin 요청이므로 CORS가 발생하지 않는다.

### webpack DevServer 프록시 설정 (CRA)

```javascript
// src/setupProxy.js
const { createProxyMiddleware } = require('http-proxy-middleware');

module.exports = function(app) {
  app.use(
    '/api',
    createProxyMiddleware({
      target: 'http://localhost:8080',
      changeOrigin: true,
    })
  );
};
```

### Next.js 리라이트

```javascript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'http://localhost:8080/api/:path*',
      },
    ];
  },
};
```

### Nginx 개발 환경 프록시

컨테이너 기반 로컬 개발 환경에서는 Nginx를 프론트엔드 리버스 프록시로 두고 백엔드를 프록시하면 프로덕션 구성과 유사한 환경을 만들 수 있다.

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://frontend:3000;
    }

    location /api/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`docker-compose.yml`:
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.dev.conf:/etc/nginx/conf.d/default.conf

  frontend:
    build: ./frontend
    command: npm run dev

  backend:
    build: ./backend
    ports:
      - "8080:8080"
```

### 개발용 임시 방편은 프로덕션에 절대 들어가면 안 된다

간혹 개발 편의를 위해 `Access-Control-Allow-Origin: *`을 프로덕션 코드에 남기는 경우가 있다.
환경변수로 분리하고, CI/CD 파이프라인에서 보안 헤더를 검증하는 단계를 추가하는 게 안전하다.

보안 헤더 검증 스크립트 예시:
```bash
#!/bin/bash
ENDPOINT="https://api.example.com/health"

CORS_HEADER=$(curl -s -I -X OPTIONS \
  -H "Origin: https://evil.com" \
  -H "Access-Control-Request-Method: GET" \
  "$ENDPOINT" | grep -i "access-control-allow-origin")

if echo "$CORS_HEADER" | grep -q "\*"; then
  echo "FAIL: Wildcard CORS detected in production"
  exit 1
fi

echo "PASS: CORS origin is restricted"
```

---

## 정리

CORS와 보안 헤더는 설정 한 줄이 보안 수준을 결정하는 영역이다.

핵심 원칙을 요약하면:

| 항목 | 권장 설정 | 피해야 할 것 |
|------|-----------|--------------|
| CORS Origin | 명시적 허용 목록 | 와일드카드 `*` (인증 API에서) |
| credentials | 필요한 경우만 허용 | 기본으로 켜두기 |
| OPTIONS 처리 | 인증 미들웨어 우회 | 401 반환 |
| CSP | strict-origin-when-cross-origin | `unsafe-eval`, `unsafe-inline` |
| X-Frame-Options | DENY | 설정 누락 |
| HSTS | max-age=31536000 | HTTP 서비스에 설정 |

서버 엔지니어의 역할은 브라우저가 협상할 수 있는 명확한 정책을 제공하는 것이다.
애매하게 열어두거나, 테스트 편의를 위해 느슨하게 설정한 것이 프로덕션까지 이어지는 순간 취약점이 된다.

---

## 참고 자료

- [MDN — Cross-Origin Resource Sharing (CORS)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN — Content-Security-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy)
- [OWASP — CORS Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [Spring Security — CORS](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html)
- [securityheaders.com](https://securityheaders.com) — 보안 헤더 온라인 검증 도구
- [CSP Evaluator](https://csp-evaluator.withgoogle.com) — Google CSP 정책 평가 도구
- [HSTS Preload List](https://hstspreload.org)
