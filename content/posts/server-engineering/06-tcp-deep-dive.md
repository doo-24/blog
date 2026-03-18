---
title: "[OS와 네트워크] 6편 — TCP 심화: 신뢰성의 원리"
date: 2026-03-18T05:00:00+09:00
draft: false
tags: ["TCP", "네트워크", "HTTP/2", "QUIC", "서버"]
series: ["OS와 네트워크"]
summary: "혼잡 제어(CUBIC, BBR), 흐름 제어, 슬라이딩 윈도우의 원리부터 Nagle 알고리즘, TCP 버퍼 튜닝, HTTP/2 멀티플렉싱, HTTP/3(QUIC)까지 — TCP의 깊은 곳을 탐구한다."
---

## 들어가며

5편에서 TCP의 연결과 종료를 다뤘다면, 이번 편은 **데이터가 실제로 어떻게 전달되는가**에 집중한다.

인터넷은 신뢰할 수 없는 네트워크다. 패킷은 유실되고, 순서가 뒤바뀌고, 중복되고, 지연된다. 이 위에서 TCP는 어떻게 신뢰성 있는 전송을 보장하는가? 그리고 그 보장이 왜 때로는 성능을 죽이는가?

---

## 1. 시퀀스 번호와 확인응답 — 신뢰성의 기초

### 기본 메커니즘

TCP의 신뢰성은 **시퀀스 번호(Sequence Number)**와 **확인응답(ACK)**에 기반한다. 모든 바이트에는 번호가 매겨지고, 수신 측은 받은 바이트를 확인해준다.

```
송신 측                                      수신 측
    │  SEG1: seq=1000, len=500              │
    │  (바이트 1000~1499)                    │
    │──────────────────────────────────────→│
    │                                        │
    │          ACK: ack=1500                │
    │←──────────────────────────────────────│
    │  "1500번째 바이트부터 보내줘"            │
    │                                        │
    │  SEG2: seq=1500, len=500              │
    │──────────────────────────────────────→│
    │                                        │
```

ACK 번호의 의미: "이 번호 **이전까지**의 모든 바이트를 받았다." 누적(cumulative) 확인응답이다.

### 패킷 유실과 재전송

```
송신 측                                      수신 측
    │  seq=1000, len=500   ─────────────→  │ ✓ 받음
    │  seq=1500, len=500   ───── X         │ 유실!
    │  seq=2000, len=500   ─────────────→  │ 받았지만 1500~1999가 빠짐
    │                                       │
    │          ACK=1500 (중복)              │
    │←─────────────────────────────────────│ "1500부터 다시 보내"
    │          ACK=1500 (중복)              │
    │←─────────────────────────────────────│
    │          ACK=1500 (중복)              │
    │←─────────────────────────────────────│
    │                                       │
    │  3개의 중복 ACK → Fast Retransmit!    │
    │  seq=1500, len=500   ─────────────→  │ ✓ 받음
    │                                       │
    │          ACK=2500                    │ 1500~2499 한꺼번에 확인
    │←─────────────────────────────────────│
```

**Fast Retransmit**: 타임아웃을 기다리지 않고, 3개의 중복 ACK를 받으면 즉시 재전송한다. 타임아웃 기반 재전송(RTO)은 수백 밀리초~수 초가 걸리므로, Fast Retransmit이 훨씬 빠르다.

### SACK — 선택적 확인응답

누적 ACK만으로는 **어떤 패킷이 유실됐는지** 정확히 알기 어렵다. SACK(Selective Acknowledgment)은 받은 범위를 명시적으로 알려준다.

```
SACK 없이:                        SACK 사용:
ACK=1500                          ACK=1500, SACK=2000-2500
→ "1500 이전까지 받았다"            → "1500 이전까지 받았고,
   (2000~2499도 받았는지 모름)         2000~2499도 받았다"
→ 1500~2499 전체를 재전송           → 1500~1999만 재전송하면 됨
```

```bash
# SACK 활성화 확인 (기본 활성화)
$ sysctl net.ipv4.tcp_sack
net.ipv4.tcp_sack = 1
```

---

## 2. 슬라이딩 윈도우 — 파이프라인의 원리

### Stop-and-Wait의 한계

