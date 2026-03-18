---
title: "네트워크와 커넥션 — 서버가 요청을 받고 응답을 돌려주기까지"
date: 2026-03-18T04:00:00+09:00
draft: false
tags: ["네트워크", "TCP", "HTTP", "소켓", "서버"]
series: ["OS와 네트워크"]
summary: "TCP 3-way handshake부터 소켓 생명주기, HTTP/1.1 요청 흐름, TIME_WAIT, 그리고 tcpdump까지 — 서버가 네트워크 요청을 처리하는 전체 과정을 깊이 파헤친다."
---

## 들어가며

브라우저에서 URL을 입력하고 엔터를 누르면 무슨 일이 벌어질까? 이 질문은 면접 단골이지만, 대부분의 답변은 표면적이다. "DNS 조회 → TCP 연결 → HTTP 요청 → 응답" 정도로 끝난다.

하지만 서버 엔지니어에게는 각 단계의 **커널 레벨 동작**이 중요하다. TCP 연결 하나가 만들어지는 과정에서 커널은 어떤 자료구조를 사용하고, 소켓 버퍼는 어떻게 관리되며, TIME_WAIT 상태는 왜 서버를 괴롭히는가? 이 편에서는 네트워크 통신의 기초를 커널의 관점에서 깊이 다룬다.

---

## 1. OSI 모델과 리눅스 네트워크 스택

### 실무에서의 네트워크 계층

OSI 7계층은 이론적 모델이다. 실제 리눅스 커널은 4계층(TCP/IP 모델)으로 동작한다:

```
OSI 7계층              TCP/IP 4계층           리눅스 커널 구현
─────────────────────────────────────────────────────────────
응용 (7)  ─┐
표현 (6)   ├─ 응용 계층 ─────── 유저 공간 (프로세스)
세션 (5)  ─┘
전송 (4)  ──── 전송 계층 ─────── TCP/UDP (net/ipv4/tcp*.c)
네트워크 (3)── 인터넷 계층 ───── IP, ICMP (net/ipv4/ip*.c)
데이터링크(2)─┐
물리 (1)     ├─ 네트워크 접근 ── 디바이스 드라이버 + NIC
             ─┘
```

패킷이 NIC에 도착하면 커널 내에서 아래에서 위로 올라간다:

```
NIC (하드웨어 인터럽트)
  → NAPI (소프트 인터럽트로 폴링 전환)
    → IP 레이어 (라우팅, 필터링)
      → TCP 레이어 (시퀀스 처리, 재조립)
        → 소켓 수신 버퍼
          → 애플리케이션의 read()/recv()
```

### sk_buff — 커널의 패킷 자료구조

리눅스 커널에서 모든 네트워크 패킷은 `sk_buff`(소켓 버퍼) 구조체로 표현된다. 패킷이 계층을 지나갈 때 데이터를 복사하지 않고 **포인터만 조정**한다:

```
sk_buff 구조:
┌──────────────────────────────────┐
│ head ──→ [여유 공간 (headroom)]   │
│ data ──→ [현재 계층 헤더 시작]    │
│          [IP 헤더]                │
│          [TCP 헤더]               │
│          [페이로드 데이터]         │
│ tail ──→ [데이터 끝]             │
│ end  ──→ [버퍼 끝]               │
└──────────────────────────────────┘

수신 시: data 포인터를 아래로 이동 (헤더 파싱)
송신 시: data 포인터를 위로 이동 (헤더 추가)
→ 데이터 복사 없이 계층 간 이동!
```

---

## 2. TCP 3-Way Handshake — 연결의 시작

### 핸드셰이크 과정

TCP는 연결 지향(connection-oriented) 프로토콜이다. 데이터 전송 전에 양쪽이 합의하는 과정이 필요하다:

```
클라이언트                          서버
    │                                │
    │  1. SYN (seq=x)               │
    │──────────────────────────────→ │  서버: SYN_RECV 상태
    │                                │  (syn queue에 추가)
    │  2. SYN+ACK (seq=y, ack=x+1)  │
    │ ←──────────────────────────────│
    │                                │
    │  3. ACK (ack=y+1)             │
    │──────────────────────────────→ │  서버: ESTABLISHED 상태
    │                                │  (accept queue로 이동)
    │         연결 수립 완료          │
    │                                │
```

이 3단계에서 교환하는 정보:
- **ISN (Initial Sequence Number)**: 각 방향의 시작 시퀀스 번호 (보안상 랜덤)
- **MSS (Maximum Segment Size)**: 한 세그먼트의 최대 크기 (보통 1460 bytes)
- **Window Scale**: 수신 윈도우 크기의 스케일링 팩터
- **SACK Permitted**: 선택적 확인응답 지원 여부
- **Timestamp**: RTT 측정과 PAWS(Protection Against Wrapped Sequences)

### 서버 커널의 두 큐

서버 측에서 핸드셰이크를 처리하는 커널 자료구조를 이해하는 것이 중요하다:

```
클라이언트 SYN 도착
        │
        ▼
┌──────────────────┐    SYN+ACK 전송
│   SYN Queue      │──────────────→ 클라이언트
│ (반개방 연결 큐)   │
│ 크기: tcp_max_syn │    ACK 도착
│      _backlog     │←──────────────
└────────┬─────────┘
         │ 핸드셰이크 완료
         ▼
┌──────────────────┐
│  Accept Queue    │
│ (완료된 연결 큐)   │
│ 크기: listen()의  │
│      backlog     │
└────────┬─────────┘
         │ accept() 호출
         ▼
    애플리케이션으로 전달
```

**SYN Queue (반개방 연결 큐)**:
- SYN을 받고 SYN+ACK를 보낸 상태 (SYN_RECV)
- `net.ipv4.tcp_max_syn_backlog`으로 크기 조절 (기본 1024)
- 가득 차면? → SYN Cookie 활성화 또는 SYN 드랍

**Accept Queue (완료된 연결 큐)**:
- 3-way handshake가 완료된 연결
- `listen(fd, backlog)`의 backlog 파라미터로 크기 설정
- 가득 차면? → 새로운 ACK를 무시 (클라이언트는 재전송)

```bash
# 큐 상태 확인
$ ss -ltn
State    Recv-Q  Send-Q  Local Address:Port   Peer Address:Port
LISTEN   0       128     *:80                 *:*
#        ↑       ↑
#        현재     backlog
#        Accept   크기
#        Queue 길이

# SYN 큐 관련 커널 파라미터
$ sysctl net.ipv4.tcp_max_syn_backlog
net.ipv4.tcp_max_syn_backlog = 1024

$ sysctl net.ipv4.tcp_syncookies
net.ipv4.tcp_syncookies = 1   # SYN Flood 방어
```

### SYN Flood와 SYN Cookie

SYN Flood 공격은 대량의 SYN 패킷을 보내고 ACK를 보내지 않아서 SYN Queue를 고갈시키는 공격이다.

```
공격자 → SYN (가짜 IP) → 서버 SYN Queue [■■■■■■■■■■] 가득 참!
공격자 → SYN (가짜 IP) →                 정상 연결 불가
공격자 → SYN (가짜 IP) →
```

**SYN Cookie**는 SYN Queue 없이 핸드셰이크를 완료하는 기법이다:
1. SYN 도착 시 Queue에 저장하지 않고, ISN에 연결 정보를 암호화해서 인코딩
2. SYN+ACK를 보냄
3. ACK가 돌아오면 ISN에서 연결 정보를 복원
4. Queue를 사용하지 않으므로 고갈이 불가

단점: SYN Cookie 사용 시 일부 TCP 옵션(Window Scale, SACK)이 제한된다. 따라서 평시에는 끄고, Queue가 넘칠 때만 활성화하는 것이 이상적이다.

