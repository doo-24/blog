---
title: "[OS와 네트워크] 3편 — 메모리 구조와 관리: 서버가 메모리를 다루는 방법"
date: 2026-03-18T06:00:00+09:00
draft: false
tags: ["리눅스", "메모리", "가상메모리", "OOM", "서버"]
series: ["OS와 네트워크"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 1
summary: "가상 메모리와 페이징, 스왑의 실체, mmap, 스택과 힙의 차이, 메모리 단편화, OOM Killer, /proc/meminfo 완전 분석, 버퍼와 캐시, 메모리 릭 탐지까지"
---

서버에 16GB RAM이 있다. `free -h`를 찍으면 "available"이 3GB다. 그런데 프로세스들의 RSS를 합산하면 8GB밖에 안 된다. 나머지 5GB는 어디 갔는가?

이 질문에 정확히 답하려면 리눅스의 메모리 관리를 이해해야 한다. 가상 메모리, 페이지 캐시, 버퍼, 스왑이 어떻게 맞물려 동작하는지를 알아야 "메모리가 부족한가"라는 판단을 내릴 수 있다.

---

## 1. 가상 메모리 — 프로세스가 보는 메모리는 가짜다

### 왜 가상 메모리인가

각 프로세스는 자신만의 가상 주소 공간을 갖는다. 64비트 리눅스에서 유저 공간은 0 ~ 0x7FFFFFFFFFFF(128TB)다. 서버에 물리 메모리가 16GB밖에 없어도, 프로세스는 128TB의 주소 공간을 사용할 수 있다.

이것이 가능한 이유는, 가상 주소가 실제 물리 메모리에 매핑되기 전까지는 어떤 물리 리소스도 소비하지 않기 때문이다.

가상 메모리가 해결하는 문제:
- **격리**: 프로세스 A가 프로세스 B의 메모리를 읽거나 쓸 수 없다
- **추상화**: 프로세스는 연속된 주소 공간을 보지만, 물리 메모리는 조각나 있어도 된다
- **오버커밋**: 물리 메모리보다 많은 가상 메모리를 할당할 수 있다

### 페이지 — 메모리 관리의 최소 단위

가상 메모리와 물리 메모리는 **페이지** 단위로 매핑된다. x86_64에서 기본 페이지 크기는 **4KB**다.

```
가상 주소 공간 (프로세스가 보는 것)
┌────────┐
│ 페이지0 │ ──→ 물리 프레임 42
│ 페이지1 │ ──→ (매핑 없음, 접근 시 page fault)
│ 페이지2 │ ──→ 물리 프레임 7
│ 페이지3 │ ──→ 물리 프레임 128
│ 페이지4 │ ──→ (스왑에 있음, 디스크 프레임 15)
│  ...    │
└────────┘

페이지 테이블이 이 매핑을 관리한다.
```

프로세스가 가상 주소에 접근하면, CPU의 MMU(Memory Management Unit)가 페이지 테이블을 조회하여 물리 주소로 변환한다. 이 조회를 빠르게 하기 위해 TLB(Translation Lookaside Buffer)가 최근 변환 결과를 캐싱한다.

### 페이지 폴트 — 메모리가 실제로 할당되는 순간

`malloc()`이나 `mmap()`으로 메모리를 할당해도, 리눅스는 즉시 물리 메모리를 배정하지 않는다. 가상 주소 공간에 영역만 예약해두고, 프로세스가 해당 주소에 **처음 접근하는 순간** 물리 페이지를 할당한다. 이것이 **페이지 폴트(Page Fault)**다.

```
1. malloc(1GB) 호출
   → 가상 주소 공간에 1GB 영역 예약 (물리 메모리 소비: 0)
   → VSZ(Virtual Size) 1GB 증가

2. 배열의 첫 번째 원소에 쓰기
   → 해당 페이지에 물리 프레임 할당 (4KB)
   → RSS 4KB 증가

3. 배열 전체를 순회하며 쓰기
   → 접근하는 페이지마다 물리 프레임 할당
   → 최종적으로 RSS ~1GB
```

이것이 **Demand Paging**이다. Java에서 `-Xmx4g`로 설정해도, 실제로 힙을 다 쓰기 전까지 RSS는 4GB보다 훨씬 작다.

`top`에서 VSZ가 5GB인데 RSS가 500MB인 Java 프로세스를 보고 놀랄 필요가 없는 이유다.

페이지 폴트에는 두 종류가 있다:

**Minor fault**: 물리 페이지를 할당하기만 하면 되는 경우. 디스크 I/O 없음. 빠르다 (~1μs).

**Major fault**: 스왑이나 디스크에서 페이지를 읽어와야 하는 경우. 디스크 I/O 발생. 느리다 (~10ms).

```bash
# 프로세스의 페이지 폴트 횟수 확인
$ ps -o pid,min_flt,maj_flt,cmd -p 12345
  PID  MINFL  MAJFL CMD
12345 1234567    23 java -jar app.jar
#            ^^^^^^  ^^
#      minor faults  major faults (이 값이 크면 메모리 부족 징후)
```

### Huge Pages — 대용량 메모리의 성능 최적화

기본 4KB 페이지로 16GB를 관리하면 페이지 테이블 엔트리가 약 400만 개 필요하다. 페이지 테이블 자체가 메모리를 상당히 차지하고, TLB 미스도 빈번해진다.

**Huge Pages**는 페이지 크기를 2MB 또는 1GB로 키워서 이 문제를 완화한다.

```bash
# Huge Pages 현황 확인
$ cat /proc/meminfo | grep Huge
AnonHugePages:    524288 kB     ← THP(Transparent Huge Pages)가 사용 중
HugePages_Total:       0        ← 명시적 Huge Pages (미설정)
HugePages_Free:        0
Hugepagesize:       2048 kB     ← 2MB

# THP 상태 확인
$ cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
```

**THP(Transparent Huge Pages)**: 커널이 자동으로 4KB 페이지를 2MB로 합치는 기능이다. 편리하지만, 합치는 과정(compaction)에서 지연이 발생할 수 있다.

Redis와 같은 지연 시간에 민감한 애플리케이션에서는 THP를 끄는 것이 권장된다:

```bash
$ echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

---

## 2. 스택과 힙

### 스택 (Stack)

함수 호출마다 **스택 프레임**이 쌓인다. 지역 변수, 함수 인자, 리턴 주소가 저장된다. 함수가 리턴하면 스택 프레임이 제거된다. LIFO(Last In, First Out) 구조다.

```
함수 호출: main() → handleRequest() → queryDB() → parseResult()

높은 주소
┌─────────────────────┐
│ main() 스택 프레임    │  ← 가장 먼저 쌓임
├─────────────────────┤
│ handleRequest() 프레임│
├─────────────────────┤
│ queryDB() 프레임      │
├─────────────────────┤
│ parseResult() 프레임  │  ← 현재 실행 중, 스택 포인터(SP)가 여기
└─────────────────────┘
낮은 주소 (스택은 아래로 성장)
```

**스레드마다 독립된 스택**을 갖는다. 이것이 스레드 간에 지역 변수가 공유되지 않는 이유다.

리눅스에서 스레드 스택의 기본 크기는 8MB(`ulimit -s`로 확인). JVM에서는 `-Xss` 옵션으로 설정하며 기본값은 보통 512KB~1MB다.

```bash
# 스레드 스택 크기 확인
$ ulimit -s
8192    # 8MB

# Java 스레드 스택 크기 설정
$ java -Xss512k -jar app.jar
# 스레드 200개 × 512KB = ~100MB의 스택 메모리
```

**StackOverflowError**: 재귀 호출이 너무 깊어서 스택 공간을 초과하면 발생한다. 스택 크기를 늘리는 것은 임시 해결이고, 근본적으로 재귀 깊이를 줄이거나 반복문으로 변환해야 한다.

### 힙 (Heap)

`malloc()`, `new`로 동적 할당되는 영역이다. 프로그래머(또는 GC)가 명시적으로 해제하기 전까지 유지된다.

```
Java 프로세스 메모리 구조:

┌─────────────────────────────────────┐
│            OS 영역                    │
├─────────────────────────────────────┤
│     JVM 내부 (Metaspace, JIT 코드)    │ ← 네이티브 메모리
├─────────────────────────────────────┤
│     스레드 스택 (N개)                  │ ← -Xss × 스레드 수
├─────────────────────────────────────┤
│              JVM 힙                   │ ← -Xms ~ -Xmx
│  ┌────────────┬──────────────┐      │
│  │  Young Gen  │   Old Gen    │      │
│  │ (Eden+S0+S1)│              │      │
│  └────────────┴──────────────┘      │
├─────────────────────────────────────┤
│     Direct Buffer, MappedByteBuffer  │ ← 힙 외부 메모리
└─────────────────────────────────────┘
```

**JVM 힙 사이징의 함정**: `-Xmx4g`로 설정하면 힙만 4GB다. 여기에 Metaspace(수백 MB), 스레드 스택(200 × 1MB = 200MB), Direct Buffer, JIT 컴파일 코드, GC 오버헤드를 더하면 실제 RSS는 5~6GB에 달한다.

16GB 서버에서 `-Xmx14g`로 설정하면 OOM Kill 대상이 된다.

```bash
# JVM의 실제 메모리 사용 분석
$ jcmd 12345 VM.native_memory summary
Total: reserved=6234MB, committed=5123MB
-                 Java Heap (reserved=4096MB, committed=3500MB)
-                     Class (reserved=1234MB, committed=200MB)  ← Metaspace
-                    Thread (reserved=500MB, committed=500MB)   ← 스택
-                      Code (reserved=256MB, committed=100MB)   ← JIT 코드
-                    Direct (reserved=128MB, committed=80MB)    ← Direct Buffer
```

### 스택 vs 힙 비교

| 특성 | 스택 | 힙 |
|------|------|-----|
| 할당 속도 | 극히 빠름 (스택 포인터 이동만) | 느림 (가용 블록 탐색 필요) |
| 해제 | 자동 (함수 리턴 시) | 수동 또는 GC |
| 크기 | 작음 (수 MB) | 큼 (수 GB) |
| 단편화 | 없음 | 발생 가능 |
| 스레드 안전 | 스레드별 독립 | 공유, 동기화 필요 |

---

## 3. 메모리 단편화

### 외부 단편화 (External Fragmentation)

전체 빈 메모리는 충분한데, 연속된 빈 공간이 부족하여 할당이 실패하는 상황이다.

```
물리 메모리 상태 (각 칸 = 1 페이지):
[사용][빈][사용][빈][사용][빈][빈][사용][빈][사용]

빈 페이지: 5개 (충분)
연속 2페이지 할당 요청: [빈][빈] → 7번째 위치에서만 가능
연속 3페이지 할당 요청: 실패! (연속 3개가 없음)
```

리눅스 커널의 **Buddy System**이 이 문제를 완화한다. 메모리를 2의 거듭제곱 크기 블록으로 관리하여, 인접한 빈 블록을 합쳐 큰 연속 영역을 만든다.

```bash
# Buddy System 상태 확인
$ cat /proc/buddyinfo
Node 0, zone   Normal   4096  2048  1024   512   256   128    64    32    16     8     4
#                        2^0  2^1   2^2   2^3   2^4   2^5   2^6   2^7   2^8   2^9  2^10
#                        4KB  8KB   16KB  ...                                      4MB
# 각 숫자는 해당 크기의 빈 블록 개수
```

높은 order(큰 블록)의 수가 0에 가까우면 단편화가 심한 것이다.

### 내부 단편화 (Internal Fragmentation)

할당된 블록 내에서 실제로 사용하지 않는 공간이 낭비되는 것이다.

```
요청: 5KB
할당: 8KB (Buddy System의 최소 단위가 4KB의 배수)
낭비: 3KB (내부 단편화)
```

커널 내부에서는 **Slab Allocator**가 자주 사용되는 크기의 객체를 캐싱하여 내부 단편화를 줄인다.

```bash
# Slab 캐시 상태 확인
$ cat /proc/slabinfo | head -5
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
kmalloc-256           12345     13000    256    16        1
dentry                89012     90000    192    21        1
inode_cache           45678     46000    600     6        1
```

### JVM에서의 단편화

JVM의 GC도 단편화와 싸운다. G1 GC는 힙을 Region(기본 1~32MB)으로 나누고, GC 시 살아있는 객체를 새 Region으로 **복사(compaction)**하여 단편화를 해소한다. 이 복사 과정이 GC 일시 정지(pause)의 주요 원인이다.

---

## 4. 스왑 (Swap)

### 스왑의 동작 원리

물리 메모리가 부족해지면, 커널은 오래 접근하지 않은 메모리 페이지를 디스크(스왑 영역)로 내보내고, 필요할 때 다시 물리 메모리로 읽어온다.

```
물리 메모리 부족 발생
  │
  ├─ 1. kswapd 데몬이 깨어남
  ├─ 2. LRU(Least Recently Used) 리스트에서 오래된 페이지 선택
  ├─ 3. 더티 페이지면 디스크에 쓰기 (swap out)
  ├─ 4. 페이지 테이블 업데이트 (물리 프레임 → 스왑 슬롯)
  ├─ 5. 물리 프레임 해제
  └─ 6. 해당 페이지 접근 시 Major Page Fault → swap in
```

### 스왑은 왜 느린가

메모리 접근: ~100ns
SSD 랜덤 읽기: ~100μs (1,000배 느림)
HDD 랜덤 읽기: ~10ms (100,000배 느림)

스왑이 활발하면 서버가 극심하게 느려진다. `top`에서 `wa`(I/O wait)가 치솟고, 프로세스들이 D(Uninterruptible Sleep) 상태에 빠진다.

### swappiness

커널이 얼마나 적극적으로 스왑을 사용할지 조절하는 파라미터다.

```bash
# 현재 값 확인 (0~200, 기본값 60)
$ cat /proc/sys/vm/swappiness
60

# 서버에서 권장하는 설정
$ echo 10 > /proc/sys/vm/swappiness
# 10: 물리 메모리가 극도로 부족할 때만 스왑 사용

# 영구 설정
$ echo "vm.swappiness=10" >> /etc/sysctl.conf
$ sysctl -p
```

| swappiness | 의미 |
|-----------|------|
| 0 | 절대 스왑 안 함 (커널 3.5+에서는 OOM 직전까지 안 함) |
| 1~10 | 거의 안 함 (서버 권장) |
| 60 | 기본값 (데스크탑에 적합) |
| 100 | 적극적으로 스왑 (메모리 회수 우선) |

**DB 서버에서는 swappiness=1**이 일반적이다. MySQL, PostgreSQL은 자체적으로 메모리를 관리하므로, 커널이 DB 버퍼 풀 페이지를 스왑으로 내보내면 쿼리 성능이 급락한다.

### 스왑 모니터링

```bash
# 스왑 사용량 확인
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           16Gi        12Gi       512Mi       256Mi        3Gi        3.2Gi
Swap:          4.0Gi       1.2Gi       2.8Gi
#                          ^^^^^ swap이 사용 중이면 과거에 메모리 부족이 있었다는 뜻

# swap I/O 실시간 확인 — vmstat
$ vmstat 1
procs ---memory--- ---swap--
 r  b   swpd  free   si   so
 2  0  51200 2048     0    0     ← si/so = 0이면 현재 스왑 활동 없음
 2  3  51200 1024   500  200    ← si=500, so=200 → 스왑 활발, 성능 저하 중!

# 어떤 프로세스가 스왑을 쓰는지
$ for pid in /proc/[0-9]*; do
    swap=$(awk '/VmSwap/{print $2}' "$pid/status" 2>/dev/null)
    [ -n "$swap" ] && [ "$swap" -gt 0 ] && \
    echo "$swap kB $(cat $pid/comm 2>/dev/null) (PID $(basename $pid))"
  done | sort -rn | head
# 결과 예:
# 524288 kB java (PID 12345)   ← 이 Java 프로세스가 512MB 스왑 중
# 102400 kB redis-server (PID 23456)
```

---

## 5. mmap — 파일을 메모리처럼 다루기

### mmap이란

`mmap()`은 파일을 프로세스의 가상 메모리에 직접 매핑하는 시스템 콜이다. 매핑된 후에는 포인터로 파일 내용에 접근할 수 있다. `read()`/`write()` 시스템 콜 없이 메모리 접근만으로 파일 I/O가 가능하다.

```
일반 파일 읽기:
  read() 시스템 콜 → 커널 버퍼에 복사 → 유저 버퍼에 복사 (2번 복사)

mmap 파일 읽기:
  파일 → 페이지 캐시에 로드 → 가상 메모리에 직접 매핑 (0번 복사)
  프로세스가 해당 주소에 접근하면 page fault → 페이지 캐시에서 서비스
```

### 어디에 쓰이는가

**DB 엔진**: MongoDB(WiredTiger 이전), SQLite, LMDB가 mmap으로 데이터 파일에 접근한다. 페이지 캐시를 활용하여 자체 버퍼 풀 없이도 성능을 낸다.

**로그 처리**: 대용량 로그 파일을 mmap으로 매핑하면, 전체를 메모리에 올리지 않고도 임의 위치에 빠르게 접근할 수 있다.

**공유 메모리**: 여러 프로세스가 같은 파일을 mmap(MAP_SHARED)하면 메모리를 공유할 수 있다. Nginx의 공유 메모리 zone이 이 방식이다.

**Java에서의 mmap**: `MappedByteBuffer`가 mmap을 사용한다. Kafka가 로그 세그먼트를 mmap으로 읽는다.

```java
// Java mmap 예시
FileChannel channel = FileChannel.open(path, READ);
MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());
// buffer를 통해 파일에 직접 접근 (시스템 콜 없음)
byte value = buffer.get(offset);
```

### mmap의 주의점

- **페이지 폴트 비용**: 매핑만 하고 접근하지 않은 영역은 물리 메모리를 쓰지 않지만, 첫 접근 시 page fault가 발생한다.
- **32비트 제한**: 32비트 프로세스는 가상 주소 공간이 4GB이므로 mmap으로 매핑할 수 있는 크기가 제한된다. 64비트에서는 사실상 무제한.
- **SIGBUS**: 매핑된 파일이 truncate 등으로 줄어들면, 매핑 범위 밖에 접근할 때 SIGBUS가 발생하여 프로세스가 죽을 수 있다.

---

## 6. OOM Killer — 커널의 최후 수단

### 발동 조건

물리 메모리와 스왑이 모두 소진되어 더 이상 페이지를 할당할 수 없으면, 커널의 OOM(Out of Memory) Killer가 동작한다.

### 누구를 죽이는가

커널은 각 프로세스에 **oom_score**(0~1000)를 매긴다. 점수가 높을수록 먼저 죽는다.

점수 산정 기준:
- 메모리를 많이 쓰는 프로세스일수록 높음
- root 프로세스는 낮게 조정
- `oom_score_adj` 값으로 수동 조정 가능

```bash
# 프로세스별 oom_score 확인
$ for pid in $(ps -eo pid --no-headers); do
    score=$(cat /proc/$pid/oom_score 2>/dev/null)
    comm=$(cat /proc/$pid/comm 2>/dev/null)
    [ -n "$score" ] && [ "$score" -gt 0 ] && echo "$score $comm ($pid)"
  done | sort -rn | head -5
