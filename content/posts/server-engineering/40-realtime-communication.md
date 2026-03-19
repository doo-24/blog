---
title: "[비동기 처리와 이벤트 드리븐] 7편 — 실시간 통신: 서버에서 클라이언트로 데이터 밀기"
date: 2026-03-17T21:01:00+09:00
draft: false
tags: ["WebSocket", "SSE", "실시간", "Long Polling", "Redis Pub/Sub", "서버"]
series: ["비동기 처리와 이벤트 드리븐"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 5
summary: "Polling vs Long Polling vs SSE vs WebSocket 비교, WebSocket 프로토콜과 연결 관리·Heartbeat, SSE 구현과 적합한 유스케이스, 실시간 기능 수평 확장(Redis Pub/Sub, Sticky Session)까지"
---

서버가 클라이언트에게 먼저 말을 거는 것은, HTTP의 관점에서 보면 꽤 부자연스러운 일이다.

HTTP는 클라이언트가 요청하고 서버가 응답하는 단방향 계약 위에서 태어났다.

그런데 현실의 서비스는 다르다. 주식 시세는 매 초 바뀌고, 채팅 메시지는 상대방이 보낼 때 즉시 화면에 나타나야 하며, 대용량 파일 업로드의 진행률은 서버가 알고 있다.

클라이언트가 "아직도 안 됐어?"를 반복하는 것보다, 서버가 "됐어"라고 알려주는 것이 훨씬 효율적이다.

실시간 통신 기술은 바로 이 간극을 메우기 위해 진화해 왔다. Polling처럼 무식하게 묻는 방법부터, WebSocket처럼 완전한 양방향 채널을 여는 방법까지, 각 기술에는 적합한 유스케이스와 피해야 할 함정이 존재한다.

---

## 실시간 통신 기술의 스펙트럼

서버에서 클라이언트로 데이터를 전달하는 방법은 크게 네 가지로 분류된다.

이 기술들은 복잡도와 실시간성 사이의 트레이드오프를 각기 다른 방식으로 해결한다.

### Polling: 가장 단순한 접근

Polling은 클라이언트가 일정 간격으로 서버에 "새로운 데이터가 있나요?"라고 묻는 방식이다.

```javascript
// 클라이언트 측 Polling 구현
function startPolling(interval = 3000) {
    setInterval(async () => {
        const response = await fetch('/api/notifications');
        const data = await response.json();
        if (data.items.length > 0) {
            renderNotifications(data.items);
        }
    }, interval);
}
```

장점은 구현이 단순하고, 기존 REST 인프라를 그대로 사용할 수 있다는 것이다.

단점은 명확하다. 데이터가 없어도 요청은 발생하고, 간격이 짧을수록 서버 부하가 선형으로 증가한다.

1000명의 사용자가 3초마다 폴링하면 초당 333번의 요청이 발생한다.

**안티패턴**: 폴링 간격을 너무 짧게 설정해 실시간처럼 보이게 하는 것. 500ms 간격의 폴링은 WebSocket 없이 실시간 채팅을 구현하려는 시도처럼 보이지만, 실제로는 서버를 DDoS하는 것과 다르지 않다.

### Long Polling: 영리한 대기

Long Polling은 서버가 응답을 즉시 반환하지 않고, 새 데이터가 생기거나 타임아웃이 될 때까지 연결을 유지하는 방식이다.

```java
// Spring MVC의 DeferredResult를 활용한 Long Polling
@RestController
@RequestMapping("/api/longpoll")
public class LongPollController {

    private final Map<String, DeferredResult<ResponseEntity<List<Message>>>> pendingRequests
        = new ConcurrentHashMap<>();

    @GetMapping("/messages")
    public DeferredResult<ResponseEntity<List<Message>>> pollMessages(
            @RequestParam String userId,
            @RequestParam(defaultValue = "30000") long timeout) {

        DeferredResult<ResponseEntity<List<Message>>> result =
            new DeferredResult<>(timeout,
                ResponseEntity.ok(Collections.emptyList())); // 타임아웃 시 빈 응답

        pendingRequests.put(userId, result);

        result.onCompletion(() -> pendingRequests.remove(userId));
        result.onTimeout(() -> pendingRequests.remove(userId));

        return result;
    }

    // 메시지 도착 시 대기 중인 클라이언트에게 응답
    public void notifyUser(String userId, List<Message> messages) {
        DeferredResult<ResponseEntity<List<Message>>> pending =
            pendingRequests.get(userId);
        if (pending != null && !pending.isSetOrExpired()) {
            pending.setResult(ResponseEntity.ok(messages));
        }
    }
}
```

클라이언트는 응답을 받는 즉시 다음 요청을 보낸다.

데이터가 있을 때만 응답하므로 폴링 대비 네트워크 트래픽은 크게 줄지만, 연결 자체는 계속 유지된다.

서버의 스레드가 연결마다 블록되는 구조에서는 이 방식도 확장성에 한계가 있다.

### SSE: 서버에서 클라이언트로의 단방향 스트림

Server-Sent Events는 HTTP 연결을 열린 상태로 유지하면서 서버가 텍스트 이벤트를 지속적으로 전송하는 표준 기술이다.

```
// SSE 응답 형식 (text/event-stream)
data: {"type":"price","symbol":"AAPL","value":175.23}\n\n

event: alert\n
data: {"message":"시스템 점검 예정"}\n\n

id: 42\n
retry: 3000\n
data: {"type":"heartbeat"}\n\n
```

SSE는 HTTP 위에서 동작하므로 기존 인프라(로드밸런서, 프록시)와 잘 호환되고, 브라우저가 자동 재연결을 처리해 준다.

단방향이라는 제약이 있지만, 서버→클라이언트 데이터 푸시 시나리오의 대부분은 이걸로 충분하다.

### WebSocket: 완전한 양방향 채널

WebSocket은 HTTP 핸드셰이크 이후 TCP 레이어에서 독립적인 전이중(full-duplex) 통신 채널을 확립한다.

클라이언트와 서버 모두 언제든지 메시지를 보낼 수 있다.

### 기술 비교 요약

| 특성 | Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| 방향성 | 단방향(요청) | 단방향(요청) | 단방향(서버→클라이언트) | 양방향 |
| 프로토콜 | HTTP | HTTP | HTTP | WS (TCP) |
| 지연시간 | 높음 | 낮음 | 낮음 | 매우 낮음 |
| 서버 부하 | 높음 | 중간 | 낮음 | 낮음 |
| 구현 복잡도 | 낮음 | 중간 | 낮음 | 높음 |
| 재연결 | 클라이언트 | 클라이언트 | 자동(브라우저) | 수동 |
| HTTP/2 호환 | 완전 | 완전 | 완전 | 별도 |
| 적합한 용도 | 간헐적 갱신 | 낮은 빈도 알림 | 대시보드, 피드 | 채팅, 게임 |

---

## WebSocket 프로토콜과 연결 관리

### 핸드셰이크 메커니즘

WebSocket 연결은 HTTP Upgrade 요청으로 시작한다.

```
// 클라이언트 → 서버
GET /ws/chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

// 서버 → 클라이언트
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

서버는 `Sec-WebSocket-Key`에 GUID를 붙여 SHA-1 해시 후 Base64로 인코딩해 `Sec-WebSocket-Accept`를 생성한다.

이 핸드셰이크가 완료된 이후부터는 HTTP가 아닌 WebSocket 프레임 형식으로 통신한다.

### Spring WebSocket 서버 구현

```java
// WebSocket 설정
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Autowired
    private ChatWebSocketHandler chatHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatHandler, "/ws/chat")
                .setAllowedOrigins("*")
                .withSockJS(); // SockJS 폴백 지원
    }
}

