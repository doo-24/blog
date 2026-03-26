---
title: "[데이터베이스] 8편 — DB 스케일링: 데이터베이스 확장의 현실"
date: 2026-03-17T23:02:00+09:00
draft: false
tags: ["DB 스케일링", "샤딩", "레플리카", "분산 DB", "데이터베이스", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "Read Replica 설계와 복제 지연, 수평 샤딩 전략(범위, 해시, 디렉토리)과 리샤딩 문제, Vitess·CockroachDB 개요, 글로벌 데이터베이스 설계와 지역성 고려까지"
---

데이터베이스는 대부분의 시스템에서 가장 먼저 병목이 된다.

웹 서버는 수평 확장이 쉽다. 로드 밸런서 뒤에 인스턴스를 추가하면 된다. 캐시도 마찬가지다.

하지만 데이터베이스는 다르다. 상태를 가지고 있고, 일관성을 보장해야 하며, 데이터는 어딘가에 영구적으로 저장되어야 한다.

이 제약 조건 안에서 어떻게 확장할 것인가 — 이것이 DB 스케일링의 핵심 질문이다.

트래픽이 10배, 100배 증가할 때 "그냥 더 큰 서버로 교체하면 되지 않나?"라고 생각한다면, 이 글을 끝까지 읽어보길 권한다. 현실은 그보다 훨씬 복잡하고, 더 흥미롭다.

---

## Read Replica: 읽기 트래픽의 분산

### 왜 Read Replica가 필요한가

대부분의 웹 애플리케이션은 읽기 비율이 쓰기보다 압도적으로 높다. 전형적인 SNS를 생각해보자.

사용자가 게시물을 작성하는 빈도보다 피드를 조회하는 빈도가 수십 배 이상 높다. 이커머스라면 주문보다 상품 조회가 몇 백 배는 더 많다.

단일 Primary 데이터베이스 서버에 모든 읽기와 쓰기를 몰아넣으면, 읽기 쿼리가 쓰기 작업과 경합하면서 전체 처리량이 제한된다. CPU, 디스크 I/O, 네트워크 대역폭이 하나의 서버에서 공유된다.

읽기 전용 Replica를 추가하면 읽기 트래픽을 분산시켜 Primary의 부하를 낮출 수 있다.

```
                    애플리케이션
                   /             \
          (쓰기 요청)           (읽기 요청)
               |                    |
           Primary            Replica Pool
          [Leader]      [Replica1] [Replica2] [Replica3]
               |              |         |          |
          쓰기 로그 -------> 복제 -----> 복제 ----> 복제
         (binlog/WAL)
```

### 복제 방식: 동기 vs 비동기

Read Replica 설계에서 가장 중요한 선택은 복제 방식이다.

**비동기 복제 (Asynchronous Replication)**

MySQL의 기본 복제 방식이다. Primary가 트랜잭션을 커밋하고 즉시 클라이언트에 응답한 뒤, 백그라운드에서 Replica에게 변경 사항을 전송한다. 성능이 뛰어나다. Primary의 응답 지연에 네트워크 왕복이 추가되지 않는다.

```
Primary ──[커밋]──> 클라이언트 응답
    │
    └──[비동기]──> Replica 전송 (나중에)
```

단점은 데이터 손실 가능성이다. Primary가 커밋 후 Replica에 전달하기 전에 장애가 발생하면, 해당 트랜잭션은 유실된다. 또한 Replica가 Primary보다 뒤처지는 "복제 지연(Replication Lag)"이 발생한다.

**동기 복제 (Synchronous Replication)**

Primary가 트랜잭션을 커밋하려면 최소 하나의 Replica가 데이터를 받았다고 확인해줘야 한다. PostgreSQL의 Synchronous Standby 설정이 이 방식이다.

```sql
-- PostgreSQL synchronous_standby_names 설정
synchronous_standby_names = 'FIRST 1 (replica1, replica2)'
```

이 설정에서 Primary는 replica1 또는 replica2 중 하나로부터 ACK를 받아야 커밋이 완료된다. 데이터 손실 가능성이 없지만, 네트워크 지연이 쓰기 응답 시간에 직접 영향을 준다. 같은 데이터센터 내에서는 1ms 미만이지만, 리전 간이라면 수십 ms가 추가된다.

**반동기 복제 (Semi-Synchronous Replication)**

MySQL의 반동기 복제는 중간 지점이다. Primary는 적어도 하나의 Replica가 relay log에 데이터를 기록했다는 ACK를 받은 후 커밋한다. Replica가 실제로 트랜잭션을 적용(apply)했는지는 확인하지 않는다.

```sql
-- MySQL 반동기 복제 설정
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000; -- 1초 타임아웃
```

타임아웃이 지나면 비동기 모드로 자동 전환되어 가용성을 유지한다. 완전한 동기 복제보다는 위험하지만 순수 비동기보다는 안전하다.

### 복제 지연: 허용 범위와 관리

복제 지연은 비동기 복제에서 피할 수 없다. 문제는 "얼마나 지연을 허용할 것인가"다.

**복제 지연이 발생하는 원인**

1. **네트워크 대역폭**: binlog 전송량이 네트워크를 포화시킬 때
2. **Replica의 처리 능력**: 단일 스레드 적용(apply) 방식에서 병목 발생
3. **대형 트랜잭션**: 수백만 행을 수정하는 DDL이나 배치 작업
4. **I/O 경합**: Replica에서 읽기 쿼리와 복제 적용이 경합

```sql
-- MySQL에서 복제 지연 확인
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 현재 복제 지연 초

-- PostgreSQL에서 복제 지연 확인
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

**복제 지연이 만드는 버그**

복제 지연에서 가장 흔한 버그는 "쓰기 후 읽기(Read-Your-Writes)" 불일치다.

시나리오: 사용자가 프로필 사진을 변경한다. 요청이 Primary에 쓰인다. 즉시 프로필 페이지로 리다이렉트된다.

이 읽기 요청은 Replica로 라우팅된다. Replica에는 아직 변경이 반영되지 않아 이전 사진이 보인다. 사용자는 변경이 실패했다고 생각하고 다시 시도한다.

이를 해결하는 방법은 몇 가지가 있다.

```python
# 방법 1: 쓰기 직후 일정 시간 동안 Primary에서 읽기
def update_profile(user_id, new_data):
    primary_db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    # 세션에 타임스탬프 저장
    session['primary_read_until'] = time.time() + 5  # 5초

def get_profile(user_id):
    if session.get('primary_read_until', 0) > time.time():
        return primary_db.query("SELECT * FROM users WHERE id = ?", user_id)
    return replica_db.query("SELECT * FROM users WHERE id = ?", user_id)

# 방법 2: 쓰기 시 LSN/binlog position을 기록하고 Replica가 따라잡을 때까지 대기
def update_profile_sync(user_id, new_data):
    primary_db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    lsn = primary_db.query("SELECT pg_current_wal_lsn()")[0][0]
    # Replica가 해당 LSN에 도달할 때까지 대기
    wait_for_replica_catchup(lsn, timeout=2.0)
```

**허용 가능한 복제 지연 기준**

서비스마다 다르지만 일반적인 기준이 있다.

- **금융/결제**: 실질적으로 0. 동기 복제 또는 Primary에서만 읽기
- **주문 처리**: 1초 미만. 주문 직후 조회는 Primary에서 처리
- **콘텐츠 피드**: 수십 초까지 허용 가능. 다른 사람의 새 게시물이 잠시 안 보여도 큰 문제가 없음
- **분석/통계**: 수 분도 허용. 대시보드가 실시간일 필요가 없음

모니터링은 필수다. 복제 지연이 갑자기 급증하면 알람이 울려야 한다.

```yaml
# Prometheus AlertManager 예시
- alert: ReplicationLagHigh
  expr: mysql_slave_lag_seconds > 30
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "MySQL replication lag is {{ $value }}s on {{ $labels.instance }}"
```

### Replica 라우팅 전략

Read Replica를 추가했다고 자동으로 트래픽이 분산되지 않는다. 애플리케이션이나 프록시 레이어에서 라우팅 결정을 내려야 한다.

**ProxySQL 기반 자동 라우팅**

```sql
-- ProxySQL에서 읽기/쓰기 분리 규칙 설정
INSERT INTO mysql_query_rules (rule_id, active, match_digest, destination_hostgroup, apply)
VALUES
    (1, 1, '^SELECT.*FOR UPDATE', 1, 1),  -- SELECT FOR UPDATE는 Primary
    (2, 1, '^SELECT', 2, 1),              -- 일반 SELECT는 Replica
    (3, 1, '.*', 1, 1);                   -- 나머지는 Primary

-- Hostgroup 설정
INSERT INTO mysql_replication_hostgroups (writer_hostgroup, reader_hostgroup, comment)
VALUES (1, 2, 'cluster1');
```

**애플리케이션 레이어 라우팅 (SQLAlchemy 예시)**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

primary_engine = create_engine("postgresql://primary-host/db")
replica_engines = [
    create_engine("postgresql://replica1-host/db"),
    create_engine("postgresql://replica2-host/db"),
]

class RoutingSession:
    def __init__(self):
        self._primary_session = sessionmaker(bind=primary_engine)()
        self._replica_sessions = [
            sessionmaker(bind=e)() for e in replica_engines
        ]
        self._replica_index = 0

    def write(self):
        return self._primary_session

    def read(self):
        # Round-robin
        session = self._replica_sessions[self._replica_index]
        self._replica_index = (self._replica_index + 1) % len(self._replica_sessions)
        return session
```

---

## 수평 샤딩: 데이터를 쪼개는 전략

### 샤딩이 필요한 시점

Read Replica는 읽기 확장을 해결하지만 쓰기 확장은 해결하지 못한다. 모든 쓰기는 여전히 Primary 하나에 집중된다. Primary의 쓰기 처리량에 한계가 오면, 더 큰 서버(Scale Up)를 사용하거나 데이터 자체를 여러 서버에 나누어야 한다.

샤딩(Sharding)은 데이터를 여러 데이터베이스 서버(샤드)에 수평으로 분산하는 기법이다. 각 샤드는 전체 데이터의 일부만 가지며, 독립적으로 쓰기와 읽기를 처리한다. 단일 서버의 물리적 한계(CPU, 디스크, 메모리)를 넘어설 수 있는 유일한 방법이지만, 그 대가로 쿼리 복잡도와 운영 난이도가 크게 높아진다.

```
전체 데이터: users 테이블 (1억 행)

샤드 1: user_id 1 ~ 25,000,000
샤드 2: user_id 25,000,001 ~ 50,000,000
샤드 3: user_id 50,000,001 ~ 75,000,000
샤드 4: user_id 75,000,001 ~ 100,000,000
```

샤딩은 복잡성의 대가가 크다. 단순한 JOIN이 크로스-샤드 쿼리가 되고, 트랜잭션이 분산 트랜잭션이 된다. 샤딩은 다른 방법을 모두 소진한 후에 선택해야 한다.

### 샤딩 전략 1: 범위 기반 샤딩 (Range Sharding)

범위 기반 샤딩은 샤드 키의 값 범위에 따라 데이터를 배분한다.

```
user_id 기반 범위 샤딩:
┌─────────────────────────────────────────────┐
│ Shard 1: user_id < 1,000,000                │
│ Shard 2: 1,000,000 <= user_id < 2,000,000   │
│ Shard 3: 2,000,000 <= user_id < 3,000,000   │
│ Shard 4: 3,000,000 <= user_id               │
└─────────────────────────────────────────────┘
```

**장점**

범위 쿼리가 효율적이다. "user_id 1000 ~ 5000의 사용자 조회"는 단일 샤드에서 처리된다. 데이터 지역성이 좋다. 시간 기반 샤딩에서 최신 데이터는 최신 샤드에 집중된다.

**단점**

핫스팟(Hot Spot)이 발생하기 쉽다. 단조 증가하는 ID나 타임스탬프를 샤드 키로 사용하면 새로운 쓰기가 항상 마지막 샤드에 집중된다.

```
시간 기반 파티셔닝에서의 핫스팟:

Shard 1 (2023년): ████░░░░░░ (과거, 쓰기 없음)
Shard 2 (2024년): ████████░░ (과거, 쓰기 거의 없음)
Shard 3 (2025년): ████████████ (현재, 모든 쓰기 집중!)
```

이를 피하기 위해 순차적 ID 대신 UUID나 Snowflake ID를 사용하거나, 범위를 동적으로 분할하는 방식을 채택한다.

### 샤딩 전략 2: 해시 기반 샤딩 (Hash Sharding)

해시 기반 샤딩은 샤드 키에 해시 함수를 적용해 균등하게 데이터를 분산한다.

```python
def get_shard(user_id: int, num_shards: int) -> int:
    return hash(user_id) % num_shards

# user_id=12345 → hash(12345) % 4 = 1 → Shard 2
# user_id=67890 → hash(67890) % 4 = 3 → Shard 4
```

**장점**

데이터가 균등하게 분산된다. 핫스팟이 발생하지 않는다. 구현이 단순하다.

**단점**

범위 쿼리가 모든 샤드를 탐색해야 한다. "1000번에서 2000번 사용자 목록"을 가져오려면 4개 샤드 모두에 쿼리를 보내고 결과를 합쳐야 한다.

더 심각한 문제는 샤드 수를 변경할 때다. 샤드가 4개에서 5개로 늘어나면 `hash(user_id) % 5` 결과가 기존의 `% 4` 결과와 달라진다.

대부분의 데이터가 잘못된 샤드에 있게 된다. 대규모 데이터 마이그레이션이 필요하다.

**일관된 해싱 (Consistent Hashing)**

이 문제를 완화하기 위해 일관된 해싱을 사용한다. 샤드와 키를 동일한 해시 링(ring)에 배치하고, 각 키는 시계 방향으로 가장 가까운 샤드에 배정된다.

```
해시 링 (0 ~ 2^32):

            0
           / \
      S4  /   \  S1
         /     \
        /       \
    ---           ---
       \         /
     S3 \       / S2
         \     /
          \   /
        2^32

샤드 추가 시: 새 샤드 S5가 S2와 S3 사이에 들어오면
S2~S5 범위의 데이터만 S3에서 S5로 이동.
나머지 데이터는 움직이지 않음.
```

일관된 해싱은 샤드 추가/제거 시 이동해야 하는 데이터를 최소화한다. 일반 해시(% N)는 N이 바뀌면 거의 모든 키가 재배치되어야 하지만, 일관된 해싱은 평균적으로 전체 키의 1/N만 이동한다. Cassandra, DynamoDB가 이 방식을 사용한다.

### 샤딩 전략 3: 디렉토리 기반 샤딩 (Directory Sharding)

디렉토리 기반 샤딩은 별도의 조회 테이블(Shard Map)을 유지하여 각 키가 어느 샤드에 있는지 기록한다.

```
Shard Directory (별도 서비스 또는 DB):
┌──────────────┬──────────┐
│ tenant_id    │ shard_id │
├──────────────┼──────────┤
│ acme-corp    │ shard-3  │
│ globex       │ shard-1  │
│ initech      │ shard-2  │
│ umbrella     │ shard-3  │
└──────────────┴──────────┘

쿼리 흐름:
1. tenant_id='acme-corp'로 요청 도착
2. Shard Directory에서 → shard-3
3. shard-3에서 실제 데이터 조회
```

**장점**

가장 유연하다. 특정 테넌트를 다른 샤드로 이동시키려면 Directory만 업데이트하면 된다.

불균등한 데이터 분산을 수동으로 조정할 수 있다. 범위 샤딩처럼 범위 쿼리도 지원된다.

**단점**

Shard Directory가 단일 장애점(SPOF)이 될 수 있다. 모든 쿼리에 디렉토리 조회가 추가된다. 이를 완화하기 위해 디렉토리를 캐시하거나, 디렉토리 자체를 고가용성으로 운영한다.

```python
class ShardRouter:
    def __init__(self, directory_db, cache_ttl=60):
        self.directory_db = directory_db
        self.cache = TTLCache(maxsize=10000, ttl=cache_ttl)

    def get_shard(self, tenant_id: str) -> str:
        if tenant_id in self.cache:
            return self.cache[tenant_id]

        row = self.directory_db.query(
            "SELECT shard_id FROM shard_directory WHERE tenant_id = ?",
            tenant_id
        )
        shard_id = row[0]['shard_id']
        self.cache[tenant_id] = shard_id
        return shard_id
```

### 리샤딩: 가장 어려운 작업

샤딩 후 데이터가 증가하면 결국 리샤딩(Resharding)이 필요하다. 샤드 수를 늘리고 데이터를 재분배하는 작업이다. 이것이 샤딩의 가장 고통스러운 부분이다.

**리샤딩의 어려움**

서비스를 중단하지 않고 리샤딩해야 한다. 데이터를 옮기는 동안에도 읽기와 쓰기가 계속 발생한다. 데이터가 이동 중인 상태에서 쿼리가 잘못된 샤드를 바라보면 안 된다.

**온라인 리샤딩 패턴**

```
리샤딩 단계:

1단계: 이중 쓰기 시작
   - 모든 새 쓰기를 Old Shard와 New Shard 양쪽에 기록

2단계: 기존 데이터 마이그레이션
   - Old Shard의 기존 데이터를 New Shard로 복사
   - 복사 중 발생한 변경은 이중 쓰기로 처리됨

3단계: 검증
   - New Shard의 데이터가 Old Shard와 일치하는지 확인

4단계: 트래픽 전환
   - 읽기 트래픽을 점진적으로 New Shard로 이동
   - 문제 발생 시 즉시 롤백

5단계: Old Shard 폐기
   - 일정 기간 관찰 후 Old Shard 삭제
```

```python
# 이중 쓰기 예시
class ReshardingRouter:
    def __init__(self, old_router, new_router, migration_state):
        self.old_router = old_router
        self.new_router = new_router
        self.migration_state = migration_state  # 어느 키가 이동 완료됐는지

    def write(self, key, data):
        # 항상 두 곳에 씀
        old_shard = self.old_router.get_shard(key)
        new_shard = self.new_router.get_shard(key)
        old_shard.write(key, data)
        new_shard.write(key, data)

    def read(self, key):
        if self.migration_state.is_migrated(key):
            # 마이그레이션 완료된 키는 New Shard에서 읽기
            return self.new_router.get_shard(key).read(key)
        else:
            # 아직 이동 중인 키는 Old Shard에서 읽기
            return self.old_router.get_shard(key).read(key)
```

**샤드 키 선택의 중요성**

잘못된 샤드 키를 선택하면 리샤딩보다 더 큰 문제가 생긴다.

나쁜 샤드 키의 예:
- **성별(gender)**: 카디널리티가 너무 낮다. 샤드가 2~3개밖에 안 된다
- **국가 코드**: 미국 트래픽이 전체의 50%라면 미국 샤드가 핫스팟
- **타임스탬프(단조 증가)**: 범위 샤딩에서 마지막 샤드에 쓰기 집중

좋은 샤드 키의 조건:
1. **높은 카디널리티**: 충분히 많은 고유값
2. **균등한 분포**: 특정 값에 쓰기가 집중되지 않음
3. **쿼리 패턴과 정렬**: 자주 함께 조회되는 데이터가 같은 샤드에 위치

---

## Vitess와 CockroachDB: 분산 DB 솔루션

직접 샤딩을 구현하면 복잡도가 폭발적으로 증가한다. 이를 플랫폼 레벨에서 해결한 솔루션들이 있다.

### Vitess: MySQL의 수평 확장

Vitess는 YouTube에서 MySQL을 수평 확장하기 위해 개발했고 현재 CNCF 졸업 프로젝트다. Vitess는 MySQL 위에서 동작하며, 샤딩 로직을 애플리케이션이 아닌 미들웨어 레이어에서 처리한다.

```
Vitess 아키텍처:

애플리케이션
    │ (MySQL 프로토콜)
    ▼
VTGate (쿼리 라우터)
  ├── 쿼리 파싱
  ├── 샤드 결정
  └── 크로스-샤드 쿼리 실행
    │
    ├──────────────────────────┐
    ▼                          ▼
VTTablet (Shard 1)        VTTablet (Shard 2)
  ├── Primary MySQL          ├── Primary MySQL
  └── Replica MySQL          └── Replica MySQL

Topology Service (etcd/ZooKeeper):
  - 샤드 메타데이터
  - 서버 상태
  - 스키마 정보
```

**Vitess의 핵심 기능**

VTGate는 MySQL 프로토콜을 완전히 구현하여 애플리케이션에서는 단일 MySQL처럼 보인다. 크로스-샤드 JOIN을 지원하되, 성능 저하를 경고한다.

```sql
-- 애플리케이션에서는 그냥 MySQL 쿼리
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.id = 12345;

-- Vitess 내부에서는:
-- 1. user_id=12345 → shard-2
-- 2. shard-2에서 users 조회
-- 3. shard-2에서 orders 조회 (같은 샤드에 있으면)
-- 4. 결과 병합
```

**MoveTables**: 서비스 중단 없이 테이블을 샤드 간 이동

```bash
# Vitess에서 테이블 이동
vtctlclient MoveTables \
  -workflow=move_orders \
  -source_keyspace=main \
  -target_keyspace=orders_keyspace \
  Orders

# 이동 완료 후 트래픽 전환
vtctlclient SwitchTraffic -workflow=move_orders orders_keyspace
```

**VReplication**: Vitess 내부의 복제 엔진으로 필터링 복제와 데이터 변환을 지원한다. Kafka 없이도 변경 데이터 캡처(CDC)가 가능하다.

Vitess를 선택할 때: MySQL 기반 시스템에서 샤딩이 필요하고, MySQL 생태계(도구, 운영 노하우)를 유지하고 싶을 때. PlanetScale이 Vitess 기반 서비스를 제공한다.

### CockroachDB: 분산 SQL 데이터베이스

CockroachDB는 다른 접근법을 취한다. MySQL이나 PostgreSQL 위에 샤딩 레이어를 얹는 게 아니라, 처음부터 분산을 고려해 설계된 SQL 데이터베이스다.

```
CockroachDB 아키텍처:

    SQL Layer (PostgreSQL 호환 인터페이스)
         │
    Transaction Layer (Serializable 격리)
         │
    Distribution Layer (Range 기반 자동 샤딩)
         │
    Replication Layer (Raft 합의 프로토콜)
         │
    Storage Layer (RocksDB/Pebble)

노드들:
Node 1    Node 2    Node 3    Node 4    Node 5
[Range A] [Range B] [Range C] [Range A] [Range B]
[Range D] [Range E] [Range F] [Range E] [Range F]
(각 Range는 3개 복제본을 Raft로 관리)
```

**CockroachDB의 핵심 특성**

*자동 샤딩*: 데이터는 내부적으로 Range(기본 64MB)로 분할되어 노드에 자동 배포된다. 개발자가 샤드 키를 결정할 필요가 없다.

*ACID 트랜잭션 (분산 환경)*: CockroachDB는 Serializable 격리 수준을 분산 환경에서 보장한다. 이는 Spanner의 TrueTime 개념에서 영감을 받아 하이브리드 논리 클록(HLC)을 사용한다.

```sql
-- CockroachDB에서 크로스-노드 트랜잭션 (애플리케이션 코드는 일반 SQL)
BEGIN;
INSERT INTO orders (id, user_id, total) VALUES (gen_random_uuid(), 1001, 99.99);
UPDATE inventory SET stock = stock - 1 WHERE product_id = 5;
COMMIT;
-- 내부에서는 2개 다른 Range에 걸친 분산 트랜잭션
```

*고가용성*: Raft 합의 프로토콜로 각 Range는 3개(또는 5개) 복제본을 유지한다. 노드가 죽어도 자동으로 복구된다.

**CockroachDB 제약사항**

완벽한 해결책은 없다. CockroachDB의 단점도 명확하다.

- 단일 행 읽기 지연이 PostgreSQL보다 높다 (Raft 합의 오버헤드)
- 핫스팟 Row에 대한 성능이 중앙화 DB보다 낮을 수 있음
- 스토리지 오버헤드: 3배 복제
- 운영 복잡도가 높다

```sql
-- CockroachDB 핫스팟 방지: 자동 증가 ID 대신 UUID 사용
CREATE TABLE orders (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    -- 순차 ID 대신 UUID를 사용하면 Range 간 균등 분산
    user_id INT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

-- 또는 HASH 샤딩으로 핫스팟 방지
CREATE TABLE events (
    id INT NOT NULL,
    data JSONB,
    PRIMARY KEY (id) USING HASH WITH BUCKET_COUNT = 8
);
```

**Vitess vs CockroachDB 선택 기준**

| 기준 | Vitess | CockroachDB |
|------|--------|-------------|
| 기존 MySQL 마이그레이션 | 쉬움 | 어려움 (다른 DB) |
| 운영 MySQL 노하우 활용 | 가능 | 불가 |
| 자동 샤딩 | 반자동 | 완전 자동 |
| 분산 트랜잭션 | 제한적 | 완전 지원 |
| 단일 row 지연 | 낮음 | 상대적으로 높음 |
| 글로벌 배포 | 가능 (복잡) | 기본 지원 |

---

## 글로벌 데이터베이스 설계

### 글로벌 배포의 도전

서비스가 전 세계 사용자를 대상으로 한다면, 단순히 한 리전에서 스케일 아웃하는 것으로는 부족하다. 서울에 있는 데이터베이스에 뉴욕 사용자가 접속하면 왕복 지연(RTT)이 150ms 이상이다. 트랜잭션마다 이 지연이 누적된다.

```
지구상 RTT 현실:
서울 ↔ 도쿄:    ~30ms
서울 ↔ 싱가포르: ~70ms
서울 ↔ 프랑크푸르트: ~200ms
서울 ↔ 뉴욕:    ~200ms
서울 ↔ 상파울루: ~300ms

10번의 직렬 쿼리: 서울 사용자 70ms vs 뉴욕 사용자 2000ms
```

글로벌 데이터베이스 설계는 세 가지 접근법으로 나뉜다.

### 접근법 1: 멀티-리전 Active-Passive

하나의 Primary 리전에서 모든 쓰기를 처리하고, 다른 리전에는 Read Replica를 배치한다.

```
                     글로벌 DNS / GeoDNS
                    /        |          \
                  /          |            \
           서울(Active)  도쿄(Passive)  프랑크푸르트(Passive)
           Primary         Replica          Replica
           ────────         ────────         ────────
           읽기/쓰기         읽기 전용         읽기 전용
               │                │                 │
               └────────────────┘─────────────────┘
                        비동기 복제
```

**읽기는 로컬, 쓰기는 Primary로**

뉴욕 사용자가 피드를 읽는다 → 가장 가까운 리전(프랑크푸르트)에서 읽기. 뉴욕 사용자가 댓글을 작성한다 → 서울 Primary로 쓰기. 쓰기 지연은 높지만 서비스가 가능하다.

```nginx
# GeoDNS 기반 라우팅 (Route 53 예시)
# 각 리전에 가중치 또는 지연 기반 라우팅 설정
aws route53 change-resource-record-sets --hosted-zone-id Z123 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "db-read.example.com",
        "Type": "CNAME",
        "Region": "ap-northeast-1",
        "SetIdentifier": "tokyo-read",
        "TTL": 60,
        "ResourceRecords": [{"Value": "db-replica-tokyo.internal"}]
      }
    }]
  }'
```

**장애 조치 (Failover)**

Active-Passive에서 Primary가 죽으면, Passive 중 하나를 새 Primary로 승격(promote)한다. 이때 문제가 있다.

- 복제 지연 때문에 새 Primary는 최신 데이터를 갖지 않을 수 있다
- 어느 Replica가 가장 최신인지 확인해야 한다
- 자동 승격 시 Split Brain(두 노드가 모두 Primary라고 생각) 가능성이 있다

```bash
# PostgreSQL 자동 Failover - Patroni 설정 예시
# patroni.yml
postgresql:
  use_pg_rewind: true
  parameters:
    synchronous_commit: "on"
    synchronous_standby_names: "ANY 1 (tokyo-replica, frankfurt-replica)"

# synchronous_standby_names: ANY 1은 두 Replica 중 하나만 동기 확인하면 됨
# 완전한 동기는 아니지만 최소 하나의 Replica에는 확실히 복제됨
```

### 접근법 2: 멀티-리전 Active-Active

모든 리전에서 읽기와 쓰기 모두 처리한다. 가장 낮은 지연을 제공하지만 가장 복잡하다.

```
서울(Active)  ←────────────→  도쿄(Active)
 Primary A                     Primary B
 서울 사용자 담당               일본 사용자 담당
     │                              │
     └─────────→ 양방향 복제 ←──────┘
                   (충돌 해결 필요)
```

**핵심 문제: 쓰기 충돌**

서울과 도쿄 양쪽에서 같은 레코드를 동시에 수정하면 충돌이 발생한다.

```
서울: UPDATE users SET balance=500 WHERE id=1  (원래 balance=1000)
도쿄: UPDATE users SET balance=800 WHERE id=1  (원래 balance=1000)

어느 쪽이 맞는가? 두 쓰기를 어떻게 해결할 것인가?
```

이를 해결하는 방법:

*지역성 기반 분리*: 각 사용자가 특정 리전에 "소속"되어 그 리전에서만 쓰기가 발생한다. 한국 사용자는 서울에서만 쓰고, 일본 사용자는 도쿄에서만 쓴다. 크로스-리전 쓰기는 발생하지 않는다.

```sql
-- 사용자 테이블에 home region 추가
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    home_region VARCHAR(20) NOT NULL,  -- 'kr', 'jp', 'us', 'eu'
    balance DECIMAL(10,2),
    ...
);

-- 애플리케이션에서 home_region != 현재 리전이면 해당 리전으로 프록시
```

*CRDT (Conflict-free Replicated Data Types)*: 충돌이 수학적으로 발생하지 않도록 데이터 구조를 설계한다. 예를 들어 카운터(좋아요 수)는 덧셈만 허용하면 충돌이 없다.

*Last-Write-Wins*: 타임스탬프가 최신인 쓰기가 이긴다. 단순하지만 데이터 손실이 발생한다.

**CockroachDB의 Multi-Region 지원**

CockroachDB는 멀티-리전을 기본 기능으로 지원한다. 테이블이나 행의 데이터 지역성을 SQL로 선언할 수 있다.

```sql
-- 데이터베이스에 리전 추가
ALTER DATABASE mydb ADD REGION "ap-northeast-1";  -- 서울
ALTER DATABASE mydb ADD REGION "ap-northeast-3";  -- 도쿄
ALTER DATABASE mydb ADD REGION "us-east-1";       -- 버지니아

-- 테이블의 생존 정책 설정
ALTER TABLE users SET LOCALITY REGIONAL BY ROW;
-- 각 행이 home_region에 해당하는 리전에 Raft Leader를 가짐
-- 한국 사용자 데이터는 서울에서 빠른 쓰기

ALTER TABLE global_config SET LOCALITY GLOBAL;
-- 모든 리전에서 빠른 읽기 (쓰기는 느릴 수 있음)
```

### 접근법 3: CAP 트레이드오프와 현실적 선택

글로벌 분산 시스템은 CAP 정리의 벽에 부딪힌다. 네트워크 파티션은 필연적이므로 Consistency와 Availability 중 하나를 희생해야 한다.

```
CAP 정리 실제 적용:

CP (일관성 우선): CockroachDB, Spanner
- 파티션 중 일부 노드는 요청 거부
- 금융 데이터에 적합

AP (가용성 우선): Cassandra, DynamoDB
- 파티션 중에도 응답 (단, 오래된 데이터일 수 있음)
- SNS 피드, 쇼핑 카트에 적합

실제로는 스펙트럼: "얼마나 일관성이 필요한가"
```

**Google Spanner의 접근법**

Spanner는 GPS와 원자시계를 사용하는 TrueTime API로 글로벌 트랜잭션 순서를 보장한다. 외부 일관성(External Consistency)을 전 지구적으로 제공한다. 이는 일반 회사가 구현하기 매우 어렵다.

AWS Aurora Global Database나 Azure Cosmos DB는 각자의 방식으로 글로벌 분산을 추상화한다.

### 데이터 지역성 고려: 컴플라이언스와 성능

글로벌 데이터베이스를 설계할 때 기술적 성능 외에도 법적 요건을 반드시 고려해야 한다.

**데이터 주권 (Data Sovereignty)**

- **EU GDPR**: 유럽 시민의 개인정보는 EU 밖으로 이전 시 엄격한 요건
- **중국**: 중국 내 서비스 데이터는 중국 서버에 저장 의무
- **러시아**: 러시아 시민 데이터는 러시아 내 서버에 저장
- **한국**: 개인정보보호법에 따른 국외 이전 고지 및 동의 요건

이는 성능 최적화와 충돌할 수 있다. 유럽 사용자 데이터는 EU 서버에 있어야 하므로, 글로벌 Active-Active에서 EU 데이터를 아시아 서버에 쓰면 안 된다.

```sql
-- 데이터 분류 예시
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    region VARCHAR(10) NOT NULL,  -- 'kr', 'eu', 'us'
    -- PII 데이터
    email VARCHAR(255),
    name VARCHAR(100),
    -- 비PII 데이터
    account_type VARCHAR(50),
    created_at TIMESTAMPTZ
) PARTITION BY LIST (region);

CREATE TABLE users_kr PARTITION OF users FOR VALUES IN ('kr');
CREATE TABLE users_eu PARTITION OF users FOR VALUES IN ('eu');
CREATE TABLE users_us PARTITION OF users FOR VALUES IN ('us');

-- eu 파티션은 EU 리전 테이블스페이스에만 저장
```

**지연 최적화를 위한 데이터 복제 전략**

읽기가 많고 쓰기가 적은 글로벌 참조 데이터(상품 카탈로그, 환율 정보, 설정 값)는 모든 리전에 복제해두는 것이 유리하다. 업데이트는 낮은 빈도로 Primary에 쓰고, 글로벌 캐시나 CDN 레이어에 배포한다.

```python
# 글로벌 읽기 전략 예시
class GlobalDataStore:
    def __init__(self, primary_db, regional_replicas, cache):
        self.primary = primary_db
        self.replicas = regional_replicas  # {'kr': ..., 'eu': ..., 'us': ...}
        self.cache = cache  # Redis 등

    def read(self, key: str, user_region: str, consistency: str = 'eventual'):
        if consistency == 'strong':
            # 강한 일관성 필요 → Primary에서 읽기
            return self.primary.get(key)

        # 캐시 확인
        cached = self.cache.get(key)
        if cached:
            return cached

        # 가장 가까운 Replica에서 읽기
        replica = self.replicas.get(user_region, self.primary)
        value = replica.get(key)
        self.cache.set(key, value, ttl=60)
        return value

    def write(self, key: str, value):
        # 항상 Primary에 쓰기
        self.primary.set(key, value)
        # 캐시 무효화
        self.cache.delete(key)
        # 비동기 복제는 Replica들이 자동으로 처리
```

### 글로벌 DB 설계의 안티패턴

**안티패턴 1: 모든 쿼리를 Primary에**

"복제 지연이 무섭다"는 이유로 모든 쿼리를 Primary에 보내는 팀이 있다. Read Replica를 추가했지만 사용하지 않는 것이다. 이는 Read Replica의 효과를 완전히 무력화한다.

**안티패턴 2: 너무 이른 샤딩**

단일 PostgreSQL 서버는 잘 튜닝되면 초당 수만 건의 트랜잭션을 처리한다. 수십만 DAU도 단일 DB로 충분한 경우가 많다. 조기에 샤딩하면 복잡도만 증가하고 실제 이익은 없다.

"지금 필요한 것 이상을 구축하지 말라"는 원칙을 DB 스케일링에서도 지켜야 한다.

**안티패턴 3: 크로스-샤드 트랜잭션 무시**

샤딩 설계 시 크로스-샤드 트랜잭션이 얼마나 발생하는지 사전에 분석하지 않으면, 샤딩 후 성능이 오히려 나빠질 수 있다. 자주 함께 쓰이는 테이블은 같은 샤드에 배치해야 한다.

**안티패턴 4: Replica에서의 장기 실행 쿼리**

분석 쿼리를 Read Replica에서 실행하는 것은 좋지만, 수십 분이 걸리는 쿼리는 Replica의 리소스를 점유해 복제 지연을 유발한다. 분석용으로는 별도의 분석 Replica나 데이터 웨어하우스(BigQuery, Redshift)를 사용해야 한다.

---

## 현실적인 DB 스케일링 로드맵

실제 프로덕션에서 DB 스케일링은 한 번에 이루어지지 않는다. 단계적으로 진행된다.

```
단계 0: 단일 DB
  └── 먼저 쿼리 최적화, 인덱스 추가, 커넥션 풀 튜닝

단계 1: Read Replica 추가
  └── 읽기 트래픽의 70-80%를 Replica로 분산

단계 2: 연결 풀링 강화 (PgBouncer, ProxySQL)
  └── 커넥션 오버헤드 제거

단계 3: 캐싱 레이어 도입 (Redis, Memcached)
  └── 반복 조회를 DB에 닿지 않게

단계 4: 기능 분리 (Functional Partitioning)
  └── 주문 DB, 사용자 DB, 상품 DB 분리

단계 5: 필요한 경우만 샤딩
  └── 단계 4가 불충분할 때만

단계 6: 글로벌 배포
  └── 다중 리전이 정말 필요할 때
```

DB 스케일링에서 가장 중요한 교훈은: **복잡성은 비용이다**.

샤딩과 글로벌 분산은 강력하지만, 그 대가로 운영 복잡도, 개발 제약, 높은 비용이 따른다. 각 단계에서 "정말 이게 지금 필요한가"를 물어야 한다. 대부분의 문제는 더 아래 단계에서 해결된다.

---

## 참고 자료

- Martin Kleppmann, *Designing Data-Intensive Applications*, O'Reilly Media (2017) — 분산 시스템과 DB 스케일링의 바이블. 복제, 파티셔닝, 트랜잭션을 깊이 있게 다룬다.
- Vitess 공식 문서, https://vitess.io/docs/ — Vitess 아키텍처와 운영 가이드.
- CockroachDB 공식 문서 "Architecture Overview", https://www.cockroachlabs.com/docs/stable/architecture/overview.html — Raft 기반 분산 트랜잭션 설계.
- Percona Blog, "MySQL Replication Best Practices" — MySQL 복제 운영 실전 경험.
- Google Spanner 논문, "Spanner: Google's Globally Distributed Database", OSDI 2012 — 글로벌 일관성 보장의 이론적 토대.
- AWS 문서, "Amazon Aurora Global Database" — 멀티-리전 Active-Passive 구현 사례.
