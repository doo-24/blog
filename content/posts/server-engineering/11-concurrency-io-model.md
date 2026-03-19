---
title: "[서버 애플리케이션 설계] 3편 — 동시성과 I/O 모델: 서버가 수만 요청을 처리하는 방법"
date: 2026-03-18T00:07:00+09:00
draft: false
tags: ["동시성", "비동기", "epoll", "이벤트루프", "서버", "코루틴"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "Blocking/Non-blocking과 동기/비동기의 정확한 구분, epoll과 이벤트 루프, 리액터 패턴, 스레드풀 설계, 가상 스레드(Project Loom), 코루틴까지."
---

## 들어가며

서버의 성능은 결국 **동시에 몇 개의 요청을 처리할 수 있느냐**에 달린다.

이것을 결정하는 핵심이 I/O 모델과 동시성 전략이다.

"비동기가 빠르다"는 알고 있지만, 정확히 **왜** 빠른지, **어떤 상황에서** 동기가 더 나은지, 이벤트 루프와 스레드풀의 차이는 무엇인지 — 이것들을 명확히 구분해야 올바른 아키텍처를 선택할 수 있다.

---

## 1. 네 가지 I/O 모델

### Blocking vs Non-blocking, 동기 vs 비동기

이 네 개념은 자주 혼동된다. 정확히 구분하자.

```
Blocking / Non-blocking: I/O 호출이 완료될 때까지 호출자가 멈추는가?
- Blocking:     read() 호출 → 데이터가 올 때까지 스레드 정지
- Non-blocking:  read() 호출 → 데이터가 없으면 즉시 반환 (EAGAIN)

동기(Synchronous) / 비동기(Asynchronous): I/O 완료를 누가 확인하는가?
- 동기:   호출자가 직접 완료를 확인 (polling 또는 blocking)
- 비동기:  커널이 완료 시 알려줌 (콜백, 시그널, completion event)
```

```
                  Blocking              Non-blocking
동기        ┌──────────────────┐  ┌──────────────────┐
            │ 가장 단순          │  │ 폴링 방식          │
            │ read()가 블록     │  │ read() → EAGAIN   │
            │ 스레드 낭비        │  │ → 다시 read()     │
            │                  │  │ → EAGAIN → ...    │
            │ (전통적 서버)     │  │ (CPU 낭비)        │
            └──────────────────┘  └──────────────────┘

비동기      ┌──────────────────┐  ┌──────────────────┐
            │ I/O 멀티플렉싱    │  │ 진짜 비동기 I/O    │
            │ select/poll/epoll│  │ io_uring, IOCP    │
            │ "준비됐나?"만 블록 │  │ 커널이 완료 통지   │
            │ → 준비된 fd 처리  │  │ (가장 효율적)      │
            │ (Nginx, Node.js) │  │ (리눅스 5.1+)     │
            └──────────────────┘  └──────────────────┘
```

### 실제 서버에서의 선택

대부분의 고성능 서버가 사용하는 것은 **I/O 멀티플렉싱 + Non-blocking I/O**다.

"비동기"라고 부르지만, 엄밀히는 **동기 Non-blocking + 이벤트 통지**에 가깝다.

```
Nginx, Node.js, Redis의 모델:
1. 소켓을 non-blocking으로 설정
2. epoll에 소켓을 등록
3. epoll_wait()로 "준비된" 소켓을 알려받음
4. 준비된 소켓에서 non-blocking read/write
5. 반복
```

---

## 2. I/O 멀티플렉싱의 진화

### select (1983)

```c
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(fd1, &readfds);
FD_SET(fd2, &readfds);

select(max_fd + 1, &readfds, NULL, NULL, &timeout);

// 어떤 fd가 준비됐는지 전체를 순회해서 확인
for (int fd = 0; fd < max_fd; fd++) {
    if (FD_ISSET(fd, &readfds)) {
        // fd 처리
    }
}
```

**한계:**
- fd 집합 크기 제한 (`FD_SETSIZE`, 보통 1024)
- 매번 전체 fd 집합을 커널에 복사
- 준비된 fd를 찾으려면 전체를 순회 → O(n)

### poll (1986)

```c
struct pollfd fds[2];
fds[0] = { .fd = fd1, .events = POLLIN };
fds[1] = { .fd = fd2, .events = POLLIN };

poll(fds, 2, timeout);

for (int i = 0; i < 2; i++) {
    if (fds[i].revents & POLLIN) {
        // fds[i].fd 처리
    }
}
```

select의 fd 수 제한을 없앴지만, 여전히 매번 전체 fd 배열을 커널에 복사하고, 전체를 순회해야 한다.

### epoll (리눅스 2.6, 2002)

```c
// 1. epoll 인스턴스 생성 (한 번만)
int epfd = epoll_create1(0);

// 2. 관심 있는 fd 등록 (한 번만)
struct epoll_event ev = { .events = EPOLLIN, .data.fd = fd1 };
epoll_ctl(epfd, EPOLL_CTL_ADD, fd1, &ev);

// 3. 준비된 이벤트만 받아옴 (반복)
struct epoll_event events[MAX_EVENTS];
int n = epoll_wait(epfd, events, MAX_EVENTS, timeout);

for (int i = 0; i < n; i++) {
    // events[i].data.fd — 준비된 fd만 반환!
    handle(events[i].data.fd);
}
```

**epoll이 빠른 이유:**

```
select/poll:
매 호출마다:
유저 공간 → [fd 집합 전체 복사] → 커널 → [전체 fd 검사] → 결과

epoll:
최초 등록: epoll_ctl(ADD) — fd를 커널 내부 레드-블랙 트리에 등록
이후 호출: epoll_wait — 준비된 fd 목록만 반환 (ready list)

fd 10,000개, 준비된 fd 10개:
select: O(10,000) — 전체 순회
epoll:  O(10) — 준비된 것만 반환
```

### Edge-Triggered vs Level-Triggered

```
Level-Triggered (LT, 기본):
- 읽을 데이터가 있으면 매번 알림
- 한 번에 다 안 읽어도 다음 epoll_wait에서 다시 알려줌
- 프로그래밍이 쉬움

Edge-Triggered (ET):
- 상태가 "변할 때만" 알림
- 데이터가 도착하면 한 번만 알려줌
- 반드시 EAGAIN이 나올 때까지 다 읽어야 함
- 놓치면 영원히 알림 안 옴!
- 더 효율적이지만 구현 주의 필요

Nginx: ET 모드 사용 (성능 최적화)
Redis: LT 모드 사용 (단순성)
```

---

## 3. 이벤트 루프와 리액터 패턴

### 리액터 패턴 (Reactor Pattern)

I/O 멀티플렉싱을 구조화한 것이 리액터 패턴이다.

Node.js, Nginx, Redis, Netty의 핵심 아키텍처다.

```
┌─────────────────────────────────────────────┐
│                 리액터 (이벤트 루프)            │
│                                             │
│  epoll_wait() ──→ 이벤트 디스패치           │
│       │                    │                │
│       ▼                    ▼                │
│  ┌─────────┐    ┌──────────────────┐       │
│  │ Accept  │    │ Read/Write       │       │
│  │ Handler │    │ Handler          │       │
│  └────┬────┘    └────────┬─────────┘       │
│       │                  │                 │
│  새 연결 등록        요청 처리 + 응답         │
│  (epoll_ctl ADD)                           │
└─────────────────────────────────────────────┘
```

```
이벤트 루프의 한 사이클:

while (running) {
    events = epoll_wait(...)         // 1. 준비된 이벤트 수집
    for (event in events) {
        if (event == NEW_CONNECTION)
            accept_and_register()    // 2. 새 연결 수락
        else if (event == READABLE)
            read_and_process()       // 3. 데이터 읽고 처리
        else if (event == WRITABLE)
            write_response()         // 4. 응답 전송
    }
    run_timers()                     // 5. 타이머 콜백 실행
    run_deferred_tasks()             // 6. 지연 작업 실행
}
```

### 싱글 스레드 이벤트 루프의 장단점

**Node.js, Redis 방식:**

```
장점:
- 락이 필요 없음 → 동시성 버그 원천 차단
- 컨텍스트 스위칭 비용 없음
- 메모리 효율 (스레드당 수 MB 스택 불필요)

단점:
- CPU 집약적 작업이 이벤트 루프를 블록
- 멀티코어 활용 불가 (프로세스를 여러 개 띄워야)
```

```
Node.js에서 CPU 블록 예시:

app.get('/hash', (req, res) => {
    // 이 동안 다른 모든 요청이 대기!
    const hash = crypto.pbkdf2Sync(password, salt, 100000, 64, 'sha512');
    res.send(hash);
});

해결: Worker Thread 또는 별도 프로세스로 위임
```

### 멀티 리액터 — Netty, Nginx 방식

```
Boss Group (Acceptor):
└── EventLoop 1개 → accept()만 담당

Worker Group:
├── EventLoop 1 → 연결 1000개 처리 (epoll)
├── EventLoop 2 → 연결 1000개 처리 (epoll)
├── EventLoop 3 → 연결 1000개 처리 (epoll)
└── EventLoop 4 → 연결 1000개 처리 (epoll)

각 EventLoop = 1 스레드 + 1 epoll 인스턴스
→ 멀티코어 활용 + 이벤트 기반 효율
```

Nginx의 `worker_processes auto;`가 바로 이 구조다.

CPU 코어 수만큼 워커 프로세스를 생성하고, 각 워커가 독립적인 이벤트 루프를 운영한다.

---

## 4. 스레드풀 설계

### 왜 스레드풀인가

모든 요청에 새 스레드를 생성하면:
- 스레드 생성/소멸 비용 (수백 마이크로초)
- 메모리 소모 (스레드당 스택 1MB)
- 무제한 스레드 생성 → OOM

스레드풀은 미리 만들어둔 스레드를 재사용한다.

```
요청 → [작업 큐] → [스레드 풀]
                    ├── 스레드 1 (작업 처리 중)
                    ├── 스레드 2 (대기)
                    ├── 스레드 3 (작업 처리 중)
                    └── 스레드 4 (대기)

새 요청 도착 → 큐에 넣음 → 대기 스레드가 가져감
```

### Java ThreadPoolExecutor 핵심 파라미터

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                              // corePoolSize: 항상 유지하는 스레드 수
    50,                              // maximumPoolSize: 최대 스레드 수
    60L, TimeUnit.SECONDS,           // keepAliveTime: 초과 스레드 유휴 시간
    new LinkedBlockingQueue<>(1000), // workQueue: 대기 큐 (용량 제한!)
    new ThreadPoolExecutor.CallerRunsPolicy()  // 거부 정책
);
```

**동작 순서:**
```
1. 요청 도착, 스레드 < corePoolSize  → 새 스레드 생성
2. 요청 도착, 스레드 >= corePoolSize → 큐에 대기
3. 큐 가득 참, 스레드 < maxPoolSize → 새 스레드 생성
4. 큐 가득 참, 스레드 >= maxPoolSize → 거부 정책 실행
```

**거부 정책 선택:**
```
AbortPolicy (기본):     RejectedExecutionException 던짐
CallerRunsPolicy:       호출 스레드가 직접 실행 (자연스러운 백프레셔)
DiscardPolicy:          조용히 버림 (위험!)
DiscardOldestPolicy:    큐의 가장 오래된 작업을 버리고 새 작업 추가
```

### 스레드 수 결정 공식

```
CPU 집약적 작업:
적정 스레드 수 = CPU 코어 수 + 1

