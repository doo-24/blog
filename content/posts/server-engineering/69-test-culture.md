---
title: "[테스트 전략] 4편 — 테스트 문화와 전략: 팀으로 테스트하기"
date: 2026-03-17T16:01:00+09:00
draft: false
tags: ["테스트 피라미드", "TDD", "JaCoCo", "커버리지", "서버"]
series: ["테스트 전략"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 10
summary: "테스트 피라미드 vs 테스트 트로피와 현실적 비율, JaCoCo 커버리지 측정과 커버리지 맹신의 위험, TDD 사이클 실전과 레거시 코드에 테스트 추가하기, Flaky 테스트 원인과 CI 연동까지"
---

테스트 코드는 쓰는 사람이 따로 있고 안 쓰는 사람이 따로 있는 게 아니다. 팀 전체가 함께 짜야만 의미가 있다. 테스트가 없으면 배포가 두렵고, 테스트가 있어도 전략 없이 쌓이면 빌드가 10분을 넘고 Flaky 테스트가 CI를 마비시킨다. 이번 편에서는 테스트 피라미드부터 TDD 사이클, JaCoCo 커버리지, Flaky 테스트 관리까지 — 팀 단위로 실제로 굴러가는 테스트 전략 전반을 다룬다.

---

## 1. 테스트 피라미드 vs 테스트 트로피

### 1-1. 고전적 테스트 피라미드

Mike Cohn이 제시한 테스트 피라미드는 세 층으로 나뉜다.

- **Unit Test (단위 테스트)**: 가장 많고, 가장 빠르다.
- **Integration Test (통합 테스트)**: 중간 비율.
- **E2E / UI Test**: 가장 적고, 가장 느리다.

피라미드의 핵심 논리는 단순하다. 위로 갈수록 느리고 비싸고 깨지기 쉬우니 아래에 더 투자하라는 것이다.

```
        /\
       /E2E\
      /------\
     / Integ  \
    /----------\
   /   Unit     \
  /--------------\
```

이 구조는 서버 사이드 백엔드, 특히 도메인 로직이 복잡한 시스템에서 여전히 유효하다.

### 1-2. 테스트 트로피 (Testing Trophy)

Kent C. Dodds가 제안한 테스트 트로피는 구조가 다르다.

```
         ___
        /E2E \
       /-------\
      / Integ   \
     /___________\
    /  Unit       \
   /_______________\
  /  Static/Lint    \
 /___________________\
```

트로피에서는 통합 테스트 비중이 가장 크다.

"사용자가 실제로 쓰는 방식에 가깝게 테스트하라"는 철학에서 출발한다.

프론트엔드나 BFF(Backend for Frontend) 레이어에는 트로피가 더 잘 맞는다. 반면 순수 도메인 로직이나 알고리즘이 핵심인 서버 백엔드는 피라미드가 더 실용적이다.

### 1-3. 현실적 비율과 선택 기준

이론보다 중요한 건 팀의 현실이다.

| 레이어 | 피라미드 권장 비율 | 일반적 현실 | 권장 목표 |
|--------|-------------------|-------------|-----------|
| Unit   | 70%               | 20~30%      | 60%       |
| Integration | 20%          | 60~70%      | 30%       |
| E2E    | 10%               | 10~20%      | 10%       |

통합 테스트가 과도하게 많은 팀은 대개 단위 테스트가 어렵게 설계된 코드를 갖고 있다.

클래스 간 의존성이 강하게 결합되어 있으면 단위 테스트보다 통합 테스트가 쉬워 보이기 때문이다.

### 1-4. 피라미드 역전 — 안티패턴

```
  /----Unit----\         (거의 없음)
 /-Integration--\        (조금)
/------E2E-------\       (대부분)
```

이 형태는 "아이스크림 콘"이라 불리며 최악의 패턴이다.

빌드 시간이 30분을 넘고, 어디서 깨졌는지 추적하기 어렵다.

E2E 테스트 하나가 수십 개의 컴포넌트를 관통하기 때문에 실패 원인이 불명확하다.

### 1-5. 서비스 유형별 전략 선택

| 서비스 유형 | 추천 전략 |
|-------------|-----------|
| 도메인 로직 중심 (결제, 재고, 정산) | 피라미드 - Unit 비중 높게 |
| REST API Gateway, BFF | 트로피 - Integration 비중 높게 |
| 이벤트 기반 MSA | Integration + Contract Test 강화 |
| 데이터 파이프라인 | 데이터 품질 테스트 + Integration |

---

## 2. 커버리지 측정과 커버리지 맹신의 위험

### 2-1. JaCoCo 기본 설정

JaCoCo(Java Code Coverage)는 Java/Kotlin 프로젝트에서 가장 널리 쓰이는 커버리지 도구다.

**`build.gradle` 설정:**

```groovy
plugins {
    id 'java'
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.11"
}

test {
    useJUnitPlatform()
    finalizedBy jacocoTestReport
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true      // SonarQube 연동용
        html.required = true     // 로컬 리포트
        csv.required = false
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.70   // 전체 라인 커버리지 70% 미만이면 빌드 실패
            }
        }
        rule {
            element = 'CLASS'
            excludes = [
                'com.example.config.*',
                'com.example.dto.*',
                'com.example.*Application'
            ]
            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.60
            }
        }
    }
}

check.dependsOn jacocoTestCoverageVerification
```

커버리지 검증을 `check` 태스크에 연결하면 `./gradlew build` 시 자동으로 적용된다.

**Kotlin + build.gradle.kts 버전:**

```kotlin
plugins {
    kotlin("jvm") version "1.9.22"
    id("jacoco")
}

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.70".toBigDecimal()
            }
        }
    }
}

tasks.check {
    dependsOn(tasks.jacocoTestCoverageVerification)
}
```

### 2-2. 커버리지 리포트 읽는 법

JaCoCo가 측정하는 지표는 여러 종류다.

| 지표 | 설명 | 중요도 |
|------|------|--------|
| Line Coverage | 실행된 라인 수 / 전체 라인 수 | 높음 |
| Branch Coverage | 분기(if/else/switch) 커버 비율 | 매우 높음 |
| Method Coverage | 호출된 메서드 비율 | 중간 |
| Class Coverage | 테스트가 닿은 클래스 비율 | 낮음 |

Branch Coverage가 가장 의미 있다.

라인 커버리지 100%여도 `if (a && b)` 에서 `a=true, b=false` 분기를 한 번도 테스트 안 했을 수 있다.

### 2-3. 커버리지 맹신의 위험 — 안티패턴

```java
// 안티패턴: assertion 없는 커버리지 올리기
@Test
void testProcessOrder() {
    // 이 테스트는 라인 커버리지를 올리지만 아무것도 검증하지 않는다
    orderService.processOrder(new Order(1L, 100));
    // assertTrue, assertEquals 등이 없음
}
```

이런 테스트는 커버리지 숫자를 올리지만 버그를 잡지 못한다.

실제로 운영에서 `NullPointerException`이 터졌는데 커버리지 85%인 코드베이스에서 발생한 사례가 흔하다.

**커버리지 목표 설정 시 흔한 실수:**

```groovy
// 잘못된 접근: 숫자 달성이 목표가 됨
// 팀이 의미 없는 테스트를 양산하게 된다
jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.90  // 너무 높은 목표
            }
        }
    }
}
```

커버리지 90% 목표를 설정하면 팀이 숫자를 맞추는 데만 집중한다.

의미 없는 getter/setter 테스트, assertion 없는 실행 테스트가 늘어난다.

### 2-4. 현실적인 커버리지 전략

커버리지는 목표가 아니라 지표다.

낮은 커버리지는 "테스트가 부족한 영역이 있다"는 신호일 뿐, 높은 커버리지가 품질을 보장하지는 않는다.

**권장 접근법:**

1. **중요 도메인 로직에 집중**: 결제 처리, 재고 계산, 할인 로직 — 이런 곳은 Branch Coverage 80% 이상 유지.
2. **설정, DTO, 단순 엔티티는 제외**: 테스트 가치가 낮다.
3. **커버리지 하락 알림**: 현재 수준에서 5% 이상 하락하면 PR 차단.

```groovy
// 실용적인 제외 목록
jacocoTestReport {
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/dto/**',
                '**/config/**',
                '**/*Application.class',
                '**/exception/**',  // 단순 예외 클래스
                '**/entity/**'      // JPA 엔티티 (대부분 getter/setter)
            ])
        }))
    }
}
```

---

## 3. TDD 사이클 실전

### 3-1. Red-Green-Refactor

TDD의 핵심 사이클은 세 단계다.

1. **Red**: 실패하는 테스트를 먼저 작성한다.
2. **Green**: 테스트를 통과하는 최소한의 코드를 작성한다.
3. **Refactor**: 코드를 정리한다. 테스트는 여전히 통과해야 한다.

이 사이클의 핵심은 "테스트가 먼저"가 아니라 "작은 단계로"다.

한 번에 큰 기능을 구현하려 하면 TDD가 오히려 방해가 된다.

### 3-2. TDD 실전 예시 — 할인 정책

**1단계: Red — 실패하는 테스트 작성**

```java
@Test
void 회원등급이_GOLD이면_10퍼센트_할인이_적용된다() {
    // given
    DiscountPolicy policy = new MemberDiscountPolicy();
    Member member = new Member(1L, "홍길동", Grade.GOLD);
    int price = 10000;

    // when
    int discount = policy.discount(member, price);

    // then
    assertThat(discount).isEqualTo(1000);
}
```

이 시점에서 `MemberDiscountPolicy` 클래스가 없으므로 컴파일 실패다.

컴파일 실패도 Red 상태다. 클래스를 만들어야 한다.

**2단계: Green — 최소한의 구현**

```java
public class MemberDiscountPolicy implements DiscountPolicy {

    @Override
    public int discount(Member member, int price) {
        if (member.getGrade() == Grade.GOLD) {
            return price / 10;
        }
        return 0;
    }
}
```

테스트가 통과한다. 코드가 예쁘지 않아도 된다.

지금은 동작하게 만드는 것만 목표다.

**3단계: 다음 케이스 추가 — 다시 Red**

```java
@Test
void 회원등급이_VIP이면_20퍼센트_할인이_적용된다() {
    DiscountPolicy policy = new MemberDiscountPolicy();
    Member member = new Member(2L, "이순신", Grade.VIP);
    int price = 10000;

    int discount = policy.discount(member, price);

    assertThat(discount).isEqualTo(2000);
}

@Test
void 일반_회원은_할인이_없다() {
    DiscountPolicy policy = new MemberDiscountPolicy();
    Member member = new Member(3L, "김철수", Grade.BASIC);
    int price = 10000;

    int discount = policy.discount(member, price);

    assertThat(discount).isEqualTo(0);
}
```

**4단계: Green 유지하며 구현 확장**

```java
public class MemberDiscountPolicy implements DiscountPolicy {

    private static final Map<Grade, Integer> DISCOUNT_RATES = Map.of(
        Grade.GOLD, 10,
        Grade.VIP, 20,
        Grade.BASIC, 0
    );

    @Override
    public int discount(Member member, int price) {
        int rate = DISCOUNT_RATES.getOrDefault(member.getGrade(), 0);
        return price * rate / 100;
    }
}
```

**5단계: Refactor**

테스트가 모두 통과하는 상태에서 코드를 정리한다.

중복을 제거하고, 명확하지 않은 로직을 리팩터링한다.

### 3-3. TDD가 어려운 상황

TDD는 모든 상황에 맞지 않는다.

- **탐색적 코딩**: 아직 무엇을 만들지 모를 때. 먼저 프로토타입을 만들고 나중에 테스트를 추가하는 게 현실적이다.
- **인프라 코드**: DB 마이그레이션, 배포 스크립트는 TDD가 어렵다.
- **외부 API 연동 초기**: API 스펙이 불분명할 때는 탐색이 먼저다.

TDD를 강요하면 오히려 팀이 테스트 자체를 기피하게 된다.

"중요한 도메인 로직만큼은 TDD로"라는 부분적 적용이 현실적이다.

---

## 4. 레거시 코드에 테스트 추가하기

### 4-1. 레거시 코드의 특징

레거시 코드의 정의는 "테스트가 없는 코드"다 (Michael Feathers, "Working Effectively with Legacy Code").

단순히 오래된 코드가 아니라, 변경하기 두려운 코드다.

레거시 코드에 테스트를 추가하는 건 쉽지 않다. 의존성이 얽혀 있고, 생성자에서 DB 연결을 하거나, `static` 메서드가 곳곳에 박혀 있다.

### 4-2. Seam 찾기

Michael Feathers의 핵심 개념은 "Seam(솔기)"이다.

Seam이란 코드를 변경하지 않고 동작을 바꿀 수 있는 지점이다.

```java
// 레거시 코드 — 테스트하기 어려운 형태
public class OrderService {

    public void processOrder(Long orderId) {
        // DB 직접 접근
        Connection conn = DriverManager.getConnection("jdbc:mysql://...");
        PreparedStatement ps = conn.prepareStatement("SELECT * FROM orders WHERE id = ?");
        ps.setLong(1, orderId);
        ResultSet rs = ps.executeQuery();

        // 외부 API 직접 호출
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://payment.api/charge"))
            .build();
        // ...
    }
}
```

이 코드는 테스트를 위해 실제 DB와 실제 외부 API가 필요하다.

Seam을 만들려면 의존성을 추출해야 한다.

### 4-3. 의존성 추출 — Seam 만들기

```java
// Step 1: 인터페이스 추출
public interface OrderRepository {
    Order findById(Long orderId);
}

public interface PaymentGateway {
    PaymentResult charge(Order order);
}

// Step 2: 레거시 구현체 래핑
public class JdbcOrderRepository implements OrderRepository {
    @Override
    public Order findById(Long orderId) {
        // 기존 JDBC 코드
    }
}

// Step 3: OrderService를 의존성 주입 방식으로 전환
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    // 기존 코드와 호환을 위한 기본 생성자 유지 (점진적 전환)
    public OrderService() {
        this(new JdbcOrderRepository(), new HttpPaymentGateway());
    }

    // 테스트 가능한 생성자 추가
    public OrderService(OrderRepository orderRepository, PaymentGateway paymentGateway) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }

    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        paymentGateway.charge(order);
    }
}
```

이제 테스트에서 Mock을 주입할 수 있다.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentGateway paymentGateway;

    @InjectMocks
    private OrderService orderService;

    @Test
    void 주문을_처리하면_결제가_호출된다() {
        // given
        Long orderId = 1L;
        Order order = new Order(orderId, 10000);
        when(orderRepository.findById(orderId)).thenReturn(order);
        when(paymentGateway.charge(order)).thenReturn(PaymentResult.success());

        // when
        orderService.processOrder(orderId);

        // then
        verify(paymentGateway).charge(order);
    }
}
```

### 4-4. Characterization Test — 동작 명세화

레거시 코드의 현재 동작을 먼저 테스트로 고정한다.

변경이 목적이 아니라 현재 동작을 문서화하는 테스트다.

```java
@Test
void characterization_할인_계산_현재_동작() {
    // 이 테스트는 현재 동작이 "맞다"고 가정하지 않는다
    // 현재 동작을 기록하는 것이 목적이다
    LegacyDiscountCalculator calc = new LegacyDiscountCalculator();

    // 현재 어떤 값이 나오는지 기록
    int result = calc.calculate(10000, "GOLD");

    // 현재 동작이 1000이라면 — 나중에 이것이 맞는지 검토 필요
    assertThat(result).isEqualTo(1000);
}
```

이 테스트를 기반으로 리팩터링 시 동작이 변하면 즉시 알 수 있다.

### 4-5. 레거시 테스트 추가 전략

무턱대고 모든 레거시 코드에 테스트를 추가하려 하면 지친다.

우선순위를 정해야 한다.

1. **자주 변경되는 코드**: 수정 빈도가 높은 코드부터.
2. **버그가 많이 나는 코드**: 히스토리를 보고 결함 밀도가 높은 모듈부터.
3. **비즈니스 크리티컬**: 결제, 인증, 데이터 변환 로직.

---

## 5. Flaky 테스트 원인과 관리

### 5-1. Flaky 테스트란

Flaky 테스트는 코드 변경 없이 실행 결과가 달라지는 테스트다.

같은 커밋에서 첫 번째 실행은 통과하고, 두 번째 실행은 실패한다.

CI에서 가장 위험한 존재다. 팀이 빨간 빌드에 무감각해지기 때문이다.

### 5-2. Flaky 테스트의 주요 원인

**원인 1: 시간 의존성**

```java
// 안티패턴: System.currentTimeMillis() 직접 사용
@Test
void 오늘_생성된_주문은_당일_처리된다() {
    Order order = new Order();
    order.setCreatedAt(LocalDateTime.now()); // 테스트 실행 시점마다 다름

    boolean result = orderService.isProcessableToday(order);

    assertTrue(result); // 자정 근처에서 실패할 수 있음
}
```

```java
// 수정: Clock 주입으로 시간 제어
public class OrderService {
    private final Clock clock;

