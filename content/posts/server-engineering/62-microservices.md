---
title: "[분산 시스템과 MSA] 4편 — 마이크로서비스: 분리의 이점과 대가"
date: 2026-03-17T17:04:00+09:00
draft: false
tags: ["MSA", "마이크로서비스", "모놀리스", "서비스 경계", "서버"]
series: ["분산 시스템과 MSA"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 9
summary: "모놀리스 vs MSA 실질적 트레이드오프, 서비스 경계 정의(도메인 기반 분리)와 너무 잘게 나누는 위험, 서비스 간 통신(동기 REST/gRPC vs 비동기 메시지), 데이터 소유권과 서비스별 DB 분리까지"
---

마이크로서비스 아키텍처(MSA)는 지난 10년간 서버 엔지니어링의 가장 큰 흐름 중 하나였다. Netflix, Amazon, Uber가 이 방식으로 수억 명의 사용자를 처리하는 모습을 보며 많은 팀이 MSA 전환을 결심했다.

하지만 현실은 냉혹하다. 분리의 이점 뒤에는 반드시 그에 상응하는 대가가 따르고, 그 대가를 과소평가한 팀은 오히려 더 큰 혼란을 맞는다.

이번 편에서는 MSA의 실질적 트레이드오프를 직시하고, 제대로 된 서비스 경계를 설정하는 방법과 그 분리를 유지하기 위한 핵심 원칙들을 살펴본다.

---

## 모놀리스 vs MSA: 트레이드오프를 직시하라

### 모놀리스의 장점은 무시되어선 안 된다

"모놀리스"라는 단어는 어느 순간부터 레거시, 구식, 문제 덩어리의 동의어처럼 쓰이기 시작했다.

하지만 모놀리스는 태생적으로 많은 장점을 가진다.

하나의 코드베이스에서 개발하므로 리팩토링이 간단하다. IDE의 "함수 이름 바꾸기" 하나로 전체 시스템에 반영된다.

배포 단위가 하나이므로 CI/CD 파이프라인이 단순하다. 하나의 바이너리 혹은 JAR 파일을 배포하면 끝이다.

트랜잭션 경계가 명확하다. 데이터베이스 트랜잭션 하나로 복잡한 비즈니스 로직을 원자적으로 처리할 수 있다.

서비스 간 네트워크 호출이 없으니 지연도 없다. 함수 호출은 밀리초가 아닌 나노초 단위다.

```
모놀리스 내부 호출 비용:
  함수 호출: ~10ns
  같은 프로세스 내 메모리 접근: ~100ns

MSA 네트워크 호출 비용:
  같은 데이터센터 내 HTTP: ~1ms
  직렬화/역직렬화 오버헤드: ~0.1ms~수ms
  재시도, 타임아웃 처리: 가변적
```

이 차이는 단순한 숫자 이상의 의미를 가진다. 하나의 사용자 요청이 10개 서비스를 거친다면, 각 홉마다 실패 가능성이 생긴다.

### MSA가 해결하는 진짜 문제

MSA는 특정 규모와 팀 구조에서 실제로 효과적인 해결책이다.

**독립 배포**: 주문 서비스를 바꾸기 위해 결제 서비스 팀과 배포 일정을 조율할 필요가 없다. 각 팀이 자신의 서비스를 독립적으로 배포할 수 있다.

**독립 확장**: 추천 서비스에 부하가 몰릴 때 추천 서비스만 스케일 아웃하면 된다. 전체 시스템을 확장할 필요가 없다.

**기술 다양성**: 머신러닝 파이프라인은 Python으로, 실시간 처리는 Go로, 관리 도구는 Ruby로 각각 최적의 기술을 선택할 수 있다.

**장애 격리**: 리뷰 서비스가 다운되어도 상품 조회와 결제는 계속 동작할 수 있다.

하지만 이 이점들은 조건부다. 팀이 아직 작고, 도메인이 명확히 정의되지 않았고, 운영 성숙도가 낮다면 MSA는 오히려 생산성을 갉아먹는다.

### 트레이드오프 정리

| 측면 | 모놀리스 | MSA |
|---|---|---|
| 개발 초기 속도 | 빠름 | 느림 (인프라 셋업 비용) |
| 배포 복잡도 | 단순 | 높음 (서비스 수 × 복잡도) |
| 디버깅 | 쉬움 (단일 스택 트레이스) | 어려움 (분산 추적 필요) |
| 트랜잭션 | ACID 보장 | 분산 트랜잭션 필요 |
| 팀 독립성 | 낮음 | 높음 |
| 확장 유연성 | 전체 단위 | 서비스 단위 |
| 운영 비용 | 낮음 | 높음 |

---

## 서비스 경계 정의: 어디서 잘라야 하는가

### 도메인 주도 설계(DDD)와 바운디드 컨텍스트

올바른 서비스 경계는 기술적 편의가 아닌 비즈니스 도메인에서 시작해야 한다.

에릭 에반스의 도메인 주도 설계(DDD)에서 핵심 개념은 **바운디드 컨텍스트(Bounded Context)**다. 같은 단어라도 컨텍스트가 다르면 의미가 다를 수 있다.

예를 들어 이커머스 플랫폼에서 "고객(Customer)"이라는 개념을 보자.

```
주문 컨텍스트의 Customer:
  - 주문자 이름
  - 배송 주소
  - 주문 이력

마케팅 컨텍스트의 Customer:
  - 관심 카테고리
  - 클릭 이력
  - 광고 반응률

지원(Support) 컨텍스트의 Customer:
  - 문의 이력
  - 불만 접수 건수
  - 담당 CS 에이전트
```

같은 "고객"이지만 각 컨텍스트에서 필요한 속성과 행동이 전혀 다르다.

이 컨텍스트 경계가 곧 서비스 경계의 후보가 된다.

### 이벤트 스토밍으로 경계 찾기

팀이 함께 도메인 이벤트를 나열하고 그룹핑하는 **이벤트 스토밍** 워크숍은 서비스 경계를 찾는 실용적인 방법이다.

```
[이커머스 이벤트 스토밍 예시]

상품 도메인:
  - 상품이 등록됨
  - 재고가 변경됨
  - 가격이 수정됨

주문 도메인:
  - 장바구니에 상품이 담김
  - 주문이 생성됨
  - 주문이 취소됨

결제 도메인:
  - 결제가 요청됨
  - 결제가 승인됨
  - 환불이 처리됨

배송 도메인:
  - 배송이 시작됨
  - 배송지가 변경됨
  - 배송이 완료됨
```

이벤트들이 자연스럽게 클러스터를 이루는 곳에 서비스 경계가 있을 가능성이 높다.

### 좋은 경계의 특징

좋은 서비스 경계는 **높은 응집도**와 **낮은 결합도**를 가진다.

서비스 내부의 기능들은 서로 밀접하게 관련되어 자주 함께 변경되어야 한다.

서비스 간에는 명확하고 안정적인 인터페이스만 존재해야 한다. 한 서비스의 내부 변경이 다른 서비스에 영향을 주어선 안 된다.

```
좋은 경계 예시:
  주문 서비스 (응집도 높음)
  ├── 주문 생성
  ├── 주문 조회
  ├── 주문 취소
  └── 주문 상태 변경

나쁜 경계 예시:
  "기타 서비스" (응집도 낮음)
  ├── 주문 이력 조회
  ├── 리뷰 작성
  ├── 포인트 조회
  └── 알림 설정
```

---

## 너무 잘게 나누는 위험: 나노서비스 안티패턴

### 분리의 강박

MSA에 입문한 팀이 흔히 저지르는 실수는 모든 기능을 별개의 서비스로 분리하려는 강박이다.

"사용자 인증", "사용자 프로필", "사용자 설정", "사용자 알림 설정"을 각각 별도의 서비스로 만드는 경우가 여기에 해당한다.

이를 **나노서비스(Nanoservice)** 안티패턴이라 부른다.

### 나노서비스의 증상

**채터링 호출 패턴**: 하나의 API 요청을 처리하기 위해 내부적으로 수십 개의 서비스 호출이 발생한다. 각 홉마다 네트워크 지연이 누적된다. 단일 서버에서라면 함수 호출 하나로 끝날 일이, 서비스가 분리되면 각각 직렬 HTTP 왕복이 되어 전체 응답 시간이 선형으로 늘어난다.

```
사용자 프로필 페이지 로딩 시:
  GET /users/123
  → 사용자 서비스 호출 (기본 정보)
  → 프로필 이미지 서비스 호출
  → 팔로워 수 서비스 호출
  → 최근 활동 서비스 호출
  → 알림 설정 서비스 호출
  → 배지 서비스 호출

총 6번의 네트워크 왕복 = 최소 6ms 추가 지연
각 서비스에 장애 가능성 × 6
```

**배포 지옥**: 하나의 기능 변경이 여러 서비스에 걸쳐 있어 동시에 여러 서비스를 배포해야 한다. MSA의 독립 배포 이점이 사라진다.

**공유 로직의 증식**: 인증, 로깅, 에러 처리 같은 공통 코드가 수십 개 서비스에 복사된다. 하나를 고치면 전부 고쳐야 한다.

### 올바른 기준: "얼마나 자주 독립적으로 변경되는가"

서비스 분리의 핵심 기준은 **독립적인 변경 빈도**다.

사용자 프로필과 사용자 설정이 항상 함께 변경된다면, 이는 같은 서비스에 있어야 한다는 신호다.

반면 결제 로직은 사용자 프로필과 거의 독립적으로 변경되며 별도의 PCI DSS 보안 요건까지 있다. 이건 명확히 분리해야 한다.

```
분리 체크리스트:
  □ 이 기능은 다른 기능과 독립적으로 배포될 필요가 있는가?
  □ 이 기능은 별도로 스케일 아웃이 필요한가?
  □ 이 기능은 별도 팀이 소유하는가?
  □ 이 기능은 다른 기술 스택을 사용해야 하는가?
  □ 이 기능은 별도의 보안/규정 준수 경계가 필요한가?

하나도 해당하지 않는다면 → 분리하지 마라
```

### 모노레포와 멀티레포

서비스가 여럿이라도 하나의 저장소에서 관리하는 **모노레포**는 유효한 선택이다.

코드 공유가 쉽고, 전체 시스템의 변경을 하나의 PR로 추적할 수 있으며, 공통 도구 설정(린팅, 포맷팅, CI)을 일관되게 유지할 수 있다.

팀이 완전히 독립적으로 움직이고 기술 스택도 다를 때는 멀티레포가 적합하다. 어떤 방식이든, 선택 기준은 기술보다 팀 구조와 조직 문화에 있다.

---

## 서비스 간 통신: 동기 vs 비동기

### 동기 통신의 직관성과 위험

동기 통신에서 호출자는 응답이 올 때까지 기다린다. 가장 직관적인 방식이다.

REST API와 gRPC가 대표적이다.

```python
# 동기 REST 호출 예시
import httpx

async def get_user_with_orders(user_id: str):
    async with httpx.AsyncClient() as client:
        # 사용자 정보 조회
        user_resp = await client.get(f"http://user-service/users/{user_id}")
        user = user_resp.json()

        # 주문 정보 조회
        order_resp = await client.get(
            f"http://order-service/orders?user_id={user_id}"
        )
        orders = order_resp.json()

    return {"user": user, "orders": orders}
```

동기 호출은 이해하기 쉽고 디버깅이 편하다. 에러가 발생하면 즉시 알 수 있다.

하지만 **시간적 결합(Temporal Coupling)**이라는 문제가 있다. 호출된 서비스가 응답하지 못하면, 호출한 서비스도 블로킹된다. 하위 서비스 하나가 느려지면 그 영향이 위로 전파된다.

```
장애 전파 예시:
  사용자 요청
    → API Gateway
      → 주문 서비스 (3초 대기)
        → 결제 서비스 (타임아웃! 5초)
          → 외부 PG사 API (응답 없음)

결과: 사용자는 5초 이상 대기 후 에러를 본다
      주문 서비스의 스레드 풀이 고갈될 수 있다
      API Gateway까지 영향을 받는다
```

### gRPC: REST보다 효율적인 동기 통신

gRPC는 Protocol Buffers를 사용한 이진 직렬화와 HTTP/2 기반의 멀티플렉싱으로 REST보다 효율적이다.

```protobuf
// order.proto
syntax = "proto3";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc ListOrders (ListOrdersRequest) returns (stream Order);
}

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string id = 1;
  string user_id = 2;
  repeated OrderItem items = 3;
  string status = 4;
  int64 created_at = 5;
}
```

gRPC는 특히 서비스 간 내부 통신에서 강점을 가진다. 스키마가 명확히 정의되고, 타입 안전성이 보장되며, 클라이언트 코드가 자동 생성된다.

스트리밍이 필요한 경우(실시간 로그, 대용량 데이터 전송)에도 gRPC가 유리하다.

### 비동기 메시지 통신: 결합도를 낮추다

비동기 통신에서 호출자는 메시지를 보내고 즉시 다음 작업을 진행한다. 응답을 기다리지 않는다.

메시지 브로커(Kafka, RabbitMQ, NATS)를 통해 이벤트를 발행하고, 관심 있는 서비스가 구독한다.

```python
# Kafka를 통한 이벤트 발행 예시
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

def place_order(order_data: dict):
    # 주문 저장
    order = save_order_to_db(order_data)

    # 이벤트 발행 (결제 서비스가 나중에 처리)
    producer.send('order-events', {
        'event_type': 'ORDER_PLACED',
        'order_id': order.id,
        'user_id': order.user_id,
        'total_amount': order.total_amount,
        'timestamp': order.created_at.isoformat()
    })

    # 즉시 반환 (결제 처리를 기다리지 않음)
    return order
```

```python
# 결제 서비스의 이벤트 소비
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'order-events',
    bootstrap_servers=['kafka:9092'],
    group_id='payment-service'
)

for message in consumer:
    event = json.loads(message.value)
    if event['event_type'] == 'ORDER_PLACED':
        process_payment(event['order_id'], event['total_amount'])
```

비동기 방식은 서비스 간 **시간적 결합을 제거**한다. 결제 서비스가 일시적으로 다운되어도 주문 이벤트는 Kafka에 보관되고, 서비스가 복구되면 처리를 재개한다.

### 어떤 통신을 언제 선택할까

```
동기 통신을 선택하는 경우:
  - 즉시 응답이 필요한 경우 (사용자에게 결과를 바로 보여줘야 할 때)
  - 간단한 CRUD 조회 (다른 서비스의 데이터를 읽어야 할 때)
  - 요청-응답 패턴이 자연스러운 경우

비동기 통신을 선택하는 경우:
  - 처리에 시간이 걸리는 작업 (이메일 발송, 보고서 생성)
  - 여러 서비스에 동일한 이벤트를 알려야 하는 경우
  - 서비스 간 결합을 최소화해야 하는 경우
  - 높은 처리량이 필요한 경우
```

실제 시스템에서는 두 방식을 혼용한다. 사용자 인터페이스와 맞닿은 API는 동기로, 백엔드 처리 파이프라인은 비동기로 구성하는 패턴이 일반적이다.

### 사가(Saga) 패턴: 분산 트랜잭션 대안

분산 환경에서 여러 서비스에 걸친 트랜잭션은 까다롭다. 하나의 DB 트랜잭션으로 묶을 수 없기 때문이다.

**사가 패턴**은 각 서비스가 로컬 트랜잭션을 처리하고 이벤트를 발행하는 방식으로 분산 트랜잭션을 대체한다.

```
주문 처리 사가:

1. 주문 서비스: 주문 생성 (상태: PENDING)
   → 이벤트: ORDER_CREATED 발행

2. 재고 서비스: ORDER_CREATED 수신 → 재고 감소
   → 이벤트: STOCK_RESERVED 발행
   → 실패 시: STOCK_RESERVATION_FAILED 발행

3. 결제 서비스: STOCK_RESERVED 수신 → 결제 처리
   → 이벤트: PAYMENT_COMPLETED 발행
   → 실패 시: PAYMENT_FAILED 발행

4. 주문 서비스: PAYMENT_COMPLETED 수신 → 주문 확정
   → 이벤트: ORDER_CONFIRMED 발행

보상 트랜잭션 (실패 시):
  PAYMENT_FAILED 수신
    → 재고 서비스: 재고 복원
    → 주문 서비스: 주문 취소
```

사가 패턴은 결과적 일관성(Eventual Consistency)을 받아들이는 대신 서비스 간 결합을 최소화한다.

---

## 데이터 소유권과 서비스별 DB 분리

### Database per Service 원칙

MSA에서 가장 중요하면서도 지키기 어려운 원칙 중 하나는 **서비스별 독립 데이터베이스**다.

각 서비스는 자신의 데이터를 직접 소유하고, 다른 서비스는 그 데이터에 직접 접근해서는 안 된다. 반드시 해당 서비스의 API를 통해야 한다.

```
잘못된 방식:
  주문 서비스 ──→ 공유 DB ←── 사용자 서비스
                    ↑
               결제 서비스

올바른 방식:
  주문 서비스 ──→ 주문 DB
  사용자 서비스 ──→ 사용자 DB
  결제 서비스 ──→ 결제 DB

  서비스 간 데이터 접근 = API 호출 또는 이벤트
```

공유 DB는 MSA의 핵심 이점을 모두 무력화한다.

주문 서비스의 테이블 스키마를 바꾸려면 사용자 서비스와 결제 서비스 팀의 동의가 필요해진다. 독립 배포는 불가능해진다.

### 데이터 소유권 경계가 모호할 때

실제로 데이터 소유권 경계가 불명확한 경우가 많다.

"상품 가격"은 상품 서비스 소유인가, 아니면 주문 서비스도 가격 정보를 가져야 하는가?

주문이 생성될 때의 가격은 상품 가격이 나중에 변경되어도 보존되어야 한다. 이 경우 주문 서비스는 주문 시점의 가격을 자신의 DB에 저장해야 한다.

이것이 **데이터 비정규화**다. MSA에서는 전통적인 정규화 원칙보다 서비스 자율성이 우선한다.

```sql
-- 주문 서비스의 주문 아이템 테이블
-- 주문 시점의 가격을 직접 저장 (상품 서비스 의존 없음)
CREATE TABLE order_items (
    id          UUID PRIMARY KEY,
    order_id    UUID NOT NULL,
    product_id  UUID NOT NULL,          -- 참조용 ID (외래키 아님)
    product_name VARCHAR(255) NOT NULL, -- 주문 시점 상품명 복사
    unit_price  DECIMAL(10,2) NOT NULL, -- 주문 시점 가격 복사
    quantity    INT NOT NULL,
    created_at  TIMESTAMP NOT NULL
);
```

데이터 중복을 두려워하지 마라. MSA에서 데이터 중복은 결합도를 낮추기 위한 의도적인 선택이다.

### 서비스별 DB 기술 선택

서비스별 DB 분리의 또 다른 이점은 각 서비스에 맞는 DB 기술을 자유롭게 선택할 수 있다는 점이다.

```
서비스별 최적 DB 선택 예시:

사용자 서비스
  → PostgreSQL (관계형 데이터, 트랜잭션 보장)

상품 카탈로그 서비스
  → Elasticsearch (전문 검색, 필터링 성능)

세션/캐시 서비스
  → Redis (인메모리, 빠른 접근)

이벤트 로그 서비스
  → Cassandra (시계열 데이터, 대용량 쓰기)

추천 서비스
  → Neo4j (그래프 데이터, 관계 탐색)
```

이처럼 **폴리글랏 퍼시스턴스(Polyglot Persistence)**는 MSA의 자연스러운 귀결이다.

### CQRS: 조회와 쓰기의 분리

여러 서비스의 데이터를 조합한 뷰가 필요할 때 어떻게 할 것인가?

예를 들어 "주문 목록 + 사용자 정보 + 상품 정보"를 한 화면에 보여줘야 한다면, 매번 세 서비스를 동기 호출하는 것은 비효율적이다.

**CQRS(Command Query Responsibility Segregation)**와 **읽기 전용 뷰**를 결합하는 방식이 효과적이다.

```
CQRS + 이벤트 소싱 패턴:

1. 각 서비스가 이벤트 발행:
   ORDER_PLACED → {order_id, user_id, items, total}
   USER_UPDATED → {user_id, name, email}
   PRODUCT_UPDATED → {product_id, name, price}

2. 읽기 모델 서비스가 이벤트 소비하여 비정규화된 뷰 생성:
   order_dashboard_view 테이블:
   {order_id, user_name, user_email, items_with_names, total, status}

3. 대시보드 API는 읽기 모델에서 단일 쿼리로 조회
```

쓰기는 각 서비스가 담당하고, 읽기를 위한 별도 모델을 구성하는 방식이다.

### 분산 조인을 피하라

SQL의 JOIN은 단일 DB 내에서 강력하지만, 서비스 경계를 넘은 "분산 조인"은 피해야 한다.

```python
# 안티패턴: 런타임에 여러 서비스를 호출해 조인
def get_order_details(order_id: str):
    order = order_service.get_order(order_id)

    # N+1 문제: 아이템 수만큼 호출 발생
    enriched_items = []
    for item in order.items:
        product = product_service.get_product(item.product_id)  # N번 호출!
        enriched_items.append({**item, 'product_name': product.name})

    user = user_service.get_user(order.user_id)

    return {**order, 'items': enriched_items, 'user': user}
```

이 패턴은 N+1 문제를 분산 환경에서 재현한다. 상품이 100개면 100번의 네트워크 호출이 발생한다.

대신 필요한 데이터를 사전에 비정규화해두거나, 배치로 묶어 한 번에 조회하거나, 앞서 설명한 읽기 모델을 사용해야 한다.

---

## 서비스 메시와 사이드카

### 공통 관심사를 인프라로

인증, TLS 처리, 재시도, 서킷 브레이커, 메트릭 수집 등 모든 서비스에 필요한 공통 기능을 각 서비스마다 구현하는 것은 비효율적이다.

**서비스 메시(Service Mesh)**는 이 공통 관심사를 **사이드카 프록시**로 분리해 인프라 레이어에서 처리한다.

```
사이드카 패턴:

[주문 서비스 Pod]
┌─────────────────────────────┐
│  [주문 서비스 컨테이너]       │
│  + [Envoy 사이드카 프록시]    │
│    - mTLS 처리               │
│    - 재시도 로직              │
│    - 서킷 브레이커            │
│    - 메트릭 수집              │
└─────────────────────────────┘
          ↕ (모든 트래픽 통과)
[결제 서비스 Pod]
┌─────────────────────────────┐
│  [결제 서비스 컨테이너]       │
│  + [Envoy 사이드카 프록시]    │
└─────────────────────────────┘
```

Istio, Linkerd 같은 서비스 메시 솔루션은 각 서비스가 비즈니스 로직에만 집중하도록 해준다.

대신 운영 복잡도가 크게 높아진다는 점을 기억해야 한다. 서비스 메시는 MSA 성숙도가 높은 팀에서 고려할 만한 옵션이다.

---

## 언제 MSA로 전환해야 하는가

Martin Fowler는 이를 명쾌하게 정리했다. "모놀리스를 먼저 만들어라(Monolith First)."

새로운 도메인에서 MSA로 시작하면 서비스 경계를 잘못 잡을 가능성이 높다. 나중에 경계를 바꾸는 비용은 엄청나다.

모놀리스로 시작해 도메인을 깊이 이해한 후, 분리의 필요가 명확히 느껴질 때 점진적으로 서비스를 추출하는 전략이 현실적이다.

```
MSA 전환을 고려할 시점:
  □ 팀이 여러 개로 나뉘어 있고 배포 충돌이 잦다
  □ 특정 기능만 과부하가 걸려 전체를 확장해야 한다
  □ 특정 모듈의 기술 부채가 심각해 다시 작성하고 싶다
  □ 규정상 특정 데이터를 물리적으로 분리해야 한다
  □ 각 팀이 독립적인 기술 선택권을 원한다
```

분리 자체가 목적이 되어선 안 된다. 분리가 해결하는 구체적인 문제가 명확해야 한다.

---

## 참고 자료

- Martin Fowler, [Microservices](https://martinfowler.com/articles/microservices.html), 2014 — MSA 개념의 원전
- Sam Newman, *Building Microservices*, 2nd Edition, O'Reilly, 2021 — 실무 구현의 교과서
- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software*, Addison-Wesley, 2003 — 바운디드 컨텍스트와 서비스 경계의 이론적 기반
- Martin Fowler, [MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html), 2015 — 모놀리스 우선 전략
- Chris Richardson, [Microservices Patterns](https://microservices.io/patterns/index.html) — 사가, CQRS, 이벤트 소싱 등 MSA 패턴 레퍼런스
- Gregor Hohpe & Bobby Woolf, *Enterprise Integration Patterns*, Addison-Wesley, 2003 — 메시지 기반 통신 패턴의 고전
