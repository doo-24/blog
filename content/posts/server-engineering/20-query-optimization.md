---
title: "[데이터베이스] 3편 — 쿼리 최적화: 느린 쿼리를 빠르게"
date: 2026-03-17T23:07:00+09:00
draft: false
tags: ["쿼리 최적화", "EXPLAIN", "실행 계획", "JOIN", "데이터베이스", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "실행 계획 완전 분석(type, key, rows, Extra), 풀스캔 방지 전략, JOIN 전략(Nested Loop, Hash Join, Sort Merge), 서브쿼리 vs JOIN 성능 비교, 슬로우 쿼리 로그와 pt-query-digest 활용까지"
---

프로덕션에서 장애가 발생했다. 응답 시간이 갑자기 10배로 뛰었다.

애플리케이션 로그를 뒤지고, 서버 CPU를 확인하고, 네트워크를 점검해도 이상이 없다. 마지막으로 데이터베이스 슬로우 쿼리 로그를 열어보니 한 쿼리가 8초씩 걸리고 있었다.

데이터가 어제까지는 100만 건이었는데 오늘 배치 작업 후 1,000만 건이 됐고, 풀스캔 쿼리가 그제야 발목을 잡은 것이다.

이런 상황을 미연에 방지하는 것, 그리고 발생했을 때 신속하게 해결하는 것이 쿼리 최적화의 핵심이다. 이 글에서는 MySQL 옵티마이저가 실행 계획을 어떻게 수립하는지부터, JOIN 전략의 내부 동작, 서브쿼리와 EXISTS의 성능 차이, 그리고 슬로우 쿼리를 운영 환경에서 추적하는 방법까지 실전 깊이로 다룬다.

---

## 1. 실행 계획 완전 분석

MySQL에서 쿼리 앞에 `EXPLAIN`을 붙이면 옵티마이저가 해당 쿼리를 어떻게 실행할지 계획을 보여준다. 이 계획을 제대로 읽을 수 있으면 대부분의 성능 문제를 사전에 감지하거나 사후에 진단할 수 있다.

```sql
EXPLAIN SELECT u.name, COUNT(o.id) AS order_count
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id;
```

결과는 여러 행으로 나오는데, 각 행이 하나의 테이블 접근 단계를 나타낸다. 컬럼 하나하나가 옵티마이저의 의도를 담고 있다.

### 1.1 `type` 컬럼 — 접근 방식

`type`은 해당 테이블에 어떻게 접근하는지를 나타낸다. 성능 순서는 다음과 같다:

```
system > const > eq_ref > ref > range > index > ALL
```

| type | 의미 | 언제 등장하는가 |
|------|------|----------------|
| `system` | 테이블에 행이 1개뿐 | 시스템 테이블 |
| `const` | PK/Unique Key로 단 1행 접근 | `WHERE id = 5` |
| `eq_ref` | JOIN에서 PK/Unique Key로 정확히 1행 | `ON a.id = b.id` |
| `ref` | 비유니크 인덱스로 여러 행 매칭 | `WHERE user_id = ?` |
| `range` | 인덱스 범위 스캔 | `WHERE created_at > ?` |
| `index` | 인덱스 풀스캔 (데이터 파일 안 읽음) | `SELECT id FROM t` |
| `ALL` | 테이블 풀스캔 | 인덱스 없거나 미사용 |

`ALL`이 나오는 순간 경계해야 한다. 1만 건까지는 티가 안 나지만 100만 건부터는 초 단위 지연이 발생한다. `index`도 안심하면 안 된다 — 인덱스 전체를 읽는 것이지, 효율적인 접근이 아니다.

**실전 예시:**

```sql
-- type: ALL (풀스캔) — 위험
EXPLAIN SELECT * FROM orders WHERE YEAR(created_at) = 2025;

-- type: range (인덱스 범위 스캔) — 안전
EXPLAIN SELECT * FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01';
```

첫 번째 쿼리는 함수를 컬럼에 적용해 인덱스를 못 쓴다. 두 번째는 범위 조건으로 인덱스를 활용한다.

### 1.2 `key` 컬럼 — 실제 사용된 인덱스

`possible_keys`는 옵티마이저가 고려한 후보 인덱스들이고, `key`는 실제로 선택한 인덱스다. `key`가 `NULL`이면 인덱스를 전혀 사용하지 않는다는 뜻이다.

```sql
-- possible_keys: idx_user_id, idx_created_at
-- key: idx_user_id  (옵티마이저가 idx_user_id 선택)
EXPLAIN SELECT * FROM orders
WHERE user_id = 100 AND created_at > '2025-01-01';
```

`key_len`도 중요하다. 복합 인덱스 `(user_id, status, created_at)`에서 `key_len`이 user_id 크기만큼만 나온다면, 인덱스의 앞 컬럼만 사용되고 있다는 뜻이다. 이 경우 인덱스 설계를 재검토할 필요가 있다.

### 1.3 `rows` 컬럼 — 예상 처리 행 수

옵티마이저가 이 단계에서 읽어야 한다고 추정한 행 수다. 실제 값이 아니라 통계 기반 추정치다. 이 값이 실제 결과 행 수보다 훨씬 크면 인덱스 선택성이 낮거나 통계가 오래된 것이다.

```sql
-- rows가 너무 클 때: 통계 업데이트 시도
ANALYZE TABLE orders;
```

`rows` 값의 곱이 전체 예상 처리량이다. 쿼리 계획에서 각 단계의 `rows`를 곱하면 총 처리 행 수가 나온다. 이 수가 수백만이면 문제가 있는 것이다.

### 1.4 `Extra` 컬럼 — 핵심 부가 정보

`Extra`는 짧은 문구들의 조합인데, 성능에 결정적인 힌트를 담고 있다.

| Extra 값 | 의미 | 좋은가? |
|----------|------|---------|
| `Using index` | 커버링 인덱스, 데이터 파일 미접근 | 매우 좋음 |
| `Using where` | WHERE 필터를 스토리지 엔진 이후 적용 | 보통 |
| `Using temporary` | 임시 테이블 생성 | 나쁨 |
| `Using filesort` | 메모리/디스크 정렬 수행 | 나쁨 |
| `Using index condition` | ICP (Index Condition Pushdown) | 좋음 |
| `Impossible WHERE` | WHERE 조건이 항상 false | 논리 오류 |

`Using temporary`와 `Using filesort`가 함께 나오면 GROUP BY나 ORDER BY가 인덱스를 활용하지 못하고 있다는 신호다. 대용량 테이블에서 이 둘이 함께 등장하면 반드시 인덱스를 조정해야 한다.

```sql
-- Using temporary + Using filesort 발생 (나쁨)
EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status ORDER BY COUNT(*) DESC;

-- 인덱스로 GROUP BY를 처리 (Using index)
-- orders 테이블에 (status) 인덱스가 있고 COUNT(*)가 아닌 경우
EXPLAIN SELECT status FROM orders GROUP BY status;
```

### 1.5 `EXPLAIN ANALYZE` — 실제 실행 통계

MySQL 8.0.18+에서는 `EXPLAIN ANALYZE`를 쓸 수 있다. 옵티마이저 추정치와 실제 실행 결과를 함께 보여준다.

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.id\G
```

출력 예시:
```
-> Table scan on <temporary>  (cost=...) (actual time=2.1..2.1 rows=5000 loops=1)
    -> Aggregate using temporary table  (actual time=45.3..45.3 rows=5000 loops=1)
        -> Nested loop inner join  (cost=...) (actual time=0.1..38.2 rows=50000 loops=1)
            -> Index scan on u using PRIMARY  (actual time=0.05..1.2 rows=5000 loops=1)
            -> Index lookup on o using idx_user_id  (actual time=0.006..0.007 rows=10 loops=5000)
```

`actual time`과 `rows`가 추정치와 크게 다르다면 옵티마이저 통계가 실제 데이터 분포를 반영하지 못하고 있는 것이다.

---

## 2. 풀스캔 방지 전략, 인덱스 힌트, 옵티마이저 통계

### 2.1 풀스캔이 발생하는 패턴

풀스캔은 인덱스가 없어서만 발생하는 게 아니다. 인덱스가 있어도 다음 패턴에서는 인덱스를 사용하지 못한다.

**패턴 1: 컬럼에 함수 적용**

```sql
-- 풀스캔 (인덱스 사용 불가)
SELECT * FROM users WHERE LOWER(email) = 'hong@example.com';
SELECT * FROM orders WHERE DATE(created_at) = '2025-03-17';
SELECT * FROM products WHERE price * 1.1 > 10000;

-- 인덱스 사용 가능
SELECT * FROM users WHERE email = 'hong@example.com';  -- 앱에서 소문자로 저장
SELECT * FROM orders WHERE created_at >= '2025-03-17' AND created_at < '2025-03-18';
SELECT * FROM products WHERE price > 9091;  -- 역산
```

**패턴 2: 묵시적 형변환**

```sql
-- phone이 VARCHAR인데 숫자로 비교 → 풀스캔
SELECT * FROM users WHERE phone = 01012345678;

-- 문자열로 비교
SELECT * FROM users WHERE phone = '010-1234-5678';
```

MySQL은 문자열 컬럼을 숫자와 비교할 때 컬럼 값을 숫자로 변환한다. 이 과정에서 인덱스가 무력화된다.

**패턴 3: 와일드카드 앞 위치**

```sql
-- 풀스캔 (앞에 % 있음)
SELECT * FROM products WHERE name LIKE '%맥북%';

-- 인덱스 range 스캔 (뒤에만 %)
SELECT * FROM products WHERE name LIKE '맥북%';
```

앞에 `%`가 오면 어디서 시작할지 알 수 없어 전체를 다 봐야 한다. 중간 검색이 필요하다면 풀텍스트 인덱스나 ElasticSearch 같은 검색 엔진을 고려해야 한다.

**패턴 4: OR 조건**

```sql
-- 각각 인덱스가 있어도 OR이면 풀스캔이 될 수 있음
SELECT * FROM orders WHERE user_id = 100 OR status = 'PENDING';

-- UNION ALL로 분리 (각각 인덱스 사용)
SELECT * FROM orders WHERE user_id = 100
UNION ALL
SELECT * FROM orders WHERE status = 'PENDING' AND user_id != 100;
```

### 2.2 인덱스 힌트

옵티마이저가 잘못된 인덱스를 선택할 때 힌트로 강제할 수 있다.

```sql
-- 특정 인덱스 강제 사용
SELECT * FROM orders FORCE INDEX (idx_created_at)
WHERE created_at > '2025-01-01';

-- 인덱스 사용 권장 (옵티마이저가 무시할 수 있음)
SELECT * FROM orders USE INDEX (idx_user_id)
WHERE user_id = 100;

-- 특정 인덱스 제외
SELECT * FROM orders IGNORE INDEX (idx_status)
WHERE status = 'COMPLETED';
```

`FORCE INDEX`는 강력하지만 위험하다. 데이터 분포가 바뀌면 더 나쁜 결과를 낳을 수 있다. 운영 환경에서는 `USE INDEX`를 선호하고, 근본적으로는 통계를 업데이트하거나 인덱스를 재설계하는 것이 맞다.

### 2.3 옵티마이저 통계와 히스토그램

MySQL 8.0부터 컬럼 히스토그램을 지원한다. 옵티마이저는 통계를 바탕으로 어떤 인덱스를 쓸지, 어떤 테이블을 먼저 읽을지 결정한다.

```sql
-- 통계 업데이트
ANALYZE TABLE orders;

-- 히스토그램 생성 (MySQL 8.0+)
ANALYZE TABLE orders UPDATE HISTOGRAM ON status, user_id WITH 100 BUCKETS;

-- 히스토그램 조회
SELECT * FROM information_schema.COLUMN_STATISTICS
WHERE TABLE_NAME = 'orders';
```

히스토그램은 특히 카디널리티가 낮은 컬럼(status, type 등)에 효과적이다. 인덱스가 없어도 옵티마이저가 "이 status 값은 전체의 2%밖에 없다"는 걸 알면 더 나은 실행 계획을 수립할 수 있다.

**통계 자동 업데이트 설정:**

```sql
-- InnoDB 통계 영속화 (재시작 후에도 유지)
SET GLOBAL innodb_stats_persistent = ON;

-- 통계 샘플 페이지 수 (높을수록 정확하지만 느림)
SET GLOBAL innodb_stats_persistent_sample_pages = 20;
```

### 2.4 커버링 인덱스 — 가장 효과적인 최적화

쿼리에서 필요한 모든 컬럼이 인덱스에 포함되면 데이터 파일을 전혀 읽지 않아도 된다. 이를 커버링 인덱스(Covering Index)라 한다. `EXPLAIN`의 `Extra`에 `Using index`가 나온다.

```sql
-- users 테이블에 (created_at, name, email) 복합 인덱스가 있다면
SELECT name, email FROM users WHERE created_at > '2025-01-01';
-- Extra: Using index (데이터 파일 미접근)

-- id만 SELECT해도 PK가 인덱스에 포함되어 있어 커버링 가능
SELECT id FROM orders WHERE user_id = 100;
-- Extra: Using index
```

인덱스 구조를 고려한 쿼리 작성이 중요하다. 자주 실행되는 쿼리의 SELECT 컬럼과 WHERE 컬럼을 분석해 복합 인덱스를 설계하면 극적인 성능 향상을 얻을 수 있다.

---

## 3. JOIN 전략 — 내부에서 무슨 일이 벌어지는가

JOIN은 SQL의 꽃이지만, 잘못 사용하면 가장 큰 성능 주범이 된다. MySQL은 세 가지 주요 JOIN 알고리즘을 사용한다.

### 3.1 Nested Loop Join (NLJ)

MySQL의 기본 JOIN 알고리즘이다. 외부 테이블(Outer)을 순회하면서 각 행에 대해 내부 테이블(Inner)을 검색한다.

```
for each row R1 in Outer:
    for each row R2 in Inner where join_condition(R1, R2):
        output (R1, R2)
```

```
[Outer: users]        [Inner: orders]
user_id=1  ─────────► idx_user_id=1 → orders(1,2,3)
user_id=2  ─────────► idx_user_id=2 → orders(4)
user_id=3  ─────────► idx_user_id=3 → orders(5,6,7,8)
...
```

**성능 특성:**
- Inner 테이블에 인덱스가 있으면 매우 효율적
- Inner 테이블이 크고 인덱스가 없으면 O(N²) 성능
- 메모리를 거의 사용하지 않음
- 작은 Outer + 인덱스 있는 Inner 조합이 최적

```sql
-- users가 Outer, orders가 Inner인 경우
-- orders.user_id에 인덱스가 있으면 NLJ가 효율적
EXPLAIN SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.vip = 1;
-- type: eq_ref on orders (인덱스 활용)
```

### 3.2 Block Nested Loop Join (BNL)

Inner 테이블에 인덱스가 없을 때 MySQL이 사용하는 방식이다. Outer 테이블의 행들을 메모리 버퍼(Join Buffer)에 담고, Inner 테이블을 한 번에 스캔하면서 버퍼와 매칭한다.

```
load chunk of Outer rows into join_buffer
for each row R2 in Inner:
    for each row R1 in join_buffer where join_condition(R1, R2):
        output (R1, R2)
```

버퍼 크기(`join_buffer_size`)를 늘리면 Inner 테이블 스캔 횟수를 줄일 수 있다.

```sql
-- join_buffer_size 조정
SET SESSION join_buffer_size = 4 * 1024 * 1024;  -- 4MB
```

MySQL 8.0.20+에서는 BNL이 Hash Join으로 대체됐다.

### 3.3 Hash Join (MySQL 8.0.18+)

등가 조건(equi-join)에서 인덱스 없이도 효율적으로 동작하는 알고리즘이다.

**Phase 1 — Build:** 작은 테이블(Build Table)을 해시 테이블로 만든다.
**Phase 2 — Probe:** 큰 테이블(Probe Table)을 순회하면서 해시 테이블을 조회한다.

```
[Build Phase]                    [Probe Phase]
users → hash_table               for each row in orders:
  hash(user_id=1) → {row1}          key = hash(orders.user_id)
  hash(user_id=2) → {row2}          match = hash_table.lookup(key)
  ...                               if match: output
```

**성능 특성:**
- 인덱스 없이도 O(N+M)으로 동작 (N: build table, M: probe table)
- 메모리 사용량이 높음 (build table이 메모리에 올라감)
- 등가 조건에서만 사용 가능 (>, <, LIKE 불가)
- 대용량 테이블 간 JOIN에서 NLJ보다 빠를 수 있음

```sql
-- Hash Join 확인
EXPLAIN FORMAT=TREE
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id\G

-- 출력에 "Hash" 키워드 확인
-- -> Inner hash join (o.user_id = u.id)
--     -> Table scan on o
--     -> Hash
--         -> Table scan on u
```

### 3.4 Sort Merge Join

MySQL InnoDB에서는 일반적으로 직접 지원하지 않지만, ORDER BY + JOIN 최적화 맥락에서 유사한 동작을 한다. PostgreSQL에서는 명시적으로 지원한다.

원리: 두 테이블을 JOIN 키로 정렬 후, 포인터를 앞으로 이동하며 병합한다.

```
sorted_A: [1,1,2,3,4,5]
sorted_B: [1,2,2,3,5]

merge:
  A[0]=1, B[0]=1 → match, output
  A[1]=1, B[0]=1 → match, output
  A[2]=2, B[1]=2 → match, output
  ...
```

O(N log N + M log M) (정렬) + O(N+M) (병합). 이미 정렬된 인덱스가 있다면 정렬 비용 없이 O(N+M).

### 3.5 JOIN 최적화 실전 팁

**Outer 테이블을 작게 만들어라:**

```sql
-- 나쁜 예: 전체 users와 JOIN 후 필터
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country = 'KR' AND o.status = 'COMPLETED';

-- 좋은 예: 작은 집합을 먼저 필터링 (서브쿼리/파생 테이블)
SELECT u.name, o.amount
FROM (SELECT id, name FROM users WHERE country = 'KR') u
JOIN orders o ON u.id = o.user_id
WHERE o.status = 'COMPLETED';
```

**JOIN 컬럼의 데이터 타입을 일치시켜라:**

```sql
-- users.id = INT, orders.user_id = BIGINT → 묵시적 형변환 발생
-- 컬럼 타입을 통일하거나 명시적 CAST 사용
SELECT * FROM users u
JOIN orders o ON u.id = CAST(o.user_id AS UNSIGNED);
```

**STRAIGHT_JOIN으로 JOIN 순서 강제:**

```sql
-- 옵티마이저가 잘못된 순서를 선택할 때
SELECT STRAIGHT_JOIN u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.vip = 1;
```

---

## 4. 서브쿼리 vs JOIN vs EXISTS 성능 비교

"서브쿼리 대신 JOIN을 써야 한다"는 말을 많이 듣는다. 하지만 이건 MySQL 5.x 시대의 이야기다. 현대 MySQL은 많은 경우 서브쿼리를 자동으로 최적화한다. 실제로 어떤 차이가 있는지 정확히 이해해야 한다. 맹목적으로 JOIN으로 바꾸는 것보다, EXPLAIN으로 실행 계획을 확인한 후 판단하는 것이 정확하다.

### 4.1 IN 서브쿼리 vs JOIN

```sql
-- 서브쿼리 버전
SELECT * FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE country = 'KR'
);

