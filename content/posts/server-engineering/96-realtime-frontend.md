---
title: "[Frontend 통합] 4편 — 실시간 통신 통합: WebSocket과 SSE를 프론트엔드와 연결하기"
date: 2026-03-17T12:04:00+09:00
draft: false
tags: ["WebSocket", "SSE", "Socket.IO", "실시간", "서버"]
series: ["Frontend 통합"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 14
summary: "WebSocket 인증 흐름(토큰 전달, 연결 시 검증), 재연결 전략과 오프라인 큐, SSE 실전 활용(알림, LLM 스트리밍), Socket.IO vs 네이티브 WebSocket과 수평 확장까지"
---

REST API는 요청-응답 모델이다. 클라이언트가 먼저 물어봐야 서버가 답한다.

실시간 알림, 채팅, 라이브 피드, LLM 스트리밍은 이 모델로는 구현할 수 없다. 서버가 먼저 말을 걸어야 하기 때문이다.

이 글에서는 WebSocket과 SSE(Server-Sent Events)를 프론트엔드와 실제로 연결하는 방법을 서버 엔지니어 관점에서 다룬다.

---

## 1. WebSocket 인증 흐름

### 왜 WebSocket 인증이 까다로운가

HTTP 요청은 매 요청마다 Authorization 헤더를 붙일 수 있다. WebSocket은 최초 핸드셰이크 이후 지속 연결이기 때문에 인증 시점이 명확히 정해져 있지 않다.

가장 흔한 실수는 핸드셰이크 시점에만 인증을 검사하고, 이후 메시지는 검증 없이 처리하는 것이다.

### 토큰 전달 방법

**방법 1: 쿼리 파라미터로 토큰 전달**

```javascript
// 클라이언트
const token = localStorage.getItem('access_token');
const ws = new WebSocket(`wss://api.example.com/ws?token=${token}`);
```

URL에 토큰이 노출된다. 로그, 프록시, 브라우저 히스토리에 기록될 수 있어 보안상 좋지 않다.

**방법 2: 핸드셰이크 헤더 사용 (서버-서버, 네이티브 앱)**

브라우저 WebSocket API는 커스텀 헤더를 설정할 수 없다. 이 방법은 Node.js 클라이언트나 모바일 앱에서만 유효하다.

**방법 3: 연결 직후 인증 메시지 전송 (권장)**

```javascript
// 클라이언트
const ws = new WebSocket('wss://api.example.com/ws');

ws.onopen = () => {
  ws.send(JSON.stringify({
    type: 'AUTH',
    token: localStorage.getItem('access_token')
  }));
};

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);
  if (msg.type === 'AUTH_SUCCESS') {
    // 이후 메시지 처리 시작
  } else if (msg.type === 'AUTH_FAILED') {
    ws.close(4001, 'Unauthorized');
  }
};
```

연결 직후 첫 메시지로 인증 토큰을 보내는 방식이다. 브라우저 환경에서 가장 현실적인 접근이다.

### 서버 측 인증 처리 (Spring)

```java
@Component
public class AuthWebSocketHandler extends TextWebSocketHandler {

    private final JwtTokenProvider jwtTokenProvider;
    // 인증 완료된 세션 관리
    private final Set<String> authenticatedSessions = ConcurrentHashMap.newKeySet();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        // 연결만 수락, 인증은 아직
        // 일정 시간 내 AUTH 메시지 없으면 강제 종료 (타임아웃 설정 필요)
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
        JsonNode payload = objectMapper.readTree(message.getPayload());
        String type = payload.get("type").asText();

        if ("AUTH".equals(type)) {
            String token = payload.get("token").asText();
            handleAuth(session, token);
            return;
        }

        // 인증되지 않은 세션의 메시지는 거부
        if (!authenticatedSessions.contains(session.getId())) {
            session.sendMessage(new TextMessage(
                "{\"type\":\"ERROR\",\"message\":\"Unauthorized\"}"
            ));
            return;
        }