// WebSocket 핸들러
@Component
public class ChatWebSocketHandler extends TextWebSocketHandler {

    // 연결된 세션을 roomId별로 관리
    private final Map<String, Set<WebSocketSession>> roomSessions =
        new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String roomId = extractRoomId(session);
        roomSessions.computeIfAbsent(roomId, k -> ConcurrentHashMap.newKeySet())
                    .add(session);

        log.info("WebSocket 연결: sessionId={}, roomId={}, 현재 인원={}",
            session.getId(), roomId,
            roomSessions.get(roomId).size());
    }

    @Override
    protected void handleTextMessage(WebSocketSession session,
                                     TextMessage message) throws Exception {
        ChatMessage chatMsg = objectMapper.readValue(
            message.getPayload(), ChatMessage.class);

        String roomId = extractRoomId(session);
        broadcast(roomId, chatMsg, session);
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session,
                                      CloseStatus status) {
        String roomId = extractRoomId(session);
        Set<WebSocketSession> sessions = roomSessions.get(roomId);
        if (sessions != null) {
            sessions.remove(session);
            if (sessions.isEmpty()) {
                roomSessions.remove(roomId);
            }
        }
        log.info("WebSocket 종료: sessionId={}, status={}", session.getId(), status);
    }

    private void broadcast(String roomId, ChatMessage message,
                           WebSocketSession sender) throws IOException {
        Set<WebSocketSession> sessions = roomSessions.getOrDefault(
            roomId, Collections.emptySet());

        String payload = objectMapper.writeValueAsString(message);
        TextMessage textMessage = new TextMessage(payload);

        for (WebSocketSession session : sessions) {
            if (session.isOpen()) {
                // 동시 전송 방지를 위한 동기화
                synchronized (session) {
                    session.sendMessage(textMessage);
                }
            }
        }
    }

    private String extractRoomId(WebSocketSession session) {
        URI uri = session.getUri();
        // /ws/chat?roomId=xxx 에서 roomId 추출
        return UriComponentsBuilder.fromUri(uri)
            .build().getQueryParams().getFirst("roomId");
    }
}
```

### Heartbeat와 연결 유지

TCP 연결은 중간 네트워크 장비(NAT, 로드밸런서, 방화벽)에 의해 조용히 끊길 수 있다.

특히 AWS ALB의 기본 idle timeout은 60초다.

Heartbeat는 이 문제를 해결한다.

```java
// WebSocket Heartbeat 구현
@Component
public class WebSocketHeartbeatManager {