-- JOIN 버전
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.country = 'KR';
```

MySQL 5.5 이전에는 `IN` 서브쿼리가 매 행마다 실행되는 "의존 서브쿼리(Dependent Subquery)"로 처리돼 O(N*M)이었다. MySQL 5.6+부터는 서브쿼리를 반조인(Semi-join)으로 변환해 내부적으로 JOIN과 유사하게 처리한다.

`EXPLAIN`으로 확인하는 방법:

```sql
EXPLAIN SELECT * FROM orders
WHERE user_id IN (SELECT id FROM users WHERE country = 'KR');
```

`select_type`이 `SUBQUERY`면 독립 실행, `DEPENDENT SUBQUERY`면 각 행마다 실행(위험), `SEMIJOIN`이면 최적화된 것이다.

### 4.2 EXISTS vs IN

```sql
-- EXISTS 버전
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM users u
    WHERE u.id = o.user_id AND u.country = 'KR'
);

-- IN 버전
SELECT * FROM orders o
WHERE o.user_id IN (
    SELECT id FROM users WHERE country = 'KR'
);
```

**EXISTS의 특성:**
- 조건을 만족하는 행이 하나라도 있으면 즉시 종료 (Short-circuit)
- 외부 쿼리의 각 행에 대해 서브쿼리 실행 — Outer가 작을 때 유리
- NULL 처리에서 안전: `IN`은 서브쿼리 결과에 NULL이 있으면 전체가 false

**IN의 특성:**
- 서브쿼리 결과를 먼저 구체화(materialize)
- Inner 집합이 작을 때 유리
- NULL이 포함되면 매칭 실패

```sql
-- NULL 함정
SELECT * FROM orders WHERE user_id NOT IN (SELECT id FROM users);
-- users.id에 NULL이 하나라도 있으면 결과가 0건!