# 결과:
# 850 java (12345)         ← 가장 먼저 죽을 후보
# 200 redis-server (23456)
# 150 nginx (34567)
```

### oom_score_adj로 보호하기

```bash
# -1000이면 OOM Kill 대상에서 완전히 제외
$ echo -1000 > /proc/34567/oom_score_adj   # nginx 보호

# 1000이면 최우선 kill 대상
$ echo 1000 > /proc/99999/oom_score_adj    # 중요하지 않은 프로세스

# systemd 서비스에서 설정
# /etc/systemd/system/myapp.service
[Service]
OOMScoreAdjust=-500    # 중요한 서비스는 음수로 보호
```

### OOM Kill 진단

```bash
# OOM Kill 발생 확인
$ dmesg | grep -i "oom\|killed process"
[123456.789] Out of memory: Killed process 12345 (java) total-vm:5234812kB, \
anon-rss:4194304kB, file-rss:12345kB, shmem-rss:0kB, UID:1000, pgtables:8192kB, \
score:850

# journalctl로 확인
$ journalctl -k | grep -i "oom"

# 종료 코드로 판단
# exit code 137 = 128 + 9 = SIGKILL → OOM Kill 또는 수동 kill -9
```

Docker/Kubernetes 환경에서는 cgroup 레벨의 OOM과 호스트 레벨의 OOM을 구분해야 한다:

```bash
# K8s에서 OOM Kill 확인
$ kubectl get pod myapp -o jsonpath='{.status.containerStatuses[0].lastState}'
{"terminated":{"exitCode":137,"reason":"OOMKilled"}}
# 이것은 cgroup memory.max 초과 (컨테이너 메모리 제한)