ACK를 받고 나서 다음 패킷을 보내면(Stop-and-Wait), 대역폭 활용이 극도로 나빠진다.

```
RTT = 100ms, 패킷 크기 = 1KB

Stop-and-Wait:
시간 0ms:    패킷 전송 ──→
시간 50ms:   패킷 도착
시간 100ms:  ACK 도착    ←──
시간 100ms:  다음 패킷 전송 ──→
...
처리량: 1KB / 100ms = 10 KB/s = 80 Kbps  ← 1Gbps 회선에서 이 정도!
```

### 슬라이딩 윈도우

ACK를 기다리지 않고 **윈도우 크기만큼 연속으로** 전송한다:

```
윈도우 크기 = 4 세그먼트

         ┌─────────────────────────────────────┐
         │        송신 윈도우 (4 세그먼트)        │
─────────┴─────────────────────────────────────┴─────────
│ ACK됨  │ 전송됨,   │ 전송됨,   │ 전송됨,   │ 전송됨,   │ 미전송
│        │ ACK 대기  │ ACK 대기  │ ACK 대기  │ ACK 대기  │
─────────────────────────────────────────────────────────
  seq:    1000       1500       2000       2500       3000

ACK=1500 도착:
─────────────────────────────────────────────────────────
│ ACK됨  │ ACK됨   │ 전송됨,   │ 전송됨,   │ 전송됨,   │ 전송 가능│ 미전송
│        │         │ ACK 대기  │ ACK 대기  │ ACK 대기  │ (새로!)  │
─────────────────────────────────────────────────────────
         ┌─────────────────────────────────────┐
         │     윈도우가 오른쪽으로 "슬라이드"      │
         └─────────────────────────────────────┘
```

윈도우가 클수록 한 번에 더 많은 데이터를 전송할 수 있다. 하지만 윈도우를 무한히 키울 수는 없다 — 수신 측 버퍼와 네트워크 용량이라는 두 가지 제약이 있다.

---

## 3. 흐름 제어 — 수신 측이 속도를 조절한다

### 수신 윈도우 (rwnd)

수신 측은 TCP 헤더의 **Window Size** 필드로 "내 버퍼에 여유가 이만큼 있다"고 알린다. 송신 측은 이 값보다 많은 미확인 데이터를 보내지 않는다.

```
수신 측 버퍼: 64KB
┌───────────────────────────────────────────┐
│ 수신 완료, 앱이 │      수신 완료,          │ 빈 공간  │
│   읽어감        │    앱이 아직 안 읽음      │ (rwnd)  │
└───────────────────────────────────────────┘
                   ↑                          ↑
               읽기 포인터              버퍼 끝

rwnd = 빈 공간 = 애플리케이션이 읽는 속도에 따라 변동
```

**Zero Window 상황:**
```
수신 측 버퍼가 가득 참 → rwnd = 0 → 송신 측 전송 중단!

송신 측                              수신 측
    │  데이터 ──────────────────→   │ 버퍼 가득
    │                               │
    │  ←── ACK, Window=0 ──────    │ "그만 보내!"
    │                               │
    │  (전송 중단)                   │
    │                               │ 앱이 데이터 읽음
    │  Window Probe ──────────→    │
    │                               │
    │  ←── ACK, Window=32768 ──    │ "다시 보내도 돼"
    │                               │
    │  데이터 재개 ────────────→    │
```

Zero Window는 수신 애플리케이션의 처리가 느릴 때 발생한다. `ss`로 `Recv-Q`가 쌓이는 것과 같은 현상이다.

### Window Scale 옵션

TCP 헤더의 Window Size 필드는 16비트 — 최대 65535바이트다. 현대 네트워크에서는 턱없이 작다.

Window Scale 옵션은 3-way handshake에서 교환되며, 윈도우 값에 2^n을 곱한다:

```
Window Scale = 7 → 윈도우 값 × 128
Window Size 필드 값: 512 → 실제 윈도우: 512 × 128 = 65536 바이트
최대: 65535 × 2^14 = 1,073,725,440 바이트 ≈ 1GB
```

```bash
# Window Scale 활성화 확인
$ sysctl net.ipv4.tcp_window_scaling
net.ipv4.tcp_window_scaling = 1
```

