---
title: "[테스트 전략] 3편 — E2E 테스트와 계약 테스트"
date: 2026-03-17T16:02:00+09:00
draft: false
tags: ["E2E 테스트", "Pact", "계약 테스트", "CDC", "서버"]
series: ["테스트 전략"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 10
summary: "E2E 테스트 시나리오 설계와 환경 구성, Consumer-Driven Contract Testing(Pact), E2E 테스트의 유지 비용과 선택 전략, 성능 테스트 자동화 CI 연동까지"
---

단위 테스트와 통합 테스트가 견고하게 짜여 있어도, 실제 사용자 시나리오에서 시스템이 무너지는 경우가 있다. 서비스 간 인터페이스가 어느 날 조용히 깨지거나, 배포 직후 프로덕션에서 처음 발견되는 회귀 버그가 그 증거다. E2E(End-to-End) 테스트와 계약 테스트(Contract Testing)는 이 간극을 메우는 두 가지 전략이다. 이 글에서는 두 전략의 원리와 설계 방법, 그리고 지속 가능한 유지 비용 관리 방법을 실전 코드와 함께 살펴본다.

---

## 1. E2E 테스트 — 시나리오 설계와 환경 구성

### E2E 테스트란 무엇인가

E2E 테스트는 시스템의 입구부터 출구까지 실제 흐름을 검증한다.

사용자가 브라우저에서 버튼을 누르거나 클라이언트가 API를 호출할 때, 그 요청이 여러 서비스와 데이터베이스를 거쳐 최종 응답으로 돌아오는 전체 경로를 테스트한다.

단위 테스트가 함수 하나를 검증한다면, E2E 테스트는 여러 컴포넌트가 통합된 시스템의 동작을 검증한다.

### 무엇을 E2E 테스트로 다룰 것인가

모든 기능을 E2E로 커버하려 하면 실패한다.

E2E 테스트는 비용이 크다. 느리고, 환경 의존성이 높고, 실패 원인 파악이 어렵다.

다음 기준으로 E2E 테스트 대상을 선별해야 한다.

- **핵심 비즈니스 경로 (Happy Path)**: 회원가입, 로그인, 결제 등 서비스의 핵심 흐름
- **고가치 통합 지점**: 외부 결제 게이트웨이, 알림 서비스 등 장애 시 비즈니스 임팩트가 큰 지점
- **회귀 위험이 높은 시나리오**: 여러 번 깨진 이력이 있는 흐름

안티패턴: 모든 비즈니스 로직 분기를 E2E로 검증하려는 것. 이는 통합 테스트나 단위 테스트의 역할이다.

### 시나리오 설계 원칙

좋은 E2E 시나리오는 사용자의 관점에서 시작한다.

"사용자가 상품을 장바구니에 담고 결제를 완료한다"처럼 행위를 중심으로 시나리오를 서술한다.

각 시나리오는 독립적이어야 한다. 테스트 A가 테스트 B의 데이터 상태에 의존하면, 실행 순서에 따라 결과가 달라지는 불안정한 테스트가 된다.

```
시나리오: 주문 생성 및 결제 흐름

1. 사용자 인증 (JWT 발급)
2. 상품 목록 조회
3. 장바구니에 상품 추가
4. 주문 생성
5. 결제 처리
6. 주문 상태 확인 (COMPLETED)
7. 이메일 알림 수신 확인
```

각 단계 사이에는 명시적인 상태 검증이 필요하다. 단순히 HTTP 200을 받는 것이 아니라, 응답 본문의 값과 데이터베이스 상태, 부수 효과까지 검증한다.

### E2E 테스트 환경 구성

E2E 테스트 환경은 프로덕션과 최대한 유사해야 한다.

그렇다고 프로덕션 데이터를 사용하는 것은 위험하다. 격리된 환경에서 실제와 동일한 구성을 갖추는 것이 목표다.

Docker Compose를 활용하면 로컬과 CI 모두에서 일관된 환경을 구성할 수 있다.

```yaml
# docker-compose.e2e.yml
version: "3.9"

services:
  app:
    image: myapp:${IMAGE_TAG:-latest}
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=e2e
      - DB_URL=jdbc:postgresql://db:5432/testdb
      - REDIS_URL=redis://redis:6379
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test"]
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    depends_on:
      - zookeeper
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 5s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  wiremock:
    image: wiremock/wiremock:3.3.1
    ports:
      - "9090:8080"
    volumes:
      - ./wiremock:/home/wiremock
    command: ["--global-response-templating"]
```

외부 서비스(결제 PG사, SMS 발송 등)는 WireMock으로 대체한다.

프로덕션 네트워크에 의존하는 테스트는 환경 변화에 취약하고 CI 속도를 저하시킨다.

### Java + RestAssured E2E 테스트 예시

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
@Testcontainers
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderFlowE2ETest {

    private static final String BASE_URL = "http://localhost:8080";

    @BeforeAll
    static void setup() {
        // Docker Compose 기반 환경이 이미 기동된 상태를 가정
        // 또는 @DockerComposeContainer 사용
        RestAssured.baseURI = BASE_URL;
    }

    @Test
    @Order(1)
    void 사용자_인증_후_JWT_발급() {
        String token = given()
            .contentType(ContentType.JSON)
            .body("""
                {
                  "email": "test@example.com",
                  "password": "Test1234!"
                }
                """)
        .when()
            .post("/api/auth/login")
        .then()
            .statusCode(200)
            .body("token", notNullValue())
            .extract().path("token");

        // 다음 테스트에서 사용할 토큰 저장
        System.setProperty("test.token", token);
    }

    @Test
    @Order(2)
    void 상품_목록_조회() {
        String token = System.getProperty("test.token");

        given()
            .header("Authorization", "Bearer " + token)
        .when()
            .get("/api/products?category=electronics")
        .then()
            .statusCode(200)
            .body("content", hasSize(greaterThan(0)))
            .body("content[0].id", notNullValue())
            .body("content[0].stock", greaterThan(0));
    }

    @Test
    @Order(3)
    void 주문_생성_및_결제_완료_흐름() {
        String token = System.getProperty("test.token");

        // 주문 생성
        String orderId = given()
            .header("Authorization", "Bearer " + token)
            .contentType(ContentType.JSON)
            .body("""
                {
                  "items": [
                    {"productId": "prod-001", "quantity": 2}
                  ],
                  "shippingAddress": {
                    "street": "테헤란로 123",
                    "city": "서울",
                    "zipCode": "06142"
                  }
                }
                """)
        .when()
            .post("/api/orders")
        .then()
            .statusCode(201)
            .body("status", equalTo("PENDING_PAYMENT"))
            .extract().path("orderId");

        // 결제 처리 (WireMock이 결제 PG 대체)
        given()
            .header("Authorization", "Bearer " + token)
            .contentType(ContentType.JSON)
            .body(String.format("""
                {
                  "orderId": "%s",
                  "paymentMethod": "CARD",
                  "cardToken": "tok_test_visa"
                }
                """, orderId))
        .when()
            .post("/api/payments")
        .then()
            .statusCode(200)
            .body("status", equalTo("SUCCESS"));

        // 주문 상태 최종 확인 (비동기 처리를 고려한 폴링)
        await()
            .atMost(10, SECONDS)
            .pollInterval(500, MILLISECONDS)
            .untilAsserted(() ->
                given()
                    .header("Authorization", "Bearer " + token)
                .when()
                    .get("/api/orders/" + orderId)
                .then()
                    .statusCode(200)
                    .body("status", equalTo("COMPLETED"))
            );
    }
}
```

비동기 처리가 포함된 흐름에는 `await().untilAsserted()`처럼 폴링 기반 검증을 사용한다.

단순히 `Thread.sleep()`으로 대기하는 것은 느리고 불안정하다. Awaitility 라이브러리가 이를 우아하게 해결한다.

---

## 2. Consumer-Driven Contract Testing (Pact)

### 계약 테스트가 필요한 이유

마이크로서비스 아키텍처에서는 서비스 간 API 계약이 암묵적으로 유지된다.

Provider(서버) 팀이 응답 스키마를 변경했을 때, Consumer(클라이언트) 팀은 배포 후에야 깨진 사실을 알게 된다.

E2E 테스트로 이를 검증하려면 모든 서비스를 한 환경에 올려야 한다. 이는 배포와 환경 관리의 복잡도를 크게 높인다.

Consumer-Driven Contract Testing(CDC)은 이 문제를 다른 방식으로 푼다. Consumer가 자신이 필요로 하는 계약(Pact)을 정의하고, Provider는 그 계약을 만족하는지 독립적으로 검증한다.

### Pact 워크플로우

```
1. Consumer 팀: Pact 테스트 작성 → pact 파일(.json) 생성
2. pact 파일을 Pact Broker에 게시
3. Provider 팀: Pact Broker에서 계약 내려받아 검증 실행
4. 검증 결과를 Pact Broker에 등록
5. Can-I-Deploy 체크로 안전한 배포 여부 확인
```

이 흐름이 CI/CD에 통합되면, 계약을 깨는 변경은 파이프라인에서 자동으로 차단된다.

### Consumer 측 Pact 테스트 (Java)

```java
// Consumer: order-service가 product-service를 호출하는 계약
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "product-service", port = "8081")
class ProductServiceClientPactTest {

