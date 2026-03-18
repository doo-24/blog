---
title: "[OS와 네트워크] 1편 — 리눅스와 프로세스: 서버 안에서 무슨 일이 일어나는가"
date: 2026-03-18
draft: false
tags: ["리눅스", "프로세스", "OS", "서버"]
series: ["OS와 네트워크"]
summary: "커널과 유저 공간의 경계, 프로세스와 스레드의 본질적 차이, fork/exec 메커니즘, 시그널, 좀비 프로세스, /proc 파일시스템, strace/lsof 실전 활용까지"
---

서버에 SSH로 접속해서 `java -jar app.jar`을 실행하는 순간, OS 안에서는 수십 가지 일이 벌어진다. 메모리가 할당되고, CPU 스케줄링 큐에 등록되고, 파일 디스크립터 테이블이 만들어지고, 네트워크 소켓이 열린다.

이 과정을 모르면 서버 장애 앞에서 무력해진다. "왜 프로세스가 죽었는지", "왜 CPU가 100%인지", "왜 메모리가 부족한지"를 시스템 레벨에서 읽어내지 못하기 때문이다.

이 글은 리눅스 프로세스의 전체 생명주기를 다룬다.

---

## 1. 커널 공간과 유저 공간

### 두 세계의 분리

리눅스는 메모리를 두 영역으로 나눈다.

**커널 공간(Kernel Space)** — OS 커널이 동작하는 영역이다. 하드웨어에 직접 접근할 수 있고, 모든 메모리를 읽고 쓸 수 있다. 프로세스 스케줄링, 메모리 관리, 디바이스 드라이버, 네트워크 스택이 여기서 실행된다.

**유저 공간(User Space)** — 일반 애플리케이션이 동작하는 영역이다. Spring Boot 앱, Nginx, PostgreSQL 모두 유저 공간에서 실행된다. 하드웨어에 직접 접근할 수 없고, 할당받은 가상 메모리만 사용할 수 있다.

```
┌──────────────────────────────────────────┐
│              유저 공간 (User Space)        │
│                                          │
│   java (Spring Boot)    nginx    redis   │
│                                          │
├────────────── 시스템 콜 인터페이스 ──────────┤
│                                          │
│              커널 공간 (Kernel Space)      │
│                                          │
│   프로세스 관리  │  메모리 관리  │  파일시스템  │
│   네트워크 스택  │  디바이스 드라이버         │
│                                          │
├──────────────────────────────────────────┤
│              하드웨어                      │
│   CPU  │  RAM  │  Disk  │  NIC           │
└──────────────────────────────────────────┘
```

이 분리는 CPU의 보호 링(Protection Ring) 메커니즘으로 강제된다. x86 CPU는 Ring 0(커널)부터 Ring 3(유저)까지 4개의 권한 레벨을 제공하는데, 리눅스는 Ring 0과 Ring 3만 사용한다. 유저 공간의 코드가 커널 메모리에 접근하려 하면 CPU가 예외(exception)를 발생시키고, 커널이 해당 프로세스를 종료한다.

### 시스템 콜 — 두 세계를 잇는 다리

유저 공간의 프로세스가 파일을 읽거나, 네트워크 패킷을 보내거나, 새 프로세스를 만들려면 커널의 도움이 필요하다. 이때 사용하는 것이 **시스템 콜(System Call)**이다.

시스템 콜이 호출되면 CPU는 유저 모드에서 커널 모드로 전환된다. 이 전환에는 비용이 따른다. 레지스터를 저장하고, 커널 스택으로 전환하고, 요청을 처리한 뒤 다시 유저 모드로 돌아온다.

자주 사용되는 시스템 콜:

| 시스템 콜 | 하는 일 | 사용 예시 |
|-----------|---------|-----------|
| `read()` / `write()` | 파일/소켓 읽기/쓰기 | HTTP 응답 전송, 로그 기록 |
| `open()` / `close()` | 파일 디스크립터 열기/닫기 | 설정 파일 로드 |
| `fork()` / `exec()` | 프로세스 생성/교체 | 서비스 시작 |
| `socket()` / `bind()` / `listen()` | 네트워크 소켓 생성 | 서버 포트 리스닝 |
| `mmap()` | 메모리 매핑 | 파일을 메모리에 매핑 |
| `epoll_wait()` | I/O 이벤트 대기 | Nginx 이벤트 루프 |

Java의 `Files.readAllBytes()`를 호출하면, JVM 내부에서 `read()` 시스템 콜이 발생한다. 이 사실을 직접 확인하는 방법이 `strace`다. 뒤에서 다룬다.

