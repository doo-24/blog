---
title: "[데이터베이스] 4편 — 데이터 모델링: 테이블 설계의 원칙"
date: 2026-03-17T23:06:00+09:00
draft: false
tags: ["데이터 모델링", "정규화", "ERD", "테이블 설계", "데이터베이스", "서버"]
series: ["데이터베이스"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 3
summary: "정규화(1NF~BCNF) 각 단계 실전 예시, 반정규화 판단 기준, 1:N/M:N 관계 구현, 소프트 딜리트 트레이드오프, JSON 컬럼 활용과 한계, EAV 패턴의 함정까지"
---

데이터베이스 설계의 실패는 대부분 처음부터 예고된다. 서비스 초기에 "일단 빠르게" 만든 테이블 구조가 트래픽이 늘고 요구사항이 복잡해지면서 점점 괴물이 되어간다.

중복 데이터가 여러 테이블에 흩어지고, 하나의 값을 바꾸면 세 곳을 동시에 업데이트해야 하고, 조인 없이는 아무것도 못하는데 조인을 하면 성능이 무너진다.

이 글은 그 실패 경로를 미리 막는 방법에 대한 이야기다. 정규화의 각 단계가 실제로 무엇을 해결하는지, 반정규화는 언제 정당화되는지, 관계 구현의 패턴들은 어떤 트레이드오프를 가지는지를 실전 관점에서 깊게 파고든다.

---

## 정규화: 이론이 아닌 실전 도구

정규화(Normalization)는 학교에서 배울 때 딱딱한 이론처럼 느껴지지만, 사실 "데이터 이상(anomaly)을 제거하는 체계적인 방법"이다. 이상에는 세 가지가 있다.

- **삽입 이상(Insertion Anomaly)**: 어떤 데이터를 넣으려면 불필요한 다른 데이터도 함께 넣어야 하는 상황
- **갱신 이상(Update Anomaly)**: 하나의 사실을 여러 행에 중복 저장해서, 하나만 바꾸면 불일치가 생기는 상황
- **삭제 이상(Deletion Anomaly)**: 하나의 데이터를 지웠더니 관련 없는 다른 데이터도 사라지는 상황

정규화의 각 단계는 이 이상들을 특정 방식으로 제거해 나가는 과정이다.

---

### 1NF — 원자값의 원칙

**제1정규형(1NF)**의 조건은 단순하다: 모든 컬럼이 원자값(atomic value)을 가져야 한다. 즉, 하나의 셀에 여러 값이 들어가면 안 된다.

다음은 1NF를 위반하는 테이블이다.

```sql
-- 위반: tags 컬럼에 복수의 값이 콤마로 구분되어 저장됨
CREATE TABLE articles (
    id         INT PRIMARY KEY,
    title      VARCHAR(255),
    tags       VARCHAR(500),  -- 'mysql,index,performance'
    author_ids VARCHAR(500)   -- '1,3,7'
);
```

이 구조의 문제점은 명확하다. "mysql" 태그가 붙은 글을 찾으려면 `LIKE '%mysql%'` 같은 패턴 매칭을 써야 하고, 인덱스도 제대로 못 탄다. 태그 하나의 이름을 바꾸려면 모든 행을 스캔해서 문자열 치환을 해야 한다.

```sql
-- 올바른 구조: 각 태그를 별도 행으로
CREATE TABLE article_tags (
    article_id INT NOT NULL,
    tag        VARCHAR(100) NOT NULL,
    PRIMARY KEY (article_id, tag),
    FOREIGN KEY (article_id) REFERENCES articles(id)
);
```

**배열 컬럼도 같은 함정이다.** PostgreSQL의 `INTEGER[]`나 MySQL의 JSON 배열을 PK/FK 관계 대신 쓰는 경우가 있는데, 관계형 무결성이 사라지고 쿼리가 복잡해진다.

또 다른 1NF 위반 패턴은 **반복 그룹(Repeating Group)**이다.

```sql
-- 위반: phone1, phone2, phone3으로 반복 그룹 형성
CREATE TABLE users (
    id     INT PRIMARY KEY,
    name   VARCHAR(100),
    phone1 VARCHAR(20),
    phone2 VARCHAR(20),
    phone3 VARCHAR(20)
);
```

네 번째 전화번호가 생기면 DDL을 변경해야 한다. 전화번호가 하나도 없는 사용자도 NULL 3개를 끌고 다닌다. 이것도 별도 테이블로 분리해야 한다.

---

### 2NF — 부분 함수 종속 제거

**제2정규형(2NF)**은 1NF를 만족하면서, 복합 기본키가 있을 때 발생하는 **부분 함수 종속(Partial Functional Dependency)**을 제거한 상태다. 기본키의 일부에만 종속되는 컬럼이 있으면 안 된다.

주문 상세 테이블을 예로 들어보자.

```sql
-- 위반: 복합 PK (order_id, product_id)에서 product_name은 product_id에만 종속
CREATE TABLE order_items (
    order_id     INT,
    product_id   INT,
    product_name VARCHAR(255),  -- product_id에만 종속 (부분 종속)
    unit_price   DECIMAL(10,2), -- product_id에만 종속 (부분 종속)
    quantity     INT,
    PRIMARY KEY (order_id, product_id)
);
```

`product_name`과 `unit_price`는 `product_id`만 알면 결정된다. `order_id`는 알 필요도 없다. 이 상태에서 상품 이름이 바뀌면 `order_items` 테이블의 모든 관련 행을 업데이트해야 한다. 누락되면 갱신 이상이 발생한다.

```sql
-- 분리 후: 각 테이블이 자신의 관심사만 담는다
CREATE TABLE products (
    id         INT PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL
);

CREATE TABLE order_items (
    order_id   INT,
    product_id INT,
    quantity   INT NOT NULL,
    -- 주문 당시 가격은 스냅샷으로 별도 저장 (중요!)
    price_at_order DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

여기서 중요한 실전 포인트가 있다. `price_at_order` 컬럼을 보자. 현재 상품 가격이 변해도 과거 주문의 금액은 변하면 안 된다.

이것은 정규화를 위반하는 것이 아니라 **비즈니스 의미의 스냅샷**을 저장하는 것이다. 정규화는 "현재 상태"를 올바르게 표현하는 것이고, 이력 데이터는 그 자체로 독립적인 사실이다.

---

### 3NF — 이행적 함수 종속 제거

**제3정규형(3NF)**은 2NF를 만족하면서, **이행적 함수 종속(Transitive Functional Dependency)**을 제거한 상태다. 기본키가 아닌 컬럼 A가 기본키 B를 통해 다른 컬럼 C를 결정하는 구조다. `B → A`, `A → C` 이면 `B → C`가 성립하는데, 이때 C를 A와 분리해야 한다.

```sql
-- 위반: employee_id -> department_id -> department_name (이행 종속)
CREATE TABLE employees (
    employee_id     INT PRIMARY KEY,
    employee_name   VARCHAR(100),
    department_id   INT,
    department_name VARCHAR(100)  -- department_id에 종속 (이행 종속)
);
```

부서명이 바뀌면 해당 부서의 모든 직원 행을 업데이트해야 한다. 업데이트 중 일부가 실패하면 데이터베이스에 두 개의 다른 부서명이 공존하게 된다.

```sql
-- 분리 후
CREATE TABLE departments (
    id   INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE employees (
    id            INT PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

**실전에서 3NF 위반이 자주 나타나는 패턴:**

```sql
-- 흔한 위반: zip_code -> city -> state
CREATE TABLE addresses (
    id       INT PRIMARY KEY,
    user_id  INT,
    zip_code VARCHAR(10),
    city     VARCHAR(100),  -- zip_code에 종속
    state    VARCHAR(50)    -- zip_code에 종속
);
```

우편번호가 도시와 주(州)를 결정한다. 엄밀하게 3NF를 따르면 우편번호 테이블을 별도로 분리해야 한다.

하지만 이 경우 실무에서는 자주 허용된다. 우편번호 데이터가 외부 시스템(국가 우편 API)과 동기화될 수 있고, 주소 입력 시 사용자가 직접 입력한 값을 그대로 저장하는 게 더 현실적인 경우도 많다.

이것이 정규화와 실용성 사이의 긴장이다.

---

### BCNF — 보이스-코드 정규형

**보이스-코드 정규형(BCNF)**은 3NF보다 조금 더 엄격하다. 모든 결정자(Determinant)가 후보키(Candidate Key)여야 한다.

3NF를 만족하지만 BCNF를 위반하는 경우는 **복합 후보키가 여러 개 있고 서로 겹치는 상황**에서 발생한다.

```sql
-- 시나리오: 수강 신청 시스템
-- 규칙: 학생은 과목당 한 명의 교수만 배정된다
-- 규칙: 교수는 한 과목만 담당한다 (과목이 교수를 결정)
-- 후보키: (student_id, subject), (student_id, professor_id)

CREATE TABLE enrollments (
    student_id   INT,
    subject      VARCHAR(100),
    professor_id INT,
    -- professor_id -> subject (결정자인데 후보키가 아님 → BCNF 위반)
    PRIMARY KEY (student_id, subject)
);
```

여기서 `professor_id → subject` 관계가 존재한다. `professor_id`는 결정자이지만 후보키가 아니다. 같은 교수를 여러 학생이 수강하면 `professor_id → subject` 매핑이 중복 저장되고, 교수가 담당 과목을 바꾸면 여러 행을 업데이트해야 한다.

```sql
-- BCNF 준수: 분해
CREATE TABLE professor_subjects (
    professor_id INT PRIMARY KEY,
    subject      VARCHAR(100) NOT NULL
);

CREATE TABLE enrollments (
    student_id   INT,
    professor_id INT,
    PRIMARY KEY (student_id, professor_id),
    FOREIGN KEY (professor_id) REFERENCES professor_subjects(professor_id)
);
```

BCNF 분해는 때로 **무손실 조인(Lossless Join)**은 유지되지만 **함수 종속성 보존(Dependency Preservation)**이 깨지는 경우가 있다. 이런 경우 3NF에서 타협하는 것도 합리적인 선택이다. BCNF는 항상 달성 가능한 이상적인 목표가 아니라 트레이드오프를 가진 설계 옵션이다.

---

## 반정규화: 언제, 어떻게, 얼마나

정규화된 스키마는 데이터 무결성 측면에서 완벽하지만, 성능에서 대가를 치른다. 실무에서는 읽기 성능을 위해 정규화를 의도적으로 되돌리는 **반정규화(Denormalization)**가 필요한 순간이 반드시 온다.

### 반정규화가 정당화되는 조건

반정규화는 다음 세 조건이 모두 충족될 때만 고려해야 한다.

1. **측정된 성능 문제**: 느리다는 느낌이 아니라 EXPLAIN ANALYZE로 확인된 병목
2. **다른 수단의 실패**: 인덱스 추가, 쿼리 최적화, 캐싱으로 해결이 안 됨
3. **쓰기 복잡성 감수 가능**: 중복 데이터 동기화 비용을 명시적으로 감당할 수 있음

### 반정규화 패턴들

**컬럼 중복 (Column Duplication)**

```sql
-- 정규화된 구조: 게시글의 댓글 수를 알려면 COUNT 조인이 필요
SELECT p.id, p.title, COUNT(c.id) AS comment_count
FROM posts p
LEFT JOIN comments c ON c.post_id = p.id
GROUP BY p.id, p.title;

-- 반정규화: posts 테이블에 comment_count 컬럼 추가
ALTER TABLE posts ADD COLUMN comment_count INT NOT NULL DEFAULT 0;

-- 댓글 삽입/삭제 시 동기화 필요
INSERT INTO comments (post_id, body) VALUES (1, '...');
UPDATE posts SET comment_count = comment_count + 1 WHERE id = 1;
```

이 패턴의 위험은 **동기화 실패**다. 댓글을 삽입하는 트랜잭션이 중간에 실패하면? 배치로 직접 댓글을 삭제하면? `comment_count`가 실제와 달라진다.

이를 막으려면 트리거를 쓰거나, 주기적인 재집계 작업을 돌리거나, 애플리케이션 레벨에서 두 작업을 반드시 같은 트랜잭션으로 묶어야 한다.

**파생 컬럼 사전 계산 (Derived Column)**

```sql
-- 주문 총액을 매번 계산하지 않고 저장
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(12,2);

-- 주문 아이템이 변경될 때마다 재계산
UPDATE orders o
SET total_amount = (
    SELECT SUM(quantity * price_at_order)
    FROM order_items
    WHERE order_id = o.id
)
WHERE o.id = ?;
```

**테이블 병합 (Table Merging)**

1:1 관계나 거의 항상 같이 조회되는 테이블은 병합이 나을 수 있다.

```sql
-- users와 user_profiles를 항상 JOIN해서 쓴다면 병합 고려
-- 단, user_profiles의 일부 컬럼이 NULL을 많이 가진다면 오히려 불리
```

### 반정규화의 위험성

반정규화의 가장 큰 위험은 **점진적인 부패**다. 처음에는 하나의 캐시 컬럼을 잘 관리한다. 그러다 다음 기능을 추가하면서 업데이트 로직을 한 곳에 빠뜨린다.

6개월 후에는 데이터가 조용히 불일치 상태가 되고, 아무도 그 컬럼을 신뢰하지 않게 된다. 신뢰받지 못하는 캐시 컬럼은 존재하는 것 자체가 비용이다.

**반정규화 도입 시 체크리스트:**
- 중복 데이터를 업데이트해야 하는 모든 코드 경로가 문서화되어 있는가?
- 데이터 불일치를 감지하는 모니터링/알림이 있는가?
- 불일치 발생 시 재동기화하는 스크립트가 준비되어 있는가?

---

## ERD 표기법: 설계를 소통하는 언어

ERD(Entity-Relationship Diagram)는 테이블 구조를 시각화하는 도구다. 여러 표기법이 있지만 실무에서는 **Crow's Foot 표기법**이 가장 널리 쓰인다.

```
  users                    orders
  ┌──────────────┐         ┌──────────────────┐
  │ id (PK)      │         │ id (PK)          │
  │ name         │──────<  │ user_id (FK)     │
  │ email        │         │ status           │
  └──────────────┘         │ created_at       │
                           └──────────────────┘

  기호 설명:
  ──    1 (하나)
  ──<   1 이상 (one or many)
  ──○<  0 이상 (zero or many)
  ──○   0 또는 1 (zero or one)
  ──||  정확히 1 (exactly one)
```

ERD에서 중요한 것은 **카디널리티(Cardinality)**와 **모달리티(Modality)**를 정확히 표현하는 것이다. "사용자는 주문을 0개 이상 가질 수 있다"와 "주문은 반드시 하나의 사용자에 속한다"는 서로 다른 제약이고, 이 차이가 NULL 허용 여부, 외래키 제약 설정에 직접 반영된다.

---

## 관계 구현 패턴

### 1:N 관계

가장 일반적인 관계다. "부모-자식" 구조로, 자식 테이블에 부모의 PK를 외래키로 저장한다.

```sql
-- 사용자(1) : 주문(N)
CREATE TABLE orders (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id    BIGINT NOT NULL,
    status     ENUM('pending', 'paid', 'shipped', 'delivered', 'cancelled'),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT
);

-- user_id에 인덱스가 없으면 특정 사용자의 주문 조회가 풀스캔
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

`ON DELETE` 옵션 선택이 중요하다:
- `RESTRICT` / `NO ACTION`: 자식이 있으면 부모 삭제 불가 (가장 안전)
- `CASCADE`: 부모 삭제 시 자식도 자동 삭제 (의도치 않은 대량 삭제 위험)
- `SET NULL`: 부모 삭제 시 자식의 FK를 NULL로 (고아 레코드 허용)

운영 환경에서 `CASCADE`는 매우 조심해야 한다. 실수로 사용자 한 명을 지웠는데 그 사용자의 주문, 리뷰, 포인트, 알림이 모두 사라지는 사고가 실제로 발생한다.

### M:N 관계

M:N 관계는 직접 표현할 수 없다. 반드시 **중간 테이블(Junction Table, Associative Table)**을 통해 두 개의 1:N 관계로 분해한다.

```sql
-- 게시글(M) : 태그(N) → article_tags 중간 테이블
CREATE TABLE tags (
    id   INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE article_tags (
    article_id INT NOT NULL,
    tag_id     INT NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (article_id, tag_id),
    FOREIGN KEY (article_id) REFERENCES articles(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id)     REFERENCES tags(id)     ON DELETE CASCADE
);

-- 역방향 조회를 위한 인덱스 (tag_id로 게시글 찾기)
CREATE INDEX idx_article_tags_tag_id ON article_tags(tag_id);
```

중간 테이블에 **추가 속성**이 생기면 그 테이블 자체가 독립적인 엔티티가 된다.

```sql
-- 사용자(M) : 강좌(N) → enrollments (수강 신청이라는 독립 엔티티)
CREATE TABLE enrollments (
    id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id      BIGINT NOT NULL,
    course_id    BIGINT NOT NULL,
    enrolled_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    progress     TINYINT NOT NULL DEFAULT 0,  -- 진도율 0-100
    completed_at DATETIME,
    UNIQUE KEY uq_user_course (user_id, course_id),
    FOREIGN KEY (user_id)   REFERENCES users(id),
    FOREIGN KEY (course_id) REFERENCES courses(id)
);
```

`UNIQUE KEY`로 중복 수강 신청을 방지하면서, `AUTO_INCREMENT` PK로 각 수강 기록을 독립적으로 참조할 수 있다.

### 자기 참조 테이블 (Self-Referencing Table)

계층 구조(트리)를 표현할 때 쓰는 패턴이다. 카테고리, 조직도, 댓글 답글 등에 활용된다.

```sql
-- 카테고리 계층 구조
CREATE TABLE categories (
    id        INT PRIMARY KEY AUTO_INCREMENT,
    name      VARCHAR(100) NOT NULL,
    parent_id INT,  -- NULL이면 최상위 카테고리
    FOREIGN KEY (parent_id) REFERENCES categories(id)
);

-- 데이터 예시
INSERT INTO categories (id, name, parent_id) VALUES
(1, '전자제품', NULL),
(2, '컴퓨터', 1),
(3, '노트북', 2),
(4, '데스크탑', 2),
(5, '스마트폰', 1);
```

자기 참조의 문제는 **깊이를 알 수 없는 계층**을 조회하는 것이다. SQL에서 이를 해결하는 방법은 여러 가지다.

**방법 1: 재귀 CTE (MySQL 8.0+, PostgreSQL)**
```sql
WITH RECURSIVE category_tree AS (
    -- 기저 케이스: 최상위 카테고리
    SELECT id, name, parent_id, 0 AS depth, CAST(name AS CHAR(1000)) AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- 재귀 케이스: 자식 카테고리 탐색
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, CONCAT(ct.path, ' > ', c.name)
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;
```

**방법 2: Closure Table 패턴 (쿼리 성능 우선)**

```sql
-- 모든 조상-자손 관계를 사전에 저장
CREATE TABLE category_closures (
    ancestor_id   INT NOT NULL,
    descendant_id INT NOT NULL,
    depth         INT NOT NULL,
    PRIMARY KEY (ancestor_id, descendant_id),
    FOREIGN KEY (ancestor_id)   REFERENCES categories(id),
    FOREIGN KEY (descendant_id) REFERENCES categories(id)
);

-- '컴퓨터(2)'의 모든 하위 카테고리를 O(1) 조회
SELECT c.*
FROM categories c
JOIN category_closures cc ON c.id = cc.descendant_id
WHERE cc.ancestor_id = 2 AND cc.depth > 0;
```

Closure Table은 쓰기가 복잡해지지만(노드 삽입 시 closure 행 추가 필요) 읽기 쿼리가 단순하고 빠르다. 계층 구조를 자주 읽지만 드물게 수정하는 경우에 적합하다.

**방법 3: Nested Set (범위 기반)**

```sql
-- lft, rgt 값으로 트리를 인코딩
CREATE TABLE categories (
    id   INT PRIMARY KEY,
    name VARCHAR(100),
    lft  INT NOT NULL,
    rgt  INT NOT NULL
);
-- '노트북'의 모든 조상 조회 (빠른 읽기)
SELECT * FROM categories WHERE lft < 5 AND rgt > 6;
```

Nested Set은 특정 노드의 서브트리 조회가 매우 빠르지만, 삽입/삭제 시 많은 행의 lft/rgt를 재계산해야 하므로 쓰기 비용이 매우 높다.

---

## 소프트 딜리트와 감사 컬럼

### 소프트 딜리트 (Soft Delete)

소프트 딜리트는 레코드를 실제로 삭제하지 않고 "삭제됨" 표시만 하는 패턴이다.

```sql
-- 방법 1: deleted_at 타임스탬프
ALTER TABLE users ADD COLUMN deleted_at DATETIME DEFAULT NULL;

-- 소프트 딜리트 실행
UPDATE users SET deleted_at = NOW() WHERE id = ?;

-- 유효한 사용자 조회 (모든 쿼리에 이 조건이 붙어야 함)
SELECT * FROM users WHERE deleted_at IS NULL;

-- 방법 2: is_deleted 불리언 플래그
ALTER TABLE users ADD COLUMN is_deleted TINYINT(1) NOT NULL DEFAULT 0;
```

`deleted_at`이 `is_deleted`보다 낫다. 언제 삭제됐는지 정보가 추가로 담기고, NULL/NOT NULL로 삭제 여부를 판단할 수 있으며, 부분 인덱스와 친화적이다.

### 소프트 딜리트의 트레이드오프

**장점:**
- 실수로 지운 데이터 복구 가능
- 삭제된 데이터에 대한 참조 무결성 유지 (외래키가 끊기지 않음)
- 감사/규정 준수 목적으로 이력 보관

**단점:**

```sql
-- 문제 1: UNIQUE 제약이 깨진다
-- 이메일 UNIQUE 제약이 있을 때, 같은 이메일로 재가입하면?
-- deleted_at이 있어도 UNIQUE 제약에 걸림
ALTER TABLE users ADD UNIQUE (email); -- 소프트 딜리트와 충돌!

-- 해결책: NULL을 포함한 복합 UNIQUE 인덱스 (MySQL은 NULL을 중복으로 보지 않음)
-- 또는 partial index (PostgreSQL)
CREATE UNIQUE INDEX idx_users_email_active
ON users(email)
WHERE deleted_at IS NULL;  -- PostgreSQL에서만 작동
```

MySQL에서는 deleted 행의 email을 다른 값으로 교체하는 방법을 쓴다.

```sql
-- MySQL 소프트 딜리트 + UNIQUE 이메일 처리
UPDATE users
SET
    email      = CONCAT('deleted_', id, '_', email),
    deleted_at = NOW()
WHERE id = ?;
```

```sql
-- 문제 2: 모든 쿼리에 WHERE deleted_at IS NULL 추가 필요
-- 빠뜨리면 조용히 버그가 된다
-- ORM을 쓰는 경우 글로벌 스코프(Global Scope)로 자동 적용하는 것이 안전
```

```sql
-- 문제 3: 테이블이 커진다
-- 삭제된 데이터가 쌓이면 테이블 크기가 계속 증가
-- 인덱스 크기도 증가 → 쿼리 성능 저하
-- 해결: 주기적으로 오래된 소프트 딜리트 레코드를 아카이브 테이블로 이동
```

### 감사 컬럼 설계 (Audit Columns)

모든 테이블에 공통으로 넣는 감사 컬럼들이다.

```sql
CREATE TABLE articles (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    title      VARCHAR(255) NOT NULL,
    body       TEXT NOT NULL,

    -- 생성/수정 감사
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by BIGINT,  -- users.id 참조 (FK를 걸지 않는 경우도 많음)
    updated_by BIGINT,

    -- 소프트 딜리트
    deleted_at DATETIME DEFAULT NULL,
    deleted_by BIGINT,

    FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL,
    FOREIGN KEY (updated_by) REFERENCES users(id) ON DELETE SET NULL
);
```

`created_by`, `updated_by`에 FK를 걸지 않는 경우도 있다. 사용자 계정이 삭제돼도 데이터는 보존해야 하는데, `ON DELETE SET NULL`로 처리하면 누가 만들었는지 이력이 사라진다.

이 경우 FK 없이 user_id를 숫자로 저장하고 애플리케이션에서 관리하는 방법을 택하기도 한다. 물론 참조 무결성은 포기해야 한다.

**변경 이력을 완전히 보존하고 싶다면** 별도의 히스토리 테이블 패턴을 고려한다.

```sql
-- articles_history: 변경될 때마다 스냅샷 저장
CREATE TABLE articles_history (
    history_id    BIGINT PRIMARY KEY AUTO_INCREMENT,
    article_id    BIGINT NOT NULL,
    operation     ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    changed_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    changed_by    BIGINT,
    -- 변경 전 값들 (UPDATE, DELETE)
    old_title     VARCHAR(255),
    old_body      TEXT,
    -- 변경 후 값들 (INSERT, UPDATE)
    new_title     VARCHAR(255),
    new_body      TEXT
);
```

히스토리 테이블은 강력하지만 스토리지 비용이 크다. Temporal Table(MySQL 8.0.17+의 System-Versioned Table)을 쓰면 DB가 자동으로 관리해주지만 기능이 제한적이다.

---

## JSON 컬럼: 유연성의 유혹과 함정

MySQL 5.7+, PostgreSQL 9.4+부터 네이티브 JSON 컬럼을 지원한다. 스키마리스(schemaless)한 데이터를 관계형 DB에 저장할 수 있게 됐는데, 이게 생각보다 위험한 유혹이다.

### JSON 컬럼이 적합한 경우

```sql
-- 실제로 유용한 케이스 1: 외부 API 응답 원본 저장
CREATE TABLE webhook_events (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    source      VARCHAR(50) NOT NULL,   -- 'stripe', 'github', etc.
    event_type  VARCHAR(100) NOT NULL,
    payload     JSON NOT NULL,           -- 원본 페이로드 보존
    received_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 실제로 유용한 케이스 2: 다형적 설정값 (per-user settings)
CREATE TABLE user_settings (
    user_id  BIGINT PRIMARY KEY,
    settings JSON NOT NULL DEFAULT '{}',
    FOREIGN KEY (user_id) REFERENCES users(id)
);
-- {"theme": "dark", "notifications": {"email": true, "push": false}, "language": "ko"}
```

```sql
-- JSON 인덱스 (MySQL) - 자주 조회하는 경로에만 적용
ALTER TABLE user_settings
ADD COLUMN theme VARCHAR(20)
    GENERATED ALWAYS AS (settings->>'$.theme') STORED;

CREATE INDEX idx_user_settings_theme ON user_settings(theme);
```

### JSON 컬럼의 함정

**함정 1: 쿼리 복잡도와 성능**

```sql
-- 읽기는 되지만 최적화가 어렵다
SELECT * FROM orders
WHERE JSON_EXTRACT(metadata, '$.shipping.carrier') = 'FedEx';
-- 이 쿼리는 전체 테이블 스캔. 인덱스 없이는 느리다.

-- PostgreSQL은 조금 낫다: GIN 인덱스 지원
CREATE INDEX idx_orders_metadata ON orders USING GIN(metadata);
```

**함정 2: 데이터 타입 강제 불가**

```sql
-- JSON 컬럼에는 어떤 값도 들어갈 수 있다
UPDATE orders SET metadata = JSON_SET(metadata, '$.amount', 'abc');
-- DB가 막아주지 않는다. 애플리케이션이 책임져야 한다.
```

**함정 3: 조인 불가**

```sql
-- JSON 안의 user_id로 users 테이블과 조인하고 싶다면?
SELECT u.name, o.*
FROM orders o
JOIN users u ON JSON_EXTRACT(o.metadata, '$.buyer_id') = u.id;
-- 가능하긴 하지만 인덱스 활용이 어렵고 느리다
-- buyer_id는 정규 컬럼으로 꺼내야 했다
```

**함정 4: 마이그레이션의 어려움**

나중에 JSON 안의 특정 필드가 중요해져서 별도 컬럼으로 분리하고 싶을 때, 수백만 행의 데이터를 마이그레이션해야 한다. 이 작업은 생각보다 훨씬 복잡하다.

**가이드라인:** JSON 컬럼은 "진짜로 스키마가 불확실한" 데이터에만 써야 한다. "지금 당장 스키마 짜기 귀찮다"는 이유로 쓰면 반드시 나중에 후회한다. 자주 조회하거나, 조인에 쓰거나, 집계하는 필드라면 반드시 정규 컬럼으로 모델링해야 한다.

---

## EAV 패턴: 유연성이라는 이름의 재앙

**EAV(Entity-Attribute-Value)** 패턴은 속성 이름과 값을 행으로 저장하는 구조다.

```sql
-- EAV 구조
CREATE TABLE product_attributes (
    product_id      INT NOT NULL,
    attribute_name  VARCHAR(100) NOT NULL,
    attribute_value VARCHAR(500),
    PRIMARY KEY (product_id, attribute_name),
    FOREIGN KEY (product_id) REFERENCES products(id)
);

-- 데이터 예시
INSERT INTO product_attributes VALUES
(1, 'color',  'red'),
(1, 'size',   'XL'),
(1, 'weight', '2.5kg'),
(2, 'color',  'blue'),
(2, 'material', 'cotton');
```

이 구조는 어떤 속성도 추가할 수 있다는 점에서 매력적으로 보인다. DDL 변경 없이 새 속성을 추가할 수 있고, 상품마다 다른 속성을 가질 수 있다.

### EAV가 무너지는 방식

**문제 1: 타입 안전성 없음**

모든 값이 VARCHAR로 저장된다. 숫자 비교, 날짜 범위 검색이 불가능하거나 느리다.

```sql
-- 2.5kg 이상인 상품 찾기
SELECT product_id FROM product_attributes
WHERE attribute_name = 'weight'
AND CAST(attribute_value AS DECIMAL) >= 2.5;
-- 타입 캐스팅이 인덱스를 무력화한다
```

**문제 2: 피벗 쿼리의 지옥**

상품의 모든 속성을 한 행으로 보려면:

```sql
SELECT
    p.id,
    p.name,
    MAX(CASE WHEN pa.attribute_name = 'color'    THEN pa.attribute_value END) AS color,
    MAX(CASE WHEN pa.attribute_name = 'size'     THEN pa.attribute_value END) AS size,
    MAX(CASE WHEN pa.attribute_name = 'weight'   THEN pa.attribute_value END) AS weight,
    MAX(CASE WHEN pa.attribute_name = 'material' THEN pa.attribute_value END) AS material
FROM products p
LEFT JOIN product_attributes pa ON pa.product_id = p.id
GROUP BY p.id, p.name;
-- 속성이 늘어날수록 쿼리가 커진다. 동적으로 생성하면 SQL 인젝션 위험.
```

**문제 3: 무결성 강제 불가**

"모든 전자제품은 전압 속성이 필수다" 같은 비즈니스 규칙을 DB 레벨에서 강제할 방법이 없다. 애플리케이션에서 검증해야 하는데, 검증 로직이 여러 곳에 분산되면 결국 어딘가에서 누락된다.

**문제 4: 성능 저하**

10개 속성을 가진 상품 1000개의 데이터를 EAV로 저장하면 10,000행이 된다. 정규 컬럼이면 1,000행이다. 조인, 집계, 정렬 모든 면에서 10배 이상의 오버헤드가 발생한다.

### EAV 대안

**대안 1: 타입별 테이블 분리 (Class Table Inheritance)**

```sql
-- 공통 속성은 products 테이블에
CREATE TABLE products (id INT PK, name, price, category_id);

-- 전자제품만의 속성
CREATE TABLE electronics (
    product_id INT PRIMARY KEY,
    voltage    VARCHAR(20),
    power_watt INT,
    FOREIGN KEY (product_id) REFERENCES products(id)
);

-- 의류만의 속성
CREATE TABLE clothing (
    product_id INT PRIMARY KEY,
    size       VARCHAR(10),
    material   VARCHAR(100),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

**대안 2: JSON 컬럼 (제한적 사용)**

앞서 언급한 함정을 감수하고, 정말 비정형적인 속성만 JSON으로 저장한다.

```sql
CREATE TABLE products (
    id           INT PRIMARY KEY,
    name         VARCHAR(255) NOT NULL,
    price        DECIMAL(10,2) NOT NULL,
    category_id  INT NOT NULL,
    -- 공통 속성은 정규 컬럼으로
    -- 카테고리별 고유 속성만 JSON으로
    extra_attrs  JSON
);
```

**EAV를 사용해야만 하는 경우:** 플러그인 시스템이나 사용자 정의 필드처럼 진짜로 스키마를 런타임에 확장해야 하는 경우다. 이때도 EAV의 단점을 충분히 인지하고, 자주 조회하는 속성은 별도 컬럼으로 캐시하는 하이브리드 전략을 써야 한다.

---

## 실전 설계 체크리스트

좋은 데이터 모델을 만들기 위한 설계 시점의 질문들이다.

**정규화 관련:**
- 한 컬럼에 여러 값이 들어가는가? (1NF 위반)
- 복합 PK에서 일부 컬럼에만 종속되는 컬럼이 있는가? (2NF 위반)
- PK가 아닌 컬럼이 다른 비-PK 컬럼을 결정하는가? (3NF 위반)

**관계 모델링 관련:**
- 외래키 컬럼에 인덱스가 있는가? (조인/조회 성능)
- ON DELETE 동작이 비즈니스 요구사항과 일치하는가?
- M:N 관계의 중간 테이블에 추가 속성이 생길 가능성이 있는가?

**운영 고려사항:**
- 소프트 딜리트를 쓴다면 UNIQUE 제약 처리 방법이 있는가?
- 감사 컬럼(created_at, updated_at)이 모든 중요 테이블에 있는가?
- JSON 컬럼에 저장된 필드 중 자주 조회하는 것은 정규 컬럼으로 인덱싱됐는가?

---

테이블 설계는 코드와 다르게 나중에 바꾸기가 매우 어렵다. 컬럼 추가는 쉽지만 컬럼 삭제나 테이블 분리는 운영 중인 시스템에서 큰 위험을 수반한다.

처음 설계를 잘 하는 것이 가장 싸게 먹히는 선택이다. 정규화를 철저히 하고, 반정규화는 측정된 필요에 의해서만, EAV와 무분별한 JSON 컬럼은 피하는 것이 원칙이다.

어떤 결정을 내리든 트레이드오프를 명시적으로 문서화하는 것이 미래의 팀원과 미래의 자신에게 보내는 최고의 선물이다.

---

## 참고 자료

1. **Database Design for Mere Mortals** — Michael J. Hernandez. 정규화와 관계 모델링의 실전 입문서로, 이론을 실무 예제로 명쾌하게 설명한다.

2. **SQL Antipatterns** — Bill Karwin. EAV 패턴을 포함해 실무에서 자주 저지르는 25가지 DB 설계 실수를 상세히 다룬다. 이 글의 EAV 섹션은 이 책에서 많은 영감을 받았다.

3. **Designing Data-Intensive Applications** — Martin Kleppmann. 관계형 모델부터 NoSQL까지 데이터 모델링의 전반적인 철학을 다루며, 왜 특정 설계 결정이 분산 환경에서 문제가 되는지 깊게 설명한다.

4. **MySQL 공식 문서 — JSON Data Type** (dev.mysql.com). MySQL에서 JSON 컬럼의 인덱싱, 쿼리, 성능 특성에 대한 공식 레퍼런스.

5. **Joe Celko's Trees and Hierarchies in SQL for Smarties** — Joe Celko. 자기 참조 테이블과 계층 구조 표현(Adjacency List, Nested Set, Closure Table)을 SQL로 구현하는 방법을 가장 깊게 다루는 책.

6. **Use The Index, Luke** (use-the-index-luke.com). 정규화된 스키마와 인덱스의 관계, 반정규화가 인덱스 전략에 미치는 영향을 무료로 깊게 배울 수 있는 온라인 리소스.