    @Pact(consumer = "order-service")
    public RequestResponsePact getProductById(PactDslWithProvider builder) {
        return builder
            .given("상품 prod-001이 존재하고 재고가 있다")
            .uponReceiving("상품 ID로 단일 상품 조회")
                .path("/api/products/prod-001")
                .method("GET")
                .headers(Map.of("Accept", "application/json"))
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(new PactDslJsonBody()
                    .stringType("id", "prod-001")
                    .stringType("name", "MacBook Pro 14")
                    .decimalType("price", 3500000.00)
                    .integerType("stock", 10)
                    .stringMatcher("status", "AVAILABLE|OUT_OF_STOCK", "AVAILABLE")
                )
            .toPact();
    }

    @Pact(consumer = "order-service")
    public RequestResponsePact getProductNotFound(PactDslWithProvider builder) {
        return builder
            .given("상품 prod-999가 존재하지 않는다")
            .uponReceiving("존재하지 않는 상품 ID 조회")
                .path("/api/products/prod-999")
                .method("GET")
            .willRespondWith()
                .status(404)
                .body(new PactDslJsonBody()
                    .stringType("error", "Product not found")
                    .stringType("code", "PRODUCT_NOT_FOUND")
                )
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getProductById")
    void 상품_조회_성공_계약_검증(MockServer mockServer) {
        ProductServiceClient client = new ProductServiceClient(
            "http://localhost:" + mockServer.getPort()
        );

        ProductDto product = client.getProductById("prod-001");

        assertThat(product.getId()).isEqualTo("prod-001");
        assertThat(product.getStock()).isGreaterThan(0);
        assertThat(product.getStatus()).isEqualTo("AVAILABLE");
    }

    @Test
    @PactTestFor(pactMethod = "getProductNotFound")
    void 존재하지_않는_상품_조회_404_계약_검증(MockServer mockServer) {
        ProductServiceClient client = new ProductServiceClient(
            "http://localhost:" + mockServer.getPort()
        );

        assertThatThrownBy(() -> client.getProductById("prod-999"))
            .isInstanceOf(ProductNotFoundException.class)
            .hasMessageContaining("PRODUCT_NOT_FOUND");
    }
}
```

`PactDslJsonBody`에서 `stringType`, `decimalType`, `integerType`은 값이 아닌 타입을 검증하도록 지정한다.

Consumer가 특정 값에 의존하는 계약이 아니라 구조와 타입에 의존하는 계약을 만들어야 Provider에 불필요한 제약을 주지 않는다.

### Provider 측 Pact 검증 (Java Spring)

```java
@Provider("product-service")
@PactBroker(
    url = "${PACT_BROKER_URL:http://localhost:9292}",
    authentication = @PactBrokerAuth(
        username = "${PACT_BROKER_USER:admin}",
        password = "${PACT_BROKER_PASSWORD:admin}"
    )
)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductServicePactProviderTest {

    @LocalServerPort
    private int port;

    @Autowired
    private ProductRepository productRepository;

    @BeforeEach
    void setup(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }

    // Pact State Handler: Consumer가 정의한 "given" 상태를 실제 데이터로 설정
    @State("상품 prod-001이 존재하고 재고가 있다")
    void setupProductExists() {
        productRepository.save(Product.builder()
            .id("prod-001")
            .name("MacBook Pro 14")
            .price(new BigDecimal("3500000.00"))
            .stock(10)
            .status(ProductStatus.AVAILABLE)
            .build());
    }

    @State("상품 prod-999가 존재하지 않는다")
    void setupProductNotExists() {
        productRepository.deleteById("prod-999");
    }
}
```

State Handler는 Consumer가 `given()`으로 정의한 전제 조건을 실제 데이터로 구현하는 역할을 한다.

Provider 팀이 Consumer의 기대를 이해하고 테스트 데이터를 준비하는 과정에서 자연스럽게 계약에 대한 공유 이해가 형성된다.

### Pact Broker 설정

```yaml
# pact-broker/docker-compose.yml
version: "3"
services:
  pact-broker:
    image: pactfoundation/pact-broker:latest
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: "postgres://pact:pact@db/pact"
      PACT_BROKER_LOG_LEVEL: INFO
      PACT_BROKER_SQL_LOG_LEVEL: none
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: pact
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact
    volumes:
      - pact-db-data:/var/lib/postgresql/data

volumes:
  pact-db-data:
```

### Can-I-Deploy 체크

Pact Broker의 핵심 기능 중 하나는 `can-i-deploy`다.

특정 서비스 버전을 배포해도 안전한지 계약 검증 기록을 바탕으로 판단해준다.

```bash
# CI 파이프라인에서 배포 전 안전성 확인
pact-broker can-i-deploy \
  --pacticipant product-service \
  --version ${GIT_COMMIT} \
  --to-environment production \
  --broker-base-url ${PACT_BROKER_URL}
```

검증이 통과된 버전 쌍만 배포를 허용함으로써, 계약 위반으로 인한 프로덕션 장애를 사전에 차단한다.

---

## 3. E2E 테스트의 유지 비용과 선택 전략

### E2E 테스트가 비싼 이유

E2E 테스트의 유지 비용은 시간이 지날수록 높아진다.

환경이 복잡해질수록 불안정한(Flaky) 테스트가 늘어나고, 실패 원인 파악에 시간이 걸리며, UI나 API가 변경될 때마다 테스트를 수정해야 한다.

다음은 대표적인 유지 비용 발생 요인이다.

| 요인 | 설명 |
|---|---|
| Flaky 테스트 | 타이밍 이슈, 네트워크 지연으로 간헐적 실패 |
| 느린 피드백 | E2E 스위트 전체 실행에 수십 분 소요 |
| 환경 불안정 | 의존 서비스 버전 충돌, 포트 충돌 |
| 높은 수정 비용 | API 스키마 변경 시 다수의 테스트 수정 필요 |
| 디버깅 어려움 | 실패 지점이 여러 서비스에 걸쳐 있어 원인 추적 복잡 |

### 테스트 피라미드와 선택 전략

테스트 피라미드는 단위 테스트를 넓은 기반으로, E2E 테스트를 좁은 꼭짓점으로 유지하라고 권고한다.

```
        /\
       /E2E\          ← 소수의 핵심 시나리오
      /------\
     / 통합 테스트 \    ← 서비스 간 연동 검증
    /------------\
   /   단위 테스트   \  ← 로직, 도메인 규칙
  /------------------\
```

실제로 많은 팀이 이 원칙을 어기고 E2E 테스트를 과도하게 늘린다.

E2E 테스트를 추가하기 전에 반드시 물어야 한다: "이것이 단위 테스트나 통합 테스트로 검증 가능한가?"

### 계약 테스트와 E2E 테스트의 역할 분담

계약 테스트(Pact)는 E2E 테스트의 보완재지, 대체재가 아니다.

| 검증 목적 | 적합한 테스트 |
|---|---|
| API 스키마 호환성 | 계약 테스트 (Pact) |
| 서비스 간 데이터 흐름 | 통합 테스트 |
| 핵심 비즈니스 경로 | E2E 테스트 |
| 비즈니스 로직 분기 | 단위 테스트 |
| 인프라 장애 대응 | 카오스 엔지니어링 |

마이크로서비스 환경에서는 계약 테스트를 먼저 충분히 구축하고, E2E 테스트는 가장 중요한 몇 가지 흐름에만 집중하는 것이 효율적이다.

### Flaky 테스트 관리 전략

Flaky 테스트는 E2E 테스트 신뢰성을 무너뜨리는 주된 원인이다.

팀이 "어차피 가끔 실패하는 테스트"에 익숙해지면, 실제 장애 신호를 무시하는 문화가 형성된다.

Flaky 테스트 방지를 위한 실천 방법은 다음과 같다.

```java
// 나쁜 예: 고정 대기 시간
Thread.sleep(3000); // 느리고 불안정
assertThat(order.getStatus()).isEqualTo("COMPLETED");

// 좋은 예: 조건 기반 폴링
await()
    .atMost(Duration.ofSeconds(10))
    .pollInterval(Duration.ofMillis(500))
    .ignoreExceptions()
    .untilAsserted(() -> {
        Order order = orderRepository.findById(orderId).orElseThrow();
        assertThat(order.getStatus()).isEqualTo(OrderStatus.COMPLETED);
    });
```

테스트 데이터는 각 테스트 실행 전에 초기화하고, 테스트 후에는 정리한다.

공유 상태는 실행 순서에 대한 암묵적 의존성을 만들어 Flaky 테스트의 온상이 된다.

---

## 4. 성능 테스트 자동화와 CI 연동

### 성능 테스트를 CI에 통합해야 하는 이유

성능 테스트는 흔히 릴리스 직전 수동으로 실행하는 일회성 작업으로 취급된다.

이 접근법의 문제는 성능 회귀가 누적된 후에야 발견된다는 것이다. 작은 변경들이 쌓여 수 주 후에 응답 시간이 2배가 되는 상황이 발생한다.

CI 파이프라인에 성능 테스트를 통합하면 성능 회귀를 커밋 단위로 감지할 수 있다.

### Gatling으로 성능 테스트 시나리오 작성

Gatling은 Scala DSL을 사용하는 고성능 부하 테스트 도구다.

JVM 기반이어서 Java/Spring 프로젝트와 통합이 용이하고, HTML 리포트를 자동 생성한다.

```scala
// src/gatling/simulations/OrderSimulation.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class OrderSimulation extends Simulation {

  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")

