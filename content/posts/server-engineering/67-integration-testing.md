---
title: "[테스트 전략] 2편 — 통합 테스트: 조각을 합치면"
date: 2026-03-17T16:03:00+09:00
draft: false
tags: ["통합 테스트", "Testcontainers", "WireMock", "SpringBootTest", "서버"]
series: ["테스트 전략"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 10
summary: "@SpringBootTest 슬라이스 테스트(WebMvcTest, DataJpaTest), Testcontainers로 실제 DB/Redis/Kafka 테스트, WireMock으로 외부 API 모킹, 트랜잭션 롤백 vs 테스트 데이터 정리 전략까지"
---

단위 테스트가 부품 하나하나를 검증한다면, 통합 테스트는 그 부품들이 실제로 연결됐을 때 어떻게 동작하는지 검증한다. Repository가 JPA 쿼리를 올바르게 생성하는지, Controller가 Service를 거쳐 DB까지 올바르게 관통하는지, 외부 결제 API 실패 시 롤백이 정상적으로 이뤄지는지 — 이런 질문들은 단위 테스트로는 답할 수 없다. 통합 테스트는 느리고 무겁다는 편견이 있지만, 전략적으로 설계하면 빠르면서도 신뢰도 높은 테스트 스위트를 만들 수 있다.

---

## 1. @SpringBootTest 슬라이스 테스트

### 슬라이스 테스트란?

`@SpringBootTest`는 애플리케이션 컨텍스트 전체를 띄운다.

모든 Bean을 로드하기 때문에 느리고, 테스트 격리가 어렵다.

슬라이스 테스트는 특정 레이어에 필요한 Bean만 선택적으로 로드해 이 문제를 해결한다.

### @WebMvcTest — Web 레이어 테스트

`@WebMvcTest`는 Controller, ControllerAdvice, Filter, WebMvcConfigurer 등 웹 레이어 Bean만 로드한다.

Service, Repository 등은 로드하지 않으므로 `@MockBean`으로 대체해야 한다.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private OrderService orderService;

    @MockBean
    private OrderMapper orderMapper;

    @Test
    @DisplayName("주문 생성 성공 시 201 응답과 주문 ID 반환")
    void createOrder_success() throws Exception {
        // given
        CreateOrderRequest request = new CreateOrderRequest(1L, List.of(
            new OrderItemRequest(101L, 2),
            new OrderItemRequest(102L, 1)
        ));
        OrderResponse response = new OrderResponse(999L, "PENDING", LocalDateTime.now());

        given(orderService.createOrder(any(CreateOrderRequest.class))).willReturn(response);

        // when & then
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.orderId").value(999L))
            .andExpect(jsonPath("$.status").value("PENDING"));
    }

    @Test
    @DisplayName("요청 바디 없으면 400 반환")
    void createOrder_invalidRequest() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errorCode").value("INVALID_REQUEST"));
    }
}
```

### @WebMvcTest 주의사항

Security 설정이 있다면 테스트에서도 동작한다.

인증이 필요한 엔드포인트 테스트 시 `@WithMockUser`나 `@WithUserDetails`를 사용하거나, Security 설정을 제외해야 한다.

```java
@WebMvcTest(OrderController.class)
@Import(SecurityTestConfig.class)  // 테스트용 Security 설정 적용
class OrderControllerTest {
    // ...
}