---

## 3. TCP 4-Way Teardown — 연결의 종료

### 정상 종료 과정

```
종료 요청 측 (Active Close)           수신 측 (Passive Close)
        │                                │
        │  1. FIN (seq=m)               │
        │──────────────────────────────→│  수신 측: CLOSE_WAIT
        │  FIN_WAIT_1                   │
        │                                │
        │  2. ACK (ack=m+1)            │
        │←──────────────────────────────│
        │  FIN_WAIT_2                   │  (아직 보낼 데이터가 있을 수 있음)
        │                                │
        │  3. FIN (seq=n)              │
        │←──────────────────────────────│  수신 측: LAST_ACK
        │                                │
        │  4. ACK (ack=n+1)            │
        │──────────────────────────────→│  수신 측: CLOSED
        │  TIME_WAIT (2MSL 대기)        │
        │                                │
        │  ... 60초 후 ...              │
        │  CLOSED                       │
```

왜 4-way인가? TCP는 **양방향 독립** 종료를 지원한다. 한쪽이 FIN을 보내도 반대 방향은 여전히 데이터를 보낼 수 있다 (half-close). 이것이 3-way가 아닌 4-way인 이유다.

### 소켓 상태 전이 전체도

```
                    ┌──────────┐
                    │  CLOSED  │
                    └────┬─────┘
           listen()  ────┤──── connect() (SYN 전송)
                    ┌────▼─────┐          ┌─────────────┐
                    │  LISTEN  │          │  SYN_SENT   │
                    └────┬─────┘          └──────┬──────┘
             SYN 수신    │                SYN+ACK 수신
           SYN+ACK 전송  │                ACK 전송
                    ┌────▼─────┐          ┌──────▼──────┐
                    │ SYN_RECV │          │ ESTABLISHED │
                    └────┬─────┘          └──────┬──────┘
               ACK 수신  │                       │
                    ┌────▼─────┐          FIN 수신│FIN 전송
                    │ESTABLISHED│               │      │
                    └────┬─────┘    ┌───────┐   │  ┌───▼──────┐
                    FIN 전송│       │CLOSE_ │←──┘  │FIN_WAIT_1│
                         │        │ WAIT  │       └───┬──────┘
                    ┌────▼─────┐  └───┬───┘    ACK수신│
                    │FIN_WAIT_1│  FIN전송│      ┌───▼──────┐
                    └──────────┘  ┌───▼───┐    │FIN_WAIT_2│
                                  │LAST_  │    └───┬──────┘
                                  │ ACK   │   FIN수신│
                                  └───┬───┘    ┌───▼──────┐
                               ACK수신│        │TIME_WAIT │
                                  ┌───▼───┐    └───┬──────┘
                                  │CLOSED │   2MSL후│
                                  └───────┘    ┌───▼──────┐
                                               │  CLOSED  │
                                               └──────────┘
```

---

## 4. TIME_WAIT — 서버 운영자의 숙적

### TIME_WAIT가 존재하는 이유

Active Close 측(FIN을 먼저 보낸 쪽)은 마지막 ACK를 보낸 후 바로 CLOSED로 가지 않고 **2MSL (Maximum Segment Lifetime, 보통 60초)** 동안 TIME_WAIT 상태를 유지한다.

두 가지 이유가 있다:

**1. 마지막 ACK 유실 대비**
```
종료 요청 측         수신 측
    │  ACK ──X──    │  (ACK가 유실됨)
    │               │  FIN 재전송
    │  ←── FIN ──── │
    │  ACK ────────→│  (TIME_WAIT 중이므로 ACK 재전송 가능)
```

만약 바로 CLOSED가 되면, 수신 측의 FIN 재전송에 RST로 응답하게 되어 비정상 종료가 된다.

