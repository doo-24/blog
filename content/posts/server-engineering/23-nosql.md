---
title: "[데이터베이스] 6편 — NoSQL: 언제, 왜, 어떻게"
date: 2026-03-17T23:04:00+09:00
draft: false
tags: ["NoSQL", "MongoDB", "Redis", "CAP", "데이터베이스", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "NoSQL 유형별 특성(Document, Key-Value, Column, Graph), MongoDB 데이터 모델링(임베딩 vs 참조), Redis 자료구조별 활용 패턴, CAP 정리 실전 해석, RDBMS+NoSQL 혼합 설계까지"
---

2009년 어느 컨퍼런스에서 "NoSQL"이라는 해시태그가 처음 등장했을 때, 많은 엔지니어들은 이것이 단순한 유행어라고 생각했다.

10년이 지난 지금, 거의 모든 규모 있는 서비스의 아키텍처에는 최소 하나 이상의 NoSQL 데이터베이스가 자리 잡고 있다.

그런데 현장에서 보면 "NoSQL이 빠르다고 해서 썼는데 오히려 느려졌다"거나, "MongoDB를 쓰다가 데이터 일관성 문제가 생겼다"는 이야기를 자주 듣는다.

NoSQL을 제대로 쓰려면 단순히 "SQL 없는 DB"가 아니라, 각 유형이 어떤 트레이드오프를 선택했는지를 이해해야 한다. 이 글은 그 트레이드오프를 설계 결정의 언어로 번역하는 시도다.

---

## 1. NoSQL 유형별 특성: 어떤 문제를 풀기 위해 만들어졌는가

NoSQL은 단일 기술이 아니다. "관계형 모델의 제약을 벗어난 데이터베이스들의 집합"이라는 표현이 더 정확하다.

각 유형은 서로 다른 문제를 해결하기 위해 설계되었으며, 그에 따라 완전히 다른 데이터 모델과 쿼리 패턴을 가진다.

### 1-1. Document Store

**대표 제품**: MongoDB, CouchDB, Amazon DocumentDB

Document Store는 데이터를 JSON(혹은 BSON) 형태의 문서로 저장한다. 각 문서는 스키마가 없어도 되고, 중첩 구조와 배열을 자연스럽게 표현할 수 있다.

```
// MongoDB에 저장되는 주문 문서 예시
{
  "_id": ObjectId("64a1f2c3d5e6f7a8b9c0d1e2"),
  "orderId": "ORD-2024-001",
  "customer": {
    "userId": "usr_abc123",
    "name": "홍길동",
    "email": "hong@example.com"
  },
  "items": [
    { "productId": "prod_001", "name": "노트북", "qty": 1, "price": 1500000 },
    { "productId": "prod_002", "name": "마우스", "qty": 2, "price": 35000 }
  ],
  "totalAmount": 1570000,
  "status": "PAID",
  "createdAt": ISODate("2024-07-01T10:30:00Z")
}
```

Document Store가 빛나는 순간은 **데이터 구조가 계층적이고, 함께 조회되는 데이터가 함께 저장되어야 할 때**다. 위 예시처럼 주문과 주문 상품을 함께 저장하면, 단일 쿼리로 전체 주문 정보를 조회할 수 있다. RDBMS였다면 `orders`, `order_items`, `customers` 테이블을 JOIN해야 했을 일이다. 데이터를 "함께 쓰이는 것은 함께 저장한다"는 원칙으로 구성하면 네트워크 왕복과 JOIN 비용을 동시에 줄일 수 있다.

**주의할 점**: "스키마리스"는 "스키마 없이 잘 작동한다"는 뜻이 아니다. 스키마를 애플리케이션 코드가 관리한다는 뜻이다. 문서 구조가 제각각으로 흩어지면 쿼리가 불가능해진다.

### 1-2. Key-Value Store

**대표 제품**: Redis, Amazon DynamoDB, Memcached

가장 단순한 형태의 NoSQL이다. 키로 값을 저장하고, 키로 값을 꺼낸다. 이 단순함이 극한의 성능을 낸다.

```
// 개념적 구조
SET session:user123 '{"userId":"user123","role":"admin","exp":1720000000}'
GET session:user123
DEL session:user123
```

Key-Value Store가 잘 맞는 사용 사례는 명확하다: **세션 저장**, **캐싱**, **분산 락**, **속도 제한(Rate Limiting)**. 값의 내부 구조로 쿼리할 수 없다는 제약이 있지만, O(1)에 가까운 읽기/쓰기 성능이 그 대가를 만회한다.

Redis의 경우 단순한 Key-Value를 넘어 다양한 자료구조(List, Set, Sorted Set, Hash, Stream 등)를 제공한다. 이는 3장에서 자세히 다룬다.

### 1-3. Wide-Column Store (Column-Family Store)

**대표 제품**: Apache Cassandra, HBase, Google Bigtable

Wide-Column Store는 RDBMS처럼 행과 열로 데이터를 표현하지만, 각 행이 서로 다른 열을 가질 수 있고, 열을 그룹(Column Family)으로 묶는다. 핵심은 **데이터가 컬럼 단위로 디스크에 저장된다는 점**이다.

```
// Cassandra 스키마 예시: 타임시리즈 데이터
CREATE TABLE sensor_readings (
    sensor_id  TEXT,
    bucket     TEXT,           -- 시간 버킷 (e.g., "2024-07-01")
    ts         TIMESTAMP,
    value      DOUBLE,
    PRIMARY KEY ((sensor_id, bucket), ts)
) WITH CLUSTERING ORDER BY (ts DESC);
```

Wide-Column이 강한 영역은 **시계열 데이터**, **로그 집계**, **대규모 쓰기 워크로드**다. Cassandra는 쓰기가 항상 O(1)이고, 수평 확장이 자연스럽다. 반면 JOIN이 없고, 쿼리 패턴을 미리 알고 스키마를 설계해야 한다.

```
// 시계열 데이터 조회: 특정 센서의 최근 1시간 데이터
SELECT ts, value
FROM sensor_readings
WHERE sensor_id = 'sensor-42'
  AND bucket = '2024-07-01'
  AND ts >= '2024-07-01T09:00:00Z'
  AND ts <= '2024-07-01T10:00:00Z';
```

### 1-4. Graph Database

**대표 제품**: Neo4j, Amazon Neptune, JanusGraph

그래프 DB는 데이터를 노드(Node)와 엣지(Edge)로 표현한다. RDBMS에서 N-depth JOIN으로 표현해야 하는 관계 탐색이 그래프 DB에서는 자연스러운 쿼리가 된다.

```
// Neo4j Cypher 쿼리: "A의 친구의 친구가 좋아하는 상품"
MATCH (user:User {id: 'user_A'})-[:FRIEND*2]-(fof:User)-[:LIKES]->(product:Product)
WHERE NOT (user)-[:LIKES]->(product)
RETURN DISTINCT product.name, COUNT(*) AS recommendationScore
ORDER BY recommendationScore DESC
LIMIT 10;
```

RDBMS로 같은 쿼리를 작성하면 self-join이 중첩되고, depth가 깊어질수록 쿼리 복잡도가 폭발적으로 증가한다. 그래프 DB가 빛나는 영역은 **소셜 네트워크**, **추천 엔진**, **사기 탐지(Fraud Detection)**, **지식 그래프**다.

### 1-5. 유형 비교 요약

```
┌─────────────────┬───────────────────┬─────────────────────────┬─────────────────────────┐
│    유형          │   강점             │   약점                   │   대표 사용 사례         │
├─────────────────┼───────────────────┼─────────────────────────┼─────────────────────────┤
│ Document        │ 유연한 스키마      │ 강한 JOIN 불가           │ 상품 카탈로그, CMS      │
│                 │ 계층 데이터 자연   │ 중복 데이터 발생          │ 주문, 사용자 프로필     │
├─────────────────┼───────────────────┼─────────────────────────┼─────────────────────────┤
│ Key-Value       │ 극한 성능(O(1))   │ 값 내부 쿼리 불가        │ 세션, 캐시, 분산락      │
│                 │ 단순 수평 확장    │ 복잡한 쿼리 불가          │ Rate Limiting           │
├─────────────────┼───────────────────┼─────────────────────────┼─────────────────────────┤
│ Wide-Column     │ 고속 쓰기         │ JOIN 없음                │ 시계열, 로그, IoT       │
│                 │ 무한 수평 확장    │ 쿼리 패턴 사전 설계 필요  │ 이벤트 스트림           │
├─────────────────┼───────────────────┼─────────────────────────┼─────────────────────────┤
│ Graph           │ 관계 탐색 최적화  │ 글로벌 집계 느림         │ 소셜 그래프, 추천       │
│                 │ 직관적 관계 표현  │ 수평 확장 어려움          │ 사기 탐지               │
└─────────────────┴───────────────────┴─────────────────────────┴─────────────────────────┘
```

---

## 2. MongoDB 데이터 모델링: 임베딩 vs 참조

MongoDB를 처음 접하는 엔지니어가 가장 많이 틀리는 부분이 바로 데이터 모델링이다.

"RDBMS처럼 정규화하면 되겠지"라고 생각하거나, 반대로 "다 때려 박으면 되겠지"라고 생각하는 두 극단이 모두 문제를 일으킨다.

### 2-1. 임베딩(Embedding): 함께 조회되는 것은 함께 저장한다

임베딩은 관련 데이터를 하나의 문서 안에 중첩해서 저장하는 방식이다. 단일 조회로 모든 데이터를 가져올 수 있어 성능상 이점이 있다.

**언제 임베딩을 선택하는가:**
- 두 엔티티가 항상 함께 조회될 때
- 자식 엔티티의 수가 제한적이고 예측 가능할 때 (e.g., 주문당 상품 수는 보통 1~20개)
- 자식 엔티티가 독립적으로 접근되지 않을 때

```javascript
// 좋은 임베딩 예시: 블로그 포스트와 댓글 (댓글이 포스트 없이 조회되지 않음)
db.posts.insertOne({
  _id: ObjectId("..."),
  title: "MongoDB 모델링 가이드",
  content: "...",
  author: { userId: "usr_001", name: "김철수" },  // 작성자 기본 정보 임베딩
  tags: ["mongodb", "database"],
  comments: [
    {
      commentId: ObjectId("..."),
      author: { userId: "usr_002", name: "이영희" },
      body: "유익한 글 감사합니다!",
      createdAt: ISODate("2024-07-01T11:00:00Z")
    }
  ],
  commentCount: 1,
  createdAt: ISODate("2024-07-01T10:00:00Z")
});
```

**임베딩의 함정 — 무제한 배열 성장**

MongoDB 문서의 최대 크기는 16MB다. 댓글이 수만 개 달릴 수 있는 포스트에서 댓글을 모두 임베딩하면 문서 크기가 폭발한다. 또한 배열이 커질수록 업데이트 시 문서 전체를 다시 쓰게 되어 성능이 저하된다.

```javascript
// 안티패턴: 댓글이 무제한으로 증가하는 경우
// comments 배열이 수천 개가 되면 업데이트 성능이 급격히 저하됨
db.posts.updateOne(
  { _id: postId },
  { $push: { comments: newComment } }  // 문서 전체 재작성 발생
);
```

이런 경우 **Bucket 패턴**을 고려한다:

```javascript
// Bucket 패턴: 댓글을 N개씩 묶어서 저장
{
  _id: ObjectId("..."),
  postId: ObjectId("..."),
  page: 1,
  comments: [/* 최대 50개 */],
  count: 50
}
```

### 2-2. 참조(Reference): 독립적으로 존재하는 엔티티는 분리한다

참조는 RDBMS의 외래 키처럼 다른 문서의 `_id`를 저장하는 방식이다. 참조된 문서를 조회하려면 애플리케이션 레벨에서 추가 쿼리를 해야 한다 (혹은 `$lookup`을 쓴다).

**언제 참조를 선택하는가:**
- 자식 엔티티가 독립적으로 접근될 때
- 자식 엔티티의 수가 무제한으로 증가할 수 있을 때
- 여러 부모 문서가 같은 자식 문서를 공유할 때 (중복 방지)

```javascript
// 참조 예시: 상품과 카테고리 (카테고리는 여러 상품이 공유)
// products 컬렉션
{
  _id: ObjectId("prod_001"),
  name: "MacBook Pro 14인치",
  price: 2500000,
  categoryId: ObjectId("cat_electronics"),  // 참조
  brandId: ObjectId("brand_apple")          // 참조
}

// categories 컬렉션 (독립적으로 존재)
{
  _id: ObjectId("cat_electronics"),
  name: "전자제품",
  parentCategoryId: null
}
```

```javascript
// $lookup으로 참조 데이터 조인 (aggregation pipeline)
db.products.aggregate([
  { $match: { _id: ObjectId("prod_001") } },
  {
    $lookup: {
      from: "categories",
      localField: "categoryId",
      foreignField: "_id",
      as: "category"
    }
  },
  { $unwind: "$category" }
]);
```

`$lookup`은 편리하지만 자주 사용하면 성능 문제가 생긴다. MongoDB는 분산 JOIN을 최적화하지 않는다. `$lookup`이 자주 필요하다면 설계를 재검토해야 한다.

### 2-3. 혼합 전략: 역정규화와 부분 임베딩

실무에서는 순수 임베딩이나 순수 참조 대신 **부분 임베딩(Partial Embedding)**을 쓰는 경우가 많다. 자주 함께 조회되는 필드만 임베딩하고, 나머지는 참조로 처리한다.

```javascript
// 부분 임베딩 예시: 주문 문서에 고객의 일부 정보만 임베딩
{
  "_id": ObjectId("..."),
  "orderId": "ORD-2024-001",
  // 주문 당시 고객 정보 스냅샷 (고객이 나중에 이름을 바꿔도 주문 내역은 보존)
  "customer": {
    "userId": "usr_abc123",   // 참조 용도
    "name": "홍길동",          // 임베딩 (스냅샷)
    "email": "hong@example.com"
  },
  "shippingAddress": {        // 배송지는 주문 당시 값을 그대로 저장
    "street": "테헤란로 123",
    "city": "서울",
    "zipCode": "06234"
  },
  "items": [...]
}
```

이 패턴의 핵심은 **"변경되면 함께 업데이트해야 하는가, 아니면 당시 값을 보존해야 하는가"**를 구분하는 것이다. 주문 배송지는 고객이 주소를 바꿔도 주문 당시 주소를 유지해야 하므로 임베딩이 올바른 선택이다.

### 2-4. 인덱스 전략

MongoDB 쿼리 성능의 80%는 인덱스 설계에서 결정된다.

```javascript
// 복합 인덱스: ESR(Equality-Sort-Range) 규칙 적용
// 쿼리: status가 'PAID'이고 createdAt 범위로 정렬된 결과
db.orders.createIndex({ status: 1, createdAt: -1 });

// 인덱스 사용 확인
db.orders.find({ status: "PAID" })
         .sort({ createdAt: -1 })
         .explain("executionStats");

// 중요: explain()으로 COLLSCAN이 아닌 IXSCAN인지 반드시 확인
```

```javascript
// 배열 필드 인덱싱: 멀티키 인덱스 자동 생성
db.products.createIndex({ tags: 1 });
// tags 배열의 각 요소에 대해 인덱스 엔트리가 생성됨

// TTL 인덱스: 자동 만료 (세션, 로그 등에 유용)
db.sessions.createIndex({ expireAt: 1 }, { expireAfterSeconds: 0 });
// expireAt 필드의 시간이 지나면 자동 삭제
```

**인덱스 안티패턴**: 쓰기가 많은 컬렉션에 인덱스를 남발하면 오히려 쓰기 성능이 저하된다. 인덱스는 읽기를 위한 쓰기 비용이다. `db.collection.stats()`로 인덱스 크기와 사용률을 주기적으로 점검해야 한다.

### 2-5. 트랜잭션: 필요하지만 남용하지 말 것

MongoDB 4.0부터 멀티 문서 트랜잭션을 지원한다. 그러나 트랜잭션은 성능 비용이 크고, MongoDB의 강점인 "단일 문서 원자성"을 우회한다.

```javascript
// 트랜잭션이 필요한 경우: 재고 차감 + 주문 생성이 원자적이어야 할 때
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    // 재고 차감
    const result = await db.products.updateOne(
      { _id: productId, stock: { $gte: qty } },
      { $inc: { stock: -qty } },
      { session }
    );
    if (result.matchedCount === 0) {
      throw new Error("재고 부족");
    }
    // 주문 생성
    await db.orders.insertOne(orderDoc, { session });
  });
} finally {
  await session.endSession();
}
```

트랜잭션을 자주 써야 한다면, 설계를 재검토해야 한다는 신호다. 단일 문서로 표현할 수 있는 경우를 먼저 찾아라.

---

## 3. Redis 자료구조별 활용 패턴

Redis를 "빠른 캐시"로만 쓰는 것은 스포츠카를 주차장에 세워두는 것과 같다.

Redis의 진가는 다양한 자료구조를 활용한 **인-메모리 데이터 처리 패턴**에 있다. 각 자료구조는 특정 문제를 O(1)~O(log N)으로 해결하도록 최적화되어 있어, 올바른 자료구조를 선택하면 복잡한 애플리케이션 로직을 데이터베이스 단에서 해결할 수 있다.

### 3-1. String: 카운터, 캐시, 분산 락

String은 Redis의 가장 기본적인 자료구조지만, `INCR`, `SETNX`, `EXPIRE` 조합으로 강력한 패턴을 만든다.

```bash
# 캐시 패턴: 조회 결과를 5분간 캐시
SET product:1234 '{"id":1234,"name":"노트북","price":1500000}' EX 300

# 원자적 카운터: 동시성 문제 없이 방문자 수 증가
INCR page_view:home
INCRBY page_view:home 5  # 한번에 N 증가

# 분산 락: SET NX EX 조합 (Redis 2.6.12+)
SET lock:order_process:user123 "lock_value" NX EX 30
# NX: 키가 없을 때만 설정 (획득 성공 시 1 반환)
# EX 30: 30초 후 자동 해제 (데드락 방지)
```

**분산 락 해제 시 주의사항**: 자신이 건 락만 해제해야 한다. Lua 스크립트로 원자적으로 처리하는 것이 안전하다.

```lua
-- 락 해제 Lua 스크립트 (원자적 실행 보장)
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### 3-2. Hash: 객체 저장과 부분 업데이트

Hash는 키 하나에 여러 필드-값 쌍을 저장한다. 전체 JSON을 String으로 저장하면 한 필드 변경에 전체를 읽고 써야 하지만, Hash를 쓰면 특정 필드만 업데이트할 수 있다.

```bash
# 사용자 세션 정보 저장
HSET session:user123 userId "usr_abc123" role "admin" loginAt "1720000000" cartCount "3"

# 특정 필드만 조회
HGET session:user123 role

# 장바구니 수 업데이트 (다른 필드 건드리지 않음)
HINCRBY session:user123 cartCount 1

# 전체 필드 조회
HGETALL session:user123

# 특정 필드 존재 확인
HEXISTS session:user123 role
```

**Hash의 메모리 최적화**: Redis는 Hash의 필드 수가 128개 이하이고 값이 64바이트 이하일 때 내부적으로 ziplist(compact encoding)를 사용해 메모리를 절약한다. 수백만 개의 소규모 객체를 저장할 때 이 특성을 활용하면 메모리를 크게 줄일 수 있다.

### 3-3. List: 메시지 큐, 작업 대기열

List는 양방향으로 push/pop이 가능한 연결 리스트다. 간단한 메시지 큐나 최근 N개 항목을 유지하는 데 적합하다.

```bash
# 작업 큐: 왼쪽에 추가, 오른쪽에서 꺼냄 (FIFO)
LPUSH job_queue '{"type":"send_email","to":"user@example.com"}'
RPOP job_queue  # 작업자가 폴링으로 가져감

# 블로킹 팝: 큐가 비어있으면 최대 30초 대기 (폴링 대신 블로킹)
BRPOP job_queue 30

# 최근 활동 로그: 최대 100개 유지
LPUSH activity_log:user123 '{"action":"login","ts":1720000000}'
LTRIM activity_log:user123 0 99  # 100개 초과분 자동 삭제

# 최근 활동 조회
LRANGE activity_log:user123 0 9  # 최신 10개
```

**List의 한계**: `LRANGE`는 O(N)이고, 중간 원소 접근이 비효율적이다. 큐의 길이가 매우 길어지면 메모리 문제가 생긴다. 고성능 메시지 큐가 필요하면 Stream을 고려해야 한다.

### 3-4. Set: 중복 제거, 집합 연산

Set은 중복 없는 문자열 컬렉션이다. 추가/삭제/존재 확인이 O(1)이고, 합집합/교집합/차집합을 지원한다.

```bash
# 좋아요 기능: 중복 방지
SADD post_likes:post456 "user123"  # 1 반환 (추가됨)
SADD post_likes:post456 "user123"  # 0 반환 (이미 있음, 중복 방지)
SCARD post_likes:post456           # 좋아요 수

# 특정 사용자가 좋아요 했는지 확인
SISMEMBER post_likes:post456 "user123"

# 공통 팔로워 찾기 (두 사용자의 팔로워 교집합)
SINTERSTORE common_followers followers:userA followers:userB
SMEMBERS common_followers

# 추천 대상: A를 팔로우하지만 B를 팔로우하지 않는 사용자
SDIFFSTORE recommendation followers:userA followers:userB
```

### 3-5. Sorted Set: 실시간 랭킹, 우선순위 큐

Sorted Set은 Set에 score(부동소수점)가 추가된 구조다. score 순으로 정렬된 결과를 O(log N)에 가져올 수 있다. **실시간 랭킹** 구현에 사실상 표준 솔루션이다.

```bash
# 게임 리더보드: score = 점수
ZADD leaderboard 98500 "player:alice"
ZADD leaderboard 87200 "player:bob"
ZADD leaderboard 102000 "player:charlie"

# 점수 업데이트
ZINCRBY leaderboard 5000 "player:alice"  # 103500으로 증가

# 상위 10명 조회 (내림차순)
ZREVRANGE leaderboard 0 9 WITHSCORES

# 특정 플레이어의 순위 확인 (0-based, 내림차순)
ZREVRANK leaderboard "player:alice"

# 점수 범위로 조회: 100000~110000점 사이 플레이어
ZRANGEBYSCORE leaderboard 100000 110000 WITHSCORES

# 우선순위 큐: score = 처리 시간(Unix timestamp)
ZADD delayed_jobs 1720003600 '{"jobId":"job_001","type":"report_generation"}'
# 현재 시간보다 score가 작은 항목 = 처리할 작업
ZRANGEBYSCORE delayed_jobs 0 1720000000 LIMIT 0 10
```

**Sorted Set의 활용 심화 — 페이지네이션 커서**:

```bash
# 무한 스크롤 피드: createdAt timestamp를 score로 사용
ZADD user_feed:user123 1720001000 "post:101"
ZADD user_feed:user123 1720002000 "post:102"
ZADD user_feed:user123 1720003000 "post:103"

# 커서 기반 페이지네이션: 마지막으로 본 포스트 이전 10개
ZREVRANGEBYSCORE user_feed:user123 (1720002000 -inf LIMIT 0 10
# '(' 는 exclusive bound
```

### 3-6. Stream: 이벤트 로그, 소비자 그룹

Redis Stream(5.0+)은 Kafka-lite처럼 동작하는 추가 전용(append-only) 로그 자료구조다. 소비자 그룹(Consumer Group)을 통해 여러 워커가 메시지를 분산 처리할 수 있다.

```bash
# 이벤트 추가 (자동 ID 생성)
XADD user_events * userId user123 eventType page_view page /home

# 소비자 그룹 생성
XGROUP CREATE user_events analytics_group $ MKSTREAM

# 워커가 미처리 메시지 가져오기 (최대 10개)
XREADGROUP GROUP analytics_group worker-1 COUNT 10 STREAMS user_events >

# 처리 완료 확인 (ACK)
XACK user_events analytics_group <message-id>

# 미처리 메시지 확인 (ACK 안 된 것들)
XPENDING user_events analytics_group - + 10
```

Stream은 메시지 유실 없이 at-least-once 처리를 구현할 수 있어, 단순 List 기반 큐의 한계를 넘어선다.

다만 Kafka만큼 확장성이 뛰어나지는 않으므로 초당 수십만 이벤트 이상의 워크로드에서는 Kafka를 고려해야 한다.

### 3-7. Redis 운영 시 주의사항

**`KEYS *` 절대 금지**: 프로덕션에서 `KEYS` 명령은 Redis를 블로킹한다. `SCAN`으로 대체해야 한다.

```bash
# 나쁜 예: 전체 키 목록 (블로킹)
KEYS session:*

# 좋은 예: 커서 기반 스캔 (논블로킹)
SCAN 0 MATCH session:* COUNT 100
# 반환된 커서가 0이 될 때까지 반복
```

**메모리 정책 설정**: Redis가 maxmemory에 도달하면 어떻게 동작할지 정해야 한다.

```
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru  # 전체 키 중 LRU로 제거 (캐시 용도)
# volatile-lru: TTL 있는 키만 LRU로 제거
# noeviction: 에러 반환 (중요 데이터 보관 시)
```

---

## 4. CAP 정리 실전 해석: 이론이 아닌 설계 결정

CAP 정리는 분산 시스템이 다음 세 가지를 동시에 보장할 수 없다는 정리다:

- **C (Consistency)**: 모든 노드가 동시에 같은 데이터를 본다
- **A (Availability)**: 모든 요청이 (성공 또는 실패로) 응답을 받는다
- **P (Partition Tolerance)**: 네트워크 파티션(노드 간 통신 장애)이 발생해도 시스템이 동작한다

### 4-1. "CAP에서 P는 선택이 아니다"

많은 자료에서 "CA, CP, AP 중 하나를 선택한다"고 설명한다. 그러나 이것은 오해를 불러일으킨다.

**네트워크 파티션은 피할 수 없다.** 수십 대, 수백 대의 서버가 연결된 환경에서 파티션이 발생하지 않는다고 가정하는 건 현실적이지 않다.

실질적인 선택은 이것이다: **파티션이 발생했을 때, Consistency를 포기할 것인가 Availability를 포기할 것인가?**

```
파티션 발생 시 선택:

CP (Consistency 우선):
  - 파티션된 노드는 응답 거부 (또는 에러 반환)
  - 데이터 일관성 보장
  - 예: Zookeeper, HBase, MongoDB(기본 설정)

AP (Availability 우선):
  - 파티션된 노드도 (오래된 데이터로라도) 응답
  - 일관성 보장 안 됨 (eventual consistency)
  - 예: Cassandra, CouchDB, DynamoDB(기본 설정)
```

### 4-2. 실전 시나리오: 어떤 선택이 맞는가?

**시나리오 1: 금융 거래 시스템**

계좌 잔액 조회와 이체 처리. 파티션이 발생했을 때 "잔액이 100만원인데, 파티션된 노드가 오래된 데이터로 50만원이라고 응답"하는 상황은 용납할 수 없다. **CP가 올바른 선택이다.** 응답 불가(5xx 에러) 시나리오가 잘못된 데이터보다 낫다.

**시나리오 2: 소셜 미디어 피드**

유저의 뉴스피드 조회. 파티션이 발생했을 때 "3초 전에 올라온 글이 아직 보이지 않는다"는 상황은 허용 가능하다. 피드가 아예 안 보이는 것보다 약간 오래된 피드가 훨씬 낫다. **AP가 올바른 선택이다.**

**시나리오 3: 재고 관리**

e-커머스 재고 차감. "재고가 1개 남았는데 두 노드가 각자 1개씩 판매"하는 oversell 상황은 비즈니스에 따라 허용 가능할 수도 있다. 재고 부족은 나중에 고객 응대로 처리하고, 구매 경험을 우선시하는 서비스라면 **AP + 보상 트랜잭션**이 실용적이다.

### 4-3. PACELC: CAP의 한계를 보완한 모델

CAP 정리의 한계는 "파티션이 없을 때는 어떤가"를 다루지 않는다는 점이다. 네트워크 파티션은 드문 사건이지만 지연시간(Latency)은 매 요청마다 영향을 주기 때문에, 실무에서는 지연시간 vs 일관성 트레이드오프가 더 자주 등장한다. PACELC는 이를 보완한다:

```
PACELC:
  P (파티션 있을 때): A (가용성) vs C (일관성) — CAP과 동일
  E (else, 파티션 없을 때): L (지연시간, Latency) vs C (일관성)

예시:
  - Cassandra: PA/EL — 파티션 시 A 선택, 평상시 Latency 선택
  - DynamoDB(기본): PA/EL — 강한 일관성 읽기 옵션도 있음 (추가 비용)
  - MongoDB: PC/EC — 파티션 시 C 선택, 평상시 C 선택 (읽기 지연 증가 감수)
```

### 4-4. 일관성 모델의 스펙트럼

CAP에서 "C"는 선형성(Linearizability)이라는 강한 일관성을 의미하지만, 실제로는 여러 단계가 있다:

```
강함 ← ──────────────────────────────────────────────── → 약함

Linearizability  →  Sequential  →  Causal  →  Eventual
(강한 일관성)                                  (최종 일관성)

- Linearizability: 모든 연산이 전역 순서로 실행된 것처럼 보임
- Sequential: 각 클라이언트가 자신의 연산 순서는 보장받음
- Causal: 인과 관계 있는 연산의 순서만 보장
- Eventual: 결국엔 수렴하지만 언제인지는 보장 없음
```

MongoDB의 `readConcern: "majority"`는 majority 복제본에 커밋된 데이터만 읽어 강한 일관성을 제공하지만, Cassandra의 `CONSISTENCY ONE`은 최신 데이터가 아닐 수 있다.

### 4-5. 실무 조언: 일관성 문제는 모니터링으로 발견된다

이론은 깔끔하지만, 프로덕션에서 일관성 문제는 종종 예상치 못한 곳에서 터진다. 다음을 모니터링하라:

- **Stale Read 발생률**: 특히 AP 시스템에서 오래된 읽기가 얼마나 자주 발생하는가
- **Replication Lag**: Primary-Secondary 간 복제 지연
- **Split-Brain 이벤트**: 두 노드가 모두 Primary라고 생각하는 상황

---

## 5. RDBMS + NoSQL 혼합 설계 사례

"NoSQL로 전부 갈아엎자"는 접근은 대부분 실패한다.

현실적으로 잘 작동하는 아키텍처는 RDBMS와 NoSQL의 강점을 각자의 자리에서 활용하는 혼합 설계다.

### 5-1. 전형적인 혼합 아키텍처 패턴

```
┌─────────────────────────────────────────────────────────────┐
│                    웹 애플리케이션                            │
└────────────┬──────────────────┬──────────────────┬──────────┘
             │                  │                  │
             ▼                  ▼                  ▼
    ┌─────────────┐    ┌──────────────┐   ┌──────────────┐
    │  PostgreSQL  │    │   MongoDB    │   │    Redis     │
    │  (RDBMS)     │    │  (Document)  │   │  (Cache/     │
    │              │    │              │   │   Session)   │
    │  - 사용자    │    │  - 상품 카탈 │   │  - 세션      │
    │  - 주문      │    │    로그      │   │  - 캐시      │
    │  - 결제      │    │  - 리뷰      │   │  - 랭킹      │
    │  - 재고      │    │  - 콘텐츠    │   │  - 분산락    │
    └─────────────┘    └──────────────┘   └──────────────┘
```

**RDBMS(PostgreSQL)가 담당하는 것**:
- 금전 거래, 재고 차감처럼 ACID 트랜잭션이 필수인 데이터
- 복잡한 집계 쿼리가 필요한 리포팅 데이터
- 정규화된 참조 데이터 (사용자, 상품 마스터)

**MongoDB가 담당하는 것**:
- 스키마가 다양하거나 자주 변하는 상품 속성 데이터
- 계층 구조의 콘텐츠 (블로그, CMS)
- 로그성 데이터, 이벤트 히스토리

**Redis가 담당하는 것**:
- 모든 레이어의 캐시
- 세션 스토어
- 실시간 랭킹, 카운터
- 분산 락

### 5-2. 데이터 일관성 관리: 이중 쓰기의 함정

가장 큰 도전은 **두 데이터베이스 간의 일관성 유지**다. 예를 들어 주문이 생성될 때 PostgreSQL의 `orders` 테이블과 MongoDB의 `order_events` 컬렉션에 동시에 써야 한다면 어떻게 할까?

**나쁜 패턴 — 이중 쓰기**:

```python
# 위험한 이중 쓰기: 하나가 실패하면 불일치 발생
async def create_order(order_data):
    # PostgreSQL에 저장
    await postgres.execute("INSERT INTO orders ...", order_data)
    # MongoDB에 이벤트 저장 — 여기서 실패하면?
    await mongodb.orders_events.insert_one({"type": "ORDER_CREATED", ...})
    # 두 DB가 불일치 상태가 됨!
```

**좋은 패턴 1 — Outbox Pattern**:

```
RDBMS 트랜잭션 안에서:
1. orders 테이블에 주문 INSERT
2. outbox 테이블에 이벤트 INSERT (같은 트랜잭션)

별도 워커(Outbox Poller):
3. outbox 테이블을 주기적으로 폴링
4. MongoDB에 이벤트 전송
5. outbox 레코드 삭제(또는 processed 처리)
```

```sql
-- outbox 테이블
CREATE TABLE outbox (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  VARCHAR(50) NOT NULL,
    payload     JSONB NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    processed   BOOLEAN DEFAULT FALSE
);

-- 주문 생성과 이벤트 발행을 한 트랜잭션에서 처리
BEGIN;
INSERT INTO orders (id, customer_id, total_amount, status)
VALUES ($1, $2, $3, 'PENDING');

INSERT INTO outbox (event_type, payload)
VALUES ('ORDER_CREATED', '{"orderId": $1, "customerId": $2, ...}');
COMMIT;
```

**좋은 패턴 2 — CDC (Change Data Capture)**:

Debezium 같은 CDC 툴을 사용해 RDBMS의 WAL(Write-Ahead Log)을 읽어 변경사항을 Kafka로 스트리밍하고, 컨슈머가 MongoDB나 Elasticsearch에 동기화한다.

```text
┌─────────────┐    WAL     ┌──────────┐   변경 이벤트  ┌─────────────┐
│ PostgreSQL  │──────────→│ Debezium │─────────────→│ Kafka Topic │
│  (소스 DB)  │  스트리밍   │ (CDC 툴)  │              └──────┬──────┘
└─────────────┘            └──────────┘                     │
                                                    ┌───────┴────────┐
                                                    ▼                ▼
                                              ┌──────────┐   ┌──────────────┐
                                              │ MongoDB  │   │Elasticsearch │
                                              │Consumer  │   │  Consumer    │
                                              └──────────┘   └──────────────┘
                                                    ▼
                                              ┌──────────┐
                                              │  Redis   │
                                              │ Consumer │
                                              └──────────┘
```

CDC는 소스 DB에 부하를 주지 않고, 로직 변경 없이 여러 대상 DB를 동기화할 수 있다는 장점이 있다.

### 5-3. 캐시 일관성: Cache-Aside vs Write-Through

Redis를 캐시로 쓸 때 가장 흔한 패턴인 **Cache-Aside(Lazy Loading)**:

```python
async def get_product(product_id: str):
    cache_key = f"product:{product_id}"

    # 1. 캐시 확인
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. DB 조회 (Cache Miss)
    product = await db.fetch_product(product_id)
    if not product:
        return None

    # 3. 캐시 저장 (TTL 10분)
    await redis.setex(cache_key, 600, json.dumps(product))
    return product

async def update_product(product_id: str, data: dict):
    # DB 업데이트
    await db.update_product(product_id, data)
    # 캐시 무효화 (다음 조회 시 새로 로딩)
    await redis.delete(f"product:{product_id}")
```

**Cache-Aside의 Race Condition**:

```
시간 순서:
1. Thread A: DB 조회 (캐시 미스)
2. Thread B: DB 업데이트 후 캐시 삭제
3. Thread A: 오래된 값으로 캐시 SET → 캐시에 오래된 값이 저장됨
```

이 문제의 완전한 해결은 어렵다. 실용적 완화책은 TTL을 짧게 유지하거나, **versioned cache key** (`product:{id}:v{version}`)를 사용하는 것이다.

### 5-4. 실전 사례: e-커머스 상품 페이지

```text
사용자 요청
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                   애플리케이션 서버                    │
│                                                     │
│  ① Redis 캐시 조회 (product:{id})                   │
│      Cache Hit  → 즉시 반환 (< 1ms)                 │
│      Cache Miss → ②로 이동                          │
│                                                     │
│  ② PostgreSQL 조회 → Redis에 캐시 저장               │
│      (상품 기본 정보, 재고 수량)                       │
│                                                     │
│  ③ MongoDB 조회                                     │
│      (상품 상세 속성: 색상, 사이즈, 스펙표)            │
│                                                     │
│  ④ Redis Sorted Set 업데이트                        │
│      ZINCRBY product_views:daily 1 {product_id}    │
│                                                     │
│  ⑤ Redis 세션 확인 (로그인 여부, 장바구니)            │
└─────────────────────────────────────────────────────┘
         │               │               │
         ▼               ▼               ▼
      Redis           MongoDB        PostgreSQL
   (캐시/세션/랭킹)  (상품 속성)   (재고/주문/결제)
```

이 패턴에서 각 DB는 자신이 가장 잘 할 수 있는 역할을 맡는다. PostgreSQL은 트랜잭션이 필요한 재고와 주문을, MongoDB는 유연한 상품 속성을, Redis는 세션과 캐시와 랭킹을.

### 5-5. 혼합 설계의 운영 복잡성

혼합 설계의 대가는 **운영 복잡성**이다:

- 세 개의 DB를 모두 모니터링해야 한다
- 장애 시 원인 파악이 더 어렵다
- 개발자가 세 가지 쿼리 언어와 패턴을 알아야 한다
- 데이터 일관성 검증이 어렵다

이 비용이 성능/유연성 이득보다 클 경우, PostgreSQL 하나만 쓰는 것이 더 나은 선택일 수 있다.

**MongoDB를 선택하기 전에, PostgreSQL의 JSONB 컬럼으로 같은 문제를 해결할 수 없는지 먼저 검토하라.** PostgreSQL JSONB는 Document Store의 많은 사용 사례를 커버하면서 ACID 트랜잭션도 유지할 수 있다.

```sql
-- PostgreSQL JSONB로 MongoDB와 유사한 쿼리 가능
CREATE TABLE products (
    id      UUID PRIMARY KEY,
    data    JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_products_data ON products USING GIN (data);

-- 특정 속성으로 검색
SELECT * FROM products
WHERE data @> '{"category": "electronics"}';

-- 중첩 속성 쿼리
SELECT data->>'name', (data->>'price')::numeric
FROM products
WHERE (data->'specs'->>'ram')::text = '16GB';
```

---

## 마치며

NoSQL의 핵심은 "더 빠른 데이터베이스"가 아니라 **"다른 트레이드오프를 선택한 데이터베이스"**다.

Document Store는 조인 대신 중복을 선택했고, Key-Value Store는 쿼리 유연성 대신 성능을 선택했으며, Cassandra는 일관성 대신 가용성과 확장성을 선택했다.

좋은 엔지니어는 이 트레이드오프를 이해하고 비즈니스 요구사항에 맞는 선택을 한다. 나쁜 엔지니어는 트렌드에 따라 선택하거나, 모든 문제를 새로 배운 망치로 두드린다.

MongoDB는 망치가 아니라 특정 문제에 최적화된 도구다. Redis도, Cassandra도 마찬가지다.

다음 편에서는 검색 엔진으로 종종 NoSQL로 분류되는 Elasticsearch를 다룬다. 전문 검색(Full-Text Search)의 내부 동작 원리와, Elasticsearch가 로그 분석 플랫폼(ELK Stack)으로 진화한 이유를 살펴볼 것이다.

---

## 참고 자료

1. **Designing Data-Intensive Applications** - Martin Kleppmann (O'Reilly, 2017): 분산 시스템과 데이터베이스 내부 동작의 바이블. CAP/PACELC, 일관성 모델, 복제 등을 깊이 있게 다룬다.

2. **MongoDB: The Definitive Guide** - Shannon Bradshaw, Eoin Brazil, Kristina Chodorow (O'Reilly, 3rd Edition): MongoDB 공식 가이드. 데이터 모델링 챕터는 설계 결정의 기준을 제공한다.

3. **Redis in Action** - Josiah L. Carlson (Manning, 2013): Redis 자료구조별 활용 패턴을 실전 예시로 설명한다. 오래된 책이지만 핵심 패턴은 여전히 유효하다.

4. **NoSQL Distilled** - Pramod J. Sadalage, Martin Fowler (Addison-Wesley, 2012): NoSQL 유형을 체계적으로 분류하고 각 유형의 트레이드오프를 비교한다. Martin Fowler의 명료한 서술이 돋보인다.

5. **Cassandra: The Definitive Guide** - Jeff Carpenter, Eben Hewitt (O'Reilly, 3rd Edition): Wide-Column Store의 데이터 모델링과 운영을 다룬다. 쿼리 패턴 기반 테이블 설계 원칙이 핵심이다.

6. **CAP Twelve Years Later: How the "Rules" Have Changed** - Eric Brewer (IEEE Computer, 2012): CAP 정리를 만든 Brewer가 직접 쓴 재해석 논문. "C와 A는 연속적인 스펙트럼"이라는 관점을 제시한다.
