---
title: "[서버 애플리케이션 설계] 7편 — 모듈 시스템 설계: 큰 코드베이스를 다루는 법"
date: 2026-03-18T00:03:00+09:00
draft: false
tags: ["패키지 구조", "모듈", "멀티모듈", "아키텍처", "서버"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "패키지 구조 전략(계층별 vs 도메인별 vs 기능별), 모듈 경계 정의, 공개 API와 내부 구현 분리, Gradle 멀티모듈 프로젝트 설계, 모놀리스에서 MSA 전환 준비까지"
---

코드베이스가 커지는 순간부터 '어디에 뭘 넣어야 하는가'라는 질문이 팀의 일상적인 갈등이 된다. `UserService`가 `OrderService`를 직접 호출해도 되는지, `repository` 패키지를 `domain` 아래에 둬야 하는지 `infrastructure` 아래에 둬야 하는지, 신규 기능을 기존 모듈에 추가해야 하는지 새 모듈로 분리해야 하는지. 이 질문들에 명확한 답을 내리지 못하는 팀은 결국 '빅볼오브머드(Big Ball of Mud)'를 만들어낸다. 그 안에서는 어떤 코드가 어떤 코드에 의존하는지 아무도 모르고, 한 곳을 고치면 예상치 못한 곳이 망가진다. 이 글은 모듈 시스템을 설계하는 구체적인 방법을 다룬다. 패키지 구조 전략부터 멀티모듈 프로젝트, 그리고 MSA 전환을 대비한 모놀리스 설계까지 실제로 적용 가능한 수준으로 파고든다.

---

## 1. 패키지 구조 전략: 세 가지 철학

패키지는 단순한 폴더 구조가 아니다. 패키지는 팀의 사고 방식을 코드에 투영한 결과물이다. 같은 기능을 구현하더라도 패키지를 어떻게 구성하느냐에 따라 코드를 이해하는 방식, 변경하는 방식, 팀이 협업하는 방식이 완전히 달라진다.

### 1.1 계층별 구조 (Package by Layer)

가장 흔하게 볼 수 있는 구조다. Spring 프레임워크를 처음 배울 때 자연스럽게 따라가게 되는 패턴이기도 하다.

```
com.example.shop
├── controller
│   ├── UserController.java
│   ├── OrderController.java
│   └── ProductController.java
├── service
│   ├── UserService.java
│   ├── OrderService.java
│   └── ProductService.java
├── repository
│   ├── UserRepository.java
│   ├── OrderRepository.java
│   └── ProductRepository.java
└── domain
    ├── User.java
    ├── Order.java
    └── Product.java
```

직관적이다. 새로운 개발자가 합류했을 때 "컨트롤러는 여기, 서비스는 저기"라고 설명하기 쉽다. 그런데 프로젝트가 커지기 시작하면 문제가 드러난다.

**계층별 구조의 문제점**

첫째, `OrderService`가 무슨 일을 하는지 알려면 `controller`, `service`, `repository`, `domain` 패키지를 전부 뒤져야 한다. 기능 하나를 파악하는 데 파일이 네 군데에 흩어져 있다.

둘째, 패키지가 기술적인 역할로만 분류되어 있기 때문에 도메인 경계를 강제할 수 없다. `UserService`가 `OrderRepository`를 직접 참조하는 것을 막을 방법이 없다. 접근 제한자를 써도 모두 같은 패키지 레벨에 있으니 `package-private`으로 격리할 수도 없다.

셋째, 팀이 특정 도메인을 담당하더라도 코드가 여러 패키지에 분산되어 있어 PR 충돌이 자주 발생한다.

### 1.2 도메인별 구조 (Package by Domain)

도메인을 최상위 단위로 삼고, 각 도메인 안에서 계층을 나눈다.

```
com.example.shop
├── user
│   ├── UserController.java
│   ├── UserService.java
│   ├── UserRepository.java
│   └── User.java
├── order
│   ├── OrderController.java
│   ├── OrderService.java
│   ├── OrderRepository.java
│   └── Order.java
└── product
    ├── ProductController.java
    ├── ProductService.java
    ├── ProductRepository.java
    └── Product.java
```

변경 응집도(change cohesion)가 높아진다. 주문 기능을 수정할 때 `order` 패키지만 건드리면 된다. 팀을 도메인 단위로 나누면 각 팀이 자신의 패키지에 집중할 수 있다.

**도메인별 구조의 함정**

도메인 간 의존성이 명시적이지 않으면 결국 서로 뒤엉키기 시작한다. `OrderService`가 `user.UserService`를 직접 import하고, `user.UserService`가 다시 `order.OrderRepository`를 참조하면 순환 의존성 지옥이 펼쳐진다. 패키지 구조만으로는 이를 막을 수 없다.

### 1.3 기능별 구조 (Package by Feature)

기능(Feature) 또는 유스케이스(Use Case) 단위로 패키지를 구성한다. DDD의 Bounded Context나 Clean Architecture의 Use Case 개념과 자연스럽게 맞닿아 있다.

```
com.example.shop
├── user
│   ├── api
│   │   ├── UserController.java
│   │   └── UserRequest.java
│   ├── application
│   │   ├── RegisterUserUseCase.java
│   │   ├── UpdateUserProfileUseCase.java
│   │   └── UserQueryService.java
│   ├── domain
│   │   ├── User.java
│   │   ├── UserRepository.java       ← 인터페이스
│   │   └── UserDomainService.java
│   └── infrastructure
│       ├── UserJpaRepository.java
│       ├── UserRepositoryImpl.java
│       └── UserMapper.java
├── order
│   ├── api
│   ├── application
│   ├── domain
│   └── infrastructure
└── shared
    ├── Money.java
    ├── PageRequest.java
    └── event
        └── DomainEvent.java
```

이 구조의 핵심은 `domain` 패키지 안에 `UserRepository` 인터페이스를 두고, `infrastructure` 패키지에 JPA 구현체를 두는 것이다. 의존성의 방향이 항상 바깥(infrastructure)에서 안(domain)으로 향한다. 이것이 의존성 역전(DIP)을 패키지 구조로 표현한 방식이다.

### 1.4 세 가지 전략의 선택 기준

| 기준 | 계층별 | 도메인별 | 기능별 |
|------|--------|----------|--------|
| 팀 규모 | 소규모 | 중규모 | 중~대규모 |
| 도메인 복잡도 | 낮음 | 중간 | 높음 |
| 경계 강제력 | 없음 | 약함 | 강함 |
| 학습 곡선 | 낮음 | 중간 | 높음 |
| MSA 전환 용이성 | 어려움 | 보통 | 쉬움 |

실용적인 조언을 하나 드리자면: 도메인이 5개 미만이고 팀이 5명 이하라면 계층별 구조도 충분하다. 하지만 그 이상이라면 기능별 구조로 시작하는 것이 장기적으로 이득이다. 나중에 구조를 바꾸는 비용이 처음부터 제대로 설계하는 비용보다 훨씬 크다.

---

## 2. 모듈 경계 정의: 무엇이 안이고 무엇이 밖인가

패키지 구조를 잘 잡았다고 모듈 설계가 끝난 것이 아니다. 가장 중요한 것은 모듈 경계(Module Boundary)를 명확하게 정의하고, 그 경계를 강제하는 메커니즘을 만드는 것이다.

### 2.1 모듈 경계란 무엇인가

모듈 경계는 두 가지로 정의된다.

**공개 API (Public API)**: 외부에서 사용할 수 있는 인터페이스, 클래스, 메서드의 집합이다. 한번 공개된 API는 하위 호환성을 유지해야 한다.

**내부 구현 (Internal Implementation)**: 모듈 내부에서만 사용하는 코드다. 언제든 변경해도 외부에 영향을 주지 않는다.

이 둘을 명확하게 분리하지 못하면 어떻게 되는지 실제 사례를 보자.

```java
// 잘못된 예: 내부 구현이 공개된 경우
public class OrderService {
    // 이 메서드는 내부에서만 사용하려는 의도였지만 public으로 선언됨
    public Order buildOrderFromCartItems(List<CartItem> items, String couponCode) {
        // ...
    }
}

// 다른 팀의 개발자가 이 메서드를 발견하고 사용하기 시작함
public class PromotionService {
    private final OrderService orderService;

    public BigDecimal calculatePromotionDiscount(List<CartItem> items) {
        Order tempOrder = orderService.buildOrderFromCartItems(items, null);
        // 프로모션 계산에 활용
        return tempOrder.getTotalAmount().multiply(BigDecimal.valueOf(0.1));
    }
}
```

6개월 후 `OrderService` 팀이 `buildOrderFromCartItems` 시그니처를 변경해야 할 상황이 생겼다. `CartItem` 대신 `OrderItem`을 받도록 바꾸려 했더니, `PromotionService`가 이 메서드를 사용하고 있어서 함께 수정해야 한다. 이런 상황이 반복되면 개발 속도가 점점 느려진다.

### 2.2 Java에서 경계를 강제하는 방법

**방법 1: 접근 제한자 활용**

같은 패키지 내에서만 공유해야 하는 클래스는 `package-private`(접근 제한자 없음)으로 선언한다.

```java
// order 패키지 내부에서만 사용
class OrderValidator {  // package-private
    boolean validate(Order order) {
        // 검증 로직
    }
}

// 외부에서 사용 가능한 서비스
public class OrderService {
    private final OrderValidator validator = new OrderValidator();

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.create(command);
        if (!validator.validate(order)) {
            throw new InvalidOrderException();
        }
        // ...
    }
}
```

`OrderValidator`는 `order` 패키지 밖에서 접근할 수 없다. 컴파일러가 경계를 강제한다.

**방법 2: 인터페이스로 공개 계약 정의**

```java
// 공개 API — 다른 모듈이 의존할 수 있는 계약
public interface OrderQueryFacade {
    OrderSummary getOrderSummary(OrderId orderId);
    List<OrderSummary> getRecentOrders(UserId userId, int limit);
}

// 내부 구현 — package-private 또는 internal 패키지에 위치
class OrderQueryFacadeImpl implements OrderQueryFacade {
    private final OrderRepository orderRepository;
    private final OrderItemRepository orderItemRepository;

    // 구현 세부사항은 숨겨짐
    @Override
    public OrderSummary getOrderSummary(OrderId orderId) {
        // ...
    }
}
```

외부 모듈은 `OrderQueryFacade` 인터페이스에만 의존한다. 구현체가 바뀌어도 계약만 유지되면 외부에 영향이 없다.

**방법 3: Java 9+ 모듈 시스템 (JPMS)**

Java 9부터 도입된 모듈 시스템은 패키지 수준의 캡슐화를 언어 차원에서 지원한다.

```java
// module-info.java (order 모듈)
module com.example.shop.order {
    requires com.example.shop.common;
    requires spring.context;

    // 외부에 공개할 패키지만 명시
    exports com.example.shop.order.api;
    exports com.example.shop.order.application;

    // 내부 패키지는 exports 목록에 없으므로 외부 접근 불가
    // com.example.shop.order.infrastructure 는 비공개
}
```

현실적으로 Spring Boot와 JPMS를 함께 쓰는 것은 여전히 복잡하다. JPMS보다는 멀티모듈 프로젝트와 ArchUnit을 조합하는 방식이 더 실용적이다.

### 2.3 ArchUnit으로 경계 규칙 테스트하기

코드 리뷰로 경계 위반을 잡는 것은 한계가 있다. ArchUnit은 아키텍처 규칙을 테스트 코드로 작성하고 CI에서 자동으로 검증한다.

```java
@AnalyzeClasses(packages = "com.example.shop")
class ArchitectureRulesTest {

    @ArchTest
    ArchRule domainShouldNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..infrastructure..");

    @ArchTest
    ArchRule applicationShouldNotDependOnApi =
        noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat()
            .resideInAPackage("..api..");

    @ArchTest
    ArchRule orderShouldNotDirectlyAccessUserRepository =
        noClasses()
            .that().resideInAPackage("com.example.shop.order..")
            .should().dependOnClassesThat()
            .resideInAPackage("com.example.shop.user.infrastructure..");
}
```

이 테스트들은 CI 파이프라인에서 실행되며, 규칙을 위반하는 코드가 병합되는 것을 막는다. 아키텍처 규칙이 문서가 아닌 실행 가능한 테스트로 존재하는 것이다.

---

## 3. Gradle 멀티모듈 프로젝트 설계

패키지 구조가 소프트웨어적 경계라면, Gradle 멀티모듈은 물리적 경계다. 모듈 간 의존성을 `build.gradle`에 명시적으로 선언해야 하므로, 실수로 잘못된 의존성을 추가하기 어렵다.

### 3.1 멀티모듈 구조의 장점과 적합한 시점

**장점**:
- 모듈 간 의존성이 `build.gradle`에 명시적으로 드러남
- 각 모듈을 독립적으로 빌드하고 테스트할 수 있음
- 변경된 모듈만 재빌드하므로 빌드 시간 단축 (Gradle 빌드 캐시와 함께)
- 특정 모듈만 다른 팀/프로젝트에서 재사용 가능

**멀티모듈 전환을 고려해야 할 시점**:
- 팀이 3개 이상으로 나뉘어 서로 다른 도메인을 담당할 때
- 특정 모듈(공통 라이브러리, 도메인 모델 등)을 여러 애플리케이션이 공유해야 할 때
- 빌드 시간이 5분을 넘기 시작할 때
- 하나의 배포 단위에서 여러 배포 단위로의 전환을 준비할 때

### 3.2 프로젝트 구조 설계

실제 전자상거래 서비스를 예시로 멀티모듈 구조를 설계해보자.

```
shop-backend/
├── settings.gradle
├── build.gradle (루트)
├── gradle/
│   └── libs.versions.toml
│
├── app-api/                    ← 사용자 facing API 서버
│   └── build.gradle
│
├── app-admin/                  ← 관리자 API 서버
│   └── build.gradle
│
├── domain-user/                ← 사용자 도메인
│   └── build.gradle
│
├── domain-order/               ← 주문 도메인
│   └── build.gradle
│
├── domain-product/             ← 상품 도메인
│   └── build.gradle
│
├── infrastructure-persistence/ ← JPA, DB 접근
│   └── build.gradle
│
├── infrastructure-messaging/   ← Kafka, 이벤트
│   └── build.gradle
│
└── common/                     ← 공통 유틸리티
    └── build.gradle
```

`settings.gradle`:

```groovy
rootProject.name = 'shop-backend'

include(
    'app-api',
    'app-admin',
    'domain-user',
    'domain-order',
    'domain-product',
    'infrastructure-persistence',
    'infrastructure-messaging',
    'common'
)
```

### 3.3 의존성 방향 설계

의존성 방향은 아키텍처의 핵심이다. 잘못 설계하면 멀티모듈이 단순히 폴더만 나눈 것과 다를 바 없어진다.

```
                    ┌─────────────┐
                    │   app-api   │
                    │  app-admin  │
                    └──────┬──────┘
                           │ depends on
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  domain  │ │  domain  │ │  domain  │
        │   user   │ │  order   │ │ product  │
        └─────┬────┘ └────┬─────┘ └────┬─────┘
              │            │            │
              └────────────┼────────────┘
                           │ depends on
                    ┌──────┴──────┐
                    │   common    │
                    └─────────────┘

infrastructure modules depend on domain modules (DIP)
```

`domain-order/build.gradle`:

```groovy
plugins {
    id 'java-library'
}

dependencies {
    api project(':common')

    // 다른 도메인 모듈에 직접 의존하지 않음
    // 필요하다면 도메인 이벤트나 인터페이스를 통해 간접 참조

    implementation 'org.springframework.boot:spring-boot-starter'

    testImplementation 'org.junit.jupiter:junit-jupiter'
    testImplementation 'org.assertj:assertj-core'
}
```

`infrastructure-persistence/build.gradle`:

```groovy
plugins {
    id 'java-library'
}

dependencies {
    implementation project(':domain-user')
    implementation project(':domain-order')
    implementation project(':domain-product')
    implementation project(':common')

    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.querydsl:querydsl-jpa'

    runtimeOnly 'org.postgresql:postgresql'
}
```

`app-api/build.gradle`:

```groovy
plugins {
    id 'org.springframework.boot'
    id 'java'
}

dependencies {
    implementation project(':domain-user')
    implementation project(':domain-order')
    implementation project(':domain-product')
    implementation project(':infrastructure-persistence')
    implementation project(':infrastructure-messaging')
    implementation project(':common')

    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-security'
}
```

### 3.4 공통 의존성 관리: Version Catalog

여러 모듈이 같은 라이브러리를 사용할 때 버전 관리가 골치거리가 된다. Gradle의 Version Catalog로 해결한다.

`gradle/libs.versions.toml`:

```toml
[versions]
spring-boot = "3.2.0"
spring-cloud = "2023.0.0"
querydsl = "5.0.0"
mapstruct = "1.5.5.Final"
archunit = "1.2.0"

[libraries]
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web", version.ref = "spring-boot" }
spring-boot-starter-data-jpa = { module = "org.springframework.boot:spring-boot-starter-data-jpa", version.ref = "spring-boot" }
querydsl-jpa = { module = "com.querydsl:querydsl-jpa", version.ref = "querydsl" }
mapstruct = { module = "org.mapstruct:mapstruct", version.ref = "mapstruct" }
archunit = { module = "com.tngtech.archunit:archunit-junit5", version.ref = "archunit" }

[bundles]
spring-web = ["spring-boot-starter-web", "spring-boot-starter-validation"]
```

각 서브모듈에서는 이렇게 사용한다:

```groovy
dependencies {
    implementation libs.spring.boot.starter.web
    implementation libs.querydsl.jpa
    testImplementation libs.archunit
}
```

버전을 한 곳에서 관리하므로 전체 업그레이드가 한 줄 변경으로 끝난다.

### 3.5 모듈 간 통신: 직접 의존 vs 이벤트

모듈이 분리됐어도 기능상 연관된 도메인들은 서로 소통해야 한다. 예를 들어 주문이 완료되면 사용자의 적립금을 쌓아야 한다. 이 때 `domain-order`가 `domain-user`에 직접 의존하면 경계가 흐려진다.

**안티패턴: 직접 의존**

```java
// domain-order 모듈 내부
public class OrderCompletionService {
    private final UserPointService userPointService; // domain-user 모듈 직접 참조

    public void completeOrder(Order order) {
        order.complete();
        // 도메인 경계 위반 — order가 user에 직접 의존
        userPointService.addPoints(order.getUserId(), order.calculatePoints());
    }
}
```

**올바른 패턴: 도메인 이벤트**

```java
// common 모듈에 이벤트 정의
public record OrderCompletedEvent(
    OrderId orderId,
    UserId userId,
    Money totalAmount,
    int earnedPoints,
    Instant occurredAt
) implements DomainEvent {}

// domain-order 모듈
public class OrderCompletionService {
    private final ApplicationEventPublisher eventPublisher;

    public void completeOrder(Order order) {
        order.complete();
        orderRepository.save(order);

        // 이벤트 발행 — 누가 처리하는지 알 필요 없음
        eventPublisher.publishEvent(new OrderCompletedEvent(
            order.getId(),
            order.getUserId(),
            order.getTotalAmount(),
            order.calculateEarnedPoints(),
            Instant.now()
        ));
    }
}

// domain-user 모듈
@Component
public class OrderCompletedEventHandler {
    private final UserPointService userPointService;

    @EventListener
    public void handle(OrderCompletedEvent event) {
        userPointService.addPoints(event.userId(), event.earnedPoints());
    }
}
```

`domain-order`는 `OrderCompletedEvent`를 발행할 뿐, `domain-user`가 이것을 처리한다는 사실을 모른다. 두 모듈의 의존 방향이 역전되어 있다. `common` 모듈의 이벤트 타입에만 양쪽이 의존한다.

---

## 4. 모놀리스 내 모듈 분리 → MSA 전환 준비

"처음부터 MSA로 가면 안 되나요?"라는 질문을 자주 받는다. 안 된다. 도메인이 충분히 정제되지 않은 상태에서 서비스를 물리적으로 분리하면 '분산 모놀리스(Distributed Monolith)'라는 최악의 결과를 얻게 된다. 서비스가 분리됐지만 서로 강하게 결합되어 있어 배포도 어렵고 장애도 더 복잡해지는 상황이다.

올바른 경로는 **모놀리스 내에서 모듈 경계를 명확하게 정의하고, 그 경계가 검증된 후에 물리적으로 분리**하는 것이다.

### 4.1 모듈러 모놀리스 (Modular Monolith)

모듈러 모놀리스는 단일 배포 단위이지만 내부적으로 강한 모듈 경계를 가진 아키텍처다. 배포는 하나지만 코드는 마치 여러 서비스처럼 분리되어 있다.

```
┌─────────────────────────────────────────────────┐
│                  단일 배포 단위                    │
│                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │
│  │    User     │  │    Order    │  │  Product  │ │
│  │   Module    │  │   Module    │  │  Module   │ │
│  │             │  │             │  │           │ │
│  │ - API       │  │ - API       │  │ - API     │ │
│  │ - App       │  │ - App       │  │ - App     │ │
│  │ - Domain    │  │ - Domain    │  │ - Domain  │ │
│  │ - Infra     │  │ - Infra     │  │ - Infra   │ │
│  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘ │
│         │                │               │        │
│         └────────────────┴───────────────┘        │
│                    이벤트 버스                      │
└─────────────────────────────────────────────────┘
```

모듈 간 통신은 반드시 공개 API나 이벤트를 통해서만 이루어진다. 내부 구현에는 절대 직접 접근하지 않는다.

### 4.2 MSA 전환을 위한 체크리스트

모놀리스 내 모듈이 다음 조건을 충족하면 독립 서비스로 분리할 준비가 된 것이다.

**도메인 경계 명확성 체크**

```
□ 모듈의 공개 API가 명확하게 정의되어 있는가?
□ 다른 모듈이 내부 구현에 직접 접근하지 않는가?
□ 모듈 간 통신이 이벤트나 공개 인터페이스를 통해서만 이루어지는가?
□ 순환 의존성이 없는가?
□ 모듈의 데이터가 다른 모듈 테이블에 직접 Join하지 않는가?
```

**운영 준비도 체크**

```
□ 모듈의 독립적인 빌드/테스트가 가능한가?
□ 모듈이 독립적으로 장애를 처리하는 로직이 있는가?
□ 모듈의 설정이 다른 모듈과 분리되어 있는가?
```

### 4.3 Strangler Fig 패턴으로 점진적 분리

전체를 한 번에 분리하는 것은 실패 확률이 높다. Strangler Fig(교살 무화과) 패턴은 레거시 시스템을 점진적으로 새 시스템으로 교체하는 접근법이다.

**1단계: 식별 및 격리**

MSA로 분리할 첫 번째 후보를 선택한다. 일반적으로 변경 빈도가 높거나, 트래픽이 집중되거나, 팀이 명확하게 분리된 도메인이 적합하다.

```java
// 분리 전: OrderService가 모든 것을 알고 있음
@Service
public class OrderService {
    private final ProductRepository productRepository; // product 직접 접근
    private final UserRepository userRepository;       // user 직접 접근
    private final InventoryService inventoryService;   // inventory 직접 접근

    public OrderId placeOrder(PlaceOrderCommand command) {
        User user = userRepository.findById(command.userId());
        Product product = productRepository.findById(command.productId());
        inventoryService.reserve(command.productId(), command.quantity());
        // ...
    }
}
```

**2단계: Anti-Corruption Layer 도입**

외부 서비스를 호출하는 코드와 내부 도메인 로직을 분리하는 레이어를 만든다. 나중에 이 레이어만 교체하면 된다.

```java
// Anti-Corruption Layer: 미래의 외부 서비스 호출을 추상화
public interface ProductCatalogPort {
    ProductInfo getProductInfo(ProductId productId);
    boolean isAvailable(ProductId productId, int quantity);
}

// 현재는 내부 모듈을 직접 호출하는 구현체
@Component
public class InternalProductCatalogAdapter implements ProductCatalogPort {
    private final ProductRepository productRepository;

    @Override
    public ProductInfo getProductInfo(ProductId productId) {
        Product product = productRepository.findById(productId.value())
            .orElseThrow(() -> new ProductNotFoundException(productId));
        return new ProductInfo(product.getId(), product.getName(), product.getPrice());
    }
}

// 분리 후: HTTP 클라이언트로 교체
@Component
@ConditionalOnProperty(name = "feature.product-service-separated", havingValue = "true")
public class HttpProductCatalogAdapter implements ProductCatalogPort {
    private final ProductServiceClient productServiceClient;

    @Override
    public ProductInfo getProductInfo(ProductId productId) {
        return productServiceClient.getProduct(productId.value());
    }
}
```

`ProductCatalogPort` 인터페이스 뒤에 구현체를 교체함으로써 `OrderService`의 코드 변경 없이 내부 모듈 호출을 외부 HTTP 호출로 전환할 수 있다.

**3단계: 데이터 분리**

코드 분리보다 어려운 것이 데이터 분리다. 모놀리스에서는 모든 테이블이 같은 DB에 있어 Join이 자유롭다. 서비스를 분리하려면 데이터도 분리해야 한다.

```sql
-- 분리 전: 직접 Join
SELECT o.id, o.total, p.name, p.price, u.email
FROM orders o
JOIN products p ON o.product_id = p.id
JOIN users u ON o.user_id = u.id
WHERE o.status = 'COMPLETED';

-- 분리 후: 각 서비스가 필요한 데이터를 자신의 테이블에 복사
-- orders 테이블에 비정규화된 컬럼 추가
ALTER TABLE orders
    ADD COLUMN product_name VARCHAR(255),
    ADD COLUMN product_price DECIMAL(19,2),
    ADD COLUMN user_email VARCHAR(255);

-- 주문 생성 시 해당 시점의 값을 복사해 저장
```

데이터를 비정규화하는 것이 찜찜할 수 있다. 하지만 주문이 완료된 시점의 상품명과 가격은 이후에 상품 정보가 바뀌어도 변하면 안 된다. 비정규화가 오히려 더 정확한 데이터 모델이 된다.

### 4.4 모듈 분리 시 자주 겪는 문제들

**문제 1: 트랜잭션 경계 붕괴**

모놀리스에서는 단일 트랜잭션으로 여러 테이블을 묶을 수 있다. 서비스가 분리되면 분산 트랜잭션이 필요한데, 이는 구현이 복잡하고 성능에도 영향을 준다.

해결책은 Saga 패턴이다. 각 서비스가 로컬 트랜잭션만 담당하고, 실패 시 보상 트랜잭션(compensating transaction)으로 일관성을 복구한다.

```java
// 주문 생성 Saga (Choreography 방식)
// 1. OrderService: 주문 생성 (PENDING 상태)
// 2. InventoryService: 재고 예약 이벤트 처리
// 3. PaymentService: 결제 처리 이벤트 처리
// 4. OrderService: 주문 확정 (CONFIRMED 상태)
// 실패 시: 보상 트랜잭션으로 역방향 처리

@Component
public class OrderSagaOrchestrator {

    @EventListener
    public void on(PaymentFailedEvent event) {
        // 재고 예약 해제
        inventoryPort.cancelReservation(event.orderId());
        // 주문 취소
        orderRepository.updateStatus(event.orderId(), OrderStatus.CANCELLED);
    }
}
```

**문제 2: 분산 추적의 어려움**

단일 서비스에서는 로그를 한 곳에서 볼 수 있다. 여러 서비스로 분리되면 하나의 요청이 여러 서비스를 거치기 때문에 추적이 어려워진다. 분리 전에 반드시 분산 추적 시스템(OpenTelemetry + Jaeger 또는 Zipkin)을 먼저 구축해야 한다.

```java
// 모든 서비스 간 통신에 Trace ID를 전파
@Configuration
public class TracingConfig {
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .additionalInterceptors(new TracingRestTemplateInterceptor())
            .build();
    }
}
```

**문제 3: 로컬 개발 환경의 복잡성**

서비스가 5개로 늘어나면 로컬에서 전체를 띄우는 것이 복잡해진다. Docker Compose로 이를 관리한다.

```yaml
# docker-compose.yml
services:
  user-service:
    build: ./user-service
    ports: ["8081:8080"]
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/userdb

  order-service:
    build: ./order-service
    ports: ["8082:8080"]
    depends_on: [user-service, kafka]

  kafka:
    image: confluentinc/cp-kafka:7.4.0

  db:
    image: postgres:15
```

---

## 5. 실전 패턴: 중간 규모 팀을 위한 권장 구조

이론을 정리하고, 실제로 10~30명 규모의 팀이 운영하는 서버 애플리케이션에 적용할 수 있는 구체적인 구조를 제시한다.

### 5.1 권장하는 시작점

```
shop-backend/
├── settings.gradle
├── build.gradle
│
├── app/                           ← 단일 애플리케이션 진입점
│   ├── src/main/java/
│   │   └── ShopApplication.java
│   └── build.gradle
│
├── domain/                        ← 핵심 비즈니스 로직
│   ├── user/
│   │   ├── User.java
│   │   ├── UserRepository.java    ← 인터페이스
│   │   └── UserDomainService.java
│   ├── order/
│   └── product/
│
├── application/                   ← 유스케이스 (도메인 조합)
│   ├── user/
│   │   ├── RegisterUserUseCase.java
│   │   └── RegisterUserCommand.java
│   └── order/
│       ├── PlaceOrderUseCase.java
│       └── PlaceOrderCommand.java
│
├── adapter/                       ← 외부와의 연결
│   ├── in/
│   │   └── web/
│   │       ├── UserController.java
│   │       └── OrderController.java
│   └── out/
│       ├── persistence/
│       │   ├── UserJpaRepository.java
│       │   └── UserRepositoryImpl.java
│       └── messaging/
│           └── KafkaEventPublisher.java
│
└── common/                        ← 공유 코드
    ├── exception/
    ├── event/
    └── util/
```

이 구조는 Hexagonal Architecture(포트 & 어댑터)를 따른다. `domain`은 어떤 것에도 의존하지 않는다. `application`은 `domain`에만 의존한다. `adapter`는 `application`에 의존하고, 프레임워크와 외부 시스템을 다룬다.

### 5.2 팀 성장에 따른 모듈 분리 로드맵

```
1단계 (팀 5명 이하):
  단일 Gradle 모듈, 기능별 패키지 구조

2단계 (팀 10명 내외):
  Gradle 멀티모듈 (domain, application, adapter, common)
  ArchUnit 테스트로 경계 강제

3단계 (팀 20명 내외, 도메인 3개 이상):
  도메인별 Gradle 서브모듈 분리
  모듈 간 통신은 이벤트 기반
  각 도메인 팀이 독립적으로 개발/배포 준비

4단계 (도메인 경계 검증 완료 후):
  준비된 모듈부터 독립 서비스로 분리
  Anti-Corruption Layer를 HTTP 호출로 교체
```

---

## 6. 자주 저지르는 실수들

모듈 시스템 설계에서 반복적으로 보이는 실수들을 정리한다.

### 공유 커널(Shared Kernel) 남용

"공통으로 쓰는 거 다 `common`에 넣자"는 생각이 가장 위험하다. `common` 모듈이 비대해지면 모든 모듈이 `common`에 의존하게 되고, `common` 변경 시 전체를 재빌드해야 한다. `common`에는 정말로 어떤 도메인에도 속하지 않는 것만 넣어야 한다. `Money`, `Page`, `DomainEvent` 같은 것들이다.

### 모듈 간 DTO 공유

```java
// 잘못된 예: OrderResponse를 user 모듈에서도 사용
// user 모듈이 order 모듈에 의존하게 됨
public class UserDashboardService {
    public UserDashboard getDashboard(UserId userId) {
        List<OrderResponse> recentOrders = orderService.getRecentOrders(userId);
        // ...
    }
}
```

각 모듈은 자신의 API 응답 객체를 가져야 한다. 다른 모듈의 DTO를 사용하면 그 모듈에 의존하게 된다. 필요하다면 별도의 조회 전용 모델(Read Model)을 만든다.

### 순환 의존성

```
module-A → module-B → module-A (순환!)
```

순환 의존성이 생기는 순간 두 모듈은 사실상 하나다. 별도로 빌드하거나 배포할 수 없다. 순환이 발생하면 두 모듈이 실제로 하나의 도메인인지, 아니면 공통 부분을 `common`으로 추출해야 하는지 검토해야 한다.

### 너무 이른 분리

팀이 3명인데 마이크로서비스 10개로 시작하는 경우를 실제로 본 적 있다. 운영 복잡도가 치솟고, 개발 속도는 반토막 난다. 도메인이 충분히 이해되지 않은 상태에서의 분리는 나중에 경계를 다시 그어야 하는 더 큰 비용을 치르게 한다.

---

## 마무리

모듈 시스템 설계는 코드를 어디에 넣는지의 문제가 아니다. 팀이 어떻게 협업하고, 시스템이 어떻게 성장하고, 장애가 어디서 멈추는지의 문제다. 좋은 모듈 경계는 변경의 영향 범위를 명확하게 한정해주고, 팀이 자율적으로 움직일 수 있는 공간을 만들어준다.

패키지 구조를 기능별로 정리하고, ArchUnit으로 경계를 강제하고, Gradle 멀티모듈로 의존성을 명시적으로 만들어라. 그리고 도메인이 충분히 이해되었을 때, Anti-Corruption Layer를 활용해 모놀리스에서 MSA로 점진적으로 전환하라. 이 순서를 지키는 팀과 지키지 않는 팀의 결과는 1~2년 안에 명확하게 갈린다.

---

## 참고 자료

1. **"Building Microservices" — Sam Newman** (O'Reilly, 2021): MSA 설계의 표준 교재. 서비스 경계 정의, Strangler Fig 패턴, 분산 시스템 트레이드오프를 깊이 다룬다.

2. **"Monolith to Microservices" — Sam Newman** (O'Reilly, 2019): 레거시 모놀리스를 MSA로 점진적으로 전환하는 실전 전략. Strangler Fig, Branch by Abstraction 등 구체적인 패턴을 설명한다.

3. **"Implementing Domain-Driven Design" — Vaughn Vernon** (Addison-Wesley, 2013): Bounded Context와 Context Map을 통해 모듈 경계를 도메인 관점에서 정의하는 방법을 다룬다.

4. **"Clean Architecture" — Robert C. Martin** (Prentice Hall, 2017): 의존성 규칙, 포트 & 어댑터 패턴, 계층 간 경계를 명확히 정의하는 방법을 체계적으로 설명한다.

5. **ArchUnit Documentation** (https://www.archunit.org): Java 아키텍처 규칙을 테스트 코드로 작성하고 검증하는 라이브러리의 공식 문서. 실전 예제가 풍부하다.

6. **"Gradle Multi-Project Builds" — Gradle 공식 문서** (https://docs.gradle.org/current/userguide/multi_project_builds.html): 멀티모듈 프로젝트 설정, 의존성 관리, 빌드 캐시 최적화에 대한 공식 레퍼런스.
