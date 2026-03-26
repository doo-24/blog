---
title: "[Docker & 컨테이너] 1편 — 컨테이너 원리: Docker가 격리를 만드는 방법"
date: 2026-03-20T14:00:00+09:00
draft: false
tags: ["Docker", "컨테이너", "namespace", "cgroup", "chroot", "서버"]
series: ["Docker와 컨테이너"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 11
summary: "VM과 컨테이너의 차이, Linux namespace로 프로세스/네트워크/파일시스템 격리, cgroup으로 CPU/메모리 제한, Union FS와 레이어 구조, Docker 아키텍처(daemon, containerd, runc)까지"
---

컨테이너는 "가벼운 VM"이 아니다.

이 말은 자주 들리지만, 단순히 "더 빠르고 작다"는 수준에서 이해하는 경우가 많다.

VM은 하드웨어를 가상화한다. 하이퍼바이저가 CPU, 메모리, 디스크, 네트워크 장치를 소프트웨어로 흉내 내고, 그 위에서 Guest OS가 완전히 독립적으로 동작한다.

반면 컨테이너는 호스트의 커널을 그대로 공유하면서 **프로세스 수준에서 격리**한다.

이 차이는 철학의 차이다. 그리고 그 철학이 Docker의 내부 구현 전체를 결정한다.

Docker는 마법이 아니다. 리눅스 커널이 이미 제공하는 세 가지 기술 — **namespace**, **cgroup**, **Union File System** — 을 조합한 것이다. 이 글에서는 그 세 가지가 각각 무엇을 하는지, 어떻게 합쳐져서 컨테이너라는 개념을 만드는지 살펴본다.

---

## 1. VM vs 컨테이너 — 근본적인 차이

### 하이퍼바이저 방식: 하드웨어를 가상화한다

VM은 하이퍼바이저(Hypervisor) 위에서 동작한다. 하이퍼바이저는 두 가지 유형으로 나뉜다.

**Type 1 하이퍼바이저 (Bare-metal)**

하드웨어 위에서 직접 동작한다. 별도의 Host OS 없이 하이퍼바이저 자체가 하드웨어를 관리한다. VMware ESXi, Microsoft Hyper-V, Xen이 여기에 해당한다. 데이터센터의 서버 가상화에 주로 쓰인다.

```
┌─────────────────────────────────┐
│   VM1 (Guest OS + App)          │
│   VM2 (Guest OS + App)          │
├─────────────────────────────────┤
│   Type 1 Hypervisor             │
├─────────────────────────────────┤
│   Physical Hardware             │
└─────────────────────────────────┘
```

**Type 2 하이퍼바이저 (Hosted)**

Host OS 위에서 일반 애플리케이션처럼 동작한다. VirtualBox, VMware Workstation이 여기에 해당한다. 개발자 노트북에서 테스트 환경을 만들 때 흔히 사용한다.

```
┌─────────────────────────────────┐
│   VM1 (Guest OS + App)          │
│   VM2 (Guest OS + App)          │
├─────────────────────────────────┤
│   Type 2 Hypervisor (App)       │
├─────────────────────────────────┤
│   Host OS                       │
├─────────────────────────────────┤
│   Physical Hardware             │
└─────────────────────────────────┘
```

각 VM은 완전한 Guest OS를 포함한다. Ubuntu VM이라면 Ubuntu 커널, 시스템 라이브러리, 데몬 프로세스까지 전부 포함된다. 이미지 크기가 수 GB에 달하고, 부팅하는 데 수십 초가 걸리는 이유가 여기에 있다.

### 컨테이너 방식: OS를 공유하고 프로세스를 격리한다

컨테이너는 전혀 다른 접근 방식을 취한다. Guest OS를 따로 갖지 않는다. 대신 Host OS의 커널을 직접 공유하면서, 리눅스 커널의 격리 메커니즘(namespace, cgroup)을 활용해 프로세스가 서로를 "볼 수 없게" 만든다.

```
┌────────────┬────────────┬────────────┐
│ Container1 │ Container2 │ Container3 │
│  App + Lib │  App + Lib │  App + Lib │
├────────────┴────────────┴────────────┤
│   Container Runtime (containerd)     │
├──────────────────────────────────────┤
│   Host OS Kernel (공유)              │
├──────────────────────────────────────┤
│   Physical Hardware                  │
└──────────────────────────────────────┘
```

각 컨테이너는 애플리케이션과 그것이 의존하는 라이브러리만 포함한다. 이미지 크기가 수 MB 수준으로 줄고, 시작 시간이 수백 밀리초로 단축되는 이유다.

### 비교표

| 항목 | VM | 컨테이너 |
|------|-----|---------|
| 격리 단위 | 하드웨어 수준 | 프로세스 수준 |
| OS | Guest OS 포함 | Host 커널 공유 |
| 이미지 크기 | 수 GB | 수 MB ~ 수백 MB |
| 시작 시간 | 수십 초 ~ 수 분 | 수백 ms ~ 수 초 |
| 메모리 오버헤드 | 높음 (OS per VM) | 낮음 (프로세스 수준) |
| 보안 격리 | 강함 | 상대적으로 약함 |
| 이식성 | OS에 종속적 | 컨테이너 런타임만 있으면 동작 |
| 리소스 밀도 | 낮음 | 높음 |

### 트레이드오프: 컨테이너의 보안 격리는 약하다

컨테이너는 호스트 커널을 공유하기 때문에, 커널 취약점이 발견되면 모든 컨테이너가 영향을 받는다. 하나의 컨테이너에서 커널 익스플로잇이 성공하면 Host 전체를 장악할 수 있다. VM에서라면 Guest OS 커널을 공격해야 하고, 그 다음 하이퍼바이저까지 뚫어야 Host에 접근할 수 있다.

금융권이나 보안이 극도로 중요한 환경에서 VM을 계속 사용하는 이유가 여기에 있다. Kata Containers나 gVisor 같은 프로젝트는 이 트레이드오프를 해소하기 위해 경량 하이퍼바이저와 컨테이너를 결합한 접근 방식을 취한다.

---

## 2. Linux Namespace — 프로세스가 보는 세계를 분리한다

Namespace는 리눅스 커널의 기능으로, 프로세스가 시스템 자원을 어떻게 "보는지"를 제어한다. 동일한 시스템에서 동작하지만, 서로 다른 namespace에 속한 프로세스들은 각자 독립된 시스템을 가진 것처럼 동작한다.

현재 리눅스 커널은 7종류의 namespace를 제공한다.

### Namespace 7종 개요

| Namespace | 격리 대상 | 도입 버전 |
|-----------|---------|----------|
| PID | 프로세스 ID | 커널 2.6.24 |
| Network | 네트워크 인터페이스, 라우팅, 포트 | 커널 2.6.24 |
| Mount | 파일시스템 마운트 포인트 | 커널 2.4.19 |
| UTS | 호스트명, 도메인명 | 커널 2.6.19 |
| IPC | System V IPC, POSIX 메시지 큐 | 커널 2.6.19 |
| User | 사용자 ID, 그룹 ID | 커널 3.8 |
| Cgroup | cgroup 루트 디렉터리 | 커널 4.6 |

### PID Namespace — 프로세스 ID 격리

PID namespace는 프로세스 ID 번호 공간을 격리한다. 새 PID namespace에서 최초로 생성된 프로세스는 PID 1을 부여받는다. 이 프로세스는 해당 namespace 안에서 init 프로세스 역할을 한다.

```bash
# 새 PID namespace에서 bash 실행
sudo unshare --pid --fork --mount-proc bash

# 이 bash 안에서 프로세스 목록 확인
ps aux
```

```
# 출력 결과 — 호스트의 수백 개 프로세스는 보이지 않는다
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  22744  5232 pts/0    S    14:01   0:00 bash
root        12  0.0  0.0  37368  3220 pts/0    R+   14:01   0:00 ps aux
```

이 namespace 안의 bash는 자신이 PID 1이라고 알고 있다. 호스트에서 보면 실제 PID는 훨씬 큰 숫자지만, namespace 안에서는 1이다. Docker 컨테이너 안에서 `ps aux`를 실행했을 때 PID가 1부터 시작하는 이유가 이것이다.

```bash
# 호스트에서 해당 프로세스의 실제 PID 확인
ps aux | grep bash
# root  4823  ... bash  <- 호스트에서의 실제 PID
```

PID namespace는 중첩(nested)이 가능하다. 부모 namespace에서 자식 namespace의 프로세스를 볼 수 있지만, 반대는 불가능하다. 이 단방향성이 격리의 핵심이다.

### Network Namespace — 네트워크 스택 격리

Network namespace는 네트워크 인터페이스, 라우팅 테이블, iptables 규칙, 소켓을 격리한다. 각 컨테이너가 독립적인 `eth0` 인터페이스와 IP 주소를 갖는 이유가 바로 이 namespace 덕분이다.

```bash
# 새 네트워크 namespace 생성
sudo ip netns add myns

# 새 namespace에서 네트워크 인터페이스 확인
sudo ip netns exec myns ip link show
```

```
# 출력 — loopback만 있고, 호스트의 eth0, docker0 등은 보이지 않는다
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Docker는 컨테이너를 생성할 때 veth(Virtual Ethernet) 쌍을 만든다. 한쪽 끝은 컨테이너의 network namespace 안에 `eth0`로, 다른 끝은 호스트의 `docker0` 브리지에 연결된다. 이것이 컨테이너가 외부와 통신하는 방법이다.

```bash
# 컨테이너 실행 후 호스트에서 veth 인터페이스 확인
docker run -d --name mycontainer nginx
ip link show | grep veth
# veth3a2b1c4@if5: ...  <- 이 쪽이 호스트
```

포트 매핑(`-p 8080:80`)은 iptables의 DNAT 규칙으로 구현된다. 호스트의 8080 포트로 들어오는 패킷을 컨테이너 IP의 80 포트로 변환한다.

```bash
# Docker가 생성한 iptables 규칙 확인
sudo iptables -t nat -L -n | grep 8080
# DNAT  tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80
```

### Mount Namespace — 파일시스템 격리

Mount namespace는 마운트 포인트 목록을 격리한다. 각 컨테이너가 독립적인 파일시스템 뷰를 갖도록 한다.

```bash
# 새 mount namespace에서 bash 실행
sudo unshare --mount bash

# 이 안에서 마운트 포인트 조작 — 호스트에 영향 없음
mount --bind /tmp /mnt
ls /mnt  # /tmp 내용이 보임

# 다른 터미널(호스트)에서 확인
ls /mnt  # 영향 없음 — 빈 디렉터리
```

Mount namespace와 `chroot`를 결합하면 완전히 독립적인 파일시스템 환경을 만들 수 있다. `chroot`는 프로세스의 루트 디렉터리(`/`)를 변경한다. 컨테이너 안에서 `ls /`를 실행하면 호스트의 루트가 아닌 컨테이너 이미지의 루트를 보는 이유다.

```bash
# chroot 동작 원리 직접 체험
# 먼저 최소한의 루트 파일시스템 준비 (Alpine Linux rootfs 사용)
mkdir /tmp/myroot
tar -xzf alpine-minirootfs.tar.gz -C /tmp/myroot

# 이 디렉터리를 새 루트로 사용
sudo chroot /tmp/myroot /bin/sh

# 이제 / 는 /tmp/myroot 다
ls /
# bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

### UTS Namespace — 호스트명 격리

UTS(Unix Time-sharing System) namespace는 `hostname`과 `domainname`을 격리한다. 컨테이너마다 다른 호스트명을 가질 수 있는 이유다.

```bash
# 새 UTS namespace에서 호스트명 변경
sudo unshare --uts bash
hostname mycontainer
hostname
# mycontainer

# 다른 터미널(호스트)에서 확인
hostname
# original-host  <- 영향 없음
```

Docker는 컨테이너를 생성할 때 기본적으로 컨테이너 ID의 앞 12자를 호스트명으로 설정한다. `--hostname` 플래그로 변경할 수 있다.

### IPC Namespace — 프로세스 간 통신 격리

IPC namespace는 System V IPC 객체(메시지 큐, 세마포어, 공유 메모리)와 POSIX 메시지 큐를 격리한다. 서로 다른 컨테이너의 프로세스가 실수로 공유 메모리를 통해 통신하는 사고를 막는다.

### User Namespace — 사용자 ID 격리

User namespace는 사용자 ID(UID)와 그룹 ID(GID)를 격리한다. 가장 강력하고 복잡한 namespace다.

핵심 기능은 UID 매핑이다. 컨테이너 안에서 UID 0(root)로 보이는 프로세스가 호스트에서는 일반 사용자 UID로 실행될 수 있다. rootless 컨테이너의 기반 기술이다.

```bash
# user namespace 없이 — 호스트 root 필요
sudo unshare --pid --fork --mount-proc bash
id
# uid=0(root) gid=0(root) groups=0(root)

# user namespace와 함께 — 일반 사용자가 컨테이너 안에서 root처럼 동작
unshare --user --pid --fork --mount-proc bash
id
# uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
# 이 상태에서 별도의 uid_map 설정으로 컨테이너 내 root 매핑 가능
```

### Cgroup Namespace

cgroup namespace는 각 컨테이너가 독립적인 cgroup 계층 뷰를 갖도록 한다. 컨테이너 안에서 자신이 어느 cgroup에 속해 있는지 숨긴다. 다음 섹션에서 cgroup 자체를 자세히 다룬다.

### 컨테이너 = Namespace들의 조합

Docker가 컨테이너를 생성할 때 내부적으로 하는 일을 단순화하면 다음과 같다.

```bash
# Docker 컨테이너 생성의 핵심 동작 (의사 코드)

# 1. 새로운 namespace들 생성
unshare \
  --pid \          # PID namespace
  --net \          # Network namespace
  --mount \        # Mount namespace
  --uts \          # UTS namespace
  --ipc \          # IPC namespace
  --user \         # User namespace (rootless의 경우)
  --fork \
  bash

# 2. 컨테이너 이미지의 rootfs를 마운트하고 chroot
mount --bind /var/lib/docker/overlay2/.../merged /container/rootfs
chroot /container/rootfs

# 3. cgroup으로 자원 제한 설정
echo 512M > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes
echo 100000 > /sys/fs/cgroup/cpu/mycontainer/cpu.cfs_quota_us

# 4. 프로세스 실행
exec /bin/app
```

실제로는 `runc`가 OCI 스펙에 맞춰 이 모든 과정을 처리한다. Docker는 `runc`를 호출할 뿐이다.

---

## 3. cgroup — 자원 사용량을 제한한다

Namespace가 "프로세스가 무엇을 볼 수 있는지"를 제어한다면, cgroup(Control Groups)은 "프로세스가 얼마나 많은 자원을 쓸 수 있는지"를 제어한다.

### cgroup v1 vs cgroup v2

리눅스 cgroup에는 두 버전이 있다. 현재 대부분의 최신 리눅스 배포판은 cgroup v2를 기본으로 사용하지만, 오래된 시스템에서는 v1을 쓰거나 혼용하는 경우도 있다.

| 항목 | cgroup v1 | cgroup v2 |
|------|-----------|-----------|
| 도입 | 커널 2.6.24 (2008) | 커널 4.5 (2016) |
| 계층 구조 | 리소스 유형별 독립 계층 | 단일 통합 계층 |
| 마운트 경로 | `/sys/fs/cgroup/<subsystem>/` | `/sys/fs/cgroup/` |
| 컨트롤러 | 각 독립 | 통합 관리 |
| 스레드 지원 | 미흡 | 개선됨 |
| 메모리 swap | memory.memsw.limit_in_bytes | memory.swap.max |

```bash
# 현재 시스템의 cgroup 버전 확인
stat -fc %T /sys/fs/cgroup/
# tmpfs  -> cgroup v1
# cgroup2fs -> cgroup v2

# cgroup v2 마운트 확인
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime)
```

### CPU 제한

cgroup v1에서 CPU 시간을 제한하는 핵심 파라미터는 `cpu.cfs_quota_us`와 `cpu.cfs_period_us`다.

- `cpu.cfs_period_us`: CPU 할당 주기 (마이크로초). 기본값 100,000 (=100ms)
- `cpu.cfs_quota_us`: 한 주기 안에서 사용 가능한 CPU 시간 (마이크로초)

quota / period = CPU 코어 비율

```bash
# cgroup v1: CPU를 0.5 코어로 제한하는 예시
sudo mkdir /sys/fs/cgroup/cpu/myapp

# 100ms 주기 동안 50ms만 사용 가능 = 0.5 CPU
echo 100000 > /sys/fs/cgroup/cpu/myapp/cpu.cfs_period_us
echo 50000  > /sys/fs/cgroup/cpu/myapp/cpu.cfs_quota_us

# 현재 프로세스를 이 cgroup에 추가
echo $$ > /sys/fs/cgroup/cpu/myapp/tasks

# 검증: stress 도구로 CPU 100% 사용 시도
stress --cpu 4 &
# 실제 CPU 사용률은 50%에서 제한됨
top  # CPU 사용률 확인
```

cgroup v2에서는 `cpu.max` 파일 하나로 설정한다.

```bash
# cgroup v2: CPU를 0.5 코어로 제한
sudo mkdir /sys/fs/cgroup/myapp
echo "50000 100000" > /sys/fs/cgroup/myapp/cpu.max
# 형식: <quota> <period>
```

Docker의 `--cpus` 플래그가 내부적으로 이 값을 설정한다.

```bash
# Docker --cpus 플래그
docker run --cpus="1.5" myimage

# 내부적으로 설정되는 cgroup 값
# cfs_period_us = 100000
# cfs_quota_us  = 150000  (1.5 * 100000)
```

```bash
# 실행 중인 컨테이너의 cgroup 설정 확인
CONTAINER_ID=$(docker inspect --format '{{.Id}}' mycontainer)

# cgroup v1
cat /sys/fs/cgroup/cpu/docker/${CONTAINER_ID}/cpu.cfs_quota_us

# cgroup v2
cat /sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope/cpu.max
```

### 메모리 제한과 OOM Killer

메모리 제한은 `memory.limit_in_bytes`(v1) 또는 `memory.max`(v2)로 설정한다.

```bash
# cgroup v1: 메모리 512MB로 제한
sudo mkdir /sys/fs/cgroup/memory/myapp
echo 536870912 > /sys/fs/cgroup/memory/myapp/memory.limit_in_bytes
# 536870912 = 512 * 1024 * 1024

# 현재 프로세스를 cgroup에 추가
echo $$ > /sys/fs/cgroup/memory/myapp/tasks
```

```bash
# cgroup v2: 메모리 512MB로 제한
echo "512M" > /sys/fs/cgroup/myapp/memory.max
```

프로세스가 제한을 초과하면 리눅스 커널의 OOM Killer(Out Of Memory Killer)가 동작한다. OOM Killer는 메모리를 가장 많이 사용하면서 "중요도"가 낮은 프로세스를 골라 강제 종료한다.

```bash
# OOM Killer 동작 확인
dmesg | grep -i "oom"
# [12345.678901] oom-kill:constraint=CONSTRAINT_MEMCG,...,task=myapp,pid=1234,...

# Docker 컨테이너가 OOM으로 종료된 경우 확인
docker inspect mycontainer | grep OOMKilled
# "OOMKilled": true
```

Docker에서 메모리 제한 설정:

```bash
# 메모리 512MB, 스왑 포함 1GB 제한
docker run --memory="512m" --memory-swap="1g" myimage

# 스왑 사용 금지 (메모리만 512MB)
docker run --memory="512m" --memory-swap="512m" myimage

# 메모리 제한만 설정하면 기본적으로 스왑도 같은 크기로 설정됨
docker run --memory="512m" myimage
# 실제 총 메모리+스왑 = 1GB
```

### 실제 cgroup 설정 확인

```bash
# 실행 중인 컨테이너 ID 조회
docker run -d --name testapp --memory="256m" --cpus="0.5" alpine sleep 3600
CONTAINER_ID=$(docker inspect --format '{{.Id}}' testapp)

# cgroup v2 기준으로 설정 확인
BASE="/sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope"

cat ${BASE}/memory.max
# 268435456  (256MB = 256 * 1024 * 1024)

cat ${BASE}/cpu.max
# 50000 100000  (0.5 CPU)

# cgroup v1 기준으로 확인
cat /sys/fs/cgroup/memory/docker/${CONTAINER_ID}/memory.limit_in_bytes
cat /sys/fs/cgroup/cpu/docker/${CONTAINER_ID}/cpu.cfs_quota_us
```

---

## 4. Union File System과 레이어 구조

컨테이너가 시작할 때마다 이미지 전체를 복사한다면 디스크와 시간이 엄청나게 낭비된다. Docker는 Union File System을 사용해 이 문제를 해결한다.

### Copy-on-Write(CoW) 전략

CoW는 데이터를 공유하되, 수정이 필요할 때만 복사하는 전략이다. 읽기는 원본을 공유하고, 쓰기가 발생하는 순간 해당 데이터를 복사해 새 위치에 쓴다. 이미지 레이어를 여러 컨테이너가 공유할 수 있는 이유다.

### OverlayFS: Docker의 기본 Union FS

현재 Docker의 기본 스토리지 드라이버는 `overlay2`다. OverlayFS는 두 개의 디렉터리를 하나로 합쳐 보이게 한다.

```
lower (읽기 전용 이미지 레이어들)
upper (쓰기 가능한 컨테이너 레이어)
work  (OverlayFS 내부 작업 디렉터리)
merged (컨테이너가 실제로 보는 합쳐진 뷰)
```

직접 OverlayFS를 마운트해보면 동작 방식을 이해할 수 있다.

```bash
# OverlayFS 동작 원리 직접 체험
mkdir -p /tmp/overlay/{lower,upper,work,merged}

# lower 디렉터리에 기반 파일 생성
echo "이미지에 포함된 파일" > /tmp/overlay/lower/base.txt
echo "이미지의 설정 파일" > /tmp/overlay/lower/config.txt

# OverlayFS 마운트
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/overlay/lower,\
upperdir=/tmp/overlay/upper,\
workdir=/tmp/overlay/work \
  /tmp/overlay/merged

# merged 뷰에서 파일 확인 — lower의 파일이 보임
ls /tmp/overlay/merged
# base.txt  config.txt

# merged에서 파일 수정 (CoW 발생)
echo "컨테이너에서 수정된 내용" >> /tmp/overlay/merged/config.txt

# lower의 원본은 그대로
cat /tmp/overlay/lower/config.txt
# 이미지의 설정 파일

# upper에 수정본이 생성됨
cat /tmp/overlay/upper/config.txt
# 이미지의 설정 파일
# 컨테이너에서 수정된 내용

# 새 파일 생성
echo "컨테이너에서 새로 만든 파일" > /tmp/overlay/merged/new.txt

# upper에 새 파일이 저장됨
ls /tmp/overlay/upper
# config.txt  new.txt
ls /tmp/overlay/lower
# base.txt  config.txt  <- 변화 없음
```

Docker 컨테이너의 파일 삭제는 whiteout 파일로 구현된다. 하위 레이어의 파일을 실제로 삭제하는 것이 아니라, 해당 파일을 숨기는 특수 파일을 upper 레이어에 생성한다.

```bash
# 컨테이너에서 파일 삭제
rm /tmp/overlay/merged/base.txt

# upper에 whiteout 파일 생성됨 (크기 0, 특수 파일)
ls -la /tmp/overlay/upper
# c---------  1 root root 0, 0 Mar 20 14:00 base.txt  <- whiteout 파일

# merged에서는 삭제된 것으로 보임
ls /tmp/overlay/merged
# config.txt  new.txt  <- base.txt 없음
```

### 이미지 레이어 구조

Docker 이미지는 여러 레이어의 스택이다. 각 레이어는 이전 레이어 위에 변경사항(파일 추가, 수정, 삭제)을 기록한다. 레이어는 읽기 전용이며, 콘텐츠의 SHA256 해시로 식별된다.

```dockerfile
# 이 Dockerfile은 레이어를 어떻게 생성하는지 보여준다
FROM ubuntu:22.04          # 레이어 1: Ubuntu base 이미지

RUN apt-get update && \    # 레이어 2: apt 패키지 업데이트
    apt-get install -y curl

COPY app.jar /app/         # 레이어 3: app.jar 복사

ENV APP_PORT=8080           # 레이어 4: 환경변수 설정 (메타데이터 레이어)

CMD ["java", "-jar", "/app/app.jar"]  # 레이어 5: 실행 명령 (메타데이터)
```

`docker history` 명령으로 이미지의 레이어를 확인할 수 있다.

```bash
docker history ubuntu:22.04
```

```
IMAGE          CREATED        CREATED BY                                      SIZE
174c8c134b2a   3 weeks ago    /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>      3 weeks ago    /bin/sh -c #(nop) ADD file:abc123... in /       77.8MB
```

```bash
# 애플리케이션 이미지의 레이어 확인
docker history myapp:latest
```

```
IMAGE          CREATED       CREATED BY                                      SIZE
a1b2c3d4e5f6   2 hours ago   /bin/sh -c #(nop)  CMD ["java","-jar","/a...   0B
b2c3d4e5f6a7   2 hours ago   /bin/sh -c #(nop)  ENV APP_PORT=8080            0B
c3d4e5f6a7b8   2 hours ago   /bin/sh -c #(nop) COPY app.jar /app/           52.4MB
d4e5f6a7b8c9   3 hours ago   /bin/sh -c apt-get update && apt-get insta...  42.1MB
e5f6a7b8c9d0   3 weeks ago   /bin/sh -c #(nop) ADD file:... in /            77.8MB
```

### 레이어 공유의 효율성

동일한 베이스 이미지를 사용하는 여러 이미지를 가져오면, 공통 레이어는 한 번만 다운로드된다.

```bash
# ubuntu:22.04 기반 이미지 두 개를 pull할 때
docker pull myapp-v1:latest   # ubuntu 레이어 다운로드 (77.8MB)
docker pull myapp-v2:latest   # ubuntu 레이어는 이미 로컬에 있음 — 스킵

# 디스크에서 실제 레이어 위치 확인
docker inspect myapp-v1:latest | jq '.[0].GraphDriver.Data'
# {
#   "LowerDir": "/var/lib/docker/overlay2/abc.../diff:...",
#   "MergedDir": "/var/lib/docker/overlay2/xyz.../merged",
#   "UpperDir": "/var/lib/docker/overlay2/xyz.../diff",
#   "WorkDir": "/var/lib/docker/overlay2/xyz.../work"
# }
```

### 컨테이너 레이어: 쓰기 가능한 최상위 레이어

컨테이너가 시작되면 읽기 전용 이미지 레이어 위에 쓰기 가능한 컨테이너 레이어가 추가된다. 컨테이너가 삭제되면 이 레이어도 함께 삭제된다. 이미지 레이어는 그대로 남는다.

```
이미지 레이어 (읽기 전용)     컨테이너 레이어 (쓰기 가능)
┌─────────────────────┐      ┌─────────────────────┐
│ Layer 4 (CMD)       │      │ Container Layer      │ ← 컨테이너 실행 시 추가
├─────────────────────┤      │ (삭제 시 함께 제거)  │
│ Layer 3 (ENV)       │      └─────────────────────┘
├─────────────────────┤
│ Layer 2 (COPY)      │
├─────────────────────┤
│ Layer 1 (RUN apt)   │
├─────────────────────┤
│ Layer 0 (FROM)      │
└─────────────────────┘
```

컨테이너 안에서 생성한 파일을 영구적으로 보존하려면 볼륨(Volume)을 사용해야 한다. 볼륨은 Union FS 바깥에 존재하는 호스트의 실제 디렉터리다.

```bash
# 볼륨 없이 실행: 컨테이너 삭제 시 데이터도 삭제됨
docker run --name temp-db postgres

# 볼륨을 사용: 데이터가 호스트에 보존됨
docker run --name prod-db -v pgdata:/var/lib/postgresql/data postgres
# pgdata 볼륨은 /var/lib/docker/volumes/pgdata/_data/ 에 실제 저장
```

### 이미지 크기 최적화와 레이어 전략

```dockerfile
# 나쁜 예시 — 레이어마다 캐시가 무효화되고 크기도 커진다
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get clean

# 좋은 예시 — 관련 명령을 하나의 RUN으로 묶고 cleanup도 같은 레이어에서
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

```dockerfile
# 나쁜 예시 — COPY .를 먼저 하면 소스 변경마다 모든 이후 레이어 캐시 무효화
FROM maven:3.9-eclipse-temurin-21
COPY . /app
WORKDIR /app
RUN mvn dependency:go-offline
RUN mvn package -DskipTests

# 좋은 예시 — 의존성 레이어와 소스 레이어를 분리
FROM maven:3.9-eclipse-temurin-21
WORKDIR /app
COPY pom.xml .                    # 의존성 정의만 먼저 복사
RUN mvn dependency:go-offline     # 의존성 레이어 (소스 변경 시 캐시 유지)
COPY src ./src                    # 소스코드 복사
RUN mvn package -DskipTests       # 빌드
```

---

## 5. Docker 아키텍처 — daemon, containerd, runc

Docker는 처음에는 단일 데몬이 모든 것을 처리했다. 시간이 지나면서 관심사 분리와 표준화 요구에 따라 여러 컴포넌트로 분리됐다.

### 전체 컴포넌트 흐름

```
사용자
  │
  ▼
Docker CLI (docker)
  │  REST API over Unix socket (/var/run/docker.sock)
  ▼
Docker Daemon (dockerd)
  │  gRPC
  ▼
containerd
  │  실행 요청
  ▼
containerd-shim
  │
  ▼
runc (OCI 런타임)
  │  namespace + cgroup 설정 후
  ▼
컨테이너 프로세스
```

각 컴포넌트가 어떤 역할을 하는지 살펴본다.

### Docker CLI

사용자가 직접 상호작용하는 클라이언트다. `docker run`, `docker build`, `docker ps` 같은 명령을 받아 Docker Daemon의 REST API로 변환해서 전달한다.

```bash
# Docker CLI가 실제로 보내는 API 요청 확인
docker --log-level=debug run hello-world 2>&1 | head -50

# 또는 직접 API 호출
curl --unix-socket /var/run/docker.sock \
  http://localhost/v1.43/containers/json
```

### Docker Daemon (dockerd)

Docker의 메인 프로세스다. 이미지 관리, 컨테이너 생명주기 관리, 네트워크 설정, 볼륨 관리를 담당한다. 실제 컨테이너 실행은 containerd에 위임한다.

```bash
# dockerd 프로세스 확인
ps aux | grep dockerd
# root  1234  ...  /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

# dockerd 설정 파일
cat /etc/docker/daemon.json
```

```json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "default-runtime": "runc",
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

### containerd

컨테이너 런타임 관리자다. 원래 Docker의 일부였지만 2017년에 독립 프로젝트로 분리됐고, 현재 CNCF(Cloud Native Computing Foundation)가 관리한다.

containerd가 하는 일:
- 컨테이너 이미지 pull / push
- 이미지의 스냅샷 관리 (OverlayFS 레이어)
- 컨테이너 생성, 시작, 중지, 삭제
- 컨테이너 실행 상태 관리
- 저수준 런타임(runc)에 실제 실행 위임

```bash
# containerd 프로세스 확인
ps aux | grep containerd
# root  567  ...  /usr/bin/containerd

# containerd CLI(ctr)로 직접 컨테이너 관리 가능
sudo ctr images ls
sudo ctr containers ls
sudo ctr tasks ls
```

containerd가 Docker에서 독립한 이유는 쿠버네티스 때문이다. 쿠버네티스는 컨테이너를 실행하기 위해 CRI(Container Runtime Interface)를 정의했다. containerd는 CRI 플러그인을 통해 쿠버네티스와 직접 통신할 수 있어, Docker 없이도 쿠버네티스 노드에서 컨테이너를 실행할 수 있다.

### containerd-shim

containerd와 실제 컨테이너 프로세스 사이에 위치하는 얇은 레이어다. containerd가 재시작되거나 업그레이드되더라도 실행 중인 컨테이너가 종료되지 않도록 보호한다.

```bash
# shim 프로세스 확인
ps aux | grep containerd-shim
# root  8901  ...  /usr/bin/containerd-shim-runc-v2 -namespace moby -id abc123...
```

### runc

OCI(Open Container Initiative) 런타임 스펙을 구현한 저수준 컨테이너 런타임이다. 실제로 namespace와 cgroup을 설정하고 컨테이너 프로세스를 실행하는 주체다.

```bash
# runc 직접 사용 예시 (Docker 없이 컨테이너 실행)

# 1. OCI 번들 생성
mkdir mycontainer
cd mycontainer
mkdir rootfs

# Alpine rootfs 준비
docker export $(docker create alpine) | tar -C rootfs -xvf -

# OCI config.json 생성
runc spec

# 2. runc로 컨테이너 실행
sudo runc run mycontainer

# 3. 실행 중인 컨테이너 목록
sudo runc list
```

### OCI (Open Container Initiative) 표준

OCI는 2015년 Docker와 CoreOS가 주도해 설립한 오픈 표준 기구다. 두 가지 핵심 스펙을 정의한다.

- **OCI Runtime Spec**: 컨테이너를 어떻게 실행하는지 (runc가 구현)
- **OCI Image Spec**: 컨테이너 이미지 형식 (Docker 이미지와 호환)

OCI 표준 덕분에 Docker로 빌드한 이미지를 Podman으로 실행하거나, 쿠버네티스에서 사용할 수 있다.

```bash
# OCI config.json 구조 확인
cat config.json | python3 -m json.tool | head -80
```

```json
{
    "ociVersion": "1.0.1",
    "process": {
        "terminal": true,
        "user": {"uid": 0, "gid": 0},
        "args": ["/bin/sh"],
        "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
        "cwd": "/"
    },
    "root": {
        "path": "rootfs",
        "readonly": false
    },
    "linux": {
        "namespaces": [
            {"type": "pid"},
            {"type": "network"},
            {"type": "ipc"},
            {"type": "uts"},
            {"type": "mount"}
        ],
        "resources": {
            "memory": {"limit": 536870912},
            "cpu": {"quota": 50000, "period": 100000}
        }
    }
}
```

### 대안 런타임: Podman과 CRI-O

**Podman**

Red Hat이 개발한 daemonless 컨테이너 엔진이다. Docker와 달리 중앙 데몬이 없다. 각 명령이 직접 컨테이너를 실행한다.

```bash
# Podman은 Docker CLI와 호환됨
podman run --rm -it ubuntu bash
podman build -t myimage .
podman push myimage registry.example.com/myimage

# Docker alias 설정으로 대체 가능
alias docker=podman
```

Podman의 주요 장점:
- root 권한 없이 컨테이너 실행 (rootless by default)
- 데몬 프로세스가 없어 단일 장애점이 없음
- systemd와 자연스러운 통합

```bash
# Podman rootless 컨테이너
podman run --rm -it ubuntu bash
id  # uid=0(root) gid=0(root) — 컨테이너 안에서는 root
# 호스트에서는 일반 사용자 권한으로 실행됨
```

**CRI-O**

쿠버네티스 전용으로 설계된 경량 컨테이너 런타임이다. CRI 인터페이스만 구현하며, Docker 호환 CLI를 제공하지 않는다. OpenShift의 기본 런타임이다.

```
쿠버네티스(kubelet)
    │ CRI
    ▼
CRI-O
    │
    ▼
runc / Kata Containers
```

**컨테이너 런타임 비교**

| 항목 | Docker | Podman | CRI-O |
|------|--------|--------|-------|
| 데몬 | dockerd 필요 | 불필요 | 불필요 |
| 기본 권한 | root | rootless | root |
| 쿠버네티스 CRI | 미지원(shim 필요) | 부분 지원 | 완전 지원 |
| Docker CLI 호환 | 완전 | 완전 | 없음 |
| 주 사용처 | 개발 환경 | 개발/배포 | 쿠버네티스 노드 |

---

## 6. 실전: 컨테이너 내부를 직접 들여다보기

### 실행 중인 컨테이너의 namespace 확인

```bash
# nginx 컨테이너 실행
docker run -d --name mynginx nginx

# 컨테이너의 PID 조회
NGINX_PID=$(docker inspect --format '{{.State.Pid}}' mynginx)
echo "컨테이너 메인 프로세스 PID: ${NGINX_PID}"

# 해당 프로세스의 namespace 확인
ls -la /proc/${NGINX_PID}/ns/
```

```
총 0
dr-x--x--x 2 root root 0  3월 20 14:05 .
dr-xr-xr-x 9 root root 0  3월 20 14:05 ..
lrwxrwxrwx 1 root root 0  3월 20 14:05 cgroup -> 'cgroup:[4026532456]'
lrwxrwxrwx 1 root root 0  3월 20 14:05 ipc -> 'ipc:[4026532389]'
lrwxrwxrwx 1 root root 0  3월 20 14:05 mnt -> 'mnt:[4026532387]'
lrwxrwxrwx 1 root root 0  3월 20 14:05 net -> 'net:[4026532392]'
lrwxrwxrwx 1 root root 0  3월 20 14:05 pid -> 'pid:[4026532390]'
lrwxrwxrwx 1 root root 0  3월 20 14:05 uts -> 'uts:[4026532388]'
```

각 namespace가 고유한 inode 번호를 가진 것을 볼 수 있다. 두 프로세스가 같은 namespace를 공유하면 inode 번호가 동일하다.

```bash
# nsenter로 컨테이너 namespace에 진입
sudo nsenter -t ${NGINX_PID} --pid --net --mount bash

# 이제 컨테이너의 namespace 안에 있음
ps aux       # 컨테이너 프로세스만 보임
ip addr      # 컨테이너의 eth0 보임
ls /         # 컨테이너의 파일시스템
```

### 컨테이너 파일시스템 레이어 확인

```bash
# 이미지 레이어 확인
docker inspect nginx:latest | jq '.[0].GraphDriver'
```

```json
{
  "Data": {
    "LowerDir": "/var/lib/docker/overlay2/abc1.../diff:/var/lib/docker/overlay2/def2.../diff",
    "MergedDir": "/var/lib/docker/overlay2/xyz3.../merged",
    "UpperDir": "/var/lib/docker/overlay2/xyz3.../diff",
    "WorkDir": "/var/lib/docker/overlay2/xyz3.../work"
  },
  "Name": "overlay2"
}
```

```bash
# 컨테이너 실행 후 파일 수정 시 upper 레이어에만 변경 사항이 저장됨
docker exec mynginx sh -c "echo 'custom' > /etc/nginx/custom.conf"

# upper 디렉터리에서 변경 사항 확인
CONTAINER_ID=$(docker inspect --format '{{.Id}}' mynginx)
UPPER_DIR=$(docker inspect mynginx | jq -r '.[0].GraphDriver.Data.UpperDir')
ls ${UPPER_DIR}/etc/nginx/
# custom.conf  <- 컨테이너 레이어에만 존재
```

### 컨테이너 자원 사용량 실시간 모니터링

```bash
# 컨테이너 자원 사용량 실시간 확인
docker stats mynginx

# 더 많은 정보와 포맷 지정
docker stats --format \
  "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}" \
  mynginx
```

```
CONTAINER    CPU %    MEM USAGE / LIMIT     NET I/O          BLOCK I/O
mynginx      0.01%    6.234MiB / 256MiB     1.23kB / 648B    0B / 0B
```

---

## 정리: 컨테이너는 세 가지 기술의 조합이다

컨테이너를 처음 접했을 때 마법처럼 느껴지는 이유는, 그 안에 들어간 세 가지 기술이 각자 조용히 자신의 역할을 하기 때문이다.

**namespace**는 프로세스가 보는 세계를 좁힌다. PID, 네트워크, 파일시스템, 호스트명, IPC, 사용자 ID를 각 컨테이너 전용 공간으로 격리한다. 컨테이너 안의 프로세스는 자신이 시스템 전체를 독점하고 있다고 믿는다.

**cgroup**은 프로세스가 쓸 수 있는 자원의 상한을 정한다. CPU 코어 수, 메모리 바이트, 디스크 I/O 대역폭을 제한한다. 하나의 컨테이너가 호스트 자원 전체를 독식하지 못하도록 막는다.

**Union File System**은 이미지 레이어를 CoW 방식으로 공유한다. 동일한 베이스 이미지를 사용하는 수십 개의 컨테이너가 디스크의 같은 레이어를 공유하고, 변경 사항만 각자의 쓰기 레이어에 저장한다.

이 세 기술 위에 containerd와 runc가 OCI 표준을 구현하고, Docker Daemon이 API를 제공하며, Docker CLI가 사용자 명령을 받아들인다.

```
사용자 명령 (docker run)
    │
    ▼
Docker CLI
    │ REST API
    ▼
dockerd          ← 이미지, 볼륨, 네트워크 관리
    │ gRPC
    ▼
containerd       ← 컨테이너 생명주기, 스냅샷, 이미지
    │
    ▼
containerd-shim  ← 데몬 재시작 시에도 컨테이너 보호
    │
    ▼
runc             ← namespace + cgroup 설정, 프로세스 실행
    │
    ▼
컨테이너 프로세스 ← namespace로 격리, cgroup으로 제한
```

다음 편에서는 이 원리를 바탕으로 Dockerfile을 효율적으로 작성하는 법을 다룬다. 레이어 캐시를 최대한 활용하고, 이미지 크기를 줄이고, 보안을 높이는 실전 패턴을 살펴볼 것이다.

---

## 참고 자료

1. Nigel Poulton, *Docker Deep Dive* — 컨테이너 개념부터 실전까지 빠르게 익힐 수 있는 입문서. Docker 아키텍처와 명령어를 체계적으로 다룬다.

2. Linux man pages: `namespaces(7)`, `cgroups(7)` — 커널이 직접 제공하는 가장 정확한 레퍼런스. `man 7 namespaces`, `man 7 cgroups`로 로컬에서 읽을 수 있다.

3. OCI Runtime Specification — [github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec). runc가 구현하는 컨테이너 런타임 표준 스펙 전문. config.json 구조와 컨테이너 생명주기가 정의되어 있다.

4. Michael Kerrisk, *Namespaces in Operation* — LWN.net에 연재된 7부작 시리즈. 리눅스 namespace를 가장 깊이 있게 다루는 문서 중 하나. [lwn.net/Articles/531114](https://lwn.net/Articles/531114/)

5. Docker 공식 문서, *Docker Architecture* — [docs.docker.com/get-started/docker-overview](https://docs.docker.com/get-started/docker-overview/). dockerd, containerd, runc의 관계와 전체 아키텍처를 공식적으로 설명한다.