### 시스템 콜의 비용

시스템 콜 하나의 오버헤드는 수백 나노초~수 마이크로초 수준이다. 개별로는 작지만, 초당 수만 번 호출되면 누적 비용이 커진다.

이것이 Buffered I/O가 존재하는 이유다. 매번 `write()` 시스템 콜로 1바이트씩 쓰는 대신, 유저 공간의 버퍼에 모아뒀다가 한 번에 `write()`를 호출한다. Java의 `BufferedOutputStream`이 정확히 이 역할이다.

```java
// 나쁜 예 — write() 시스템 콜이 100만 번 발생
for (int i = 0; i < 1_000_000; i++) {
    outputStream.write(data[i]);
}

// 좋은 예 — 버퍼링으로 시스템 콜 횟수 대폭 감소
BufferedOutputStream bos = new BufferedOutputStream(outputStream, 8192);
for (int i = 0; i < 1_000_000; i++) {
    bos.write(data[i]);
}
bos.flush();
```

---

## 2. 프로세스의 정체

### 프로세스란 무엇인가

프로세스는 실행 중인 프로그램의 인스턴스다. 디스크에 있는 `/usr/bin/java`는 프로그램이고, `java -jar app.jar`로 실행한 것이 프로세스다.

커널은 각 프로세스에 대해 `task_struct`라는 자료구조를 만든다. 여기에 프로세스의 모든 메타데이터가 들어간다.

```
task_struct (커널 내부 자료구조)
├── pid: 12345                          ← 프로세스 ID
├── state: TASK_RUNNING                 ← 현재 상태
├── mm: (메모리 디스크립터)                ← 가상 메모리 매핑 정보
│   ├── code segment (텍스트)
│   ├── data segment (전역/정적 변수)
│   ├── heap (동적 할당)
│   └── stack (함수 호출 스택)
├── files: (파일 디스크립터 테이블)         ← 열린 파일/소켓 목록
├── signal: (시그널 핸들러 테이블)
├── parent: (부모 프로세스 포인터)
├── children: (자식 프로세스 리스트)
├── cred: (uid, gid, 권한 정보)
├── comm: "java"                        ← 프로세스 이름
└── ... (스케줄링 정보, cgroup, namespace 등)
```

### 프로세스의 메모리 레이아웃

각 프로세스는 독립된 가상 주소 공간을 갖는다. 64비트 리눅스에서 유저 공간은 0 ~ 0x7FFFFFFFFFFF(128TB)이다.

```
높은 주소
┌─────────────────────┐ 0x7FFFFFFFFFFF
│       스택 (Stack)    │  ← 함수 호출마다 아래로 성장
│          ↓           │     지역 변수, 리턴 주소
│                      │
│       (빈 공간)       │  ← 스택과 힙 사이의 여유 공간
│                      │
│          ↑           │
│       힙 (Heap)       │  ← malloc/new로 위로 성장
│                      │     Java 객체가 여기에 할당됨
├─────────────────────┤
│   메모리 맵 영역      │  ← 공유 라이브러리, mmap 파일
├─────────────────────┤
│   BSS (초기화 안 된    │  ← 0으로 초기화된 전역 변수
│        전역 변수)     │
├─────────────────────┤
│   Data (초기화된      │  ← 초기값이 있는 전역/정적 변수
│        전역 변수)     │
├─────────────────────┤
│   Text (코드)        │  ← 실행 코드 (읽기 전용)
└─────────────────────┘ 0x00000000
낮은 주소
```

`/proc/<pid>/maps`에서 실제 매핑을 확인할 수 있다.

```bash
$ cat /proc/12345/maps | head -10
00400000-00452000 r-xp 00000000 08:01 131077  /usr/bin/java   ← Text (실행 코드)
00651000-00652000 r--p 00051000 08:01 131077  /usr/bin/java   ← Data (읽기 전용)
00652000-00653000 rw-p 00052000 08:01 131077  /usr/bin/java   ← Data (읽기/쓰기)
7f1a2c000000-7f1a30000000 rw-p 00000000 00:00 0               ← 힙 영역
7ffdd4e00000-7ffdd4e21000 rw-p 00000000 00:00 0  [stack]      ← 스택
```

권한 표기 `r-xp`는 읽기(r), 실행(x), 쓰기 불가(-), 프라이빗(p)을 의미한다. 코드 영역이 쓰기 불가인 것은 보안을 위해서다.

---

## 3. 프로세스 vs 스레드

### 본질적 차이

프로세스와 스레드의 차이는 딱 하나다: **주소 공간을 공유하는가**.