        // 실제 메시지 처리
        handleBusinessMessage(session, type, payload);
    }

    private void handleAuth(WebSocketSession session, String token) throws IOException {
        try {
            Claims claims = jwtTokenProvider.validateToken(token);
            authenticatedSessions.add(session.getId());
            session.getAttributes().put("userId", claims.getSubject());
            session.sendMessage(new TextMessage("{\"type\":\"AUTH_SUCCESS\"}"));
        } catch (JwtException e) {
            session.sendMessage(new TextMessage("{\"type\":\"AUTH_FAILED\"}"));
            session.close(CloseStatus.NOT_ACCEPTABLE);
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        authenticatedSessions.remove(session.getId());
    }
}
```

핵심은 인증 전 메시지를 처리하지 않는 것이다. 인증 메시지가 없으면 연결을 끊는 타임아웃도 반드시 구현해야 한다.

### 인증 타임아웃 구현

```java
@Override
public void afterConnectionEstablished(WebSocketSession session) {
    // 10초 내 인증 없으면 강제 종료
    scheduler.schedule(() -> {
        if (!authenticatedSessions.contains(session.getId()) && session.isOpen()) {
            try {
                session.close(new CloseStatus(4002, "Auth timeout"));
            } catch (IOException e) {
                log.error("Failed to close unauthenticated session", e);
            }
        }
    }, 10, TimeUnit.SECONDS);
}
```

미인증 연결을 무한정 열어두면 리소스 고갈 공격에 취약해진다.

---

## 2. 재연결 전략과 오프라인 큐

### 왜 재연결 로직이 필요한가

WebSocket 연결은 끊길 수 있다. 네트워크 불안정, 서버 재시작, 프록시 타임아웃, 모바일 절전 모드 등 원인은 다양하다.

클라이언트가 재연결을 처리하지 않으면 사용자는 페이지를 새로고침해야 한다.

### Exponential Backoff 재연결

```javascript
class ReliableWebSocket {
  constructor(url, options = {}) {
    this.url = url;
    this.options = options;
    this.reconnectDelay = 1000;      // 초기 1초
    this.maxDelay = 30000;           // 최대 30초
    this.reconnectAttempts = 0;
    this.offlineQueue = [];
    this.isAuthenticated = false;
    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);

    this.ws.onopen = () => {
      this.reconnectDelay = 1000;
      this.reconnectAttempts = 0;
      this.authenticate();
    };

    this.ws.onclose = (event) => {
      this.isAuthenticated = false;
      // 정상 종료(1000)가 아닌 경우에만 재연결
      if (event.code !== 1000) {
        this.scheduleReconnect();
      }
    };

    this.ws.onerror = () => {
      // onclose가 항상 뒤따르므로 여기서는 로깅만
    };

    this.ws.onmessage = (event) => {
      this.handleMessage(JSON.parse(event.data));
    };
  }

  scheduleReconnect() {
    // Jitter 추가: 다수 클라이언트 동시 재연결 방지
    const jitter = Math.random() * 1000;
    const delay = Math.min(this.reconnectDelay + jitter, this.maxDelay);

    console.log(`Reconnecting in ${Math.round(delay)}ms (attempt ${++this.reconnectAttempts})`);

    setTimeout(() => this.connect(), delay);
    this.reconnectDelay = Math.min(this.reconnectDelay * 2, this.maxDelay);
  }

  authenticate() {
    const token = localStorage.getItem('access_token');
    this.ws.send(JSON.stringify({ type: 'AUTH', token }));
  }

  handleMessage(msg) {
    if (msg.type === 'AUTH_SUCCESS') {
      this.isAuthenticated = true;
      this.flushOfflineQueue();
    } else if (msg.type === 'AUTH_FAILED') {
      // 토큰 갱신 후 재시도
      this.refreshTokenAndReconnect();
    } else {
      // 비즈니스 메시지 처리
      this.options.onMessage?.(msg);
    }
  }

  send(data) {
    if (this.isAuthenticated && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(data));
    } else {
      // 연결이 없으면 큐에 저장
      this.offlineQueue.push(data);
    }
  }

  flushOfflineQueue() {
    while (this.offlineQueue.length > 0) {
      const msg = this.offlineQueue.shift();
      this.ws.send(JSON.stringify(msg));
    }
  }
}
```

Jitter는 중요하다. 서버가 재시작된 직후 수천 개의 클라이언트가 동시에 재연결을 시도하면 서버가 다시 과부하에 걸린다. Exponential Backoff만으로는 재시도 타이밍이 일치하는 "재연결 폭풍(thundering herd)"이 발생할 수 있고, 여기에 무작위 Jitter를 더해 클라이언트들의 재연결 시점을 분산시킨다.

### 메시지 순서 보장

오프라인 큐는 메시지를 보관하지만, 순서 보장은 별도 메커니즘이 필요하다.

```javascript
class OrderedMessageQueue {
  constructor() {
    this.queue = [];
    this.sequenceNumber = 0;
  }