-- NOT EXISTS가 안전
SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id);
```

### 4.3 스칼라 서브쿼리 — 가장 위험한 패턴

```sql
-- 매 행마다 서브쿼리 실행: O(N * M)
SELECT
    o.id,
    o.amount,
    (SELECT name FROM users WHERE id = o.user_id) AS user_name
FROM orders o
WHERE o.created_at > '2025-01-01';
```

orders가 10만 건이면 users 서브쿼리가 10만 번 실행된다. 각 행마다 독립적인 쿼리가 실행되기 때문에 N+1 문제와 동일한 구조다. 반드시 JOIN으로 바꿔야 한다.

```sql
-- JOIN으로 변환: O(N + M)
SELECT o.id, o.amount, u.name AS user_name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at > '2025-01-01';
```

**예외: 집계 서브쿼리**

```sql
-- 이 패턴은 상관 서브쿼리지만 한 번만 실행됨
SELECT name,
    (SELECT MAX(created_at) FROM orders WHERE user_id = u.id) AS last_order_date
FROM users u
WHERE u.vip = 1;
```

이 경우도 users가 크다면 위험하다. `LEFT JOIN`으로 변환하는 것이 안전하다.

```sql
SELECT u.name, MAX(o.created_at) AS last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.vip = 1
GROUP BY u.id, u.name;
```

### 4.4 파생 테이블(Derived Table)과 CTE

```sql
-- 파생 테이블
SELECT d.user_id, d.total
FROM (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    WHERE created_at > '2025-01-01'
    GROUP BY user_id
) d
WHERE d.total > 100000;