  // 인증 토큰 획득
  val loginFeeder = csv("test-users.csv").circular

  val authScenario = scenario("인증")
    .feed(loginFeeder)
    .exec(
      http("로그인")
        .post("/api/auth/login")
        .body(StringBody("""{"email": "${email}", "password": "${password}"}"""))
        .check(status.is(200))
        .check(jsonPath("$.token").saveAs("authToken"))
    )

  val orderScenario = scenario("주문 생성 시나리오")
    .exec(authScenario)
    .pause(1)
    .exec(
      http("상품 목록 조회")
        .get("/api/products?category=electronics")
        .header("Authorization", "Bearer #{authToken}")
        .check(status.is(200))
        .check(jsonPath("$.content[0].id").saveAs("productId"))
    )
    .pause(500.milliseconds, 2.seconds)
    .exec(
      http("주문 생성")
        .post("/api/orders")
        .header("Authorization", "Bearer #{authToken}")
        .body(StringBody(
          """{"items": [{"productId": "#{productId}", "quantity": 1}]}"""
        ))
        .check(status.is(201))
        .check(jsonPath("$.orderId").saveAs("orderId"))
    )
    .exec(
      http("주문 상태 조회")
        .get("/api/orders/#{orderId}")
        .header("Authorization", "Bearer #{authToken}")
        .check(status.is(200))
    )

