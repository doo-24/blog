---
title: "[데이터베이스] 2편 — 트랜잭션과 락: 동시성 제어의 깊이"
date: 2026-03-17T23:08:00+09:00
draft: false
tags: ["트랜잭션", "ACID", "MVCC", "락", "데이터베이스", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "ACID 각 속성의 실제 의미, MVCC 동작 원리(InnoDB undo log), 격리 수준별 현상, 레코드 락·갭 락·넥스트 키 락, 데드락 탐지와 해소, 낙관적 락과 비관적 락까지"
---

데이터베이스를 다루다 보면 "트랜잭션 안에 넣으면 된다"는 말을 쉽게 한다. 하지만 트랜잭션이 실제로 무엇을 보장하는지, 그 보장이 어떤 비용을 수반하는지 명확히 아는 개발자는 생각보다 드물다.

서비스 트래픽이 낮을 때는 문제가 없다가 동시 요청이 폭증하는 순간 데이터 정합성이 무너지거나, 데드락 로그가 쏟아지거나, 응답 지연이 갑자기 수십 배 늘어나는 경험을 해본 적이 있다면 — 이 글이 그 원인을 정확히 짚어줄 것이다.

ACID부터 MVCC, 락의 세 가지 종류, 데드락 해소, 그리고 낙관적·비관적 락의 실전 선택 기준까지, 동시성 제어의 핵심을 처음부터 끝까지 파헤친다.

---

## 1. ACID — 네 글자가 실제로 보장하는 것

ACID는 Atomicity(원자성), Consistency(일관성), Isolation(격리성), Durability(지속성)의 약자다. 교과서에서 한 줄씩 외웠을 텐데, 실제 시스템에서 각 속성이 어떻게 구현되는지를 이해해야 실전에서 활용할 수 있다.

### Atomicity — 부분 성공은 없다

원자성은 트랜잭션 내의 모든 연산이 전부 성공하거나 전부 실패해야 한다는 규칙이다. 은행 이체를 예로 들면, A 계좌에서 출금하고 B 계좌에 입금하는 두 연산은 하나의 단위다.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;
COMMIT;
```

첫 번째 UPDATE가 성공한 뒤 서버가 다운되면 어떻게 될까? InnoDB는 **redo log**와 **undo log**를 통해 이 상황을 처리한다. redo log는 커밋된 변경사항을 재적용하고, undo log는 미완료 트랜잭션의 변경을 되돌린다.

서버가 재시작될 때 InnoDB는 redo log를 보고 완료된 트랜잭션을 복구하고, undo log를 보고 미완료 트랜잭션을 롤백한다.

중요한 점은 원자성이 "실패 시 롤백"만을 의미하지 않는다는 것이다. 애플리케이션 코드에서 예외를 잡아 ROLLBACK을 명시적으로 호출하지 않으면, 커넥션이 닫힐 때까지 트랜잭션이 열린 채로 남아 락을 계속 보유한다. 이것이 커넥션 풀 고갈의 주요 원인 중 하나다.

```java
// 안티패턴: 예외 시 롤백을 보장하지 않음
connection.setAutoCommit(false);
try {
    // 쿼리 실행
    connection.commit();
} catch (Exception e) {
    // rollback 누락!
    throw e;
}

// 올바른 패턴
connection.setAutoCommit(false);
try {
    // 쿼리 실행
    connection.commit();
} catch (Exception e) {
    connection.rollback(); // 반드시 명시적 롤백
    throw e;
}
```

### Consistency — 데이터베이스가 아닌 애플리케이션의 책임

일관성은 트랜잭션 전후로 데이터베이스가 유효한 상태여야 한다는 규칙이다. 그런데 이것은 사실 데이터베이스 엔진이 온전히 보장할 수 없다.

스키마 제약(NOT NULL, UNIQUE, FOREIGN KEY, CHECK)은 DB가 강제할 수 있지만, 비즈니스 규칙("잔액은 0 미만이 될 수 없다")은 애플리케이션 코드가 책임져야 한다.

```sql
-- DB 레벨 제약: 잔액이 음수가 되면 거부
ALTER TABLE accounts ADD CONSTRAINT chk_balance CHECK (balance >= 0);
```

이 제약이 없다면 잘못된 이체가 음수 잔액을 만들어도 DB는 조용히 허용한다. Consistency는 나머지 세 속성(AID)이 올바르게 작동할 때 비로소 의미를 갖는다.

### Isolation — 동시성의 핵심이자 가장 복잡한 속성

격리성은 동시에 실행되는 트랜잭션들이 서로 간섭하지 않아야 한다는 규칙이다. 완전한 격리는 직렬 실행(serializability)을 의미하지만, 그러면 성능이 극도로 저하된다. 트랜잭션을 하나씩 순서대로 처리하면 동시성이 사라지기 때문이다.

그래서 SQL 표준은 네 가지 격리 수준을 정의하고, 각 수준에서 허용되는 이상 현상(anomaly)을 명시한다. 이것이 3절에서 자세히 다룰 내용이다.

### Durability — 커밋은 영원하다

지속성은 커밋된 트랜잭션은 시스템 장애가 발생해도 반드시 유지된다는 규칙이다. InnoDB는 이를 위해 **WAL(Write-Ahead Logging)** 방식을 사용한다. 데이터를 실제 디스크의 데이터 파일에 쓰기 전에 먼저 redo log에 기록한다. `innodb_flush_log_at_trx_commit` 설정이 이 동작을 제어한다.

```
innodb_flush_log_at_trx_commit = 1  (기본값)
  → 커밋마다 redo log를 디스크에 fsync. 완전한 내구성 보장.

innodb_flush_log_at_trx_commit = 2
  → 커밋마다 OS 버퍼에 쓰기. 초당 1회 fsync.
  → OS 크래시 시 최대 1초치 트랜잭션 유실 가능.

innodb_flush_log_at_trx_commit = 0
  → 초당 1회 버퍼에 쓰고 fsync.
  → MySQL 크래시 시에도 데이터 유실 가능.
```

성능을 위해 2나 0으로 설정하는 경우가 있는데, 이는 내구성 보장을 포기하는 것임을 명확히 인지해야 한다. 금융 시스템에서 1을 사용하는 이유가 여기 있다.

---

## 2. MVCC — 읽기와 쓰기가 서로 블로킹하지 않는 비밀

### MVCC란 무엇인가

MVCC(Multi-Version Concurrency Control)는 데이터의 여러 버전을 동시에 유지함으로써 읽기와 쓰기가 서로를 블로킹하지 않게 하는 기법이다. InnoDB는 MVCC를 통해 일반 SELECT가 어떤 락도 획득하지 않고도 일관된 스냅샷을 읽을 수 있게 한다.

이것이 왜 중요한가? 전통적인 락 기반 읽기에서는 다른 트랜잭션이 쓰기 락을 보유하고 있으면 읽기가 차단된다. MVCC에서는 읽기 트랜잭션이 과거 버전을 읽으므로 쓰기 트랜잭션과 충돌하지 않는다.

### InnoDB의 undo log와 버전 체인

InnoDB의 모든 레코드에는 두 개의 숨겨진 컬럼이 있다.

```
┌─────────────────────────────────────────────────┐
│  실제 데이터 컬럼들                               │
├─────────────────────────────────────────────────┤
│  DB_TRX_ID  : 이 레코드를 마지막으로 변경한       │
│               트랜잭션 ID (6 bytes)              │
│  DB_ROLL_PTR: undo log 레코드를 가리키는          │
│               포인터 (7 bytes)                  │
└─────────────────────────────────────────────────┘
```

트랜잭션이 레코드를 변경하면 변경 전 이미지를 undo log에 기록하고, 레코드의 DB_ROLL_PTR이 그 undo log를 가리키도록 업데이트한다. undo log에는 다시 이전 버전을 가리키는 포인터가 있어서 **버전 체인(version chain)**을 형성한다.

```
현재 레코드 (TRX_ID=100, balance=90000)
    │
    └→ undo log v1 (TRX_ID=90, balance=100000)
           │
           └→ undo log v2 (TRX_ID=80, balance=95000)
                  │
                  └→ (더 이전 버전...)
```

읽기 트랜잭션은 자신의 **ReadView**를 기준으로 버전 체인을 따라가면서 자신이 볼 수 있는 가장 최신 버전을 선택한다.

### ReadView — 스냅샷의 실체

트랜잭션이 시작되면(또는 첫 번째 읽기 시점에) InnoDB는 ReadView를 생성한다. ReadView는 다음 정보를 포함한다.

```
ReadView {
  creator_trx_id: 현재 트랜잭션 ID
  m_ids: ReadView 생성 시점에 활성화된 트랜잭션 ID 목록
  min_trx_id: m_ids 중 최솟값
  max_trx_id: 다음에 할당될 트랜잭션 ID (현재 최댓값 + 1)
}
```

레코드를 읽을 때 가시성(visibility) 판단 규칙:

```
레코드의 DB_TRX_ID를 trx_id라 하면:

1. trx_id == creator_trx_id → 자신이 변경한 데이터 → 볼 수 있음
2. trx_id < min_trx_id → ReadView 생성 전 커밋된 트랜잭션 → 볼 수 있음
3. trx_id >= max_trx_id → ReadView 생성 후 시작된 트랜잭션 → 볼 수 없음
4. min_trx_id <= trx_id < max_trx_id:
   - trx_id가 m_ids에 있음 → ReadView 생성 당시 미완료 트랜잭션 → 볼 수 없음
   - trx_id가 m_ids에 없음 → ReadView 생성 전 커밋된 트랜잭션 → 볼 수 있음
```

볼 수 없으면 DB_ROLL_PTR을 따라 이전 버전으로 가서 다시 판단한다. 이 과정을 볼 수 있는 버전이 나올 때까지 반복한다.

### 구체적인 시나리오로 이해하기

```
Time  TrxA(id=10)              TrxB(id=11)               TrxC(id=12)
 1    BEGIN
 2    SELECT balance            BEGIN
      → 100000 (초기값)
 3                              UPDATE balance = 90000
                                COMMIT
 4    SELECT balance                                       BEGIN
      → ?                                                 SELECT balance
                                                          → ?
```

TrxA (REPEATABLE READ 기준):
- Time 1에 ReadView 생성: m_ids=[10], min=10, max=11
- Time 4에 SELECT: TrxB(id=11)는 max_trx_id(11)이므로 "볼 수 없음"에 해당 → undo log v1 읽기 → **100000** 반환

TrxC (REPEATABLE READ 기준):
- Time 4에 ReadView 생성: m_ids=[12], min=12, max=13
- TrxB(id=11) < min_trx_id(12) → 이미 커밋됨 → 현재 레코드 그대로 읽기 → **90000** 반환

같은 값(90000)을 읽는다고 해도 TrxA는 커밋 전 값을 보고 TrxC는 커밋 후 값을 본다. 이것이 MVCC의 힘이다. 락 없이 각 트랜잭션이 자신의 일관된 스냅샷을 본다.

### undo log 정리와 purge

undo log는 무한정 쌓이지 않는다. InnoDB의 purge 스레드가 주기적으로 더 이상 어떤 ReadView에서도 참조되지 않는 undo log를 삭제한다. 그런데 여기서 흔한 문제가 발생한다.

```sql
-- 위험한 패턴: 오랫동안 열려 있는 트랜잭션
BEGIN;
SELECT * FROM large_table; -- ReadView 생성
-- ... 애플리케이션에서 긴 작업 ...
-- (트랜잭션을 닫지 않음)
```

이 트랜잭션이 살아있는 한, 그 ReadView보다 오래된 undo log는 삭제할 수 없다. 결과적으로 undo log가 무한히 증가하고, 버전 체인이 길어져 모든 읽기 성능이 저하된다.

`information_schema.innodb_trx`에서 `trx_started` 컬럼으로 오래된 트랜잭션을 모니터링하는 이유다.

---

## 3. 격리 수준 — 이상 현상과 트레이드오프

### SQL 표준의 네 가지 격리 수준

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 없음 | 발생 | 발생 |
| REPEATABLE READ | 없음 | 없음 | 발생 (표준) |
| SERIALIZABLE | 없음 | 없음 | 없음 |

InnoDB의 REPEATABLE READ는 표준과 달리 MVCC와 넥스트 키 락을 통해 팬텀 리드 대부분을 방지한다. 하지만 "대부분"이지 "완전히"는 아니다.

### Dirty Read — 커밋되지 않은 데이터를 읽다

Dirty Read는 한 트랜잭션이 다른 트랜잭션에서 아직 커밋하지 않은 변경 사항을 읽는 현상이다.

```
TrxA (READ UNCOMMITTED)         TrxB
BEGIN                           BEGIN
                                UPDATE balance = 0  ← 롤백될 수도 있음
SELECT balance → 0
                                ROLLBACK
-- TrxA는 존재한 적 없는 값 0을 읽었다
```

READ UNCOMMITTED는 거의 사용해서는 안 된다. 실제로 존재한 적 없는 데이터를 기반으로 비즈니스 로직을 실행하는 것은 심각한 버그로 이어진다.

### Non-Repeatable Read — 같은 쿼리가 다른 결과를

Non-Repeatable Read는 같은 트랜잭션 내에서 같은 행을 두 번 읽었을 때 다른 결과가 나오는 현상이다.

```
TrxA (READ COMMITTED)           TrxB
BEGIN
SELECT balance → 100000         BEGIN
                                UPDATE balance = 90000
                                COMMIT
SELECT balance → 90000  ← 이전 읽기와 다름!
```

READ COMMITTED에서는 SELECT마다 새로운 ReadView를 생성한다. 그래서 TrxB가 커밋한 후 TrxA가 다시 읽으면 변경된 값을 본다.

이것이 문제가 되는 실전 시나리오: 재고 시스템에서 첫 번째 SELECT로 재고를 확인하고 비즈니스 로직을 처리한 뒤 두 번째 SELECT로 다시 확인하면 그 사이에 재고가 변해 있을 수 있다. 단순 통계 쿼리라면 허용 가능하지만, 비즈니스 결정에 영향을 주는 경우라면 위험하다.

### Phantom Read — 없던 행이 나타나다

Phantom Read는 같은 조건의 SELECT를 두 번 실행했을 때 두 번째 실행에서 첫 번째에 없던 행이 나타나는 현상이다.

```
TrxA (REPEATABLE READ, 표준)    TrxB
BEGIN
SELECT COUNT(*) FROM orders
WHERE status = 'pending'
→ 5건                           BEGIN
                                INSERT INTO orders (status) VALUES ('pending')
                                COMMIT
SELECT COUNT(*) FROM orders
WHERE status = 'pending'
→ 6건  ← 팬텀!
```

InnoDB는 어떻게 이를 방지하는가? 바로 **넥스트 키 락(Next-Key Lock)**을 통해서다. 이것은 4절에서 자세히 설명한다.

### 격리 수준 변경과 실전 선택

```sql
-- 세션 레벨 변경
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 트랜잭션 단위 변경
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
-- ...
```

실전에서 격리 수준 선택 기준:
- **REPEATABLE READ** (MySQL InnoDB 기본값): 대부분의 OLTP 워크로드에 적합
- **READ COMMITTED**: OLAP 쿼리가 많거나 갭 락으로 인한 데드락이 빈번할 때. 단, Non-Repeatable Read를 애플리케이션에서 허용해야 함
- **SERIALIZABLE**: 금융 결제처럼 절대적 정확성이 필요한 경우. 성능 비용이 크다

---

## 4. 락의 종류와 메커니즘

### 레코드 락 (Record Lock)

레코드 락은 인덱스 레코드(실제 행이 아닌 인덱스 엔트리) 하나에 걸리는 락이다. InnoDB의 모든 락은 인덱스를 통해 걸린다는 점이 중요하다.

```sql
-- id = 1인 레코드에 레코드 락
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
```

인덱스가 없는 컬럼으로 WHERE 조건을 주면 어떻게 될까?

```sql
-- 인덱스 없는 컬럼으로 조건
SELECT * FROM accounts WHERE name = 'Alice' FOR UPDATE;
```

InnoDB는 인덱스가 없으면 테이블 풀 스캔을 하면서 모든 레코드에 락을 건다. 실질적인 테이블 락이 되어버린다. 이것이 `FOR UPDATE` 쿼리에 반드시 인덱스가 필요한 이유다.

락에는 두 가지 모드가 있다.
- **공유 락(S Lock, Shared Lock)**: 읽기 락. 여러 트랜잭션이 동시에 S락을 획득할 수 있다.
- **배타 락(X Lock, Exclusive Lock)**: 쓰기 락. 하나의 트랜잭션만 X락을 가질 수 있고, X락이 있으면 S락도 불가능하다.

```sql
-- S락 획득
SELECT * FROM accounts WHERE id = 1 LOCK IN SHARE MODE;  -- MySQL 5.x
SELECT * FROM accounts WHERE id = 1 FOR SHARE;           -- MySQL 8.0+

-- X락 획득
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = 90000 WHERE id = 1;  -- 자동으로 X락
```

### 갭 락 (Gap Lock)

갭 락은 인덱스 레코드 사이의 "갭(간격)"에 걸리는 락이다. 실제 존재하는 레코드가 아니라, 그 사이의 빈 공간에 대한 락이다.

```
인덱스: ... 10, 20, 30 ...

갭 락 영역:
  (-∞, 10): 10 이전 갭
  (10, 20): 10과 20 사이 갭
  (20, 30): 20과 30 사이 갭
  (30, +∞): 30 이후 갭
```

갭 락은 왜 필요한가? 팬텀 리드를 방지하기 위해서다. 갭 락이 없다면 다른 트랜잭션이 조회 범위 안에 새 행을 삽입할 수 있고, 같은 SELECT를 반복할 때마다 결과 행 수가 달라지는 팬텀 현상이 발생한다.

```sql
-- TrxA: id가 10~20 사이 데이터 잠금 (REPEATABLE READ)
SELECT * FROM t WHERE id BETWEEN 10 AND 20 FOR UPDATE;
-- (10, 20) 갭에 갭 락이 걸린다

-- TrxB: 이 갭에 새 레코드 삽입 시도
INSERT INTO t (id) VALUES (15);  -- 갭 락이 있으므로 차단됨
```

갭 락의 중요한 특성: **갭 락끼리는 충돌하지 않는다.** 두 트랜잭션이 같은 갭에 갭 락을 걸어도 서로 블로킹하지 않는다. 갭 락은 INSERT를 막기 위한 것이지, 다른 트랜잭션의 갭 락을 막기 위한 것이 아니다.

READ COMMITTED에서는 갭 락이 비활성화된다. 그래서 READ COMMITTED로 바꾸면 INSERT 관련 데드락이 줄어드는 경우가 많다.

### 넥스트 키 락 (Next-Key Lock)

넥스트 키 락은 레코드 락과 그 레코드 앞의 갭 락을 합친 것이다. InnoDB의 REPEATABLE READ에서 기본으로 사용하는 락 형태다.

```
넥스트 키 락: (-∞, 10], (10, 20], (20, 30], (30, +∞)
              ↑ 닫힌 구간: 레코드 자체도 포함
```

예를 들어 `WHERE id = 20`으로 FOR UPDATE를 하면, 레코드 20에 레코드 락, 그리고 (10, 20] 넥스트 키 락이 걸린다.

```sql
-- 인덱스 값: 10, 20, 30이 존재한다고 가정
BEGIN;
SELECT * FROM t WHERE id = 20 FOR UPDATE;
-- 레코드 락: id=20
-- 갭 락: (10, 20)
-- → 사실상 넥스트 키 락: (10, 20]
```

범위 검색의 경우:

```sql
SELECT * FROM t WHERE id > 15 FOR UPDATE;
-- id=20, id=30 레코드 락
-- (15, 20], (20, 30], (30, +∞) 넥스트 키 락
```

이 때문에 범위 UPDATE/DELETE는 생각보다 훨씬 넓은 범위를 잠근다. 이것을 이해하지 못하면 불필요하게 넓은 락으로 인한 성능 저하와 데드락을 경험하게 된다.

### 인텐션 락 (Intention Lock)

인텐션 락은 테이블 레벨 락으로, 이 테이블의 어떤 행에 락이 걸릴 예정임을 선언하는 락이다. IS(Intention Shared)와 IX(Intention Exclusive)가 있다.

```
트랜잭션이 행 레벨 S락을 걸기 전 → 테이블에 IS 락
트랜잭션이 행 레벨 X락을 걸기 전 → 테이블에 IX 락
```

인텐션 락의 목적은 테이블 레벨 락과 행 레벨 락의 효율적인 공존이다. `LOCK TABLE ... WRITE`가 테이블 전체를 잠그려면 모든 행을 검사하지 않고도 IX 락의 존재로 충돌을 감지할 수 있다.

### 데드락 탐지와 해소

데드락은 두 트랜잭션이 서로 상대방이 보유한 락을 기다리는 상황이다. 두 사람이 각각 열쇠를 하나씩 쥔 채 상대방 열쇠를 기다리는 것과 같다. 누구도 먼저 포기하지 않으면 영원히 진행이 불가능하다.

```
TrxA                            TrxB
BEGIN                           BEGIN
SELECT * FROM A WHERE id=1      SELECT * FROM B WHERE id=1
FOR UPDATE;  ← A 레코드 잠금    FOR UPDATE;  ← B 레코드 잠금

SELECT * FROM B WHERE id=1      SELECT * FROM A WHERE id=1
FOR UPDATE;  ← B 기다림 →      FOR UPDATE;  ← A 기다림
                ← 데드락! →
```

InnoDB는 데드락을 **wait-for graph**로 탐지한다. 각 트랜잭션을 노드로, 락 대기를 엣지로 하는 그래프에서 사이클이 발생하면 데드락이다. InnoDB는 이 사이클을 감지하면 트랜잭션 중 하나를 victim으로 선택해 롤백한다(보통 더 적은 undo log를 가진 트랜잭션).

```sql
-- 데드락 발생 시 에러
ERROR 1213 (40001): Deadlock found when trying to get lock;
try restarting transaction
```

애플리케이션에서는 이 에러를 받으면 트랜잭션을 재시도해야 한다.

```java
int maxRetries = 3;
for (int i = 0; i < maxRetries; i++) {
    try {
        executeTransaction();
        break;
    } catch (DeadlockException e) {
        if (i == maxRetries - 1) throw e;
        Thread.sleep(exponentialBackoff(i));
    }
}
```

**데드락을 줄이는 전략:**

1. **항상 같은 순서로 락 획득**: 여러 테이블에 접근할 때 순서를 통일한다. TrxA와 TrxB가 둘 다 A→B 순으로 접근하면 데드락이 발생하지 않는다.

2. **트랜잭션을 짧게 유지**: 트랜잭션이 짧을수록 락 보유 시간이 줄고 충돌 확률이 낮아진다.

3. **인덱스를 제대로 사용**: 인덱스 없는 조건은 불필요하게 넓은 범위를 잠근다.

4. **격리 수준 조정**: READ COMMITTED로 낮추면 갭 락이 비활성화되어 INSERT 관련 데드락이 줄어든다.

5. **`innodb_deadlock_detect` 설정 확인**: 기본 ON이지만, 극단적인 고부하 환경에서는 탐지 비용이 클 수 있다. 이 경우 `innodb_lock_wait_timeout`으로 타임아웃을 설정하는 방식으로 대체하기도 한다.

**데드락 로그 분석:**

```sql
SHOW ENGINE INNODB STATUS;
```

출력에서 `LATEST DETECTED DEADLOCK` 섹션을 찾아 어떤 트랜잭션이 어떤 락을 기다리고 있었는지 확인한다.

```
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 0 sec starting index read
...
*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 67 page no 3 n bits 72 index PRIMARY
  of table `mydb`.`accounts` trx id 12345 lock_mode X locks rec but not gap
...
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
...
*** (2) TRANSACTION:
...
*** WE ROLL BACK TRANSACTION (2)
```

이 로그를 통해 데드락이 발생한 테이블, 인덱스, 락 모드를 정확히 파악하고 코드를 수정할 수 있다.

---

## 5. 낙관적 락과 비관적 락 — 언제 무엇을 선택할까

### 비관적 락 (Pessimistic Lock)

비관적 락은 "충돌이 발생할 것이다"고 가정하고 데이터에 접근하기 전에 먼저 락을 획득하는 방식이다. `SELECT FOR UPDATE`가 대표적이다.

```sql
BEGIN;
-- 재고를 먼저 잠근다
SELECT stock FROM products WHERE id = 100 FOR UPDATE;

-- 잠금 획득 후 비즈니스 로직
-- (다른 트랜잭션은 여기서 블로킹됨)
UPDATE products SET stock = stock - 1 WHERE id = 100 AND stock > 0;
COMMIT;
```

비관적 락의 장점:
- 충돌이 발생하면 즉시 감지되고 대기한다
- 데이터 정합성이 강하게 보장된다

비관적 락의 단점:
- 락 획득부터 커밋까지 다른 트랜잭션이 블로킹된다
- 처리량이 낮아진다
- 데드락 위험이 있다
- 락을 잡은 채 외부 서비스 호출을 하면 재앙이 된다

```java
// 치명적인 안티패턴
BEGIN; // 락 획득
SELECT ... FOR UPDATE;
callExternalPaymentAPI(); // 외부 API 응답 대기 (수초~수십초!)
UPDATE ...
COMMIT; // 이 사이 다른 모든 요청이 블로킹
```

외부 API 호출, 파일 I/O, 장시간 계산 등을 트랜잭션 내에서 락을 보유한 채 하는 것은 절대로 피해야 한다.

### SKIP LOCKED와 NOWAIT

MySQL 8.0부터 지원하는 유용한 옵션이다.

```sql
-- 락이 걸린 행은 건너뛰고 잠글 수 있는 행만 반환
SELECT * FROM jobs WHERE status = 'pending' LIMIT 1 FOR UPDATE SKIP LOCKED;

-- 락 획득 불가 시 즉시 에러 반환 (대기 안 함)
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
```

`SKIP LOCKED`는 작업 큐 구현에 매우 유용하다. 여러 워커가 pending 작업을 동시에 가져갈 때 서로 블로킹하지 않고 각자 다른 작업을 가져간다.

### 낙관적 락 (Optimistic Lock)

낙관적 락은 "충돌이 드물다"고 가정하고 락 없이 데이터를 읽고 수정한 뒤, 커밋 시점에 충돌 여부를 확인하는 방식이다. 일반적으로 **버전 컬럼(version column)**을 사용한다.

```sql
-- 테이블 설계
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100),
    stock INT,
    version INT DEFAULT 0  -- 버전 컬럼
);