**2. 지연 패킷 오염 방지**
```
이전 연결의 지연 패킷:  src=A:5000 dst=B:80 seq=12345
새 연결 (같은 포트):    src=A:5000 dst=B:80

TIME_WAIT 없이 같은 포트로 새 연결을 맺으면,
이전 연결의 지연 패킷이 새 연결의 데이터로 오인될 수 있다.
```

### TIME_WAIT 폭증 문제

서버가 Active Close 측일 때 TIME_WAIT가 쌓인다. 예를 들어 HTTP/1.0에서 서버가 응답 후 연결을 끊는 경우:

```bash
# TIME_WAIT 상태 소켓 수 확인
$ ss -s
TCP:   12567 (estab 456, closed 11234, orphaned 0, timewait 10890)

# TIME_WAIT 상태의 상세 목록
$ ss -tan state time-wait | head
Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
0       0       10.0.1.5:80         203.0.113.1:54321
0       0       10.0.1.5:80         203.0.113.2:54322
...
```

TIME_WAIT 소켓이 수만 개 쌓이면 두 가지 문제가 발생한다:
- **메모리 소비**: 소켓당 약 200바이트 (큰 문제는 아님)
- **포트 고갈**: 클라이언트 측에서 발생. 로컬 포트 범위(기본 32768-60999)를 다 쓰면 새 연결 불가

### TIME_WAIT 대응 전략

```bash
# 1. tcp_tw_reuse: TIME_WAIT 소켓을 새 나가는(outgoing) 연결에 재사용
#    안전: 타임스탬프를 검사해서 지연 패킷 오인을 방지
$ sysctl -w net.ipv4.tcp_tw_reuse=1

# 2. tcp_max_tw_buckets: TIME_WAIT 소켓 수 상한
#    초과하면 TIME_WAIT 즉시 파괴 (로그에 경고 출력)
$ sysctl -w net.ipv4.tcp_max_tw_buckets=20000

# 3. 로컬 포트 범위 확대
$ sysctl -w net.ipv4.ip_local_port_range="10000 65535"
```

**가장 좋은 해법은 TIME_WAIT 자체를 줄이는 것이다:**
- **HTTP Keep-Alive** 활성화 → 연결 재사용으로 종료 횟수 감소
- **Connection Pool** 사용 → 데이터베이스/외부 API 연결 재사용
- **서버가 Active Close를 하지 않도록** 설계 → 클라이언트가 먼저 끊게 유도

---

## 5. 소켓 생명주기 — 서버 프로그래밍의 뼈대

### 서버 소켓의 전체 흐름

```c
// 1. 소켓 생성
int server_fd = socket(AF_INET, SOCK_STREAM, 0);

// 2. 주소 바인딩
struct sockaddr_in addr = {
    .sin_family = AF_INET,
    .sin_addr.s_addr = INADDR_ANY,  // 모든 인터페이스
    .sin_port = htons(80)
};
bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));

// 3. 리스닝 상태 전환
listen(server_fd, 128);  // backlog = 128

// 4. 연결 수락 (블로킹)
struct sockaddr_in client_addr;
socklen_t len = sizeof(client_addr);
int client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &len);

// 5. 데이터 송수신
char buf[4096];
ssize_t n = read(client_fd, buf, sizeof(buf));
write(client_fd, response, response_len);

// 6. 연결 종료
close(client_fd);
```

```
시스템 콜 흐름:

서버                                 클라이언트
socket() → fd 생성                   socket() → fd 생성
bind()   → 포트 할당
listen() → LISTEN 상태
         ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ connect() → SYN 전송
         ← SYN+ACK ← ─ ─ ─ ─ ─ ─ ─
         ← ─ ─ ─ ─ ─ ─ ─ ─ ACK ─ → ESTABLISHED
accept() → 새 fd 반환
read()   ← ─ ─ ─ ─ ─ ─ ─ DATA ─ ─  write()
write()  ─ ─ ─ DATA ─ ─ ─ ─ ─ ─ →  read()
close()  → FIN ─ ─ ─ ─ ─ ─ ─ ─ →  close()
```