  enqueue(data) {
    this.queue.push({
      seq: ++this.sequenceNumber,
      timestamp: Date.now(),
      data
    });
  }

  // 오래된 메시지 제거 (예: 5분 이상)
  pruneStale() {
    const cutoff = Date.now() - 5 * 60 * 1000;
    this.queue = this.queue.filter(msg => msg.timestamp > cutoff);
  }

  flush(sendFn) {
    // seq 순으로 정렬 후 전송
    this.queue.sort((a, b) => a.seq - b.seq);
    while (this.queue.length > 0) {
      const msg = this.queue.shift();
      sendFn(msg);
    }
  }
}
```

서버에서도 시퀀스 번호를 기반으로 중복 처리나 순서 역전을 감지해야 한다.

### 서버 측 Last-Event-ID 패턴

```java
// 클라이언트가 재연결 시 마지막으로 받은 seq를 전송
// { type: "RESUME", lastSeq: 1234 }

@Override
protected void handleTextMessage(WebSocketSession session, TextMessage message) {
    JsonNode payload = objectMapper.readTree(message.getPayload());

    if ("RESUME".equals(payload.get("type").asText())) {
        long lastSeq = payload.get("lastSeq").asLong();
        // lastSeq 이후 놓친 메시지 재전송
        replayMissedMessages(session, lastSeq);
    }
}

private void replayMissedMessages(WebSocketSession session, long lastSeq) {
    List<StoredMessage> missed = messageStore.findAfter(
        session.getAttributes().get("userId").toString(),
        lastSeq
    );
    missed.forEach(msg -> {
        try {
            session.sendMessage(new TextMessage(objectMapper.writeValueAsString(msg)));
        } catch (IOException e) {
            log.error("Failed to replay message", e);
        }
    });
}
```

메시지를 일정 기간 저장하고, 재연결 시 누락분을 재전송하는 패턴이다. Kafka나 Redis Stream을 메시지 스토어로 활용하면 확장에 유리하다.

---

## 3. SSE 실전 활용

### SSE vs WebSocket 선택 기준

SSE는 서버에서 클라이언트로의 단방향 스트림이다. HTTP/1.1 위에서 동작하고, 자동 재연결이 내장되어 있다.

클라이언트가 서버로 데이터를 보낼 필요가 없다면 SSE가 더 단순하다. 알림, 피드 업데이트, 진행률 표시, LLM 스트리밍이 대표적인 사례다.

### Spring에서 SSE 구현

```java
@RestController
@RequestMapping("/api/notifications")
public class NotificationController {

    private final SseEmitterRegistry emitterRegistry;

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter stream(
        @RequestHeader(value = "Authorization") String authHeader,
        @RequestParam(required = false) String lastEventId
    ) {
        String userId = jwtTokenProvider.extractUserId(authHeader.replace("Bearer ", ""));

        // 타임아웃: 30분 (0은 타임아웃 없음, 권장하지 않음)
        SseEmitter emitter = new SseEmitter(30 * 60 * 1000L);

        emitterRegistry.register(userId, emitter);

        emitter.onCompletion(() -> emitterRegistry.remove(userId));
        emitter.onTimeout(() -> emitterRegistry.remove(userId));
        emitter.onError(e -> emitterRegistry.remove(userId));

        // 연결 즉시 초기 이벤트 전송 (연결 확인 용도)
        try {
            emitter.send(SseEmitter.event()
                .name("connected")
                .data("{\"status\":\"ok\"}")
            );

            // lastEventId가 있으면 놓친 이벤트 재전송
            if (lastEventId != null) {
                replayMissedEvents(emitter, userId, lastEventId);
            }
        } catch (IOException e) {
            emitter.completeWithError(e);
        }

        return emitter;
    }
}
```

### SseEmitter 레지스트리

```java
@Component
public class SseEmitterRegistry {

