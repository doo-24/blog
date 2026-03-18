---
title: "파일 시스템과 I/O — 디스크와 서버의 대화"
date: 2026-03-18T03:00:00+09:00
draft: false
tags: ["리눅스", "파일시스템", "I/O", "서버", "디스크"]
series: ["OS와 네트워크"]
summary: "파일 디스크립터부터 inode, 저널링 파일 시스템, I/O 스케줄러까지 — 서버가 디스크와 대화하는 모든 방법을 깊이 있게 다룬다."
---

## 들어가며

서버 프로그래밍에서 가장 느린 연산은 거의 항상 디스크 I/O다. CPU가 나노초 단위로 연산을 처리하는 동안, HDD의 랜덤 읽기는 밀리초 단위 — 약 **100만 배** 느리다. SSD가 이 격차를 줄였지만, 여전히 메모리 접근 대비 수천 배 느리다.

서버 엔지니어가 파일 시스템과 I/O를 깊이 이해해야 하는 이유가 여기에 있다. 어떤 I/O 전략을 선택하느냐에 따라 동일한 하드웨어에서 처리량이 10배 이상 달라질 수 있다.

---

## 1. 파일 디스크립터 — 모든 I/O의 출발점

### 파일 디스크립터란

리눅스에서 **모든 I/O는 파일 디스크립터(File Descriptor, fd)**를 통해 이루어진다. 파일만이 아니다 — 소켓, 파이프, 디바이스, 심지어 `/proc` 아래의 가상 파일까지 전부 fd로 추상화된다.

```
프로세스 A
├── fd 0  →  /dev/pts/0       (stdin)
├── fd 1  →  /dev/pts/0       (stdout)
├── fd 2  →  /dev/pts/0       (stderr)
├── fd 3  →  /var/log/app.log (일반 파일)
├── fd 4  →  socket:[12345]   (TCP 연결)
└── fd 5  →  pipe:[67890]     (파이프)
```

프로세스가 `open()` 시스템 콜을 호출하면 커널은 **가장 작은 사용 가능한 정수**를 fd로 할당한다. 이 fd는 커널 내부의 **파일 테이블 엔트리(file table entry)**를 가리키고, 이 엔트리가 다시 **inode**를 참조한다.

```
프로세스 fd 테이블         커널 파일 테이블           inode 테이블
┌────────────┐      ┌──────────────────┐      ┌─────────────┐
│ fd 3 ──────┼─────→│ offset: 1024     │─────→│ inode #4821 │
│ fd 4 ──────┼──┐   │ flags: O_RDONLY  │      │ size: 8192  │
└────────────┘  │   └──────────────────┘      │ blocks: 16  │
                │   ┌──────────────────┐      └─────────────┘
                └──→│ offset: 0        │─────→ (다른 inode)
                    │ flags: O_WRONLY  │
                    └──────────────────┘
```

핵심 포인트: **같은 파일을 두 번 `open()`하면 별도의 파일 테이블 엔트리가 생긴다.** 각각 독립적인 오프셋을 가지므로 동시에 다른 위치를 읽을 수 있다. 반면 `fork()`로 자식 프로세스를 만들면 **같은 파일 테이블 엔트리를 공유**한다 — 부모가 읽으면 자식의 오프셋도 바뀐다.

### fd 한계와 서버 운영

프로세스당 열 수 있는 fd 수에는 제한이 있다:

```bash
# 현재 프로세스의 소프트 리밋 확인
$ ulimit -n
1024

# 시스템 전체 최대 fd 수
$ cat /proc/sys/fs/file-max
9223372036854775807

# 현재 시스템이 사용 중인 fd 수
$ cat /proc/sys/fs/file-nr
3456    0    9223372036854775807
# 할당된 fd / 사용 가능한 fd / 최대값
```

Nginx나 데이터베이스처럼 수만 개의 동시 연결을 처리하는 서버에서 기본값 1024는 턱없이 부족하다. `Too many open files` 에러를 만나본 적이 있다면 바로 이 제한 때문이다.

```bash
# /etc/security/limits.conf
nginx    soft    nofile    65536
nginx    hard    nofile    65536

# systemd 서비스인 경우 유닛 파일에서 설정
# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65536
```

### 프로세스의 fd 상태 확인

운영 중인 서버의 fd 상태를 확인하는 건 디버깅의 기본이다:

```bash
# 특정 프로세스가 열고 있는 모든 fd
$ ls -la /proc/<pid>/fd/
lr-x------ 1 www www 64 Mar 18 10:00 0 -> /dev/null
lrwx------ 1 www www 64 Mar 18 10:00 1 -> /var/log/nginx/access.log
lrwx------ 1 www www 64 Mar 18 10:00 6 -> socket:[123456]

# fd 유형별 카운트
$ ls -la /proc/<pid>/fd/ | awk '{print $NF}' | sed 's/.*\///' | sort | uniq -c | sort -rn

# lsof로 더 상세한 정보
$ lsof -p <pid> | head -20
```

fd 누수(leak)는 서버 장애의 흔한 원인 중 하나다. 파일이나 소켓을 열고 닫지 않으면 fd가 계속 쌓이다가 한계에 도달한다. 특히 예외 처리 경로에서 `close()`를 빠뜨리는 실수가 잦다.

---

## 2. inode — 파일의 진짜 정체

### inode 구조

우리가 "파일"이라고 부르는 것은 사실 두 가지로 구성된다:
- **디렉토리 엔트리**: 파일 이름 → inode 번호 매핑
- **inode**: 파일의 실제 메타데이터와 데이터 블록 위치

```
디렉토리 엔트리              inode #4821
┌──────────────────┐      ┌─────────────────────────┐
│ "app.log" → 4821 │─────→│ 소유자: www-data         │
│ "config"  → 4822 │      │ 권한: 644                │
│ "data"    → 4823 │      │ 크기: 8192 bytes         │
└──────────────────┘      │ 링크 카운트: 1           │
                          │ 타임스탬프:              │
                          │   atime (접근)           │
                          │   mtime (수정)           │
                          │   ctime (변경)           │
                          │ 데이터 블록 포인터:       │
                          │   [0] → block 1001       │
                          │   [1] → block 1002       │
                          │   ...                    │
                          │   indirect → block 2000  │
                          └─────────────────────────┘
```

inode에는 파일 이름이 **없다**. 이름은 디렉토리 엔트리에만 존재한다. 이 분리가 하드 링크를 가능하게 만든다.

```bash
# inode 정보 확인
$ stat /var/log/syslog
  File: /var/log/syslog
  Size: 234567      Blocks: 464        IO Block: 4096   regular file
Device: 801h/2049d  Inode: 131074      Links: 1
Access: (0640/-rw-r-----)  Uid: (  104/  syslog)   Gid: (    4/     adm)
Access: 2026-03-18 10:00:00.000000000 +0900
Modify: 2026-03-18 09:55:30.000000000 +0900
Change: 2026-03-18 09:55:30.000000000 +0900

# 파일 시스템의 inode 사용량
$ df -i
Filesystem      Inodes   IUsed   IFree  IUse% Mounted on
/dev/sda1      6553600  234567 6319033     4% /
```

inode가 고갈되면 디스크 공간이 남아 있어도 새 파일을 만들 수 없다. 작은 파일을 수백만 개 생성하는 애플리케이션(메일 서버, 캐시 서버 등)에서 실제로 발생하는 문제다.

### 데이터 블록 주소 지정

전통적인 ext 파일 시스템에서 inode는 데이터 블록을 **직접 포인터 + 간접 포인터** 구조로 참조한다:

```
inode
├── 직접 포인터 12개  → 각각 4KB 블록  = 48KB
├── 1차 간접 포인터   → 1024개 블록     = 4MB
├── 2차 간접 포인터   → 1024² 블록      = 4GB
└── 3차 간접 포인터   → 1024³ 블록      = 4TB
```

이 구조에서 큰 파일의 끝부분을 읽으려면 간접 블록을 여러 번 거쳐야 한다. ext4는 이를 개선하기 위해 **extent** 방식을 도입했다 — 연속된 블록을 시작 블록 + 길이로 표현해서, 대용량 파일도 적은 수의 extent로 커버한다.

```
ext3 (블록 맵)                    ext4 (extent)
블록 100 → 데이터                 extent: 블록 100, 길이 500
블록 101 → 데이터                   → 블록 100~599의 데이터를
블록 102 → 데이터                     하나의 엔트리로 표현
...
블록 599 → 데이터
(500개의 포인터 필요)              (1개의 extent로 충분)
```

---

## 3. 하드 링크와 심볼릭 링크

### 하드 링크 (Hard Link)