프로세스는 독립된 주소 공간을 갖는다. 프로세스 A의 메모리 주소 0x1000에 있는 데이터와 프로세스 B의 0x1000에 있는 데이터는 완전히 다른 물리 메모리를 가리킨다.

스레드는 같은 프로세스 내에서 주소 공간을 공유한다. 힙, 데이터 영역, 코드 영역, 파일 디스크립터 테이블을 모두 공유하고, 각 스레드가 독립적으로 갖는 것은 **스택**과 **레지스터 상태**뿐이다.

```
프로세스 A (PID 100)              프로세스 B (PID 200)
┌───────────────────┐            ┌───────────────────┐
│ 스레드1    스레드2  │            │ 스레드1            │
│ [스택1]   [스택2]  │            │ [스택1]            │
│                   │            │                   │
│ ──── 공유 영역 ─── │            │ ──── 공유 영역 ─── │
│ 힙                │            │ 힙                │
│ 전역 변수          │            │ 전역 변수          │
│ 코드              │            │ 코드              │
│ fd 테이블          │            │ fd 테이블          │
└───────────────────┘            └───────────────────┘
      격리됨 ◄─────────────────────────► 격리됨
```

리눅스 커널의 관점에서 스레드와 프로세스의 구분은 모호하다. 둘 다 `task_struct`로 표현되며, 스레드는 `clone()` 시스템 콜에 `CLONE_VM` 플래그를 주어 생성한 것일 뿐이다. 이 플래그가 주소 공간 공유를 의미한다.

### 왜 이게 서버 개발에서 중요한가

**Spring Boot(Tomcat)**는 요청마다 스레드를 할당한다. 200개의 동시 요청이면 200개의 스레드가 같은 힙을 공유하며 동작한다. 같은 Service 빈, 같은 커넥션 풀, 같은 캐시에 동시에 접근한다. 이것이 `synchronized`, `ConcurrentHashMap`, `AtomicInteger`가 필요한 근본 이유다.

**Nginx**는 다른 전략을 쓴다. master 프로세스가 여러 worker 프로세스를 fork한다. 각 worker는 독립된 주소 공간을 가지므로, 한 worker가 segfault로 죽어도 다른 worker에 영향이 없다. 대신 worker 간 데이터 공유는 공유 메모리(shared memory zone)를 명시적으로 설정해야 한다.

**Node.js**는 싱글 스레드 이벤트 루프다. 동시성 문제가 원천적으로 없지만, CPU 집약적인 작업이 이벤트 루프를 블로킹하면 모든 요청이 멈춘다. `cluster` 모듈로 멀티 프로세스를 사용하거나, `worker_threads`로 별도 스레드를 쓴다.

### 컨텍스트 스위칭 비용

CPU 코어 하나는 한 번에 하나의 실행 흐름만 처리한다. 여러 스레드/프로세스가 번갈아 실행되려면 전환이 필요하고, 이 전환에 비용이 든다.

**스레드 간 전환**: 레지스터와 스택 포인터만 교체하면 된다. 같은 주소 공간이므로 TLB(Translation Lookaside Buffer)를 비울 필요가 없다. 약 1~10 마이크로초.

**프로세스 간 전환**: 레지스터 교체에 더해 TLB를 플러시해야 한다. 새 프로세스의 페이지 테이블로 전환하면서 캐시 미스가 급증한다. 약 10~100 마이크로초.

이것이 Tomcat의 스레드 풀 크기를 CPU 코어 수와 맞춰 설정하는 이유다. 200개 스레드가 4코어 CPU에서 돌면, 196개는 항상 대기 중이고, 전환 비용만 쌓인다. 다만 I/O 대기가 많은 웹 서버에서는 스레드 수가 코어 수보다 많아도 괜찮다. I/O 대기 중인 스레드는 CPU를 쓰지 않기 때문이다.

---

## 4. fork와 exec — 프로세스의 탄생

### fork()

리눅스에서 새 프로세스를 만드는 유일한 방법이다. 현재 프로세스를 **복제**한다.

```
fork() 전:
  bash (PID 100, PPID 1)

fork() 후:
  bash (PID 100, PPID 1)    ← 부모: fork() 리턴값 = 자식 PID (101)
  bash (PID 101, PPID 100)  ← 자식: fork() 리턴값 = 0
```

자식은 부모의 모든 것을 복제받는다. 메모리, 파일 디스크립터, 환경 변수, 시그널 핸들러. 단, PID와 PPID(부모 PID)는 다르다.