    private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor();

    @PostConstruct
    public void startHeartbeat() {
        scheduler.scheduleAtFixedRate(
            this::sendPingToAllSessions,
            30, 30, TimeUnit.SECONDS
        );
    }

    private void sendPingToAllSessions() {
        sessionRegistry.getAllSessions().forEach(session -> {
            if (session.isOpen()) {
                try {
                    // WebSocket 표준 Ping 프레임 전송
                    session.sendMessage(new PingMessage());
                } catch (IOException e) {
                    log.warn("Ping 실패, 세션 제거: {}", session.getId());
                    closeQuietly(session);
                }
            }
        });
    }
}

// 클라이언트 측 Pong 처리는 브라우저가 자동으로 응답
// 애플리케이션 레벨 Heartbeat도 병행하는 것이 권장
@Component
public class AppLevelHeartbeatHandler extends TextWebSocketHandler {

    @Override
    protected void handleTextMessage(WebSocketSession session,
                                     TextMessage message) throws Exception {
        JsonNode node = objectMapper.readTree(message.getPayload());

        if ("ping".equals(node.get("type").asText())) {
            // 애플리케이션 레벨 Pong 응답
            session.sendMessage(new TextMessage(
                "{\"type\":\"pong\",\"timestamp\":" +
                System.currentTimeMillis() + "}"
            ));
            return;
        }

        // 일반 메시지 처리
        handleChatMessage(session, node);
    }
}
```

**연결 종료 코드 처리**: WebSocket 종료 시 Close Frame에 포함된 상태 코드를 구분해야 한다.

```java
@Override
public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
    switch (status.getCode()) {
        case 1000: // 정상 종료
            handleGracefulDisconnect(session);
            break;
        case 1001: // 브라우저 탭 닫힘
            handleBrowserClose(session);
            break;
        case 1006: // 비정상 종료 (네트워크 오류 등)
            handleAbnormalDisconnect(session);
            scheduleReconnectNotification(session); // 재연결 안내
            break;
        case 1008: // 정책 위반 (인증 만료 등)
            handlePolicyViolation(session);
            break;
    }
}
```

### STOMP over WebSocket

대규모 채팅이나 구독 기반 메시징은 STOMP 프로토콜을 WebSocket 위에 얹는 것이 표준 관행이다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class StompWebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // 메모리 내 브로커 사용 (단일 인스턴스)
        config.enableSimpleBroker("/topic", "/queue");
        // 클라이언트 → 서버 메시지 prefix
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}

@Controller
public class StompChatController {

    @MessageMapping("/chat.sendMessage")
    @SendTo("/topic/room.{roomId}")
    public ChatMessage sendMessage(@DestinationVariable String roomId,
                                   @Payload ChatMessage message,
                                   Principal principal) {
        message.setSender(principal.getName());
        message.setTimestamp(Instant.now());
        return message;
    }

    // 특정 사용자에게만 전송
    @MessageMapping("/chat.privateMessage")
    public void privateMessage(@Payload PrivateMessage message,
                                SimpMessageHeaderAccessor headerAccessor) {
        messagingTemplate.convertAndSendToUser(
            message.getRecipient(),
            "/queue/private",
            message
        );
    }
}
```