하드 링크는 **같은 inode를 가리키는 또 다른 디렉토리 엔트리**다. 원본과 하드 링크는 완전히 동등하다 — 어느 쪽이 "원본"인지 구분할 방법이 없다.

```bash
$ echo "hello" > original.txt
$ ln original.txt hardlink.txt

$ ls -li original.txt hardlink.txt
131074 -rw-r--r-- 2 user user 6 Mar 18 10:00 hardlink.txt
131074 -rw-r--r-- 2 user user 6 Mar 18 10:00 original.txt
#  ↑ 같은 inode     ↑ 링크 카운트 2
```

하드 링크의 특성:
- **같은 파일 시스템** 내에서만 생성 가능 (inode 번호는 파일 시스템 로컬)
- **디렉토리에는 생성 불가** (순환 참조 방지)
- 원본을 삭제해도 하드 링크로 접근 가능 (링크 카운트가 0이 되어야 실제 삭제)

서버에서 하드 링크가 유용한 사례: 로그 로테이션에서 현재 로그 파일을 하드 링크로 백업한 뒤, 원본을 truncate하면 열려 있는 fd에 영향을 주지 않으면서 백업을 유지할 수 있다.

### 심볼릭 링크 (Symbolic Link)

심볼릭 링크는 **경로 문자열을 담고 있는 별도의 파일**이다. 하드 링크와 달리 별도의 inode를 가진다.

```bash
$ ln -s /var/log/app/current.log symlink.log

$ ls -li symlink.log
131080 lrwxrwxrwx 1 user user 26 Mar 18 10:00 symlink.log -> /var/log/app/current.log
# ↑ 다른 inode  ↑ 'l' = 심볼릭 링크
```

심볼릭 링크의 특성:
- **파일 시스템을 넘어서** 생성 가능
- **디렉토리에도** 생성 가능
- **대상이 삭제되면 깨진다** (dangling symlink)
- 경로 해석 오버헤드가 있다

```
하드 링크                         심볼릭 링크
┌──────────┐   ┌──────────┐      ┌──────────┐   ┌──────────┐
│ fileA.txt│──→│ inode    │      │ linkB    │──→│ inode B  │
│          │   │ #4821    │      │          │   │ 내용:    │
└──────────┘   │          │      └──────────┘   │"/a/fileA"│
┌──────────┐   │ 데이터   │                      └────┬─────┘
│ fileB.txt│──→│ 블록     │                           │ 경로 해석
│          │   │          │      ┌──────────┐         │
└──────────┘   └──────────┘      │ fileA.txt│──→inode A
                                 └──────────┘
```

실무에서 심볼릭 링크의 대표적 활용:
- **배포 전환**: `/app/current` → `/app/releases/v2.3.1` 형태로, 심볼릭 링크만 교체해서 무중단 배포
- **설정 관리**: `/etc/nginx/sites-enabled/` 안의 심볼릭 링크로 가상 호스트 활성화/비활성화
- **로그 관리**: 현재 로그 디렉토리를 심볼릭 링크로 관리

---

## 4. Buffered I/O vs Direct I/O

### Page Cache — 커널의 디스크 캐시

리눅스 커널은 디스크에서 읽은 데이터를 **Page Cache**에 보관한다. 같은 데이터를 다시 읽을 때 디스크까지 가지 않고 메모리에서 바로 반환한다.

```
일반적인 read() 흐름
┌─────────────┐     ┌──────────────┐     ┌──────────┐
│ 애플리케이션  │────→│ Page Cache   │────→│ 디스크    │
│ (유저 공간)   │←────│ (커널 공간)   │←────│          │
└─────────────┘     └──────────────┘     └──────────┘
      ↕                    ↕
  유저 버퍼            커널 버퍼

1. read() 시스템 콜 호출
2. 커널이 Page Cache 확인
3. Cache HIT → 커널 버퍼에서 유저 버퍼로 복사
4. Cache MISS → 디스크에서 커널 버퍼로 읽고, 유저 버퍼로 복사
```

### Buffered I/O (기본 동작)

대부분의 파일 I/O는 Buffered I/O다. `write()` 호출 시 데이터가 Page Cache에 기록되고, 커널이 나중에 디스크에 쓴다 (write-back).