**Copy-on-Write(COW)**: 실제로 메모리를 전부 복사하면 느리다. 리눅스는 fork 직후 부모와 자식이 같은 물리 페이지를 공유하게 한다. 어느 한쪽이 페이지를 **수정하는 순간**에만 해당 페이지를 복사한다. 읽기만 하면 복사가 일어나지 않으므로, fork 자체는 매우 빠르다.

Redis의 `BGSAVE`가 이 원리를 활용한다. fork로 자식을 만들고, 자식이 메모리의 스냅샷을 디스크에 쓴다. 부모는 계속 요청을 처리한다. COW 덕분에 메모리가 두 배 필요하지 않다 (변경된 페이지만 복사됨).

### exec()

fork로 만든 자식은 부모와 같은 프로그램을 실행 중이다. 다른 프로그램을 실행하려면 `exec()`을 호출한다. exec는 현재 프로세스의 메모리를 새 프로그램으로 **교체**한다. PID는 유지된다.

```
bash(PID 100)
  └── fork() → bash(PID 101)
                 └── exec("java -jar app.jar") → java(PID 101)
```

셸에서 명령어를 실행하는 모든 과정이 fork + exec다.

### exec 형태와 Docker의 PID 1 문제

Docker 컨테이너에서 이 메커니즘이 실질적으로 중요하다.

```dockerfile
# Shell form — /bin/sh -c "java -jar app.jar"로 실행됨
CMD java -jar app.jar
# 프로세스 트리:
#   PID 1: /bin/sh -c "java -jar app.jar"
#   PID 7: java -jar app.jar

# Exec form — java가 직접 PID 1이 됨
CMD ["java", "-jar", "app.jar"]
# 프로세스 트리:
#   PID 1: java -jar app.jar
```

Shell form을 쓰면 `sh`가 PID 1이 된다. Docker는 컨테이너 종료 시 PID 1에 SIGTERM을 보내는데, `sh`는 기본적으로 SIGTERM을 자식에게 전달하지 않는다. 결과적으로 Java 프로세스가 Graceful Shutdown을 할 기회를 얻지 못하고, timeout 후 SIGKILL로 강제 종료된다.

---

## 5. 프로세스 상태와 좀비

### 프로세스의 상태 전이

```
                   fork()
                     │
                     ▼
              ┌─────────────┐
              │  생성 (New)   │
              └──────┬──────┘
                     │ 스케줄러에 의해 큐에 등록
                     ▼
              ┌─────────────┐    CPU 할당     ┌─────────────┐
              │ 준비 (Ready) │ ─────────────→ │ 실행 (Running)│
              │     (R)      │ ←───────────── │     (R)      │
              └──────┬──────┘  타임슬라이스    └──────┬──────┘
                     │         만료                   │
                     │                               │
              I/O 완료                          I/O 요청 / sleep
                     │                               │
                     │         ┌─────────────┐       │
                     └──────── │ 대기 (Sleep) │ ←─────┘
                               │   (S / D)   │
                               └──────┬──────┘
                                      │ exit() 또는 시그널
                                      ▼
                               ┌─────────────┐
                               │ 좀비 (Zombie)│
                               │     (Z)      │
                               └──────┬──────┘
                                      │ 부모가 wait()
                                      ▼
                                    소멸
```

`ps`에서 보이는 상태 코드:

| 상태 | 의미 | 서버에서의 의미 |
|------|------|----------------|
| **R** | Running 또는 Run Queue | CPU를 사용 중이거나 대기 중 |
| **S** | Interruptible Sleep | 이벤트 대기 (대부분의 프로세스가 이 상태) |
| **D** | Uninterruptible Sleep | 디스크 I/O 대기. SIGKILL로도 못 죽임 |
| **Z** | Zombie | 종료됐지만 부모가 회수 안 함 |
| **T** | Stopped | SIGSTOP 또는 Ctrl+Z |

`top`에서 R 프로세스가 코어 수보다 많으면 CPU 포화다. D 프로세스가 쌓이면 디스크 I/O 병목이다.

### 좀비 프로세스 — 죽었지만 사라지지 않는 것

자식 프로세스가 `exit()`하면, 커널은 대부분의 리소스(메모리, fd)를 해제하지만 `task_struct`의 일부(PID, 종료 상태)는 남겨둔다. 부모가 `wait()` 시스템 콜로 이 정보를 회수해야 완전히 사라진다.

부모가 wait()를 호출하지 않으면? 자식은 좀비(Z)로 남는다.

```bash
# 좀비 확인
$ ps aux | awk '$8 ~ /Z/'
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
app       5678  0.0  0.0      0     0 ?        Z    10:30   0:00 [worker] <defunct>
```

