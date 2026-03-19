---
title: "[성능 최적화와 안정성] 1편 — JVM과 성능 분석: 서버가 느릴 때"
date: 2026-03-17T18:06:00+09:00
draft: false
tags: ["JVM", "GC", "힙 덤프", "프로파일링", "서버"]
series: ["성능 최적화와 안정성"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 8
summary: "GC 종류(Serial, G1, ZGC, Shenandoah)와 선택 기준, GC 튜닝 파라미터와 GC 로그 분석, 힙 덤프 분석(MAT, VisualVM)과 메모리 릭 찾기, 스레드 덤프 분석과 CPU 프로파일링(async-profiler, JFR)까지"
---

서버가 갑자기 느려졌다. 응답 시간이 평소의 5배다. CPU는 정상인데 GC 시간이 튀고 있다. 힙이 가득 찼다는 경고가 온다. 이 상황에서 로그만 뒤지는 것은 낙엽 사이에서 바늘 찾기다. JVM 내부에서 무슨 일이 벌어지고 있는지 데이터로 보지 않으면 원인을 찾을 수 없다. 이 글에서는 GC 알고리즘의 선택부터 힙 덤프 분석, 스레드 덤프, CPU 프로파일링까지 JVM 성능 분석의 전 과정을 실전 위주로 다룬다.

---

## GC 알고리즘의 종류와 선택 기준

JVM의 Garbage Collector는 "어떤 객체를 수집하느냐"보다 "언제, 어떻게 멈추느냐"로 구분된다.

모든 GC는 Young Generation(Eden + Survivor)과 Old Generation을 관리하지만, Stop-The-World(STW) 시간과 처리량(throughput) 사이의 트레이드오프를 다르게 해결한다.

### Serial GC

단일 스레드로 GC를 수행한다. Minor GC와 Major GC 모두 단 하나의 스레드가 전담한다.

멀티코어 환경에서는 CPU를 낭비하고 STW 시간이 길어 서버에서는 거의 사용하지 않는다.

```bash
# Serial GC 활성화 (테스트/임베디드 환경 전용)
java -XX:+UseSerialGC -Xmx512m -jar app.jar
```

적합한 케이스: 힙이 100MB 미만이거나 단일 코어 컨테이너, 배치 CLI 도구.

### Parallel GC (Throughput Collector)

STW 동안 여러 스레드가 병렬로 GC를 수행한다. Java 8의 기본 GC다.

처리량이 높은 대신 STW가 길게 발생한다. 배치 처리처럼 응답 시간보다 전체 처리량이 중요한 애플리케이션에 적합하다.

```bash
java -XX:+UseParallelGC \
     -XX:ParallelGCThreads=8 \
     -Xms4g -Xmx4g \
     -jar batch-app.jar
```

### G1 GC (Garbage-First)

Java 9부터 기본 GC다. 힙을 고정 크기의 Region으로 나누어 가비지가 많은 Region부터 수집한다.

STW를 예측 가능한 범위로 제한하는 것이 목표다. `-XX:MaxGCPauseMillis`로 목표 일시 정지 시간을 지정할 수 있다.

```bash
java -XX:+UseG1GC \
     -Xms8g -Xmx8g \
     -XX:MaxGCPauseMillis=200 \
     -XX:G1HeapRegionSize=16m \
     -XX:G1NewSizePercent=30 \
     -XX:G1MaxNewSizePercent=60 \
     -jar server.jar
```

G1은 힙이 4GB 이상이고 응답 시간과 처리량을 균형 있게 원하는 웹 서버에 가장 적합하다.

### ZGC

Java 15에서 GA, Java 21에서 대폭 개선된 저지연 GC다. 대부분의 GC 작업을 애플리케이션 스레드와 병행(concurrent)으로 수행한다.

STW가 1ms 미만으로 유지되며, 수 TB 힙에서도 일시 정지 시간이 늘지 않는다.

```bash
java -XX:+UseZGC \
     -Xms16g -Xmx16g \
     -XX:ConcGCThreads=4 \
     -XX:ZCollectionInterval=5 \
     -jar low-latency-server.jar
```

적합한 케이스: 99th 퍼센타일 응답 시간이 중요한 금융, 게임, 실시간 API 서버.

### Shenandoah GC

Red Hat이 개발한 또 다른 저지연 GC로, OpenJDK 12부터 포함되었다. ZGC처럼 대부분의 작업을 concurrent하게 수행한다.

ZGC와의 차이점은 Shenandoah가 compaction(객체 이동)도 concurrent하게 수행한다는 점이다.

```bash
java -XX:+UseShenandoahGC \
     -Xms8g -Xmx8g \
     -XX:ShenandoahGCHeuristics=adaptive \
     -jar server.jar
```

### GC 선택 가이드

| 상황 | 추천 GC |
|---|---|
| 힙 < 1GB, 단순 배치 | Serial or Parallel |
| 힙 4-32GB, 웹 서버 | G1 (기본값으로 충분) |
| 힙 > 32GB, 저지연 필수 | ZGC |
| OpenJDK, 저지연, compaction 중요 | Shenandoah |

**안티패턴**: `-Xmx`만 크게 올리고 GC를 바꾸지 않는 것. 힙이 클수록 Full GC의 STW가 길어진다.

---

## GC 튜닝 파라미터와 GC 로그 분석

### 핵심 튜닝 파라미터

힙 크기는 초기값과 최대값을 동일하게 설정하는 것이 좋다. 힙 크기를 JVM이 동적으로 조절하면 그 과정에서 GC가 더 자주 발생한다.

```bash
# 힙 고정 (권장)
-Xms8g -Xmx8g

# Young Generation 비율 (G1에서는 G1NewSizePercent 사용)
-XX:NewRatio=2          # Old:Young = 2:1 → Young이 전체의 1/3

# Survivor 공간
-XX:SurvivorRatio=8     # Eden:Survivor = 8:1

# 객체가 Old로 넘어가는 나이
-XX:MaxTenuringThreshold=15

# Metaspace (클래스 메타데이터)
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

### GC 로그 활성화

Java 9 이후의 통합 로깅 옵션을 사용한다.

```bash
java -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m \
     -XX:+UseG1GC \
     -Xms8g -Xmx8g \
     -jar server.jar
```

Java 8에서는 구식 플래그를 사용한다.

```bash
java -XX:+PrintGCDetails \
     -XX:+PrintGCDateStamps \
     -XX:+PrintGCTimeStamps \
     -Xloggc:/var/log/app/gc.log \
     -XX:+UseGCLogFileRotation \
     -XX:NumberOfGCLogFiles=5 \
     -XX:GCLogFileSize=20m \
     -jar server.jar
```

### GC 로그 해석

```
[2026-03-17T09:23:41.123+0900][1234ms][gc] GC(42) Pause Young (Normal) (G1 Evacuation Pause)
[2026-03-17T09:23:41.123+0900][1234ms][gc] GC(42)   Eden regions: 150->0(130)
[2026-03-17T09:23:41.123+0900][1234ms][gc] GC(42)   Survivor regions: 10->12(24)
[2026-03-17T09:23:41.123+0900][1234ms][gc] GC(42)   Old regions: 200->205
[2026-03-17T09:23:41.123+0900][1234ms][gc] GC(42)   Humongous regions: 3->3
[2026-03-17T09:23:41.123+0900][1234ms][gc] GC(42) Pause Young (Normal) 2048M->1024M(8192M) 45.231ms
```

핵심은 마지막 줄이다. `2048M->1024M(8192M) 45.231ms`는 GC 전후 힙 사용량과 STW 시간이다.

Old regions 숫자가 계속 증가한다면 메모리 릭이나 Old Generation 부족을 의심한다.

### GCViewer로 시각화

GC 로그를 파일로 저장했다면 GCViewer나 GCEasy 같은 도구로 시각화할 수 있다.

```bash
# GCViewer 실행
java -jar gcviewer.jar /var/log/app/gc.log
```

GCEasy(https://gceasy.io)는 웹 기반 분석 도구로 GC 로그를 업로드하면 병목 원인을 자동으로 분석해준다.

**안티패턴**: GC 로그 없이 "GC가 문제인 것 같다"고 추측하는 것. 데이터 없이 튜닝하면 오히려 악화될 수 있다.

---

## 힙 덤프 분석과 메모리 릭 찾기

메모리 릭(Memory Leak)은 더 이상 사용하지 않는 객체가 GC에 의해 수집되지 못하고 힙에 쌓이는 현상이다.

Java의 메모리 릭은 객체에 대한 참조가 남아 있어 GC가 수집하지 못하는 경우다. 즉, 실제 메모리 릭이 아니라 **논리적 메모리 릭**이다.

### 힙 덤프 생성

OOM 발생 시 자동으로 힙 덤프를 생성하도록 설정한다.

```bash
# OOM 발생 시 자동 힙 덤프
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/dumps/heapdump.hprof \
     -jar server.jar
```

실행 중인 프로세스에서 수동으로 힙 덤프를 생성한다.

```bash
# jmap으로 힙 덤프 생성
jmap -dump:format=b,file=/tmp/heap.hprof <PID>

# live 옵션: GC 후 살아있는 객체만 덤프 (파일 크기 감소)
jmap -dump:live,format=b,file=/tmp/heap-live.hprof <PID>

# jcmd 사용 (Java 7+ 권장)
jcmd <PID> GC.heap_dump /tmp/heap.hprof
```

### MAT (Eclipse Memory Analyzer Tool)로 분석

MAT는 수 GB의 힙 덤프도 효율적으로 분석하는 도구다.

```bash
# MAT 실행 (힙 덤프 크기의 절반 이상 메모리 필요)
./MemoryAnalyzer -vmargs -Xmx4g
```

MAT의 핵심 개념 두 가지:

**Shallow Heap**: 객체 자체가 차지하는 메모리 (참조 제외).

**Retained Heap**: 해당 객체가 GC되면 함께 해제될 수 있는 모든 메모리의 합.

Retained Heap이 큰 객체가 메모리 릭의 주범인 경우가 많다.

**Leak Suspects Report** 기능을 사용하면 MAT가 자동으로 메모리 릭 의심 객체를 분석해준다.

```
Problem Suspect 1:
  One instance of "com.example.CacheManager"
  loaded by "app" @ 0x7f3a2c000
  occupies 1,234,567,890 (89.23%) bytes.

  The memory is accumulated in one instance of
  "java.util.HashMap$Node[]", loaded by "<system class loader>".
```

이 경우 `CacheManager`가 `HashMap`을 통해 대량의 데이터를 보유하고 있는 것이다.

### 메모리 릭 패턴과 코드 예시

**패턴 1: static 컬렉션에 계속 추가**

```java
// 문제 코드
public class SessionManager {
    // static Map은 애플리케이션 종료까지 살아있음
    private static final Map<String, UserSession> sessions = new HashMap<>();

    public void addSession(String id, UserSession session) {
        sessions.put(id, session);
        // 세션 만료 후 제거 로직이 없음 → 메모리 릭
    }
}

// 수정 코드: 약한 참조 또는 만료 정책 추가
public class SessionManager {
    private static final Map<String, UserSession> sessions =
        new ConcurrentHashMap<>();
    private final ScheduledExecutorService cleaner =
        Executors.newSingleThreadScheduledExecutor();

    public SessionManager() {
        // 1분마다 만료 세션 제거
        cleaner.scheduleAtFixedRate(this::evictExpired, 1, 1, TimeUnit.MINUTES);
    }

    private void evictExpired() {
        sessions.entrySet().removeIf(e -> e.getValue().isExpired());
    }
}
```

**패턴 2: 리스너/콜백 미제거**

```java
// 문제 코드
public class EventHandler {
    public void setup(EventBus bus) {
        // 리스너 등록 후 해제하지 않으면 EventBus가 EventHandler를 참조함
        bus.register(this::onEvent);
    }
    // unregister 로직 없음 → EventHandler 인스턴스가 GC되지 않음
}

// 수정 코드
public class EventHandler implements AutoCloseable {
    private final EventBus bus;
    private final EventListener listener;

    public EventHandler(EventBus bus) {
        this.bus = bus;
        this.listener = this::onEvent;
        bus.register(listener);
    }

    @Override
    public void close() {
        bus.unregister(listener);  // 반드시 해제
    }
}
```

**패턴 3: ThreadLocal 미정리**

```java
// 문제 코드
public class RequestContext {
    private static final ThreadLocal<Context> CTX = new ThreadLocal<>();

    public static void set(Context ctx) {
        CTX.set(ctx);
    }
    // remove() 없음 → 스레드 풀에서 재사용 시 이전 Context가 남아있음
}

// 수정 코드: try-finally로 반드시 정리
public class RequestContext {
    private static final ThreadLocal<Context> CTX = new ThreadLocal<>();

    public static void run(Context ctx, Runnable task) {
        CTX.set(ctx);
        try {
            task.run();
        } finally {
            CTX.remove();  // 스레드 풀 환경에서 필수
        }
    }
}
```

### VisualVM으로 실시간 모니터링

VisualVM은 실행 중인 JVM에 연결해 실시간으로 힙 사용량, GC 활동, 스레드 상태를 모니터링한다.

```bash
# VisualVM 실행 (JDK에 포함)
jvisualvm

# 원격 JVM에 JMX로 연결
java -Dcom.sun.management.jmxremote \
     -Dcom.sun.management.jmxremote.port=9090 \
     -Dcom.sun.management.jmxremote.authenticate=false \
     -Dcom.sun.management.jmxremote.ssl=false \
     -jar server.jar
```

**안티패턴**: 힙을 무조건 크게 올리는 것. 힙이 크면 Full GC 발생 시 STW가 수십 초에 달할 수 있다. 메모리 릭을 먼저 찾아야 한다.

---

## 스레드 덤프 분석

스레드 덤프는 JVM의 모든 스레드 상태를 스냅샷으로 찍은 것이다. 응답이 느려지거나 멈출 때 데드락이나 락 경합을 찾는 데 사용한다.

### 스레드 덤프 생성

```bash
# jstack으로 스레드 덤프 생성
jstack <PID> > /tmp/thread-dump.txt

# 3초 간격으로 3회 수집 (패턴 파악에 유리)
for i in 1 2 3; do
    jstack <PID> > /tmp/thread-dump-$i.txt
    sleep 3
done

# jcmd 사용
jcmd <PID> Thread.print > /tmp/thread-dump.txt

# kill 시그널 (JVM이 stderr로 출력)
kill -3 <PID>
```

### 스레드 상태 해석

```
"http-nio-8080-exec-5" #42 daemon prio=5 os_prio=0 cpu=1234.56ms elapsed=890.12s tid=0x00007f3a... nid=0x1a2b waiting for monitor entry [0x00007f3b...]
   java.lang.Thread.State: BLOCKED (on object monitor)
    at com.example.DatabasePool.getConnection(DatabasePool.java:87)
    - waiting to lock <0x000000071b234567> (a com.example.DatabasePool)
    at com.example.UserService.findUser(UserService.java:43)
    ...

"http-nio-8080-exec-1" #38 daemon prio=5 os_prio=0 cpu=8901.23ms elapsed=900.45s tid=0x00007f3c... nid=0x1a1a runnable
   java.lang.Thread.State: RUNNABLE
    at com.example.DatabasePool.getConnection(DatabasePool.java:92)
    - locked <0x000000071b234567> (a com.example.DatabasePool)
```

exec-1이 락을 보유하고 있고, exec-5가 그 락을 기다리며 BLOCKED 상태다. 이런 스레드가 수십 개라면 심각한 락 경합이다.

### 데드락 감지

jstack은 자동으로 데드락을 탐지하고 보고한다.

```
Found one Java-level deadlock:
=============================

"Thread-A":
  waiting to lock monitor 0x00007f3a... (object 0x000000071b111111, a java.lang.Object),
  which is held by "Thread-B"

"Thread-B":
  waiting to lock monitor 0x00007f3b... (object 0x000000071b222222, a java.lang.Object),
  which is held by "Thread-A"
```

데드락은 락 획득 순서를 일관되게 유지하거나 `java.util.concurrent` 패키지의 `tryLock(timeout)` 을 사용해 예방한다.

```java
// 데드락 예방: tryLock 사용
public boolean transfer(Account from, Account to, int amount) {
    try {
        if (from.lock.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                if (to.lock.tryLock(100, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        return true;
                    } finally {
                        to.lock.unlock();
                    }
                }
            } finally {
                from.lock.unlock();
            }
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    return false;  // 타임아웃: 재시도 or 실패 처리
}
```

### FastThread으로 분석 자동화

FastThread(https://fastthread.io)는 스레드 덤프를 업로드하면 BLOCKED/WAITING 스레드, 데드락, 핫 메서드를 자동으로 분석해준다.

---

## CPU 프로파일링

CPU 프로파일링은 "어떤 메서드가 CPU 시간을 많이 사용하는가"를 측정한다. 두 가지 방식이 있다.

**Sampling**: 일정 간격으로 스택을 샘플링한다. 오버헤드가 낮아 프로덕션에서 사용 가능하다.

**Instrumentation**: 모든 메서드 진입/종료에 코드를 삽입한다. 정확하지만 오버헤드가 크다.

### async-profiler

async-profiler는 Linux perf_events와 AsyncGetCallTrace API를 사용하는 저오버헤드 프로파일러다.

safepoint bias 문제가 없어 JVM 내장 프로파일러보다 정확하다.

```bash
# async-profiler 다운로드 및 실행
wget https://github.com/async-profiler/async-profiler/releases/download/v3.0/async-profiler-3.0-linux-x64.tar.gz
tar xzf async-profiler-3.0-linux-x64.tar.gz
cd async-profiler-3.0-linux-x64

# CPU 프로파일링 30초, Flame Graph 생성
./asprof -d 30 -f /tmp/flamegraph.html <PID>

# 힙 할당 프로파일링
./asprof -d 30 -e alloc -f /tmp/alloc.html <PID>

# 락 경합 프로파일링
./asprof -d 30 -e lock -f /tmp/lock.html <PID>

# 특정 메서드만 프로파일링
./asprof -d 30 -f /tmp/cpu.html \
    --include 'com/example/*' \
    <PID>
```

### Flame Graph 읽는 법

Flame Graph에서 X축은 CPU 시간의 비율이고, Y축은 스택 깊이다.

가장 넓은 박스가 가장 많은 CPU를 소비하는 메서드다. 넓고 평평한 "고원(plateau)"이 보이면 그 메서드가 병목이다.

```
# 예시 Flame Graph 해석
root
└── http-nio-exec (100%)
    ├── UserService.findUser (70%)     ← 여기가 병목
    │   ├── DatabasePool.getConn (60%)
    │   │   └── DriverManager.connect (55%)  ← 실제 병목
    │   └── UserMapper.map (10%)
    └── AuthService.validate (30%)
```

`DriverManager.connect`가 55%의 CPU를 차지한다면 커넥션 풀 부족이나 잘못된 설정 문제다.

### JFR (Java Flight Recorder)

JFR은 Java 11부터 OpenJDK에 무료로 포함된 프로덕션 수준 프로파일러다. 오버헤드가 1-2% 수준으로 매우 낮다.

```bash
# JFR 시작 (실행 중인 JVM)
jcmd <PID> JFR.start name=profile duration=60s filename=/tmp/app.jfr

# JFR 덤프 (지속적 기록 후 현재 시점 덤프)
jcmd <PID> JFR.dump name=profile filename=/tmp/app.jfr

# JFR 중지
jcmd <PID> JFR.stop name=profile

# 시작부터 JFR 활성화
java -XX:StartFlightRecording=duration=60s,filename=/tmp/app.jfr \
     -jar server.jar
```

JFR 파일은 JDK Mission Control(JMC)로 분석한다.

```bash
# JMC 실행
jmc
```

JMC는 CPU 사용률, 메모리 할당, GC 이벤트, 스레드 상태, I/O 이벤트를 통합해서 보여준다. 스레드별 CPU 시간, 가장 많이 할당하는 클래스, GC 원인까지 한 화면에서 분석할 수 있다.

### 실전: 성능 저하 원인 찾기 워크플로

```bash
# 1단계: 현재 JVM 상태 빠르게 확인
jcmd <PID> VM.flags
jcmd <PID> GC.heap_info
jcmd <PID> Thread.print | grep -c "BLOCKED"

# 2단계: GC 로그에서 최근 GC 빈도 확인
tail -100 /var/log/app/gc.log | grep "Pause"

# 3단계: CPU가 높으면 async-profiler 실행
./asprof -d 30 -f /tmp/cpu-$(date +%Y%m%d%H%M%S).html <PID>

# 4단계: 힙이 높으면 객체 히스토그램 확인 (Full GC 유발 없음)
jcmd <PID> GC.class_histogram | head -30

# 5단계: 의심스러우면 힙 덤프 (서비스 영향 최소화를 위해 live 옵션)
jmap -dump:live,format=b,file=/tmp/heap-live.hprof <PID>
```

객체 히스토그램은 힙 덤프보다 훨씬 빠르고 서비스 영향이 적다. 먼저 히스토그램으로 의심 클래스를 추린 후 힙 덤프를 분석한다.

```
# jcmd GC.class_histogram 출력 예시
 num     #instances         #bytes  class name (module)
-------------------------------------------------------
   1:       5234567      125629608  [B (byte arrays)
   2:        892341       71387280  java.lang.String
   3:        234567       56296008  com.example.UserDto    ← 이상하게 많음
   4:        123456       19752960  java.util.HashMap$Node
```

`UserDto`가 23만 개라면 메모리 릭이나 과도한 객체 생성을 의심한다.

**안티패턴 모음**:

- JVM 재시작으로 해결하고 원인 분석 생략. 재발하면 더 나쁜 상황이 된다.
- 프로덕션에서 Instrumentation 방식 프로파일러 사용. 오버헤드가 크다.
- 힙 덤프 없이 "메모리 릭일 것 같다"고 추측하는 것.
- `-Xss`(스택 크기)를 과도하게 줄여 `StackOverflowError` 유발.

---

JVM 성능 분석은 추측이 아닌 계측(instrumentation)에 기반해야 한다.

GC 로그로 GC 패턴을 확인하고, 힙 덤프로 메모리 릭을 찾고, 스레드 덤프로 락 경합을 확인하고, async-profiler나 JFR로 CPU 병목을 찾는 것이 올바른 순서다.

각 도구는 서로 다른 각도에서 JVM을 들여다보는 렌즈다. 문제 증상에 맞는 도구를 선택하고 데이터로 원인을 확정한 후 변경하면, 재발 없는 근본적인 해결이 가능하다.

---

## 참고 자료

1. [Oracle JVM Tuning Guide — Garbage Collection](https://docs.oracle.com/en/java/javase/21/gctuning/) — JVM GC 알고리즘과 튜닝 옵션의 공식 레퍼런스
2. [async-profiler GitHub](https://github.com/async-profiler/async-profiler) — 저오버헤드 CPU/메모리 프로파일러 공식 문서 및 사용 예시
3. [Eclipse MAT (Memory Analyzer Tool)](https://eclipse.dev/mat/) — 힙 덤프 분석 도구 공식 사이트 및 튜토리얼
4. [JDK Mission Control (JMC)](https://adoptium.net/jmc/) — JFR 파일 분석을 위한 공식 GUI 도구
5. [Alexey Shipilёv — JVM Anatomy Quarks](https://shipilev.net/jvm/anatomy-quarks/) — JVM 내부 동작을 깊이 이해하기 위한 블로그 시리즈
6. [GCEasy — GC Log Analyzer](https://gceasy.io/) — 웹 기반 GC 로그 자동 분석 도구