---

## SSE 구현과 적합한 유스케이스

### Spring에서 SSE 구현

```java
// SseEmitter를 활용한 SSE 구현
@RestController
@RequestMapping("/api/sse")
public class SseController {

    // 사용자 ID → SseEmitter 매핑
    private final Map<String, SseEmitter> emitters = new ConcurrentHashMap<>();

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamEvents(@RequestParam String userId,
                                   HttpServletResponse response) {
        // nginx 등 프록시의 버퍼링 비활성화
        response.setHeader("X-Accel-Buffering", "no");
        response.setHeader("Cache-Control", "no-cache");

        // 30분 타임아웃으로 SseEmitter 생성
        SseEmitter emitter = new SseEmitter(30 * 60 * 1000L);

        emitters.put(userId, emitter);

        // 연결 즉시 확인 이벤트 전송 (일부 프록시는 첫 데이터 수신 전까지 버퍼링)
        try {
            emitter.send(SseEmitter.event()
                .name("connected")
                .data("{\"status\":\"connected\",\"userId\":\"" + userId + "\"}")
            );
        } catch (IOException e) {
            emitter.completeWithError(e);
            return emitter;
        }

        emitter.onCompletion(() -> emitters.remove(userId));
        emitter.onTimeout(() -> {
            emitters.remove(userId);
            emitter.complete();
        });
        emitter.onError(ex -> emitters.remove(userId));

        return emitter;
    }

    // 특정 사용자에게 이벤트 전송
    public void sendToUser(String userId, String eventType, Object data) {
        SseEmitter emitter = emitters.get(userId);
        if (emitter == null) return;

        try {
            emitter.send(SseEmitter.event()
                .id(String.valueOf(System.currentTimeMillis()))
                .name(eventType)
                .data(objectMapper.writeValueAsString(data))
                .reconnectTime(3000) // 클라이언트 재연결 대기 시간 (ms)
            );
        } catch (IOException e) {
            log.warn("SSE 전송 실패: userId={}", userId);
            emitters.remove(userId);
            emitter.completeWithError(e);
        }
    }

    // 전체 브로드캐스트
    public void broadcast(String eventType, Object data) {
        String payload;
        try {
            payload = objectMapper.writeValueAsString(data);
        } catch (JsonProcessingException e) {
            log.error("직렬화 실패", e);
            return;
        }

        List<String> deadEmitters = new ArrayList<>();
        emitters.forEach((userId, emitter) -> {
            try {
                emitter.send(SseEmitter.event()
                    .name(eventType)
                    .data(payload)
                );
            } catch (IOException e) {
                deadEmitters.add(userId);
            }
        });

        deadEmitters.forEach(emitters::remove);
    }
}
```