### SO_REUSEADDR — 서버 재시작의 필수 옵션

서버를 종료하고 즉시 재시작하면 `bind()` 에러가 발생한다: "Address already in use". 이전 연결의 TIME_WAIT 소켓이 해당 포트를 점유하고 있기 때문이다.

```c
int opt = 1;
setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
// 이제 TIME_WAIT 소켓이 있어도 같은 포트에 bind 가능
```

이것은 서버 프로그래밍에서 **거의 항상** 설정해야 하는 옵션이다. Nginx, Apache, Node.js 등 모든 주요 서버 소프트웨어가 이 옵션을 사용한다.

### SO_REUSEPORT — 멀티코어 확장

리눅스 3.9에서 추가된 `SO_REUSEPORT`는 여러 프로세스가 같은 포트에 `bind()`할 수 있게 한다. 커널이 들어오는 연결을 프로세스들에게 분배한다.

```
SO_REUSEPORT 없이:              SO_REUSEPORT 사용:
┌────────────┐                  ┌────────────┐
│  프로세스 1  │ ← accept()     │  프로세스 1  │ ← accept() ← 커널이
│ (listen fd) │   경쟁 (thundering │ (listen fd) │              고르게
└────────────┘    herd 문제)    ├────────────┤              분배
                                │  프로세스 2  │ ← accept()
                                │ (listen fd) │
                                ├────────────┤
                                │  프로세스 3  │ ← accept()
                                │ (listen fd) │
                                └────────────┘
```

Nginx 1.9.1+에서 `reuseport` 옵션으로 사용 가능:
```nginx
server {
    listen 80 reuseport;
}
```

---

## 6. DNS 조회 — IP를 찾기까지

### DNS 해석 과정

```
브라우저/애플리케이션
    │
    ▼
로컬 캐시 (/etc/hosts, nscd, systemd-resolved)
    │ MISS
    ▼
Recursive Resolver (ISP 또는 8.8.8.8 등)
    │ MISS
    ▼
Root Name Server (.)
    │ ".com은 이쪽" → TLD 서버 주소
    ▼
TLD Name Server (.com)
    │ "example.com은 이쪽" → Authoritative 서버 주소
    ▼
Authoritative Name Server (example.com)
    │ "93.184.216.34" → A 레코드
    ▼
응답 반환 (TTL과 함께 캐시)
```

### 서버에서의 DNS 문제

**DNS 조회는 블로킹이다.** glibc의 `getaddrinfo()`는 동기 함수다. 서버가 외부 API를 호출할 때 DNS 조회가 느리면 전체 스레드가 멈춘다.

```bash
# DNS 조회 시간 측정
$ dig +stats example.com | grep "Query time"
;; Query time: 23 msec

# DNS 캐시 상태 확인 (systemd-resolved 사용 시)
$ resolvectl statistics
Current Transactions: 0
Total Transactions: 45678
  Current Cache Size: 234
          Cache Hits: 34567
        Cache Misses: 11111
```

서버 애플리케이션에서의 DNS 최적화:
- **로컬 캐시 DNS** 사용: `dnsmasq`, `systemd-resolved`, `unbound`
- **TTL 존중**: DNS 레코드의 TTL을 무시하고 무한 캐시하면 IP 변경 시 장애
- **DNS 프리페치**: 자주 사용하는 도메인을 미리 조회해두기
- **커넥션 풀**: DNS 조회 자체를 줄이는 가장 효과적인 방법

---

## 7. HTTP/1.1 요청 흐름

### TCP 위의 HTTP

HTTP/1.1은 TCP 연결 위에서 텍스트 기반 프로토콜로 동작한다:

```
TCP 연결 수립 (3-way handshake)
    │
    ▼
클라이언트 → 서버:
GET /api/users HTTP/1.1\r\n
Host: api.example.com\r\n
Connection: keep-alive\r\n
Accept: application/json\r\n
\r\n                          ← 빈 줄이 헤더 끝을 의미

    │
    ▼
서버 → 클라이언트:
HTTP/1.1 200 OK\r\n
Content-Type: application/json\r\n
Content-Length: 128\r\n
Connection: keep-alive\r\n
\r\n
{"users": [...]}              ← Content-Length만큼 읽으면 끝

    │
    ▼
같은 TCP 연결로 다음 요청... (Keep-Alive)
```

### Keep-Alive — 연결 재사용

HTTP/1.0에서는 요청마다 TCP 연결을 새로 맺었다. 매번 3-way handshake 비용이 발생한다.

HTTP/1.1에서는 **Keep-Alive가 기본**이다. 하나의 TCP 연결로 여러 요청을 순차적으로 처리한다.

```
HTTP/1.0 (Keep-Alive 없음):
TCP 연결 → 요청1 → 응답1 → TCP 종료
TCP 연결 → 요청2 → 응답2 → TCP 종료    각각 handshake 비용!
TCP 연결 → 요청3 → 응답3 → TCP 종료

HTTP/1.1 (Keep-Alive):
TCP 연결 → 요청1 → 응답1 → 요청2 → 응답2 → 요청3 → 응답3 → TCP 종료
           handshake는 한 번만!
```

서버 측 Keep-Alive 설정 (Nginx):
```nginx
http {
    keepalive_timeout 65;       # Keep-Alive 유지 시간
    keepalive_requests 1000;    # 하나의 연결로 처리할 최대 요청 수
}
```

### Head-of-Line Blocking 문제

HTTP/1.1의 근본적 한계: 하나의 TCP 연결에서 요청은 **순서대로** 처리된다. 앞선 요청이 느리면 뒤의 요청이 블록된다.

```
HTTP/1.1 파이프라이닝 (이론):
→ 요청1, 요청2, 요청3 을 연속 전송
← 응답은 반드시 1, 2, 3 순서로 와야 함

요청1 처리에 3초 걸리면:
[████████████ 요청1 처리 (3초) ████████████][요청2][요청3]
                                            ↑ 요청2,3은 대기

실제로는 대부분의 브라우저가 파이프라이닝을 비활성화하고,
대신 6개의 TCP 연결을 병렬로 맺어서 우회한다.
```

이 문제는 HTTP/2의 멀티플렉싱으로 해결된다 (6편에서 다룸).

---

## 8. 네트워크 진단 도구

### ss — 소켓 통계의 표준 도구

`netstat`의 현대적 대체. 커널의 `netlink` 인터페이스를 직접 사용해서 훨씬 빠르다.

```bash
# 모든 TCP 연결 (상태별)
$ ss -tan
State      Recv-Q  Send-Q  Local Address:Port   Peer Address:Port
LISTEN     0       128     0.0.0.0:80            0.0.0.0:*
ESTAB      0       0       10.0.1.5:80           203.0.113.1:54321
TIME-WAIT  0       0       10.0.1.5:80           203.0.113.2:54322

# 상태별 카운트
$ ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
   4523 ESTAB
   1890 TIME-WAIT
     89 CLOSE-WAIT
     12 LISTEN

# 특정 포트의 연결만
$ ss -tan 'sport = :80'

# 수신 큐가 0이 아닌 연결 (처리 지연 의심)
$ ss -tan 'recv-q > 0'

# 프로세스 정보 포함
$ ss -tanp
ESTAB  0  0  10.0.1.5:80  203.0.113.1:54321  users:(("nginx",pid=1234,fd=6))
```