좀비 자체는 CPU나 메모리를 소비하지 않는다. 하지만 PID를 점유한다. `/proc/sys/kernel/pid_max` (기본 32768 또는 4194304)에 도달하면 새 프로세스를 만들 수 없다.

**좀비를 없애는 방법**: 좀비 자체를 kill 할 수는 없다 (이미 죽었으니까). 부모 프로세스가 wait()를 호출하게 하거나, 부모를 종료시켜야 한다. 부모가 죽으면 좀비의 새 부모는 PID 1(systemd)이 되고, systemd는 자동으로 wait()를 호출하여 좀비를 정리한다.

**Docker에서의 좀비 문제**: 컨테이너의 PID 1 프로세스가 Java 앱이면, Java는 wait()를 구현하지 않으므로 고아 프로세스가 좀비로 남을 수 있다. 해결책은 `tini`를 PID 1으로 사용하는 것이다.

```dockerfile
# tini를 PID 1으로 사용
ENTRYPOINT ["tini", "--"]
CMD ["java", "-jar", "app.jar"]
```

Docker 1.13+에서는 `docker run --init`으로 tini를 자동 주입할 수 있다.

---

## 6. 시그널 — 프로세스에 보내는 메시지

### 시그널 메커니즘

시그널은 프로세스에 비동기적으로 전달되는 소프트웨어 인터럽트다. 커널이나 다른 프로세스가 보낼 수 있다.

프로세스는 시그널에 대해 세 가지 중 하나를 할 수 있다:
1. **기본 동작 수행** — 대부분의 시그널은 기본적으로 프로세스를 종료시킴
2. **핸들러 등록** — 시그널을 잡아서 원하는 로직 실행
3. **무시** — 시그널을 받아도 아무것도 하지 않음

단, SIGKILL(9)과 SIGSTOP(19)은 잡거나 무시할 수 없다. 커널이 직접 처리한다.

### 서버 운영에서 중요한 시그널

**SIGTERM (15) — 정중한 종료 요청**

`kill <pid>` 또는 `kill -15 <pid>`가 보내는 시그널이다. 프로세스에게 "깨끗하게 종료해라"라는 의미다. 프로세스는 이 시그널을 잡아서 Graceful Shutdown을 수행할 수 있다.

```
SIGTERM 수신 → 새 요청 거부 → 진행 중 요청 완료 대기 → 리소스 정리 → exit(0)
```

Kubernetes가 Pod를 종료할 때, ECS가 Task를 중지할 때 SIGTERM을 먼저 보낸다.

**SIGKILL (9) — 강제 사살**

`kill -9 <pid>`가 보내는 시그널이다. 프로세스가 잡을 수 없으며, 커널이 즉시 프로세스를 제거한다. 처리 중인 요청은 끊기고, 열린 파일은 정리 없이 닫히고, DB 커넥션은 비정상 종료된다.

종료 코드가 137(128 + 9)이면 SIGKILL로 죽은 것이다. OOM Killer도 SIGKILL을 보낸다.

```bash
# SIGTERM → 30초 대기 → SIGKILL (Kubernetes 기본 동작)
$ kubectl delete pod myapp-pod
# terminationGracePeriodSeconds: 30 (기본값)
```

**SIGINT (2) — Ctrl+C**

터미널에서 Ctrl+C를 누르면 포그라운드 프로세스에 SIGINT가 전달된다. 기본 동작은 종료지만, 핸들러를 등록하면 잡을 수 있다.

**SIGHUP (1) — 설정 재로드**

원래 의미는 "터미널 연결이 끊어졌다"이지만, 관례적으로 "설정을 다시 읽어라"로 사용된다.

```bash
$ nginx -s reload       # 내부적으로 master에 SIGHUP 전송
$ kill -HUP $(cat /var/run/nginx.pid)  # 직접 보내기
```

Nginx는 SIGHUP을 받으면 새 설정으로 worker를 생성하고, 기존 worker는 현재 요청을 마무리한 뒤 종료된다. 프로세스 재시작 없이 설정 변경이 가능한 이유다.

**SIGCHLD (17) — 자식 종료 알림**

자식 프로세스가 종료되면 부모에게 SIGCHLD가 전달된다. 부모는 이 시그널의 핸들러에서 `wait()`를 호출하여 자식의 종료 상태를 회수한다. 이걸 안 하면 좀비가 쌓인다.

### Java에서의 시그널 처리

Spring Boot의 Graceful Shutdown은 내부적으로 JVM의 Shutdown Hook을 사용한다.

