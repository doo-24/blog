---
title: "[Frontend 통합] 3편 — API 설계: 프론트엔드가 쓰기 좋은 API의 조건"
date: 2026-03-17T12:05:00+09:00
draft: false
tags: ["GraphQL", "페이지네이션", "API 버저닝", "에러 응답", "서버"]
series: ["Frontend 통합"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 14
summary: "프론트엔드 친화 페이지네이션(커서 기반, 무한 스크롤), 에러 응답 표준화, GraphQL 실전(스키마 설계, N+1 문제, DataLoader), API 버저닝과 하위 호환성까지"
---

프론트엔드 개발자가 API를 쓰다가 막히는 순간은 공통적이다.

페이지를 넘기면 데이터가 튀고, 에러가 왔는데 뭘 보여줄지 모르겠고, 새 버전 배포했더니 앱이 깨진다.

이 글은 서버 엔지니어 관점에서 프론트가 실제로 편하게 쓸 수 있는 API를 어떻게 설계하는지 다룬다.

---

## 1. 커서 기반 페이지네이션

### 오프셋 방식의 문제

가장 흔한 페이지네이션 구현은 `LIMIT`과 `OFFSET`을 쓰는 것이다.

```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 40;
```

단순하고 이해하기 쉽지만 문제가 있다.

사용자가 2페이지를 보는 사이 새 글이 올라오면, `OFFSET 20`의 기준이 밀려서 이전 페이지 마지막 글이 다시 나온다.

데이터가 많아질수록 성능도 나빠진다. `OFFSET 10000`이면 DB는 10020개를 읽은 뒤 앞 10000개를 버린다. 인덱스를 아무리 잘 만들어도 OFFSET이 크면 스킵해야 할 행을 물리적으로 읽어야 하기 때문에 피할 수 없는 구조적 비용이다.

### 커서 기반 방식

커서 기반은 "마지막으로 본 항목 이후"를 기준으로 조회한다.

```sql
SELECT * FROM posts
WHERE created_at < :cursor
ORDER BY created_at DESC
LIMIT 20;
```

커서는 보통 마지막 항목의 ID나 타임스탬프다.

데이터가 중간에 끼어들어도 기준이 흔들리지 않는다.

인덱스를 타기 때문에 오프셋 대비 성능이 훨씬 좋다.

### API 응답 구조

커서 정보를 응답에 포함시켜야 프론트가 다음 요청을 만들 수 있다.

```json
{
  "data": [
    { "id": "post_100", "title": "...", "createdAt": "2026-03-17T10:00:00Z" },
    { "id": "post_99",  "title": "...", "createdAt": "2026-03-17T09:50:00Z" }
  ],
  "pagination": {
    "hasNextPage": true,
    "nextCursor": "eyJpZCI6InBvc3RfOTkiLCJjcmVhdGVkQXQiOiIyMDI2LTAzLTE3VDA5OjUwOjAwWiJ9"
  }
}
```

커서는 Base64로 인코딩한 JSON을 많이 쓴다.

불투명(opaque) 문자열로 취급해서 프론트가 파싱하지 않도록 하는 게 좋다.

커서 내부 구현이 바뀌어도 API 인터페이스가 유지된다.

### 서버 구현 예시 (Node.js)

```typescript
interface CursorPayload {
  id: string;
  createdAt: string;
}

function encodeCursor(payload: CursorPayload): string {
  return Buffer.from(JSON.stringify(payload)).toString('base64');
}

function decodeCursor(cursor: string): CursorPayload {
  return JSON.parse(Buffer.from(cursor, 'base64').toString('utf-8'));
}

async function getPosts(limit: number, cursor?: string) {
  let whereClause = {};

  if (cursor) {
    const decoded = decodeCursor(cursor);
    whereClause = {
      OR: [
        { createdAt: { lt: new Date(decoded.createdAt) } },
        {
          createdAt: new Date(decoded.createdAt),
          id: { lt: decoded.id },
        },
      ],
    };
  }

  // limit + 1개를 가져와서 다음 페이지 존재 여부를 확인한다
  const posts = await prisma.post.findMany({
    where: whereClause,
    orderBy: [{ createdAt: 'desc' }, { id: 'desc' }],
    take: limit + 1,
  });

  const hasNextPage = posts.length > limit;
  const data = hasNextPage ? posts.slice(0, limit) : posts;

  const nextCursor = hasNextPage
    ? encodeCursor({
        id: data[data.length - 1].id,
        createdAt: data[data.length - 1].createdAt.toISOString(),
      })
    : null;

  return { data, pagination: { hasNextPage, nextCursor } };
}
```

`limit + 1` 트릭은 단 한 번의 쿼리로 다음 페이지 존재 여부를 확인하는 일반적인 패턴이다. `limit`개만 요청하면 다음 페이지가 있는지 알 방법이 없어 추가 COUNT 쿼리가 필요한데, 하나 더 가져와 실제로 잘라냄으로써 COUNT 없이 `hasNextPage`를 결정할 수 있다.

### Relay-style Connection

GraphQL 생태계에서는 Relay가 정의한 Connection 스펙을 많이 따른다.

```graphql
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  node: Post!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  posts(first: Int, after: String, last: Int, before: String): PostConnection!
}
```

`edges`와 `node`로 한 번 감싸는 게 번거로워 보이지만, 엣지 자체에 메타데이터(예: 연결 날짜, 정렬 순서)를 붙일 수 있어 확장성이 높다.

프론트엔드 프레임워크(Relay, Apollo)가 이 스펙에 맞춰 캐싱과 페이지네이션을 자동 처리한다.

---

## 2. 에러 응답 표준화

### 에러가 일관성 없으면 프론트가 힘들다

HTTP 상태 코드만으로는 부족하다.

`400 Bad Request`라고만 오면 프론트가 어떤 필드가 문제인지, 사용자에게 뭐라고 보여줄지 알 수 없다.

서버마다 에러 구조가 다르면 프론트는 엔드포인트마다 다른 파싱 로직을 만들어야 한다.

### 표준화된 에러 응답 구조

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "입력값 검증에 실패했습니다.",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "올바른 이메일 형식이 아닙니다."
      },
      {
        "field": "password",
        "code": "TOO_SHORT",
        "message": "비밀번호는 8자 이상이어야 합니다."
      }
    ],
    "traceId": "abc-123-xyz"
  }
}
```

`code`는 기계가 읽는 값이다. 프론트는 이걸 보고 분기한다.

`message`는 사람이 읽는 값이다. 디버깅용이고, UI에 그대로 노출하는 건 지양한다.

`details`는 필드 단위 에러다. 폼 유효성 검사 결과를 보여줄 때 필수다.

`traceId`는 운영팀이 로그를 추적할 때 쓴다. 프론트가 이걸 "요청 ID: abc-123-xyz를 지원팀에 알려주세요"처럼 표시하면 디버깅이 쉬워진다.

### 에러 코드 체계 설계

에러 코드를 체계 없이 만들면 나중에 관리가 안 된다.

도메인별로 네임스페이스를 나누는 방식을 추천한다.

```
AUTH_INVALID_CREDENTIALS      -- 인증 실패
AUTH_TOKEN_EXPIRED            -- 토큰 만료
AUTH_INSUFFICIENT_PERMISSIONS -- 권한 부족