### 클라이언트 측 SSE 처리

```javascript
// EventSource API 활용
class SseClient {
    constructor(userId) {
        this.userId = userId;
        this.eventSource = null;
        this.reconnectDelay = 1000;
        this.maxReconnectDelay = 30000;
    }

    connect() {
        const url = `/api/sse/stream?userId=${this.userId}`;
        this.eventSource = new EventSource(url);

        this.eventSource.addEventListener('connected', (e) => {
            console.log('SSE 연결 확립:', JSON.parse(e.data));
            this.reconnectDelay = 1000; // 재연결 성공 시 딜레이 초기화
        });

        this.eventSource.addEventListener('notification', (e) => {
            const notification = JSON.parse(e.data);
            this.handleNotification(notification);
        });

        this.eventSource.addEventListener('price-update', (e) => {
            const priceData = JSON.parse(e.data);
            this.updatePriceDisplay(priceData);
        });

        this.eventSource.onerror = (error) => {
            console.error('SSE 오류:', error);
            // EventSource는 자동 재연결 시도
            // readyState === 2 (CLOSED)이면 수동으로 재연결
            if (this.eventSource.readyState === EventSource.CLOSED) {
                this.scheduleReconnect();
            }
        };
    }

    scheduleReconnect() {
        setTimeout(() => {
            this.connect();
            this.reconnectDelay = Math.min(
                this.reconnectDelay * 2,
                this.maxReconnectDelay
            );
        }, this.reconnectDelay);
    }

    disconnect() {
        if (this.eventSource) {
            this.eventSource.close();
        }
    }
}
```

### Last-Event-ID를 활용한 재연결 후 누락 이벤트 복구

SSE의 강점 중 하나는 브라우저가 마지막으로 받은 이벤트 ID를 자동으로 `Last-Event-ID` 헤더에 담아 재연결 요청을 보낸다는 것이다.

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter streamWithReplay(
        @RequestParam String userId,
        @RequestHeader(value = "Last-Event-ID", required = false) String lastEventId) {

    SseEmitter emitter = new SseEmitter(30 * 60 * 1000L);
    emitters.put(userId, emitter);

    // 비동기로 누락 이벤트 재전송
    CompletableFuture.runAsync(() -> {
        if (lastEventId != null) {
            // 마지막 수신 ID 이후의 이벤트를 Redis나 DB에서 조회하여 재전송
            List<StoredEvent> missedEvents =
                eventStore.findEventsAfter(userId, lastEventId);

            for (StoredEvent event : missedEvents) {
                try {
                    emitter.send(SseEmitter.event()
                        .id(event.getId())
                        .name(event.getType())
                        .data(event.getPayload())
                    );
                } catch (IOException e) {
                    emitter.completeWithError(e);
                    return;
                }
            }
        }
    });

    return emitter;
}
```

### SSE 적합한 유스케이스

SSE는 다음과 같은 시나리오에 특히 적합하다.

- **실시간 대시보드**: 서버 메트릭, 주식 시세, IoT 센서 데이터
- **진행률 스트리밍**: 파일 업로드/변환, 배치 작업 진행 상황
- **알림 피드**: SNS 알림, 이메일 알림, 시스템 경보
- **라이브 로그 뷰어**: 배포 로그, 빌드 출력
- **AI 응답 스트리밍**: ChatGPT 스타일의 토큰 단위 출력

**SSE 안티패턴**: 클라이언트가 서버로도 데이터를 자주 보내야 하는 채팅 애플리케이션에 SSE를 쓰는 것. 이 경우 SSE(서버→클라이언트)와 REST POST(클라이언트→서버)를 혼용하게 되는데, 이는 WebSocket 하나로 해결할 수 있는 복잡성을 두 배로 늘리는 선택이다.

---

## 실시간 기능의 수평 확장

단일 서버에서 WebSocket이나 SSE는 쉽게 동작한다.

문제는 서버를 여러 대로 늘릴 때 시작된다.

A 서버에 연결된 사용자가 B 서버에 연결된 사용자에게 메시지를 보내려면, 두 서버 간의 메시지 라우팅이 필요하다.

### Sticky Session: 간단하지만 위험한 해법

Sticky Session(세션 어피니티)은 로드밸런서가 특정 클라이언트의 요청을 항상 같은 서버로 라우팅하는 방식이다.

```nginx
# nginx Sticky Session 설정 (ip_hash 방식)
upstream websocket_backend {
    ip_hash; # 클라이언트 IP 기반 라우팅

    server app-server-1:8080;
    server app-server-2:8080;
    server app-server-3:8080;
}