```java
// JVM이 SIGTERM을 받으면 등록된 Shutdown Hook을 실행
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // 리소스 정리 로직
    connectionPool.close();
    executorService.shutdown();
}));
```

Spring Boot 설정:

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

이 설정이 활성화되면:
1. SIGTERM 수신
2. 새 요청에 503 응답
3. 진행 중인 요청 완료 대기 (최대 30초)
4. 모든 빈의 `@PreDestroy` 실행
5. JVM 종료

---

## 7. /proc 파일시스템 — 커널이 펼쳐 보이는 정보

### /proc이란

`/proc`은 실제 디스크에 존재하지 않는 가상 파일시스템이다. 커널이 프로세스와 시스템 정보를 파일 형태로 노출하는 인터페이스다. 읽기 요청이 올 때마다 커널이 실시간 데이터를 생성한다.

```bash
$ ls /proc/
1/  2/  3/  ... 12345/  ... acpi/  cpuinfo  meminfo  loadavg  ...
#   숫자 디렉토리 = 프로세스     시스템 전역 정보 파일
```

### 프로세스별 정보 — /proc/\<pid\>/

| 파일 | 내용 | 활용 |
|------|------|------|
| `/proc/<pid>/status` | 프로세스 상태 요약 | 메모리, 스레드 수, UID 확인 |
| `/proc/<pid>/cmdline` | 실행 명령어 | 어떤 옵션으로 실행됐는지 |
| `/proc/<pid>/environ` | 환경 변수 | 실행 환경 확인 (null 구분) |
| `/proc/<pid>/fd/` | 열린 파일 디스크립터 | fd 누수 진단 |
| `/proc/<pid>/maps` | 메모리 매핑 | 어떤 라이브러리가 로드됐는지 |
| `/proc/<pid>/limits` | 리소스 제한 | ulimit 실제 적용값 확인 |
| `/proc/<pid>/io` | I/O 통계 | 디스크 읽기/쓰기량 |
| `/proc/<pid>/stat` | CPU 사용 시간 등 | 스케줄링 통계 |

```bash
# Java 프로세스의 실제 메모리 사용량
$ cat /proc/12345/status | grep -E 'VmRSS|VmSwap|Threads'
VmRSS:    524288 kB    ← 실제 물리 메모리 (RSS)
VmSwap:     4096 kB    ← swap으로 밀려난 양
Threads:      243      ← 스레드 수

# 열린 fd 수
$ ls /proc/12345/fd | wc -l
1847

# fd 중 소켓이 몇 개인지
$ ls -la /proc/12345/fd | grep socket | wc -l
1203

# 실제 적용된 리소스 제한
$ cat /proc/12345/limits
Limit                     Soft Limit   Hard Limit   Units
Max open files            65536        65536        files
Max processes             4096         62442        processes
```

### 시스템 전역 정보

```bash
# CPU 정보
$ cat /proc/cpuinfo | grep "model name" | head -1
model name : Intel(R) Xeon(R) Platinum 8275CL CPU @ 3.00GHz

# 메모리 상세
$ cat /proc/meminfo | head -5
MemTotal:       16384000 kB
MemFree:          512000 kB
MemAvailable:    3276800 kB    ← 실제 사용 가능한 메모리 (free + 회수 가능한 캐시)
Buffers:          131072 kB
Cached:          2621440 kB

# Load Average
$ cat /proc/loadavg
2.15 1.87 1.65 3/456 12789
# 1분/5분/15분 평균, 실행중/전체 프로세스, 마지막 PID

# 시스템 전체 fd 사용량
$ cat /proc/sys/fs/file-nr
3456    0    1048576
# 할당된 fd / 사용되지 않는 fd / 최대 fd
```

---

## 8. strace — 시스템 콜을 엿보다

### strace란

프로세스가 호출하는 모든 시스템 콜을 가로채서 보여주는 도구다. 애플리케이션이 커널에 무엇을 요청하는지 실시간으로 볼 수 있다. 디버깅의 최후의 무기라고 불린다.

### 기본 사용법

```bash
# 새 프로세스를 추적하며 실행
$ strace ls /tmp

# 이미 실행 중인 프로세스에 붙기
$ strace -p 12345

# 특정 시스템 콜만 필터링
$ strace -e trace=open,read,write -p 12345

# 네트워크 관련 시스템 콜만
$ strace -e trace=network -p 12345

# 파일 관련만
$ strace -e trace=file -p 12345

# 시간 정보 포함 (각 시스템 콜의 소요 시간)
$ strace -T -p 12345

# 타임스탬프 포함
$ strace -t -p 12345

# 요약 통계 (어떤 시스템 콜이 가장 많이/오래 호출됐는지)
$ strace -c -p 12345
```