-- CTE (MySQL 8.0+)
WITH monthly_totals AS (
    SELECT user_id, SUM(amount) AS total
    FROM orders
    WHERE created_at > '2025-01-01'
    GROUP BY user_id
)
SELECT * FROM monthly_totals WHERE total > 100000;
```

MySQL 8.0에서 CTE는 기본적으로 한 번 물질화(materialize)된 후 재사용된다. 반복 참조 시 유리하다. 반면 파생 테이블은 옵티마이저가 인라인(merge)하거나 물질화할지 결정한다.

### 4.5 성능 결정 요인 정리

```
┌─────────────────────────────────────────────────────────────┐
│ 상황                          │ 권장                         │
├─────────────────────────────────────────────────────────────┤
│ 단순 존재 여부 확인              │ EXISTS                      │
│ 소규모 결과 집합 필터링           │ IN (서브쿼리)               │
│ 대규모 JOIN + 추가 컬럼 필요      │ JOIN                        │
│ NULL 포함 가능성 있는 NOT IN     │ NOT EXISTS                  │
│ 스칼라 서브쿼리 (SELECT 절)      │ JOIN으로 변환               │
│ 복잡한 다단계 집계               │ CTE                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. 슬로우 쿼리 로그 설정과 pt-query-digest 활용

### 5.1 슬로우 쿼리 로그 설정