**중요 컬럼 해석:**
- `Recv-Q` (LISTEN 상태): Accept Queue에 대기 중인 연결 수. 0이 아니면 `accept()`가 느린 것
- `Recv-Q` (ESTAB 상태): 소켓 수신 버퍼에서 애플리케이션이 아직 읽지 않은 데이터. 쌓이면 처리 지연
- `Send-Q` (ESTAB 상태): 전송했으나 ACK를 받지 못한 데이터. 쌓이면 네트워크 문제 또는 상대방 처리 지연

### CLOSE_WAIT 폭증 — 애플리케이션 버그의 징후

```bash
$ ss -tan | grep CLOSE-WAIT | wc -l
5678    # 비정상적으로 많음!
```

CLOSE_WAIT는 상대방이 FIN을 보냈는데 내가 아직 `close()`를 호출하지 않은 상태다. 이것이 쌓인다면 **애플리케이션 코드에서 소켓을 닫지 않는 버그**가 있다는 뜻이다.

```
정상: 상대 FIN → CLOSE_WAIT → close() → LAST_ACK → CLOSED
버그: 상대 FIN → CLOSE_WAIT → (close() 미호출) → 영원히 CLOSE_WAIT
```

### tcpdump — 패킷 레벨 분석

문제의 원인을 정확히 알려면 실제 패킷을 봐야 한다. tcpdump는 커널의 패킷 필터(BPF)를 사용해서 네트워크 인터페이스의 패킷을 캡처한다.

```bash
# 80번 포트의 TCP 패킷 캡처
$ sudo tcpdump -i eth0 port 80 -nn

# 특정 호스트와의 통신만
$ sudo tcpdump -i eth0 host 203.0.113.1 and port 80 -nn

# 파일로 저장 (나중에 Wireshark로 분석)
$ sudo tcpdump -i eth0 port 80 -w capture.pcap -c 1000

# SYN 패킷만 필터링 (새 연결 추적)
$ sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0' -nn

# 패킷 내용까지 출력 (ASCII)
$ sudo tcpdump -i eth0 port 80 -A -nn
```

**실전 진단 예시 — 3-way handshake 지연 확인:**

```bash
$ sudo tcpdump -i eth0 port 80 -nn -ttt
 00:00:00.000000 IP 203.0.113.1.54321 > 10.0.1.5.80: Flags [S], seq 1234
 00:00:00.000089 IP 10.0.1.5.80 > 203.0.113.1.54321: Flags [S.], seq 5678, ack 1235
 00:00:00.045123 IP 203.0.113.1.54321 > 10.0.1.5.80: Flags [.], ack 5679
#                                                          ↑ 45ms — 클라이언트 RTT

 00:00:00.045456 IP 203.0.113.1.54321 > 10.0.1.5.80: Flags [P.], seq 1235:1589
# HTTP 요청 전송

 00:00:00.312789 IP 10.0.1.5.80 > 203.0.113.1.54321: Flags [P.], seq 5679:6234
# 267ms 후 HTTP 응답 — 서버 처리 시간
```

### Wireshark — GUI 기반 심층 분석

tcpdump로 캡처한 `.pcap` 파일을 Wireshark에서 열면 훨씬 강력한 분석이 가능하다:

- **TCP Stream 재조립**: 흩어진 패킷을 하나의 대화로 재구성
- **RTT 그래프**: 시간에 따른 지연 변화 시각화
- **재전송 분석**: 패킷 유실과 재전송 패턴 파악
- **프로토콜 디코딩**: HTTP, TLS, DNS 등 상위 프로토콜 자동 파싱

```bash
# 서버에서 캡처 → 로컬로 복사 → Wireshark로 분석
$ sudo tcpdump -i eth0 port 80 -w /tmp/capture.pcap -c 5000
$ scp server:/tmp/capture.pcap .
$ wireshark capture.pcap
```

Wireshark 필수 필터:
```
tcp.analysis.retransmission     → 재전송 패킷
tcp.analysis.zero_window        → 수신 윈도우가 0 (수신측 처리 지연)
tcp.analysis.duplicate_ack      → 중복 ACK (패킷 유실 징후)
http.time > 1                   → 1초 이상 걸린 HTTP 응답
dns.time > 0.5                  → 500ms 이상 걸린 DNS 응답
```