  // 부하 패턴 정의
  setUp(
    orderScenario.inject(
      // 워밍업: 10초 동안 10명까지 점진적 증가
      rampUsers(10).during(10.seconds),
      // 정상 부하: 50 RPS를 60초간 유지
      constantUsersPerSec(50).during(60.seconds),
      // 피크 부하: 100 RPS를 30초간 유지
      constantUsersPerSec(100).during(30.seconds)
    )
  ).protocols(httpProtocol)
   .assertions(
     // 성능 기준 (SLA)
     global.responseTime.percentile(95).lte(500),    // P95 < 500ms
     global.responseTime.percentile(99).lte(1000),   // P99 < 1000ms
     global.successfulRequests.percent.gte(99.5),    // 성공률 > 99.5%
     forAll.failedRequests.count.lte(10)              // 전체 실패 < 10건
   )
}
```

`assertions` 블록이 핵심이다. SLA(Service Level Agreement) 기준을 코드로 정의하고, 기준을 초과하면 Gatling이 exit code 1을 반환해 CI 파이프라인을 실패시킨다.

### Maven/Gradle에서 Gatling 플러그인 설정

```xml
<!-- pom.xml -->
<plugin>
    <groupId>io.gatling</groupId>
    <artifactId>gatling-maven-plugin</artifactId>
    <version>4.6.0</version>
    <configuration>
        <simulationClass>OrderSimulation</simulationClass>
        <resultsFolder>${project.build.directory}/gatling-results</resultsFolder>
    </configuration>