-- 1. 읽기 (락 없음)
SELECT id, stock, version FROM products WHERE id = 100;
-- 결과: id=100, stock=10, version=5

-- 2. 비즈니스 로직 처리 (애플리케이션에서)

-- 3. 수정 시 버전 검사
UPDATE products
SET stock = 9, version = version + 1
WHERE id = 100 AND version = 5;  -- 읽을 때의 버전으로 조건

-- 4. 영향받은 행 수 확인
-- affected_rows == 1 → 성공 (충돌 없음)
-- affected_rows == 0 → 충돌! 다른 트랜잭션이 먼저 수정함
```

버전 대신 타임스탬프를 쓰기도 한다.

```sql
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    stock INT,
    updated_at TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6)
);

UPDATE products
SET stock = 9
WHERE id = 100 AND updated_at = '2026-03-17 12:00:00.123456';
```

타임스탬프는 밀리초 미만의 동시 업데이트에서 충돌을 놓칠 수 있으므로 정수 버전 컬럼이 더 안전하다.

### JPA/Hibernate에서의 낙관적 락

JPA는 낙관적 락을 기본 지원한다.

```java
@Entity
public class Product {
    @Id
    private Long id;

    private int stock;

    @Version  // 이 어노테이션 하나로 낙관적 락 활성화
    private int version;
}