@TestConfiguration
public class SecurityTestConfig {
    @Bean
    public SecurityFilterChain testSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .build();
    }
}
```

### @DataJpaTest — JPA 레이어 테스트

`@DataJpaTest`는 JPA 관련 Bean(EntityManager, Repository, DataSource 등)만 로드한다.

기본적으로 인메모리 H2 데이터베이스를 사용하며, 각 테스트는 트랜잭션으로 감싸져 롤백된다.

```java
@DataJpaTest
class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    @DisplayName("사용자 ID로 최근 30일 주문 조회")
    void findRecentOrdersByUserId() {
        // given
        User user = entityManager.persistAndFlush(
            User.builder()
                .email("test@example.com")
                .name("테스터")
                .build()
        );

        Order recentOrder = entityManager.persistAndFlush(
            Order.builder()
                .user(user)
                .status(OrderStatus.COMPLETED)
                .createdAt(LocalDateTime.now().minusDays(10))
                .build()
        );

        Order oldOrder = entityManager.persistAndFlush(
            Order.builder()
                .user(user)
                .status(OrderStatus.COMPLETED)
                .createdAt(LocalDateTime.now().minusDays(40))
                .build()
        );

        entityManager.clear(); // 1차 캐시 초기화

        // when
        List<Order> orders = orderRepository.findRecentByUserId(
            user.getId(),
            LocalDateTime.now().minusDays(30)
        );

        // then
        assertThat(orders).hasSize(1);
        assertThat(orders.get(0).getId()).isEqualTo(recentOrder.getId());
    }
}
```

### @DataJpaTest의 함정: H2 방언 문제

H2는 MySQL/PostgreSQL과 SQL 문법이 다르다.

MySQL 전용 함수(`DATE_FORMAT`, `GROUP_CONCAT`, JSON 함수 등)를 JPQL이나 Native Query에서 사용하면 H2에서 실패한다.

이 경우 Testcontainers로 실제 DB를 띄우거나, `@AutoConfigureTestDatabase(replace = NONE)`으로 H2 대신 실제 DB를 사용해야 한다.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryWithMySQLTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    // 실제 MySQL 문법으로 테스트 가능
}
```

---

## 2. Testcontainers로 실제 인프라 테스트

### Testcontainers란?

Testcontainers는 테스트 코드에서 Docker 컨테이너를 프로그래밍 방식으로 제어하는 라이브러리다.

실제 MySQL, Redis, Kafka, Elasticsearch를 테스트 시작 시 컨테이너로 띄우고, 테스트 종료 시 자동으로 정리한다.

### 의존성 설정

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers-bom</artifactId>
            <version>1.19.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mysql</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>kafka</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 컨테이너 공유 — 성능의 핵심

각 테스트 클래스마다 컨테이너를 새로 띄우면 테스트가 매우 느려진다.

`static` 필드로 컨테이너를 선언하면 JUnit 5의 생명주기에 따라 클래스 내에서 공유된다.

더 나아가 추상 기반 클래스로 컨테이너를 공유하면 테스트 스위트 전체에서 컨테이너를 재사용할 수 있다.

```java
// 추상 기반 클래스 — 한 번만 컨테이너 시작
@SpringBootTest
@Testcontainers
public abstract class IntegrationTestBase {

    @Container
    static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("integration_test")
        .withUsername("test")
        .withPassword("test")
        .withReuse(true);  // 컨테이너 재사용 (로컬 개발 환경에서 유용)

    @Container
    static final GenericContainer<?> REDIS = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379)
        .withReuse(true);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
        registry.add("spring.datasource.username", MYSQL::getUsername);
        registry.add("spring.datasource.password", MYSQL::getPassword);

        registry.add("spring.data.redis.host", REDIS::getHost);
        registry.add("spring.data.redis.port", () -> REDIS.getMappedPort(6379));
    }
}
```

```java
// 개별 테스트는 기반 클래스 상속
class OrderServiceIntegrationTest extends IntegrationTestBase {

    @Autowired
    private OrderService orderService;

    @Test
    void createAndRetrieveOrder() {
        // 실제 MySQL과 Redis로 테스트
    }
}
```

### Kafka 통합 테스트

Kafka 테스트는 비동기 특성 때문에 특별한 처리가 필요하다.

메시지 발행 후 소비자가 처리할 때까지 기다려야 하므로 `Awaitility`를 함께 사용한다.

```java
@SpringBootTest
@Testcontainers
class OrderEventIntegrationTest {

    @Container
    static final KafkaContainer KAFKA = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    );

    @DynamicPropertySource
    static void configureKafka(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", KAFKA::getBootstrapServers);
    }

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderEventRepository orderEventRepository;

    @Test
    @DisplayName("주문 생성 시 OrderCreated 이벤트가 발행되고 이벤트 저장소에 기록됨")
    void createOrder_publishesEvent() {
        // given
        CreateOrderRequest request = buildCreateOrderRequest();

        // when
        orderService.createOrder(request);

        // then - Kafka 소비자가 메시지를 처리할 때까지 대기
        await()
            .atMost(10, SECONDS)
            .pollInterval(500, MILLISECONDS)
            .untilAsserted(() -> {
                List<OrderEvent> events = orderEventRepository.findByType("ORDER_CREATED");
                assertThat(events).hasSize(1);
                assertThat(events.get(0).getPayload()).contains("\"status\":\"PENDING\"");
            });
    }
}
```