I/O 집약적 작업 (대부분의 서버):
적정 스레드 수 = CPU 코어 수 × (1 + I/O 대기 시간 / CPU 처리 시간)

예: 4코어, 요청당 CPU 10ms + DB 대기 90ms
→ 4 × (1 + 90/10) = 40 스레드

예: 4코어, 요청당 CPU 50ms + 외부 API 200ms
→ 4 × (1 + 200/50) = 20 스레드
```

**주의**: 이 공식은 출발점일 뿐이다. 실제로는 부하 테스트로 최적값을 찾아야 한다.

### Spring Boot의 스레드 모델

```yaml
# application.yml
server:
  tomcat:
    threads:
      min-spare: 10    # 최소 유지 스레드
      max: 200         # 최대 스레드 (기본 200)
    max-connections: 8192
    accept-count: 100  # 연결 대기 큐 (TCP backlog)
```

```
요청 흐름:
클라이언트 → [Accept Queue (100)] → [Connection Pool (8192)]
              → [Thread Pool (200)] → Controller → Service → Repository

200개 스레드가 모두 바쁘면?
→ 새 요청은 큐에서 대기
→ 큐도 가득 차면? → 연결 거부
```

---

## 5. 가상 스레드 (Project Loom)

### 기존 스레드의 문제

```
Java 스레드 = OS 스레드 (1:1 매핑)
- 스택 메모리: ~1MB
- 생성 비용: 수백 마이크로초
- 컨텍스트 스위칭: 수 마이크로초