// 사용 시
Product product = em.find(Product.class, 100L);
product.setStock(product.getStock() - 1);
// em.flush() 시점에 UPDATE ... WHERE id=? AND version=? 실행
// 충돌 시 OptimisticLockException 발생
```

### 낙관적 락 vs 비관적 락 선택 기준

```
충돌 빈도가 낮다 (읽기 비중이 높거나 같은 데이터 수정이 드문 경우)
  → 낙관적 락

충돌 빈도가 높다 (핫 데이터, 재고/좌석/쿠폰 등 경쟁이 심한 리소스)
  → 비관적 락

처리 시간이 길다 (외부 API 연동 등)
  → 낙관적 락 (비관적 락으로 긴 시간 락을 잡으면 안 됨)

실패 재시도 비용이 크다 (복잡한 계산, 여러 단계의 작업)
  → 비관적 락 (재시도 시 처음부터 다시 해야 하면 낭비)

분산 환경 (다른 DB 인스턴스, MSA)
  → 낙관적 락 또는 분산 락 (Redis 등)
```

### 분산 환경에서의 낙관적 락

MSA 환경에서 서로 다른 서비스가 같은 데이터를 수정하는 경우, DB 레벨 락은 한계가 있다. 이때 Redis를 활용한 분산 락을 함께 사용한다.

```python
# Redis를 이용한 분산 락 (Redlock 패턴)
import redis
import uuid

