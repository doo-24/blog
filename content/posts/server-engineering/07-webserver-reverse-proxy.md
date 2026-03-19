---
title: "[OS와 네트워크] 7편 — 웹 서버와 리버스 프록시: WAS 앞에 왜 한 계층이 더 필요한가"
date: 2026-03-18T02:00:00+09:00
draft: false
tags: ["Nginx", "Apache", "리버스프록시", "로드밸런싱", "서버"]
series: ["OS와 네트워크"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 1
summary: "Nginx와 Apache의 아키텍처 차이, 이벤트 기반 처리의 원리, 정적 파일 서빙, 압축, 캐시 헤더, 로드밸런싱 알고리즘, SSL 종단, upstream 헬스체크까지 — 리버스 프록시의 모든 것."
---

## 들어가며

웹 애플리케이션을 배포할 때, 대부분의 아키텍처는 이런 구조를 따른다:

```
클라이언트 → Nginx (리버스 프록시) → WAS (Node.js, Spring, Django, ...)
```

WAS가 직접 클라이언트 요청을 받으면 안 될까? 기술적으로는 가능하다. 하지만 실무에서 거의 항상 앞에 웹 서버를 두는 데는 명확한 이유가 있다:

- **정적 파일**을 WAS가 처리하면 애플리케이션 스레드를 낭비한다
- **SSL 종단**을 WAS마다 설정하면 인증서 관리가 복잡해진다
- **느린 클라이언트 대응** — WAS 스레드가 느린 클라이언트에 묶여 고갈될 수 있다
- **로드밸런싱**, **헬스체크**, **요청 제한** 같은 인프라 기능을 애플리케이션에 넣으면 관심사가 섞인다

이번 편에서는 웹 서버의 내부 아키텍처부터 실전 설정까지 다룬다.

---

## 1. Apache vs Nginx — 아키텍처의 근본적 차이

### Apache: 프로세스/스레드 기반

Apache는 1995년에 탄생했다. 당시의 자연스러운 설계는 **연결 하나에 프로세스(또는 스레드) 하나**를 할당하는 것이었다.

**Prefork MPM (Multi-Processing Module)**:
```
Apache (Prefork)
├── 마스터 프로세스
├── 워커 프로세스 1 ← 연결 1개 전담
├── 워커 프로세스 2 ← 연결 1개 전담
├── 워커 프로세스 3 ← 연결 1개 전담
...
└── 워커 프로세스 N ← 연결 1개 전담

특징:
- 프로세스 격리 → 하나가 죽어도 다른 연결에 영향 없음
- mod_php가 프로세스에 임베딩됨 → PHP에서 주로 사용
- 메모리 사용량이 큼 (프로세스당 수십 MB)
- 동시 연결 = 프로세스 수 → 만 단위 연결 처리 어려움
```

**Worker MPM**:
```
Apache (Worker)
├── 마스터 프로세스
├── 자식 프로세스 1
│   ├── 스레드 1 ← 연결 전담
│   ├── 스레드 2 ← 연결 전담
│   └── ...
├── 자식 프로세스 2
│   ├── 스레드 1
│   └── ...
└── ...

특징:
- 프로세스 내 스레드 공유 → 메모리 효율 개선
- 그래도 연결 하나 = 스레드 하나 → C10K 문제
```

**Event MPM** (Apache 2.4+):

Keep-Alive 대기 중인 연결을 전담 스레드에서 분리해서 이벤트 기반으로 처리. Nginx에 대응하기 위한 개선이지만, 근본 아키텍처는 여전히 스레드 기반이다.

### Nginx: 이벤트 기반 비동기

Nginx는 2004년, C10K 문제(동시 1만 연결 처리)를 해결하기 위해 설계되었다. 핵심은 **소수의 프로세스가 수만 개의 연결을 비동기로 처리**하는 것이다.

```
Nginx
├── 마스터 프로세스 (설정 관리, 워커 관리)
├── 워커 프로세스 1 (CPU 코어에 바인딩)
│   └── 이벤트 루프 (epoll)
│       ├── 연결 1 (읽기 준비) → 처리
│       ├── 연결 2 (대기 중)
│       ├── 연결 3 (쓰기 준비) → 처리
│       ├── 연결 4 (대기 중)
│       ├── ...
│       └── 연결 5000 (읽기 준비) → 처리
├── 워커 프로세스 2
│   └── 이벤트 루프 (epoll)
│       ├── 연결 5001 ...
│       └── ...
└── ...

worker_processes = CPU 코어 수 (보통 4~16)
각 워커가 수천~수만 연결을 이벤트 기반으로 처리
```

### 이벤트 루프의 원리

```c
// 개념적인 이벤트 루프 (실제 Nginx 코드를 단순화)
while (true) {
    // epoll_wait: 준비된 이벤트가 있을 때까지 대기
    events = epoll_wait(epoll_fd, event_list, MAX_EVENTS, timeout);

    for (event in events) {
        if (event.type == READ_READY) {
            // 논블로킹으로 데이터 읽기
            data = read(event.fd);  // 즉시 반환
            process_request(data);
        }
        if (event.type == WRITE_READY) {
            // 논블로킹으로 데이터 쓰기
            write(event.fd, response);  // 즉시 반환
        }
    }
}
```

핵심: **절대 블로킹하지 않는다.** 디스크 I/O도 가능한 한 비동기로 처리하고 (AIO, sendfile), 불가피한 블로킹 작업은 별도 스레드 풀에 위임한다.

### 성능 비교

```
10,000 동시 Keep-Alive 연결 기준:

Apache (Prefork):
- 메모리: ~10,000 프로세스 × 30MB = 300GB → 불가능
- 실질 한계: 수백~수천 동시 연결

Apache (Worker):
- 메모리: 개선되지만 여전히 스레드당 수 MB
- 실질 한계: 수천~1만 동시 연결

Nginx:
- 메모리: 4 워커 × ~50MB + 연결당 ~수 KB = 수십 MB
- 실질 한계: 수만~수십만 동시 연결
- 정적 파일 서빙에서 Apache 대비 2~10배 빠름
```

**Apache가 여전히 유용한 경우:**
- `.htaccess` 파일 기반 디렉토리별 설정이 필요할 때
- mod_php 같은 모듈이 프로세스에 임베딩되어야 할 때
- 공유 호스팅 환경

---

## 2. 정적 파일 서빙

### 왜 WAS로 정적 파일을 서빙하면 안 되는가

```
WAS로 정적 파일 서빙:
1. 클라이언트 요청 → WAS 스레드 할당
2. 파일 읽기 (디스크 I/O)
3. 응답 전송 (네트워크 I/O)
4. 느린 클라이언트면? → 스레드가 전송 완료까지 점유됨!

Nginx로 정적 파일 서빙:
1. 클라이언트 요청 → 이벤트 루프에서 처리
2. sendfile()로 커널에서 직접 전송 (zero-copy)
3. 느린 클라이언트? → 이벤트 루프가 다른 연결 처리, 나중에 다시 돌아옴
```

### Nginx 정적 파일 설정

```nginx
server {
    listen 80;
    server_name example.com;

    # 정적 파일 직접 서빙
    location /static/ {
        root /var/www;
        # 실제 파일 경로: /var/www/static/...

        sendfile on;           # 커널 레벨 zero-copy
        tcp_nopush on;         # TCP_CORK — 헤더+데이터 한 패킷으로
        tcp_nodelay on;        # Keep-Alive 연결에서 작은 패킷 즉시 전송

        # 클라이언트 캐시 (1년 — 파일명에 해시가 있는 경우)
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 나머지는 WAS로 프록시
    location / {
        proxy_pass http://localhost:3000;
    }
}
```

### try_files — 우아한 라우팅

SPA(Single Page Application)나 프론트엔드 프레임워크에서 필수적인 설정:

```nginx
location / {
    root /var/www/app;
    try_files $uri $uri/ /index.html;
    # 1. 요청된 파일이 있으면 서빙
    # 2. 디렉토리면 index 파일 찾기
    # 3. 둘 다 없으면 index.html (SPA 라우팅)
}
```

---

## 3. 압축 — gzip과 Brotli

### gzip 압축

텍스트 기반 리소스(HTML, CSS, JS, JSON)는 gzip으로 60~80% 크기를 줄일 수 있다.

```nginx
http {
    gzip on;
    gzip_vary on;              # Vary: Accept-Encoding 헤더 추가
    gzip_proxied any;          # 프록시 응답도 압축
    gzip_comp_level 6;         # 압축 레벨 (1=빠름, 9=최대 압축, 6=적정선)
    gzip_min_length 256;       # 256바이트 미만은 압축 안 함 (오히려 커질 수 있음)
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        image/svg+xml;
    # image/jpeg, image/png 등은 이미 압축되어 있으므로 제외
}
```

**압축 레벨별 성능:**
```
Level  압축률    CPU 시간
1      ~60%     1x (가장 빠름)
4      ~70%     2x
6      ~75%     4x (기본값, 적정선)
9      ~78%     10x (큰 차이 없이 CPU만 소모)
```

### Brotli 압축

Google이 개발한 Brotli는 gzip 대비 15~25% 더 높은 압축률을 제공한다. 모든 주요 브라우저가 지원한다.

```nginx
# ngx_brotli 모듈 필요
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/javascript application/json image/svg+xml;
```

**gzip vs Brotli:**
```
100KB JavaScript 파일:
gzip level 6:   → 28KB  (72% 압축)
brotli level 6: → 23KB  (77% 압축)

실전에서는 두 가지를 동시에 설정:
- Brotli 지원 브라우저 → Brotli
- 미지원 → gzip 폴백
```

### 사전 압축 (Pre-compression)

빌드 시점에 압축된 파일을 미리 만들어두면 런타임 CPU 비용이 사라진다:

```nginx
# Nginx가 .gz, .br 파일을 자동으로 서빙
location /static/ {
    gzip_static on;     # .gz 파일이 있으면 그대로 서빙
    brotli_static on;   # .br 파일이 있으면 그대로 서빙
}
```

```bash
# 빌드 스크립트에서 사전 압축
find /var/www/static -type f \( -name '*.js' -o -name '*.css' \) \
    -exec gzip -9 -k {} \; \
    -exec brotli -9 -k {} \;

# 결과:
# app.js      (원본)
# app.js.gz   (gzip 사전 압축)
# app.js.br   (brotli 사전 압축)
```

---

## 4. 캐시 헤더 — 브라우저 캐시 제어

### Cache-Control 전략

```
리소스 유형별 권장 전략:

HTML (매번 최신 확인):
Cache-Control: no-cache
→ 캐시하되, 사용 전 서버에 검증(ETag/Last-Modified)

해시된 정적 파일 (app.a1b2c3.js):
Cache-Control: public, max-age=31536000, immutable
→ 1년 캐시, 변경 시 파일명이 바뀌므로 안전

API 응답 (캐시 금지):
Cache-Control: no-store
→ 캐시 자체를 하지 않음 (개인 정보 등)

이미지/폰트 (적당히):
Cache-Control: public, max-age=86400
→ 1일 캐시, 이후 재검증
```

### ETag와 조건부 요청

```
첫 번째 요청:
GET /style.css HTTP/1.1

응답:
HTTP/1.1 200 OK
ETag: "a1b2c3d4"
Cache-Control: no-cache
(본문 전체 전송)

두 번째 요청 (캐시 검증):
GET /style.css HTTP/1.1
If-None-Match: "a1b2c3d4"

파일이 안 바뀌었으면:
HTTP/1.1 304 Not Modified
(본문 없음 → 대역폭 절약)

파일이 바뀌었으면:
HTTP/1.1 200 OK
ETag: "e5f6g7h8"
(새 본문 전송)
```

```nginx
server {
    # ETag 자동 생성 (기본 활성화)
    etag on;

    # Last-Modified 헤더도 자동 (파일 수정 시간 기반)

    location /static/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    location / {
        add_header Cache-Control "no-cache";
    }
}
```

---

## 5. 리버스 프록시와 로드밸런싱

### 리버스 프록시 기본 설정

```nginx
upstream backend {
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;

        # 원본 클라이언트 정보 전달
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 타임아웃 설정
        proxy_connect_timeout 5s;    # 백엔드 연결 시도 시간
        proxy_read_timeout 60s;      # 백엔드 응답 대기 시간
        proxy_send_timeout 30s;      # 백엔드로 요청 전송 시간
    }
}
```

### 로드밸런싱 알고리즘

**Round Robin (기본)**:
```nginx
upstream backend {
    server 10.0.1.10:3000;    # 33%
    server 10.0.1.11:3000;    # 33%
    server 10.0.1.12:3000;    # 33%
}
# 순서대로 돌아가며 분배
# 장점: 단순, 균등 분배
# 단점: 서버 처리 능력 차이 무시
```

**Weighted Round Robin**:
```nginx
upstream backend {
    server 10.0.1.10:3000 weight=5;    # 50%
    server 10.0.1.11:3000 weight=3;    # 30%
    server 10.0.1.12:3000 weight=2;    # 20%
}
# 서버 성능에 비례하게 가중치 설정
```

**Least Connections**:
```nginx
upstream backend {
    least_conn;
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;
}
# 현재 활성 연결이 가장 적은 서버로 분배
# 장점: 요청 처리 시간이 불균일할 때 효과적
# 예: 빠른 API + 느린 보고서 생성이 섞인 서비스
```

**IP Hash**:
```nginx
upstream backend {
    ip_hash;
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;
}
# 클라이언트 IP의 해시로 서버 결정 → 같은 클라이언트는 항상 같은 서버
# 용도: 세션 고정(session affinity)이 필요한 경우
# 주의: 서버 추가/제거 시 해시가 바뀌어 세션 재배치 발생
```

**Consistent Hashing (hash 지시문)**:
```nginx
upstream backend {
    hash $request_uri consistent;
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
    server 10.0.1.12:3000;
}
# URI 기반 해시 → 같은 URL은 항상 같은 서버 → 캐시 효율 극대화
# consistent 옵션: 서버 추가/제거 시 최소한의 재배치 (ketama 알고리즘)
```

**알고리즘 선택 가이드:**
```
상태 없는 API 서버          → Round Robin / Least Connections
세션 기반 웹 애플리케이션    → IP Hash (또는 외부 세션 스토어 + Round Robin)
캐시 프록시                 → Hash (consistent)
서버 스펙이 다름            → Weighted Round Robin
```

---

## 6. Upstream 헬스체크

### Passive 헬스체크 (기본)

Nginx OSS(오픈소스)는 패시브 헬스체크를 지원한다. 실제 요청 실패를 감지해서 서버를 제외한다:

```nginx
upstream backend {
    server 10.0.1.10:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:3000 max_fails=3 fail_timeout=30s;
}
# max_fails=3: 30초 내 3번 실패하면 비활성화
# fail_timeout=30s: 비활성화 후 30초 뒤 다시 시도

# 무엇이 "실패"인지 정의
location / {
    proxy_pass http://backend;
    proxy_next_upstream error timeout http_500 http_502 http_503;
    proxy_next_upstream_tries 2;     # 최대 2번 다른 서버로 재시도
    proxy_next_upstream_timeout 10s; # 재시도 전체 시간 제한
}
```

**패시브 헬스체크의 한계:**
- 실제 사용자 요청이 실패해야 감지 → 일부 사용자는 에러를 경험
- 트래픽이 없으면 장애를 감지할 수 없음

### Active 헬스체크

Nginx Plus(상용) 또는 OpenResty, Tengine 같은 포크에서 지원한다. 주기적으로 헬스체크 요청을 보내서 사전에 장애를 감지한다:

```nginx
# Nginx Plus 예시
upstream backend {
    zone backend 64k;   # 공유 메모리 (워커 간 상태 공유)
    server 10.0.1.10:3000;
    server 10.0.1.11:3000;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=5s fails=3 passes=2 uri=/health;
        # 5초마다 /health 요청
        # 3번 연속 실패 → 비활성화
        # 2번 연속 성공 → 재활성화
    }
}
```

**무료 대안 — HAProxy의 액티브 헬스체크:**
```
backend app_servers
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    server app1 10.0.1.10:3000 check inter 5s fall 3 rise 2
    server app2 10.0.1.11:3000 check inter 5s fall 3 rise 2
```

### 헬스체크 엔드포인트 설계

```json
// GET /health
// 단순 버전
{ "status": "ok" }

// 상세 버전 (내부용)
{
    "status": "ok",
    "checks": {
        "database": { "status": "ok", "latency_ms": 5 },
        "redis": { "status": "ok", "latency_ms": 1 },
        "disk": { "status": "ok", "free_gb": 45.2 }
    },
    "uptime_seconds": 86400
}
```

로드밸런서 헬스체크에는 단순 버전을, 모니터링 시스템에는 상세 버전을 사용하는 것이 일반적이다.

데이터베이스 연결 확인을 헬스체크에 넣을 때는, DB 장애 시 모든 서버가 동시에 비활성화되지 않도록 주의해야 한다.

---

## 7. SSL/TLS 종단

### SSL 종단이란

SSL 종단(SSL Termination)은 리버스 프록시에서 TLS를 해제하고, 백엔드에는 평문 HTTP로 전달하는 패턴이다:

```
클라이언트 ──HTTPS──→ Nginx ──HTTP──→ WAS
                       ↑
                  여기서 TLS 해제
                  (SSL 종단)

장점:
- 인증서 관리를 한 곳에서
- WAS는 TLS 처리 부담 없음
- 내부 네트워크 통신 단순화
```

### Nginx SSL 설정

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # 인증서
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # 프로토콜 (TLS 1.2, 1.3만)
    ssl_protocols TLSv1.2 TLSv1.3;

    # 암호화 스위트 (서버 우선순위 사용)
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

    # SSL 세션 캐시 (핸드셰이크 비용 절감)
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;   # PFS(Perfect Forward Secrecy) 보장

    # OCSP Stapling (인증서 유효성 확인 최적화)
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;

    # HSTS (브라우저가 항상 HTTPS로 접속하도록)
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";

    location / {
        proxy_pass http://backend;
    }
}