```
write() → Page Cache (dirty page) → [나중에] → 디스크

장점:
- write()가 즉시 반환 (디스크 대기 없음)
- 연속된 소규모 쓰기를 모아서 한 번에 플러시
- 읽기 성능 향상 (캐시된 데이터 재사용)

위험:
- 시스템 크래시 시 dirty page 유실 가능
- Page Cache가 메모리를 차지
```

커널은 다음 조건에서 dirty page를 디스크에 플러시한다:
- `/proc/sys/vm/dirty_expire_centisecs` (기본 3000 = 30초) 동안 dirty 상태가 유지된 경우
- dirty page 비율이 `/proc/sys/vm/dirty_ratio` (기본 20%)를 초과하면 쓰기 프로세스가 블록됨
- dirty page 비율이 `/proc/sys/vm/dirty_background_ratio` (기본 10%)를 초과하면 백그라운드 플러시 시작

### Direct I/O — Page Cache 우회

Direct I/O는 Page Cache를 건너뛰고 애플리케이션 버퍼에서 디스크로 직접 읽고 쓴다.

```c
// Direct I/O로 파일 열기
int fd = open("/data/db.dat", O_RDWR | O_DIRECT);

// 주의: 버퍼, 오프셋, 크기가 모두 블록 크기(보통 512B 또는 4KB)에 정렬되어야 한다
void *buf;
posix_memalign(&buf, 4096, 4096);  // 4KB 정렬된 버퍼 할당
read(fd, buf, 4096);               // 정렬된 크기로 읽기
```

```
Direct I/O 흐름
┌─────────────┐                      ┌──────────┐
│ 애플리케이션  │─────────────────────→│ 디스크    │
│ (유저 공간)   │←─────────────────────│          │
└─────────────┘  Page Cache 우회      └──────────┘
```

**왜 Direct I/O를 쓰는가?**

데이터베이스가 대표적이다. MySQL InnoDB, PostgreSQL, RocksDB 같은 스토리지 엔진은 자체 버퍼 풀을 운영한다. Page Cache와 자체 캐시에 같은 데이터가 **이중으로** 존재하면 메모리 낭비이고, 캐시 퇴거(eviction) 전략도 데이터베이스 엔진이 더 잘 알고 있다.

```
Buffered I/O + 데이터베이스             Direct I/O + 데이터베이스
┌───────────────┐                     ┌───────────────┐
│ DB Buffer Pool│ ← 데이터 중복!       │ DB Buffer Pool│ ← 유일한 캐시
├───────────────┤                     └───────┬───────┘
│ Page Cache    │ ← 여기에도 같은 데이터       │
├───────────────┤                             ↓
│ 디스크         │                      ┌───────────┐
└───────────────┘                      │ 디스크      │
                                       └───────────┘
```

---

## 5. fsync — 데이터 영속성 보장

### fsync가 필요한 이유

Buffered I/O에서 `write()`는 데이터가 Page Cache에 들어간 시점에 성공을 반환한다. 디스크에 실제로 기록되었다는 보장이 없다. 전원이 꺼지면 데이터가 유실된다.

```c
write(fd, data, len);   // Page Cache에만 기록됨
fsync(fd);              // 이 시점에 디스크에 실제로 기록됨이 보장
```

### fsync vs fdatasync vs sync

```
sync()       — 시스템 전체의 dirty page를 플러시 요청 (비동기, 기다리지 않음)
fsync(fd)    — fd의 데이터 + 메타데이터를 디스크에 기록 (완료까지 블록)
fdatasync(fd)— fd의 데이터만 디스크에 기록 (파일 크기 변경 없으면 메타데이터 스킵)
```

`fdatasync()`가 `fsync()`보다 빠를 수 있다. 메타데이터(수정 시간 등)의 디스크 쓰기를 건너뛸 수 있기 때문이다. 로그 파일처럼 append만 하는 경우에는 파일 크기가 변하므로 `fdatasync()`도 메타데이터를 써야 해서 차이가 없다.

### fsync의 성능 비용

fsync는 비싸다. 디스크의 쓰기 캐시까지 플러시해야 하므로, HDD에서는 한 번에 수 밀리초, SSD에서도 수백 마이크로초가 소요된다. 매 write마다 fsync를 호출하면 처리량이 급격히 떨어진다.