def acquire_lock(conn, lock_name, acquire_timeout=10, lock_timeout=10):
    identifier = str(uuid.uuid4())
    lock_name = f'lock:{lock_name}'
    lock_timeout = int(lock_timeout)

    end = time.time() + acquire_timeout
    while time.time() < end:
        if conn.set(lock_name, identifier, nx=True, ex=lock_timeout):
            return identifier
        time.sleep(0.001)
    return None

def release_lock(conn, lock_name, identifier):
    lock_name = f'lock:{lock_name}'
    pipe = conn.pipeline(True)
    while True:
        try:
            pipe.watch(lock_name)
            if pipe.get(lock_name) == identifier.encode():
                pipe.multi()
                pipe.delete(lock_name)
                pipe.execute()
                return True
            pipe.unwatch()
            break
        except redis.WatchError:
            pass
    return False
```

---

## 6. 실전 함정과 체크리스트

### 락 대기 타임아웃

```sql
-- 기본값 확인
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';  -- 기본 50초

-- 세션 레벨 변경
SET SESSION innodb_lock_wait_timeout = 5;
```

락 대기가 50초 지속된다는 것은 그 시간 동안 커넥션이 묶인다는 뜻이다. 커넥션 풀이 20개라면 20개 요청이 동시에 락 대기에 들어가면 서비스가 마비된다.

타임아웃을 적절히 짧게 설정하고 애플리케이션에서 `LockWaitTimeoutException`을 처리해야 한다.

### 묵시적 트랜잭션과 autocommit

```sql
-- autocommit이 ON이면 각 문장이 자동으로 트랜잭션
SET autocommit = 1;  -- 기본값