### @ServiceConnection — Spring Boot 3.1+

Spring Boot 3.1부터는 `@ServiceConnection`으로 `@DynamicPropertySource` 없이 자동으로 연결 설정을 주입할 수 있다.

```java
@SpringBootTest
@Testcontainers
class ModernIntegrationTest {

    @Container
    @ServiceConnection
    static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.0");

    @Container
    @ServiceConnection
    static final RedisContainer REDIS = new RedisContainer("redis:7-alpine");

    // @DynamicPropertySource 불필요
    // Spring Boot가 자동으로 연결 속성 설정
}
```

---

## 3. WireMock으로 외부 API 모킹

### 왜 WireMock인가?

외부 API(결제사, 문자 발송, 소셜 로그인 등)를 실제로 호출하면 테스트가 외부 시스템에 종속된다.

네트워크 불안정, API 할당량 소진, 과금 발생 등의 문제가 생긴다.

WireMock은 HTTP 서버를 로컬에 띄워 외부 API 응답을 완전히 제어할 수 있게 한다.

### 의존성 설정

```xml
<dependency>
    <groupId>org.wiremock.integrations</groupId>
    <artifactId>wiremock-spring-boot</artifactId>
    <version>3.1.0</version>
    <scope>test</scope>
</dependency>
```

### 기본 WireMock 설정

```java
@SpringBootTest
@EnableWireMock({
    @ConfigureWireMock(name = "payment-service", property = "payment.api.base-url"),
    @ConfigureWireMock(name = "notification-service", property = "notification.api.base-url")
})
class PaymentIntegrationTest {

    @InjectWireMock("payment-service")
    private WireMockServer paymentWireMock;

    @Autowired
    private PaymentService paymentService;

    @Test
    @DisplayName("결제 API 성공 응답 시 주문 상태를 PAID로 변경")
    void processPayment_success() {
        // given - WireMock 스텁 설정
        paymentWireMock.stubFor(
            post(urlEqualTo("/api/v1/payments"))
                .withHeader("Content-Type", equalTo("application/json"))
                .withRequestBody(matchingJsonPath("$.amount", equalTo("50000")))
                .willReturn(aResponse()
                    .withStatus(200)
                    .withHeader("Content-Type", "application/json")
                    .withBody("""
                        {
                            "transactionId": "TXN-20240101-001",
                            "status": "APPROVED",
                            "approvedAt": "2024-01-01T10:00:00"
                        }
                        """)
                )
        );

        // when
        PaymentResult result = paymentService.processPayment(999L, BigDecimal.valueOf(50000));

        // then
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.APPROVED);
        assertThat(result.getTransactionId()).isEqualTo("TXN-20240101-001");

        // 실제로 외부 API가 호출됐는지 검증
        paymentWireMock.verify(1, postRequestedFor(urlEqualTo("/api/v1/payments")));
    }

    @Test
    @DisplayName("결제 API 타임아웃 시 PaymentTimeoutException 발생")
    void processPayment_timeout() {
        // given
        paymentWireMock.stubFor(
            post(urlEqualTo("/api/v1/payments"))
                .willReturn(aResponse()
                    .withFixedDelay(5000)  // 5초 지연
                    .withStatus(200)
                )
        );

        // when & then
        assertThatThrownBy(() -> paymentService.processPayment(999L, BigDecimal.valueOf(50000)))
            .isInstanceOf(PaymentTimeoutException.class);
    }

    @Test
    @DisplayName("결제 API 500 에러 시 3번 재시도 후 실패")
    void processPayment_retryOnServerError() {
        // given
        paymentWireMock.stubFor(
            post(urlEqualTo("/api/v1/payments"))
                .willReturn(serverError())
        );

        // when
        assertThatThrownBy(() -> paymentService.processPayment(999L, BigDecimal.valueOf(50000)))
            .isInstanceOf(PaymentException.class);

        // then - 재시도 포함 총 3번 호출됐는지 검증
        paymentWireMock.verify(3, postRequestedFor(urlEqualTo("/api/v1/payments")));
    }
}
```