운영 환경에서 느린 쿼리를 잡으려면 슬로우 쿼리 로그가 필수다.

**MySQL 설정 (`my.cnf` 또는 `my.ini`):**

```ini
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1       # 1초 이상 걸리는 쿼리 기록
log_queries_not_using_indexes = ON  # 인덱스 미사용 쿼리도 기록
min_examined_row_limit = 1000  # 1000행 이상 검사한 쿼리만
log_slow_admin_statements = ON  # DDL도 포함
```

**런타임 설정 변경 (재시작 없이):**

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.5;  -- 0.5초로 낮춰서 더 많이 잡기
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

**슬로우 쿼리 로그 형식:**

```
# Time: 2025-03-17T14:23:45.123456+09:00
# User@Host: appuser[appuser] @ web01 [192.168.1.10]
# Query_time: 8.234567  Lock_time: 0.000123 Rows_sent: 1234 Rows_examined: 9876543
SET timestamp=1742268225;
SELECT * FROM orders
WHERE created_at > '2025-01-01' AND status = 'COMPLETED'
ORDER BY amount DESC
LIMIT 100;
```

`Query_time`과 `Rows_examined`의 비율이 중요하다. Rows_examined가 Rows_sent보다 1000배 이상 크다면 인덱스 효율이 매우 낮은 것이다.