UPDATE accounts SET balance = 90000 WHERE id = 1;
-- 즉시 커밋됨

-- autocommit이 OFF면 명시적 COMMIT 필요
SET autocommit = 0;
UPDATE accounts SET balance = 90000 WHERE id = 1;
-- COMMIT 또는 ROLLBACK을 안 하면 트랜잭션이 계속 열려있음
```

ORM이나 커넥션 풀을 사용할 때 autocommit 설정을 확인하는 것이 중요하다. 예상치 못하게 트랜잭션이 열린 채로 있으면 락이 유지되고 undo log가 쌓인다.

### INFORMATION_SCHEMA로 락 현황 모니터링

```sql
-- 현재 실행 중인 트랜잭션
SELECT trx_id, trx_started, trx_state, trx_tables_locked, trx_rows_locked
FROM information_schema.innodb_trx
ORDER BY trx_started;

-- 락 대기 현황
SELECT
    r.trx_id waiting_trx_id,
    r.trx_mysql_thread_id waiting_thread,
    r.trx_query waiting_query,
    b.trx_id blocking_trx_id,
    b.trx_mysql_thread_id blocking_thread,
    b.trx_query blocking_query
FROM information_schema.innodb_lock_waits w
INNER JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
INNER JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

MySQL 8.0에서는 `performance_schema.data_locks`와 `data_lock_waits`가 더 상세한 정보를 제공한다.