# HTTP → HTTPS 리다이렉트
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### SSL 세션 재사용

TLS 핸드셰이크는 비싸다 (RSA 키 교환 시 수 밀리초). 세션을 캐시하면 재연결 시 비용을 줄인다:

```
첫 연결:  Full Handshake (2 RTT, 비쌈)
재연결:   Session Resumption (1 RTT, 빠름)
TLS 1.3:  0-RTT Resumption (0 RTT, 가장 빠름)

ssl_session_cache shared:SSL:10m;
→ 10MB 공유 메모리 ≈ 약 40,000 세션 캐시
→ 모든 워커 프로세스가 공유
```

---

## 8. 실전 Nginx 설정 패턴

### Rate Limiting — 요청 속도 제한

```nginx
http {
    # 존 정의: 클라이언트 IP별 초당 10개 요청
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            # burst=20: 순간적으로 20개까지 허용 (버킷에 쌓임)
            # nodelay: 버스트 요청을 지연 없이 즉시 처리
            # 초과 시 503 반환

            proxy_pass http://backend;
        }
    }
}
```

### Connection Limiting — 동시 연결 제한

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn:10m;

    server {
        location /download/ {
            limit_conn conn 5;       # IP당 동시 5개 연결
            limit_rate 1m;           # 연결당 1MB/s 속도 제한
        }
    }
}
```

### 버퍼링 — 느린 클라이언트 보호

```nginx
location / {
    proxy_pass http://backend;

    # 백엔드 응답을 Nginx가 버퍼링
    proxy_buffering on;
    proxy_buffer_size 4k;          # 첫 번째 응답 (헤더)
    proxy_buffers 8 16k;           # 본문 버퍼 (8 × 16KB)
    proxy_busy_buffers_size 32k;   # 클라이언트 전송 중 버퍼

    # 큰 응답은 임시 파일로
    proxy_max_temp_file_size 1024m;
}
```

```
버퍼링의 효과:

[백엔드] ─── 빠르게 응답 (100ms) ───→ [Nginx 버퍼] ─── 천천히 전송 ───→ [느린 클라이언트]
         백엔드 스레드 즉시 해제                       Nginx가 전담

버퍼링 없으면:
[백엔드] ─── 느린 클라이언트 속도에 맞춰 ─── 직접 전송 ───→ [느린 클라이언트]
         백엔드 스레드가 수 초간 점유됨!
```

이것이 WAS 앞에 Nginx를 두는 가장 중요한 이유 중 하나다. 느린 클라이언트(모바일, 저대역폭)가 WAS 스레드를 고갈시키는 것을 방지한다.

### 웹소켓 프록시

```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # 웹소켓은 장시간 유지되므로 타임아웃 늘림
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

### Graceful 재시작

```bash
# 설정 파일 문법 검사
$ nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

# Graceful reload — 기존 연결 유지하면서 새 설정 적용
$ nginx -s reload
# 내부 동작:
# 1. 마스터가 새 설정으로 새 워커 생성
# 2. 기존 워커는 현재 요청 처리 완료 후 종료
# 3. 다운타임 없음!

# 워커 프로세스 상태 확인
$ ps aux | grep nginx
root     1234  0.0  master process nginx
www-data 5678  0.1  worker process      (새 워커)
www-data 5679  0.1  worker process      (새 워커)
www-data 4321  0.0  worker process is shutting down  (기존 워커, 완료 대기)
```

