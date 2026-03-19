---
title: "[OS와 네트워크] 8편 — TLS/HTTPS 심화: 암호화 통신의 원리"
date: 2026-03-18T01:00:00+09:00
draft: false
tags: ["TLS", "HTTPS", "암호화", "인증서", "보안", "서버"]
series: ["OS와 네트워크"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 1
summary: "대칭/비대칭 암호화의 원리, TLS 1.2와 1.3 핸드셰이크 단계별 분해, 인증서 체인, CA, mTLS, HSTS, Certificate Pinning, 그리고 Let's Encrypt 자동화까지."
---

## 들어가며

HTTPS는 이제 선택이 아니라 기본이다. 브라우저는 HTTP 사이트에 "주의 요함"을 표시하고, 검색 엔진은 HTTPS 사이트에 가산점을 준다. HTTP/2와 HTTP/3는 사실상 TLS를 전제한다.

하지만 "인증서 설치하면 끝"이라고 생각하면 큰 그림을 놓친다. TLS가 정확히 무엇을 보장하는지, 핸드셰이크에서 무슨 일이 벌어지는지, 인증서 체인은 어떻게 신뢰를 구축하는지 — 이것들을 이해해야 장애 상황에서 문제를 진단하고, 보안 요구사항에 맞는 설정을 할 수 있다.

---

## 1. 암호화의 기초

### TLS가 보장하는 세 가지

```
1. 기밀성 (Confidentiality)
   → 통신 내용을 제3자가 볼 수 없다
   → 대칭 암호화 (AES-GCM 등)

2. 무결성 (Integrity)
   → 통신 내용이 변조되지 않았음을 보장
   → MAC (Message Authentication Code)

3. 인증 (Authentication)
   → 통신 상대가 진짜 그 서버인지 확인
   → 인증서 + 디지털 서명
```

### 대칭 암호화

하나의 키로 암호화와 복호화를 모두 수행한다. 빠르다.

```
평문: "Hello, World!"
  │
  ▼  암호화 (키: 0xABCD...)
암호문: 0x7F3A...
  │
  ▼  복호화 (같은 키: 0xABCD...)
평문: "Hello, World!"
```

대표 알고리즘:
- **AES-128-GCM**: 128비트 키, 가장 널리 사용. GCM 모드는 암호화 + 무결성을 동시에 제공
- **AES-256-GCM**: 256비트 키, 더 높은 보안 (양자 내성 관점에서 선호)
- **ChaCha20-Poly1305**: 모바일/임베디드에서 AES 하드웨어 가속이 없을 때 빠름

**문제점**: 키를 어떻게 안전하게 공유할 것인가? 네트워크로 키를 보내면 도청당한다.

### 비대칭 암호화 (공개키 암호화)

공개키와 개인키 쌍을 사용한다. 공개키로 암호화하면 개인키로만 복호화할 수 있다.

```
서버의 키 쌍:
┌─────────────┐    ┌─────────────┐
│  공개키 (PK) │    │  개인키 (SK) │
│  (누구에게나  │    │  (서버만 보유)│
│   공개)      │    │              │
└──────┬──────┘    └──────┬──────┘
       │                   │
       ▼                   ▼
  공개키로 암호화     →    개인키로 복호화
  개인키로 서명       →    공개키로 검증
```

대표 알고리즘:
- **RSA**: 가장 오래된 방식. 2048비트 이상 권장. 키 교환과 서명 모두 가능
- **ECDSA**: 타원 곡선 기반. RSA 대비 짧은 키로 동일 보안 수준 (256비트 ≈ RSA 3072비트)
- **Ed25519**: 빠르고 안전한 서명 알고리즘. 점점 더 많이 채택

**문제점**: 비대칭 암호화는 대칭 암호화보다 100~1000배 느리다. 모든 데이터를 비대칭으로 암호화하는 것은 비현실적이다.

### 하이브리드 방식 — TLS의 해법

TLS는 두 가지를 결합한다:
1. **비대칭 암호화**로 대칭키를 안전하게 교환 (핸드셰이크)
2. **대칭 암호화**로 실제 데이터를 암호화 (데이터 전송)

```
핸드셰이크 (느리지만 안전한 키 교환):
클라이언트 ←──비대칭 암호화──→ 서버
              │
              ▼
         대칭키 합의 (세션 키)

데이터 전송 (빠른 암호화):
클라이언트 ←──대칭 암호화──→ 서버
           (세션 키 사용)
```

---

## 2. 키 교환 — Diffie-Hellman

### 왜 RSA 키 교환은 사라지는가

전통적 RSA 키 교환:
```
1. 클라이언트가 랜덤 premaster secret 생성
2. 서버의 공개키로 암호화해서 전송
3. 서버가 개인키로 복호화
4. 양쪽이 premaster secret으로 세션 키 생성
```

문제: 서버의 개인키가 유출되면 **과거의 모든 통신**을 복호화할 수 있다. 공격자가 암호화된 트래픽을 저장해뒀다가, 나중에 키가 유출되면 전부 해독하는 시나리오다.

### ECDHE — Perfect Forward Secrecy

ECDHE(Elliptic Curve Diffie-Hellman Ephemeral)는 매 연결마다 **임시(ephemeral) 키 쌍**을 생성한다:

```
클라이언트                              서버
    │                                    │
    │  임시 키 쌍 생성                    │  임시 키 쌍 생성
    │  (a, g^a)                         │  (b, g^b)
    │                                    │
    │  ── 클라이언트 공개값 (g^a) ──→    │
    │  ←── 서버 공개값 (g^b) ─────      │
    │                                    │
    │  공유 비밀 = (g^b)^a              │  공유 비밀 = (g^a)^b
    │           = g^(ab)                │           = g^(ab)
    │                                    │
    │        같은 값! → 세션 키 생성      │
    │                                    │

도청자는 g^a와 g^b만 볼 수 있고,
g^(ab)를 알아내는 것은 이산 로그 문제로 현실적 불가능
```

**Perfect Forward Secrecy (PFS)**: 임시 키는 핸드셰이크 후 폐기된다. 서버 개인키가 나중에 유출되어도 과거 세션의 키를 복구할 수 없다.

```
RSA 키 교환:     서버 개인키 유출 → 과거 모든 세션 해독 가능 ✗
ECDHE 키 교환:   서버 개인키 유출 → 과거 세션 해독 불가 ✓
                 (임시 키는 이미 폐기됨)
```

TLS 1.3에서는 RSA 키 교환이 완전히 제거되었다. ECDHE만 허용한다.

---

## 3. TLS 1.2 핸드셰이크 — 단계별 분해

```
클라이언트                                    서버
    │                                          │
    │  1. ClientHello                         │
    │  - 지원 TLS 버전                         │
    │  - 지원 암호 스위트 목록                   │
    │  - 클라이언트 랜덤 (32바이트)              │
    │  - 지원 확장 (SNI, ALPN 등)              │
    │─────────────────────────────────────────→│
    │                                          │
    │  2. ServerHello                         │
    │  - 선택된 TLS 버전                       │
    │  - 선택된 암호 스위트                     │
    │  - 서버 랜덤 (32바이트)                   │
    │←─────────────────────────────────────────│
    │                                          │
    │  3. Certificate                         │
    │  - 서버 인증서 (+ 체인)                   │
    │←─────────────────────────────────────────│
    │                                          │
    │  4. ServerKeyExchange (ECDHE 사용 시)    │
    │  - 서버의 ECDHE 공개값                    │
    │  - 서버의 디지털 서명                     │
    │←─────────────────────────────────────────│
    │                                          │
    │  5. ServerHelloDone                     │
    │←─────────────────────────────────────────│
    │                                          │
    │  [클라이언트: 인증서 검증]                 │
    │  [클라이언트: ECDHE 공유 비밀 계산]        │
    │                                          │
    │  6. ClientKeyExchange                   │
    │  - 클라이언트의 ECDHE 공개값              │
    │─────────────────────────────────────────→│
    │                                          │
    │  7. ChangeCipherSpec                    │
    │  "이제부터 암호화합니다"                    │
    │─────────────────────────────────────────→│
    │                                          │
    │  8. Finished (암호화됨)                  │
    │  - 핸드셰이크 검증 해시                    │
    │─────────────────────────────────────────→│
    │                                          │
    │  9. ChangeCipherSpec                    │
    │←─────────────────────────────────────────│
    │                                          │
    │  10. Finished (암호화됨)                 │
    │←─────────────────────────────────────────│
    │                                          │
    │  ═══ 2 RTT 후 데이터 전송 시작 ═══       │
```

**TLS 1.2: 핸드셰이크에 2 RTT 필요.** RTT가 100ms이면 첫 데이터 전송까지 200ms가 추가된다.

### SNI (Server Name Indication)

하나의 IP에 여러 도메인을 호스팅하는 경우, 서버는 어떤 인증서를 보여줘야 하는지 알아야 한다. SNI는 ClientHello에 요청 도메인을 포함한다:

```
ClientHello:
  ...
  extension: server_name = "api.example.com"
  ...

→ 서버가 api.example.com의 인증서를 선택
```

**주의**: SNI는 암호화되지 않는다 (TLS 1.2). 네트워크 관찰자가 어떤 사이트에 접속하는지 알 수 있다. TLS 1.3의 ECH(Encrypted Client Hello)가 이를 해결한다.

### ALPN (Application-Layer Protocol Negotiation)

TLS 핸드셰이크 중에 애플리케이션 프로토콜을 합의한다:

```
ClientHello:
  extension: ALPN = ["h2", "http/1.1"]

ServerHello:
  extension: ALPN = "h2"

→ 별도의 라운드트립 없이 HTTP/2 사용 합의
```

---

## 4. TLS 1.3 핸드셰이크 — 더 빠르고, 더 안전하게

### TLS 1.3의 핵심 변화

```
제거된 것:
- RSA 키 교환 (PFS 미지원이므로)
- CBC 모드 암호 (패딩 오라클 공격에 취약)
- RC4, 3DES, MD5, SHA-1
- 압축 (CRIME 공격에 취약)
- 재협상 (renegotiation)

남은 암호 스위트 (5개만):
- TLS_AES_128_GCM_SHA256
- TLS_AES_256_GCM_SHA384
- TLS_CHACHA20_POLY1305_SHA256
- TLS_AES_128_CCM_SHA256
- TLS_AES_128_CCM_8_SHA256

키 교환: ECDHE만 (x25519, secp256r1 등)
```

### 1-RTT 핸드셰이크

TLS 1.3은 키 교환과 핸드셰이크를 병합해서 1 RTT만에 완료한다:

```
클라이언트                                    서버
    │                                          │
    │  1. ClientHello                         │
    │  - 지원 키 공유 그룹 (x25519, ...)       │
    │  - 키 공유: 클라이언트 ECDHE 공개값       │  ← TLS 1.2와 다른 점!
    │  - 지원 암호 스위트                       │    키 교환을 첫 메시지에 포함
    │─────────────────────────────────────────→│
    │                                          │
    │  2. ServerHello                         │
    │  - 선택된 암호 스위트                     │
    │  - 키 공유: 서버 ECDHE 공개값            │
    │←─────────────────────────────────────────│
    │                                          │
    │  [이 시점에 양쪽 모두 세션 키 생성 가능]    │
    │                                          │
    │  3. {EncryptedExtensions}               │
    │  4. {Certificate}                       │  ← 여기서부터
    │  5. {CertificateVerify}                 │    전부 암호화됨!
    │  6. {Finished}                          │
    │←─────────────────────────────────────────│
    │                                          │
    │  7. {Finished}                          │
    │─────────────────────────────────────────→│
    │                                          │
    │  ═══ 1 RTT 후 데이터 전송 시작 ═══       │
```

핵심 차이: 클라이언트가 ClientHello에 ECDHE 공개값을 **미리** 포함한다. 서버가 응답하면 즉시 세션 키를 생성할 수 있다.

### 0-RTT Resumption (Early Data)

이전에 연결한 적이 있는 서버에 재연결할 때, 첫 번째 메시지에 데이터를 함께 보낼 수 있다:

```
클라이언트                                    서버
    │                                          │
    │  ClientHello + Early Data (0-RTT)       │
    │  "이전 세션 키로 암호화된 HTTP 요청"       │
    │─────────────────────────────────────────→│
    │                                          │  즉시 처리 시작!
    │  ServerHello + Finished                 │
    │←─────────────────────────────────────────│
    │                                          │
    │  ═══ 0 RTT — 즉시 데이터 전송! ═══       │
```

**0-RTT의 보안 주의사항:**
- **재전송 공격(Replay Attack)** 취약: 공격자가 0-RTT 데이터를 캡처해서 재전송할 수 있음
- 따라서 0-RTT는 **멱등(idempotent) 요청에만** 사용해야 함 (GET은 OK, POST는 위험)
- 서버 측에서 재전송 방지 메커니즘 필요

```nginx
# Nginx에서 0-RTT 설정
ssl_early_data on;

# 백엔드에 0-RTT 여부 전달 (재전송 공격 방어용)
proxy_set_header Early-Data $ssl_early_data;

# 백엔드에서: Early-Data: 1이면 멱등이 아닌 요청 거부
```

---

## 5. 인증서 체인과 CA

### 인증서의 구조

X.509 인증서에는 다음 정보가 들어있다:

```
┌─────────────────────────────────────┐
│ 인증서 (X.509 v3)                    │
├─────────────────────────────────────┤
│ Subject: CN=api.example.com         │  ← 이 인증서의 주체
│ Issuer: CN=Let's Encrypt R3        │  ← 발급한 CA
│ Validity:                           │
│   Not Before: 2026-01-01            │
│   Not After:  2026-03-31            │
│ Public Key: (RSA 2048 또는 ECDSA)   │  ← 서버의 공개키
│ Subject Alternative Names:          │
│   DNS: api.example.com              │
│   DNS: *.example.com                │  ← 와일드카드
│ Signature Algorithm: SHA256withRSA  │
│ Signature: (CA의 개인키로 서명)       │  ← CA가 보증
└─────────────────────────────────────┘
```

### 인증서 체인 — 신뢰의 사슬

```
루트 CA (Root CA)
│ 자체 서명 (self-signed)
│ OS/브라우저에 미리 설치됨 (Trust Store)
│
├── 서명 ──→ 중간 CA (Intermediate CA)
│            │
│            ├── 서명 ──→ 서버 인증서 (Leaf Certificate)
│            │            api.example.com
│            │
│            └── 서명 ──→ 다른 서버 인증서
│                         shop.example.com
│
└── 서명 ──→ 다른 중간 CA
```

검증 과정:
```
1. 서버가 [서버 인증서 + 중간 CA 인증서]를 전송
2. 클라이언트가 체인을 검증:
   a. 서버 인증서의 서명을 중간 CA 공개키로 검증 ✓
   b. 중간 CA 인증서의 서명을 루트 CA 공개키로 검증 ✓
   c. 루트 CA가 Trust Store에 있는지 확인 ✓
   d. 각 인증서의 유효기간 확인 ✓
   e. 인증서의 도메인(SAN)이 요청 도메인과 일치하는지 확인 ✓
3. 모든 검증 통과 → 신뢰!
```

**흔한 실수**: 서버에 중간 CA 인증서를 포함하지 않는 것. 브라우저는 캐시된 중간 인증서로 해결할 수 있지만, API 클라이언트나 curl은 실패한다.

```bash
# 인증서 체인 확인
$ openssl s_client -connect api.example.com:443 -showcerts
# Certificate chain 에서 중간 인증서가 포함되어 있는지 확인

# 체인 검증
$ openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt \
    -untrusted intermediate.pem server.pem
server.pem: OK
```

### 인증서 투명성 (Certificate Transparency)

악의적 CA가 가짜 인증서를 발급하는 것을 방지하기 위한 시스템. 모든 발급된 인증서가 공개 로그에 기록된다:

```
CA가 인증서 발급 → CT 로그에 등록 → SCT(Signed Certificate Timestamp) 발급
                                      ↓
                              인증서에 SCT 포함
                                      ↓
                              브라우저가 SCT 검증
                              (CT 로그에 기록된 인증서인지)
```

```bash
# 특정 도메인의 CT 로그 조회
# https://crt.sh/?q=example.com 에서 확인 가능
```

---

## 6. mTLS — 상호 인증

### 일반 TLS vs mTLS

```
일반 TLS:
클라이언트 → "인증서 보여줘" → 서버 인증서 검증 ✓
(클라이언트는 인증서 불필요)

mTLS (Mutual TLS):
클라이언트 → "인증서 보여줘" → 서버 인증서 검증 ✓
서버 → "너도 인증서 보여줘" → 클라이언트 인증서 검증 ✓
```

mTLS는 **서비스 간 통신**(service-to-service)에서 주로 사용된다. API 키 대신 인증서로 상호 인증한다.

```
mTLS 핸드셰이크 추가 단계:

서버 → CertificateRequest (클라이언트 인증서 요청)
클라이언트 → Certificate (클라이언트 인증서 전송)
클라이언트 → CertificateVerify (개인키로 서명하여 소유 증명)
```

### Nginx에서 mTLS 설정

```nginx
server {
    listen 443 ssl;

    # 서버 인증서 (일반 TLS와 동일)
    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    # 클라이언트 인증서 검증 (mTLS)
    ssl_client_certificate /etc/nginx/certs/ca.crt;  # 클라이언트 인증서를 발급한 CA
    ssl_verify_client on;          # 필수 (optional로 하면 선택적)
    ssl_verify_depth 2;            # 인증서 체인 깊이

    location / {
        # 클라이언트 인증서 정보를 백엔드에 전달
        proxy_set_header X-Client-Cert-DN $ssl_client_s_dn;
        proxy_set_header X-Client-Cert-Serial $ssl_client_serial;
        proxy_pass http://backend;
    }
}
```

### mTLS의 실제 사용 사례

```
마이크로서비스 간 통신:
┌─────────┐  mTLS   ┌─────────┐  mTLS   ┌─────────┐
│ Service │←──────→│ Service │←──────→│ Service │
│    A    │        │    B    │        │    C    │
└─────────┘        └─────────┘        └─────────┘
          모든 서비스가 서로의 인증서를 검증

Service Mesh (Istio, Linkerd):
- 사이드카 프록시가 mTLS를 자동으로 처리
- 애플리케이션 코드 변경 없이 mTLS 적용
- 인증서 자동 발급/갱신 (SPIFFE)
```

---

## 7. HSTS — 강제 HTTPS

### HSTS (HTTP Strict Transport Security)

HSTS는 브라우저에게 "이 사이트는 항상 HTTPS로만 접속하라"고 지시한다:

```
첫 HTTPS 방문 시 서버 응답:
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload

이후:
- 사용자가 http://example.com 입력 → 브라우저가 자동으로 https://로 변환
- 인증서 오류 시 → 우회 불가 (경고를 무시할 수 없음)
- max-age 기간 동안 유지 (63072000초 = 2년)
```

### HSTS가 없으면 — SSL Stripping 공격

```
HSTS 없을 때:
사용자 → http://example.com → 서버 (301 → https)

공격자 (중간자):
사용자 → http://example.com → [공격자] → https://example.com → 서버
         ↑ 평문 HTTP!          ↓
         사용자에게는 HTTP로     서버에게는 정상 HTTPS로 연결
         보여줌 (자물쇠 없음)

HSTS가 있으면:
사용자 → (브라우저가 자동으로 https://로 전환) → https://example.com
         공격자가 개입할 여지 없음
```

### HSTS Preload

첫 방문 전에도 HTTPS를 강제하려면 HSTS Preload List에 등록한다. 브라우저에 하드코딩된다:

```bash
# 등록: https://hstspreload.org/
# 조건:
# 1. 유효한 인증서
# 2. HTTP → HTTPS 리다이렉트
# 3. HSTS 헤더에 includeSubDomains와 preload 포함
# 4. max-age >= 1년

# 주의: 한 번 등록하면 제거하기 매우 어려움!
# HTTPS를 포기할 가능성이 있다면 등록하지 말 것
```

---

## 8. Certificate Pinning

### 개념

인증서 피닝은 "이 서버의 인증서(또는 공개키)는 정확히 이것이어야 한다"를 클라이언트에 하드코딩하는 것이다.

```
일반 TLS:
인증서가 유효한 CA에서 발급됨 → 신뢰 ✓
(어떤 CA든 상관없음)

Certificate Pinning:
인증서가 유효한 CA에서 발급됨 AND 공개키가 미리 등록된 것과 일치 → 신뢰 ✓
(특정 CA/인증서만 허용)
```

### 왜 필요한가

전 세계에 수백 개의 CA가 있고, 하나라도 해킹당하면 어떤 도메인의 인증서든 발급할 수 있다. 실제 사례:
- 2011년 DigiNotar 해킹 → google.com에 대한 가짜 인증서 발급
- 2015년 CNNIC → google.com 중간 인증서 발급

### 구현 방법

**HPKP (HTTP Public Key Pinning)** — 더 이상 사용하지 않음:

브라우저에서 제거됨 (2019년). 잘못된 핀 설정 시 사이트가 완전히 접근 불가능해지는 위험 때문.

**클라이언트 앱에서 직접 구현** — 모바일 앱에서 주로 사용:
```
// Android (OkHttp)
CertificatePinner pinner = new CertificatePinner.Builder()
    .add("api.example.com",
         "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")  // 주 핀
    .add("api.example.com",
         "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")  // 백업 핀
    .build();
```

**주의사항:**
- 반드시 **백업 핀**을 포함해야 함 (인증서 갱신 시 핀도 바꿔야 하므로)
- 인증서가 아닌 **공개키의 해시**를 피닝하면 인증서 갱신이 더 유연함
- 피닝 실패 시 앱이 완전히 작동 불가 → 매우 신중하게 운영해야 함

---

## 9. TLS 1.2 vs 1.3 비교 정리

```
항목                    TLS 1.2              TLS 1.3
────────────────────────────────────────────────────────
핸드셰이크 RTT         2 RTT                1 RTT (재연결: 0 RTT)
키 교환                RSA, DHE, ECDHE      ECDHE만 (PFS 강제)
암호 스위트             수십 개               5개
암호 모드              CBC, GCM             AEAD만 (GCM, CCM, ChaCha20)
핸드셰이크 암호화       일부 평문             ServerHello 이후 전부 암호화
압축                   지원                  제거 (CRIME 방어)
재협상                  지원                  제거
0-RTT                  없음                  지원 (재연결 시)
SNI 암호화             없음                  ECH (실험적)

성능 차이:
TLS 1.2: TCP 핸드셰이크 (1 RTT) + TLS 핸드셰이크 (2 RTT) = 3 RTT
TLS 1.3: TCP 핸드셰이크 (1 RTT) + TLS 핸드셰이크 (1 RTT) = 2 RTT
TLS 1.3 재연결: TCP (1 RTT) + TLS 0-RTT = 1 RTT (+데이터)

RTT 50ms 기준:
TLS 1.2: 150ms
TLS 1.3: 100ms  (33% 개선)
TLS 1.3 0-RTT: 50ms  (67% 개선)
```

---

## 10. Let's Encrypt — 인증서 자동화

### 왜 Let's Encrypt인가

- **무료**: DV(Domain Validation) 인증서 무료 발급
- **자동화**: ACME 프로토콜로 발급/갱신 자동화
- **짧은 유효기간**: 90일 (자동 갱신 전제, 유출 시 영향 최소화)
- **신뢰**: 모든 주요 브라우저/OS Trust Store에 포함

### ACME 프로토콜과 도메인 검증

```
인증서 발급 과정:

1. 클라이언트(certbot 등)가 Let's Encrypt에 인증서 요청
2. Let's Encrypt: "이 도메인이 진짜 당신 것인지 증명하세요"

HTTP-01 Challenge:
   LE: "http://example.com/.well-known/acme-challenge/TOKEN 에
       이 값을 넣어두세요"
   클라이언트: 웹 서버에 파일 배치
   LE: 해당 URL 접근해서 검증

DNS-01 Challenge:
   LE: "_acme-challenge.example.com TXT 레코드에
       이 값을 넣어두세요"
   클라이언트: DNS에 TXT 레코드 추가
   LE: DNS 조회로 검증

3. 검증 성공 → 인증서 발급
```

### Certbot으로 자동화

```bash
# 설치
$ sudo apt install certbot python3-certbot-nginx

# Nginx 자동 설정 + 인증서 발급
$ sudo certbot --nginx -d example.com -d www.example.com

# 갱신 테스트
$ sudo certbot renew --dry-run

# 자동 갱신 (systemd 타이머로 자동 설정됨)
$ systemctl list-timers | grep certbot
certbot.timer    loaded active waiting    Run certbot twice daily
```

### 와일드카드 인증서

```bash
# DNS-01 챌린지 필요 (HTTP-01로는 와일드카드 불가)
$ sudo certbot certonly \
    --dns-cloudflare \
    --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
    -d "example.com" \
    -d "*.example.com"
```

### 인증서 갱신과 Nginx 리로드

```bash
# /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
#!/bin/bash
nginx -t && systemctl reload nginx

# 권한 설정
$ chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

### 실전 TLS 설정 검증

```bash
# SSL Labs 테스트 (가장 포괄적)
# https://www.ssllabs.com/ssltest/analyze.html?d=example.com

# 커맨드라인 확인
$ openssl s_client -connect example.com:443 -tls1_3
# TLS 1.3 연결 성공 여부

$ openssl s_client -connect example.com:443 2>/dev/null | \
    openssl x509 -noout -dates
# 인증서 유효기간 확인

# testssl.sh (상세 TLS 진단)
$ ./testssl.sh example.com
```

---

## 마치며

TLS는 현대 인터넷 보안의 기반이다. 핵심을 정리하면:

- **대칭 암호화**는 빠르고, **비대칭 암호화**는 안전한 키 교환을 제공한다. TLS는 둘을 결합한 하이브리드 방식이다
- **ECDHE**가 Perfect Forward Secrecy를 보장하며, TLS 1.3에서는 유일한 키 교환 방법이다
- **TLS 1.3**은 1.2 대비 1 RTT 절약, 더 적은 암호 스위트, 핸드셰이크 암호화로 보안과 성능 모두 개선했다
- **인증서 체인**은 루트 CA → 중간 CA → 서버 인증서로 이어지는 신뢰의 사슬이며, 중간 인증서 누락은 흔한 장애 원인이다
- **mTLS**는 서비스 간 상호 인증에, **HSTS**는 SSL Stripping 방어에 필수적이다
- **Let's Encrypt + Certbot**으로 인증서 발급과 갱신을 완전히 자동화할 수 있다

이것으로 [1] OS와 네트워크 시리즈를 마친다. 리눅스 프로세스부터 CPU 스케줄링, 메모리, 파일 시스템, 네트워크, TCP, 웹 서버, TLS까지 — 서버가 동작하는 기반을 깊이 있게 다뤘다.

---

## 참고 자료

- *Bulletproof TLS and PKI, 2nd Edition* — Ivan Ristić (Feisty Duck)
- RFC 8446 (TLS 1.3), RFC 5246 (TLS 1.2)
- *High Performance Browser Networking* — Ilya Grigorik, Chapter 4: TLS — [https://hpbn.co/transport-layer-security-tls/](https://hpbn.co/transport-layer-security-tls/)
- Let's Encrypt Documentation — [https://letsencrypt.org/docs/](https://letsencrypt.org/docs/)
- Mozilla SSL Configuration Generator — [https://ssl-config.mozilla.org/](https://ssl-config.mozilla.org/)
- Qualys SSL Labs — [https://www.ssllabs.com/](https://www.ssllabs.com/)
- Cloudflare Learning: What is TLS? — [https://www.cloudflare.com/learning/ssl/transport-layer-security-tls/](https://www.cloudflare.com/learning/ssl/transport-layer-security-tls/)