### 실전 시나리오

**"설정 파일을 못 찾겠다"**

```bash
$ strace -e trace=open,openat java -jar app.jar 2>&1 | grep config
openat(AT_FDCWD, "/opt/app/config/application.yml", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/opt/app/application.yml", O_RDONLY) = 3
```

어떤 경로에서 파일을 찾는지, 실패하는 경로와 성공하는 경로를 정확히 볼 수 있다.

**"응답이 느리다"**

```bash
$ strace -T -e trace=network -p 12345
sendto(15, "SELECT * FROM users...", 45, 0, NULL, 0) = 45  <0.000023>
recvfrom(15, ..., 8192, 0, NULL, NULL) = 4096               <2.340512>
#                                                             ^^^^^^^^
#                                                             2.3초 대기!
```

`recvfrom()`에서 2.3초가 걸렸다. DB 쿼리가 느린 것이다.

**"시스템 콜 빈도 분석"**

```bash
$ strace -c -p 12345
^C
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.23    1.234567          12    102400           epoll_wait
 30.12    0.823456           8    102400           write
 15.45    0.422345           4    102400           read
  5.67    0.154987          15     10240           futex
  3.53    0.096543          94      1024           mmap
```

어떤 시스템 콜이 시간을 많이 쓰는지, 호출 빈도가 비정상적이지 않은지 파악할 수 있다.

> **주의**: strace는 프로세스 성능에 영향을 준다. 프로덕션에서는 짧은 시간만 붙이고, 가능하면 `-p`로 특정 스레드만 추적하거나, `-e`로 필요한 시스템 콜만 필터링한다.

---

## 9. lsof — 열린 파일과 연결 상태 파악

### lsof란

"List Open Files"의 약자다. 프로세스가 열고 있는 모든 파일(일반 파일, 소켓, 파이프, 디바이스)을 보여준다. 리눅스에서는 "모든 것이 파일"이므로, 사실상 프로세스의 모든 I/O 리소스를 확인하는 도구다.

### 핵심 사용법

```bash
# 특정 프로세스가 열고 있는 모든 파일
$ lsof -p 12345

# 특정 포트를 사용하는 프로세스 찾기
$ lsof -i :8080
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    12345  app   45u  IPv6 123456      0t0  TCP *:8080 (LISTEN)

# 특정 파일을 열고 있는 프로세스 찾기
$ lsof /var/log/app/spring.log
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
java    12345  app    7w   REG    8,1  15728640 131077 /var/log/app/spring.log

# 특정 프로세스의 네트워크 연결만
$ lsof -i -a -p 12345
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java    12345  app   45u  IPv6 123456      0t0  TCP *:8080 (LISTEN)
java    12345  app   67u  IPv6 123789      0t0  TCP 10.0.1.5:8080->10.0.2.3:54321 (ESTABLISHED)
java    12345  app   68u  IPv6 123790      0t0  TCP 10.0.1.5:43210->10.0.3.7:5432 (ESTABLISHED)

# ESTABLISHED 상태의 연결 수 집계
$ lsof -i -a -p 12345 | grep ESTABLISHED | wc -l
247
```

### 실전 시나리오

**"Too many open files" 원인 파악**

```bash
# fd 수 확인
$ lsof -p 12345 | wc -l
65538   ← 한계에 도달!

# fd 유형별 분류
$ lsof -p 12345 | awk '{print $5}' | sort | uniq -c | sort -rn
  42000 IPv6        ← 소켓이 42000개! 커넥션 누수
  15000 REG         ← 일반 파일
   8000 IPv4
    500 FIFO
     38 CHR
```

소켓이 42000개 열려 있다면 커넥션 풀의 반환 로직이나 HTTP 클라이언트의 close 처리를 점검해야 한다.

**"이 포트 누가 쓰고 있지?"**

```bash
$ lsof -i :8080
# "Address already in use" 에러 날 때 어떤 프로세스가 점유 중인지 확인
```

**"삭제한 로그 파일이 디스크를 차지하고 있다"**

```bash
$ lsof | grep deleted
java    12345 app  7w  REG  8,1 15728640 131077 /var/log/app/spring.log (deleted)
```

파일을 `rm`했지만, 프로세스가 아직 fd를 열고 있으면 디스크 공간이 해제되지 않는다. 프로세스를 재시작하거나, fd를 통해 파일 내용을 비워야 한다.

