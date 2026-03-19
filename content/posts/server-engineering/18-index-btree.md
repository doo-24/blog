---
title: "[데이터베이스] 1편 — 인덱스: B-Tree가 실제로 하는 일"
date: 2026-03-17T23:09:00+09:00
draft: false
tags: ["인덱스", "B-Tree", "데이터베이스", "MySQL", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "B-Tree 구조와 탐색/삽입/삭제 원리, 클러스터드/논클러스터드 인덱스, InnoDB 페이지 구조, 복합 인덱스 설계와 선행 컬럼 규칙, 커버링 인덱스, EXPLAIN 읽기까지"
---

데이터베이스 성능 문제의 80%는 인덱스 문제다. 이것은 과장이 아니다. 쿼리가 느리다는 티켓이 들어왔을 때, 가장 먼저 열어봐야 하는 것은 `EXPLAIN` 출력이고, 거기서 `type: ALL`이 보이는 순간 이미 답은 나와 있다. 그런데 많은 엔지니어들이 인덱스를 "컬럼에 붙이는 마법"처럼 사용한다. 어떤 인덱스를 왜 만들어야 하는지, 왜 어떤 쿼리는 인덱스가 있어도 풀스캔을 하는지 정확히 설명하지 못한다. 이 글은 그 간격을 메운다. B-Tree가 내부적으로 어떻게 동작하는지부터, InnoDB의 페이지 구조, 복합 인덱스 설계 원칙, 커버링 인덱스까지 — 실제 MySQL에서 일어나는 일을 바닥부터 짚어본다.

---

## 1. B-Tree: 왜 이 자료구조인가

### 디스크와 메모리의 근본적인 차이

인덱스를 이해하려면 먼저 디스크 I/O의 특성을 알아야 한다. 메모리 접근은 임의 접근(random access)이 빠르다. 포인터 하나를 따라가는 비용이 거의 없다. 하지만 디스크는 다르다. HDD는 헤드가 물리적으로 이동해야 하고, SSD도 페이지 단위로 읽는다. 결정적인 특성은 **디스크는 한 번 I/O할 때 단일 바이트가 아니라 블록 단위(보통 4KB~16KB)로 읽는다**는 것이다.

이 특성 때문에 자료구조 선택이 달라진다. 메모리라면 이진 탐색 트리(BST)가 O(log N) 탐색을 제공하므로 훌륭하다. 하지만 BST를 디스크에 올리면 노드 하나를 읽을 때마다 디스크 I/O가 발생한다. 100만 개 레코드를 가진 BST의 높이는 약 20이다. 즉 탐색 하나에 최대 20번의 디스크 I/O가 발생한다. SSD라도 랜덤 I/O 하나에 0.1ms라면, 20번이면 2ms다. 초당 500개 쿼리도 처리 못 한다.

B-Tree는 이 문제를 해결하기 위해 설계됐다. 핵심 아이디어는 **각 노드가 여러 개의 키와 자식 포인터를 가지도록 트리의 분기 계수(branching factor)를 극대화**하는 것이다. 분기 계수가 크면 트리 높이가 낮아지고, 디스크 I/O 횟수가 줄어든다.

### B-Tree의 구조

B-Tree에서 차수(order) M은 "하나의 노드가 최대 M개의 자식을 가질 수 있다"는 의미다. 실제 MySQL InnoDB는 16KB 페이지를 사용하고, 각 페이지(노드)에 수백 개의 키를 저장한다. 이렇게 하면 수억 개의 레코드도 트리 높이가 3~4 정도에 불과하다.

```
                    [  30  |  60  ]
                   /         |         \
          [10|20]         [40|50]       [70|80|90]
         /  |  \          /  |  \       /  |  |  \
       [.] [.] [.]      [.] [.] [.]  [.] [.] [.] [.]
```

B-Tree의 핵심 속성:

1. **모든 리프 노드는 같은 깊이에 있다** — 균형 트리(balanced tree)다.
2. **루트 노드를 제외한 모든 노드는 최소 ⌈M/2⌉개의 키를 가진다** — 낭비를 방지한다.
3. **내부 노드의 키는 자식 노드들의 키 범위를 구분하는 경계값이다.**

MySQL InnoDB가 실제로 사용하는 것은 B-Tree의 변형인 **B+Tree**다. B+Tree는 B-Tree와 다음 점에서 다르다:

- **실제 데이터(또는 포인터)는 리프 노드에만 존재한다.** 내부 노드는 라우팅 키만 가진다.
- **모든 리프 노드는 연결 리스트로 연결된다.** 이것이 범위 스캔을 효율적으로 만든다.

```
내부 노드:  [30 | 60]  <-- 라우팅 키만, 실제 데이터 없음
            /         \
리프 노드: [10→20→30] ↔ [40→50→60] ↔ [70→80→90]
           실제 데이터 저장, 양방향 연결
```

범위 쿼리 `WHERE age BETWEEN 30 AND 70`을 생각해보자. B-Tree라면 각 값을 개별적으로 찾아야 한다. B+Tree는 30을 리프에서 찾은 다음, 연결 리스트를 따라 70까지 순차적으로 읽으면 된다. 순차 I/O는 랜덤 I/O보다 훨씬 빠르다.

### 탐색 연산

탐색은 단순하다. 루트에서 시작해서 찾는 키가 현재 노드의 어느 범위에 속하는지 판단하고, 해당 자식으로 내려간다. 리프에 도달하면 탐색 종료.

```
찾는 값: 45

루트 [30 | 60] → 30 < 45 < 60이므로 가운데 자식으로
내부 [40 | 50] → 40 < 45 < 50이므로 가운데 자식으로
리프 [42 | 45 | 48] → 45 발견!
```

시간 복잡도는 O(log_M N)이다. M이 크면 클수록(각 노드에 키가 많을수록) 높이가 낮아진다. 1억 개 레코드, M=200이라면 높이는 약 4다. 디스크 I/O 4번으로 탐색 완료.

### 삽입 연산

삽입은 항상 리프에서 일어난다. 문제는 리프가 가득 찼을 때다.

```
삽입 전:
리프 [10 | 20 | 30] (최대 3개)

35 삽입 시도 → 노드가 가득 참 → 분할(split) 발생

분할 후:
[20]       ← 부모 노드로 올라간 중앙값
/     \
[10]   [20 | 30 | 35]
```

이 분할이 연쇄적으로 부모까지 전파될 수 있다. 부모도 가득 찼다면 부모도 분할한다. 루트가 분할되면 트리의 높이가 1 증가한다. **B-Tree는 루트에서 분할이 발생할 때만 높이가 증가한다** — 이것이 항상 균형을 유지하는 이유다.

삽입이 많으면 페이지 분할이 잦아지고, 이는 쓰기 성능을 저하시킨다. **UUID를 기본키로 사용하면 안 된다**는 말을 들어봤을 것이다. UUID는 무작위 순서이기 때문에 삽입 위치가 기존 리프 어디에나 될 수 있다. 이미 가득 찬 리프 중간에 삽입해야 하는 경우가 빈번하게 발생해 분할이 폭발적으로 증가한다. AUTO_INCREMENT를 사용하면 항상 마지막 리프에만 삽입되므로 분할이 최소화된다.

### 삭제 연산

삭제도 리프에서 일어난다. 리프에서 키를 제거한 후, 노드에 남은 키가 최소 개수(⌈M/2⌉ - 1)보다 적어지면 **재분배(redistribution)** 또는 **병합(merge)**이 발생한다.

- **재분배**: 인접 형제 노드에 키가 여유 있으면 빌려온다.
- **병합**: 형제 노드도 최소라면 두 노드를 합친다. 부모에서 분리 키를 제거하고, 이게 연쇄될 수 있다.

실무에서 삭제는 보통 즉각적인 병합을 하지 않고 "삭제 마킹(soft delete)"을 사용하는 경우도 많다. InnoDB는 삭제된 레코드 공간을 재사용 가능한 상태로 표시하고 나중에 재사용한다.

---

## 2. 클러스터드 인덱스와 InnoDB 페이지 구조

### InnoDB의 페이지

InnoDB는 데이터를 **페이지(page)** 단위로 관리한다. 기본 페이지 크기는 16KB다. 모든 것 — 데이터, 인덱스, 롤백 로그 등 — 이 페이지로 저장된다.

페이지의 내부 구조를 간략히 보면:

```
┌─────────────────────────────────────┐
│  File Header (38 bytes)             │ ← 페이지 번호, 이전/다음 페이지 포인터
├─────────────────────────────────────┤
│  Page Header (56 bytes)             │ ← 레코드 수, 슬롯 수, 힙 크기
├─────────────────────────────────────┤
│  Infimum + Supremum records         │ ← 경계 레코드 (최소/최대 센티널)
├─────────────────────────────────────┤
│  User Records                       │ ← 실제 데이터
│  (가변 길이, 레코드들이 연결 리스트)   │
├─────────────────────────────────────┤
│  Free Space                         │ ← 사용 가능한 공간
├─────────────────────────────────────┤
│  Page Directory                     │ ← 슬롯 배열 (이진 탐색용)
├─────────────────────────────────────┤
│  File Trailer (8 bytes)             │ ← 체크섬
└─────────────────────────────────────┘
```

페이지 내부에서 레코드들은 키 순서로 정렬된 **단방향 연결 리스트**로 저장된다. 페이지 디렉토리는 일정 간격으로 레코드를 가리키는 슬롯을 가져서 페이지 내 이진 탐색을 가능하게 한다.

### 클러스터드 인덱스 (Clustered Index)

InnoDB의 가장 중요한 특성 중 하나다. **클러스터드 인덱스는 테이블 자체다**. 데이터가 기본키 순서로 리프 페이지에 물리적으로 정렬되어 저장된다. 별도의 힙(heap) 파일이 없다.

```
클러스터드 인덱스 B+Tree:

         [PK:100 | PK:300]
         /                 \
[PK:50|PK:75|PK:100]   [PK:200|PK:300|PK:400]
  row data    row data      row data    row data
```

리프 노드에 실제 행 데이터 전체가 저장된다. 기본키로 행을 찾으면 I/O 한 번(리프 페이지 하나)으로 모든 컬럼을 읽을 수 있다.

InnoDB가 클러스터드 인덱스를 결정하는 우선순위:
1. `PRIMARY KEY`로 명시된 컬럼
2. 없으면 `NOT NULL`이고 `UNIQUE`한 첫 번째 인덱스
3. 둘 다 없으면 InnoDB가 내부적으로 숨겨진 6바이트 row ID를 생성

기본키가 없는 테이블을 만드는 것은 나쁜 습관이다. 숨겨진 row ID는 외부에서 접근 불가능하고, 세컨더리 인덱스에서 참조하기 비효율적이다.

### 논클러스터드 인덱스 (Secondary Index)

세컨더리 인덱스는 클러스터드 인덱스와 별개의 B+Tree다. 핵심적인 차이는 **리프 노드에 실제 데이터 대신 해당 행의 기본키 값이 저장된다**는 것이다.

```
세컨더리 인덱스 (name 컬럼):

       ["Kim" | "Park"]
      /                  \
["Ahn"|"Choi"|"Kim"]  ["Lee"|"Park"|"Song"]
  PK:75  PK:200 PK:50   PK:100 PK:300 PK:400
    ↑       ↑      ↑
  기본키 값을 저장 (실제 데이터 없음)
```

`WHERE name = 'Kim'`으로 조회하면:
1. 세컨더리 인덱스에서 'Kim'을 찾아 PK:50을 얻는다.
2. PK:50으로 클러스터드 인덱스를 다시 탐색해서 실제 행을 읽는다.

이 두 번째 단계를 **북마크 룩업(Bookmark Lookup)** 또는 **클러스터드 인덱스 룩업**이라고 한다. 세컨더리 인덱스 탐색에서 클러스터드 인덱스로 다시 들어가는 이 과정이 추가 비용이다.

이것이 InnoDB에서 기본키를 작게 유지해야 하는 이유다. 기본키가 크면(예: UUID 16바이트), 모든 세컨더리 인덱스가 기본키를 복사하여 저장하므로 인덱스 크기가 폭발적으로 증가한다.

### MyISAM의 차이

비교를 위해 MyISAM을 보자. MyISAM은 힙 파일에 데이터를 저장하고, 모든 인덱스(프라이머리 포함)가 물리적인 레코드 위치(파일 오프셋)를 가리킨다. 클러스터드 인덱스 개념이 없다. InnoDB와 달리 프라이머리 키와 세컨더리 키의 리프 구조가 동일하다.

| 특성 | InnoDB (클러스터드) | MyISAM (힙) |
|------|---------------------|-------------|
| 기본키 탐색 | 1회 B+Tree 탐색 | 1회 탐색 + 파일 오프셋 접근 |
| 세컨더리 인덱스 | 2회 B+Tree 탐색 (+ 커버링 예외) | 1회 탐색 + 파일 오프셋 접근 |
| 범위 스캔 | 순차 페이지 읽기 (빠름) | 랜덤 파일 오프셋 (느림) |
| 기본키 크기 영향 | 세컨더리 인덱스에 전파 | 없음 |

---

## 3. 복합 인덱스 설계와 선행 컬럼 규칙

### 복합 인덱스의 내부 구조

복합 인덱스(composite index)는 두 개 이상의 컬럼으로 만든 인덱스다. `INDEX (last_name, first_name, age)` 같은 식이다.

내부적으로는 여전히 B+Tree 하나다. 차이는 키가 여러 컬럼의 조합이라는 것이다. 정렬 순서는 첫 번째 컬럼 기준으로 정렬하고, 같은 값이면 두 번째 컬럼 기준, 같은 값이면 세 번째... 식이다.

```
INDEX (last_name, first_name):

리프 노드 예시:
[("Choi", "Alice") | ("Choi", "Bob") | ("Kim", "Alice") | ("Kim", "Zoe") | ("Lee", "Mike")]
```

### 선행 컬럼 규칙 (Leftmost Prefix Rule)

이것이 복합 인덱스에서 가장 중요한 규칙이다: **인덱스를 사용하려면 항상 첫 번째(선행) 컬럼이 조건에 포함되어야 한다**.

`INDEX (a, b, c)`가 있을 때:

```sql
-- 인덱스 사용 가능
WHERE a = 1                    -- a만 사용
WHERE a = 1 AND b = 2          -- a, b 사용
WHERE a = 1 AND b = 2 AND c = 3 -- a, b, c 모두 사용
WHERE a = 1 AND c = 3          -- a는 사용, c는 못 씀 (b를 건너뜀)

-- 인덱스 사용 불가
WHERE b = 2                    -- a 없음
WHERE b = 2 AND c = 3          -- a 없음
WHERE c = 3                    -- a 없음
```

`WHERE a = 1 AND c = 3`의 경우, `a`로 인덱스를 좁힌 다음 `c`에 대해서는 인덱스를 사용하지 못하고 해당 범위에서 필터링만 한다.

왜 이런 규칙이 생기는가? B+Tree의 정렬 방식 때문이다. `(last_name, first_name)` 인덱스에서 `first_name = 'Alice'`로만 찾으려면 모든 last_name 값에 대한 Alice를 찾아야 한다. 이들은 인덱스에서 물리적으로 흩어져 있다. 연속 범위를 이용한 탐색이 불가능하다.

### 범위 조건과 인덱스 중단

`INDEX (a, b, c)`에서:

```sql
WHERE a = 1 AND b > 5 AND c = 10
```

이 쿼리에서 `a`와 `b`는 인덱스를 사용하지만 `c`는 사용하지 못한다. `b > 5` 같은 범위 조건 이후의 컬럼은 인덱스 탐색에서 배제된다.

이유: `b > 5`를 만족하는 범위 내에서 `c` 값은 다시 정렬되어 있지 않다. 예를 들어 `(a=1, b=6, c=10)`, `(a=1, b=6, c=2)`, `(a=1, b=7, c=8)` 순으로 저장될 수 있다. c의 순서가 보장되지 않으므로 B-Tree 탐색을 할 수 없다.

실무 규칙: **범위 조건 컬럼은 복합 인덱스에서 가능한 뒤쪽에 배치한다.**

```sql
-- 나쁜 설계: (status, created_at, user_id)
-- WHERE user_id = 1 AND status = 'active' AND created_at > '2024-01-01'
-- status와 created_at(범위)으로만 인덱스를 사용하고 user_id는 필터링

-- 좋은 설계: (user_id, status, created_at)
-- user_id = 1, status = 'active' 두 컬럼을 정확히 사용, created_at 범위 마지막
```

### 카디널리티와 인덱스 선택도

**카디널리티(cardinality)**는 컬럼의 고유값 수다. `gender` 컬럼은 카디널리티가 2~3이고, `user_id`는 수백만이다.

카디널리티가 높을수록 인덱스의 선택도(selectivity)가 좋다. 선택도 = 고유값 수 / 전체 행 수. 선택도가 1에 가까울수록 인덱스가 효과적이다.

```sql
-- 카디널리티 확인
SELECT COUNT(DISTINCT column_name) / COUNT(*) AS selectivity
FROM table_name;
```

선택도가 낮은 컬럼(예: boolean, status 같은 enum)에 단독 인덱스를 만드는 것은 의미 없다. `status = 'active'`가 전체 행의 90%를 반환한다면, MySQL 옵티마이저는 인덱스를 사용하지 않고 풀스캔을 선택한다. 인덱스 탐색 + 북마크 룩업 비용이 순차 스캔보다 더 비싸기 때문이다.

**복합 인덱스에서의 컬럼 순서:**

일반적인 원칙은:
1. **동등 조건(=) 컬럼을 앞에** 배치한다.
2. **선택도가 높은 컬럼을 앞에** 배치한다.
3. **범위 조건(>, <, BETWEEN, LIKE) 컬럼을 뒤에** 배치한다.

하지만 1번과 2번이 충돌할 때는 실제 쿼리 패턴에 따라 결정해야 한다. `WHERE status = 'active' AND user_id = 1`이라면, `user_id`가 선택도는 높지만 동등 조건이므로 `(user_id, status)` 또는 `(status, user_id)` 모두 동일하게 두 컬럼을 사용한다. 이 경우는 실제 데이터 분포로 판단한다.

### 인덱스 병합 (Index Merge)

MySQL은 여러 인덱스를 동시에 사용하는 **인덱스 병합(Index Merge)** 기능을 지원한다.

```sql
-- a와 b에 각각 인덱스가 있을 때
WHERE a = 1 OR b = 2
```

각 인덱스로 결과 집합을 구하고 합집합(union) 또는 교집합(intersection)을 만든다. `EXPLAIN`에서 `type: index_merge`로 나타난다.

이것은 복합 인덱스가 없을 때의 차선책이다. 인덱스 병합은 두 번의 B-Tree 탐색과 결과 합산 비용이 발생한다. 빈번히 함께 조회되는 컬럼이라면 복합 인덱스를 만드는 것이 훨씬 효율적이다.

---

## 4. 커버링 인덱스와 인덱스 사용 불가 패턴

### 커버링 인덱스 (Covering Index)

앞서 세컨더리 인덱스 탐색은 두 단계라고 했다. 인덱스 탐색 후 클러스터드 인덱스로 북마크 룩업. 그런데 **쿼리가 필요한 모든 컬럼이 인덱스에 포함되어 있다면** 두 번째 단계가 필요 없다. 인덱스만으로 결과를 반환할 수 있다. 이것이 커버링 인덱스다.

```sql
-- 테이블: users(id, email, name, age, ...)
-- 인덱스: INDEX (email, name)

-- 커버링 인덱스 작동: email, name만 조회
SELECT name FROM users WHERE email = 'user@example.com';
-- 인덱스 리프에 email과 name 모두 있음. 클러스터드 인덱스 불필요.

-- 커버링 인덱스 작동 안 함: age 필요
SELECT name, age FROM users WHERE email = 'user@example.com';
-- age가 인덱스에 없으므로 클러스터드 인덱스 룩업 필요.
```

`EXPLAIN`에서 `Extra: Using index`가 보이면 커버링 인덱스가 작동 중이다. 이것은 매우 좋은 신호다.

커버링 인덱스는 읽기 성능을 극적으로 향상시킨다. 인덱스 페이지는 데이터 페이지보다 훨씬 작고(기본키와 인덱스 컬럼만 저장하므로), 버퍼 풀에 더 많이 캐시된다. 디스크 I/O가 절반 이하로 줄어드는 경우가 흔하다.

### 인덱스를 포함한 COUNT 최적화

```sql
-- users 테이블에 INDEX (status)가 있을 때
SELECT COUNT(*) FROM users WHERE status = 'active';
```

이 쿼리는 인덱스를 사용한다. InnoDB는 커버링 인덱스를 통해 클러스터드 인덱스를 건드리지 않고 결과를 낼 수 있다. status 인덱스 리프만 스캔하면 된다. `EXPLAIN`에서 `Extra: Using index`로 확인된다.

반면 `COUNT(*)`는 `COUNT(column)`과 다르다. `COUNT(*)`는 NULL을 포함한 모든 행을 센다. `COUNT(column)`은 NULL이 아닌 값만 센다. 성능 차이는 MySQL 버전과 옵티마이저에 따라 다르지만, 일반적으로 `COUNT(*)`가 최적화되어 있다.

### 인덱스 사용 불가 패턴

이것이 실무에서 가장 자주 실수하는 부분이다.

#### 1) 함수 적용

```sql
-- 인덱스 사용 불가: created_at에 함수 적용
WHERE YEAR(created_at) = 2024
WHERE DATE(created_at) = '2024-01-01'

-- 인덱스 사용 가능: 범위로 변환
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
WHERE created_at >= '2024-01-01' AND created_at < '2024-01-02'
```

컬럼에 함수를 적용하면 인덱스가 있는 원래 컬럼값과 비교할 수 없어서 인덱스를 사용하지 못한다.

```sql
-- 인덱스 사용 불가: 컬럼 변환
WHERE UPPER(email) = 'USER@EXAMPLE.COM'

-- 인덱스 사용 가능: 비교값을 변환
WHERE email = LOWER('USER@EXAMPLE.COM')
-- 또는 함수 기반 인덱스(MySQL 8.0+)
CREATE INDEX idx_email_upper ON users ((UPPER(email)));
```

#### 2) 묵시적 타입 변환

```sql
-- phone_number가 VARCHAR인데 숫자로 비교
WHERE phone_number = 01012345678  -- 문자열 vs 숫자 → 타입 변환 → 인덱스 미사용

-- 올바른 방법
WHERE phone_number = '01012345678'
```

이것은 매우 흔한 실수다. 특히 ORM에서 파라미터 타입을 잘못 바인딩하면 조용히 인덱스를 사용하지 않는다.

#### 3) 부정 연산자와 NOT IN

```sql
-- 인덱스 사용 어려움
WHERE status != 'inactive'
WHERE id NOT IN (1, 2, 3)
```

부정 조건은 원칙적으로 B-Tree로 효율적으로 탐색할 수 없다. 전체를 스캔해야 하기 때문이다. 데이터 분포에 따라 옵티마이저가 인덱스를 사용할 수도 있지만, 보통은 풀스캔에 가깝다.

#### 4) LIKE 앞 와일드카드

```sql
-- 인덱스 사용 불가: 앞에 %
WHERE name LIKE '%홍길동'
WHERE name LIKE '%길동%'

-- 인덱스 사용 가능: 뒤에만 %
WHERE name LIKE '홍길동%'
```

B-Tree는 앞에서부터 키를 비교한다. `%`로 시작하면 어디서 시작할지 알 수 없으므로 전체 스캔이 필요하다. 앞 와일드카드 검색이 필요하다면 전문 검색 인덱스(Full-Text Index)나 Elasticsearch 같은 별도 솔루션을 고려해야 한다.

#### 5) OR 조건

```sql
-- 복합 인덱스 (a, b)가 있을 때
WHERE a = 1 OR b = 2  -- 인덱스 병합이 아니면 풀스캔
```

OR은 인덱스 사용을 복잡하게 만든다. `a`에 대한 인덱스와 `b`에 대한 인덱스가 별도로 있으면 인덱스 병합이 가능하지만, 없으면 풀스캔이다. UNION으로 분리하는 게 명확하다:

```sql
SELECT * FROM table WHERE a = 1
UNION ALL
SELECT * FROM table WHERE b = 2 AND a != 1
```

#### 6) SELECT *

직접적으로 인덱스를 못 쓰게 하는 건 아니지만, 커버링 인덱스를 불가능하게 만든다. 항상 필요한 컬럼만 명시적으로 선택하는 습관이 성능에 도움이 된다.

---

## 5. EXPLAIN 읽기

### EXPLAIN 기본

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

주요 컬럼 설명:

| 컬럼 | 의미 |
|------|------|
| `type` | 접근 방식. ALL이 최악, const가 최선 |
| `key` | 실제 사용된 인덱스 |
| `key_len` | 사용된 인덱스 바이트 수 |
| `rows` | 스캔 예상 행 수 (통계 기반 추정값) |
| `filtered` | rows 중 WHERE 조건 통과 예상 비율 |
| `Extra` | 추가 정보 |

### type 컬럼 이해

`type`은 쿼리 성능을 한눈에 판단하는 가장 중요한 컬럼이다.

```
성능 순서 (좋음 → 나쁨):
const > eq_ref > ref > range > index > ALL
```

- **const**: PK 또는 UNIQUE 인덱스로 하나의 행을 정확히 조회. 최적이다.
  ```sql
  WHERE id = 1  -- PK 조회
  ```

- **eq_ref**: JOIN에서 PK 또는 UNIQUE NOT NULL 인덱스를 사용. 각 행마다 정확히 하나의 행을 읽는다.
  ```sql
  SELECT * FROM a JOIN b ON a.id = b.a_id  -- b.a_id가 PK이면 eq_ref
  ```

- **ref**: UNIQUE가 아닌 인덱스를 사용한 동등 조건. 여러 행이 매칭될 수 있다.
  ```sql
  WHERE status = 'active'  -- status에 일반 인덱스
  ```

- **range**: 인덱스를 사용한 범위 스캔. `BETWEEN`, `>`, `<`, `IN` 등.
  ```sql
  WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'
  ```

- **index**: 인덱스 풀스캔. 데이터 파일 대신 인덱스 전체를 읽는다. ALL보다 낫지만 여전히 느리다.

- **ALL**: 테이블 풀스캔. 최악. 작은 테이블이나 대부분의 행을 읽어야 하면 오히려 합리적일 수 있다.

### key_len으로 복합 인덱스 분석

`key_len`은 인덱스에서 실제로 사용된 바이트 수다. 복합 인덱스에서 몇 번째 컬럼까지 사용됐는지 역산할 수 있다.

```sql
-- INDEX (a INT, b VARCHAR(50))
-- INT = 4 bytes, VARCHAR(50) with utf8mb4 = 50*4 + 2(length prefix) = 202 bytes

EXPLAIN SELECT * FROM t WHERE a = 1;
-- key_len: 4 → a만 사용

EXPLAIN SELECT * FROM t WHERE a = 1 AND b = 'hello';
-- key_len: 206 → a + b 모두 사용
```

NULL 허용 컬럼은 1바이트가 추가된다(NULL 플래그). `INT NOT NULL`은 4, `INT NULL`은 5다.

### Extra 컬럼 해석

- **Using index**: 커버링 인덱스. 매우 좋다.
- **Using where**: 인덱스 탐색 후 추가 WHERE 필터링. 인덱스가 일부만 사용됨을 의미할 수 있다.
- **Using filesort**: 정렬을 위해 별도의 정렬 작업이 필요. 인덱스로 정렬을 해결하지 못한 경우.
- **Using temporary**: 임시 테이블 사용. GROUP BY, ORDER BY, DISTINCT 등에서 발생. 느리다.
- **Using index condition**: 인덱스 조건 푸시다운(ICP, Index Condition Pushdown). MySQL 5.6+에서 지원. 스토리지 엔진 레벨에서 인덱스 조건을 미리 필터링해서 불필요한 행 읽기를 줄인다.
- **Using MRR**: Multi-Range Read Optimization. 세컨더리 인덱스로 기본키 목록을 모아 정렬한 후 순차적으로 클러스터드 인덱스를 읽어 랜덤 I/O를 줄인다.

### EXPLAIN ANALYZE (MySQL 8.0.18+)

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email LIKE 'test%';
```

`EXPLAIN ANALYZE`는 실제로 쿼리를 실행하고 실제 소요 시간과 실제 행 수를 보고한다. `EXPLAIN`만으로는 통계 기반 추정값만 보이지만, `ANALYZE`는 실제 값을 보여준다. 성능 병목을 정확히 찾는 데 필수적이다.

```
-> Filter: (users.email like 'test%')  (actual time=0.045..15.234 rows=128 loops=1)
    -> Index range scan on users using idx_email  (actual time=0.032..12.456 rows=128 loops=1)
```

`actual time=시작..종료 rows=실제행수 loops=반복횟수` 형식이다.

### 실전 EXPLAIN 분석 예시

```sql
CREATE TABLE orders (
    id BIGINT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    status VARCHAR(20) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    created_at DATETIME NOT NULL,
    INDEX idx_user_status_created (user_id, status, created_at)
);

-- 쿼리 1: 좋은 경우
EXPLAIN SELECT id, amount
FROM orders
WHERE user_id = 123 AND status = 'paid'
ORDER BY created_at DESC
LIMIT 10;
```

예상 결과:
- `type: ref` — user_id, status 동등 조건으로 인덱스 사용
- `key: idx_user_status_created`
- `key_len: 인덱스 전체` — 세 컬럼 모두 사용
- `Extra: Using index` — amount는 없지만 id는 PK로 커버됨? 아니다. amount는 인덱스에 없으므로 `Using where`가 될 수 있다.

```sql
-- 커버링 인덱스로 만들려면:
CREATE INDEX idx_covering ON orders (user_id, status, created_at, amount);

-- 또는 SELECT를 조정:
SELECT id, created_at FROM orders
WHERE user_id = 123 AND status = 'paid'
ORDER BY created_at DESC
LIMIT 10;
-- 여기서 id는 클러스터드 인덱스 키이므로 세컨더리 인덱스 리프에 포함됨 → 커버링
```

```sql
-- 쿼리 2: 나쁜 경우
EXPLAIN SELECT * FROM orders WHERE status = 'paid';
-- type: ALL (user_id가 없어서 인덱스 앞 컬럼 불충족)
-- status 카디널리티가 낮으므로 단독 인덱스도 효과 없을 수 있음
```

---

## 6. 인덱스 설계 실전 가이드

### 인덱스를 만들 때의 비용

인덱스는 읽기를 빠르게 하지만 쓰기를 느리게 한다. `INSERT`, `UPDATE`, `DELETE` 시마다 관련된 모든 인덱스를 업데이트해야 한다. 인덱스가 5개인 테이블에 레코드를 삽입하면 6개의 B-Tree를 갱신한다(클러스터드 포함).

또한 인덱스는 스토리지를 차지한다. 대형 테이블에서는 인덱스 크기가 데이터 크기를 초과하는 경우도 있다. 인덱스가 많으면 버퍼 풀 효율도 떨어진다.

원칙: **필요한 인덱스만 만들고, 사용하지 않는 인덱스는 제거한다.**

```sql
-- 사용되지 않는 인덱스 찾기 (MySQL 8.0+)
SELECT * FROM sys.schema_unused_indexes;

-- 또는 Performance Schema
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
AND count_star = 0
AND object_schema = 'your_database';
```

### 인덱스 힌트와 옵티마이저 우회

옵티마이저가 잘못된 인덱스를 선택할 때가 있다. 통계가 오래됐거나 데이터 분포가 특수한 경우다.

```sql
-- 강제로 인덱스 사용
SELECT * FROM users FORCE INDEX (idx_email) WHERE email = 'test@example.com';

-- 인덱스 사용 금지
SELECT * FROM users IGNORE INDEX (idx_status) WHERE status = 'active';

-- 힌트 (MySQL 8.0+, 더 권장되는 방식)
SELECT /*+ INDEX(users idx_email) */ * FROM users WHERE email = 'test@example.com';
```

인덱스 힌트는 임시방편이다. 근본 원인(통계 업데이트, 인덱스 재설계)을 먼저 확인해야 한다.

```sql
-- 통계 갱신
ANALYZE TABLE users;
```

### 파티셔닝과 인덱스

파티셔닝된 테이블에서는 인덱스가 파티션별로 독립적으로 존재한다(로컬 인덱스). 글로벌 인덱스는 MySQL에서 지원하지 않는다. 파티션 프루닝이 효과적이려면 파티션 키가 WHERE 조건에 포함되어야 한다.

```sql
-- 파티셔닝 예시
CREATE TABLE logs (
    id BIGINT AUTO_INCREMENT,
    created_at DATETIME NOT NULL,
    message TEXT,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- 이 쿼리는 p2024 파티션만 스캔 (파티션 프루닝)
EXPLAIN SELECT * FROM logs WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
-- partitions: p2024
```

### 흔한 안티패턴 정리

**안티패턴 1: 모든 컬럼에 인덱스**

```sql
-- 나쁜 예
CREATE INDEX idx_a ON t(a);
CREATE INDEX idx_b ON t(b);
CREATE INDEX idx_c ON t(c);
CREATE INDEX idx_ab ON t(a, b);
CREATE INDEX idx_bc ON t(b, c);
-- 쓰기 성능 폭락, 옵티마이저 혼란
```

**안티패턴 2: 인덱스 컬럼 순서 무시**

```sql
-- 자주 실행되는 쿼리
WHERE user_id = ? AND status = ?

-- 나쁜 인덱스 (선택도 낮은 status를 앞에)
INDEX (status, user_id)

-- 좋은 인덱스
INDEX (user_id, status)
```

**안티패턴 3: 중복 인덱스**

```sql
INDEX (a)         -- 이미 있는데
INDEX (a, b)      -- (a, b)가 있으면 (a) 단독은 불필요
-- (a, b) 인덱스는 a 단독 조건에도 사용됨
```

**안티패턴 4: 정렬에 인덱스 미활용**

```sql
-- 인덱스: (user_id, created_at)
-- 좋음: 인덱스 순서대로 정렬
WHERE user_id = 1 ORDER BY created_at DESC

-- 나쁨: 인덱스 순서와 혼합 정렬 방향 (MySQL 8.0 이전)
WHERE user_id = 1 ORDER BY created_at ASC, amount DESC
-- MySQL 8.0+에서는 내림차순 인덱스 지원: INDEX (user_id, created_at DESC, amount ASC)
```

---

인덱스는 데이터베이스 성능 최적화의 첫 번째 도구다. B+Tree의 동작 원리를 이해하면 어떤 인덱스가 왜 효과적인지, 왜 어떤 쿼리는 인덱스가 있어도 느린지 직관적으로 알 수 있다. 클러스터드 인덱스가 테이블 자체라는 사실, 세컨더리 인덱스가 기본키를 통해 두 번 탐색한다는 사실, 복합 인덱스에서 선행 컬럼 규칙이 B-Tree 정렬 방식에서 비롯된다는 사실 — 이것들을 이해하면 인덱스 설계가 단순한 암기가 아니라 원리에서 도출되는 논리가 된다. 다음 편에서는 트랜잭션과 잠금, MVCC를 다룬다.

---

## 참고 자료

1. **High Performance MySQL, 4th Edition** — Silvia Botros, Jeremy Tinley (O'Reilly, 2022): InnoDB 내부 구조와 인덱스 최적화의 사실상 바이블. Chapter 8 "Indexing for High Performance"는 필독.

2. **Database Internals** — Alex Petrov (O'Reilly, 2019): B-Tree 구현의 세부 사항을 페이지 레이아웃, 분할/병합 알고리즘 수준까지 다룬다. 스토리지 엔진을 깊이 이해하고 싶다면 Part 1 전체를 읽어야 한다.

3. **MySQL 8.0 Reference Manual: EXPLAIN Statement** — [dev.mysql.com/doc](https://dev.mysql.com/doc/refman/8.0/en/explain.html): EXPLAIN 출력 컬럼의 공식 정의. type별 의미와 Extra 값 목록은 항상 공식 문서가 기준.

4. **Use The Index, Luke** — Markus Winand ([use-the-index-luke.com](https://use-the-index-luke.com)): B-Tree 인덱스 원리부터 실전 쿼리 최적화까지 무료 온라인 도서. 시각적 다이어그램이 풍부하고 MySQL, PostgreSQL, Oracle을 아우른다.

5. **MySQL InnoDB Storage Engine: Understanding the Architecture** — Percona Blog 시리즈: InnoDB 버퍼 풀, redo log, 페이지 구조에 대한 실무 수준 설명. Percona는 MySQL 내부를 가장 깊이 파는 회사 중 하나.

6. **Designing Data-Intensive Applications** — Martin Kleppmann (O'Reilly, 2017): Chapter 3 "Storage and Retrieval"에서 B-Tree와 LSM-Tree의 trade-off를 명확하게 비교한다. 단순 인덱스를 넘어 스토리지 엔진 설계 전반을 이해하고 싶을 때.