USER_NOT_FOUND                -- 사용자 없음
USER_ALREADY_EXISTS           -- 중복 가입

POST_NOT_FOUND                -- 게시글 없음
POST_FORBIDDEN                -- 수정 권한 없음

VALIDATION_ERROR              -- 입력값 오류 (details에 필드별 정보)
RATE_LIMIT_EXCEEDED           -- 요청 한도 초과
INTERNAL_ERROR                -- 서버 내부 오류
```

에러 코드 목록은 문서로 관리하고, 프론트와 미리 합의한다.

### HTTP 상태 코드와 에러 코드의 관계

```
400 → VALIDATION_ERROR, INVALID_INPUT
401 → AUTH_INVALID_CREDENTIALS, AUTH_TOKEN_EXPIRED
403 → AUTH_INSUFFICIENT_PERMISSIONS, POST_FORBIDDEN
404 → USER_NOT_FOUND, POST_NOT_FOUND
409 → USER_ALREADY_EXISTS, CONFLICT
429 → RATE_LIMIT_EXCEEDED
500 → INTERNAL_ERROR
```

HTTP 상태 코드는 큰 분류고, 에러 코드는 세분화된 이유다.

프론트는 상태 코드로 1차 분기하고, 에러 코드로 2차 분기하면 된다.

### 프론트가 에러를 다루는 방식에서 역으로 설계하기

프론트가 에러 응답으로 해야 하는 일은 크게 세 가지다.

첫째, 특정 필드에 에러 메시지를 표시한다. `details[].field`가 필요하다.

둘째, 토스트나 알림으로 전역 에러를 보여준다. 에러 코드별 메시지 매핑이 필요하다.

셋째, 재시도 여부를 결정한다. `Retry-After` 헤더나 `retryable` 플래그가 필요하다.

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "요청이 너무 많습니다.",
    "retryable": true,
    "retryAfter": 30
  }
}
```