```sql
-- MySQL 8.0+
SELECT
    dl.ENGINE_TRANSACTION_ID,
    dl.OBJECT_SCHEMA,
    dl.OBJECT_NAME,
    dl.LOCK_TYPE,
    dl.LOCK_MODE,
    dl.LOCK_STATUS
FROM performance_schema.data_locks dl;
```

---

## 정리

트랜잭션과 락은 데이터베이스의 가장 핵심적인 메커니즘이다.

ACID가 교과서적 정의에 그치지 않고 실제 시스템에서 어떻게 구현되는지, MVCC가 왜 읽기 성능을 희생하지 않으면서도 일관성을 유지할 수 있는지, 각 격리 수준에서 어떤 이상 현상이 허용되는지를 이해하면 장애 상황에서 훨씬 빠르게 원인을 파악할 수 있다.

레코드 락, 갭 락, 넥스트 키 락의 범위를 정확히 알아야 불필요하게 넓은 락으로 인한 성능 저하와 데드락을 피할 수 있다. 낙관적 락과 비관적 락은 기술적 우열의 문제가 아니라 충돌 빈도와 처리 비용에 따른 선택의 문제다.

다음 편에서는 인덱스의 내부 구조(B+Tree)와 쿼리 실행 계획 최적화를 다룬다.