---

## 4. 혼잡 제어 — 네트워크가 감당할 수 있는 속도

### 흐름 제어 vs 혼잡 제어

```
흐름 제어: 수신 측이 "내 버퍼 여유가 X다" → 수신 윈도우(rwnd)
혼잡 제어: 네트워크가 "내 용량이 이 정도다" → 혼잡 윈도우(cwnd)

실제 전송 가능량 = min(rwnd, cwnd)
```

수신 측은 자기 상태를 직접 알려주지만, 네트워크는 말이 없다. 혼잡 제어 알고리즘은 패킷 유실과 지연을 단서로 네트워크 상태를 **추론**해야 한다.

### CUBIC — 리눅스 기본 혼잡 제어

CUBIC은 리눅스의 기본 혼잡 제어 알고리즘이다 (커널 2.6.19부터). 이름의 유래는 윈도우 크기가 **3차 함수(cubic function)**를 따르기 때문이다.

```
cwnd
  ↑
  │         ╭──────╮
  │        ╱        ╲
  │       ╱          ╲  ← 패킷 유실! cwnd를 줄임
  │      ╱            │
  │     ╱             │
  │    ╱              │
  │   ╱               ╱
  │  ╱        ╭──────╮
  │ ╱        ╱        ╲
  │╱        ╱          ...
  └───────────────────────→ 시간

CUBIC의 동작:
1. 패킷 유실 후 cwnd를 줄임 (보통 70%로)
2. 유실 전 최대값(Wmax)을 기억
3. 빠르게 Wmax에 근접 → Wmax 근처에서 천천히 → Wmax를 넘으면 다시 빠르게
```

CUBIC의 핵심 특성:
- **RTT에 독립적**: 윈도우 증가가 시간 기반이라 RTT가 긴 회선에서도 공정하게 동작
- **유실 기반**: 패킷 유실을 혼잡의 신호로 해석
- 한계: 버퍼블로트(bufferbloat) 환경에서 지연이 커져도 유실이 없으면 계속 보냄

### BBR — 구글의 혁신

BBR(Bottleneck Bandwidth and Round-trip propagation time)은 패킷 유실 대신 **측정된 대역폭과 RTT**를 기준으로 동작한다.

```
CUBIC의 세계관:
"패킷이 유실됐다 → 네트워크가 혼잡하다 → 속도를 줄이자"

BBR의 세계관:
"최대 대역폭(BtlBw)과 최소 RTT(RTprop)를 측정하자
 → 최적 전송률 = BtlBw
 → 최적 inflight = BtlBw × RTprop
 → 이 범위 안에서 보내면 유실도 없고 지연도 최소"
```

```
대역폭
  ↑        ┌──────────────
  │       ╱                  ← 대역폭 포화 (BtlBw)
  │      ╱
  │     ╱
  │    ╱
  │   ╱
  └──┴─────────────────────→ inflight 데이터
          ↑               ↑
       최적 지점      버퍼 오버플로우
     (BBR 목표)       (유실 시작, CUBIC이 감지)

BBR은 여기서 동작         CUBIC은 여기서 반응
   ↓                        ↓
   최소 지연                 높은 지연 + 유실
```

BBR의 장점:
- **버퍼블로트에 강함**: 라우터 버퍼가 가득 차기 전에 속도를 조절
- **유실에 둔감**: 1~2% 유실에도 처리량 유지 (CUBIC은 급감)
- **높은 대역폭 활용**: 장거리 고대역폭 회선에서 CUBIC 대비 2~25배 처리량

```bash
# 현재 혼잡 제어 알고리즘 확인
$ sysctl net.ipv4.tcp_congestion_control
net.ipv4.tcp_congestion_control = cubic

# BBR로 변경 (커널 4.9+)
$ sysctl -w net.ipv4.tcp_congestion_control=bbr

# BBR 사용에 필요한 fq 큐잉 디시플린 설정
$ tc qdisc replace dev eth0 root fq
```

**BBR의 주의점**: BBR v1은 다른 CUBIC 플로우와 공존 시 불공정하게 대역폭을 많이 가져가는 문제가 있다. BBR v2(커널 5.18+)에서 상당 부분 개선되었다.

### Slow Start — 연결 시작 시 속도 탐색