---

## 3. GraphQL 실전

### REST vs GraphQL 선택 기준

REST는 명확한 리소스 경계가 있고 캐싱이 중요할 때 유리하다.

GraphQL은 프론트가 필요한 데이터 모양이 화면마다 크게 다를 때 유리하다.

모바일과 웹이 같은 API를 쓰는데 응답 크기를 최적화해야 할 때도 GraphQL이 빛난다.

### 스키마 설계 원칙

타입 이름은 도메인 언어를 그대로 쓴다.

```graphql
type User {
  id: ID!
  email: String!
  displayName: String!
  avatarUrl: String
  createdAt: DateTime!
  posts(first: Int, after: String): PostConnection!
  followersCount: Int!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  tags: [Tag!]!
  publishedAt: DateTime
  isPublished: Boolean!
}
```

`!`(Non-null)를 적극적으로 쓴다. null이 아님을 보장하면 프론트에서 옵셔널 체이닝을 줄일 수 있다.

Input 타입을 분리한다. Create와 Update는 필수 필드가 다르다.

```graphql
input CreatePostInput {
  title: String!
  content: String!
  tags: [String!]
}

input UpdatePostInput {
  title: String
  content: String
  tags: [String!]
}

type Mutation {
  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
}
```

### N+1 문제

GraphQL의 가장 유명한 함정이다.

```graphql
query {
  posts(first: 10) {
    edges {
      node {
        title
        author {
          displayName  # posts 10개면 author 쿼리가 10번 나간다
        }
      }
    }
  }
}
```

단순 구현이면 `posts` 1번, `author` 10번, 총 11개의 쿼리가 실행된다.

### DataLoader로 해결하기

DataLoader는 여러 개의 개별 요청을 배치 쿼리 하나로 묶는다.

```typescript
import DataLoader from 'dataloader';

// 배치 함수: 여러 userID를 한 번에 받아서 한 번의 쿼리로 조회
const userLoader = new DataLoader<string, User>(async (userIds) => {
  const users = await prisma.user.findMany({
    where: { id: { in: [...userIds] } },
  });

  // DataLoader는 입력 순서와 출력 순서가 일치해야 한다
  const userMap = new Map(users.map((u) => [u.id, u]));
  return userIds.map((id) => userMap.get(id) ?? new Error(`User not found: ${id}`));
});

// 리졸버에서 사용
const resolvers = {
  Post: {
    author: (post: Post, _: unknown, context: Context) => {
      return context.loaders.user.load(post.authorId);
    },
  },
};
```

이제 10개 포스트를 조회해도 `author` 쿼리는 1번만 나간다.

DataLoader는 요청 사이클(이벤트 루프 틱) 단위로 배치를 모은다.

컨텍스트당 새 인스턴스를 만들어야 한다. 그래야 다른 사용자 요청 간에 캐시가 공유되지 않는다.

```typescript
// 각 요청마다 새 DataLoader 인스턴스를 생성
function createContext(req: Request) {
  return {
    loaders: {
      user: new DataLoader(batchUsers),
      post: new DataLoader(batchPosts),
    },
  };
}
```

### Query Depth와 Complexity 제한