server {
    listen 80;

    location /ws {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s; # WebSocket idle timeout
        proxy_send_timeout 3600s;
    }
}
```

Sticky Session의 문제점은 하나의 서버가 다운되면 해당 서버에 연결된 모든 클라이언트가 동시에 연결을 잃는다는 것이다.

또한 서버 간 부하 불균형이 발생하기 쉽다.

### Redis Pub/Sub: 권장되는 확장 패턴

Redis Pub/Sub는 모든 서버가 동일한 채널을 구독하고, 메시지를 Redis를 통해 교환하는 방식이다.

```
[클라이언트A] <-> [서버1] --publish--> [Redis]
                                          |
[클라이언트B] <-> [서버2] <--subscribe-- [Redis]
```

```java
// Redis Pub/Sub 기반 WebSocket 브로드캐스트
@Configuration
public class RedisMessagingConfig {

    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            WebSocketBroadcastService broadcastService) {

        RedisMessageListenerContainer container =
            new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);

        // 채팅 채널 구독
        container.addMessageListener(
            (message, pattern) -> broadcastService.onRedisMessage(message),
            new PatternTopic("chat:room:*")
        );

        return container;
    }
}

@Service
@RequiredArgsConstructor
public class WebSocketBroadcastService {

    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;

    // 로컬 세션 레지스트리 (이 서버에 연결된 세션만)
    private final Map<String, Set<WebSocketSession>> localSessions =
        new ConcurrentHashMap<>();

    // 메시지 발행: Redis를 통해 모든 서버에 전달
    public void publishToRoom(String roomId, ChatMessage message) {
        try {
            String channel = "chat:room:" + roomId;
            String payload = objectMapper.writeValueAsString(message);
            redisTemplate.convertAndSend(channel, payload);
        } catch (JsonProcessingException e) {
            log.error("메시지 직렬화 실패", e);
        }
    }

    // Redis로부터 메시지 수신 → 로컬 세션에 전달
    public void onRedisMessage(Message message) {
        try {
            String channel = new String(message.getChannel());
            String roomId = channel.replace("chat:room:", "");
            String payload = new String(message.getBody());

            Set<WebSocketSession> sessions =
                localSessions.getOrDefault(roomId, Collections.emptySet());

            TextMessage wsMessage = new TextMessage(payload);
            List<String> deadSessions = new ArrayList<>();

            for (WebSocketSession session : sessions) {
                if (!session.isOpen()) {
                    deadSessions.add(session.getId());
                    continue;
                }
                try {
                    synchronized (session) {
                        session.sendMessage(wsMessage);
                    }
                } catch (IOException e) {
                    deadSessions.add(session.getId());
                }
            }

            // 종료된 세션 정리
            if (!deadSessions.isEmpty()) {
                sessions.removeIf(s -> deadSessions.contains(s.getId()));
            }

        } catch (Exception e) {
            log.error("Redis 메시지 처리 실패", e);
        }
    }
}
```

### STOMP + Redis 브로커 통합

Spring의 STOMP 지원은 Redis를 메시지 브로커로 사용하는 것을 직접 지원한다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class ScalableStompConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // 인메모리 브로커 대신 Redis 기반 외부 브로커 사용
        // Redis를 직접 사용하거나, Redis 위에 RabbitMQ/ActiveMQ를 얹는 방식도 가능
        config.enableSimpleBroker("/topic", "/queue")
              .setHeartbeatValue(new long[]{10000, 10000}) // 서버/클라이언트 heartbeat
              .setTaskScheduler(heartbeatScheduler());

        config.setApplicationDestinationPrefixes("/app");
        config.setUserDestinationPrefix("/user");
    }

    @Bean
    public TaskScheduler heartbeatScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(1);
        scheduler.setThreadNamePrefix("ws-heartbeat-");
        scheduler.initialize();
        return scheduler;
    }
}

// Redis Streams를 활용한 내구성 있는 메시지 전달
@Service
public class DurableMessageService {

    private static final String STREAM_KEY = "chat-messages";

    public void publishDurable(ChatMessage message) {
        Map<String, String> fields = Map.of(
            "roomId", message.getRoomId(),
            "sender", message.getSender(),
            "content", message.getContent(),
            "timestamp", message.getTimestamp().toString()
        );

        // Redis Streams에 메시지 추가 (내구성 보장)
        redisTemplate.opsForStream().add(STREAM_KEY, fields);

        // Pub/Sub로 실시간 전달 (빠른 전달용)
        redisTemplate.convertAndSend("chat:room:" + message.getRoomId(),
            objectMapper.writeValueAsString(message));
    }
}
```