### 5.2 pt-query-digest 설치 및 기본 사용

Percona Toolkit의 `pt-query-digest`는 슬로우 쿼리 로그를 분석해 패턴별로 집계해준다.

```bash
# 설치 (CentOS/RHEL)
yum install percona-toolkit

# 설치 (Ubuntu/Debian)
apt-get install percona-toolkit

# 기본 분석
pt-query-digest /var/log/mysql/slow.log

# 최근 1시간 로그만 분석
pt-query-digest --since=1h /var/log/mysql/slow.log

# 특정 데이터베이스만
pt-query-digest --filter '$event->{db} eq "production"' /var/log/mysql/slow.log
```

**출력 구조:**

```
# Profile
# Rank Query ID           Response time  Calls  R/Call V/M   Item
# ==== ================== ============= ====== ====== ===== ====
#    1 0xABC123...         45.2345 60.1%    234  0.193  0.45 SELECT orders
#    2 0xDEF456...         18.1234 24.1%   1823  0.009  0.02 SELECT users
#    3 0xGHI789...          8.3456 11.1%     12  0.695  0.89 UPDATE products
```

**가장 중요한 지표:**
- **Response time %**: 전체 응답 시간에서 차지하는 비율 — 여기가 높은 쿼리부터 최적화
- **Calls**: 호출 횟수 — 빈번한 쿼리는 조금만 개선해도 효과 큼
- **R/Call**: 호출당 평균 응답 시간
- **V/M**: 변동 계수(Variance/Mean) — 높으면 성능이 불안정