# 호스트 OOM은 dmesg에서 확인
$ dmesg | tail
# 글로벌 OOM이면 호스트의 물리 메모리 자체가 부족한 것
```

### 오버커밋 정책

리눅스는 기본적으로 메모리를 **오버커밋**한다. 물리 메모리보다 많은 가상 메모리를 할당할 수 있다. 대부분의 프로세스가 할당한 메모리를 전부 사용하지 않기 때문에 보통은 문제없지만, 모든 프로세스가 동시에 많은 메모리를 쓰면 OOM이 발생한다.

```bash
# 오버커밋 정책 확인
$ cat /proc/sys/vm/overcommit_memory
0     # 기본값: 휴리스틱 오버커밋 (적당히 허용)
      # 1: 항상 허용 (Redis가 fork/BGSAVE를 위해 권장)
      # 2: 오버커밋 금지 (swap + 물리메모리 × ratio까지만 허용)

$ cat /proc/sys/vm/overcommit_ratio
50    # overcommit_memory=2일 때, 물리메모리의 50%까지 추가 허용
```

---

## 7. /proc/meminfo 완전 분석

### 핵심 필드

```bash
$ cat /proc/meminfo
MemTotal:       16384000 kB    # 전체 물리 메모리
MemFree:          512000 kB    # 완전히 비어있는 메모리
MemAvailable:    3276800 kB    # 실제 사용 가능한 메모리 ★
Buffers:          131072 kB    # 블록 디바이스 메타데이터 캐시
Cached:          2621440 kB    # 페이지 캐시 (파일 내용 캐시)
SwapCached:        51200 kB    # 스왑에서 읽어왔지만 아직 메모리에도 있는 페이지
SwapTotal:       4194304 kB    # 전체 스왑 크기
SwapFree:        3145728 kB    # 사용 가능한 스왑
Dirty:             12288 kB    # 디스크에 아직 안 쓴 더티 페이지
Slab:             256000 kB    # 커널 Slab 캐시
SReclaimable:     200000 kB    # 회수 가능한 Slab
SUnreclaim:        56000 kB    # 회수 불가능한 Slab
PageTables:        65536 kB    # 페이지 테이블이 차지하는 메모리
```

### "메모리가 부족한가?" 판단법

**MemFree를 보면 안 된다.** 리눅스는 빈 메모리를 캐시로 활용하므로, 정상적인 서버에서 MemFree는 매우 작다. 이것이 문제가 아니다.

**MemAvailable을 봐야 한다.** 이 값은 "새 프로세스에 할당할 수 있는 메모리"의 추정치다. MemFree + 회수 가능한 캐시(Cached + SReclaimable의 일부)를 합산한 값이다.

```
MemAvailable ≈ MemFree + (Cached + SReclaimable) × 회수 가능 비율
```

```bash
# 메모리 상태 한눈에 보기
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           16Gi        10Gi       512Mi       256Mi        5Gi        5.2Gi
Swap:          4.0Gi          0B       4.0Gi