    private final Map<String, List<SseEmitter>> userEmitters = new ConcurrentHashMap<>();

    public void register(String userId, SseEmitter emitter) {
        userEmitters.computeIfAbsent(userId, k -> new CopyOnWriteArrayList<>()).add(emitter);
    }

    public void remove(String userId, SseEmitter emitter) {
        List<SseEmitter> emitters = userEmitters.get(userId);
        if (emitters != null) {
            emitters.remove(emitter);
        }
    }

    public void send(String userId, String eventName, Object data) {
        List<SseEmitter> emitters = userEmitters.getOrDefault(userId, List.of());
        List<SseEmitter> deadEmitters = new ArrayList<>();

        for (SseEmitter emitter : emitters) {
            try {
                emitter.send(SseEmitter.event()
                    .id(generateEventId())
                    .name(eventName)
                    .data(objectMapper.writeValueAsString(data))
                );
            } catch (IOException e) {
                deadEmitters.add(emitter);
            }
        }

        deadEmitters.forEach(emitters::remove);
    }
}
```

한 사용자가 여러 탭을 열 수 있으므로 userId당 여러 Emitter를 관리해야 한다.

### 알림 발송

```java
@Service
public class NotificationService {

    private final SseEmitterRegistry registry;

    public void sendNotification(String userId, NotificationDto notification) {
        // SSE로 실시간 발송 시도
        registry.send(userId, "notification", notification);

        // 오프라인 사용자를 위해 DB에도 저장
        notificationRepository.save(toEntity(notification, userId));
    }
}
```

SSE 연결이 없는 오프라인 사용자를 위해 DB 저장을 병행해야 한다. 클라이언트가 온라인이 될 때 미읽음 알림을 REST API로 조회하게 한다.

### LLM 스트리밍 응답

ChatGPT, Claude 등 LLM API는 응답을 스트리밍으로 반환한다. 이를 프론트엔드에 그대로 중계하는 것이 SSE의 대표 사례다.

```java
@PostMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamChat(
    @RequestBody ChatRequest request,
    @RequestHeader("Authorization") String authHeader
) {
    SseEmitter emitter = new SseEmitter(5 * 60 * 1000L); // 5분 타임아웃

    // LLM 호출은 비동기로
    CompletableFuture.runAsync(() -> {
        try {
            llmClient.streamCompletion(request.getPrompt(), chunk -> {
                try {
                    emitter.send(SseEmitter.event()
                        .name("chunk")
                        .data(objectMapper.writeValueAsString(Map.of("text", chunk)))
                    );
                } catch (IOException e) {
                    throw new RuntimeException("Client disconnected", e);
                }
            });

            // 스트림 완료 신호
            emitter.send(SseEmitter.event().name("done").data("{}"));
            emitter.complete();

        } catch (Exception e) {
            try {
                emitter.send(SseEmitter.event()
                    .name("error")
                    .data("{\"message\":\"" + e.getMessage() + "\"}")
                );
            } catch (IOException ignored) {}
            emitter.completeWithError(e);
        }
    }, executor);

    return emitter;
}
```

### 클라이언트 SSE 연결

```javascript
class SseClient {
  constructor(url, token) {
    this.url = url;
    this.token = token;
    this.eventSource = null;
    this.lastEventId = null;
    this.connect();
  }

  connect() {
    // EventSource는 Authorization 헤더를 지원하지 않음
    // 토큰을 쿼리 파라미터로 전달하거나, 쿠키 인증을 사용해야 함
    const url = `${this.url}?token=${this.token}` +
      (this.lastEventId ? `&lastEventId=${this.lastEventId}` : '');

    this.eventSource = new EventSource(url);

    this.eventSource.addEventListener('notification', (event) => {
      this.lastEventId = event.lastEventId;
      const data = JSON.parse(event.data);
      this.onNotification(data);
    });

    this.eventSource.addEventListener('chunk', (event) => {
      const data = JSON.parse(event.data);
      this.onChunk(data.text);
    });

    this.eventSource.addEventListener('done', () => {
      this.onComplete();
    });

    this.eventSource.onerror = () => {
      // EventSource는 자동 재연결 내장
      // lastEventId를 서버에 전달해 놓친 이벤트를 받을 수 있음
    };
  }

