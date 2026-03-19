---
title: "[분산 시스템과 MSA] 1편 — 분산 시스템의 기초: 네트워크는 믿을 수 없다"
date: 2026-03-17T17:07:00+09:00
draft: false
tags: ["분산 시스템", "CAP", "Raft", "Lamport Clock", "서버"]
series: ["분산 시스템과 MSA"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 9
summary: "분산 시스템 8가지 오류 가정, 시간 동기화 문제와 클럭 스큐·논리 시계(Lamport Clock), 합의 알고리즘 개요(Raft, Paxos), Chaos Engineering 개념까지"
---

분산 시스템의 세계에 발을 들이는 순간, 가장 먼저 배우는 교훈은 하나다. "네트워크는 믿을 수 없다." 단일 서버에서는 함수 호출이 항상 성공하고, 시간은 단조롭게 흐르며, 데이터는 메모리에 정직하게 존재한다. 하지만 여러 노드에 걸쳐 시스템을 구성하는 순간, 모든 전제가 흔들린다. 패킷은 사라지고, 노드는 멈추며, 같은 시각에 서로 다른 노드는 서로 다른 상태를 본다. 이 글은 그 불확실성과 정면으로 마주하는 방법을 다룬다.

---

## 분산 시스템의 8가지 오류 가정 (Fallacies of Distributed Computing)

1994년 Peter Deutsch와 Sun Microsystems 팀이 정리한 "분산 컴퓨팅의 8가지 오류 가정"은 오늘날에도 유효한 경고다.

단일 서버 개발에 익숙한 엔지니어가 분산 시스템으로 넘어올 때 반드시 깨야 하는 믿음들이다.

### 오류 1: 네트워크는 믿을 수 있다

TCP가 연결을 보장한다고 해서 데이터 전달도 보장되지는 않는다.

로드밸런서가 연결을 끊거나, 방화벽이 패킷을 조용히 삭제하거나, 중간 라우터가 재시작될 수 있다.

```text
Client ---[SYN]---> Server
Client <--[SYN-ACK]-- Server
Client ---[ACK]---> Server    ← 여기까지 성공
Client ---[DATA]--> (라우터 재시작, 패킷 소멸)
Client <-- 응답 없음 (무한 대기)
```

실제로 AWS나 GCP의 내부 네트워크에서도 패킷 손실은 발생한다.

타임아웃과 재시도 로직은 선택이 아니라 필수다.

### 오류 2: 지연 시간은 0이다

같은 데이터센터 내 서버 간 RTT(Round Trip Time)는 0.1~1ms 수준이다.

하지만 다른 리전 간에는 100ms를 쉽게 넘긴다.

마이크로서비스 아키텍처에서 하나의 HTTP 요청이 5개 서비스를 연쇄 호출한다면, 100ms × 5 = 500ms가 된다.

이것이 서비스 메시와 비동기 통신이 필요한 이유다.

### 오류 3: 대역폭은 무한하다

10Gbps 네트워크 카드가 있어도, 수백 개의 서비스가 동시에 대용량 데이터를 주고받으면 병목이 생긴다.

gRPC 대신 HTTP/JSON을 쓰는 팀이 페이로드 크기를 무시하다가 네트워크 비용이 폭증하는 일은 흔하다.

### 오류 4: 네트워크는 안전하다

내부 네트워크라도 TLS 없이 통신하는 것은 위험하다.

Zero Trust 아키텍처가 등장한 이유가 바로 이 오류 가정 때문이다.

### 오류 5: 토폴로지는 변하지 않는다

쿠버네티스 환경에서 Pod의 IP 주소는 매 배포마다 바뀐다.

하드코딩된 IP 주소나 정적 설정은 분산 환경에서 즉각적인 장애 원인이 된다.

서비스 디스커버리(Consul, Kubernetes DNS)가 필수인 이유다.

### 오류 6: 관리자는 한 명이다

마이크로서비스 환경에서는 팀마다 서비스를 독립적으로 배포한다.

인프라 정책, 보안 설정, 타임아웃 기준이 팀마다 다를 수 있다.

이것이 운영 복잡도를 기하급수적으로 높이는 이유다.

### 오류 7: 전송 비용은 0이다

서비스 간 API 호출 한 번에는 DNS 조회, TCP 핸드셰이크, TLS 협상, HTTP 헤더 직렬화가 따라온다.

"그냥 API 하나 더 호출하면 되지"라는 사고가 레이턴시 폭증의 원인이 된다.

### 오류 8: 네트워크는 동질적이다

클라우드 환경에서는 인스턴스 타입, 가용 영역, 리전, CDN 엣지까지 다양한 네트워크 계층이 공존한다.

같은 리전 내에서도 가용 영역 간 레이턴시는 다르다.

```text
[안티패턴] 동질적 네트워크 가정

서비스 A ──────────────────→ 서비스 B
           "항상 1ms 이하"

현실:
서비스 A (us-east-1a)  →  서비스 B (us-east-1c)
            = 0.5~3ms 변동

서비스 A (us-east-1)   →  서비스 B (eu-west-1)
            = 80~120ms 변동
```

---

## 시간 동기화 문제: 분산 시스템에서 시간은 거짓말한다

### 물리 시계의 한계

모든 서버는 자체 하드웨어 클럭을 가진다.

하지만 이 클럭은 완벽하지 않다. 온도, 전압, 수정 진동자의 물리적 특성에 따라 조금씩 빨라지거나 느려진다.

이것을 **클럭 드리프트(Clock Drift)**라 한다.

```text
서버 A 클럭:  09:00:00.000 → 09:00:01.003 (1.003초 경과)
서버 B 클럭:  09:00:00.000 → 09:00:00.997 (0.997초 경과)

1시간 후:
서버 A: 09:59:59 (실제 10:00)
서버 B: 10:00:01 (실제 10:00)

차이: 2초
```

데이터센터에서 NTP(Network Time Protocol)로 동기화해도 오차가 발생한다.

인터넷을 통한 NTP 동기화의 정확도는 최대 수십 밀리초다.

### 클럭 스큐 (Clock Skew)

두 노드의 시계가 서로 다른 시각을 가리키는 현상이 **클럭 스큐**다.

실제 장애 시나리오를 보자.

```text
시나리오: 분산 데이터베이스에서 클럭 스큐로 인한 데이터 손실

서버 A (클럭: 10:00:00.100): "x = 1" 쓰기
서버 B (클럭: 10:00:00.050): "x = 2" 쓰기  ← 클럭이 50ms 느림

타임스탬프 기준 정렬:
  - 서버 B의 쓰기: 10:00:00.050 (먼저로 처리됨)
  - 서버 A의 쓰기: 10:00:00.100 (나중으로 처리됨)

결과: x = 1 (서버 A의 값이 최종 승리)
실제 순서: x = 2가 나중에 쓰였음에도 덮어씌워짐
```

이 문제를 물리 시계만으로는 해결할 수 없다.

Google Spanner는 GPS와 원자시계를 사용해 TrueTime API로 이 문제를 해결하지만, 일반 환경에서는 사치다.

### 논리 시계: Lamport Clock

1978년 Leslie Lamport가 제안한 **논리 시계(Lamport Clock)**는 물리 시간 없이 이벤트의 순서를 추론하는 방법이다.

핵심 아이디어는 "A가 B보다 먼저 일어났다면, A의 타임스탬프는 B보다 작다"는 관계를 유지하는 것이다.

**Lamport Clock 규칙:**

1. 프로세스 내에서 이벤트가 발생할 때마다 카운터를 1 증가시킨다.
2. 메시지를 보낼 때 현재 카운터 값을 함께 전송한다.
3. 메시지를 받을 때 `max(자신의 카운터, 수신한 카운터) + 1`로 카운터를 업데이트한다.

```text
프로세스 P1:   a(1) ──────────────────→ b(4) ─── c(5)
                         \
                          └──[msg, t=1]──→
프로세스 P2:   d(1) ─── e(2) ─── f(3) ──────────────→ g(max(3,1)+1=4+1→5)
                                              ↑
                                       메시지 수신, t=1 받음
                                       max(3, 1) + 1 = 4
```

실제 Go 코드로 구현하면 이렇다.

```go
type LamportClock struct {
    mu    sync.Mutex
    value uint64
}

// 로컬 이벤트 발생 시 호출
func (c *LamportClock) Tick() uint64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
    return c.value
}

// 메시지 전송 시 현재 타임스탬프 반환
func (c *LamportClock) Send() uint64 {
    return c.Tick()
}

// 메시지 수신 시 타임스탬프 업데이트
func (c *LamportClock) Receive(received uint64) uint64 {
    c.mu.Lock()
    defer c.mu.Unlock()
    if received > c.value {
        c.value = received
    }
    c.value++
    return c.value
}
```

### Lamport Clock의 한계와 Vector Clock

Lamport Clock은 인과 관계를 일부만 포착한다.

"타임스탬프가 같거나 역전된 두 이벤트가 실제로 어떤 관계인지" 알 수 없다.

이를 해결하기 위해 **Vector Clock**이 등장했다.

Vector Clock은 N개 프로세스 각각에 대한 카운터를 배열로 유지한다.

```text
Vector Clock 예시 (3개 노드):

노드 1: [1, 0, 0] → 이벤트 발생 → [2, 0, 0]
노드 2: [0, 1, 0] → 노드1의 메시지([2,0,0]) 수신 → [2, 2, 0]
노드 3: [0, 0, 1] → 노드2의 메시지([2,2,0]) 수신 → [2, 2, 2]

비교:
  [2, 0, 0] < [2, 2, 0]  → 인과 관계 있음 (노드1 이벤트가 먼저)
  [2, 0, 1] vs [0, 2, 0] → 비교 불가 (동시 발생, 충돌 처리 필요)
```

DynamoDB의 충돌 해결, CRDTs(Conflict-free Replicated Data Types) 등이 Vector Clock을 기반으로 한다.

---

## 합의 알고리즘: 분산 환경에서 하나의 결론을 내리는 법

### 왜 합의가 어려운가

여러 노드가 하나의 값에 동의해야 하는 문제를 **분산 합의(Distributed Consensus)**라 한다.

리더 선출, 분산 트랜잭션 커밋, 설정 변경 등 모든 곳에서 합의가 필요하다.

FLP Impossibility Theorem(1985)은 충격적인 사실을 증명했다.

"비동기 네트워크에서 단 하나의 노드 장애만 있어도, 합의를 보장하는 결정적 알고리즘은 존재하지 않는다."

실용적인 합의 알고리즘은 이 한계를 "충분히 오랜 안정적 기간"을 가정함으로써 우회한다.

### Paxos: 합의 알고리즘의 원조

Leslie Lamport가 1989년 제안한 **Paxos**는 분산 합의의 이론적 토대다.

Paxos는 세 단계로 동작한다.

```text
Phase 1 (Prepare):
  Proposer → 모든 Acceptor: "제안 번호 n으로 준비됐어?"
  Acceptor → Proposer: "OK, 이전에 받은 최고 제안은 (m, v)야"

Phase 2 (Accept):
  Proposer → 과반수 Acceptor: "제안 (n, v)를 수락해줘"
  Acceptor → Proposer: "수락했어"

Phase 3 (Learn):
  Proposer → Learner: "값 v로 합의됐어"
```

Paxos는 정확하지만 이해하기 어렵기로 악명 높다.

Lamport 본인도 "논문을 쓴 뒤 구현하는 데 1년이 걸렸다"고 회고했다.

### Raft: 이해하기 위해 설계된 합의 알고리즘

2013년 Diego Ongaro와 John Ousterhout가 발표한 **Raft**는 "이해 가능성"을 최우선으로 설계됐다.

etcd, CockroachDB, TiKV가 Raft를 사용한다.

Raft는 세 가지 역할로 구성된다.

- **Leader**: 모든 쓰기 요청을 처리하고 다른 노드에 복제한다.
- **Follower**: Leader의 명령을 따른다. 리더가 없으면 후보자가 된다.
- **Candidate**: 리더 선출에 참여하는 상태.

**리더 선출 과정:**

```text
초기 상태: 모든 노드가 Follower

1. Follower가 일정 시간(election timeout, 150~300ms 랜덤) 동안
   Leader의 heartbeat를 받지 못하면 Candidate가 됨

2. Candidate가 자신의 term을 1 증가시키고
   모든 노드에게 RequestVote RPC 전송

3. 과반수(N/2 + 1)의 투표를 받으면 Leader가 됨

4. Leader는 주기적으로 heartbeat(빈 AppendEntries RPC) 전송
```

**로그 복제 과정:**

```text
Client ──[Write: x=1]──→ Leader
                         Leader: 로그에 (term=3, index=5, x=1) 추가
                         Leader ──[AppendEntries]──→ Follower 1
                         Leader ──[AppendEntries]──→ Follower 2

Follower 1, 2가 ACK 응답:
  Leader: 과반수(3/3) 확인 → Commit
  Leader ──[x=1 committed]──→ Client
  Leader는 다음 heartbeat에서 Follower에게 commit 사실 전달
```

**Split Brain 방지:**

Raft에서 네트워크 파티션이 발생하면 어떻게 될까.

```text
파티션 전: [Node1(Leader), Node2, Node3, Node4, Node5]

파티션 발생:
  그룹 A: [Node1(Leader), Node2]       ← 과반수 미달 (2/5)
  그룹 B: [Node3, Node4, Node5]        ← 과반수 충족 (3/5)

그룹 B: 새 리더 선출 (Node3가 Leader, term 4)
그룹 A: Node1은 term 3으로 계속 시도하지만 과반수 확보 불가

파티션 복구 후:
  Node1이 더 높은 term(4)을 발견 → 자동으로 Follower로 전환
```

이 메커니즘 덕분에 두 개의 Leader가 동시에 커밋을 진행하는 일이 발생하지 않는다.

**Raft 구현 시 흔한 안티패턴:**

```text
[안티패턴 1] Election Timeout 고정값 사용

모든 노드가 동일한 election timeout을 가지면,
동시에 Candidate가 되어 split vote가 반복된다.

해결: 150~300ms 사이 랜덤 타임아웃 (Raft 논문 권장)

[안티패턴 2] 로그 압축(Snapshot) 미구현

장기 운영 시 Raft 로그가 무한정 증가한다.
주기적 스냅샷과 로그 트리밍은 필수다.

[안티패턴 3] 리더 변경 중 클라이언트 요청 처리

리더 선출 중(수백 ms)에는 쓰기 요청을 거부해야 한다.
이 기간을 숨기려다 부분 쓰기나 중복 처리가 발생한다.
```

### Paxos vs Raft 비교

| 항목 | Paxos (Multi-Paxos) | Raft |
|------|---------------------|------|
| 이해 난이도 | 매우 높음 | 중간 |
| 리더 제약 | 없음 (어떤 노드도 제안 가능) | 강한 리더 (리더만 쓰기 가능) |
| 로그 순서 보장 | 복잡한 재정렬 필요 | 강하게 보장 |
| 대표 구현 | Google Chubby, ZooKeeper (ZAB 기반) | etcd, CockroachDB, TiKV |
| 성능 | 최적화 여지 많음 | 예측 가능한 레이턴시 |

---

## 분산 시스템 테스트의 어려움

### 왜 일반 테스트로는 부족한가

단위 테스트와 통합 테스트는 "정상 경로"를 검증한다.

하지만 분산 시스템의 장애는 네트워크 지연, 노드 다운, 클럭 스큐, 메모리 압박이 동시에 발생할 때 나타난다.

이런 조합을 로컬 테스트로 재현하기는 사실상 불가능하다.

```text
[분산 시스템 테스트의 어려움]

재현 불가능성: 네트워크 타이밍에 따라 결과가 달라짐
폭발적 상태 공간: N개 노드 × M개 이벤트 = 수십억 가지 조합
부분 장애: 노드가 "살아있지만 응답이 느린" 상태 재현 어려움
시간 의존성: 타임아웃, 재시도 간격이 테스트 속도와 충돌
```

### Jepsen: 분산 시스템 안전성 검증 도구

Kyle Kingsbury(aphyr)가 만든 **Jepsen**은 분산 데이터베이스의 안전성 속성을 자동으로 검증한다.

Jepsen은 실제 클러스터에서 네트워크 파티션, 클럭 스큐, 노드 재시작을 주입하면서 데이터 불일치가 발생하는지 확인한다.

MongoDB, Cassandra, Redis, etcd 등 유명 시스템들이 Jepsen 테스트에서 치명적 버그가 발견됐다.

### Chaos Engineering: 넷플릭스의 철학

**Chaos Engineering**은 "프로덕션에서 고의적으로 장애를 일으켜, 시스템이 그 장애를 견딜 수 있음을 증명하는" 방법론이다.

넷플릭스가 2010년 AWS로 마이그레이션하면서 만든 **Chaos Monkey**가 시초다.

```text
Chaos Engineering의 핵심 원칙:

1. 정상 상태 행동 정의
   "초당 처리량 10,000 RPS, P99 레이턴시 50ms 이하"

2. 가설 수립
   "하나의 가용 영역이 다운돼도 정상 상태가 유지된다"

3. 다양한 장애 주입
   - 인스턴스 종료 (Chaos Monkey)
   - 가용 영역 차단 (Chaos Kong)
   - 레이턴시 주입 (Latency Monkey)
   - CPU/메모리 압박

4. 프로덕션에서 실험 (또는 스테이징)

5. 반경 최소화
   문제 발생 시 즉시 중단할 수 있어야 함
```

**Chaos Engineering 도구 생태계:**

```text
Netflix Chaos Monkey:    EC2 인스턴스 랜덤 종료
Gremlin:                 SaaS 형태의 포괄적 장애 주입
Chaos Mesh (쿠버네티스): Pod 장애, 네트워크 파티션, 클럭 스큐
Litmus:                  CNCF 프로젝트, 쿠버네티스 네이티브
AWS Fault Injection Simulator: AWS 매니지드 Chaos Engineering
```

### Chaos Engineering 실전 예시

```yaml
# Chaos Mesh - 네트워크 지연 주입 예시
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-service-latency
spec:
  action: delay
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: payment-service
  delay:
    latency: "200ms"
    correlation: "25"
    jitter: "50ms"
  duration: "5m"
  scheduler:
    cron: "@every 1h"
```

이 설정은 매 1시간마다 5분 동안 결제 서비스에 200ms 레이턴시를 주입한다.

서킷 브레이커와 타임아웃이 제대로 동작하는지 확인할 수 있다.

### 분산 시스템 테스트 전략 계층

```text
Level 0: Unit Test
  - 개별 컴포넌트 로직 검증
  - 빠르고 결정적, 하지만 분산 특성 미반영

Level 1: Integration Test
  - 실제 의존성(DB, 메시지 큐) 사용
  - Docker Compose로 로컬 환경 구성

Level 2: Contract Test (Pact)
  - 서비스 간 API 계약 검증
  - Consumer-Driven Contract Testing

Level 3: Chaos Test (Staging)
  - 장애 주입 후 SLO 유지 확인
  - Jepsen, Chaos Mesh 활용

Level 4: GameDay (Production)
  - 전체 팀이 참여하는 실제 장애 시뮬레이션
  - 넷플릭스의 Chaos Kong Exercise
```

**안티패턴: Happy Path만 테스트하기**

```text
[나쁜 예]
func TestOrderService(t *testing.T) {
    // 항상 성공하는 환경에서만 테스트
    result := orderService.CreateOrder(validRequest)
    assert.Equal(t, "success", result.Status)
}

// 테스트가 통과해도 다음 상황은 검증 안 됨:
// - 결제 서비스가 3초 후 응답할 때
// - 재고 서비스가 두 번째 호출에서만 실패할 때
// - 데이터베이스 연결이 50%의 확률로 끊길 때
```

```text
[좋은 예 - Fault Injection 포함]
func TestOrderService_PaymentTimeout(t *testing.T) {
    // 결제 서비스를 5초 지연 mock으로 교체
    paymentMock := NewSlowMock(5 * time.Second)
    orderService := NewOrderService(paymentMock, ...)

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    result, err := orderService.CreateOrder(ctx, validRequest)

    // 서킷 브레이커가 동작해 빠른 실패를 반환해야 함
    assert.Error(t, err)
    assert.Equal(t, ErrPaymentTimeout, err)
    assert.Equal(t, "pending", result.Status) // 롤백 상태
}
```

---

## 핵심 정리

분산 시스템을 제대로 다루려면 세 가지 사실을 몸으로 받아들여야 한다.

**첫째, 네트워크와 노드는 항상 실패할 수 있다.** 이것을 예외 상황이 아니라 정상 동작의 일부로 설계해야 한다.

**둘째, 물리 시계는 거짓말한다.** 분산 환경에서 시간 순서는 물리 시계로 보장할 수 없다. 논리 시계나 인과성 추적이 필요하다.

**셋째, 합의는 비싸다.** Raft나 Paxos가 제공하는 강한 일관성은 레이턴시 비용을 수반한다. 필요한 곳에만 사용하고, 나머지는 최종 일관성(Eventual Consistency)을 수용해야 한다.

다음 편에서는 이 기초 위에서 CAP 정리와 실제 분산 데이터베이스 설계를 다룬다.

---

## 참고 자료

1. Lamport, L. (1978). "Time, Clocks, and the Ordering of Events in a Distributed System." *Communications of the ACM*, 21(7), 558–565.
2. Ongaro, D., & Ousterhout, J. (2014). "In Search of an Understandable Consensus Algorithm." *USENIX ATC 2014*. https://raft.github.io/raft.pdf
3. Deutsch, P. et al. (1994). "The Eight Fallacies of Distributed Computing." Sun Microsystems.
4. Fischer, M. J., Lynch, N. A., & Paterson, M. S. (1985). "Impossibility of Distributed Consensus with One Faulty Process." *Journal of the ACM*, 32(2), 374–382.
5. Basiri, A. et al. (2016). "Chaos Engineering." *IEEE Software*, 33(3), 35–41. Netflix Technology Blog.
6. Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. (특히 8~9장: 분산 시스템의 트러블과 일관성·합의)