# free=512Mi지만 available=5.2Gi → 메모리 여유 충분
# buff/cache 5Gi가 필요 시 회수 가능하기 때문
```

**경고 신호**:
- `MemAvailable`이 전체의 10% 이하
- `SwapFree`가 줄어들고 있음
- `vmstat`에서 `si`/`so`가 0이 아님

---

## 8. 버퍼와 캐시의 차이

### Buffers

**블록 디바이스의 메타데이터** 캐시다. 파일 시스템의 슈퍼블록, inode 정보, 디렉토리 엔트리 같은 것들이 여기에 캐싱된다.

```bash
# 버퍼 사용량
$ cat /proc/meminfo | grep Buffers
Buffers: 131072 kB    # 약 128MB
```

직접 I/O(`O_DIRECT`)를 사용하면 이 버퍼를 우회한다. DB 엔진(MySQL InnoDB)이 자체 버퍼 풀을 관리하면서 `O_DIRECT`를 사용하는 이유다 — 커널 버퍼와 DB 버퍼에 이중으로 캐싱하는 것을 방지한다.

### Cached (Page Cache)

**파일 내용**의 캐시다. 파일을 `read()`하면 커널이 디스크에서 읽은 데이터를 페이지 캐시에 보관한다. 같은 파일을 다시 읽으면 디스크 접근 없이 캐시에서 서비스한다.

```
첫 번째 read("/var/log/app.log"):
  디스크 → 페이지 캐시 → 유저 버퍼   (느림, ~10ms)