스레드 10,000개 = 10GB 스택 메모리
→ 동시 연결이 많은 서버에서 한계
```

### 가상 스레드 (Java 21+)

```java
// 기존 플랫폼 스레드
Thread platformThread = new Thread(() -> {
    // OS 스레드 1개 점유
    callDatabase();  // 블로킹 I/O 동안 OS 스레드 낭비
});

// 가상 스레드 (Java 21)
Thread virtualThread = Thread.ofVirtual().start(() -> {
    // OS 스레드를 점유하지 않음
    callDatabase();  // 블로킹 시 자동으로 OS 스레드 반환!
});
```

```
가상 스레드 동작 원리:

가상 스레드 1 ──mount──→ 캐리어 스레드 (OS 스레드 A)
                         │ 실행 중...
                         │ I/O 블로킹 발생!
가상 스레드 1 ──unmount──│ (자동으로 분리됨)
가상 스레드 2 ──mount──→ 캐리어 스레드 (OS 스레드 A)  ← 같은 OS 스레드 재활용!
                         │ 실행 중...
                         │
I/O 완료 ──→ 가상 스레드 1이 캐리어에 다시 mount

핵심: I/O 대기 중에 OS 스레드를 다른 가상 스레드가 사용
→ 적은 OS 스레드로 수만 가상 스레드 실행 가능
```

**가상 스레드의 장점:**
```
- 스택 메모리: 수 KB (필요에 따라 성장)
- 100만 개 가상 스레드도 가능
- 기존 블로킹 코드를 수정 없이 사용
- Thread-per-request 모델의 단순함 + 이벤트 기반의 효율
```

### Spring Boot + 가상 스레드

```yaml
# application.yml (Spring Boot 3.2+)
spring:
  threads:
    virtual:
      enabled: true