---

## 참고 자료

1. **High Performance MySQL, 4th Edition** — Silvia Botros, Jeremy Tinley (O'Reilly, 2022). InnoDB 내부 구조와 실전 최적화의 가장 권위 있는 레퍼런스.
2. **MySQL Internals Manual** — [dev.mysql.com/doc/internals/en](https://dev.mysql.com/doc/internals/en/). InnoDB 스토리지 엔진의 공식 내부 문서.
3. **Designing Data-Intensive Applications** — Martin Kleppmann (O'Reilly, 2017). 트랜잭션 격리와 분산 시스템 일관성을 가장 명확하게 설명한 책.
4. **A Critique of ANSI SQL Isolation Levels** — Hal Berenson et al. (SIGMOD 1995). SQL 표준 격리 수준의 불완전함을 분석한 고전 논문. Snapshot Isolation 개념 도입.
5. **MySQL 8.0 Reference Manual: InnoDB Locking** — [dev.mysql.com/doc/refman/8.0/en/innodb-locking.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html). 레코드 락, 갭 락, 넥스트 키 락의 공식 설명.
6. **Use The Index, Luke** — [use-the-index-luke.com](https://use-the-index-luke.com). 인덱스와 락의 관계를 실용적으로 설명하는 무료 온라인 가이드.