두 번째 read("/var/log/app.log"):
  페이지 캐시 → 유저 버퍼              (빠름, ~1μs)
```

이것이 서버를 재부팅하면 처음에 느린 이유다. 페이지 캐시가 비어있어서 모든 파일 접근이 디스크를 거친다. 시간이 지나면 자주 사용하는 파일이 캐시에 올라와서 빨라진다. 이를 **"캐시 워밍"**이라 한다.

```bash
# 페이지 캐시에 올라있는 파일 확인 (pcstat 도구)
$ pcstat /var/lib/mysql/ibdata1
|-------- Filename --------|  Pages  |  Cached  |  Percent  |
| /var/lib/mysql/ibdata1   |  262144 |  250000  |   95.4%   |
# 데이터 파일의 95%가 캐시에 올라가 있음 → 쿼리가 빠른 이유

# 수동으로 페이지 캐시 비우기 (테스트/벤치마크 용도)
$ echo 3 > /proc/sys/vm/drop_caches
# 1: 페이지 캐시만, 2: slab만, 3: 둘 다
# 프로덕션에서는 하지 마라!
```

### 정리

```
free 명령의 buff/cache 컬럼 = Buffers + Cached + SReclaimable

이 영역은 "사용 중"이지만 "필요하면 즉시 회수 가능"하다.
새 프로세스가 메모리를 요청하면 캐시를 밀어내고 할당한다.