**개별 쿼리 상세 분석:**

```
# Query 1: 234 times, avg 0.193s, total 45.2s
# Attribute    pct   total     min     max     avg     95%  stddev  median
# ============ === ======= ======= ======= ======= ======= ======= =======
# Count         60     234
# Exec time     60  45234ms   180ms  2340ms   193ms   450ms   234ms   165ms
# Rows sent      5   12345       1    1000      52     200      87      20
# Rows exam     92 9876543    1000 2000000   42208  150000   89234   12000

# Databases     production
#
SELECT * FROM orders
WHERE created_at > '2025-01-01' AND status = 'COMPLETED'
ORDER BY amount DESC LIMIT 100\G
```

`Rows exam`이 `Rows sent`보다 42208/52 ≈ 800배 많다. 인덱스 없이 대량 스캔하고 있다는 뜻이다.

### 5.3 pt-query-digest 고급 활용

**MySQL 프로세스 리스트 실시간 분석:**

```bash
# 현재 실행 중인 쿼리 수집 (5분간)
pt-query-digest --processlist h=localhost,u=root,p=secret \
    --interval 0.5 \
    --run-time 5m
```

**tcpdump로 네트워크 트래픽 분석:**

```bash
# MySQL 포트(3306) 트래픽 캡처
tcpdump -s 65535 -x -nn -q -tttt -i any -c 1000 port 3306 > mysql.pcap

# 분석
pt-query-digest --type tcpdump mysql.pcap
```

이 방식은 슬로우 쿼리 로그에 잡히지 않는 빠른 쿼리들도 모두 분석할 수 있어, 빈번하게 호출되는 "작지만 많은" 쿼리를 찾는 데 유용하다.

**결과를 MySQL에 저장:**

```bash
pt-query-digest --review h=localhost,D=percona,t=query_review \
    --history h=localhost,D=percona,t=query_history \
    /var/log/mysql/slow.log
```

쿼리 분석 결과를 DB에 누적하면 시간별 트렌드를 추적할 수 있다. 특정 배포 후 느려진 쿼리를 비교 분석하는 데 매우 유용하다.

### 5.4 Performance Schema로 실시간 분석

MySQL 8.0에서는 Performance Schema의 `events_statements_summary_by_digest`가 강력한 대안이다.

```sql
-- 평균 실행 시간이 가장 긴 쿼리 TOP 10
SELECT
    ROUND(AVG_TIMER_WAIT / 1e9, 3) AS avg_ms,
    COUNT_STAR AS executions,
    ROUND(SUM_TIMER_WAIT / 1e9, 3) AS total_ms,
    SUM_ROWS_EXAMINED / COUNT_STAR AS avg_rows_examined,
    LEFT(DIGEST_TEXT, 100) AS query
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME = 'production'
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;
```

```sql
-- 풀스캔이 많은 쿼리
SELECT
    DIGEST_TEXT,
    COUNT_STAR,
    SUM_NO_INDEX_USED,
    SUM_NO_GOOD_INDEX_USED
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_NO_INDEX_USED > 0
ORDER BY SUM_NO_INDEX_USED DESC
LIMIT 20;
```

### 5.5 슬로우 쿼리 발견 후 처리 프로세스