---

## 9. 커널 네트워크 파라미터 튜닝

### 소켓 버퍼 크기

```bash
# TCP 수신 버퍼 (min / default / max)
$ sysctl net.ipv4.tcp_rmem
net.ipv4.tcp_rmem = 4096    131072    6291456
#                   최소     기본      최대(6MB)

# TCP 송신 버퍼
$ sysctl net.ipv4.tcp_wmem
net.ipv4.tcp_wmem = 4096    16384     4194304

# 시스템 전체 TCP 메모리 한도 (페이지 단위)
$ sysctl net.ipv4.tcp_mem
net.ipv4.tcp_mem = 383520   511360    767040
#                  최소      압박 시작  최대
```

소켓 버퍼가 너무 작으면 대역폭 활용이 안 된다. 이론적 최대 처리량은:

```
처리량 = 버퍼 크기 / RTT

예: 버퍼 128KB, RTT 50ms
→ 128KB / 0.05s = 2.56 MB/s = 20.5 Mbps

10Gbps 회선을 활용하려면:
버퍼 = 10Gbps × 50ms = 62.5 MB
```

### 연결 관련 파라미터

```bash
# 동시 연결 추적 테이블 크기 (conntrack/NAT 사용 시)
$ sysctl net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_max = 65536

# SYN 재전송 횟수 (연결 시도 시간 결정)
$ sysctl net.ipv4.tcp_syn_retries
net.ipv4.tcp_syn_retries = 6   # 약 127초까지 재시도

# FIN_WAIT_2 타임아웃
$ sysctl net.ipv4.tcp_fin_timeout
net.ipv4.tcp_fin_timeout = 60  # 서버에서 30으로 줄이기도
```

### 고성능 서버를 위한 권장 설정

```bash
# /etc/sysctl.d/99-server-tuning.conf

# TIME_WAIT 소켓 재사용
net.ipv4.tcp_tw_reuse = 1

# 로컬 포트 범위 확대
net.ipv4.ip_local_port_range = 10000 65535

# SYN Cookie 활성화 (SYN Flood 방어)
net.ipv4.tcp_syncookies = 1

# Accept Queue 크기
net.core.somaxconn = 65535

# SYN Queue 크기
net.ipv4.tcp_max_syn_backlog = 65535

# 소켓 버퍼 크기 (대역폭 × 지연 기반으로 조정)
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# FIN_WAIT_2 타임아웃 단축
net.ipv4.tcp_fin_timeout = 30
```

---

## 마치며

네트워크는 서버의 생명선이다. 이번 편에서 다룬 핵심을 정리하면:

- **3-way handshake**에서 커널은 SYN Queue와 Accept Queue를 통해 연결을 관리하며, 각 큐의 크기는 서버 수용력에 직접 영향을 준다
- **TIME_WAIT**는 TCP의 안전장치지만, 서버에서는 포트 고갈의 원인이 된다. Keep-Alive와 커넥션 풀이 가장 효과적인 대응이다
- **소켓 생명주기**를 이해하면 서버 프로그래밍의 구조가 보인다. `SO_REUSEADDR`는 사실상 필수다
- **ss**는 소켓 상태를, **tcpdump**는 패킷 레벨을, **Wireshark**는 프로토콜 레벨의 분석을 제공한다
- 커널 파라미터 튜닝은 `somaxconn`, `tcp_tw_reuse`, 소켓 버퍼 크기가 가장 영향이 크다

다음 편에서는 TCP를 더 깊이 파고든다 — 혼잡 제어(CUBIC, BBR), 흐름 제어, 슬라이딩 윈도우의 원리, 그리고 HTTP/2와 HTTP/3(QUIC)가 왜 등장했는지를 다룬다.