새 TCP 연결은 네트워크 용량을 모른다. Slow Start는 cwnd를 1 MSS부터 시작해서 **지수적으로** 증가시킨다:

```
RTT 1: cwnd = 1 MSS  →  1개 세그먼트 전송
RTT 2: cwnd = 2 MSS  →  2개 전송
RTT 3: cwnd = 4 MSS  →  4개 전송
RTT 4: cwnd = 8 MSS  →  8개 전송
...
cwnd가 ssthresh(slow start threshold)에 도달하면 → 혼잡 회피(선형 증가)로 전환
```

"Slow" Start라는 이름이 붙었지만 실제로는 지수적 증가라 꽤 빠르다. 그래도 RTT가 길면 최대 속도에 도달하기까지 수 초가 걸린다.

```
1Gbps 회선, RTT 100ms, MSS 1460B:
1Gbps를 채우려면 cwnd ≈ 8.5MB 필요
2^n × 1460B = 8.5MB → n ≈ 13 RTT
13 × 100ms = 1.3초

→ 새 연결은 1.3초 동안 최대 속도를 못 냄!
→ HTTP Keep-Alive, 커넥션 풀이 중요한 이유
```

---

## 5. Nagle 알고리즘과 TCP_NODELAY

### Nagle 알고리즘

작은 패킷을 많이 보내면 헤더 오버헤드가 커진다 (40바이트 헤더에 1바이트 데이터). Nagle 알고리즘은 작은 데이터를 모아서 한 번에 보낸다:

```
Nagle 알고리즘 규칙:
1. 미확인(unACKed) 데이터가 없으면 → 즉시 전송
2. 미확인 데이터가 있으면:
   a. 데이터가 MSS만큼 모이면 → 전송
   b. 이전 ACK가 도착하면 → 모인 데이터 전송

효과:
write("H") → 즉시 전송 [H]
write("e") → 대기 (ACK 아직)
write("l") → 대기
write("l") → 대기
ACK 도착  → [ello] 한꺼번에 전송
```

### Nagle + Delayed ACK = 지연의 함정

수신 측에는 **Delayed ACK**가 있다 — ACK를 즉시 보내지 않고 40~200ms 기다려서 데이터와 함께 보내거나 ACK를 모아서 보낸다.

이 둘이 만나면:

```
클라이언트 (Nagle ON)              서버 (Delayed ACK ON)
    │  write(헤더, 100B)           │
    │  [100B 패킷] ──────────→    │ 받았지만 ACK 지연 (40ms 대기)
    │                              │
    │  write(본문, 50B)            │
    │  (Nagle: ACK 대기중, 보류)    │
    │                              │
    │  ... 40ms 대기 ...           │
    │                              │  타이머 만료, ACK 전송
    │  ←──── ACK ────────────     │
    │                              │
    │  [50B 패킷] ──────────→     │
    │                              │

→ 40ms 불필요한 지연 발생!
```

### TCP_NODELAY — Nagle 비활성화

대화형 프로토콜이나 저지연이 중요한 서비스에서는 Nagle을 끈다:

```c
int flag = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
// write() 호출 즉시 전송 — 소규모 패킷이 많아질 수 있음
```

**TCP_NODELAY를 사용해야 하는 경우:**
- 실시간 게임 서버
- Redis, Memcached 같은 저지연 키-밸류 스토어
- HTTP/2 (자체적으로 프레이밍하므로 Nagle 불필요)
- RPC 프레임워크 (gRPC 등)
- 채팅, 웹소켓 기반 실시간 서비스

**TCP_NODELAY를 쓰지 않아도 되는 경우:**
- 대용량 파일 전송 (이미 MSS 단위로 보내므로 Nagle 영향 없음)
- 배치 로그 전송

### TCP_CORK — 의도적 지연

`TCP_CORK`는 Nagle의 강화판이다. 설정하면 MSS가 채워질 때까지 **절대** 전송하지 않는다. 여러 `write()` 호출로 구성된 응답을 하나의 패킷으로 보내고 싶을 때 사용한다:

```c
int on = 1;
setsockopt(fd, IPPROTO_TCP, TCP_CORK, &on, sizeof(on));

write(fd, http_header, header_len);   // 보내지 않음
write(fd, http_body, body_len);       // 보내지 않음

int off = 0;
setsockopt(fd, IPPROTO_TCP, TCP_CORK, &off, sizeof(off));
// Cork 해제 → 모아둔 데이터 한꺼번에 전송
```

Nginx의 `tcp_nopush on;`이 내부적으로 `TCP_CORK`를 사용한다.

---

## 6. TCP 버퍼 튜닝

### 소켓 버퍼가 성능에 미치는 영향

TCP의 처리량은 **BDP(Bandwidth-Delay Product)**에 의해 결정된다:

```
BDP = 대역폭 × RTT

예: 1Gbps 회선, RTT 50ms
BDP = 1,000,000,000 bits/s × 0.05s = 50,000,000 bits = 6.25 MB

→ 소켓 버퍼가 최소 6.25MB는 되어야 1Gbps를 활용 가능
   버퍼가 1MB이면 → 최대 160 Mbps (회선의 16%만 사용)
```

### 자동 튜닝

리눅스는 `tcp_rmem`/`tcp_wmem`의 min-default-max 범위 내에서 소켓 버퍼를 자동 조절한다:

```bash
# TCP 수신 버퍼: min / default / max
$ sysctl net.ipv4.tcp_rmem
net.ipv4.tcp_rmem = 4096    131072    6291456

# TCP 송신 버퍼: min / default / max
$ sysctl net.ipv4.tcp_wmem
net.ipv4.tcp_wmem = 4096    16384     4194304

# 자동 튜닝 활성화 여부
$ sysctl net.ipv4.tcp_moderate_rcvbuf
net.ipv4.tcp_moderate_rcvbuf = 1   # 1 = 활성화
```

자동 튜닝은 대부분 잘 동작하지만, max 값이 BDP보다 작으면 한계에 도달한다.

### 고대역폭/장거리 환경 튜닝

```bash
# 10Gbps 회선, RTT 100ms → BDP = 125MB
# /etc/sysctl.d/99-tcp-tuning.conf

# TCP 버퍼 최대값 확대
net.ipv4.tcp_rmem = 4096 131072 134217728    # max 128MB
net.ipv4.tcp_wmem = 4096 65536 134217728     # max 128MB

# 시스템 전체 TCP 메모리 한도도 함께 올려야 함
net.ipv4.tcp_mem = 786432 1048576 1572864    # 페이지 단위

# 소켓 단위 최대 버퍼 (tcp_rmem/wmem의 상한)
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
```

### 애플리케이션 레벨 버퍼 확인

```bash
# 특정 연결의 버퍼 상태 확인
$ ss -tim dst 10.0.1.100
State  Recv-Q  Send-Q  Local:Port  Peer:Port
ESTAB  0       36864   10.0.1.5:80 10.0.1.100:54321
     skmem:(r0,rb131072,t0,tb87040,f4096,w36864,o0,bl0,d0)
     # rb = 수신 버퍼 크기 (131072 = 128KB)
     # tb = 송신 버퍼 크기 (87040 ≈ 85KB)
     # w  = 송신 대기 중인 데이터
```

---

## 7. HTTP/2 멀티플렉싱 — Head-of-Line Blocking 해결

### HTTP/1.1의 한계 다시 보기

HTTP/1.1에서 하나의 TCP 연결은 한 번에 하나의 요청-응답만 처리한다. 브라우저는 이를 우회하기 위해 도메인당 6개의 TCP 연결을 병렬로 맺는다.

```
HTTP/1.1:
TCP 연결 1: [──── 요청A ────][──── 요청D ────]
TCP 연결 2: [──── 요청B ────][──── 요청E ────]
TCP 연결 3: [── 요청C ──]   [──── 요청F ────]

문제: 6개 TCP 연결 × Slow Start 비용 + 서버 자원 소모
```

### HTTP/2의 스트림과 프레임

HTTP/2는 **하나의 TCP 연결**에서 여러 요청-응답을 동시에 처리한다:

```
HTTP/2:
하나의 TCP 연결:
┌───────────────────────────────────────────┐
│ [A 헤더][B 헤더][A 데이터][C 헤더]         │
│ [B 데이터][A 데이터][C 데이터][B 데이터]    │
│ [C 데이터][A 완료][B 완료][C 완료]         │
└───────────────────────────────────────────┘

→ 여러 스트림의 프레임이 인터리빙(interleaving)
→ 느린 응답이 빠른 응답을 블록하지 않음
```

핵심 개념:
- **스트림(Stream)**: 하나의 요청-응답 쌍. 고유 ID를 가짐
- **프레임(Frame)**: 데이터 전송의 최소 단위. 스트림 ID가 태그됨
- **멀티플렉싱**: 여러 스트림의 프레임이 하나의 TCP 연결에서 교차 전송

```
HTTP/2 프레임 구조:
┌─────────────────────────────────────┐
│ Length (24 bit)                      │
│ Type (8 bit): HEADERS, DATA, ...    │
│ Flags (8 bit): END_STREAM, ...      │
│ Stream ID (31 bit)                  │
│ Payload                             │
└─────────────────────────────────────┘
```

### HTTP/2의 추가 기능

**헤더 압축 (HPACK)**:
HTTP 헤더는 반복이 많다 (Host, User-Agent, Cookie...). HPACK은 정적/동적 테이블로 헤더를 인덱싱해서 크기를 85~90% 줄인다.

```
첫 번째 요청:
  :method: GET
  :path: /api/users
  host: api.example.com
  authorization: Bearer eyJ...  (긴 토큰)
  → 전체 전송

두 번째 요청:
  :method: GET
  :path: /api/posts           ← path만 다름
  (나머지는 인덱스 참조)
  → 몇 바이트만 전송
```

**서버 푸시 (Server Push)**:
HTML 요청 시 서버가 CSS, JS를 미리 보내준다. 하지만 실제로는 캐시 무효화 문제로 잘 사용되지 않으며, Chrome 106에서 제거되었다.

**스트림 우선순위**:
CSS, JS 같은 렌더링 필수 리소스에 높은 우선순위를 부여할 수 있다.

### HTTP/2의 남은 문제 — TCP Head-of-Line Blocking

HTTP/2는 HTTP 레벨의 HoL Blocking을 해결했지만, **TCP 레벨**에서는 여전히 존재한다:

```
TCP 연결에서 패킷 유실 발생:

[스트림1][스트림2][스트림3][유실!][스트림1][스트림2][스트림3]
                           ↑
                    이 패킷이 재전송될 때까지
                    뒤의 모든 스트림이 블록됨!

→ HTTP/2의 멀티플렉싱이 오히려 불리할 수 있음
   (HTTP/1.1은 6개 독립 TCP 연결 → 1개 유실이 나머지에 영향 안 줌)
```

이것이 HTTP/3가 탄생한 배경이다.

---

## 8. HTTP/3과 QUIC — TCP를 넘어서

### QUIC의 핵심 아이디어

HTTP/3는 TCP 대신 **QUIC** 프로토콜 위에서 동작한다. QUIC는 UDP를 기반으로 TCP의 기능(신뢰성, 순서 보장, 혼잡 제어)을 유저 공간에서 구현한다.

```
HTTP/1.1, HTTP/2:               HTTP/3:
┌──────────────┐               ┌──────────────┐
│    HTTP       │               │    HTTP       │
├──────────────┤               ├──────────────┤
│    TLS 1.2/3 │               │    QUIC       │ ← TCP + TLS 통합
├──────────────┤               │    (신뢰성,    │
│    TCP        │               │     암호화,    │
├──────────────┤               │     멀티플렉싱)│
│    IP         │               ├──────────────┤
└──────────────┘               │    UDP        │
                                ├──────────────┤
                                │    IP         │
                                └──────────────┘
```

### QUIC가 해결하는 문제들

**1. 스트림 독립성 — TCP HoL Blocking 제거**

```
QUIC:
스트림 1: [데이터][데이터][유실][데이터]
스트림 2: [데이터][데이터][데이터][데이터]  ← 영향 없음!
스트림 3: [데이터][데이터][데이터][데이터]  ← 영향 없음!

→ 각 스트림이 독립적으로 재전송/순서 보장
→ 한 스트림의 패킷 유실이 다른 스트림을 블록하지 않음
```

**2. 0-RTT 연결 수립**