실제 서버 소프트웨어는 이를 최적화한다:
- **그룹 커밋 (Group Commit)**: 여러 트랜잭션의 fsync를 모아서 한 번에 수행 (MySQL, PostgreSQL)
- **WAL (Write-Ahead Log)**: 순차 쓰기인 로그 파일에만 fsync하고, 실제 데이터 파일은 나중에 기록
- **배터리 백업 캐시 (BBU)**: 하드웨어 RAID 컨트롤러에 배터리가 있으면, 디스크 캐시까지 플러시하지 않아도 안전

```
fsync 없이                    매번 fsync              그룹 커밋
write → write → write        write → fsync            write ─┐
    → write → write          write → fsync            write ─┤→ fsync
        (빠르지만 위험)         (안전하지만 느림)        write ─┘
                                                      (안전 + 합리적 성능)
```

### 디렉토리 fsync — 잊기 쉬운 함정

새 파일을 생성할 때, 파일 내용뿐만 아니라 **디렉토리 엔트리**도 디스크에 기록되어야 한다. 그렇지 않으면 크래시 후 파일이 사라질 수 있다.

```c
int fd = open("/data/new_file.dat", O_CREAT | O_WRONLY, 0644);
write(fd, data, len);
fsync(fd);         // 파일 내용은 안전
close(fd);

int dir_fd = open("/data", O_RDONLY);
fsync(dir_fd);     // 디렉토리 엔트리도 안전
close(dir_fd);
```

많은 애플리케이션이 디렉토리 fsync를 빠뜨려서, 크래시 후 "파일이 0바이트"가 되는 문제를 겪는다. ext4의 `data=journal` 모드나 파일 시스템의 자체 보장에 의존하는 것도 방법이지만, 안전하게 하려면 디렉토리 fsync를 명시적으로 수행해야 한다.

---

## 6. 저널링 파일 시스템

### 크래시 일관성 문제

파일에 데이터를 쓰는 작업은 내부적으로 여러 단계로 이루어진다:
1. 데이터 블록 기록
2. inode 업데이트 (크기, 블록 포인터)
3. 블록 비트맵 업데이트 (사용 중 표시)

이 중간에 크래시가 발생하면?

```
정상 완료:  데이터 ✓  inode ✓  비트맵 ✓  → 문제 없음
부분 완료1: 데이터 ✓  inode ✗  비트맵 ✗  → 데이터 블록이 미아 (공간 누수)
부분 완료2: 데이터 ✗  inode ✓  비트맵 ✓  → inode가 쓰레기 데이터를 가리킴
부분 완료3: 데이터 ✓  inode ✓  비트맵 ✗  → 같은 블록이 중복 할당될 수 있음
```

### 저널링의 원리

저널링 파일 시스템은 **실제 변경을 적용하기 전에 변경 내용을 저널(로그)에 먼저 기록**한다. Write-Ahead Logging(WAL)과 같은 원리다.

```
기록 과정:
1. 저널에 "이런 변경을 할 예정" 기록 (journal write)
2. 저널 기록 완료 확인 (journal commit)
3. 실제 위치에 변경 적용 (checkpoint)
4. 저널 항목 삭제

크래시 복구:
- 저널 커밋 완료된 항목 → 실제 위치에 재적용 (replay)
- 저널 커밋 미완료 항목 → 폐기 (변경 없었던 것으로)
```

### ext4의 저널 모드

ext4는 세 가지 저널 모드를 제공한다:

```
모드               저널에 기록하는 것        안전성    성능
───────────────────────────────────────────────────────
journal            데이터 + 메타데이터       최고      최저
ordered (기본)     메타데이터만              높음      중간
                   (데이터를 먼저 쓴 후
                    메타데이터 저널 기록)
writeback          메타데이터만              보통      최고
                   (데이터/메타 순서 보장 X)
```

`ordered` 모드가 기본인 이유: 메타데이터만 저널링하면서도, 데이터를 항상 메타데이터보다 먼저 쓰도록 순서를 보장한다. 크래시 후에도 inode가 쓰레기 데이터를 가리키는 일이 없다.

```bash
# 현재 파일 시스템의 저널 모드 확인
$ mount | grep "on / "
/dev/sda1 on / type ext4 (rw,relatime,errors=remount-ro,data=ordered)

# 저널 크기와 상태
$ sudo dumpe2fs /dev/sda1 | grep -i journal
Journal inode:            8
Journal backup:           inode blocks
Journal features:         journal_incompat_revoke journal_64bit
Journal size:             128M
```

### 다른 파일 시스템들