---

## 마치며

웹 서버와 리버스 프록시는 서버 아키텍처의 첫 번째 방어선이다.

- **Nginx의 이벤트 기반 아키텍처**가 소수의 프로세스로 수만 연결을 처리할 수 있는 이유를 이해해야 한다
- **정적 파일은 Nginx가**, **동적 요청은 WAS가** — 이 분리가 리소스 효율의 기본이다
- **gzip/Brotli 압축**과 **캐시 헤더**는 네트워크 비용을 60~80% 줄인다
- **로드밸런싱 알고리즘**은 서비스 특성에 맞게 선택해야 한다 — 상태 없는 API는 Round Robin, 캐시 프록시는 Consistent Hash
- **버퍼링**이 느린 클라이언트로부터 WAS를 보호하는 핵심 메커니즘이다
- **SSL 종단**을 프록시에서 처리하면 인증서 관리와 성능 최적화가 한 곳에서 이루어진다

다음 편에서는 TLS/HTTPS를 더 깊이 다룬다 — 암호화의 원리, 인증서 체인, mTLS, 그리고 TLS 1.3의 핸드셰이크를 단계별로 분해한다.

---

## 참고 자료

- Nginx Official Documentation — [https://nginx.org/en/docs/](https://nginx.org/en/docs/)
- *High Performance Browser Networking* — Ilya Grigorik (O'Reilly) — [https://hpbn.co/](https://hpbn.co/)
- Apache MPM Documentation — [https://httpd.apache.org/docs/2.4/mpm.html](https://httpd.apache.org/docs/2.4/mpm.html)
- "The C10K Problem" — Dan Kegel — [http://www.kegel.com/c10k.html](http://www.kegel.com/c10k.html)
- Mozilla MDN: HTTP Caching — [https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- Let's Encrypt Documentation — [https://letsencrypt.org/docs/](https://letsencrypt.org/docs/)