```

이 한 줄로 Tomcat이 가상 스레드를 사용한다.

요청마다 가상 스레드가 할당되고, JDBC 호출에서 블로킹되어도 OS 스레드를 낭비하지 않는다.

### 가상 스레드 주의사항

```java
// 1. synchronized 블록 내에서 I/O하면 캐리어 스레드가 고정됨 (pinning)
synchronized (lock) {
    database.query(...);  // 가상 스레드가 OS 스레드에 고정!
}
// 해결: ReentrantLock 사용
lock.lock();
try {
    database.query(...);  // 정상적으로 unmount 가능
} finally {
    lock.unlock();
}

// 2. 스레드 풀에 가상 스레드를 넣지 마라
// 가상 스레드는 풀링할 필요 없음 — 생성이 매우 가볍기 때문
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
// 요청마다 새 가상 스레드 생성 (정상)

// 3. ThreadLocal 주의
// 가상 스레드가 수만 개이면 ThreadLocal도 수만 개 → 메모리 이슈
// ScopedValue (Java 21 preview)로 대체 권장
```

---

## 6. 코루틴 개요

### 코루틴의 핵심 아이디어

코루틴은 **중단 가능한 함수**다.

실행을 일시 중지하고 나중에 이어갈 수 있다.

```kotlin
// Kotlin 코루틴
suspend fun fetchUserData(userId: Long): UserData {
    val user = userApi.getUser(userId)      // 중단점 1: API 호출 동안 스레드 반환
    val orders = orderApi.getOrders(userId)  // 중단점 2: 또 다른 API 호출
    return UserData(user, orders)
}
```

```
코루틴 실행 흐름:

스레드 A: [코루틴1 실행] → [API 호출, 중단] → [코루틴2 실행] → [코루틴2 중단]
스레드 A: [코루틴1 재개] → [완료]