| 파일 시스템 | 특성 | 대표 사용처 |
|---|---|---|
| **ext4** | 안정적, 범용, 저널링 | 대부분의 리눅스 서버 |
| **XFS** | 대용량 파일/파티션에 최적화, 병렬 I/O 우수 | 대규모 스토리지, Red Hat 기본 |
| **Btrfs** | CoW(Copy-on-Write), 스냅샷, 체크섬 | NAS, 데이터 무결성 중시 환경 |
| **ZFS** | CoW, RAID 통합, 압축, 중복 제거 | 대규모 스토리지, Proxmox |

CoW(Copy-on-Write) 파일 시스템은 저널링과 다른 접근을 취한다. 데이터를 제자리에서 수정하지 않고, 항상 새 위치에 쓴 다음 포인터를 교체한다. 원자적으로 포인터를 교체하므로 중간 상태가 존재하지 않는다.

---

## 7. I/O 스케줄러 — 디스크 요청의 교통 정리

### I/O 스케줄러의 역할

여러 프로세스가 동시에 디스크 I/O를 요청하면, 커널의 I/O 스케줄러가 요청 순서를 재배치하여 처리량을 최적화한다. 특히 HDD에서는 헤드 이동(seek)을 최소화하는 것이 핵심이다.

### 리눅스 I/O 스케줄러 종류

**mq-deadline (Multi-Queue Deadline)**
```
요청 큐:
├── 정렬 큐 (섹터 순서로 정렬) → 탐색 최소화
└── 만료 큐 (도착 시간 순서)   → 기아 방지

- 읽기 만료: 500ms
- 쓰기 만료: 5000ms
- 읽기 우선 (읽기는 프로세스가 블록되므로)
```

**BFQ (Budget Fair Queuing)**
```
- 프로세스별 I/O 예산(budget) 할당
- 대화형 작업(짧은 I/O burst)에 높은 우선순위
- 데스크탑과 지연 시간 민감 환경에 적합
- SSD에서도 공정한 대역폭 분배 제공
```

**none (No-op)**
```
- 스케줄링 없이 FIFO 순서로 전달
- NVMe SSD에 적합 (하드웨어가 자체 스케줄링)
- 소프트웨어 오버헤드 최소화
```

```bash
# 디바이스의 현재 I/O 스케줄러 확인
$ cat /sys/block/sda/queue/scheduler
[mq-deadline] kyber bfq none

# NVMe SSD는 보통 none이 적합
$ cat /sys/block/nvme0n1/queue/scheduler
[none] mq-deadline kyber bfq

# 스케줄러 변경
$ echo "bfq" | sudo tee /sys/block/sda/queue/scheduler
```

**디바이스 유형별 권장 스케줄러:**
- **HDD**: `mq-deadline` 또는 `bfq` (seek 최소화가 중요)
- **SATA SSD**: `mq-deadline` (적당한 병합과 정렬)
- **NVMe SSD**: `none` (하드웨어 큐가 충분히 우수)

---

## 8. iostat — 디스크 성능 모니터링

### 기본 사용법

```bash
$ iostat -xz 1
# -x: 확장 통계, -z: 활동 없는 디바이스 숨김, 1: 1초 간격

Device   r/s    w/s   rkB/s   wkB/s  rrqm/s  wrqm/s  %rrqm  %wrqm  r_await  w_await  aqu-sz  rareq-sz  wareq-sz  svctm  %util
sda     45.00  120.00 1800.0  4800.0    5.00   30.00  10.0%  20.0%    1.20     2.50    0.45     40.0      40.0     2.10   34.7%
```

### 핵심 지표 해석

```
r/s, w/s        — 초당 읽기/쓰기 요청 수 (IOPS)
rkB/s, wkB/s    — 초당 읽기/쓰기 처리량 (KB)
r_await, w_await— 평균 읽기/쓰기 대기 시간 (ms) ★
aqu-sz          — 평균 큐 길이 (높으면 병목)
%util           — 디바이스 사용률 (100%에 가까우면 포화) ★
rrqm/s, wrqm/s  — 초당 병합된 요청 수 (인접 요청을 하나로 합침)
```

**디스크 병목 판단 기준:**

```
%util ≈ 100%  AND  await 급증     → 디스크가 포화 상태
%util 낮음    AND  await 높음     → 큰 I/O 또는 랜덤 패턴 문제
aqu-sz > 1    AND  await 정상     → 디스크가 잘 처리 중 (깊은 큐)
aqu-sz > 4~8                      → 병목 가능성 높음
```

