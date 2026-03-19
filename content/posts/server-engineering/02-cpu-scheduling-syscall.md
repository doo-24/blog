---
title: "[OS와 네트워크] 2편 — CPU 스케줄링과 시스템 콜: OS가 프로세스를 다루는 법"
date: 2026-03-18T07:00:00+09:00
draft: false
tags: ["리눅스", "CPU", "스케줄링", "cgroup", "NUMA"]
series: ["OS와 네트워크"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 1
summary: "CFS 스케줄러의 동작 원리, 우선순위 역전 문제, 시스템 콜 오버헤드 측정, 컨텍스트 스위칭 비용, NUMA 아키텍처, CPU 친화성, cgroup 리소스 제한까지"
---

서버에 4코어 CPU가 있고, Spring Boot 앱이 200개의 스레드를 돌리고 있다. 4개의 코어로 200개의 스레드를 어떻게 공정하게 실행할까? 특정 스레드가 CPU를 독점하면 안 되고, 급한 작업은 먼저 처리해야 한다.

이것이 CPU 스케줄링의 문제다. 그리고 이 스케줄링의 품질이 서버의 응답 시간을 결정한다.

---

## 1. CPU 스케줄러의 역할

### 왜 스케줄러가 필요한가

CPU 코어 하나는 한 번에 하나의 스레드만 실행할 수 있다. 서버에는 수백 개의 스레드가 존재한다. Tomcat 스레드 200개, GC 스레드 4개, Kafka 컨슈머 스레드 10개, OS 자체 스레드 수십 개. 이 모든 스레드가 CPU를 원한다.

스케줄러는 "어떤 스레드에게, 얼마나 오래, CPU를 줄 것인가"를 결정한다. 이 결정이 잘못되면:

- 웹 요청 처리 스레드가 CPU를 못 받아서 응답 지연
- GC 스레드가 너무 오래 돌아서 Stop-the-World 시간 증가
- 배치 작업이 API 서버의 CPU를 빼앗아서 서비스 품질 저하

### 스케줄링의 두 가지 축

**CPU-bound 작업**: CPU 연산이 대부분인 작업. 암호화, 이미지 처리, JSON 직렬화. CPU를 오래 쓰고 싶어한다.

**I/O-bound 작업**: 대부분의 시간을 I/O 대기에 쓰는 작업. DB 쿼리 대기, HTTP 응답 대기, 디스크 읽기 대기. CPU를 잠깐 쓰고 다시 대기 상태로 들어간다.

웹 서버는 대부분 I/O-bound다. 요청을 받아서 DB 조회하고 응답을 보내는 과정에서 실제 CPU 연산 시간은 전체의 일부에 불과하다. 스케줄러는 I/O에서 깨어난 스레드를 빨리 실행해줘야 응답 지연이 줄어든다.

---

## 2. CFS — Completely Fair Scheduler

### 리눅스의 기본 스케줄러

CFS는 리눅스 2.6.23(2007년)부터 기본 스케줄러다. 이름 그대로 "완전히 공정한 스케줄링"을 목표로 한다. 모든 프로세스가 CPU 시간을 공정하게 나눠 받아야 한다는 원칙이다.

### vruntime — 공정성의 척도

CFS의 핵심 개념은 **vruntime(virtual runtime)**이다. 각 프로세스가 실제로 CPU를 사용한 시간을 가중치를 적용하여 계산한 값이다.

```
vruntime = 실제 실행 시간 × (기본 가중치 / 프로세스 가중치)
```

스케줄러는 항상 **vruntime이 가장 작은 프로세스**를 다음에 실행한다. 즉, CPU를 가장 적게 사용한 프로세스에게 우선권을 준다.

```
프로세스 A: vruntime = 100ms  ← 다음 실행 대상
프로세스 B: vruntime = 150ms
프로세스 C: vruntime = 200ms
```

A가 실행되면 A의 vruntime이 증가하고, 어느 순간 B보다 커지면 B가 실행된다. 이렇게 모든 프로세스의 vruntime이 비슷하게 유지된다.

### Red-Black Tree

CFS는 실행 가능한 프로세스들을 **레드-블랙 트리**에 저장한다. vruntime을 키로 하는 자기 균형 이진 탐색 트리다.

```
            [B: 150ms]
           /          \
    [A: 100ms]      [C: 200ms]
    /         \
[D: 80ms]  [E: 120ms]

가장 왼쪽 노드 = vruntime이 가장 작은 프로세스 = 다음 실행 대상
→ D (80ms)
```

가장 왼쪽 노드를 찾는 연산은 O(1)이다(캐싱됨). 삽입/삭제는 O(log N)이다. 수천 개의 프로세스가 있어도 스케줄링 결정이 빠르다.

### nice 값과 가중치의 관계

1편에서 다룬 nice 값이 CFS에서 어떻게 동작하는지가 여기서 명확해진다.

nice 값은 가중치(weight)로 변환된다. nice 0의 가중치는 1024이고, nice 값이 1 증가할 때마다 가중치가 약 1.25배 감소한다.

| nice | 가중치 | CPU 비율 (nice 0 대비) |
|------|--------|----------------------|
| -20 | 88761 | ~86배 |
| -10 | 9548 | ~9.3배 |
| 0 | 1024 | 1배 (기준) |
| 10 | 110 | ~0.1배 |
| 19 | 15 | ~0.015배 |

가중치가 높으면 같은 실제 실행 시간에 대해 vruntime이 느리게 증가한다. 결과적으로 vruntime이 항상 작은 쪽에 머물러서 CPU를 더 자주 받는다.

```
nice 0 (가중치 1024): 실제 10ms 실행 → vruntime += 10ms
nice 10 (가중치 110):  실제 10ms 실행 → vruntime += 93ms (10 × 1024/110)
```

nice 10인 프로세스는 같은 시간을 실행해도 vruntime이 빠르게 증가하므로, 스케줄링 우선순위에서 밀린다. 이것이 nice 값이 "다른 프로세스에게 양보한다"는 의미의 실체다.

### 타임슬라이스

CFS에서 타임슬라이스(한 번에 연속 실행되는 최대 시간)는 고정값이 아니다. 실행 가능한 프로세스 수에 따라 동적으로 조정된다.

```
스케줄링 주기 (sched_latency) = 6ms (기본값, 프로세스 ≤ 8개일 때)
프로세스가 N개면:
  타임슬라이스 = sched_latency / N = 6ms / N

프로세스가 8개보다 많으면:
  타임슬라이스 = sched_min_granularity × N
  sched_min_granularity = 0.75ms (기본값)
```

프로세스가 4개면 각각 1.5ms씩 받는다. 200개면 각각 0.75ms(최솟값)씩 받는다. 프로세스가 많아질수록 컨텍스트 스위칭 빈도가 높아지고, 그만큼 오버헤드가 커진다.

```bash
# 현재 시스템의 스케줄링 파라미터 확인
$ cat /proc/sys/kernel/sched_latency_ns
6000000          # 6ms

$ cat /proc/sys/kernel/sched_min_granularity_ns
750000           # 0.75ms

$ cat /proc/sys/kernel/sched_nr_migrate
32               # 한 번에 마이그레이션할 최대 태스크 수
```

### EEVDF — CFS의 후계자

리눅스 6.6(2023년)부터 CFS를 대체하는 **EEVDF(Earliest Eligible Virtual Deadline First)** 스케줄러가 도입되었다. vruntime 기반의 공정성은 유지하면서, 각 프로세스에 가상 데드라인(virtual deadline)을 부여하여 지연 시간(latency)을 더 정밀하게 제어한다.

EEVDF에서는 "eligible(실행 자격이 있는)" 프로세스 중에서 데드라인이 가장 빠른 것을 실행한다. I/O-bound 프로세스가 깨어났을 때 더 빠르게 CPU를 받을 수 있어, 웹 서버의 응답 지연 개선에 유리하다.

---

## 3. 실시간 스케줄링과 우선순위 역전

### 실시간 스케줄링 정책

CFS 외에 리눅스에는 실시간(Real-Time) 스케줄링 정책이 있다. RT 프로세스는 CFS 프로세스보다 항상 먼저 실행된다.

| 정책 | 설명 | 용도 |
|------|------|------|
| `SCHED_FIFO` | 선입선출, 같은 우선순위 내에서 자발적으로 양보할 때까지 계속 실행 | 오디오 처리, 산업 제어 |
| `SCHED_RR` | 라운드 로빈, 같은 우선순위 내에서 타임슬라이스로 순환 | RT 프로세스 간 공정성 필요 시 |
| `SCHED_OTHER` | CFS 일반 스케줄링 | 대부분의 서버 프로세스 |
| `SCHED_BATCH` | CPU-bound 배치 작업에 최적화 | 로그 분석, 리포트 생성 |
| `SCHED_IDLE` | 시스템이 완전히 놀 때만 실행 | 최저 우선순위 백그라운드 작업 |

```bash
# 프로세스의 스케줄링 정책 확인
$ chrt -p 12345
pid 12345's current scheduling policy: SCHED_OTHER
pid 12345's current scheduling priority: 0

# 실시간 우선순위로 실행
$ sudo chrt -f 50 java -jar critical-app.jar
# SCHED_FIFO, 우선순위 50으로 실행
```

서버에서 RT 스케줄링을 직접 쓸 일은 드물다. 하지만 커널 스레드 중 일부(ksoftirqd, 인터럽트 핸들러)가 RT로 동작하기 때문에, RT 프로세스가 CPU를 독점하면 커널 스레드가 밀려서 네트워크 패킷 처리가 지연될 수 있다.

### 우선순위 역전 (Priority Inversion)

우선순위 역전은 높은 우선순위 프로세스가 낮은 우선순위 프로세스보다 **늦게** 실행되는 역설적인 상황이다.

```
시나리오:
  프로세스 L (low priority) — 락을 획득하고 작업 중
  프로세스 M (medium priority) — 실행 가능, 락 불필요
  프로세스 H (high priority) — L이 가진 락을 기다림

실행 순서:
  1. L이 락을 잡고 실행 중
  2. H가 깨어남 → L이 가진 락이 필요 → 블록됨
  3. M이 깨어남 → L보다 우선순위 높음 → L을 밀어내고 실행
  4. M이 끝날 때까지 L은 실행 못 함 → L이 락을 못 풀음 → H는 계속 대기

결과: H > M > L인데, 실행 순서는 M → L → H
       H가 M보다 우선순위가 높은데 M이 먼저 실행됨
```

이것은 이론적인 문제가 아니다. 1997년 화성 탐사 로봇 패스파인더(Mars Pathfinder)가 이 문제로 반복 리부팅되었다.

### 해결: Priority Inheritance

**우선순위 상속(Priority Inheritance)**: 높은 우선순위 프로세스 H가 락을 기다리면, 락을 가진 L의 우선순위를 일시적으로 H와 같은 수준으로 올린다. L이 빨리 실행되어 락을 풀면, H가 바로 락을 획득할 수 있다.

```
Priority Inheritance 적용:
  1. L이 락을 잡고 실행 중 (우선순위: low)
  2. H가 L의 락을 기다림
  3. L의 우선순위가 H 수준으로 상승 → M보다 높아짐
  4. L이 먼저 실행되어 락을 풀음
  5. L의 우선순위 원복, H가 락 획득하여 실행

결과: H → L(일시 승격) → M (의도한 순서대로)
```

리눅스 커널의 `FUTEX_LOCK_PI`와 `rt_mutex`가 우선순위 상속을 지원한다. Java의 `synchronized`는 커널 수준 PI를 활용하지 않지만, `ReentrantLock`에 `fair=true` 옵션을 주면 대기 순서를 보장할 수 있다.

서버 개발에서의 교훈: 락을 가진 채로 오래 걸리는 작업을 하면 안 된다. 특히 락 안에서 I/O(DB 쿼리, HTTP 호출)를 하면, 해당 락을 기다리는 모든 스레드가 블록된다. 이것은 우선순위 역전의 소프트웨어 레벨 변형이다.

---

## 4. 시스템 콜 오버헤드

### 시스템 콜이 비싼 이유

1편에서 시스템 콜의 개념을 다뤘다. 여기서는 **비용**을 구체적으로 측정한다.

시스템 콜이 호출되면:

```
유저 모드 코드 실행 중
  │
  ├─ 1. 소프트웨어 인터럽트 (syscall 명령어)
  ├─ 2. CPU가 커널 모드로 전환
  ├─ 3. 레지스터를 커널 스택에 저장
  ├─ 4. 시스템 콜 번호로 핸들러 테이블 조회
  ├─ 5. 핸들러 실행 (실제 작업)
  ├─ 6. 결과를 레지스터에 저장
  ├─ 7. 유저 모드로 복귀 (sysret 명령어)
  │
유저 모드 코드 계속 실행
```

단계 2~3, 6~7이 순수 오버헤드다. 실제 작업(단계 5)과 무관하게 매번 발생한다.

### 비용 측정

```bash
# getpid() — 가장 간단한 시스템 콜의 비용 측정
$ cat /dev/null | strace -c -e trace=getpid dd if=/dev/zero of=/dev/null bs=1 count=1000000 2>&1
# 단순 시스템 콜 하나: ~100-300 나노초 (x86_64)

# 실제 측정 도구: sysbench
$ sudo apt install sysbench
$ sysbench cpu run
```

시스템 콜 종류별 대략적 비용:

| 시스템 콜 | 대략적 비용 | 설명 |
|-----------|-----------|------|
| `getpid()` | ~100ns | 캐싱되어 커널 진입 비용만 |
| `gettimeofday()` | ~20ns | vDSO로 커널 진입 없이 처리 |
| `read()` (캐시 히트) | ~1μs | 페이지 캐시에서 읽기 |
| `read()` (디스크) | ~10ms | 실제 디스크 I/O 대기 |
| `write()` (버퍼링) | ~1μs | 커널 버퍼에 쓰기 |
| `fork()` | ~100μs | COW 페이지 테이블 복사 |
| `mmap()` | ~10μs | 가상 메모리 매핑 설정 |

### vDSO — 시스템 콜 없이 커널 데이터 읽기

`gettimeofday()` 같은 읽기 전용 시스템 콜은 매번 커널 모드로 전환할 필요가 없다. 리눅스는 **vDSO(virtual Dynamic Shared Object)**라는 기법으로, 커널 데이터를 유저 공간에 매핑하여 시스템 콜 없이 읽을 수 있게 한다.

```bash
# vDSO가 프로세스에 매핑된 것 확인
$ cat /proc/self/maps | grep vdso
7ffd12345000-7ffd12347000 r-xp 00000000 00:00 0  [vdso]
```

Java에서 `System.currentTimeMillis()`를 호출하면, 내부적으로 `clock_gettime()`이 호출되는데, 이것이 vDSO를 통해 유저 공간에서 처리된다. 시스템 콜 진입 비용(~100ns)을 아끼고 ~20ns에 완료된다.

이것이 중요한 이유: 고성능 서버에서는 시간 조회가 초당 수십만 번 발생한다. 로깅, 메트릭, 타임아웃 체크 등에서. 매번 시스템 콜을 하면 이것만으로 CPU의 상당 부분을 소비한다.

---

## 5. 컨텍스트 스위칭 비용

### 컨텍스트 스위칭이란

CPU가 실행 중인 스레드를 바꾸는 것이다. 현재 스레드의 상태(레지스터, 프로그램 카운터, 스택 포인터)를 저장하고, 다음 스레드의 상태를 복원한다.

### 직접 비용 vs 간접 비용

**직접 비용** — 레지스터 저장/복원, 커널 스케줄러 실행. 약 1~10 마이크로초.

**간접 비용** — 이것이 더 크다.

- **TLB 플러시**: 프로세스가 바뀌면 가상→물리 주소 변환 캐시(TLB)를 비워야 한다. 이후 메모리 접근마다 TLB 미스가 발생하여 페이지 테이블을 다시 조회한다.
- **L1/L2/L3 캐시 오염**: 새 스레드의 데이터가 캐시에 없으므로 캐시 미스가 대량 발생한다. L3 캐시 미스 하나가 ~50ns, 메인 메모리 접근이 ~100ns.
- **파이프라인 플러시**: CPU의 명령어 파이프라인이 초기화되어 수 클럭의 지연 발생.

스레드 간 전환(같은 프로세스 내)은 TLB 플러시가 필요 없으므로 프로세스 간 전환보다 가볍다.

### 비용 측정

```bash
# 컨텍스트 스위칭 횟수 확인 — vmstat
$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 204800  65536 307200    0    0     4    12  256  3500 45  3 51  1  0
#                                                         ^^^^
#                                                   cs = context switches/sec

# 특정 프로세스의 컨텍스트 스위칭 횟수
$ cat /proc/12345/status | grep ctxt
voluntary_ctxt_switches:        15234   ← 자발적 (I/O 대기 등)
nonvoluntary_ctxt_switches:      3456   ← 비자발적 (타임슬라이스 소진)

# perf로 정밀 측정
$ sudo perf stat -e context-switches,cpu-migrations -p 12345 sleep 10
 Performance counter stats for process id '12345':
         12,345      context-switches          # 1234.5 /sec
            234      cpu-migrations            # 23.4 /sec
```

**voluntary**(자발적): 프로세스가 I/O를 요청하거나 `sleep()`을 호출하여 스스로 CPU를 양보한 횟수. I/O-bound 프로세스에서 높다.

**nonvoluntary**(비자발적): 타임슬라이스가 끝나서 스케줄러가 강제로 빼앗은 횟수. CPU-bound 프로세스에서 높다.

### 서버에서의 실전 의미

웹 서버에서 비자발적 컨텍스트 스위칭이 급증하면:
- 스레드 수가 코어 수 대비 과도함
- CPU-bound 작업(JSON 직렬화, 암호화)이 타임슬라이스를 다 소진

자발적 컨텍스트 스위칭이 매우 높으면:
- I/O 대기가 많음 (DB 쿼리 지연, 외부 API 지연)
- 락 경합이 심함 (`synchronized` 진입 대기)

```bash
# 실전: Spring Boot 앱의 스레드 수와 컨텍스트 스위칭 상관관계 모니터링
$ pidstat -w -t -p 12345 1
# 스레드별 자발적/비자발적 컨텍스트 스위칭 실시간 확인
```

---

## 6. NUMA 아키텍처

### NUMA란

**NUMA(Non-Uniform Memory Access)**는 멀티 CPU 서버의 메모리 접근 구조다. CPU마다 "가까운 메모리(local memory)"와 "먼 메모리(remote memory)"가 존재하며, 접근 속도가 다르다.

```
┌─────────────────────┐     QPI/UPI      ┌─────────────────────┐
│     NUMA Node 0     │ ◄──────────────► │     NUMA Node 1     │
│                     │   (인터커넥트)     │                     │
│  CPU 0   CPU 1      │                  │  CPU 2   CPU 3      │
│  Core0-7 Core8-15   │                  │  Core16-23 Core24-31│
│                     │                  │                     │
│  Local Memory 64GB  │                  │  Local Memory 64GB  │
│  (접근: ~80ns)       │                  │  (접근: ~80ns)       │
│                     │                  │                     │
└─────────────────────┘                  └─────────────────────┘
         │                                        │
         └── Remote Memory 접근: ~150ns ──────────┘
              (약 2배 느림)
```

4소켓, 8소켓 서버에서는 NUMA 노드가 4~8개이고, 가장 먼 노드의 메모리 접근은 로컬의 3~4배 느릴 수 있다.

### 왜 서버 개발자가 알아야 하는가

대부분의 클라우드 인스턴스(AWS c5.xlarge, m5.2xlarge 등)는 단일 NUMA 노드다. 하지만 대형 인스턴스(c5.9xlarge 이상)나 베어메탈 서버는 멀티 NUMA 노드다.

JVM이 NUMA를 인식하지 못하면, 모든 메모리를 하나의 노드에 할당하고 다른 CPU들은 매번 remote 접근을 하게 된다. 이것만으로 성능이 20~30% 떨어질 수 있다.

```bash
# NUMA 토폴로지 확인
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 65536 MB
node 0 free: 32000 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 65536 MB
node 1 free: 48000 MB
node distances:
node   0   1
  0:  10  21     ← Node 0 → Node 1 접근이 2.1배 느림
  1:  21  10

# NUMA 노드별 메모리 사용량
$ numastat
                           node0           node1
numa_hit              1234567890      987654321
numa_miss                  12345         234567    ← miss가 많으면 remote 접근 빈번
numa_foreign              234567          12345
local_node            1234567890      987654321
other_node                 12345         234567
```

### NUMA-aware 설정

```bash
# 특정 NUMA 노드에서 프로세스 실행
$ numactl --cpunodebind=0 --membind=0 java -jar app.jar
# CPU와 메모리를 모두 Node 0에 바인딩

# JVM NUMA 지원 활성화
$ java -XX:+UseNUMA -XX:+UseParallelGC -jar app.jar
# GC가 NUMA 토폴로지를 인식하여 객체를 로컬 메모리에 할당
```

대부분의 경우 OS와 JVM이 NUMA를 자동으로 처리한다. 하지만 대형 인스턴스에서 성능이 기대에 못 미치면, `numastat`의 `numa_miss` 수치를 확인해보자.

---

## 7. CPU 친화성 (CPU Affinity)

### CPU 친화성이란

특정 프로세스/스레드를 특정 CPU 코어에 고정(pin)하는 것이다. 스케줄러가 다른 코어로 마이그레이션하는 것을 방지한다.

### 왜 필요한가

스케줄러가 스레드를 코어 0에서 실행하다가 코어 3으로 옮기면, 코어 0의 L1/L2 캐시에 쌓인 데이터를 코어 3에서는 쓸 수 없다. 캐시를 처음부터 다시 채워야 한다.

CPU-bound 작업이나 지연 시간에 민감한 작업에서는 이 캐시 미스 비용이 크다.

```bash
# 프로세스의 현재 CPU 친화성 확인
$ taskset -p 12345
pid 12345's current affinity mask: ff    ← 0xFF = CPU 0-7 모두 사용 가능

# CPU 0-3에만 고정
$ taskset -cp 0-3 12345
pid 12345's current affinity list: 0-3

# 시작 시 특정 CPU에 고정하여 실행
$ taskset -c 0-3 java -jar app.jar
```

### 실전 활용 패턴

**네트워크 집약 서버**: 네트워크 인터럽트를 처리하는 CPU와 애플리케이션을 같은 NUMA 노드에 배치하면, 패킷 데이터가 로컬 캐시에 남아있어 처리가 빠르다.

```bash
# 네트워크 인터럽트가 어느 CPU에서 처리되는지 확인
$ cat /proc/interrupts | grep eth0
# CPU 0-3에서 처리 중이면, 앱도 0-3에 배치

# RPS(Receive Packet Steering) 설정으로 네트워크 처리 CPU 분산
$ echo "f" > /sys/class/net/eth0/queues/rx-0/rps_cpus
```

**Redis**: 싱글 스레드이므로 CPU 친화성을 설정하면 캐시 미스가 줄어 성능이 향상된다.

```bash
$ taskset -c 0 redis-server /etc/redis.conf
```

**주의**: 컨테이너 환경(Docker, K8s)에서는 cgroup이 CPU 할당을 관리하므로, `taskset`보다 cgroup/K8s의 CPU 리소스 설정을 사용해야 한다.

---

## 8. cgroup — 리소스 격리와 제한

### cgroup이란

**cgroup(Control Group)**은 프로세스 그룹에 대해 CPU, 메모리, 디스크 I/O, 네트워크 대역폭 등의 리소스 사용량을 제한하고 격리하는 리눅스 커널 기능이다.

Docker 컨테이너의 리소스 제한이 cgroup으로 구현된다. Kubernetes의 `resources.requests/limits`도 최종적으로 cgroup 설정으로 변환된다.

### cgroup v1 vs v2

| 특성 | cgroup v1 | cgroup v2 |
|------|-----------|-----------|
| 구조 | 리소스 종류별 독립 계층 | 단일 통합 계층 |
| CPU 제어 | `cpu`, `cpuacct` 별도 | `cpu` 통합 |
| 메모리 제어 | `memory` | `memory` (OOM 처리 개선) |
| 도입 시기 | 커널 2.6.24 | 커널 4.5 (안정화 5.x) |
| Docker 지원 | 기본 | Docker 20.10+ |

```bash
# 현재 시스템의 cgroup 버전 확인
$ stat -fc %T /sys/fs/cgroup
cgroup2fs     ← v2
tmpfs         ← v1
```

### CPU 리소스 제한

cgroup은 CPU를 두 가지 방식으로 제한한다.

**CPU shares (상대적 가중치)**

```bash
# Docker에서 CPU shares 설정
$ docker run --cpu-shares=512 myapp    # 기본값 1024의 절반
```

CPU shares는 CPU 경합이 있을 때만 의미가 있다. shares 1024인 컨테이너와 512인 컨테이너가 경합하면, 1024가 2/3, 512가 1/3의 CPU를 받는다. 경합이 없으면 둘 다 100%를 쓸 수 있다.

**CPU quota (절대적 상한)**

```bash
# 2 CPU 코어까지만 사용 가능
$ docker run --cpus=2 myapp
# 내부적으로 cpu.max = "200000 100000" (200ms / 100ms 주기)
```

`--cpus=2`는 "100ms 주기 중 200ms까지 사용 가능"을 의미한다. 이것은 2개의 코어를 100% 쓰는 것과 같다. 이 한도를 넘으면 **CPU 쓰로틀링(throttling)**이 발생하여 프로세스가 강제로 대기한다.

### CPU 쓰로틀링 — 보이지 않는 성능 킬러

쓰로틀링은 서버 성능을 죽이는데, 모니터링에서 잡기 어렵다. CPU 사용률이 50%로 보이는데 응답이 느리다면, 쓰로틀링을 의심해야 한다.

```bash
# cgroup에서 쓰로틀링 상태 확인 (cgroup v2)
$ cat /sys/fs/cgroup/mycontainer/cpu.stat
usage_usec 123456789
user_usec 100000000
system_usec 23456789
nr_periods 100000
nr_throttled 15000        ← 쓰로틀링 발생 횟수
throttled_usec 5000000    ← 쓰로틀링으로 대기한 총 시간 (5초)

# Docker stats로 간단히 확인
$ docker stats myapp
CONTAINER ID  NAME   CPU %   MEM USAGE / LIMIT
a1b2c3d4e5f6  myapp  195%    1.2GiB / 2GiB
#                     ^^^
# --cpus=2이면 200%가 최대. 195%면 거의 한계에 도달
```

**GC와 쓰로틀링의 악순환**: JVM의 GC는 순간적으로 모든 CPU 코어를 사용한다. `--cpus=2`로 제한된 컨테이너에서 G1 GC가 8개 스레드로 동작하면, CPU quota를 순식간에 소진하고 GC 이후 애플리케이션 스레드가 쓰로틀링된다.

```bash
# JVM GC 스레드 수를 CPU 제한에 맞추기
$ java -XX:ParallelGCThreads=2 -XX:ConcGCThreads=1 -jar app.jar
# --cpus=2이면 GC 스레드도 2개로 제한
```

### 메모리 리소스 제한

```bash
# 메모리 2GB로 제한
$ docker run --memory=2g --memory-swap=2g myapp
# --memory-swap을 같은 값으로 하면 swap 비활성화

# cgroup에서 메모리 상태 확인
$ cat /sys/fs/cgroup/mycontainer/memory.current
2147483648    ← 현재 사용량 (바이트)

$ cat /sys/fs/cgroup/mycontainer/memory.max
2147483648    ← 최대 제한

$ cat /sys/fs/cgroup/mycontainer/memory.events
low 0
high 0
max 1234       ← 제한에 도달한 횟수
oom 0          ← OOM으로 프로세스가 kill된 횟수
oom_kill 0
```

메모리 제한에 도달하면 cgroup OOM이 발생한다. 이것은 커널의 글로벌 OOM Killer와 별개다. 컨테이너의 메모리 제한에 걸려 죽은 것인지, 호스트의 메모리 부족으로 죽은 것인지 구분해야 한다.

```bash
# Kubernetes에서 OOM Kill 확인
$ kubectl describe pod myapp-pod | grep -A5 "Last State"
    Last State:     Terminated
      Reason:       OOMKilled          ← cgroup 메모리 제한 초과
      Exit Code:    137
```

### Kubernetes 리소스 설정과 cgroup의 관계

```yaml
resources:
  requests:
    cpu: "500m"      # 0.5 CPU — 스케줄링 기준 (최소 보장)
    memory: "512Mi"  # 512MB — 스케줄링 기준 (최소 보장)
  limits:
    cpu: "2"         # 2 CPU — cpu.max quota로 변환
    memory: "2Gi"    # 2GB — memory.max로 변환
```

`requests`는 K8s 스케줄러가 노드에 Pod를 배치할 때 참고하는 값이다. 이 CPU/메모리가 보장된다.

`limits`는 cgroup으로 강제되는 상한이다. CPU limit를 넘으면 쓰로틀링, 메모리 limit를 넘으면 OOM Kill.

**흔한 실수**: `requests`와 `limits`의 차이가 너무 크면, 노드에 Pod가 과잉 배치되어 리소스 경합이 심해진다. `limits` 없이 `requests`만 설정하면 노드의 나머지 리소스를 전부 사용할 수 있어 다른 Pod에 영향을 준다.

---

## 정리

| 개념 | 핵심 |
| --- | --- |
| CFS | vruntime 기반 공정 스케줄링, nice → 가중치 → vruntime 증가 속도 |
| EEVDF | CFS 후계자 (커널 6.6+), 가상 데드라인으로 지연 시간 개선 |
| 우선순위 역전 | 락을 가진 저우선순위 프로세스가 중간 우선순위에 밀림, Priority Inheritance로 해결 |
| 시스템 콜 비용 | 순수 진입/복귀 ~100ns, vDSO로 시간 조회는 ~20ns |
| 컨텍스트 스위칭 | 직접 비용보다 캐시/TLB 미스 간접 비용이 큼, voluntary/nonvoluntary 구분 |
| NUMA | CPU마다 가까운/먼 메모리, remote 접근 2배+ 느림, `numastat` 확인 |
| CPU Affinity | `taskset`으로 코어 고정, 캐시 미스 감소 |
| cgroup | Docker/K8s의 리소스 제한 핵심, CPU 쓰로틀링이 보이지 않는 성능 킬러 |

---

## 참고 자료

- *Understanding the Linux Kernel, 3rd Edition* — Daniel P. Bovet, Marco Cesati (O'Reilly)
- Linux Kernel Documentation: CFS Scheduler — [https://docs.kernel.org/scheduler/sched-design-CFS.html](https://docs.kernel.org/scheduler/sched-design-CFS.html)
- EEVDF Scheduler — Peter Zijlstra, LWN.net [https://lwn.net/Articles/925371/](https://lwn.net/Articles/925371/)
- `man 7 sched`, `man 2 sched_setaffinity`, `man 7 cgroups`
- Brendan Gregg, *Systems Performance, 2nd Edition* (Addison-Wesley)
- Linux vDSO documentation — [https://man7.org/linux/man-pages/man7/vdso.7.html](https://man7.org/linux/man-pages/man7/vdso.7.html)