  disconnect() {
    this.eventSource?.close();
  }
}
```

`EventSource`는 재연결 시 `Last-Event-ID` 헤더를 자동으로 포함한다. 서버에서 이 헤더를 읽어 누락된 이벤트를 재전송할 수 있다.

브라우저의 `EventSource` API는 커스텀 헤더를 설정하는 기능을 제공하지 않는다. 이 때문에 토큰을 URL에 노출하거나 쿠키를 사용하는 방식 중 하나를 선택해야 한다. SSE의 인증 한계를 해결하는 더 깔끔한 방법은 쿠키 기반 세션 인증을 사용하는 것이다. HttpOnly 쿠키는 EventSource 요청에 자동으로 포함된다.

---

## 4. Socket.IO vs 네이티브 WebSocket

### Socket.IO가 제공하는 것

Socket.IO는 WebSocket 위에 올라가는 추상화 레이어다. 다음 기능을 기본 제공한다.

- 자동 재연결 (Exponential Backoff 내장)
- 룸(Room) 기반 메시지 브로드캐스트
- 이벤트 기반 메시지 라우팅
- WebSocket 미지원 환경에서 Long Polling 폴백
- 서버 간 이벤트 동기화 (Redis Adapter)

### Socket.IO 서버 (Node.js)

```javascript
const { Server } = require('socket.io');
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const io = new Server(httpServer, {
  cors: { origin: process.env.FRONTEND_URL }
});

// Redis Adapter: 여러 서버 인스턴스 간 이벤트 공유
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));

// 미들웨어로 인증 처리
io.use(async (socket, next) => {
  const token = socket.handshake.auth.token;
  try {
    const payload = await verifyJwt(token);
    socket.userId = payload.sub;
    next();
  } catch (err) {
    next(new Error('Unauthorized'));
  }
});

io.on('connection', (socket) => {
  console.log(`User ${socket.userId} connected`);

  // 사용자별 룸 입장
  socket.join(`user:${socket.userId}`);

  socket.on('join-channel', (channelId) => {
    // 권한 확인 후 룸 입장
    socket.join(`channel:${channelId}`);
  });

  socket.on('send-message', async (data) => {
    const message = await saveMessage(data);

    // 채널의 모든 연결된 클라이언트에게 브로드캐스트
    io.to(`channel:${data.channelId}`).emit('new-message', message);
  });

  socket.on('disconnect', () => {
    console.log(`User ${socket.userId} disconnected`);
  });
});

// 서버 내부에서 특정 사용자에게 알림 전송
function notifyUser(userId, event, data) {
  io.to(`user:${userId}`).emit(event, data);
}
```

### Socket.IO 클라이언트

```javascript
import { io } from 'socket.io-client';

const socket = io('wss://api.example.com', {
  auth: {
    token: localStorage.getItem('access_token')
  },
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 30000,
  reconnectionAttempts: Infinity
});

socket.on('connect', () => {
  console.log('Connected');
  socket.emit('join-channel', 'general');
});

socket.on('new-message', (message) => {
  renderMessage(message);
});

socket.on('connect_error', (err) => {
  if (err.message === 'Unauthorized') {
    // 토큰 갱신 후 재연결
    refreshTokenAndReconnect();
  }
});
```

### 네이티브 WebSocket이 더 나은 경우

Socket.IO는 클라이언트 라이브러리 의존성이 생긴다. 서버도 Socket.IO 서버여야 한다.

다음 상황에서는 네이티브 WebSocket이 더 적합하다.

- 클라이언트가 다양함 (브라우저, 모바일 앱, IoT 기기)
- 서버가 Java, Go, Rust 등 비Node.js 환경
- 프로토콜 레벨 제어가 필요한 경우 (바이너리 프레임, 커스텀 서브프로토콜)
- 최소한의 오버헤드가 필요한 고성능 환경

### 수평 확장 시 세션 공유 문제

WebSocket은 지속 연결이다. 로드 밸런서가 요청을 라운드 로빈으로 분산하면 같은 사용자의 연결이 매번 다른 서버로 갈 수 있다.

**문제 시나리오:**

```
클라이언트 → LB → 서버 A (연결 수립)
서버 B가 이 사용자에게 메시지 전송 시도 → 연결 없음
```

**해결책 1: Sticky Session (IP Hash)**

```nginx
upstream websocket {
  ip_hash;  # 같은 IP는 항상 같은 서버로
  server ws1.example.com;
  server ws2.example.com;
}
```

단순하지만 서버 장애 시 해당 서버의 모든 연결이 끊긴다.

**해결책 2: Redis Pub/Sub (권장)**

```java
// 서버 A에서: 사용자 userId의 연결을 관리
// 서버 B에서: 해당 사용자에게 메시지를 보내야 할 때