→ 가상 스레드와 개념적으로 비슷하지만:
   - 가상 스레드: JVM 레벨, 기존 블로킹 API 자동 지원
   - 코루틴: 언어 레벨, suspend 키워드로 명시적 표시
```

### 코루틴의 구조적 동시성

```kotlin
// 병렬 실행 + 구조적 동시성
suspend fun fetchDashboard(userId: Long): Dashboard = coroutineScope {
    // 세 API를 동시에 호출
    val userDeferred = async { userApi.getUser(userId) }
    val ordersDeferred = async { orderApi.getOrders(userId) }
    val statsDeferred = async { statsApi.getStats(userId) }

    // 모두 완료될 때까지 대기
    Dashboard(
        user = userDeferred.await(),
        orders = ordersDeferred.await(),
        stats = statsDeferred.await()
    )
    // 하나라도 실패하면 나머지 자동 취소 (구조적 동시성)
}
```

---

## 7. 모델 선택 가이드

```
상황별 권장 모델:

전통적 웹 서버 (Spring MVC):
→ 스레드풀 + 블로킹 I/O
→ Java 21+이면 가상 스레드 활성화
→ 가장 단순하고 디버깅 쉬움

고성능 API 게이트웨이:
→ 이벤트 루프 (Netty, Nginx)
→ 논블로킹 I/O

실시간 서비스 (채팅, 게임):
→ 이벤트 루프 + WebSocket
→ Netty 또는 Node.js

마이크로서비스 간 통신:
→ 리액티브 (WebFlux) 또는 가상 스레드
→ 외부 호출이 많으면 비동기의 이점이 큼

CPU 집약적 작업:
→ 스레드풀 (코어 수만큼)
→ 이벤트 루프는 부적합 (블록됨)

Kotlin 서비스:
→ 코루틴 (언어 네이티브 지원)
→ Spring WebFlux + 코루틴 조합
```

```
처리량 비교 (동일 하드웨어, I/O 중심 워크로드):

모델                        동시 연결     메모리     복잡도
───────────────────────────────────────────────────────
스레드-per-request (200개)   ~200         높음       낮음
스레드풀 + NIO              ~10,000      중간       높음
이벤트 루프 (Netty)          ~100,000     낮음       높음
가상 스레드                  ~100,000     낮음       낮음 ★
```

---

## 마치며

동시성과 I/O 모델은 서버 아키텍처의 근간이다.

- **Blocking/Non-blocking**은 호출의 반환 시점을, **동기/비동기**는 완료 통지 방식을 구분한다
- **epoll**은 O(n) 순회를 O(준비된 fd)로 줄여서 수만 연결을 효율적으로 관리한다
- **리액터 패턴(이벤트 루프)**은 적은 스레드로 높은 동시성을 달성하지만, CPU 블로킹에 취약하다
- **스레드풀**은 이해하기 쉽고 디버깅이 용이하지만, I/O 대기 시 스레드를 낭비한다
- **가상 스레드(Java 21)**는 "블로킹 코드의 단순함 + 비동기의 효율"을 결합한 혁신이다
- **코루틴**은 언어 레벨에서 동일한 문제를 해결하며, Kotlin 생태계에서 표준이다

가장 중요한 판단 기준: **팀이 유지보수할 수 있는 복잡도** 안에서 가장 효율적인 모델을 선택하라.

---

## 참고 자료

- *UNIX Network Programming, Volume 1, 3rd Edition* — W. Richard Stevens
- Linux `epoll(7)` man page — [https://man7.org/linux/man-pages/man7/epoll.7.html](https://man7.org/linux/man-pages/man7/epoll.7.html)
- JEP 444: Virtual Threads — [https://openjdk.org/jeps/444](https://openjdk.org/jeps/444)
- Kotlin Coroutines Guide — [https://kotlinlang.org/docs/coroutines-guide.html](https://kotlinlang.org/docs/coroutines-guide.html)
- Netty Documentation — [https://netty.io/wiki/](https://netty.io/wiki/)
- Brendan Gregg, *Systems Performance, 2nd Edition* — Chapter 5: Applications