```
1. pt-query-digest로 상위 쿼리 식별
        ↓
2. EXPLAIN으로 실행 계획 확인
        ↓
3. 문제 패턴 분류
   ├── 풀스캔(ALL) → 인덱스 추가/수정
   ├── Using filesort → ORDER BY 컬럼 인덱스 추가
   ├── Using temporary → GROUP BY 최적화
   └── 스칼라 서브쿼리 → JOIN 변환
        ↓
4. 개선안 작성 (인덱스 추가, 쿼리 리팩토링)
        ↓
5. 스테이징 환경에서 EXPLAIN ANALYZE로 검증
        ↓
6. 운영 적용 (온라인 DDL 사용: ALTER TABLE ... ALGORITHM=INPLACE)
        ↓
7. 슬로우 쿼리 로그로 효과 확인
```

**온라인 인덱스 추가 (서비스 중단 없이):**

```sql
-- InnoDB Online DDL
ALTER TABLE orders
ADD INDEX idx_user_status_created (user_id, status, created_at)
ALGORITHM=INPLACE, LOCK=NONE;

-- pt-online-schema-change (Percona Toolkit)
pt-online-schema-change \
    --alter "ADD INDEX idx_user_status_created (user_id, status, created_at)" \
    D=production,t=orders \
    --execute
```

`ALGORITHM=INPLACE, LOCK=NONE`은 테이블 잠금 없이 인덱스를 추가한다. 단, 이 과정에서 DML(INSERT/UPDATE/DELETE)은 허용되지만 추가 오버헤드가 있으므로 트래픽이 낮은 시간대에 실행하는 것이 좋다.

---

## 마무리 — 쿼리 최적화의 마인드셋

쿼리 최적화는 한 번 하고 끝나는 일이 아니다. 데이터는 계속 쌓이고, 트래픽 패턴은 바뀌고, 기능은 추가된다. 오늘 최적화한 쿼리가 6개월 후에 다시 슬로우 쿼리가 될 수 있다.

핵심은 측정 기반으로 접근하는 것이다. 느낌으로 최적화하지 말고, `EXPLAIN`과 `EXPLAIN ANALYZE`로 실행 계획을 확인하고, 슬로우 쿼리 로그와 pt-query-digest로 실제 병목을 찾아야 한다. 인덱스를 맹목적으로 추가하는 것도 금물이다 — 인덱스는 쓰기 비용을 높이고 공간을 차지한다.

가장 효과가 큰 순서는 대체로 이렇다:
1. 풀스캔 제거 (인덱스 추가)
2. 스칼라 서브쿼리 → JOIN 변환
3. 커버링 인덱스 적용
4. 불필요한 데이터 조회 제거 (SELECT * 지양)
5. 쿼리 캐싱 또는 애플리케이션 레벨 캐싱 도입

데이터베이스 시리즈 다음 편에서는 MySQL 아키텍처의 핵심인 버퍼 풀, Redo/Undo 로그, 그리고 InnoDB의 물리적 구조를 다룬다. 실행 계획이 "왜 그렇게 결정됐는가"를 이해하려면 스토리지 엔진의 내부 구조를 알아야 한다.

---

## 참고 자료

- **High Performance MySQL, 4th Edition** — Silvia Botros, Jeremy Tinley (O'Reilly, 2022) — MySQL 성능 최적화의 표준 레퍼런스. EXPLAIN 분석, 인덱스 전략, 스키마 설계 심화 내용 수록
- **MySQL 8.0 Reference Manual: EXPLAIN Statement** — https://dev.mysql.com/doc/refman/8.0/en/explain.html — 공식 문서, 각 컬럼과 Extra 값의 정확한 의미
- **Use The Index, Luke** — https://use-the-index-luke.com — Markus Winand의 무료 온라인 책. 인덱스 내부 동작과 SQL 최적화를 시각적으로 설명
- **Percona Toolkit Documentation: pt-query-digest** — https://docs.percona.com/percona-toolkit/pt-query-digest.html — 슬로우 쿼리 분석 도구 공식 매뉴얼
- **Database Internals** — Alex Petrov (O'Reilly, 2019) — B-Tree, LSM-Tree 등 스토리지 엔진의 물리적 구조와 JOIN 알고리즘 이론적 배경
- **MySQL Performance Blog** — https://www.percona.com/blog — Percona 엔지니어들의 실전 MySQL 성능 분석 사례 모음