@Service
public class CrossServerMessageService {

    private final RedisTemplate<String, String> redisTemplate;
    private final LocalSessionRegistry localRegistry;

    // 메시지 발신: 어느 서버에 연결되어 있든 전달
    public void sendToUser(String userId, Object message) {
        String channel = "ws:user:" + userId;
        redisTemplate.convertAndSend(channel, objectMapper.writeValueAsString(message));
    }

    // 구독: 내 서버에 연결된 사용자 메시지만 처리
    @PostConstruct
    public void subscribeToUserMessages() {
        redisTemplate.execute((RedisCallback<Void>) connection -> {
            connection.subscribe((message, pattern) -> {
                String channel = new String(message.getChannel());
                String userId = channel.replace("ws:user:", "");

                // 이 서버에 해당 사용자 연결이 있을 때만 전달
                WebSocketSession session = localRegistry.getSession(userId);
                if (session != null && session.isOpen()) {
                    session.sendMessage(new TextMessage(new String(message.getBody())));
                }
            }, "ws:user:*".getBytes());
            return null;
        });
    }
}
```

모든 서버가 Redis Pub/Sub을 구독한다. 메시지를 보낼 때 Redis에 발행하면, 해당 사용자 연결을 가진 서버만 실제로 WebSocket으로 전달한다.

**해결책 3: Kafka를 이용한 확장**

```java
// 메시지 발행
@Service
public class WebSocketKafkaProducer {
    public void sendToUser(String userId, Object message) {
        kafkaTemplate.send("websocket-messages",
            userId,  // 파티션 키: 같은 사용자는 같은 파티션
            objectMapper.writeValueAsString(message)
        );
    }
}

// 서버별 Kafka Consumer
@KafkaListener(topics = "websocket-messages", groupId = "${server.instance-id}")
public void consumeMessage(ConsumerRecord<String, String> record) {
    String userId = record.key();
    WebSocketSession session = localRegistry.getSession(userId);
    if (session != null) {
        session.sendMessage(new TextMessage(record.value()));
    }
}
```

Kafka는 메시지 영속성과 재처리가 필요한 경우에 적합하다. 단순한 실시간 알림에는 Redis Pub/Sub이 더 가볍다.

---

## 정리

WebSocket과 SSE는 각각 적합한 사용 사례가 있다.

| 구분 | WebSocket | SSE |
|------|-----------|-----|
| 통신 방향 | 양방향 | 서버 → 클라이언트 |
| 재연결 | 수동 구현 필요 | 브라우저 자동 처리 |
| 인증 | 연결 후 메시지로 처리 | 쿠키 또는 쿼리 파라미터 |
| 주요 사례 | 채팅, 게임, 협업 도구 | 알림, 피드, LLM 스트리밍 |

수평 확장은 Redis Pub/Sub이 현실적인 선택이다. Sticky Session은 단순하지만 서버 장애 내성이 낮다.

오프라인 큐와 재연결 전략은 선택이 아닌 필수다. 네트워크는 항상 불안정하다고 가정하고 설계해야 한다.

---

## 참고 자료

- [MDN: WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [Socket.IO Documentation](https://socket.io/docs/v4/)
- [Socket.IO Redis Adapter](https://socket.io/docs/v4/redis-adapter/)
- [Spring WebSocket Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#websocket)
- [RFC 6455: The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- [HTML Living Standard: Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
