---
title: "[서버 애플리케이션 설계] 4편 — API 설계: 좋은 서버 인터페이스의 조건"
date: 2026-03-18T00:06:00+09:00
draft: false
tags: ["API", "REST", "gRPC", "설계", "서버"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "REST 원칙과 Richardson Maturity Model, HTTP 메서드 의미론, 상태 코드, 버저닝, 페이지네이션, OpenAPI, gRPC 비교까지 — 실전 API 설계의 모든 것."
---

## 들어가며

API는 서버의 얼굴이다. 내부 구현이 아무리 깔끔해도 API가 일관성 없고 직관적이지 않으면, 클라이언트 개발자(프론트엔드, 모바일, 다른 서비스)의 생산성이 떨어지고 버그가 늘어난다.

좋은 API는 **문서를 읽기 전에도 예측 가능**한 API다. 이번 편에서는 RESTful API 설계의 핵심 원칙을 다루고, gRPC와의 트레이드오프까지 살펴본다.

---

## 1. REST의 본질

### REST는 스펙이 아니라 아키텍처 스타일

REST(Representational State Transfer)는 HTTP API의 규격서가 아니다. Roy Fielding의 박사 논문에서 정의한 **아키텍처 제약 조건**이다:

```
REST의 6가지 제약:
1. Client-Server: 클라이언트와 서버의 관심사 분리
2. Stateless: 각 요청은 독립적, 서버가 클라이언트 상태를 저장하지 않음
3. Cacheable: 응답은 캐시 가능 여부를 명시해야 함
4. Uniform Interface: 통일된 인터페이스 (리소스 식별, 표현을 통한 조작, 자기 서술적 메시지)
5. Layered System: 계층 구조 허용 (프록시, 로드밸런서)
6. Code on Demand (선택): 서버가 클라이언트에 코드를 전송할 수 있음
```

실무에서 "RESTful API"라 부르는 것은 대부분 REST의 일부만 따른다.

그래도 핵심 원칙을 이해하면 더 나은 설계를 할 수 있다.

### Richardson Maturity Model

REST 성숙도를 4단계로 나눈 모델이다:

```
Level 0: The Swamp of POX
- 하나의 엔드포인트, 하나의 HTTP 메서드
- POST /api에 모든 요청
- 예: SOAP, XML-RPC

Level 1: Resources
- URL로 리소스를 구분
- POST /api/users, POST /api/orders
- 하지만 HTTP 메서드를 제대로 사용하지 않음

Level 2: HTTP Verbs ★ (대부분의 실무 API가 여기)
- HTTP 메서드를 의미에 맞게 사용
- GET /users, POST /users, PUT /users/123, DELETE /users/123
- 상태 코드도 의미에 맞게 사용

Level 3: Hypermedia Controls (HATEOAS)
- 응답에 다음 가능한 액션의 링크 포함
- 거의 사용되지 않음 (실용성 논란)
```

**실무 권장**: Level 2를 목표로 하되, 엄격한 REST 순수주의보다 **일관성과 실용성**을 우선한다.

---

## 2. 리소스 설계와 URL 구조

### 좋은 URL의 원칙

```
1. 명사를 사용한다 (동사 아님)
   ✗ POST /api/getUsers
   ✗ POST /api/createUser
   ✓ GET  /api/users
   ✓ POST /api/users

2. 복수형을 사용한다
   ✗ /api/user/123
   ✓ /api/users/123

3. 계층 관계를 URL 경로로 표현한다
   ✓ /api/users/123/orders          (사용자 123의 주문 목록)
   ✓ /api/users/123/orders/456      (사용자 123의 주문 456)

4. 2단계 이상 중첩은 피한다
   ✗ /api/users/123/orders/456/items/789/reviews
   ✓ /api/order-items/789/reviews

5. 케밥 케이스를 사용한다
   ✗ /api/orderItems
   ✗ /api/order_items
   ✓ /api/order-items
```

### 리소스가 아닌 동작은?

모든 것을 리소스로 표현하기 어려운 경우가 있다:

```
결제 처리:
  ✗ PUT /api/orders/123 { "status": "paid" }     (상태 변경으로 억지 표현)
  ✓ POST /api/orders/123/payments                 (하위 리소스로 표현)

이메일 발송:
  ✓ POST /api/users/123/verification-emails       (리소스 생성으로 표현)

검색:
  ✓ GET /api/users/search?q=kim&role=admin        (search를 하위 경로로)

일괄 작업:
  ✓ POST /api/users/batch-delete                  (동사가 불가피한 경우)
  ✓ DELETE /api/users?ids=1,2,3                   (쿼리 파라미터 활용)
```

---

## 3. HTTP 메서드 의미론

### 각 메서드의 정확한 의미

```
GET     리소스 조회. 부작용 없음. 캐시 가능.
POST    리소스 생성 또는 복잡한 동작 실행. 멱등하지 않음.
PUT     리소스 전체 교체. 멱등. 리소스가 없으면 생성할 수도 있음.
PATCH   리소스 부분 수정. (보통 멱등하게 설계)
DELETE  리소스 삭제. 멱등.
```

```
멱등(Idempotent)이란?
같은 요청을 여러 번 보내도 결과가 동일한 것.
전등 스위치에 비유하면: 켜진 전등을 다시 "켜기" 해도 결과는 여전히 켜진 상태다.

GET  /users/123     → 항상 같은 사용자 반환           ✓ 멱등
PUT  /users/123     → 같은 데이터로 덮어씌우기         ✓ 멱등
DELETE /users/123   → 처음: 삭제, 이후: 이미 없음(404) ✓ 멱등
POST /users         → 호출할 때마다 새 사용자 생성      ✗ 멱등하지 않음

멱등성이 중요한 이유: 네트워크 오류로 요청이 중복 전송될 수 있음
→ 멱등한 API는 재시도해도 안전
```

### 실전 CRUD 매핑

```
리소스: 사용자(User)

GET    /api/users              사용자 목록 조회
GET    /api/users/123          사용자 상세 조회
POST   /api/users              사용자 생성
PUT    /api/users/123          사용자 정보 전체 수정
PATCH  /api/users/123          사용자 정보 부분 수정
DELETE /api/users/123          사용자 삭제

리소스: 사용자의 주문(Order)

GET    /api/users/123/orders           주문 목록
POST   /api/users/123/orders           주문 생성
GET    /api/orders/456                 주문 상세 (전역 접근도 제공)
PATCH  /api/orders/456                 주문 수정
DELETE /api/orders/456                 주문 취소
```

---

## 4. 상태 코드 올바른 사용

### 자주 사용하는 상태 코드

```
2xx 성공:
200 OK              일반적 성공 (GET, PUT, PATCH, DELETE)
201 Created         리소스 생성 성공 (POST). Location 헤더에 새 리소스 URL
204 No Content      성공이지만 응답 본문 없음 (DELETE)

3xx 리다이렉트:
301 Moved Permanently  영구 이동
304 Not Modified       캐시 유효 (ETag/Last-Modified 조건부 요청)

4xx 클라이언트 오류:
400 Bad Request     잘못된 요청 (유효성 검증 실패, 잘못된 JSON)
401 Unauthorized    인증 필요 (로그인 안 됨)
403 Forbidden       인가 실패 (권한 없음)
404 Not Found       리소스 없음
405 Method Not Allowed  허용되지 않은 HTTP 메서드
409 Conflict        충돌 (중복 생성, 동시 수정 충돌)
422 Unprocessable Entity  문법은 맞지만 의미적으로 처리 불가
429 Too Many Requests    요청 속도 제한 초과

5xx 서버 오류:
500 Internal Server Error  서버 내부 오류
502 Bad Gateway     업스트림 서버 오류
503 Service Unavailable  일시적 서비스 불가 (점검, 과부하)
504 Gateway Timeout 업스트림 응답 시간 초과
```

### 흔한 실수

```
✗ 모든 에러에 200 + error 필드
  { "status": 200, "error": "사용자를 찾을 수 없습니다" }
  → HTTP 상태 코드를 무시하면 캐시, 모니터링, 클라이언트 라이브러리가 제대로 동작하지 않음

✗ 400과 422를 구분하지 않음
  400: JSON 파싱 실패, 필수 필드 누락 (문법 오류)
  422: JSON은 유효하지만 "나이: -5" 같은 의미적 오류

✗ 401과 403을 혼동
  401: "누구세요?" → 로그인하세요
  403: "당신인 건 알겠는데, 권한이 없습니다" → 다른 계정으로 시도
```

---

## 5. 에러 응답 표준화 — RFC 7807

### Problem Details 형식

```json
// RFC 7807 (RFC 9457로 업데이트) — Problem Details for HTTP APIs
{
    "type": "https://api.example.com/errors/insufficient-balance",
    "title": "잔액이 부족합니다",
    "status": 400,
    "detail": "주문 금액 50,000원에 대해 현재 잔액이 12,000원입니다",
    "instance": "/api/orders/789",
    "balance": 12000,
    "required": 50000
}
```

```
필드 설명:
type:     에러 유형 식별 URI (문서 링크로 활용 가능)
title:    사람이 읽을 수 있는 에러 요약
status:   HTTP 상태 코드
detail:   구체적 에러 설명
instance: 에러가 발생한 리소스 경로
+ 추가 필드: 도메인 특화 정보 (balance, required 등)
```

### 유효성 검증 에러

```json
{
    "type": "https://api.example.com/errors/validation-error",
    "title": "입력값 검증 실패",
    "status": 422,
    "detail": "2개의 필드에서 검증 오류가 발생했습니다",
    "errors": [
        {
            "field": "email",
            "message": "올바른 이메일 형식이 아닙니다",
            "rejected_value": "not-an-email"
        },
        {
            "field": "age",
            "message": "0 이상이어야 합니다",
            "rejected_value": -5
        }
    ]
}
```

---

## 6. 버저닝 전략

### URL 경로 버저닝 (가장 일반적)

```
GET /api/v1/users/123
GET /api/v2/users/123
```

- 장점: 명확하고 직관적, 브라우저에서 바로 테스트 가능
- 단점: URL이 바뀌므로 캐시 키가 달라짐
- 사용: GitHub API, Stripe API, Google Cloud API

### 헤더 버저닝

```
GET /api/users/123
Accept: application/vnd.myapi.v2+json
```

- 장점: URL이 깨끗, 같은 리소스에 대한 다른 표현
- 단점: 테스트/디버깅이 불편, 브라우저에서 바로 확인 불가

### 쿼리 파라미터 버저닝

```
GET /api/users/123?version=2
```

- 장점: URL 변경 최소화
- 단점: 캐시 키 관리 복잡

### 실전 권장

```
공개 API (외부 개발자용):
→ URL 경로 버저닝 (/v1, /v2)
→ 명확하고 문서화 용이

내부 API (마이크로서비스 간):
→ 하위 호환을 유지하면서 점진적으로 진화
→ 버저닝보다 호환성 있는 변경(additive change)을 우선
→ 필드 추가는 OK, 필드 삭제/변경은 deprecation 후 제거
```

---

## 7. 페이지네이션

### Offset 기반 (전통적)

```
GET /api/users?page=3&size=20

응답:
{
    "content": [...],
    "page": 3,
    "size": 20,
    "total_elements": 1523,
    "total_pages": 77
}
```

```
SQL: SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 40

장점:
- 구현 단순
- 특정 페이지로 바로 이동 가능
- total_elements 제공 가능

단점:
- OFFSET이 커지면 성능 저하 (DB가 N개를 읽고 버림)
- 데이터가 추가/삭제되면 페이지가 밀림 (중복/누락)
```

### Cursor 기반 (현대적)

```
GET /api/users?size=20&after=eyJpZCI6MTIzfQ==

응답:
{
    "content": [...],
    "cursors": {
        "after": "eyJpZCI6MTQzfQ==",
        "has_more": true
    }
}
```

```
SQL: SELECT * FROM users WHERE id > 123 ORDER BY id LIMIT 20

cursor = Base64({"id": 123}) — 마지막으로 본 항목의 식별자

장점:
- 일정한 성능 (OFFSET 없이 WHERE 조건)
- 실시간 데이터에서도 중복/누락 없음

단점:
- 특정 페이지로 점프 불가
- total_elements를 알기 어려움 (별도 COUNT 필요)
- 구현이 복잡 (정렬 기준이 여러 개일 때)
```

### 선택 기준

```
관리자 목록 (테이블 UI, 페이지 번호 필요):
→ Offset 기반
→ 데이터가 수만 건 이하이면 성능 문제 없음

피드/타임라인 (무한 스크롤):
→ Cursor 기반
→ 실시간 데이터 추가에 안정적

대규모 데이터 API:
→ Cursor 기반 필수
→ OFFSET 100만은 치명적 성능 문제
```

---

## 8. 필터와 정렬

### 필터 설계

```
단순 필터 (equality):
GET /api/users?role=admin&status=active

범위 필터:
GET /api/orders?created_after=2026-01-01&created_before=2026-03-31
GET /api/products?price_min=10000&price_max=50000

복합 필터 (LHS Brackets 방식):
GET /api/users?age[gte]=20&age[lte]=30&name[contains]=kim

검색:
GET /api/users?q=김철수                    (전문 검색)
GET /api/users?search=kim&search_fields=name,email  (필드 지정)
```

### 정렬 설계

```
단일 정렬:
GET /api/users?sort=created_at&order=desc

다중 정렬:
GET /api/users?sort=role,-created_at
# role ASC, created_at DESC (-는 내림차순)

# 또는
GET /api/users?sort=role:asc,created_at:desc
```

---

## 9. API 문서화 — OpenAPI (Swagger)

### OpenAPI 스펙

```yaml
# openapi.yaml
openapi: 3.0.3
info:
  title: User API
  version: 1.0.0

paths:
  /api/users:
    get:
      summary: 사용자 목록 조회
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 0
        - name: size
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserListResponse'

    post:
      summary: 사용자 생성
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: 생성 완료
          headers:
            Location:
              schema:
                type: string
                example: /api/users/123
        '422':
          description: 유효성 검증 실패
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ValidationError'
```

### Code-First vs Design-First

```
Code-First (코드에서 문서 생성):
- Java 코드 + Swagger 어노테이션 → OpenAPI 문서 자동 생성
- Spring: springdoc-openapi
- 장점: 코드와 문서가 항상 일치
- 단점: API 설계 논의가 코드 중심이 됨

Design-First (문서를 먼저 작성):
- OpenAPI YAML 작성 → 코드 생성 (또는 수동 구현)
- 장점: API 설계를 먼저 검토/합의 가능
- 단점: 코드와 문서의 불일치 위험

권장: 팀 내부 API는 Code-First, 공개 API는 Design-First
```

---

## 10. gRPC vs REST

### gRPC의 핵심

```
REST:
- 텍스트 (JSON)
- HTTP/1.1 또는 HTTP/2
- 사람이 읽기 쉬움
- 브라우저에서 직접 호출 가능

gRPC:
- 바이너리 (Protocol Buffers)
- HTTP/2 필수
- 사람이 읽기 어려움
- 전용 클라이언트 필요
```

### Protocol Buffers 정의

```protobuf
// user.proto
syntax = "proto3";

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);  // 서버 스트리밍
    rpc CreateUser(CreateUserRequest) returns (User);
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    google.protobuf.Timestamp created_at = 4;
}

message GetUserRequest {
    int64 id = 1;
}
```

### gRPC의 장점

```
1. 성능: Protobuf는 JSON 대비 직렬화/역직렬화 2~10배 빠름, 크기 50~80% 작음
2. 타입 안전: .proto 파일에서 클라이언트/서버 코드 자동 생성
3. 스트리밍: 서버→클라이언트, 클라이언트→서버, 양방향 스트리밍 지원
4. 코드 생성: 다양한 언어 지원 (Java, Go, Python, C++, ...)
5. Deadline/Timeout: 요청 단위 타임아웃 전파
```

### 선택 기준

```
REST를 선택할 때:
- 공개 API (외부 개발자용)
- 브라우저에서 직접 호출
- 단순한 CRUD
- 팀이 REST에 익숙

gRPC를 선택할 때:
- 마이크로서비스 간 내부 통신
- 높은 처리량과 낮은 지연이 중요
- 스트리밍이 필요
- 다양한 언어의 클라이언트가 존재

하이브리드 (실무에서 많음):
- 외부: REST (JSON)
- 내부: gRPC (Protobuf)
- API Gateway가 REST ↔ gRPC 변환
```

---

## 마치며

API 설계는 한 번 공개하면 바꾸기 어렵다. 처음부터 일관된 원칙을 세우는 것이 중요하다.

- **리소스 중심 URL**과 **HTTP 메서드의 의미**를 지키면 API가 예측 가능해진다
- **상태 코드**를 올바르게 사용하면 클라이언트가 에러를 프로그래밍적으로 처리할 수 있다
- **RFC 7807**로 에러 응답을 표준화하면 클라이언트 개발이 편해진다
- **페이지네이션**은 UI 패턴(페이지 번호 vs 무한 스크롤)에 맞게 Offset 또는 Cursor를 선택한다
- **OpenAPI**로 문서를 자동화하고, 공개 API는 Design-First를 고려한다
- **gRPC**는 내부 통신에서 성능과 타입 안전성을 제공하며, REST와 상호 보완적이다

---

## 참고 자료

- Roy Fielding, "Architectural Styles and the Design of Network-based Software Architectures" (2000)
- *RESTful Web APIs* — Leonard Richardson, Mike Amundsen (O'Reilly)
- RFC 9457 — Problem Details for HTTP APIs
- OpenAPI Specification — [https://spec.openapis.org/oas/v3.1.0](https://spec.openapis.org/oas/v3.1.0)
- gRPC Documentation — [https://grpc.io/docs/](https://grpc.io/docs/)
- Google API Design Guide — [https://cloud.google.com/apis/design](https://cloud.google.com/apis/design)
- Microsoft REST API Guidelines — [https://github.com/microsoft/api-guidelines](https://github.com/microsoft/api-guidelines)