주의: NVMe SSD에서 `%util`은 의미가 제한적이다. 내부적으로 수십 개의 병렬 큐를 가지므로 `%util`이 100%여도 실제로는 여유가 있을 수 있다. NVMe에서는 `await`와 `aqu-sz`에 더 집중해야 한다.

### 실전 진단 시나리오

**시나리오 1: 데이터베이스가 갑자기 느려졌다**

```bash
$ iostat -xz 1 3
Device  r/s    w/s   r_await  w_await  %util
sda     500    200   15.3     45.2     99.8%
```

`%util` 99.8%, `w_await` 45ms — 디스크가 완전히 포화 상태. 원인을 찾아보자:

```bash
# 어떤 프로세스가 I/O를 많이 발생시키는지
$ sudo iotop -oP
Total DISK READ: 200.0 MB/s | Total DISK WRITE: 80.0 MB/s
  PID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 3456  be/4  mysql    195.0 MB/s  75.0 MB/s   0.00 %  85.0 %  mysqld

# MySQL이 원인. 어떤 파일에 I/O가 집중되는지
$ sudo strace -p 3456 -e trace=read,write,fsync -c
```

**시나리오 2: wrqm/s가 0이고 w/s가 비정상적으로 높다**

```bash
Device  r/s    w/s   wrqm/s  wareq-sz  w_await  %util
sda     10     5000  0       4.0       8.5      95%
```

쓰기 병합이 전혀 안 되고(`wrqm/s=0`), 요청 크기가 4KB로 아주 작다. 소규모 랜덤 쓰기가 폭주하는 패턴이다. 애플리케이션이 매번 fsync를 호출하거나, O_DIRECT + O_SYNC로 열고 있을 가능성이 높다.

---

## 9. 디스크 병목 진단 종합 도구

### iotop — 프로세스별 I/O 사용량

```bash
# 실시간 프로세스별 I/O 모니터링
$ sudo iotop -oP
# -o: I/O 발생 중인 프로세스만, -P: 스레드 대신 프로세스 단위

# 누적 I/O 사용량
$ sudo iotop -oPa
```

### blktrace — 블록 I/O 추적

블록 레이어에서 일어나는 모든 I/O 이벤트를 기록한다. 매우 상세한 분석이 가능하지만 오버헤드도 크다.

```bash
# 10초간 블록 I/O 추적
$ sudo blktrace -d /dev/sda -w 10 -o trace

# 사람이 읽을 수 있는 형태로 변환
$ blkparse -i trace.blktrace.0 | head -20
  8,0    0    1     0.000000000  3456  Q  WS 123456 + 8 [mysqld]
  8,0    0    2     0.000001234  3456  G  WS 123456 + 8 [mysqld]
  8,0    0    3     0.000002345  3456  I  WS 123456 + 8 [mysqld]
  8,0    0    4     0.000003456  3456  D  WS 123456 + 8 [mysqld]
  8,0    0    5     0.001234567  3456  C  WS 123456 + 8 [0]

# Q=큐잉, G=할당, I=삽입, D=디스패치, C=완료
```

### /proc/diskstats — 시스템 레벨 통계

```bash
$ cat /proc/diskstats
 259    0 nvme0n1 12345 6789 234567 45678 89012 34567 890123 12345 0 56789 58023

# 필드 의미 (주요):
# 읽기 완료 수, 읽기 병합 수, 읽기 섹터 수, 읽기 시간(ms),
# 쓰기 완료 수, 쓰기 병합 수, 쓰기 섹터 수, 쓰기 시간(ms),
# 진행 중 I/O, I/O 소요 시간(ms), 가중 I/O 시간(ms)
```

### 병목 진단 체크리스트

```
1. iostat으로 전체 현황 파악
   └→ %util, await, aqu-sz 확인

2. 병목 확인되면 iotop으로 원인 프로세스 특정
   └→ 어떤 프로세스가 I/O를 유발하는가?

3. strace/lsof로 해당 프로세스의 I/O 패턴 분석
   └→ 어떤 파일? 랜덤 vs 순차? fsync 빈도?

4. 파일 시스템 레벨 확인
   └→ df -i (inode 고갈?), df -h (공간 부족?)
   └→ mount 옵션 확인 (noatime, data=ordered 등)

5. 하드웨어 레벨 확인
   └→ smartctl -a /dev/sda (디스크 건강 상태)
   └→ SSD인 경우 TRIM 설정 확인 (fstrim)
```