GraphQL은 중첩 쿼리를 허용하기 때문에 악의적이거나 실수로 만든 쿼리가 서버를 죽일 수 있다.

```graphql
# 이런 쿼리를 막아야 한다
query {
  user(id: "1") {
    followers {
      followers {
        followers {
          # 무한 중첩...
        }
      }
    }
  }
}
```

depth 제한과 complexity 제한을 함께 건다.

```typescript
import depthLimit from 'graphql-depth-limit';
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  schema,
  validationRules: [
    depthLimit(7),                          // 최대 7단계 중첩
    createComplexityLimitRule(1000),         // 복잡도 점수 1000 초과 거부
  ],
});
```

각 필드에 복잡도 점수를 부여하고, 쿼리 실행 전에 합산해서 제한을 초과하면 거부한다.

### Subscription으로 실시간 데이터 전달

REST에서 실시간 업데이트는 폴링이나 SSE로 구현한다.

GraphQL은 Subscription을 언어 수준에서 지원한다.

```graphql
type Subscription {
  commentAdded(postId: ID!): Comment!
  notificationReceived(userId: ID!): Notification!
}
```

```typescript
const resolvers = {
  Subscription: {
    commentAdded: {
      subscribe: (_: unknown, { postId }: { postId: string }, context: Context) => {
        // 특정 채널만 구독
        return pubSub.asyncIterator(`COMMENT_ADDED:${postId}`);
      },
    },
  },
  Mutation: {
    createComment: async (_: unknown, { input }: any, context: Context) => {
      const comment = await createComment(input);

      // Mutation 성공 시 Subscription 트리거
      pubSub.publish(`COMMENT_ADDED:${comment.postId}`, {
        commentAdded: comment,
      });

      return comment;
    },
  },
};
```

프론트는 WebSocket을 통해 실시간으로 댓글을 받는다.

폴링보다 서버 부하가 낮고 latency가 짧다.

### Persisted Query로 프로덕션 최적화

프로덕션에서는 쿼리 문자열 전체를 매번 전송하는 게 비효율적이다.

Persisted Query는 쿼리를 서버에 미리 등록하고, 이후 요청에선 해시만 보낸다.

```typescript
// 클라이언트가 첫 번째 요청에서 쿼리를 등록
// POST /graphql
{
  "extensions": {
    "persistedQuery": {
      "version": 1,
      "sha256Hash": "abc123..."
    }
  }
}

// 이후 요청에선 해시만 전송 (쿼리 문자열 없음)
```

페이로드 크기가 줄고, 서버에서 화이트리스트 쿼리만 허용하는 보안 이점도 있다.

---

## 4. API 버저닝과 하위 호환성

### 프론트 배포와의 타이밍 문제

서버를 먼저 배포하면 구 버전 앱이 새 API를 만난다.

앱을 먼저 배포하면 새 앱이 구 API를 만난다.

모바일 앱은 사용자가 업데이트를 안 하면 구 버전이 계속 서버에 요청을 보낸다.

이 문제를 해결하는 방법이 버저닝과 하위 호환성 유지다.

### URL 기반 버저닝

가장 명시적이고 이해하기 쉬운 방법이다.

```
/api/v1/users
/api/v2/users
```

주요 변경(Breaking Change)이 생길 때 버전을 올린다.

구 버전을 일정 기간 유지하다가 deprecated → sunset 순서로 종료한다.

```
Deprecation-Notice: 2026-06-01
Sunset: 2026-09-01
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

`Deprecation`과 `Sunset` 헤더는 RFC 8594에 정의된 표준이다.

### 헤더 기반 버저닝

URL을 깔끔하게 유지하고 싶을 때 사용한다.

```
Accept: application/vnd.example.v2+json
API-Version: 2026-03-01
```

Stripe가 날짜 기반 API 버저닝으로 유명하다.

계정마다 "이 계정은 2025-01-15 버전의 API 동작을 사용한다"고 고정한다.

새 기능은 추가되지만, 기존 동작은 해당 날짜 버전 기준으로 유지된다.

### 하위 호환성을 유지하는 변경

하위 호환성을 깨는 변경(Breaking Change)과 안전한 변경을 구분해야 한다.

**안전한 변경 (Non-breaking)**:
- 새 필드 추가
- 선택적 파라미터 추가
- 새 API 엔드포인트 추가
- 에러 코드 추가

**하위 호환성을 깨는 변경 (Breaking Change)**:
- 필드 삭제 또는 이름 변경
- 필드 타입 변경 (예: `string` → `array`)
- 필수 파라미터 추가
- 기존 에러 코드 의미 변경
- 응답 구조 재편

```typescript
// 나쁜 예: 필드 이름을 바꾸면 기존 클라이언트가 깨진다
// Before
{ "userName": "홍길동" }