따라서:
  실제 사용 가능한 메모리 ≈ free + buff/cache (의 일부)
  이것이 MemAvailable이다.
```

---

## 9. 메모리 릭 탐지

### 증상

- RSS가 시간이 지남에 따라 지속적으로 증가 (재시작 전까지 줄어들지 않음)
- OOM Kill이 주기적으로 발생
- GC가 점점 자주 일어나고, Full GC 후에도 힙 사용량이 줄지 않음

### Java 힙 메모리 릭 탐지

```bash
# 1. GC 로그 활성화
$ java -Xlog:gc*:file=gc.log:time -jar app.jar

# 2. 힙 사용량 추이 모니터링
$ jstat -gc 12345 5000
#  S0C    S1C    S0U    S1U      EC       EU        OC         OU
# 1024.0 1024.0  0.0   512.0  8192.0  4096.0   20480.0    18432.0
#                                                          ^^^^^^^^
# Old Gen 사용량(OU)이 계속 증가하면 릭 의심

# 3. 힙 덤프 수집
$ jmap -dump:format=b,file=heapdump.hprof 12345
# 또는 OOM 시 자동 수집
$ java -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/ -jar app.jar

# 4. MAT(Memory Analyzer Tool)로 분석
# → Dominator Tree에서 메모리를 가장 많이 차지하는 객체 확인
# → Leak Suspects 보고서 자동 생성
```

### 네이티브 메모리 릭 탐지

JVM 힙이 아닌 네이티브 메모리(Direct Buffer, JNI, Metaspace)에서 릭이 발생하면 RSS는 증가하지만 힙 사용량은 정상으로 보인다.

```bash
# JVM 네이티브 메모리 트래킹 활성화
$ java -XX:NativeMemoryTracking=summary -jar app.jar