### 연결 상태 관리: Redis를 활용한 프레즌스

누가 온라인 상태인지 추적하는 것도 분산 환경에서는 복잡해진다.

```java
@Service
@RequiredArgsConstructor
public class PresenceService {

    private final StringRedisTemplate redisTemplate;
    private static final String PRESENCE_PREFIX = "presence:";
    private static final long PRESENCE_TTL_SECONDS = 90; // 90초 TTL

    // 사용자 온라인 상태 등록
    public void setOnline(String userId, String serverId) {
        String key = PRESENCE_PREFIX + userId;
        redisTemplate.opsForValue().set(key, serverId,
            Duration.ofSeconds(PRESENCE_TTL_SECONDS));
    }

    // WebSocket 연결 유지 중 주기적으로 TTL 갱신
    @Scheduled(fixedDelay = 30000)
    public void renewPresence() {
        localSessionRegistry.getConnectedUserIds().forEach(userId -> {
            String key = PRESENCE_PREFIX + userId;
            redisTemplate.expire(key, Duration.ofSeconds(PRESENCE_TTL_SECONDS));
        });
    }

    // 사용자 오프라인 처리
    public void setOffline(String userId) {
        redisTemplate.delete(PRESENCE_PREFIX + userId);
    }

    // 온라인 여부 확인
    public boolean isOnline(String userId) {
        return redisTemplate.hasKey(PRESENCE_PREFIX + userId);
    }

    // 여러 사용자 온라인 상태 일괄 조회
    public Map<String, Boolean> getPresenceBatch(List<String> userIds) {
        List<String> keys = userIds.stream()
            .map(id -> PRESENCE_PREFIX + id)
            .collect(Collectors.toList());

        List<String> values = redisTemplate.opsForValue().multiGet(keys);

        Map<String, Boolean> result = new HashMap<>();
        for (int i = 0; i < userIds.size(); i++) {
            result.put(userIds.get(i), values.get(i) != null);
        }
        return result;
    }
}
```

### 확장 전략 선택 가이드

| 상황 | 권장 전략 |
|---|---|
| 단일 서버, 소규모 | 인메모리 세션 관리 |
| 멀티 서버, 같은 룸의 사용자가 같은 서버에 있어도 되는 경우 | Sticky Session |
| 멀티 서버, 크로스 서버 메시징 필요 | Redis Pub/Sub |
| 고가용성, 메시지 유실 불가 | Redis Streams + Pub/Sub 조합 |
| 대규모 팬아웃 (방송형) | Kafka + Redis Pub/Sub |

### 대규모 확장 시 안티패턴

**안티패턴 1: 메모리 내 세션 공유 시도**