### Scenario를 활용한 상태 기반 테스트

결제 후 조회처럼 호출 순서에 따라 다른 응답이 필요한 시나리오를 테스트할 수 있다.

```java
@Test
@DisplayName("결제 대기 → 결제 완료 상태 변경 폴링")
void checkPaymentStatus_stateTransition() {
    // 첫 번째 호출: PENDING
    paymentWireMock.stubFor(
        get(urlEqualTo("/api/v1/payments/TXN-001"))
            .inScenario("payment-status")
            .whenScenarioStateIs(Scenario.STARTED)
            .willReturn(okJson("""
                {"transactionId": "TXN-001", "status": "PENDING"}
                """))
            .willSetStateTo("PROCESSING")
    );

    // 두 번째 호출: PROCESSING
    paymentWireMock.stubFor(
        get(urlEqualTo("/api/v1/payments/TXN-001"))
            .inScenario("payment-status")
            .whenScenarioStateIs("PROCESSING")
            .willReturn(okJson("""
                {"transactionId": "TXN-001", "status": "APPROVED"}
                """))
    );

    // when - 완료될 때까지 폴링
    PaymentStatus finalStatus = paymentService.pollUntilComplete("TXN-001", Duration.ofSeconds(10));

    // then
    assertThat(finalStatus).isEqualTo(PaymentStatus.APPROVED);
}
```

### WireMock 안티패턴

**안티패턴 1: 스텁 응답을 프로덕션 코드에서 분기 처리**

테스트 전용 로직이 프로덕션 코드에 침투하면 코드가 오염된다.

WireMock은 HTTP 레벨에서 완전히 투명하게 동작해야 한다.

**안티패턴 2: 검증 없이 스텁만 설정**

스텁을 설정해도 `verify()`로 실제 호출 여부를 검증하지 않으면, 코드가 외부 API를 전혀 호출하지 않아도 테스트가 통과한다.

**안티패턴 3: 너무 세밀한 요청 매칭**

요청 바디 전체를 정확히 매칭하면 내부 구현 변경 시 테스트가 깨진다.

핵심 필드만 `matchingJsonPath`로 검증하는 것이 유지보수에 유리하다.

---

## 4. 트랜잭션 롤백 vs 테스트 데이터 정리 전략

### 두 가지 접근법

통합 테스트에서 테스트 간 데이터 격리는 가장 까다로운 문제 중 하나다.

잘못 관리하면 테스트 순서에 따라 결과가 달라지는 플리키(flaky) 테스트가 만들어진다.

### 전략 1: @Transactional 롤백

`@Transactional`을 테스트 메서드에 붙이면 테스트 종료 후 자동으로 롤백된다.

```java
@SpringBootTest
@Transactional  // 각 테스트 후 롤백
class OrderServiceTransactionalTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void createOrder_savedToDatabase() {
        // given
        CreateOrderRequest request = buildRequest();

        // when
        OrderResponse response = orderService.createOrder(request);

        // then - 같은 트랜잭션 내에서 조회
        Order saved = orderRepository.findById(response.getOrderId()).orElseThrow();
        assertThat(saved.getStatus()).isEqualTo(OrderStatus.PENDING);
        // 테스트 종료 후 롤백 — DB 오염 없음
    }
}
```

**@Transactional 롤백의 한계:**

이벤트 기반 처리(Kafka, 비동기 로직)는 별도 트랜잭션에서 실행되므로 롤백되지 않는다.

`REQUIRES_NEW`로 선언된 메서드나 `TransactionSynchronizationManager`를 사용하는 코드도 별도 트랜잭션으로 커밋되어 롤백이 안 된다.

실제 서비스 코드의 트랜잭션 경계와 테스트 트랜잭션이 달라 숨겨진 버그를 놓칠 수 있다.

### 전략 2: @Sql 스크립트로 초기화

각 테스트 전후 SQL 스크립트를 실행해 데이터를 초기화한다.

```java
@SpringBootTest
class OrderServiceSqlTest {

    @Autowired
    private OrderService orderService;

    @Test
    @Sql(scripts = "/sql/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    @Sql(scripts = "/sql/seed-users.sql", executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD)
    void createOrder_withExistingUser() {
        // seed 데이터로 테스트
    }
}
```

