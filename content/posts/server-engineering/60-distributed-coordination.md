---
title: "[분산 시스템과 MSA] 2편 — 분산 코디네이션과 리더 선출: 조율의 기술"
date: 2026-03-17T17:06:00+09:00
draft: false
tags: ["리더 선출", "ZooKeeper", "etcd", "Fencing Token", "서버"]
series: ["분산 시스템과 MSA"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 9
summary: "리더 선출 알고리즘(Bully, Ring, ZooKeeper 기반), 분산 락 이론(Fencing Token, 정확성 증명), ZooKeeper vs etcd 비교, 분산 세마포어·카운터·배리어까지"
---

분산 시스템에서 가장 어려운 문제 중 하나는 "지금 누가 책임자인가?"라는 질문에 모든 노드가 동의하도록 만드는 것이다.

단일 서버 환경에서는 OS 스케줄러가 알아서 처리해주지만, 여러 노드가 네트워크로 연결된 순간 모든 것이 달라진다. 메시지는 지연되고, 노드는 죽고, 일부 노드는 자신이 살아있다고 생각하지만 실제로는 네트워크 파티션 너머에 고립되어 있다.

이 글에서는 분산 코디네이션의 핵심인 리더 선출 알고리즘부터 분산 락의 이론적 정확성 증명, 그리고 ZooKeeper와 etcd의 설계 철학까지 깊이 파고든다.

---

## 1. 리더 선출: 누가 책임자인가

### 왜 리더 선출이 필요한가

분산 시스템에서 리더는 단순히 "우두머리" 역할이 아니다.

리더는 쓰기 요청을 직렬화(serialize)하고, 클러스터 상태를 조율하며, 분산 트랜잭션의 코디네이터 역할을 수행한다. 리더가 없거나, 두 노드가 동시에 자신이 리더라고 믿는 **스플릿 브레인(split brain)** 상태가 발생하면 데이터 일관성이 무너진다.

### Bully 알고리즘

Bully 알고리즘은 각 노드에 고유한 ID(정수)를 부여하고, 가장 높은 ID를 가진 노드가 리더가 되는 방식이다.

```
노드 구성: N1(ID=1), N2(ID=2), N3(ID=3), N4(ID=4), N5(ID=5)
현재 리더: N5 (장애 발생)

1. N3이 N5의 장애를 감지
2. N3은 자신보다 높은 ID를 가진 노드(N4, N5)에게 ELECTION 메시지 전송
3. N4가 응답(OK 메시지) → N3의 선출 시도 중단
4. N4는 N5에게 ELECTION 메시지 전송
5. N5 응답 없음 → N4가 COORDINATOR 메시지를 모든 노드에 브로드캐스트
6. N4가 새로운 리더로 확정
```

```python
import threading
import time
from enum import Enum

class MessageType(Enum):
    ELECTION = "ELECTION"
    OK = "OK"
    COORDINATOR = "COORDINATOR"

class BullyNode:
    def __init__(self, node_id: int, all_nodes: list):
        self.node_id = node_id
        self.all_nodes = sorted(all_nodes)
        self.leader_id = max(all_nodes)
        self.is_alive = True
        self.election_in_progress = False
        self.received_ok = False

    def start_election(self):
        print(f"[Node {self.node_id}] 선출 시작")
        self.election_in_progress = True
        self.received_ok = False

        higher_nodes = [n for n in self.all_nodes if n > self.node_id]

        if not higher_nodes:
            # 자신이 가장 높은 ID → 즉시 리더 선언
            self.declare_coordinator()
            return

        for node_id in higher_nodes:
            self._send_election(node_id)

        # 타임아웃 후 OK 수신 여부 확인
        time.sleep(0.5)  # T 타임아웃
        if not self.received_ok:
            self.declare_coordinator()

    def on_receive_election(self, from_node_id: int):
        # 더 높은 ID를 가진 나는 선출에 개입
        self._send_ok(from_node_id)
        if not self.election_in_progress:
            self.start_election()

    def on_receive_ok(self, from_node_id: int):
        self.received_ok = True
        print(f"[Node {self.node_id}] OK 수신 from Node {from_node_id}, 대기")

    def declare_coordinator(self):
        self.leader_id = self.node_id
        self.election_in_progress = False
        print(f"[Node {self.node_id}] 새 리더 선언: Node {self.node_id}")
        for node_id in self.all_nodes:
            if node_id != self.node_id:
                self._send_coordinator(node_id)

    def _send_election(self, target_id):
        print(f"[Node {self.node_id}] → ELECTION → Node {target_id}")

    def _send_ok(self, target_id):
        print(f"[Node {self.node_id}] → OK → Node {target_id}")

    def _send_coordinator(self, target_id):
        print(f"[Node {self.node_id}] → COORDINATOR → Node {target_id}")
```

Bully 알고리즘의 문제점은 메시지 복잡도가 O(n²)이라는 것이다.

노드 수가 증가할수록 선출 과정에서 발생하는 메시지 수가 기하급수적으로 늘어나 대규모 클러스터에는 적합하지 않다.

### Ring 알고리즘

Ring 알고리즘은 노드들을 논리적인 링 구조로 배치하고, 각 노드는 자신의 다음 노드에게만 메시지를 전달하는 방식이다.

```
링 구성: N1 → N2 → N3 → N4 → N5 → N1
현재 리더: N3 (장애 발생)

1. N4가 N3의 장애를 감지
2. N4가 ELECTION 메시지에 자신의 ID(4)를 담아 N5로 전달
3. N5는 자신의 ID(5)를 메시지에 추가하여 N1으로 전달
4. N1은 자신의 ID(1)를 추가하여 N2로 전달
5. N2는 자신의 ID(2)를 추가하여 N4로 전달
6. N4는 메시지가 자신에서 시작했음을 확인 → 메시지 내 최대 ID인 5가 새 리더
7. N4가 COORDINATOR(leader=5) 메시지를 링 전체에 전파
```

Ring 알고리즘의 장점은 메시지 복잡도가 O(n)으로 줄어든다는 점이다.

하지만 링 구조에서 여러 노드가 동시에 선출을 시작하면 여러 ELECTION 메시지가 링을 순환하는 문제가 발생하며, 링 자체가 끊기는 경우의 처리가 복잡해진다.

### ZooKeeper 기반 리더 선출

실무에서는 직접 선출 알고리즘을 구현하는 것보다 ZooKeeper나 etcd의 기본 기능을 활용하는 것이 훨씬 안전하다.

ZooKeeper의 **에페머럴 시퀀셜 노드(Ephemeral Sequential Node)**를 이용한 리더 선출은 다음과 같이 동작한다.

```python
from kazoo.client import KazooClient
from kazoo.recipe.election import Election

class ZooKeeperLeaderElection:
    def __init__(self, hosts: str, election_path: str, identifier: str):
        self.zk = KazooClient(hosts=hosts)
        self.zk.start()
        self.election = Election(self.zk, election_path, identifier)
        self.is_leader = False

    def run(self):
        """리더 선출에 참여하고 리더가 되면 작업 수행"""
        print(f"[{self.election.identifier}] 리더 선출 참여 중...")
        self.election.run(self._leader_task)

    def _leader_task(self):
        """리더로 선출된 경우 실행되는 콜백"""
        self.is_leader = True
        print(f"[{self.election.identifier}] 리더로 선출됨! 작업 시작...")
        try:
            self._do_leader_work()
        finally:
            self.is_leader = False
            print(f"[{self.election.identifier}] 리더 작업 종료")

    def _do_leader_work(self):
        # 실제 리더가 수행할 작업
        while self.is_leader:
            time.sleep(1)
            print(f"[{self.election.identifier}] 리더 작업 수행 중...")

    def stop(self):
        self.zk.stop()
```

ZooKeeper 기반 선출의 핵심 메커니즘은 각 노드가 `/election/` 경로 아래에 에페머럴 시퀀셜 노드를 생성하고, 가장 낮은 시퀀스 번호를 가진 노드가 리더가 된다는 것이다.

노드가 장애로 연결이 끊기면 에페머럴 노드가 자동으로 삭제되고, 다음 번호를 가진 노드에게 와처(watcher) 이벤트가 발생하여 자신이 리더임을 알게 된다. 에페머럴 노드는 ZooKeeper와의 세션이 살아있는 동안만 존재하는 임시 노드다. 마치 "출석부에 서명한 동안만 자리를 유지하는 구조"라고 볼 수 있다. 세션이 끊기면 서명이 자동으로 지워지므로 별도의 장애 감지 로직 없이 선출이 자동으로 트리거된다.

---

## 2. 분산 락: 이론과 정확성 증명

### 분산 락의 세 가지 요건

올바른 분산 락은 다음 세 가지 속성을 반드시 만족해야 한다.

**상호 배제(Mutual Exclusion):** 어떠한 순간에도 단 하나의 클라이언트만 락을 보유할 수 있다.

**데드락 프리(Deadlock Free):** 락을 보유한 클라이언트가 충돌하거나 파티션되어도 결국 락은 해제된다. 이는 보통 TTL(Time-To-Live) 기반의 자동 만료로 구현한다.

**결함 내성(Fault Tolerance):** 락 서비스의 일부 노드가 장애를 겪어도 락은 올바르게 동작해야 한다.

### Fencing Token: 왜 필요한가

TTL 기반의 분산 락에는 근본적인 취약점이 존재한다.

```
시나리오 (스플릿 브레인 위험):

1. 클라이언트 A가 락 획득 (TTL: 30초)
2. 클라이언트 A가 GC 일시 정지로 30초 이상 멈춤
3. 락이 TTL 만료로 자동 해제
4. 클라이언트 B가 동일한 락 획득
5. 클라이언트 A의 GC 완료 → 자신이 아직 락을 보유했다고 착각
6. 클라이언트 A와 B 모두 락을 보유했다고 믿고 공유 자원 수정
→ 데이터 손상!
```

Fencing Token은 이 문제를 해결하는 우아한 방법이다.

락을 획득할 때마다 단조 증가(monotonically increasing)하는 토큰 번호를 부여하고, 락으로 보호되는 자원은 이 토큰 번호를 검증하여 오래된 토큰을 가진 요청을 거부한다.

```python
import threading
from dataclasses import dataclass
from typing import Optional

@dataclass
class FencingToken:
    token: int
    holder: str
    expires_at: float

class FencedResource:
    """Fencing Token을 강제하는 공유 자원"""
    def __init__(self):
        self._data = {}
        self._last_seen_token = 0
        self._lock = threading.Lock()

    def write(self, key: str, value: str, fencing_token: int) -> bool:
        with self._lock:
            if fencing_token <= self._last_seen_token:
                # 오래된 토큰 → 거부 (스플릿 브레인 방지)
                print(f"거부: 토큰 {fencing_token} ≤ 마지막 토큰 {self._last_seen_token}")
                return False
            self._last_seen_token = fencing_token
            self._data[key] = value
            print(f"쓰기 성공: key={key}, token={fencing_token}")
            return True

class DistributedLockWithFencing:
    def __init__(self, lock_service):
        self.lock_service = lock_service
        self._token_counter = 0
        self._token_counter_lock = threading.Lock()

    def acquire(self, client_id: str, ttl: float) -> Optional[FencingToken]:
        with self._token_counter_lock:
            self._token_counter += 1
            token = FencingToken(
                token=self._token_counter,
                holder=client_id,
                expires_at=time.time() + ttl
            )
        # 락 서비스에 등록 (실제로는 ZooKeeper/etcd에 저장)
        success = self.lock_service.set_lock(token)
        return token if success else None

    def release(self, token: FencingToken, client_id: str) -> bool:
        # 본인이 획득한 락만 해제 가능
        if token.holder != client_id:
            return False
        return self.lock_service.release_lock(token)
```

### 정확성 증명: 선형화 가능성(Linearizability)

분산 락이 올바르다는 것을 어떻게 증명할 수 있을까?

**선형화 가능성**은 분산 시스템에서 가장 강력한 일관성 모델이다. 각 연산이 호출(invocation)과 완료(completion) 사이의 어느 한 시점에 원자적으로 실행된 것처럼 보여야 한다.

```
올바른 실행 예시:
시간: ─────────────────────────────────────────→

Client A: [acquire(lock)]────────[release(lock)]
Client B:                  [acquire(lock)]────────[release(lock)]

A의 lock 보유 구간과 B의 lock 보유 구간이 절대 겹치지 않음 ✓

잘못된 실행 예시 (상호 배제 위반):
Client A: [acquire(lock)]─────────────────────────────────
Client B:                  [acquire(lock)]────────
                                   ↑
                          두 클라이언트가 동시에 락 보유 → 위반 ✗
```

Fencing Token 방식의 정확성을 귀납법으로 증명할 수 있다.

**기저 사례:** 첫 번째 락 획득 시 토큰 T=1이 발급된다. 자원의 `_last_seen_token=0 < 1`이므로 쓰기가 허용된다.

**귀납 단계:** 현재 토큰 T=k가 올바르게 동작한다고 가정하자. 다음 락 획득 시 T=k+1이 발급된다. T=k를 보유했던 스테일 클라이언트가 요청을 보내도 `k < k+1`이므로 자원이 이를 거부한다. 따라서 T=k+1을 보유한 클라이언트만 쓰기에 성공한다.

### 안티패턴: 락 해제 시 주의사항

```python
# 안티패턴: 락 해제 시 타인의 락을 해제할 수 있음
def bad_release(key: str, redis_client):
    redis_client.delete(key)  # 위험! 내 락인지 확인 안 함

# 올바른 방법: Lua 스크립트로 원자적 확인 + 삭제
RELEASE_SCRIPT = """
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
"""

def correct_release(key: str, token: str, redis_client):
    result = redis_client.eval(RELEASE_SCRIPT, 1, key, token)
    return result == 1
```

락 해제는 반드시 "내가 보유한 락인지 확인"과 "삭제"가 원자적으로 수행되어야 한다.

이 두 연산 사이에 다른 클라이언트가 락을 획득할 수 있기 때문에, Redis에서는 Lua 스크립트를 사용하여 원자성을 보장한다.

---

## 3. ZooKeeper vs etcd: 두 코디네이터의 설계 철학

### ZooKeeper의 아키텍처

ZooKeeper는 Yahoo에서 개발된 분산 코디네이션 서비스로, **ZAB(ZooKeeper Atomic Broadcast)** 프로토콜을 사용한다.

ZAB는 Paxos와 유사하지만 리더 기반의 스트리밍 복제에 최적화되어 있다. 모든 쓰기는 리더를 통해 처리되고, 팔로워는 리더의 트랜잭션 로그를 순서대로 재실행하여 상태를 동기화한다.

```
ZooKeeper 클러스터 구성:

    Client         Client         Client
       │               │               │
       ▼               ▼               ▼
  ┌─────────┐    ┌─────────┐    ┌─────────┐
  │Follower │    │ Leader  │    │Follower │
  │  ZK-1   │◄──│  ZK-2   │──►│  ZK-3   │
  └─────────┘    └─────────┘    └─────────┘
       │               │               │
       └───────────────┴───────────────┘
                   ZAB Protocol

- 읽기: 모든 노드에서 처리 가능 (단, 최신 데이터 보장 안 됨)
- 쓰기: 리더가 수신 → 과반수 팔로워에 복제 → 커밋 → 클라이언트 응답
```

ZooKeeper의 핵심 데이터 모델은 **znode**다.

znode는 파일 시스템의 디렉터리와 유사한 트리 구조를 이루며, 퍼시스턴트(persistent), 에페머럴(ephemeral), 시퀀셜(sequential) 세 가지 타입을 조합할 수 있다.

```python
from kazoo.client import KazooClient
from kazoo.exceptions import NodeExistsError, NoNodeError

class ZooKeeperCoordinator:
    def __init__(self, hosts: str):
        self.zk = KazooClient(hosts=hosts)
        self.zk.start()

    def create_ephemeral_node(self, path: str, data: bytes) -> str:
        """에페머럴 노드: 클라이언트 세션이 끊기면 자동 삭제"""
        return self.zk.create(path, data, ephemeral=True)

    def create_sequential_node(self, path: str, data: bytes) -> str:
        """시퀀셜 노드: 경로에 자동 증가 번호 부여"""
        return self.zk.create(path, data, sequence=True)

    def watch_node(self, path: str, callback):
        """와처 등록: 노드 변경 시 1회 호출"""
        @self.zk.DataWatch(path)
        def watcher(data, stat, event):
            if event:
                callback(event)

    def distributed_lock_acquire(self, lock_path: str) -> str:
        """ZooKeeper 기반 분산 락 구현"""
        # 에페머럴 시퀀셜 노드 생성
        node = self.zk.create(
            f"{lock_path}/lock-",
            b"",
            ephemeral=True,
            sequence=True
        )
        node_name = node.split("/")[-1]

        while True:
            # 모든 락 경쟁자 조회
            children = sorted(self.zk.get_children(lock_path))
            my_index = children.index(node_name)

            if my_index == 0:
                # 내가 첫 번째 → 락 획득
                return node
            else:
                # 바로 앞 노드를 감시 (불필요한 허드 효과 방지)
                predecessor = children[my_index - 1]
                predecessor_path = f"{lock_path}/{predecessor}"
                event = threading.Event()

                @self.zk.DataWatch(predecessor_path)
                def watch(data, stat, ev):
                    if ev and ev.type == "DELETED":
                        event.set()
                        return False  # 와처 해제

                event.wait()  # 앞 노드 삭제 대기
```

### etcd의 아키텍처

etcd는 CoreOS(현 Red Hat)에서 Kubernetes의 상태 저장소로 개발한 분산 키-값 저장소다.

**Raft** 합의 알고리즘을 사용하며, 이해하기 쉬운 알고리즘 설계로 유명하다. etcd는 HTTP/gRPC 기반의 RESTful API를 제공하며, 와처 스트리밍을 통해 변경 사항을 실시간으로 구독할 수 있다.

```python
import etcd3
import contextlib

class EtcdCoordinator:
    def __init__(self, host: str = 'localhost', port: int = 2379):
        self.client = etcd3.client(host=host, port=port)

    @contextlib.contextmanager
    def distributed_lock(self, lock_name: str, ttl: int = 30):
        """etcd 기반 분산 락 (컨텍스트 매니저)"""
        lock = self.client.lock(lock_name, ttl=ttl)
        lock.acquire()
        try:
            yield lock
        finally:
            lock.release()

    def watch_prefix(self, prefix: str, callback):
        """키 접두사 변경 감시"""
        events_iterator, cancel = self.client.watch_prefix(prefix)
        for event in events_iterator:
            callback(event)

    def compare_and_swap(self, key: str, new_value: bytes,
                         expected_value: bytes) -> bool:
        """원자적 CAS 연산"""
        status, responses = self.client.transaction(
            compare=[
                self.client.transactions.value(key) == expected_value,
            ],
            success=[
                self.client.transactions.put(key, new_value),
            ],
            failure=[],
        )
        return status

    def leader_election(self, election_name: str, value: bytes):
        """etcd v3 리더 선출 API"""
        election = etcd3.Election(self.client, election_name)
        # 리더가 될 때까지 블로킹
        election.campaign(value)
        print(f"리더로 선출됨: {value.decode()}")
        return election
```

### ZooKeeper vs etcd 비교

| 항목 | ZooKeeper | etcd |
|------|-----------|------|
| 합의 알고리즘 | ZAB | Raft |
| API | 전용 클라이언트 | gRPC / HTTP |
| 데이터 모델 | 계층적 znode | 플랫 키-값 |
| 와처 모델 | 1회성 트리거 | 지속적 스트림 |
| 선호 사용처 | Kafka, HBase | Kubernetes, 클라우드 네이티브 |
| 성능 (쓰기) | ~10,000 ops/s | ~10,000~30,000 ops/s |
| 운영 복잡도 | 높음 (JVM, 복잡한 설정) | 낮음 (단일 바이너리) |

ZooKeeper의 1회성 와처는 이벤트 누락의 위험이 있다.

이벤트를 받고 다음 와처를 등록하기 전에 변경이 발생하면 해당 이벤트를 놓칠 수 있다. etcd의 스트리밍 와처는 리비전(revision) 번호를 기반으로 이 문제를 해결한다.

### 안티패턴: 헤르드 효과(Herd Effect)

```python
# 안티패턴: 모든 락 경쟁자가 리더 노드만 감시
def bad_lock_watch(zk, lock_path, all_children):
    first_child = sorted(all_children)[0]
    # 모든 노드가 첫 번째 노드만 감시
    # → 첫 번째 노드 삭제 시 N-1개 와처 동시 발화 → 서버 과부하
    zk.exists(f"{lock_path}/{first_child}", watch=True)

# 올바른 방법: 자신의 바로 앞 노드만 감시
def good_lock_watch(zk, lock_path, my_node, all_children):
    sorted_children = sorted(all_children)
    my_index = sorted_children.index(my_node)
    if my_index > 0:
        predecessor = sorted_children[my_index - 1]
        # 바로 앞 노드만 감시 → 노드 삭제 시 1개 와처만 발화
        zk.exists(f"{lock_path}/{predecessor}", watch=True)
```

헤르드 효과는 락 경쟁자 수가 많을수록 코디네이션 서버에 스파이크 부하를 일으킨다.

ZooKeeper 기반 락에서 각 노드가 자신의 바로 앞 순번 노드만 감시하도록 설계하면 이 문제를 O(n)에서 O(1)로 줄일 수 있다.

---

## 4. 분산 프리미티브: 세마포어, 카운터, 배리어

### 분산 세마포어

단순한 뮤텍스(1개만 접근 허용)를 넘어, 동시에 N개의 클라이언트가 자원에 접근할 수 있도록 허용하는 것이 분산 세마포어다.

```python
class DistributedSemaphore:
    """ZooKeeper 기반 분산 세마포어"""
    def __init__(self, zk: KazooClient, path: str, max_leases: int):
        self.zk = zk
        self.path = path
        self.max_leases = max_leases
        self.zk.ensure_path(path)

    def acquire(self, timeout: float = None) -> Optional[str]:
        """세마포어 획득 시도"""
        # 임시 노드 생성
        lease_path = self.zk.create(
            f"{self.path}/lease-",
            b"",
            ephemeral=True,
            sequence=True
        )
        lease_name = lease_path.split("/")[-1]

        deadline = time.time() + timeout if timeout else None

        while True:
            leases = sorted(self.zk.get_children(self.path))
            my_index = leases.index(lease_name)

            if my_index < self.max_leases:
                # 세마포어 획득 성공 (상위 N개 이내)
                print(f"세마포어 획득: {lease_name} (위치: {my_index+1}/{self.max_leases})")
                return lease_path
            else:
                # N번째 바로 앞 노드 감시
                predecessor = leases[my_index - self.max_leases]
                pred_path = f"{self.path}/{predecessor}"
                event = threading.Event()

                @self.zk.DataWatch(pred_path)
                def watch(data, stat, ev):
                    if ev and ev.type == "DELETED":
                        event.set()
                        return False

                if deadline:
                    remaining = deadline - time.time()
                    if remaining <= 0 or not event.wait(remaining):
                        self.zk.delete(lease_path)
                        return None
                else:
                    event.wait()

    def release(self, lease_path: str):
        """세마포어 해제"""
        try:
            self.zk.delete(lease_path)
            print(f"세마포어 해제: {lease_path}")
        except NoNodeError:
            pass  # 이미 해제됨
```

### 분산 카운터

분산 환경에서 원자적 카운터를 구현하는 것은 단순해 보이지만 까다롭다.

여러 노드가 동시에 값을 읽고 쓰면 갱신 손실(lost update) 문제가 발생한다. etcd의 트랜잭션 API를 이용하면 원자적 CAS(Compare-And-Swap)로 이를 해결할 수 있다.

```python
import struct

class DistributedCounter:
    """etcd 기반 분산 카운터"""
    def __init__(self, client: etcd3.Etcd3Client, key: str):
        self.client = client
        self.key = key
        # 초기값 설정 (없으면 0)
        self.client.put(key, struct.pack('>q', 0))

    def increment(self, delta: int = 1) -> int:
        """원자적 증가 (CAS 재시도 루프)"""
        while True:
            value, metadata = self.client.get(self.key)
            if value is None:
                current = 0
                version = 0
            else:
                current = struct.unpack('>q', value)[0]
                version = metadata.version

            new_value = current + delta
            packed_new = struct.pack('>q', new_value)

            # 버전 기반 CAS: 내가 읽은 이후 변경이 없었으면 성공
            success, _ = self.client.transaction(
                compare=[
                    self.client.transactions.version(self.key) == version,
                ],
                success=[
                    self.client.transactions.put(self.key, packed_new),
                ],
                failure=[],
            )

            if success:
                return new_value
            # 실패 시 재시도 (다른 노드가 먼저 변경함)

    def get(self) -> int:
        value, _ = self.client.get(self.key)
        if value is None:
            return 0
        return struct.unpack('>q', value)[0]

    def reset(self) -> bool:
        return self.client.put(self.key, struct.pack('>q', 0))
```

### 분산 배리어

분산 배리어는 여러 노드가 특정 지점에 모두 도달할 때까지 기다리게 하는 동기화 프리미티브다.

MapReduce나 분산 배치 처리에서 모든 워커가 한 단계를 완료한 후 다음 단계로 진입하도록 조율할 때 사용된다.

```python
class DistributedBarrier:
    """ZooKeeper 기반 분산 배리어"""
    def __init__(self, zk: KazooClient, path: str, num_participants: int):
        self.zk = zk
        self.path = path
        self.num_participants = num_participants
        self.ready_path = f"{path}/ready"
        self.zk.ensure_path(path)

    def enter(self, participant_id: str):
        """배리어 진입: 참가자가 준비 완료를 알림"""
        node_path = f"{self.path}/participant-{participant_id}"
        self.zk.create(node_path, b"", ephemeral=True)
        print(f"[{participant_id}] 배리어 진입, 대기 중...")

        # 모든 참가자가 도착할 때까지 대기
        while True:
            participants = self.zk.get_children(self.path)
            # 'ready' 노드 제외한 실제 참가자 수 확인
            actual_participants = [p for p in participants if p != "ready"]

            if len(actual_participants) >= self.num_participants:
                # 마지막 참가자가 ready 노드 생성
                try:
                    self.zk.create(self.ready_path, b"", ephemeral=True)
                except NodeExistsError:
                    pass
                print(f"[{participant_id}] 모든 참가자 도착! 배리어 통과")
                return
            else:
                # ready 노드가 생길 때까지 대기
                event = threading.Event()

                @self.zk.ChildrenWatch(self.path)
                def watch(children):
                    if "ready" in children or len([c for c in children if c != "ready"]) >= self.num_participants:
                        event.set()
                        return False

                event.wait(timeout=5.0)

    def leave(self, participant_id: str):
        """배리어 이탈"""
        node_path = f"{self.path}/participant-{participant_id}"
        try:
            self.zk.delete(node_path)
        except NoNodeError:
            pass

# 사용 예시
def worker_task(worker_id: str, barrier: DistributedBarrier):
    print(f"[Worker {worker_id}] 1단계 작업 시작...")
    time.sleep(random.uniform(0.5, 2.0))  # 작업 수행
    print(f"[Worker {worker_id}] 1단계 완료, 동기화 대기...")

    barrier.enter(worker_id)  # 모든 워커가 여기 도달할 때까지 블로킹

    print(f"[Worker {worker_id}] 2단계 작업 시작...")
    barrier.leave(worker_id)
```

### 분산 큐: 코디네이션의 응용

코디네이션 서비스를 이용한 분산 큐는 간단한 작업 분배에 유용하다.

다만 ZooKeeper는 대용량 메시지 처리에 설계된 것이 아니므로, 큐 아이템 수가 수만 개를 넘어가면 Kafka나 RabbitMQ처럼 전용 메시지 브로커를 사용하는 것이 바람직하다.

```python
class ZooKeeperQueue:
    """ZooKeeper 기반 분산 FIFO 큐 (소규모 작업 조율용)"""
    def __init__(self, zk: KazooClient, path: str):
        self.zk = zk
        self.path = path
        self.zk.ensure_path(path)

    def enqueue(self, data: bytes) -> str:
        """큐에 아이템 추가"""
        node = self.zk.create(
            f"{self.path}/item-",
            data,
            sequence=True  # 순서 보장
        )
        return node

    def dequeue(self) -> Optional[bytes]:
        """큐에서 아이템 꺼내기 (FIFO)"""
        while True:
            children = sorted(self.zk.get_children(self.path))
            if not children:
                return None

            first = children[0]
            first_path = f"{self.path}/{first}"

            try:
                data, _ = self.zk.get(first_path)
                self.zk.delete(first_path)
                return data
            except NoNodeError:
                # 다른 컨슈머가 먼저 가져감 → 재시도
                continue
```

---

## 마치며

분산 코디네이션은 단순히 "락을 걸면 되는 것 아닌가?"라는 생각으로 접근하면 반드시 문제가 터진다.

GC 일시 정지, 네트워크 파티션, 클락 드리프트, 스플릿 브레인 — 분산 시스템의 적들은 예상치 못한 순간에 나타난다. Fencing Token과 같은 설계 패턴이 왜 존재하는지, ZooKeeper가 왜 1회성 와처를 선택했는지 이해하면 이런 함정을 미리 피할 수 있다.

코디네이션 서비스는 데이터베이스가 아니다. ZooKeeper와 etcd는 메타데이터와 소량의 조율 데이터를 위한 것이지, 비즈니스 데이터를 저장하는 용도가 아니다. 이 경계를 명확히 유지하는 것이 분산 시스템을 건강하게 운영하는 첫걸음이다.

---

## 참고 자료

- Martin Kleppmann, *Designing Data-Intensive Applications*, O'Reilly, 2017 — 7장 트랜잭션, 8장 분산 시스템의 고난, 9장 일관성과 합의
- Leslie Lamport, "The Part-Time Parliament" (1998) — Paxos 합의 알고리즘 원문
- Diego Ongaro & John Ousterhout, "In Search of an Understandable Consensus Algorithm (Raft)" (2014) — USENIX ATC
- Apache ZooKeeper Documentation, *ZooKeeper Recipes and Solutions* — https://zookeeper.apache.org/doc/current/recipes.html
- Martin Kleppmann, "How to do distributed locking" (2016) — https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
- etcd Documentation, *etcd API reference* — https://etcd.io/docs/v3.5/learning/api/