```java
// 잘못된 접근: 서버 간 공유되지 않는 정적 맵
@Component
public class GlobalSessionRegistry {
    // 이 Map은 이 서버의 메모리에만 존재한다.
    // 다른 서버의 세션은 절대 보이지 않는다.
    private static final Map<String, WebSocketSession> sessions =
        new ConcurrentHashMap<>();
}
```

**안티패턴 2: 브로드캐스트 루프 내 동기 IO**

```java
// 잘못된 접근: 수천 개의 세션에 동기 전송
public void broadcastToAll(String message) {
    allSessions.forEach(session -> {
        // 각 session.sendMessage()가 블록될 수 있음
        // 하나가 느리면 전체 브로드캐스트가 지연됨
        session.sendMessage(new TextMessage(message));
    });
}

// 올바른 접근: 비동기 + 배치
public void broadcastToAll(String message) {
    TextMessage wsMessage = new TextMessage(message);
    List<CompletableFuture<Void>> futures = allSessions.stream()
        .filter(WebSocketSession::isOpen)
        .map(session -> CompletableFuture.runAsync(() -> {
            try {
                synchronized (session) {
                    session.sendMessage(wsMessage);
                }
            } catch (IOException e) {
                markSessionForRemoval(session);
            }
        }, broadcastExecutor))
        .collect(Collectors.toList());

    CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .orTimeout(5, TimeUnit.SECONDS)
        .exceptionally(ex -> {
            log.error("브로드캐스트 타임아웃", ex);
            return null;
        });
}
```

---

## 기술 선택 결정 트리

실시간 통신 기술을 선택할 때 다음 질문을 순서대로 던져라.

1. **클라이언트가 서버로 데이터를 실시간으로 보내야 하는가?**
   - Yes → WebSocket
   - No → 다음 질문으로

2. **초당 수십 번 이상의 빈번한 업데이트가 필요한가?**
   - Yes → SSE 또는 WebSocket
   - No → Long Polling으로 충분할 수 있음

3. **기존 HTTP 인프라(프록시, 캐시, 로드밸런서)를 그대로 활용해야 하는가?**
   - Yes → SSE (HTTP 기반)
   - No → WebSocket 검토

4. **브라우저 지원 범위가 중요한가?**
   - IE 지원 필요 → Long Polling 또는 SockJS(WebSocket 폴백)
   - 모던 브라우저만 → SSE 또는 WebSocket

결론적으로, 대부분의 단방향 서버 푸시 시나리오는 SSE로 충분하다.

채팅, 게임, 실시간 협업 도구처럼 진정한 양방향 통신이 필요한 경우에만 WebSocket의 복잡성을 감수할 가치가 있다.

그리고 어떤 기술을 선택하든, 분산 환경에서의 확장 전략은 처음부터 고려해야 한다.

Redis Pub/Sub를 나중에 끼워 넣는 것은 처음부터 설계하는 것보다 훨씬 고통스럽다.

---

## 참고 자료

1. [RFC 6455 - The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455) — WebSocket 프로토콜 표준 명세. 핸드셰이크, 프레이밍, 클로징 핸드셰이크의 세부 사항을 다룬다.

2. [W3C Server-Sent Events Specification](https://html.spec.whatwg.org/multipage/server-sent-events.html) — SSE의 공식 표준. `EventSource` API와 이벤트 스트림 형식을 정의한다.

3. [Spring Framework Reference - WebSocket Support](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#websocket) — Spring의 WebSocket, STOMP, SockJS 지원에 대한 공식 문서.

4. [Redis Pub/Sub Documentation](https://redis.io/docs/manual/pubsub/) — Redis Pub/Sub와 Streams의 차이점 및 적합한 사용 시나리오.

5. [High Performance Browser Networking — Chapter 17: WebSocket](https://hpbn.co/websocket/) — Ilya Grigorik의 저서. WebSocket 성능 특성과 HTTP/2와의 비교를 심층 분석한다.

6. [Scaling WebSockets — Ably Engineering Blog](https://ably.com/blog/websockets-vs-sse) — 대규모 실시간 서비스에서 WebSocket과 SSE를 수평 확장하는 실전 경험을 공유한다.