```sql
-- /test/resources/sql/cleanup.sql
DELETE FROM order_items WHERE created_at > '2020-01-01';
DELETE FROM orders WHERE created_at > '2020-01-01';
TRUNCATE TABLE order_events;
```

### 전략 3: @DirtiesContext (최후의 수단)

`@DirtiesContext`는 테스트 후 Spring ApplicationContext를 재생성한다.

완전한 격리를 보장하지만 컨텍스트 재생성 비용이 매우 크다.

정말 불가피한 상황이 아니면 사용을 피해야 한다.

### 전략 4: DatabaseCleaner 유틸리티 클래스

가장 실용적인 방법은 테스트 전 관련 테이블을 직접 정리하는 유틸리티를 만드는 것이다.

```java
@Component
@Profile("test")
public class DatabaseCleaner {

    @Autowired
    private DataSource dataSource;

    @Value("${test.clean.tables:orders,order_items,payments}")
    private List<String> tablesToClean;

    @Transactional
    public void cleanUp() {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {

            stmt.execute("SET FOREIGN_KEY_CHECKS = 0");  // MySQL

            for (String table : tablesToClean) {
                stmt.execute("TRUNCATE TABLE " + table);
            }

            stmt.execute("SET FOREIGN_KEY_CHECKS = 1");
        } catch (SQLException e) {
            throw new RuntimeException("데이터베이스 정리 실패", e);
        }
    }
}
```

```java
@SpringBootTest
class OrderServiceCleanTest extends IntegrationTestBase {

    @Autowired
    private OrderService orderService;

    @Autowired
    private DatabaseCleaner dbCleaner;

    @BeforeEach
    void setUp() {
        dbCleaner.cleanUp();  // 각 테스트 전 초기화
    }

    @Test
    void createOrder() {
        // 깨끗한 상태에서 테스트
    }
}
```

### 전략 비교

| 전략 | 속도 | 격리 수준 | 비동기 지원 | 권장 상황 |
|------|------|-----------|------------|-----------|
| `@Transactional` 롤백 | 빠름 | 중간 | X | 단순 CRUD, 동기 로직 |
| `@Sql` 스크립트 | 중간 | 높음 | O | 복잡한 초기 데이터 필요 |
| `DatabaseCleaner` | 중간 | 높음 | O | 일반적인 통합 테스트 |
| `@DirtiesContext` | 매우 느림 | 완전 | O | 컨텍스트 오염이 불가피한 경우 |

### 실전 권장 패턴

비동기와 이벤트 기반 로직이 많은 현대 서비스에서는 `@Transactional` 롤백에 의존하지 말아야 한다.

`DatabaseCleaner`를 `@BeforeEach`에서 실행하고, Testcontainers로 격리된 DB 환경을 제공하는 조합이 가장 실용적이다.

---

## 마치며

통합 테스트는 단위 테스트가 잡지 못하는 레이어 간 계약과 인프라 연동 버그를 잡아낸다.

슬라이스 테스트로 빠른 피드백 루프를 만들고, Testcontainers로 운영 환경에 가까운 인프라를 테스트하고, WireMock으로 외부 의존성을 통제하는 것 — 이 세 가지가 통합 테스트 전략의 핵심이다.

테스트 데이터 관리는 처음부터 전략적으로 접근해야 한다. 나중에 플리키 테스트를 디버깅하는 비용이 훨씬 크다.

---

## 참고 자료

1. [Testcontainers 공식 문서](https://testcontainers.com/guides/getting-started-with-testcontainers-for-java/) — Java 통합 테스트 가이드
2. [WireMock 공식 문서](https://wiremock.org/docs/) — HTTP 모킹 레퍼런스
3. [Spring Boot Testing Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing) — @WebMvcTest, @DataJpaTest 공식 문서
4. [Testcontainers Spring Boot Module](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing.testcontainers) — @ServiceConnection 활용
5. [Awaitility GitHub](https://github.com/awaitility/awaitility) — 비동기 테스트 유틸리티
6. Phil Haack, "Integration Testing with Testcontainers" — 컨테이너 재사용 전략 심층 분석