    public OrderService(Clock clock) {
        this.clock = clock;
    }

    public boolean isProcessableToday(Order order) {
        LocalDate today = LocalDate.now(clock);
        return order.getCreatedAt().toLocalDate().equals(today);
    }
}

@Test
void 오늘_생성된_주문은_당일_처리된다() {
    Clock fixedClock = Clock.fixed(
        Instant.parse("2026-03-17T10:00:00Z"),
        ZoneId.of("Asia/Seoul")
    );
    OrderService service = new OrderService(fixedClock);
    Order order = new Order();
    order.setCreatedAt(LocalDateTime.now(fixedClock));

    boolean result = service.isProcessableToday(order);

    assertTrue(result);
}
```

**원인 2: 테스트 간 상태 공유**

```java
// 안티패턴: static 상태 공유
class UserServiceTest {
    private static List<User> users = new ArrayList<>(); // 공유 상태

    @Test
    void 사용자_추가() {
        users.add(new User("홍길동"));
        assertThat(users).hasSize(1);
    }

    @Test
    void 사용자_목록_조회() {
        // 이전 테스트가 먼저 실행되면 size가 1, 아니면 0
        assertThat(users).isEmpty(); // 실행 순서에 따라 결과가 다름
    }
}
```

```java
// 수정: 각 테스트마다 독립된 상태
class UserServiceTest {