// After (Breaking Change)
{ "displayName": "홍길동" }

// 좋은 예: 과도기에 두 필드를 모두 보낸다
{ "userName": "홍길동", "displayName": "홍길동" }
// 충분한 기간 후 userName 제거
```

### Expand 패턴으로 유연하게 대응하기

응답에 포함할 관련 리소스를 클라이언트가 선택하게 한다.

```
GET /api/v1/posts?expand=author,tags
GET /api/v1/posts?expand=author.followers
```

기본 응답은 가볍게 유지하고, 필요한 곳에서만 추가 데이터를 요청한다.

버전 없이도 클라이언트마다 다른 응답 모양을 지원할 수 있다.

```typescript
async function getPost(id: string, expand: string[]) {
  const post = await prisma.post.findUnique({ where: { id } });

  if (expand.includes('author')) {
    post.author = await prisma.user.findUnique({ where: { id: post.authorId } });
  }

  if (expand.includes('tags')) {
    post.tags = await prisma.tag.findMany({ where: { postId: id } });
  }

  return post;
}
```

### 버전 종료 전략

버전을 영원히 유지할 수는 없다.

종료 절차를 명확히 해야 프론트팀이 대응할 수 있다.

1단계: Deprecation 헤더를 추가하고 문서에 공지한다.
2단계: 해당 버전의 활성 사용자를 모니터링한다.
3단계: Sunset 날짜 전에 주요 사용자(파트너, 내부팀)에게 직접 통지한다.
4단계: Sunset 날짜에 `410 Gone`으로 응답한다.

```typescript
// Sunset 이후 응답
app.use('/api/v1', (req, res) => {
  res.status(410).json({
    error: {
      code: 'API_VERSION_SUNSET',
      message: 'API v1은 2026-09-01에 종료되었습니다.',
      migrationGuide: 'https://docs.example.com/migration/v1-to-v2',
    },
  });
});
```

`404`가 아니라 `410 Gone`을 써야 한다. "없는 것"이 아니라 "있었는데 영구히 사라진 것"이기 때문이다.

---

## 정리

프론트엔드가 편한 API는 프론트엔드가 무엇을 해야 하는지를 먼저 이해하고 설계된다.

커서 기반 페이지네이션은 무한 스크롤을 안정적으로 지원하고, 대규모 데이터에서 성능을 보장한다.

에러 응답 표준화는 프론트가 에러 처리를 예측 가능하게 만든다. 에러 코드, 필드별 세부 정보, traceId가 핵심이다.

GraphQL은 프론트에 유연한 데이터 조회를 허용하지만, N+1 문제와 쿼리 복잡도 제한을 반드시 함께 설계해야 한다.

API 버저닝은 서버와 프론트의 독립 배포를 가능하게 한다. Breaking Change를 최소화하고, 불가피할 땐 충분한 유예 기간을 준다.

좋은 API는 "쓰는 쪽을 먼저 생각한" API다.

---

## 참고 자료

- [Relay Cursor Connections Specification](https://relay.dev/graphql/connections.htm)
- [GraphQL DataLoader](https://github.com/graphql/dataloader)
- [Stripe API Versioning](https://stripe.com/docs/api/versioning)
- [RFC 8594 - The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594)
- [Apollo Server - Subscriptions](https://www.apollographql.com/docs/apollo-server/data/subscriptions/)
- [graphql-depth-limit](https://github.com/stems/graphql-depth-limit)
- [Problem Details for HTTP APIs - RFC 7807](https://www.rfc-editor.org/rfc/rfc7807)
