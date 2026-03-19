---
title: "[서버 애플리케이션 설계] 5편 — API 설계 심화: 현실의 복잡한 요구사항 다루기"
date: 2026-03-18T00:05:00+09:00
draft: false
tags: ["API", "멱등성", "GraphQL", "설계", "서버"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "멱등성 설계, PATCH vs PUT, Batch API, Long Polling, 파일 업로드, API 하위 호환성, Deprecation 정책, 그리고 GraphQL과 REST의 트레이드오프."
---

## 들어가며

4편에서 API 설계의 기본 원칙을 다뤘다. 하지만 현실은 기본 CRUD보다 훨씬 복잡하다.

결제 요청이 중복 전송되면? 대용량 파일은 어떻게 업로드하지? 기존 API를 변경하면 구버전 클라이언트가 깨지는데?

이번 편은 실전에서 마주치는 복잡한 요구사항을 다룬다.

---

## 1. 멱등성 설계 — 안전한 재시도

### 왜 멱등성이 중요한가

```
네트워크 타임아웃 시나리오:

클라이언트 → POST /api/payments → 서버 (결제 처리 성공!)
클라이언트 ← ??? (응답이 네트워크에서 유실)
클라이언트: "응답이 없네? 다시 보내자"
클라이언트 → POST /api/payments → 서버 (이중 결제!)

→ 멱등하지 않은 API + 네트워크 불안정 = 재앙
```

### Idempotency Key 패턴

```
클라이언트가 고유한 키를 생성해서 요청에 포함:

POST /api/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{
    "order_id": 123,
    "amount": 50000,
    "method": "card"
}
```

```
서버 처리 흐름:

1. Idempotency-Key가 있는지 확인
2. 이미 처리된 키인가?
   → YES: 저장된 응답을 그대로 반환 (재처리 없음)
   → NO: 정상 처리 후, 키 + 응답을 저장

┌──────────────────────────────────────────┐
│  Idempotency Store (Redis 또는 DB)       │
├──────────────────────────────────────────┤
│ Key: 550e8400-...                        │
│ Status: completed                        │
│ Response: {"id": 789, "status": "paid"}  │
│ Created: 2026-03-18T10:00:00             │
│ TTL: 24h                                 │
└──────────────────────────────────────────┘
```

```java
@PostMapping("/api/payments")
public ResponseEntity<?> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody PaymentRequest request) {

    // 1. 이미 처리된 요청인지 확인
    Optional<SavedResponse> cached = idempotencyStore.find(idempotencyKey);
    if (cached.isPresent()) {
        return cached.get().toResponseEntity();  // 저장된 응답 반환
    }

    // 2. 동시 요청 방지 (같은 키로 동시에 들어올 수 있음)
    if (!idempotencyStore.tryLock(idempotencyKey)) {
        return ResponseEntity.status(409).body("요청 처리 중입니다");
    }

    try {
        // 3. 실제 처리
        PaymentResult result = paymentService.process(request);

        // 4. 응답 저장
        idempotencyStore.save(idempotencyKey, result, Duration.ofHours(24));
        return ResponseEntity.status(201).body(result);
    } finally {
        idempotencyStore.unlock(idempotencyKey);
    }
}
```

### 어떤 API에 멱등성이 필요한가

```
필수:
- 결제, 송금, 포인트 적립 (금전 관련)
- 주문 생성
- 외부 시스템 호출을 포함하는 요청

권장:
- 모든 POST 요청 (리소스 생성)
- 부작용이 있는 PATCH 요청

불필요:
- GET (이미 멱등)
- PUT, DELETE (이미 멱등)
- 단순 조회 API
```

---

## 2. PATCH vs PUT — 부분 수정의 올바른 설계

### PUT — 전체 교체

```
PUT /api/users/123
{
    "name": "김철수",
    "email": "kim@example.com",
    "phone": "010-1234-5678",
    "address": "서울시 강남구"
}

→ 리소스 전체를 교체. 보내지 않은 필드는 null/기본값이 됨.
→ 이름만 바꾸고 싶어도 모든 필드를 보내야 함.
```

### PATCH — 부분 수정

**JSON Merge Patch (RFC 7396)** — 가장 직관적:

```
PATCH /api/users/123
Content-Type: application/merge-patch+json

{
    "name": "김영희"
}

→ name만 변경, 나머지 필드는 그대로

null을 보내면 필드 삭제:
{
    "phone": null
}
→ phone 필드를 삭제(null로 설정)
```

**JSON Patch (RFC 6902)** — 세밀한 제어:

```
PATCH /api/users/123
Content-Type: application/json-patch+json

[
    { "op": "replace", "path": "/name", "value": "김영희" },
    { "op": "remove", "path": "/phone" },
    { "op": "add", "path": "/tags/-", "value": "VIP" }
]

→ 배열 조작, 조건부 수정 등 복잡한 변경 가능
→ 하지만 대부분의 API에서 과한 복잡도
```

### 실전 권장

```
대부분의 경우: JSON Merge Patch (직관적)
복잡한 문서 편집: JSON Patch (세밀한 제어)
단순한 상태 변경: 전용 엔드포인트
  POST /api/orders/123/cancel
  POST /api/users/123/deactivate
```

---

## 3. Batch API — 일괄 처리

### 왜 Batch가 필요한가

```
100개 항목을 삭제하려면:
- 개별: DELETE /api/items/1, DELETE /api/items/2, ..., DELETE /api/items/100
  → 100번의 HTTP 요청, 100번의 TCP 라운드트립
- Batch: 한 번의 요청으로 100개 처리
```

### Batch 설계 패턴

**방법 1: 쿼리 파라미터**
```
DELETE /api/items?ids=1,2,3,4,5
```
간단하지만 URL 길이 제한이 있다 (보통 2048~8192자).

**방법 2: 요청 본문**
```
POST /api/items/batch-delete
{
    "ids": [1, 2, 3, 4, 5, ...]
}
```

**방법 3: 개별 결과 반환**
```
POST /api/items/batch
{
    "operations": [
        { "method": "DELETE", "id": 1 },
        { "method": "DELETE", "id": 2 },
        { "method": "PATCH", "id": 3, "body": { "status": "active" } }
    ]
}

응답 (각 작업의 성공/실패를 개별 보고):
{
    "results": [
        { "id": 1, "status": 200 },
        { "id": 2, "status": 200 },
        { "id": 3, "status": 404, "error": "항목을 찾을 수 없습니다" }
    ]
}

→ HTTP 상태 코드: 207 Multi-Status (일부 성공, 일부 실패)
```

### Batch 크기 제한

```
반드시 상한을 둔다:
- 요청당 최대 100개 (또는 적절한 숫자)
- 초과 시 400 Bad Request
- 이유: 서버 메모리, 트랜잭션 크기, 응답 시간 제한
```

---

## 4. Long Polling — 실시간에 가까운 데이터

### 일반 폴링의 문제

```
클라이언트: "새 메시지 있어?" → 서버: "없음"  (0.5초 후)
클라이언트: "새 메시지 있어?" → 서버: "없음"  (0.5초 후)
클라이언트: "새 메시지 있어?" → 서버: "없음"  (0.5초 후)
클라이언트: "새 메시지 있어?" → 서버: "있음! [메시지]"
클라이언트: "새 메시지 있어?" → 서버: "없음"
...

→ 대부분의 응답이 "없음". 서버 부하 낭비.
```

### Long Polling

```
클라이언트: "새 메시지 있어?" → 서버: (대기...... 30초간)
                                        ↓ 메시지 도착!
                              → 서버: "있음! [메시지]"
클라이언트: "새 메시지 있어?" → 서버: (대기......)
                                        ↓ 30초 타임아웃
                              → 서버: "없음" (204)
클라이언트: "새 메시지 있어?" → 서버: (대기......)

→ 이벤트가 없으면 연결을 유지하다가, 이벤트 발생 시 즉시 응답
→ 불필요한 요청 대폭 감소
```

```java
@GetMapping("/api/messages/poll")
public DeferredResult<ResponseEntity<?>> pollMessages(
        @RequestParam String channelId,
        @RequestParam(required = false) String lastEventId) {

    DeferredResult<ResponseEntity<?>> result = new DeferredResult<>(30000L);

    // 타임아웃 시 빈 응답
    result.onTimeout(() -> result.setResult(ResponseEntity.noContent().build()));

    // 새 메시지 리스너 등록
    messageService.subscribe(channelId, lastEventId, messages -> {
        result.setResult(ResponseEntity.ok(messages));
    });

    return result;
}
```

### Long Polling vs WebSocket vs SSE

```
Long Polling:
- 단방향 (서버 → 클라이언트)
- HTTP 호환성 최고
- 연결당 비용 (스레드/메모리) 있음
- 용도: 알림, 간단한 실시간 업데이트

SSE (Server-Sent Events):
- 단방향 (서버 → 클라이언트)
- HTTP 기반, 자동 재연결
- 텍스트만 전송 가능
- 용도: 뉴스피드, 주가 업데이트

WebSocket:
- 양방향
- HTTP 업그레이드 후 별도 프로토콜
- 바이너리 전송 가능
- 용도: 채팅, 게임, 협업 도구
```

---

## 5. 파일 업로드 설계

### 소규모 파일 — Multipart

```
POST /api/users/123/avatar
Content-Type: multipart/form-data; boundary=----FormBoundary

------FormBoundary
Content-Disposition: form-data; name="file"; filename="avatar.jpg"
Content-Type: image/jpeg

(바이너리 데이터)
------FormBoundary--
```

### 대용량 파일 — 청크 업로드

```
1단계: 업로드 세션 생성
POST /api/uploads
{ "filename": "backup.zip", "size": 5368709120, "content_type": "application/zip" }

→ 201 Created
{ "upload_id": "abc123", "chunk_size": 5242880 }

2단계: 청크 업로드 (병렬 가능)
PUT /api/uploads/abc123/chunks/0
Content-Range: bytes 0-5242879/5368709120
(바이너리 데이터 5MB)

PUT /api/uploads/abc123/chunks/1
Content-Range: bytes 5242880-10485759/5368709120
(바이너리 데이터 5MB)
...

3단계: 업로드 완료
POST /api/uploads/abc123/complete

→ 서버가 청크를 합치고 최종 파일 생성
```

### Presigned URL — 서버 우회

```
서버를 거치지 않고 클라이언트가 직접 S3에 업로드:

1. 클라이언트 → 서버: "업로드 URL 줘"
   POST /api/upload-urls
   { "filename": "video.mp4", "content_type": "video/mp4" }

2. 서버 → 클라이언트: Presigned URL 반환
   { "upload_url": "https://s3.amazonaws.com/bucket/video.mp4?X-Amz-Signature=...",
     "expires_in": 3600 }

3. 클라이언트 → S3: 직접 업로드
   PUT https://s3.amazonaws.com/bucket/video.mp4?X-Amz-Signature=...
   (바이너리 데이터)

장점: 서버 대역폭/CPU 절약, S3의 멀티파트 업로드 활용
단점: 클라이언트 구현 복잡도 증가
```

---

## 6. API 하위 호환성

### 호환되는 변경 (Breaking하지 않는)

```
✓ 안전한 변경:
- 새 필드 추가 (기존 클라이언트는 무시)
- 새 엔드포인트 추가
- 선택적 파라미터 추가
- 새 enum 값 추가 (클라이언트가 unknown을 처리한다면)
- 에러 메시지 변경

✗ Breaking 변경:
- 필드 제거 또는 이름 변경
- 필드 타입 변경 (string → int)
- 필수 파라미터 추가
- URL 경로 변경
- 상태 코드 변경
- 기존 enum 값 제거
```

### Deprecation 정책

```
1단계: 새 버전 공개 + 구 버전 deprecated 표시
   Sunset: Sat, 01 Jan 2027 00:00:00 GMT
   Deprecation: true
   Link: </api/v2/users>; rel="successor-version"

2단계: 구 버전 사용자에게 경고 (로그, 응답 헤더)
   Warning: 299 - "이 API는 2027-01-01에 종료됩니다. /api/v2를 사용하세요."

3단계: 마이그레이션 기간 (최소 6개월~1년)
   - 사용 통계 모니터링
   - 주요 사용자에게 직접 안내

4단계: 구 버전 종료
   - 410 Gone 반환
   - 새 버전으로 리다이렉트 (선택)
```

---

## 7. GraphQL — REST의 대안

### GraphQL의 핵심 아이디어

```
REST의 문제:
1. Over-fetching: 이름만 필요한데 사용자의 모든 필드를 받음
2. Under-fetching: 사용자 + 주문 + 리뷰를 보려면 3번 요청

GraphQL의 해법: 클라이언트가 필요한 데이터를 정확히 명시
```

```graphql
# 클라이언트가 쿼리 작성
query {
    user(id: 123) {
        name
        email
        orders(last: 5) {
            id
            total
            items {
                productName
                quantity
            }
        }
    }
}

# 서버가 요청한 형태 그대로 응답
{
    "data": {
        "user": {
            "name": "김철수",
            "email": "kim@example.com",
            "orders": [
                {
                    "id": 456,
                    "total": 50000,
                    "items": [
                        { "productName": "키보드", "quantity": 1 }
                    ]
                }
            ]
        }
    }
}
```

### GraphQL의 장단점

```
장점:
- 정확히 필요한 데이터만 요청 → 네트워크 효율
- 하나의 요청으로 여러 리소스 조회 → 라운드트립 감소
- 스키마가 곧 문서 (자기 서술적)
- 프론트엔드 주도 데이터 패칭 → 백엔드 수정 없이 UI 변경 가능

단점:
- 캐싱이 어려움 (POST 요청 + 동적 쿼리 → HTTP 캐시 활용 불가)
- N+1 문제 (중첩 쿼리 시 DB 호출 폭증 → DataLoader 필요)
- 복잡한 쿼리로 서버 과부하 가능 (쿼리 깊이/복잡도 제한 필요)
- 파일 업로드가 표준이 아님
- 에러 처리가 HTTP 상태 코드와 분리 (항상 200 반환)
- 학습 곡선
```

### REST vs GraphQL 선택 기준

```
REST가 나은 경우:
- 단순한 CRUD API
- 공개 API (캐싱 중요)
- 파일 다운로드/업로드 중심
- 팀이 REST에 숙련

GraphQL이 나은 경우:
- 프론트엔드가 다양한 화면에서 다른 데이터 조합 필요
- 모바일 앱 (대역폭 절약 중요)
- 여러 백엔드를 하나의 API로 통합 (API Gateway)
- 빠른 프로토타이핑

혼용 (실무에서 많음):
- 기본: REST
- 복잡한 조회가 필요한 화면: GraphQL
- 또는 BFF(Backend for Frontend) 패턴으로 REST 위에 GraphQL
```

---

## 마치며

API 설계의 심화 주제를 정리하면:

- **멱등성**은 안전한 재시도의 전제 조건이다. 금전 관련 API는 Idempotency Key가 필수다
- **PATCH**는 부분 수정에, **PUT**은 전체 교체에 사용한다. 대부분의 경우 JSON Merge Patch면 충분하다
- **Batch API**는 대량 처리의 효율을 높이되, 반드시 크기 제한을 둔다
- **Long Polling**은 WebSocket 없이도 실시간에 가까운 데이터를 제공한다
- **대용량 파일**은 청크 업로드나 Presigned URL로 처리한다
- **하위 호환성**을 유지하면서 API를 진화시키는 것이 가장 어려운 과제다. 추가는 안전하고, 삭제/변경은 위험하다
- **GraphQL**은 REST의 over/under-fetching을 해결하지만, 캐싱과 복잡성이라는 트레이드오프가 있다

---

## 참고 자료

- RFC 7396 — JSON Merge Patch
- RFC 6902 — JSON Patch
- Stripe API Idempotency — [https://stripe.com/docs/api/idempotent_requests](https://stripe.com/docs/api/idempotent_requests)
- GraphQL Specification — [https://spec.graphql.org/](https://spec.graphql.org/)
- *Learning GraphQL* — Eve Porcello, Alex Banks (O'Reilly)
- Google API Improvement Proposals — [https://google.aip.dev/](https://google.aip.dev/)