# 기준점 설정
$ jcmd 12345 VM.native_memory baseline

# 일정 시간 후 비교
$ jcmd 12345 VM.native_memory summary.diff
# [diff] 양수가 큰 영역이 릭 후보
```

### pmap — 프로세스 메모리 맵 상세 분석

```bash
# 프로세스의 메모리 매핑 상세 확인
$ pmap -x 12345 | head -20
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000    4096    2048       0 r-x-- java
00007f1a2c000000  262144  200000  200000 rw---   [ anon ]   ← 힙 영역
00007f1a3c000000   65536   65536   65536 rw---   [ anon ]   ← Direct Buffer?
...
total kB         5234812  512340  312340

# RSS 기준으로 정렬하여 큰 매핑 찾기
$ pmap -x 12345 | sort -k3 -rn | head -10

# 시간에 따른 RSS 증가 추적
$ while true; do
    echo "$(date): $(pmap -x 12345 | tail -1 | awk '{print $4}') kB RSS"
    sleep 60
  done
```

### valgrind — C/C++ 메모리 릭 탐지

Java가 아닌 네이티브 프로세스(Nginx, Redis, PostgreSQL)의 메모리 릭은 valgrind로 탐지한다.

```bash
# memcheck로 메모리 릭 검출
$ valgrind --leak-check=full --show-leak-kinds=all ./myapp
==12345== LEAK SUMMARY:
==12345==    definitely lost: 1,234 bytes in 5 blocks     ← 확실한 릭
==12345==    indirectly lost: 5,678 bytes in 12 blocks
==12345==    possibly lost: 2,345 bytes in 8 blocks
==12345==    still reachable: 10,000 bytes in 20 blocks   ← 종료 시 미해제 (보통 무해)
```

> **주의**: valgrind는 프로세스를 10~50배 느리게 만든다. 프로덕션에서는 사용 불가. 개발/테스트 환경에서만 사용한다.

### AddressSanitizer — 빠른 대안

```bash
# gcc/clang의 AddressSanitizer (valgrind보다 2~3배만 느림)
$ gcc -fsanitize=address -g myapp.c -o myapp
$ ./myapp
# 메모리 오류 발생 시 상세 스택 트레이스 출력
```

---

## 정리

| 개념 | 핵심 |
| --- | --- |
| 가상 메모리 | Demand Paging, 접근 시 물리 페이지 할당, VSZ ≠ 실제 사용량 |
| 페이지 폴트 | Minor(빠름) vs Major(디스크 I/O), major가 많으면 메모리 부족 |
| Huge Pages | 2MB 페이지로 TLB 미스 감소, Redis에서는 THP 끄기 |
| 스택 vs 힙 | 스택: 스레드별 독립, 빠름. 힙: 공유, GC 관리 |
| 단편화 | 외부(Buddy System), 내부(Slab Allocator), JVM(G1 Compaction) |
| 스왑 | swappiness=10 권장, si/so > 0이면 성능 저하 |
| mmap | 파일을 메모리로 매핑, Kafka/MongoDB 활용, 복사 비용 제거 |
| OOM Killer | oom_score 기반, oom_score_adj로 보호, exit code 137 |
| /proc/meminfo | MemAvailable이 핵심, MemFree만 보면 오판 |
| 버퍼 vs 캐시 | Buffers=메타데이터, Cached=파일 내용, 둘 다 회수 가능 |
| 메모리 릭 | Java: jstat→jmap→MAT, 네이티브: pmap, valgrind, ASan |

---

## 참고 자료

- *Understanding the Linux Virtual Memory Manager* — Mel Gorman ([https://www.kernel.org/doc/gorman/](https://www.kernel.org/doc/gorman/))
- Linux Kernel Documentation: Memory Management — [https://docs.kernel.org/mm/](https://docs.kernel.org/mm/)
- `man 5 proc` (`/proc/meminfo`, `/proc/[pid]/maps`, `/proc/[pid]/status`)
- Brendan Gregg, *Systems Performance, 2nd Edition* (Addison-Wesley)
- *The Linux Programming Interface* — Michael Kerrisk, Chapter 49: Memory Mappings
- Valgrind Documentation — [https://valgrind.org/docs/manual/mc-manual.html](https://valgrind.org/docs/manual/mc-manual.html)