    private List<User> users;

    @BeforeEach
    void setUp() {
        users = new ArrayList<>(); // 매 테스트마다 새로 초기화
    }
}
```

**원인 3: 비동기 처리**

```java
// 안티패턴: 충분하지 않은 대기
@Test
void 이벤트_발행_후_리스너가_처리된다() throws InterruptedException {
    eventPublisher.publish(new OrderCreatedEvent(1L));
    Thread.sleep(100); // 100ms면 충분하다고 가정 — 환경에 따라 다름
    verify(eventListener).onOrderCreated(any());
}
```

```java
// 수정: Awaitility로 조건 기반 대기
@Test
void 이벤트_발행_후_리스너가_처리된다() {
    eventPublisher.publish(new OrderCreatedEvent(1L));

    await()
        .atMost(5, SECONDS)
        .pollInterval(100, MILLISECONDS)
        .untilAsserted(() ->
            verify(eventListener).onOrderCreated(any())
        );
}
```

**원인 4: 외부 서비스 의존**

```java
// 안티패턴: 실제 외부 API 호출
@Test
void 날씨_API_정상_응답() {
    WeatherResponse response = weatherClient.getWeather("Seoul");
    assertThat(response.getTemperature()).isNotNull();
    // 외부 API 장애 시 실패
}
```

```java
// 수정: WireMock으로 HTTP 스터빙
@ExtendWith(WireMockExtension.class)
class WeatherClientTest {