```bash
# 삭제된 파일의 fd를 통해 내용 비우기 (디스크 즉시 회수)
$ : > /proc/12345/fd/7
```

---

## 10. 프로세스 우선순위 — nice와 renice

### nice 값이란

모든 프로세스에는 **nice 값**이 있다. -20(최고 우선순위)부터 19(최저 우선순위)까지의 범위이며, 기본값은 0이다.

이름이 "nice"인 이유는, 높은 값을 가진 프로세스가 다른 프로세스에게 "친절하게(nicely)" CPU를 양보한다는 의미다.

```bash
# nice 값 확인
$ ps -eo pid,ni,comm | grep java
12345   0 java            ← nice 0 (기본)
12346  10 java-batch      ← nice 10 (낮은 우선순위)

# top에서 NI 컬럼으로 확인
$ top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
12345 app       20   0 5234812 512340  23456 S  45.2  6.4   1:23.45 java
12346 app       30  10 2345678 256000  12345 S   5.1  3.2   0:45.67 java-batch
```

PR(Priority)은 `20 + nice`로 계산된다. nice 0이면 PR 20, nice 10이면 PR 30이다.

### 실전 활용

```bash
# 배치 작업을 낮은 우선순위로 실행
$ nice -n 15 java -jar batch-job.jar
# CPU 경합 시 웹 서버에 더 많은 CPU 시간을 양보

# 이미 실행 중인 프로세스의 우선순위 변경
$ renice 10 -p 12346
12346 (process ID) old priority 0, new priority 10

# 사용자의 모든 프로세스 우선순위 변경
$ renice 5 -u app

# nice 값을 마이너스로 설정 (root만 가능)
$ sudo renice -5 -p 12345
# 웹 서버 프로세스에 높은 우선순위 부여
```

### 언제 쓰는가

**배치 작업**: 로그 분석, 보고서 생성, 데이터 마이그레이션 같은 배치 작업은 `nice 10~15`로 실행한다. CPU 경합이 없으면 정상 속도로 동작하고, 웹 서버와 경합하면 양보한다.

**백업 프로세스**: `mysqldump`나 `pg_dump` 같은 백업 작업은 I/O와 CPU를 많이 쓴다. 높은 nice 값으로 실행하면 서비스 영향을 줄일 수 있다.

```bash
$ nice -n 19 ionice -c 3 mysqldump mydb > backup.sql
# nice 19: CPU 최저 우선순위
# ionice -c 3: I/O도 idle 클래스 (다른 프로세스가 I/O를 안 할 때만 수행)
```

**주의**: nice 값은 CPU 스케줄링에만 영향을 준다. 디스크 I/O 우선순위를 조정하려면 `ionice`를, 메모리 제한은 `cgroup`을 사용해야 한다. 컨테이너 환경에서는 Kubernetes의 `resources.requests/limits`가 이 역할을 대신한다.

---

## 정리

| 개념 | 핵심 |
| --- | --- |
| 커널/유저 공간 | 시스템 콜이 유일한 연결 다리, 호출마다 모드 전환 비용 발생 |
| 프로세스 | 독립된 주소 공간, task_struct로 관리 |
| 스레드 | 주소 공간 공유, 동시성 문제의 근원 |
| fork/exec | COW로 빠른 복제, Docker PID 1 문제의 배경 |
| 좀비 | 부모의 wait() 누락, tini로 해결 |
| 시그널 | SIGTERM(정중한 종료) vs SIGKILL(강제), Graceful Shutdown의 전제 |
| /proc | 커널이 노출하는 실시간 정보, status/fd/maps/limits |
| strace | 시스템 콜 추적, 성능 병목과 파일 접근 문제 진단 |
| lsof | 열린 파일/소켓 확인, fd 누수와 포트 충돌 진단 |
| nice/renice | CPU 우선순위 조정, 배치 작업에 활용 |

---

## 참고 자료

- *Understanding the Linux Kernel, 3rd Edition* — Daniel P. Bovet, Marco Cesati (O'Reilly)
- *The Linux Programming Interface* — Michael Kerrisk (No Starch Press)
- Linux Kernel Documentation — [https://docs.kernel.org/](https://docs.kernel.org/)
- `man 2 fork`, `man 2 execve`, `man 7 signal`, `man 5 proc`
- Brendan Gregg, *Systems Performance, 2nd Edition* (Addison-Wesley)
- Linux `proc(5)` filesystem documentation — [https://man7.org/linux/man-pages/man5/proc.5.html](https://man7.org/linux/man-pages/man5/proc.5.html)
