---
title: "[서버 애플리케이션 설계] 1편 — 객체지향 설계 원칙: SOLID가 실제로 의미하는 것"
date: 2026-03-18T00:09:00+09:00
draft: false
tags: ["SOLID", "객체지향", "설계", "서버", "아키텍처"]
series: ["서버 애플리케이션 설계"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 2
summary: "SRP, OCP, LSP, ISP, DIP 각 원칙의 실전 위반/준수 사례, 원칙 간 충돌에서의 판단 기준, 그리고 SOLID를 과하게 적용하는 함정까지."
---

## 들어가며

SOLID는 로버트 마틴(Uncle Bob)이 정리한 객체지향 설계의 다섯 가지 원칙이다. 면접에서 단골로 나오고, 모든 설계 책이 다루지만, 실무에서 **정확히 어떻게 적용하는지** 명확하게 설명할 수 있는 개발자는 많지 않다.

그 이유 중 하나는 SOLID가 "항상 이렇게 하라"는 절대 규칙이 아니기 때문이다. 원칙들은 서로 충돌하기도 하고, 과도하게 적용하면 오히려 코드를 복잡하게 만든다. 이번 편에서는 각 원칙의 본질을 이해하고, 실전에서의 판단 기준을 다룬다.

---

## 1. SRP — 단일 책임 원칙

### 정의

> "모듈은 하나의, 오직 하나의 액터(actor)에 대해서만 책임져야 한다."

흔히 "하나의 클래스는 하나의 일만 해야 한다"로 알려져 있지만, 원래 정의는 **액터(변경을 요청하는 주체)** 중심이다. 핵심은: **서로 다른 이유로 변경되는 코드를 분리하라.**

### 위반 사례

```java
public class UserService {
    // 1. 비즈니스 로직 (기획팀이 변경 요청)
    public User register(String email, String password) {
        validate(email, password);
        User user = new User(email, hashPassword(password));
        userRepository.save(user);
        sendWelcomeEmail(user);  // 여기서 이메일도 보냄
        return user;
    }

    // 2. 이메일 발송 (마케팅팀이 변경 요청)
    private void sendWelcomeEmail(User user) {
        String html = renderTemplate("welcome", user);
        emailClient.send(user.getEmail(), "환영합니다!", html);
    }

    // 3. 비밀번호 해싱 (보안팀이 변경 요청)
    private String hashPassword(String raw) {
        return BCrypt.hashpw(raw, BCrypt.gensalt(12));
    }
}
```

이 클래스는 세 가지 이유로 변경된다:
- 가입 정책 변경 (기획)
- 이메일 템플릿/발송 방식 변경 (마케팅)
- 암호화 알고리즘 변경 (보안)

### 준수 사례

```java
// 각 책임을 분리
public class UserService {
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;
    private final UserEventPublisher eventPublisher;

    public User register(String email, String password) {
        validate(email, password);
        String hashed = passwordEncoder.encode(password);
        User user = userRepository.save(new User(email, hashed));
        eventPublisher.publish(new UserRegisteredEvent(user));
        return user;
    }
}

public class WelcomeEmailListener {
    @EventListener
    public void onUserRegistered(UserRegisteredEvent event) {
        // 이메일 관련 변경은 여기서만
    }
}

public class BcryptPasswordEncoder implements PasswordEncoder {
    // 암호화 관련 변경은 여기서만
}
```

### 판단 기준

"이 클래스를 변경해야 하는 사람이 둘 이상인가?"를 자문한다. 만약 기획자의 요청과 DBA의 요청이 같은 클래스를 건드린다면 SRP 위반 가능성이 높다.

하지만 **모든 메서드를 별도 클래스로 분리하라는 뜻이 아니다.** 단순한 CRUD 서비스를 10개 클래스로 쪼개면 오히려 응집도가 떨어진다.

---

## 2. OCP — 개방-폐쇄 원칙

### 정의

> "소프트웨어는 확장에 열려 있고, 수정에 닫혀 있어야 한다."

기존 코드를 수정하지 않고 새로운 동작을 추가할 수 있어야 한다.

### 위반 사례

```java
public class PaymentProcessor {
    public void process(Payment payment) {
        switch (payment.getMethod()) {
            case CREDIT_CARD:
                processCreditCard(payment);
                break;
            case BANK_TRANSFER:
                processBankTransfer(payment);
                break;
            case KAKAO_PAY:
                processKakaoPay(payment);
                break;
            // 새 결제 수단 추가 → 이 switch문을 수정해야 함!
        }
    }
}
```

네이버페이를 추가하려면 `PaymentProcessor`를 직접 수정해야 한다. 결제 수단이 늘어날 때마다 이 클래스를 건드려야 하고, 다른 결제 수단에 영향을 줄 위험이 있다.

### 준수 사례

```java
// 추상화 (확장 포인트)
public interface PaymentHandler {
    boolean supports(PaymentMethod method);
    void process(Payment payment);
}

// 각 결제 수단은 독립적 구현
public class CreditCardHandler implements PaymentHandler {
    public boolean supports(PaymentMethod method) {
        return method == CREDIT_CARD;
    }
    public void process(Payment payment) { /* 카드 처리 */ }
}

public class KakaoPayHandler implements PaymentHandler {
    public boolean supports(PaymentMethod method) {
        return method == KAKAO_PAY;
    }
    public void process(Payment payment) { /* 카카오페이 처리 */ }
}

// 새 결제 수단 추가 = 새 클래스 추가 (기존 코드 수정 없음)
public class NaverPayHandler implements PaymentHandler {
    public boolean supports(PaymentMethod method) {
        return method == NAVER_PAY;
    }
    public void process(Payment payment) { /* 네이버페이 처리 */ }
}

// 라우터 (한 번만 작성하면 수정 불필요)
public class PaymentProcessor {
    private final List<PaymentHandler> handlers;

    public void process(Payment payment) {
        handlers.stream()
            .filter(h -> h.supports(payment.getMethod()))
            .findFirst()
            .orElseThrow(() -> new UnsupportedPaymentException(payment.getMethod()))
            .process(payment);
    }
}
```

### 판단 기준

OCP는 **변경이 예상되는 지점**에 적용해야 한다. 모든 곳에 추상화를 만들면 코드가 과도하게 복잡해진다.

```
적용해야 할 때:
- 결제 수단, 알림 채널, 내보내기 형식처럼 확장 가능성이 명확한 경우
- switch/if-else 체인이 여러 곳에 중복되는 경우
- 서드파티 연동처럼 구현이 바뀔 가능성이 높은 경우

적용하지 않아도 될 때:
- 변경 가능성이 거의 없는 경우
- 분기가 2~3개이고 향후 늘어날 가능성이 낮은 경우
- 단순한 내부 로직을 추상화하면 오히려 읽기 어려워지는 경우
```

---

## 3. LSP — 리스코프 치환 원칙

### 정의

> "하위 타입은 상위 타입을 대체할 수 있어야 한다."

부모 클래스를 사용하는 코드에 자식 클래스를 넣어도 **프로그램의 정확성이 깨지면 안 된다.** 단순히 컴파일이 되는 것이 아니라, **행동 계약(behavioral contract)**을 지켜야 한다.

### 고전적 위반 — 정사각형/직사각형 문제

```java
public class Rectangle {
    protected int width, height;

    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w) {
        this.width = w;
        this.height = w;  // 정사각형이니까 높이도 변경
    }

    @Override
    public void setHeight(int h) {
        this.width = h;
        this.height = h;
    }
}

// 이 코드는 Rectangle에서는 정상, Square에서는 깨진다
void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.area() == 50;  // Square이면 실패! area() = 100
}
```

### 실전 위반 — 예외를 던지는 하위 타입

```java
public interface Cache {
    void put(String key, Object value);
    Object get(String key);
    void delete(String key);
}

// 읽기 전용 캐시: delete를 지원하지 않음
public class ReadOnlyCache implements Cache {
    public void put(String key, Object value) { /* 저장 */ }
    public Object get(String key) { return /* ... */; }

    public void delete(String key) {
        throw new UnsupportedOperationException("읽기 전용입니다");
        // Cache를 기대하는 코드에서 예상치 못한 예외!
    }
}
```

`Cache`를 사용하는 코드는 `delete()`가 정상 동작할 것을 기대한다. `ReadOnlyCache`를 넣으면 런타임에 폭발한다. 이것이 LSP 위반이다.

### 올바른 해결

```java
public interface ReadableCache {
    Object get(String key);
}

public interface WritableCache extends ReadableCache {
    void put(String key, Object value);
    void delete(String key);
}

// 읽기만 필요한 곳은 ReadableCache를 받으면 됨
public class CacheConsumer {
    private final ReadableCache cache;  // delete()가 없으므로 안전
}
```

### LSP 위반의 징후

- 하위 타입에서 `throw new UnsupportedOperationException()`
- `instanceof` 체크 후 분기 처리
- 상위 타입의 사전 조건(precondition)을 강화하거나, 사후 조건(postcondition)을 약화

---

## 4. ISP — 인터페이스 분리 원칙

### 정의

> "클라이언트는 자신이 사용하지 않는 메서드에 의존하면 안 된다."

하나의 거대한 인터페이스보다 여러 개의 작은 인터페이스가 낫다.

### 위반 사례

```java
public interface UserRepository {
    User findById(Long id);
    List<User> findAll();
    User save(User user);
    void delete(Long id);
    List<User> findByName(String name);
    List<User> findByEmail(String email);
    Page<User> findAll(Pageable pageable);
    List<User> findActiveUsers();
    long countByRole(Role role);
    void bulkUpdateStatus(Status status, List<Long> ids);
    List<UserStatistics> getStatistics(DateRange range);
}

// 관리자 대시보드: 통계만 필요
public class AdminDashboard {
    private final UserRepository repo;  // 11개 메서드 중 1개만 사용

    public DashboardData load() {
        return new DashboardData(repo.getStatistics(thisMonth()));
    }
}
```

`AdminDashboard`는 통계 메서드 하나만 쓰는데, 11개 메서드가 있는 인터페이스에 의존한다. `UserRepository`에 메서드가 추가되거나 시그니처가 바뀌면, 이 클래스도 영향을 받을 수 있다.

### 준수 사례

```java
// 역할별로 인터페이스 분리
public interface UserReader {
    User findById(Long id);
    List<User> findByName(String name);
    List<User> findByEmail(String email);
}

public interface UserWriter {
    User save(User user);
    void delete(Long id);
    void bulkUpdateStatus(Status status, List<Long> ids);
}

public interface UserStatisticsProvider {
    List<UserStatistics> getStatistics(DateRange range);
    long countByRole(Role role);
}

// 구현체는 여러 인터페이스를 한꺼번에 구현 가능
public class JpaUserRepository implements UserReader, UserWriter, UserStatisticsProvider {
    // 전체 구현
}

// 각 클라이언트는 필요한 인터페이스만 의존
public class AdminDashboard {
    private final UserStatisticsProvider stats;  // 깔끔!
}

public class UserRegistration {
    private final UserWriter writer;
    private final UserReader reader;
}
```

### CQRS와의 연결

ISP를 데이터 접근 계층에 적용하면 자연스럽게 **CQRS(Command Query Responsibility Segregation)** 패턴에 가까워진다:

```
Command (쓰기) → UserWriter
Query (읽기)   → UserReader
통계/분석      → UserStatisticsProvider
```

---

## 5. DIP — 의존성 역전 원칙

### 정의

> "상위 수준 모듈은 하위 수준 모듈에 의존해서는 안 된다. 둘 다 추상화에 의존해야 한다."

### 위반 사례

```java
// 상위 수준: 비즈니스 로직
public class OrderService {
    // 하위 수준 모듈에 직접 의존
    private final MySqlOrderRepository repository = new MySqlOrderRepository();
    private final SmtpEmailSender emailSender = new SmtpEmailSender();

    public void placeOrder(Order order) {
        repository.save(order);
        emailSender.send(order.getCustomerEmail(), "주문 확인");
    }
}
```

`OrderService`(비즈니스 로직)가 `MySqlOrderRepository`(인프라 상세)에 직접 의존한다. 데이터베이스를 PostgreSQL로 바꾸거나, 이메일 대신 카카오톡 알림을 보내려면 `OrderService`를 수정해야 한다.

```
의존 방향 (위반):
OrderService → MySqlOrderRepository
OrderService → SmtpEmailSender

상위 수준이 하위 수준에 의존!
```

### 준수 사례

```java
// 추상화 (상위 수준 모듈이 정의)
public interface OrderRepository {
    void save(Order order);
    Order findById(Long id);
}

public interface NotificationSender {
    void send(String to, String message);
}

// 상위 수준: 추상화에 의존
public class OrderService {
    private final OrderRepository repository;
    private final NotificationSender notificationSender;

    public OrderService(OrderRepository repository, NotificationSender sender) {
        this.repository = repository;
        this.notificationSender = sender;
    }

    public void placeOrder(Order order) {
        repository.save(order);
        notificationSender.send(order.getCustomerEmail(), "주문 확인");
    }
}

// 하위 수준: 추상화를 구현
public class PostgresOrderRepository implements OrderRepository { ... }
public class KakaoNotificationSender implements NotificationSender { ... }
```

```
의존 방향 (준수):
OrderService → OrderRepository (인터페이스)
                    ↑
PostgresOrderRepository (구현)

OrderService → NotificationSender (인터페이스)
                    ↑
KakaoNotificationSender (구현)

상위/하위 모두 추상화에 의존 → 의존성이 "역전"됨
```

### DIP의 핵심: "인터페이스의 소유권"

DIP에서 자주 놓치는 포인트: **인터페이스는 상위 수준 모듈이 소유해야 한다.**

```
잘못된 구조:
infrastructure/
  ├── mysql/
  │   ├── MysqlOrderRepository.java
  │   └── OrderRepository.java      ← 인프라 패키지에 인터페이스가 있음!
domain/
  └── OrderService.java             ← 인프라에 의존

올바른 구조:
domain/
  ├── OrderService.java
  └── OrderRepository.java          ← 도메인 패키지에 인터페이스
infrastructure/
  └── mysql/
      └── MysqlOrderRepository.java ← 인프라가 도메인을 구현
```

인터페이스가 어떤 패키지에 있느냐가 의존 방향을 결정한다.

---

## 6. 원칙 간의 충돌

### SRP vs DRY (Don't Repeat Yourself)

```java
// SRP를 위해 분리한 서비스들
public class UserRegistrationService {
    public void register(User user) {
        validate(user);  // 검증 로직
        // 가입 처리
    }
}

public class UserProfileService {
    public void update(User user) {
        validate(user);  // 같은 검증 로직이 중복!
        // 프로필 수정
    }
}
```

SRP를 위해 서비스를 분리하면 검증 로직이 중복될 수 있다. 이때 검증을 별도 클래스로 추출하면 DRY를 지키면서 SRP도 유지할 수 있다:

```java
public class UserValidator {
    public void validate(User user) { /* 검증 로직 한 곳에 */ }
}
```

### OCP vs YAGNI (You Aren't Gonna Need It)

OCP는 "확장에 열려 있어야 한다"고 말하고, YAGNI는 "아직 필요 없는 것을 만들지 마라"고 말한다.

```java
// YAGNI 관점: 현재 결제 수단이 카드뿐이면 이걸로 충분
public class PaymentService {
    public void pay(Payment payment) {
        processCreditCard(payment);
    }
}

// OCP 관점: 나중을 대비해서 추상화를 미리 만들어야 할까?
```

**판단 기준**: 확장이 **구체적으로 예정**되어 있으면 OCP를 적용한다. "언젠가 필요할지도 몰라"는 YAGNI를 따른다. 결제 수단 추가가 분기 로드맵에 있다면 추상화를 만들고, 아직 카드만 쓰인다면 단순하게 유지한다.

### ISP vs 편의성

인터페이스를 너무 잘게 쪼개면:

```java
// 과한 분리
public interface Findable<T> { T findById(Long id); }
public interface Saveable<T> { T save(T entity); }
public interface Deletable { void delete(Long id); }
public interface Countable { long count(); }
public interface Listable<T> { List<T> findAll(); }

// 사용하는 곳에서 5개 인터페이스를 조합해야 함
public class SomeService {
    private final Findable<User> finder;
    private final Saveable<User> saver;
    private final Deletable deleter;
    // ... 너무 번거로움
}
```

인터페이스 분리의 단위는 **클라이언트의 역할** 기준이지, 메서드 하나하나가 아니다.

---

## 7. SOLID 과적용의 함정

### 추상화 폭발

```java
// 과한 SOLID: 10줄짜리 기능에 6개 클래스, 3개 인터페이스
public interface MessageFormatter { ... }
public interface MessageSender { ... }
public interface MessageValidator { ... }
public class DefaultMessageFormatter implements MessageFormatter { ... }
public class EmailMessageSender implements MessageSender { ... }
public class LengthMessageValidator implements MessageValidator { ... }
public class MessageService {
    public MessageService(
        MessageFormatter formatter,
        MessageSender sender,
        MessageValidator validator
    ) { ... }
}
public class MessageServiceFactory { ... }
public class MessageServiceConfig { ... }
```

이 정도 규모의 기능이라면:

```java
// 실용적: 한 클래스로 충분
public class MessageService {
    public void send(String to, String body) {
        if (body.length() > 1000) throw new IllegalArgumentException("메시지가 너무 깁니다");
        String formatted = "[알림] " + body;
        emailClient.send(to, formatted);
    }
}
```

### "나중에 바뀔 수 있으니까" 함정

미래의 변경을 예측해서 미리 추상화하는 것은 대부분 틀린다:

```
예측: "나중에 이메일 외에 SMS도 보낼 거야"
결과: 2년이 지나도 이메일만 사용
비용: 불필요한 인터페이스와 팩토리가 코드 복잡도를 높이고 있음
```

**Rule of Three**: 같은 패턴이 세 번 나타날 때 추상화한다. 한두 번은 중복을 허용하고, 세 번째에 확실한 패턴이 보이면 그때 리팩토링한다.

### 실용적 SOLID 적용 가이드

```
규모별 적용 강도:

소규모 (10줄 이하 메서드, 단순 CRUD):
→ SOLID 약하게. 직접적이고 단순한 코드 우선.
→ 클래스 하나에 관련 로직을 모아두는 것이 읽기 쉬울 수 있음.

중규모 (수백 줄, 여러 의존성):
→ SRP와 DIP를 중심으로 적용.
→ 서비스-레포지토리 분리, 외부 의존성 인터페이스화.

대규모 (수천 줄, 여러 팀이 작업):
→ SOLID 전체를 적극 적용.
→ 모듈 경계 명확히, 인터페이스로 계약 정의.
→ OCP로 플러그인 방식 확장 가능하게.
```

---

## 마치며

SOLID는 법칙이 아니라 **경험적 지침**이다. 핵심을 정리하면:

- **SRP**: 변경의 이유가 하나여야 한다. "액터"를 기준으로 판단한다
- **OCP**: 확장이 예상되는 지점에만 추상화를 만든다. 모든 곳에 적용하면 과잉 설계다
- **LSP**: 하위 타입은 상위 타입의 행동 계약을 지켜야 한다. `UnsupportedOperationException`은 위반의 징후다
- **ISP**: 클라이언트 역할 기준으로 인터페이스를 분리한다. 메서드 단위가 아니다
- **DIP**: 비즈니스 로직이 인프라 상세에 의존하지 않도록 추상화를 사이에 둔다. 인터페이스 소유권이 핵심이다

가장 중요한 것: **코드가 해결하는 문제의 복잡도에 맞게** SOLID를 적용하는 것이다. 단순한 문제에 복잡한 구조를 적용하는 것은 SOLID의 정신에 오히려 반한다.

---

## 참고 자료

- *Clean Architecture* — Robert C. Martin (Prentice Hall)
- *Agile Software Development, Principles, Patterns, and Practices* — Robert C. Martin
- *Head First Design Patterns, 2nd Edition* — Eric Freeman, Elisabeth Robson (O'Reilly)
- Martin Fowler, "Is Design Dead?" — [https://martinfowler.com/articles/designDead.html](https://martinfowler.com/articles/designDead.html)
- Dan North, "CUPID — for joyful coding" — [https://dannorth.net/cupid-for-joyful-coding/](https://dannorth.net/cupid-for-joyful-coding/)