    @WireMock
    private WireMockServer wireMockServer;

    @Test
    void 날씨_API_정상_응답() {
        wireMockServer.stubFor(get(urlEqualTo("/weather?city=Seoul"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("{\"temperature\": 15, \"city\": \"Seoul\"}")));

        WeatherResponse response = weatherClient.getWeather("Seoul");

        assertThat(response.getTemperature()).isEqualTo(15);
    }
}
```

**원인 5: 랜덤/순서 의존성**

```java
// 안티패턴: 컬렉션 순서에 의존
@Test
void 사용자_목록_조회() {
    List<User> users = userRepository.findAll();
    assertThat(users.get(0).getName()).isEqualTo("홍길동");
    // DB 반환 순서는 보장되지 않음
}
```

```java
// 수정: 순서 독립적으로 검증
@Test
void 사용자_목록_조회() {
    List<User> users = userRepository.findAll();
    assertThat(users)
        .extracting(User::getName)
        .containsExactlyInAnyOrder("홍길동", "이순신", "김유신");
}
```

### 5-3. Flaky 테스트 탐지와 격리

CI에서 Flaky 테스트를 탐지하는 방법은 재실행 통계를 수집하는 것이다.

```yaml
# GitHub Actions — 실패 시 재시도로 Flaky 탐지
- name: Run tests with retry
  run: |
    ./gradlew test || \
    ./gradlew test || \
    ./gradlew test
  # 3회 중 1회라도 통과하면 Flaky 테스트 존재 가능성
```

더 정교한 방법은 JUnit 5의 `@RepeatedTest`나 별도 Flaky 추적 도구를 쓰는 것이다.

```java
// 특정 테스트의 Flaky 여부 조사
@RepeatedTest(10)
void Flaky_의심_테스트_반복_실행(RepetitionInfo info) {
    // 10번 실행하여 실패 여부 확인
    orderService.processOrder(1L);
    // assertions...
}
```

Flaky 테스트가 확인되면 즉시 `@Disabled`로 격리하고 티켓을 생성한다.

방치하면 팀 전체가 빨간 CI에 무감각해지는 더 큰 문제가 생긴다.

---

## 6. CI 파이프라인 연동

### 6-1. GitHub Actions 기본 파이프라인

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: testdb
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      - name: Run unit tests
        run: ./gradlew test

      - name: Run integration tests
        run: ./gradlew integrationTest
        env:
          SPRING_DATASOURCE_URL: jdbc:mysql://localhost:3306/testdb
          SPRING_DATASOURCE_USERNAME: root
          SPRING_DATASOURCE_PASSWORD: test
          SPRING_REDIS_HOST: localhost
          SPRING_REDIS_PORT: 6379

      - name: Generate coverage report
        run: ./gradlew jacocoTestReport

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: ./build/reports/jacoco/test/jacocoTestReport.xml
          fail_ci_if_error: false

      - name: Check coverage threshold
        run: ./gradlew jacocoTestCoverageVerification

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: |
            build/reports/tests/
            build/reports/jacoco/
```

### 6-2. 빌드 단계 분리 — 빠른 피드백

단위 테스트와 통합 테스트를 분리하면 빠른 피드백이 가능하다.

```yaml
jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - name: Unit tests (fast)
        run: ./gradlew test -x integrationTest
        # 보통 1~2분 이내

  integration-test:
    runs-on: ubuntu-latest
    needs: unit-test  # 단위 테스트 통과 후 실행
    services:
      mysql:
        image: mysql:8.0
        # ...
    steps:
      - uses: actions/checkout@v4
      - name: Integration tests
        run: ./gradlew integrationTest
        # 5~10분 소요
```

단위 테스트가 실패하면 통합 테스트는 실행하지 않는다.

가장 빠른 피드백이 먼저 오도록 설계한다.

### 6-3. Gradle 테스트 소스셋 분리

```groovy
// build.gradle
sourceSets {
    integrationTest {
        java {
            compileClasspath += main.output + test.output
            runtimeClasspath += main.output + test.output
            srcDir file('src/integration-test/java')
        }
        resources.srcDir file('src/integration-test/resources')
    }
}

configurations {
    integrationTestImplementation.extendsFrom testImplementation
    integrationTestRuntimeOnly.extendsFrom testRuntimeOnly
}

task integrationTest(type: Test) {
    description = 'Runs integration tests.'
    group = 'verification'

    testClassesDirs = sourceSets.integrationTest.output.classesDirs
    classpath = sourceSets.integrationTest.runtimeClasspath

    useJUnitPlatform()
    shouldRunAfter test

    testLogging {
        events "passed", "skipped", "failed"
    }
}
```

이렇게 분리하면 `./gradlew test`는 단위 테스트만, `./gradlew integrationTest`는 통합 테스트만 실행한다.

### 6-4. PR 게이트 — 테스트 통과 없이 머지 불가

GitHub Branch Protection Rules와 연동하면 테스트 통과가 머지 조건이 된다.

```yaml
# .github/workflows/pr-check.yml
name: PR Check

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  required-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Run all tests
        run: ./gradlew check  # test + jacocoTestCoverageVerification 포함

      - name: Comment coverage on PR
        uses: madrapps/jacoco-report@v1.6.1
        with:
          paths: ${{ github.workspace }}/build/reports/jacoco/test/jacocoTestReport.xml
          token: ${{ secrets.GITHUB_TOKEN }}
          min-coverage-overall: 70
          min-coverage-changed-files: 80  # 변경된 파일은 더 높은 기준
```

변경된 파일에 더 높은 커버리지 기준을 적용하는 것이 실용적이다.

기존 레거시 코드의 낮은 커버리지로 인해 전체 기준을 낮게 설정해야 하는 문제를 우회한다.

### 6-5. 테스트 병렬화로 속도 개선

JUnit 5는 테스트 병렬 실행을 지원한다.

```properties
# src/test/resources/junit-platform.properties
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.mode.classes.default = concurrent
junit.jupiter.execution.parallel.config.strategy = dynamic
junit.jupiter.execution.parallel.config.dynamic.factor = 2
```

단, 병렬 실행은 테스트 간 상태 공유가 없을 때만 안전하다.

`@TestMethodOrder`와 상태를 공유하는 테스트는 `@Execution(SAME_THREAD)`로 직렬 실행을 강제해야 한다.

```java
@SpringBootTest
@Execution(ExecutionMode.SAME_THREAD) // DB 상태를 공유하는 통합 테스트
class OrderIntegrationTest {
    // 순서 의존적인 테스트들
}
```

---

## 7. 테스트 문화 정착을 위한 팀 전략

### 7-1. 테스트 리뷰를 코드 리뷰의 일부로

PR 리뷰 시 테스트 코드도 반드시 검토한다.

체크리스트 예시:
- assertion이 실제로 의미 있는가?
- 경계값, 예외 케이스가 포함되어 있는가?
- 테스트 이름이 의도를 명확하게 설명하는가?
- Mock이 과도하게 사용되지 않았는가?

### 7-2. 테스트 작성 비용 줄이기

테스트가 어렵다고 느끼는 이유는 대부분 설계 문제다.

테스트하기 어려운 코드는 변경하기도 어렵다.

테스트 작성이 힘들다면 설계를 개선하는 신호로 받아들여야 한다.

### 7-3. 실패 허용 문화

테스트가 실패할 때 비난하지 않는 문화가 중요하다.

테스트 실패는 버그를 일찍 발견한 성공이다.

운영 배포 후 발견된 버그보다 테스트에서 잡힌 버그가 훨씬 저렴하다.

---

팀의 테스트 전략은 한 번에 완성되지 않는다. 피라미드 비율을 맞추고, 커버리지 숫자를 쫓고, TDD를 강요하는 대신 — 지금 가장 아프게 느끼는 문제 하나부터 해결하는 것이 현실적이다. Flaky 테스트 하나를 잡거나, 레거시 코드 한 클래스에 Characterization Test를 추가하는 작은 시작이 팀 전체의 테스트 문화를 바꾼다.

---

## 참고 자료

- Michael Feathers, *Working Effectively with Legacy Code* (Prentice Hall, 2004) — 레거시 코드 테스트 전략의 바이블
- Kent Beck, *Test-Driven Development: By Example* (Addison-Wesley, 2002) — TDD 원조 서적
- [JaCoCo 공식 문서](https://www.jacoco.org/jacoco/trunk/doc/) — 커버리지 측정 설정 레퍼런스
- [JUnit 5 User Guide — Parallel Execution](https://junit.org/junit5/docs/current/user-guide/#writing-tests-parallel-execution) — 테스트 병렬화 공식 가이드
- [Awaitility GitHub](https://github.com/awaitility/awaitility) — 비동기 테스트 대기 라이브러리
- Martin Fowler, ["TestPyramid"](https://martinfowler.com/bliki/TestPyramid.html) — 테스트 피라미드 원문