</plugin>
```

```kotlin
// build.gradle.kts
plugins {
    id("io.gatling.gradle") version "3.10.3"
}

gatling {
    simulations = listOf("OrderSimulation")
}
```

### GitHub Actions CI 통합

```yaml
# .github/workflows/performance-test.yml
name: Performance Tests

on:
  push:
    branches: [main, release/**]
  schedule:
    - cron: "0 2 * * *"  # 매일 새벽 2시 정기 실행

jobs:
  performance-test:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: "21"
          distribution: "temurin"

      - name: Start test environment
        run: |
          docker compose -f docker-compose.e2e.yml up -d
          # 서비스 준비 대기
          ./scripts/wait-for-health.sh http://localhost:8080/actuator/health 60

      - name: Run Gatling performance tests
        run: ./mvnw gatling:test -Pperformance

      - name: Upload Gatling report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: gatling-report-${{ github.run_id }}
          path: target/gatling-results/
          retention-days: 30

      - name: Parse and comment results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(
              fs.readFileSync('target/gatling-results/stats.json', 'utf8')
            );
            const p95 = results.stats.meanResTs;
            const successRate = results.stats.percentiles4;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 성능 테스트 결과\n- P95 응답시간: ${p95}ms\n- 성공률: ${successRate}%`
            });

      - name: Teardown test environment
        if: always()
        run: docker compose -f docker-compose.e2e.yml down -v
```

PR마다 성능 테스트를 실행하면 CI 시간이 크게 늘어난다.

`main` 브랜치 머지 후, 또는 야간 스케줄로 실행하되 PR에는 결과를 코멘트로 표시하는 방식이 현실적인 균형점이다.

### 성능 기준 베이스라인 관리

```yaml
# .github/workflows/performance-baseline.yml
# 베이스라인 저장 (main 브랜치 머지 시)
- name: Save performance baseline
  if: github.ref == 'refs/heads/main'
  run: |
    cp target/gatling-results/stats.json .performance-baseline.json
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add .performance-baseline.json
    git commit -m "chore: update performance baseline [skip ci]"
    git push

# 성능 회귀 감지 (PR 시)
- name: Check performance regression
  run: |
    BASELINE_P95=$(jq '.p95' .performance-baseline.json)
    CURRENT_P95=$(jq '.p95' target/gatling-results/stats.json)
    REGRESSION=$(echo "$BASELINE_P95 $CURRENT_P95" | awk '{print ($2/$1 - 1) * 100}')

    # 20% 이상 성능 저하 시 실패
    if (( $(echo "$REGRESSION > 20" | bc -l) )); then
      echo "성능 회귀 감지: P95가 ${REGRESSION}% 증가했습니다 (베이스라인: ${BASELINE_P95}ms, 현재: ${CURRENT_P95}ms)"
      exit 1
    fi
```

절대 임계값만으로는 성능 회귀를 놓칠 수 있다.

200ms P95가 기준이라면, 코드 변경 전 100ms였다가 180ms가 되어도 통과한다. 베이스라인 대비 상대적 변화를 감지하는 것이 더 엄격하다.

### k6를 활용한 경량 성능 테스트

Gatling 대신 k6를 사용하면 JavaScript로 간결하게 부하 테스트를 작성할 수 있다.

```javascript
// k6/order-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const orderDuration = new Trend('order_creation_duration');

export const options = {
  stages: [
    { duration: '10s', target: 10 },   // 워밍업
    { duration: '60s', target: 50 },   // 정상 부하
    { duration: '30s', target: 100 },  // 피크 부하
    { duration: '10s', target: 0 },    // 쿨다운
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.005'],
    errors: ['rate<0.01'],
  },
};

export default function () {
  // 로그인
  const loginRes = http.post(
    `${__ENV.BASE_URL}/api/auth/login`,
    JSON.stringify({ email: 'test@example.com', password: 'Test1234!' }),
    { headers: { 'Content-Type': 'application/json' } }
  );

  check(loginRes, { '로그인 성공': (r) => r.status === 200 });
  const token = loginRes.json('token');

  sleep(0.5);

  // 주문 생성
  const start = Date.now();
  const orderRes = http.post(
    `${__ENV.BASE_URL}/api/orders`,
    JSON.stringify({ items: [{ productId: 'prod-001', quantity: 1 }] }),
    { headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' } }
  );

  orderDuration.add(Date.now() - start);
  errorRate.add(orderRes.status !== 201);

  check(orderRes, {
    '주문 생성 성공': (r) => r.status === 201,
    '주문 ID 존재': (r) => r.json('orderId') !== undefined,
  });

  sleep(1);
}
```

```bash
# CI에서 k6 실행
k6 run \
  --env BASE_URL=http://localhost:8080 \
  --out json=results.json \
  k6/order-load-test.js
```

k6는 단일 바이너리로 배포가 간단하고, Docker 이미지도 제공한다.

GitHub Actions에서 `grafana/k6-action`을 사용하면 별도 설치 없이 바로 실행할 수 있다.

---

## 요약

E2E 테스트는 핵심 비즈니스 흐름을 검증하는 마지막 안전망이지만, 그 비용을 인식하고 선택적으로 적용해야 한다.

계약 테스트(Pact)는 마이크로서비스 간 인터페이스 호환성을 빠르고 독립적으로 검증하는 수단으로, E2E 테스트의 부담을 크게 줄인다.

성능 테스트를 CI에 통합하면 성능 회귀를 코드 변경 단위로 조기에 감지하고, 베이스라인 비교를 통해 점진적 저하를 막을 수 있다.

세 가지 전략을 적절히 조합하면, 빠른 피드백과 높은 신뢰성을 동시에 달성하는 테스트 파이프라인을 구축할 수 있다.

---

## 참고 자료

1. [Pact Documentation — Consumer-Driven Contract Testing](https://docs.pact.io/)
2. [Gatling Documentation — Simulation Setup](https://docs.gatling.io/reference/script/core/simulation/)
3. [k6 Documentation — Load Testing Best Practices](https://grafana.com/docs/k6/latest/testing-guides/load-testing-websites/)
4. [Martin Fowler — Contract Tests](https://martinfowler.com/bliki/ContractTest.html)
5. [Google Testing Blog — Just Say No to More End-to-End Tests](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html)
6. [Awaitility — Java DSL for Asynchronous Testing](https://github.com/awaitility/awaitility)