---

## 10. 실전 I/O 최적화 패턴

### noatime 마운트 옵션

리눅스는 기본적으로 파일을 읽을 때마다 **atime(접근 시간)**을 업데이트한다. 읽기 연산인데 쓰기가 발생하는 셈이다. 서버에서는 거의 불필요하므로 끄는 것이 좋다.

```bash
# /etc/fstab
/dev/sda1  /  ext4  defaults,noatime  0  1
#                            ↑ 읽기 시 atime 업데이트 안 함

# relatime (기본값): mtime보다 오래된 경우에만 atime 업데이트
# noatime: atime 업데이트 완전 비활성화
# 대부분의 서버에서 noatime 권장
```

### 순차 읽기 최적화 — readahead

커널은 순차 읽기 패턴을 감지하면 앞으로 읽을 데이터를 미리 Page Cache에 올려둔다.

```bash
# 현재 readahead 크기 확인 (단위: 512바이트 섹터)
$ cat /sys/block/sda/queue/read_ahead_kb
128

# 순차 읽기 워크로드에서 readahead 증가
$ echo 1024 | sudo tee /sys/block/sda/queue/read_ahead_kb

# 프로그래밍 레벨에서 힌트 제공
$ posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL);  // 순차 접근 예고
$ posix_fadvise(fd, 0, 0, POSIX_FADV_RANDOM);       // 랜덤 접근 예고
```

### 대량 파일 처리 — splice와 sendfile

파일을 네트워크로 전송할 때 전통적인 방식은 커널 → 유저 → 커널로 데이터가 두 번 복사된다. `sendfile()`은 이 복사를 제거한다.

```
전통적 방식:
디스크 → [커널 버퍼] → [유저 버퍼] → [커널 소켓 버퍼] → 네트워크
              복사 1         복사 2

sendfile():
디스크 → [커널 버퍼] ─────────────→ [커널 소켓 버퍼] → 네트워크
                    복사 없음 (zero-copy)
```

Nginx의 `sendfile on;` 설정이 바로 이것이다. 정적 파일 서빙 성능을 크게 향상시킨다.

```nginx
# nginx.conf
http {
    sendfile on;
    tcp_nopush on;   # sendfile과 함께 사용, 헤더+데이터를 한 패킷으로
}
```

### tmpfs — 메모리 기반 파일 시스템

임시 파일이나 빈번하게 접근하는 작은 데이터는 tmpfs에 두면 디스크 I/O를 완전히 제거할 수 있다.

```bash
# /tmp를 tmpfs로 마운트 (리부트 시 데이터 유실)
$ mount -t tmpfs -o size=2G tmpfs /tmp

# 이미 마운트되어 있는 tmpfs 확인
$ df -h -t tmpfs
Filesystem      Size  Used  Avail  Use%  Mounted on
tmpfs           2.0G  156M  1.9G    8%   /tmp
tmpfs           3.9G     0  3.9G    0%   /dev/shm
```

`/dev/shm`은 공유 메모리용 tmpfs로, 프로세스 간 대용량 데이터 공유에 유용하다.

---

## 마치며

파일 시스템과 I/O는 서버 성능의 기반이다. 핵심을 정리하면:

- **fd**는 리눅스 I/O의 보편적 인터페이스이고, 그 한계(`ulimit`)를 아는 것은 서버 운영의 기본이다
- **inode**는 파일의 실체이며, 이름과 분리된 구조가 하드 링크와 심볼릭 링크를 가능하게 한다
- **Buffered I/O**는 성능을, **Direct I/O**는 제어권을, **fsync**는 영속성을 제공한다 — 셋의 균형이 중요하다
- **저널링**은 크래시 일관성을 보장하며, ext4의 `ordered` 모드가 성능과 안전성의 합리적 타협점이다
- **I/O 스케줄러**는 디바이스 특성에 맞게 선택해야 하며, NVMe 시대에는 `none`이 기본이 되었다
- **iostat + iotop**은 디스크 병목 진단의 출발점이다

다음 편에서는 네트워크의 세계로 들어간다. TCP/IP 스택을 커널 레벨에서 분석하고, 소켓 프로그래밍의 본질을 탐구한다.