```
TCP + TLS 1.3:                    QUIC:
1 RTT: TCP SYN                    0 RTT: 이전 연결 정보로
1 RTT: TCP SYN+ACK                       즉시 데이터 전송 가능!
1 RTT: TLS ClientHello            (최초 연결은 1 RTT)
= 2~3 RTT 후 데이터 전송 시작

RTT 100ms인 경우:
TCP+TLS: 200~300ms 후 첫 데이터
QUIC 재연결: 0ms (이전 키로 즉시 전송)
```

**3. 연결 마이그레이션**

TCP 연결은 (src IP, src port, dst IP, dst port) 4-tuple로 식별된다. Wi-Fi → 셀룰러 전환 시 IP가 바뀌면 연결이 끊긴다.

QUIC는 **Connection ID**로 연결을 식별한다. IP가 바뀌어도 Connection ID가 같으면 연결을 유지한다.

```
Wi-Fi:    IP 192.168.1.5   ─┐
                              ├→ Connection ID: 0xABCD → 서버
셀룰러:   IP 10.0.0.100    ─┘

→ IP가 바뀌어도 Connection ID로 연결 유지
→ 모바일 환경에서 끊김 없는 통신
```

### QUIC의 현실

```
지원 현황 (2025):
- Chrome, Firefox, Safari, Edge: 지원
- Nginx: 실험적 지원 (1.25.0+)
- Caddy: 네이티브 지원
- HAProxy: 2.6+
- curl: --http3 플래그

서버 설정 예시 (Nginx):
server {
    listen 443 quic reuseport;     # QUIC (UDP)
    listen 443 ssl;                 # HTTPS (TCP) 폴백

    http2 on;
    http3 on;

    add_header Alt-Svc 'h3=":443"; ma=86400';  # HTTP/3 광고
}
```

주의점:
- **UDP 방화벽**: 많은 기업 네트워크가 UDP 443을 차단 → TCP 폴백 필수
- **디버깅 복잡성**: 암호화된 UDP이므로 tcpdump로 분석이 어려움
- **CPU 오버헤드**: 유저 공간 구현이므로 TCP 대비 CPU 사용량이 높을 수 있음

---

## 마치며

TCP는 단순해 보이지만, 그 안에는 수십 년간 축적된 네트워크 공학의 정수가 담겨 있다.

- **시퀀스 번호와 ACK**가 신뢰성의 기초이며, **SACK**가 재전송 효율을 높인다
- **슬라이딩 윈도우**가 파이프라인 전송을 가능하게 하고, **흐름 제어(rwnd)**가 수신 측 과부하를 방지한다
- **혼잡 제어**는 CUBIC(유실 기반)에서 BBR(측정 기반)으로 진화하고 있으며, 알고리즘 선택이 처리량에 직접 영향을 준다
- **Nagle + Delayed ACK**의 함정을 이해하고, 필요한 곳에 **TCP_NODELAY**를 적용해야 한다
- **HTTP/2**가 HTTP 레벨 HoL Blocking을 해결했고, **HTTP/3(QUIC)**가 TCP 레벨까지 해결하며 연결 수립 지연도 제거했다

다음 편에서는 웹 서버와 리버스 프록시를 다룬다 — Nginx의 이벤트 기반 아키텍처, 로드밸런싱, SSL 종단, 그리고 실전 설정을 탐구한다.

---

## 참고 자료

- *TCP/IP Illustrated, Volume 1, 2nd Edition* — Kevin R. Fall, W. Richard Stevens (Addison-Wesley)
- RFC 5681 (TCP Congestion Control), RFC 6298 (TCP RTO), RFC 2018 (TCP SACK)
- Google BBR Paper: "BBR: Congestion-Based Congestion Control" — Neal Cardwell et al., ACM Queue 2016
- CUBIC Paper: "CUBIC: A New TCP-Friendly High-Speed TCP Variant" — Sangtae Ha et al.
- RFC 9000 (QUIC), RFC 9114 (HTTP/3)
- Linux Kernel Documentation: TCP — [https://docs.kernel.org/networking/ip-sysctl.html](https://docs.kernel.org/networking/ip-sysctl.html)
- `man 7 tcp`, `man 2 setsockopt`
