---
title: "[인증과 보안] 7편 — 보안 감사와 취약점 관리: 지속적인 보안"
date: 2026-03-17T22:01:00+09:00
draft: false
tags: ["OWASP", "취약점 스캔", "Threat Modeling", "보안 감사", "보안", "서버"]
series: ["인증과 보안"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 4
summary: "OWASP Top 10 항목별 실전 점검, 의존성 취약점 스캔(Snyk, Dependabot), 침투 테스트와 Threat Modeling(STRIDE), 보안 로그 설계와 이상 탐지, 사고 대응 플로우까지"
---

보안은 한 번 구현하면 끝나는 기능이 아니다. 새벽에 배포된 의존성 업데이트 하나, 개발자가 무심코 남긴 SQL 쿼리 한 줄, 예외 처리에서 흘러나온 스택 트레이스 — 이런 사소해 보이는 요소들이 실제 침해 사고의 출발점이 된다. 수많은 기업들이 "우리는 보안에 투자했다"고 말하지만, 정기적인 감사와 취약점 관리 없이는 그 투자가 언제 무너질지 알 수 없다. 이번 편에서는 OWASP Top 10 기준의 실전 점검부터 의존성 스캔, Threat Modeling, 그리고 사고 대응 플로우까지 — 지속적인 보안 체계를 어떻게 구축할 것인지를 다룬다.

---

## 1. OWASP Top 10: 항목별 실전 점검

OWASP(Open Web Application Security Project)는 매년 웹 애플리케이션의 주요 보안 위협을 집계해 Top 10을 발표한다. 단순한 참고 문서가 아니라, 실제 침해 사고 데이터를 기반으로 한 실전 점검 체크리스트다.

### A01: Broken Access Control (접근 제어 실패)

가장 빈번하게 발생하는 취약점이다. 인증은 통과했지만, 권한 없는 리소스에 접근할 수 있는 상황이다.

```java
// 안티패턴: URL 파라미터로 사용자 ID를 직접 받아 조회
@GetMapping("/api/orders/{userId}")
public List<Order> getOrders(@PathVariable Long userId) {
    return orderService.findByUserId(userId); // 다른 사용자의 주문도 조회 가능
}

// 올바른 패턴: 인증된 사용자 컨텍스트에서 ID 추출
@GetMapping("/api/orders")
public List<Order> getOrders(@AuthenticationPrincipal UserDetails userDetails) {
    Long userId = ((CustomUserDetails) userDetails).getUserId();
    return orderService.findByUserId(userId);
}
```

Spring Security에서는 메서드 레벨 보안을 통해 한 번 더 방어선을 구축할 수 있다.

```java
@Service
public class OrderService {

    @PreAuthorize("@orderSecurityService.isOwner(#orderId, authentication)")
    public Order findById(Long orderId) {
        return orderRepository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order not found"));
    }
}

@Component("orderSecurityService")
public class OrderSecurityService {

    private final OrderRepository orderRepository;

    public boolean isOwner(Long orderId, Authentication auth) {
        Long currentUserId = ((CustomUserDetails) auth.getPrincipal()).getUserId();
        return orderRepository.findById(orderId)
            .map(order -> order.getUserId().equals(currentUserId))
            .orElse(false);
    }
}
```

### A02: Cryptographic Failures (암호화 실패)

평문으로 저장된 비밀번호, 약한 해시 알고리즘(MD5, SHA-1), TLS 미적용이 여기에 해당한다.

```java
// 안티패턴: MD5 해시 사용
String hash = DigestUtils.md5Hex(password); // 레인보우 테이블 공격에 취약

// 올바른 패턴: BCrypt (work factor 조절 가능)
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12); // strength: 10~12 권장
}

// 패스워드 검증
boolean matches = passwordEncoder.matches(rawPassword, encodedPassword);
```

데이터베이스 컬럼 수준 암호화가 필요한 경우:

```java
@Entity
public class UserProfile {

    @Convert(converter = AesEncryptConverter.class)
    @Column(name = "social_security_number")
    private String ssn; // 저장 시 암호화, 조회 시 복호화
}

@Converter
public class AesEncryptConverter implements AttributeConverter<String, String> {

    private final AesEncryptor encryptor;

    @Override
    public String convertToDatabaseColumn(String attribute) {
        return attribute == null ? null : encryptor.encrypt(attribute);
    }

    @Override
    public String convertToEntityAttribute(String dbData) {
        return dbData == null ? null : encryptor.decrypt(dbData);
    }
}
```

### A03: Injection (인젝션)

SQL Injection, Command Injection, LDAP Injection 등이 포함된다.

```java
// 안티패턴: 문자열 연결로 쿼리 작성
String query = "SELECT * FROM users WHERE username = '" + username + "'";
// username = "admin' OR '1'='1" 입력 시 모든 사용자 조회

// 올바른 패턴 1: PreparedStatement
String query = "SELECT * FROM users WHERE username = ?";
PreparedStatement stmt = conn.prepareStatement(query);
stmt.setString(1, username);

// 올바른 패턴 2: JPA JPQL 파라미터 바인딩
@Query("SELECT u FROM User u WHERE u.username = :username")
Optional<User> findByUsername(@Param("username") String username);

// 올바른 패턴 3: QueryDSL
public Optional<User> findByUsername(String username) {
    return Optional.ofNullable(
        queryFactory.selectFrom(user)
            .where(user.username.eq(username)) // 자동으로 파라미터 바인딩
            .fetchOne()
    );
}
```

### A04: Insecure Design (불안전한 설계)

구현 문제가 아닌 설계 자체의 결함이다. 비밀번호 재설정 시 보안 질문만으로 인증하는 패턴이 대표적이다. 이 항목은 Threat Modeling 절에서 더 깊이 다룬다.

### A05: Security Misconfiguration (보안 설정 오류)

```yaml
# 안티패턴: 프로덕션에서 디버그 모드 활성화
spring:
  jpa:
    show-sql: true      # SQL 쿼리가 로그에 노출
  thymeleaf:
    cache: false
management:
  endpoints:
    web:
      exposure:
        include: "*"    # Actuator 전체 노출 (heapdump, env 포함)

# 올바른 패턴
management:
  endpoints:
    web:
      exposure:
        include: "health,info,metrics"
  endpoint:
    health:
      show-details: when-authorized
```

Spring Security에서 보안 헤더를 강화한다:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'; script-src 'self'; img-src 'self' data:"))
                .frameOptions(frame -> frame.deny())
                .xssProtection(xss -> xss.headerValue(XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK))
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000))
            );
        return http.build();
    }
}
```

### A06: Vulnerable and Outdated Components (취약한 구성 요소)

이 항목은 다음 절에서 별도로 심도 있게 다룬다.

### A07: Identification and Authentication Failures (인증 실패)

```java
// 브루트포스 방어: 로그인 시도 횟수 제한
@Service
public class LoginAttemptService {

    private final Cache<String, Integer> attemptsCache;

    public LoginAttemptService() {
        this.attemptsCache = CacheBuilder.newBuilder()
            .expireAfterWrite(1, TimeUnit.HOURS)
            .build();
    }

    public void loginFailed(String key) {
        int attempts = getAttempts(key);
        attemptsCache.put(key, attempts + 1);
    }

    public boolean isBlocked(String key) {
        return getAttempts(key) >= 5;
    }

    private int getAttempts(String key) {
        Integer attempts = attemptsCache.getIfPresent(key);
        return attempts == null ? 0 : attempts;
    }
}

@RestController
public class AuthController {

    @PostMapping("/api/auth/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request, HttpServletRequest httpRequest) {
        String clientIp = getClientIp(httpRequest);

        if (loginAttemptService.isBlocked(clientIp)) {
            return ResponseEntity.status(429)
                .body(Map.of("message", "Too many login attempts. Try again later."));
        }

        try {
            // 인증 처리
            String token = authService.authenticate(request);
            loginAttemptService.loginSucceeded(clientIp);
            return ResponseEntity.ok(Map.of("token", token));
        } catch (AuthenticationException e) {
            loginAttemptService.loginFailed(clientIp);
            return ResponseEntity.status(401).body(Map.of("message", "Invalid credentials"));
        }
    }
}
```

### A08: Software and Data Integrity Failures

빌드 파이프라인이나 업데이트 메커니즘에 대한 무결성 검증 부재다. CI/CD에서 서명 검증을 추가하는 것이 기본이다.

### A09: Security Logging and Monitoring Failures

보안 이벤트를 기록하지 않으면 침해 사실조차 모른다. 이 항목은 4절에서 구체적으로 다룬다.

### A10: Server-Side Request Forgery (SSRF)

```java
// 안티패턴: 사용자 입력 URL을 그대로 요청
@GetMapping("/api/fetch")
public String fetchUrl(@RequestParam String url) throws IOException {
    return new URL(url).openConnection().getInputStream().toString();
    // http://169.254.169.254/latest/meta-data/ (AWS 메타데이터 접근 가능)
}

// 올바른 패턴: 허용 목록(allowlist) 기반 검증
@GetMapping("/api/fetch")
public ResponseEntity<String> fetchUrl(@RequestParam String url) {
    if (!isAllowedUrl(url)) {
        return ResponseEntity.badRequest().body("URL not allowed");
    }
    // 요청 처리
}

private boolean isAllowedUrl(String url) {
    List<String> allowedHosts = List.of("api.trusted-partner.com", "cdn.example.com");
    try {
        URI uri = new URI(url);
        return allowedHosts.contains(uri.getHost())
            && List.of("https").contains(uri.getScheme());
    } catch (URISyntaxException e) {
        return false;
    }
}
```

---

## 2. 의존성 취약점 스캔: Snyk과 Dependabot

소스 코드에 취약점이 없더라도, 사용하는 라이브러리에 CVE(Common Vulnerabilities and Exposures)가 존재한다면 그 시스템은 취약하다. Log4Shell(CVE-2021-44228)이 대표적인 사례다.

### Snyk 통합

Snyk은 의존성 트리를 분석해 알려진 취약점을 찾아준다. Maven/Gradle 프로젝트에서 CI 파이프라인에 통합하는 방법이다.

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  snyk-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Run Snyk to check for vulnerabilities
        uses: snyk/actions/maven@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: >
            --severity-threshold=high
            --fail-on=upgradable
            --project-name=${{ github.repository }}

      - name: Upload Snyk results to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: snyk.sarif
```

Snyk 설정 파일로 특정 취약점을 예외 처리할 수 있다:

```yaml
# .snyk
version: v1.25.0
ignore:
  SNYK-JAVA-ORGAPACHELOGGINGLOG4J-2314720:
    - '*':
        reason: "Runtime에서 log4j2 직접 미사용, logback만 사용"
        expires: 2026-06-01T00:00:00.000Z
        created: 2026-01-15T09:00:00.000Z
patch: {}
```

### Dependabot 설정

GitHub Dependabot은 자동으로 PR을 생성해 의존성을 최신 상태로 유지한다.

```yaml
# .github/dependabot.yml
version: 2
updates:
  # Maven 의존성
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Seoul"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
    ignore:
      # 메이저 버전 업그레이드는 수동 검토
      - dependency-name: "org.springframework.boot:spring-boot-starter-parent"
        update-types: ["version-update:semver-major"]

  # Docker 이미지
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

### OWASP Dependency-Check (오프라인 환경)

인터넷 접근이 제한된 환경에서는 OWASP Dependency-Check를 사용한다.

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.2.0</version>
    <configuration>
        <nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <formats>
            <format>HTML</format>
            <format>JUNIT</format>
            <format>JSON</format>
        </formats>
        <suppressionFiles>
            <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
        </suppressionFiles>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```xml
<!-- dependency-check-suppressions.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
    <suppress>
        <notes>H2 인메모리 DB는 테스트 스코프에만 사용되며 프로덕션 미적용</notes>
        <packageUrl regex="true">^pkg:maven/com\.h2database/h2@.*$</packageUrl>
        <cve>CVE-2022-45868</cve>
    </suppress>
</suppressions>
```

### 취약점 심각도 기준

| CVSS 점수 | 심각도 | 대응 기준 |
|-----------|--------|-----------|
| 9.0 ~ 10.0 | Critical | 24시간 이내 패치 |
| 7.0 ~ 8.9 | High | 1주일 이내 패치 |
| 4.0 ~ 6.9 | Medium | 다음 스프린트 내 처리 |
| 0.1 ~ 3.9 | Low | 분기별 정기 패치 |

---

## 3. 침투 테스트와 Threat Modeling (STRIDE)

### Threat Modeling: STRIDE 방법론

STRIDE는 Microsoft가 개발한 Threat Modeling 프레임워크로, 시스템의 각 구성 요소에 대해 6가지 위협 유형을 체계적으로 분석한다.

| 위협 유형 | 설명 | 예시 |
|-----------|------|------|
| **S**poofing (스푸핑) | 다른 사용자/시스템으로 위장 | JWT 토큰 위조, IP 스푸핑 |
| **T**ampering (변조) | 데이터/코드 무단 수정 | 파라미터 변조, 메시지 조작 |
| **R**epudiation (부인) | 행위를 한 사실을 부정 | 로그 삭제, 감사 추적 미비 |
| **I**nformation Disclosure (정보 노출) | 권한 없는 정보 접근 | 스택 트레이스, 디렉토리 리스팅 |
| **D**enial of Service (서비스 거부) | 서비스 가용성 저하 | 요청 폭주, 자원 고갈 |
| **E**levation of Privilege (권한 상승) | 낮은 권한에서 높은 권한 획득 | 수평/수직 권한 상승 |

#### 실전 Threat Modeling 프로세스

**Step 1: 시스템 다이어그램 작성**

```
[브라우저] --HTTPS--> [API Gateway] --내부망--> [Auth Service]
                           |                         |
                           v                         v
                      [App Server] ---------> [Database]
                           |
                           v
                      [Message Queue] --> [Worker Service]
```

**Step 2: 각 구성 요소별 STRIDE 위협 식별**

API Gateway에 대한 분석 예시:
- Spoofing: JWT 토큰 위조 → 대응: 서명 검증, 짧은 만료 시간
- Tampering: 요청 본문 변조 → 대응: HTTPS, 요청 서명(HMAC)
- Repudiation: 요청 기록 없음 → 대응: 구조화 로깅, 감사 로그
- Information Disclosure: 오류 메시지에 내부 구조 노출 → 대응: 오류 메시지 정제
- Denial of Service: Rate Limit 미적용 → 대응: API Gateway 수준 Rate Limiting
- Elevation of Privilege: 관리자 엔드포인트 노출 → 대응: 네트워크 분리, IP 허용 목록

**Step 3: 위험 우선순위 책정 (DREAD 모델)**

```
위험도 = (Damage + Reproducibility + Exploitability + Affected Users + Discoverability) / 5
```

각 항목을 1~10으로 평가해 우선순위를 결정한다.

### 침투 테스트 기본 개념

침투 테스트(Penetration Testing)는 실제 공격자의 관점에서 시스템의 취약점을 찾는 작업이다.

#### 유형별 특징

- **블랙박스 테스트**: 내부 정보 없이 외부 공격자 관점에서 수행. 실제 공격 시나리오에 가깝다.
- **화이트박스 테스트**: 소스코드, 아키텍처 문서를 보유한 상태에서 수행. 내부 취약점까지 식별 가능.
- **그레이박스 테스트**: 부분적인 정보만 제공. 내부 위협(악의적 내부자)을 시뮬레이션.

#### OWASP ZAP을 활용한 자동화 스캔

```bash
# Docker로 OWASP ZAP 실행
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
    -t https://staging.example.com \
    -g gen.conf \
    -r zap-report.html \
    --auto

# CI/CD 통합 (GitHub Actions)
```

```yaml
# .github/workflows/zap-scan.yml
- name: ZAP Scan
  uses: zaproxy/action-full-scan@v0.10.0
  with:
    target: 'https://staging.example.com'
    rules_file_name: '.zap/rules.tsv'
    cmd_options: '-a'
```

```tsv
# .zap/rules.tsv (규칙별 임계값 설정)
10202	IGNORE	(Absence of Anti-CSRF Tokens)
10038	WARN	(Content Security Policy (CSP) Header Not Set)
10020	FAIL	(X-Frame-Options Header)
```

#### 정기 침투 테스트 체크리스트

```markdown
## 인증/인가
- [ ] 비밀번호 브루트포스 가능 여부
- [ ] JWT 알고리즘 변경 공격 (alg: none, RS256 -> HS256)
- [ ] 세션 고정(Session Fixation) 취약점
- [ ] 수평적 권한 상승 (다른 사용자 리소스 접근)
- [ ] 수직적 권한 상승 (일반 사용자 → 관리자)

## 입력 검증
- [ ] SQL Injection (Union-based, Blind)
- [ ] XSS (Stored, Reflected, DOM-based)
- [ ] XXE (XML External Entity)
- [ ] Path Traversal (../../../etc/passwd)

## API 보안
- [ ] Rate Limiting 우회
- [ ] Mass Assignment 취약점
- [ ] GraphQL Introspection 노출
- [ ] HTTP 메서드 변조

## 설정 오류
- [ ] 디버그 엔드포인트 노출
- [ ] 기본 자격증명(default credentials)
- [ ] 불필요한 포트 오픈
- [ ] CORS 설정 오류
```

---

## 4. 보안 로그 설계, 이상 탐지, 사고 대응

### 보안 로그 설계

좋은 보안 로그는 공격을 탐지하고, 침해 후 포렌식(forensics)을 가능하게 하며, 부인 방지를 제공한다.

#### 로그에 반드시 포함해야 할 필드

```java
@Slf4j
@Component
public class SecurityAuditLogger {

    public void logSecurityEvent(SecurityEvent event) {
        // 구조화 로깅 (JSON)
        log.info(StructuredArguments.entries(Map.of(
            "event_type", event.getType(),        // LOGIN_SUCCESS, LOGIN_FAILURE, etc.
            "user_id", event.getUserId(),          // 사용자 식별자
            "session_id", event.getSessionId(),    // 세션 ID
            "ip_address", event.getIpAddress(),    // 클라이언트 IP
            "user_agent", event.getUserAgent(),    // 브라우저/클라이언트 정보
            "resource", event.getResource(),       // 접근한 리소스
            "action", event.getAction(),           // CREATE, READ, UPDATE, DELETE
            "result", event.getResult(),           // SUCCESS, FAILURE, DENIED
            "timestamp", Instant.now().toString(), // ISO-8601 타임스탬프
            "request_id", event.getRequestId(),    // 분산 추적 ID
            "geo_country", event.getGeoCountry()   // 지리 정보
        )));
    }
}
```

```java
public enum SecurityEventType {
    // 인증 이벤트
    LOGIN_SUCCESS,
    LOGIN_FAILURE,
    LOGIN_BLOCKED,
    LOGOUT,
    PASSWORD_CHANGED,
    PASSWORD_RESET_REQUESTED,
    MFA_ENABLED,
    MFA_BYPASS_ATTEMPTED,

    // 권한 이벤트
    ACCESS_DENIED,
    PRIVILEGE_ESCALATION_ATTEMPTED,

    // 데이터 이벤트
    SENSITIVE_DATA_ACCESSED,
    BULK_EXPORT_PERFORMED,

    // 설정 이벤트
    USER_CREATED,
    USER_DELETED,
    ROLE_CHANGED,
    SYSTEM_CONFIG_CHANGED
}
```

#### Spring Security Audit Event 연동

```java
@Component
public class SecurityAuditEventListener implements ApplicationListener<AbstractAuthenticationEvent> {

    private final SecurityAuditLogger auditLogger;

    @Override
    public void onApplicationEvent(AbstractAuthenticationEvent event) {
        SecurityEvent.Builder builder = SecurityEvent.builder()
            .requestId(MDC.get("requestId"))
            .ipAddress(getCurrentIp());

        if (event instanceof AuthenticationSuccessEvent success) {
            String username = success.getAuthentication().getName();
            auditLogger.logSecurityEvent(builder
                .type(SecurityEventType.LOGIN_SUCCESS)
                .userId(username)
                .result("SUCCESS")
                .build());

        } else if (event instanceof AuthenticationFailureBadCredentialsEvent failure) {
            String username = (String) failure.getAuthentication().getPrincipal();
            auditLogger.logSecurityEvent(builder
                .type(SecurityEventType.LOGIN_FAILURE)
                .userId(username)
                .result("FAILURE")
                .details("Bad credentials")
                .build());
        }
    }
}
```

#### 로그 탬퍼링 방지

보안 로그 자체가 변조되면 감사 추적이 의미 없어진다.

```java
// HMAC 서명으로 로그 무결성 보장
@Component
public class TamperProofLogger {

    private final SecretKey hmacKey;

    public void log(Map<String, Object> fields) {
        String payload = objectMapper.writeValueAsString(fields);
        String signature = computeHmac(payload, hmacKey);

        fields.put("_sig", signature);
        fields.put("_prev_hash", getLastLogHash()); // 체인 연결

        log.info(objectMapper.writeValueAsString(fields));
        updateLastLogHash(computeHash(payload));
    }
}
```

### 이상 탐지 패턴

#### 규칙 기반 탐지

```java
@Service
public class AnomalyDetectionService {

    // 1. 비정상적인 로그인 시간 탐지
    public boolean isAbnormalLoginTime(String userId, LocalTime loginTime) {
        UserLoginPattern pattern = patternRepository.findByUserId(userId);
        LocalTime usualStart = pattern.getUsualLoginStart(); // 예: 09:00
        LocalTime usualEnd = pattern.getUsualLoginEnd();     // 예: 22:00

        boolean isOffHours = loginTime.isBefore(usualStart) || loginTime.isAfter(usualEnd);
        if (isOffHours) {
            alertService.sendAlert(AlertLevel.MEDIUM,
                "Unusual login time for user: " + userId + " at " + loginTime);
        }
        return isOffHours;
    }

    // 2. 불가능한 여행(Impossible Travel) 탐지
    public boolean isImpossibleTravel(String userId, String newCountry, Instant loginTime) {
        LoginEvent lastLogin = loginEventRepository.findLastByUserId(userId);

        if (lastLogin == null) return false;

        long minutesBetween = Duration.between(lastLogin.getTimestamp(), loginTime).toMinutes();
        double distanceKm = geoService.calculateDistance(lastLogin.getCountry(), newCountry);

        // 비행기로 이동 불가능한 거리 (최대 속도 900km/h 기준)
        double maxPossibleDistance = (minutesBetween / 60.0) * 900;

        if (distanceKm > maxPossibleDistance) {
            alertService.sendAlert(AlertLevel.HIGH,
                String.format("Impossible travel detected for %s: %s -> %s in %d minutes",
                    userId, lastLogin.getCountry(), newCountry, minutesBetween));
            return true;
        }
        return false;
    }

    // 3. 대량 데이터 접근 탐지
    @Scheduled(fixedRate = 60000) // 1분마다
    public void detectBulkAccess() {
        Instant since = Instant.now().minus(5, ChronoUnit.MINUTES);

        Map<String, Long> accessCountByUser = auditLogRepository
            .countAccessesByUserSince(since);

        accessCountByUser.forEach((userId, count) -> {
            if (count > 1000) { // 5분 내 1000건 이상
                alertService.sendAlert(AlertLevel.HIGH,
                    "Bulk data access by user: " + userId + " (" + count + " requests in 5 min)");
            }
        });
    }
}
```

#### 임계값 기반 알림 설정

```yaml
# application-security.yml
security:
  anomaly-detection:
    login-failure-threshold: 5        # 5회 실패 시 계정 잠금
    login-failure-window-minutes: 15  # 15분 내
    bulk-access-threshold: 1000       # 5분 내 1000건
    new-country-alert: true           # 새 국가에서 로그인 시 알림
    off-hours-alert: true             # 업무 외 시간 로그인 알림
  alerting:
    slack-webhook: ${SLACK_SECURITY_WEBHOOK}
    pagerduty-key: ${PAGERDUTY_INTEGRATION_KEY}
    critical-threshold: HIGH          # HIGH 이상은 PagerDuty 즉시 알림
```

### 사고 대응 플로우 (Incident Response)

체계적인 사고 대응 절차가 없으면 침해 발생 시 패닉에 빠져 잘못된 대응을 하게 된다.

#### IR 5단계 프로세스

```
1. 준비(Preparation)
   └─ IR 팀 구성, Runbook 작성, 통신 채널 설정, 포렌식 도구 준비

2. 탐지 & 분석(Detection & Analysis)
   └─ 이상 탐지 알림 수신 → 초기 분류(triage) → 심각도 판단

3. 봉쇄(Containment)
   ├─ 단기: 공격자 계정 비활성화, 해당 IP 차단
   └─ 장기: 취약 시스템 격리, 백업에서 복구 준비

4. 제거(Eradication)
   └─ 악성코드 제거, 취약점 패치, 자격증명 전체 교체

5. 복구(Recovery)
   └─ 서비스 재개, 모니터링 강화, 재발 여부 확인

6. 교훈(Lessons Learned)
   └─ 사후 분석 보고서 작성, 프로세스 개선
```

#### 계정 침해 대응 Runbook

```java
@Service
public class IncidentResponseService {

    // 계정 침해 의심 시 즉각 실행
    @Transactional
    public void initiateAccountCompromiseResponse(String userId, String incidentId) {
        log.warn("Initiating account compromise response for userId={}, incidentId={}",
            userId, incidentId);

        // 1. 즉각적인 세션 무효화
        sessionManager.invalidateAllSessions(userId);

        // 2. 계정 잠금
        userService.lockAccount(userId, "Security incident: " + incidentId);

        // 3. 모든 활성 토큰 블랙리스트 등록
        tokenBlacklistService.revokeAllTokensForUser(userId);

        // 4. API 키 비활성화
        apiKeyService.deactivateAllKeysForUser(userId);

        // 5. 최근 1시간 활동 로그 수집 (포렌식용)
        List<AuditLog> recentActivities = auditLogService
            .findByUserIdSince(userId, Instant.now().minus(1, ChronoUnit.HOURS));
        incidentService.attachForensicData(incidentId, recentActivities);

        // 6. 보안팀 즉시 알림
        alertService.sendCriticalAlert(
            "Account compromised: " + userId + " | Incident: " + incidentId,
            "All sessions invalidated, account locked. Review forensic data."
        );

        // 7. 감사 로그 기록
        auditLogger.logSecurityEvent(SecurityEvent.builder()
            .type(SecurityEventType.ACCOUNT_COMPROMISED)
            .userId(userId)
            .details("Incident ID: " + incidentId)
            .build());
    }
}
```

#### 사후 분석(Post-Mortem) 템플릿

```markdown
## 보안 인시던트 보고서

**인시던트 ID**: SEC-2026-001
**발생 일시**: 2026-03-15 02:34 KST
**감지 일시**: 2026-03-15 02:41 KST (탐지 지연: 7분)
**해결 일시**: 2026-03-15 04:15 KST (해결 소요: 94분)
**심각도**: HIGH
**영향 범위**: 사용자 123명의 개인정보 조회 시도

### 타임라인
- 02:34 - 공격자가 취약한 API 엔드포인트 발견
- 02:37 - 자동화된 데이터 스크래핑 시작
- 02:41 - 이상 탐지 시스템 알림 발생
- 02:45 - 온콜 엔지니어 대응 시작
- 03:02 - 해당 엔드포인트 차단
- 04:15 - 취약점 패치 및 서비스 재개

### 근본 원인
페이지네이션 API에서 사용자 ID 순차 탐색 가능
Rate Limiting이 해당 엔드포인트에 미적용

### 재발 방지 대책
- [ ] 모든 목록 API에 Rate Limiting 적용
- [ ] 순차적 ID 대신 UUID 사용으로 전환
- [ ] 대량 접근 탐지 임계값 하향 조정 (1000 → 200)
- [ ] 침투 테스트에 IDOR 항목 추가
```

---

## 5. 지속적인 보안 체계 구축

### 보안을 DevOps에 통합 (DevSecOps)

```yaml
# 완전한 보안 파이프라인
stages:
  - name: "Commit Stage"
    tools:
      - SAST: "SonarQube, SpotBugs"
      - Secret Scan: "Gitleaks, truffleHog"
      - License Check: "FOSSA"

  - name: "Build Stage"
    tools:
      - Dependency Scan: "Snyk, OWASP Dependency-Check"
      - Container Scan: "Trivy, Grype"

  - name: "Test Stage"
    tools:
      - DAST: "OWASP ZAP"
      - API Security: "42Crunch"

  - name: "Deploy Stage"
    tools:
      - Infrastructure Scan: "Checkov, tfsec"
      - Compliance Check: "OPA (Open Policy Agent)"
```

### 비밀값 관리 (Secret Management)

```java
// 안티패턴: 소스코드에 시크릿 하드코딩
@Value("${aws.secretKey:AKIAIOSFODNN7EXAMPLE}")
private String awsSecretKey;

// 올바른 패턴: Vault 또는 AWS Secrets Manager 연동
@Configuration
public class SecretsConfig {

    // Spring Cloud Vault 사용
    @Bean
    @ConfigurationProperties(prefix = "secret")
    public SecretsProperties secretsProperties() {
        return new SecretsProperties();
    }
}
```

```yaml
# bootstrap.yml (Spring Cloud Vault)
spring:
  cloud:
    vault:
      host: vault.internal.example.com
      port: 8200
      scheme: https
      authentication: AWS_IAM
      aws-iam:
        role: app-role
      kv:
        enabled: true
        default-context: application
        backend: secret
```

비밀값이 코드에 커밋되는 것을 방지하는 pre-commit 훅:

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Gitleaks로 시크릿 스캔
if command -v gitleaks &> /dev/null; then
    gitleaks detect --staged --no-git -v
    if [ $? -ne 0 ]; then
        echo "ERROR: Potential secrets detected. Commit blocked."
        echo "Review the output above and remove secrets before committing."
        exit 1
    fi
fi
```

---

보안 감사는 분기에 한 번 하는 이벤트가 아니다. 코드가 머지될 때마다, 의존성이 업데이트될 때마다, 아키텍처가 변경될 때마다 보안 체계가 함께 진화해야 한다. OWASP Top 10은 시작점이고, Threat Modeling은 설계 단계의 나침반이며, 자동화된 스캔과 이상 탐지는 일상적인 방어선이다. 사고는 반드시 발생한다. 중요한 것은 발생했을 때 얼마나 빠르게 탐지하고, 얼마나 체계적으로 대응하느냐다.

---

## 참고 자료

1. [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/) — OWASP 공식 Top 10 문서
2. [Microsoft Threat Modeling Tool & STRIDE](https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool) — STRIDE 방법론 공식 가이드
3. [Snyk Developer Security Platform](https://snyk.io/learn/application-security/vulnerability-management/) — 의존성 취약점 관리 실전 가이드
4. [NIST SP 800-61: Computer Security Incident Handling Guide](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r2.pdf) — 미국 국립표준기술연구소의 사고 대응 가이드
5. [OWASP Application Security Verification Standard (ASVS)](https://owasp.org/www-project-application-security-verification-standard/) — 애플리케이션 보안 검증 기준
6. [Spring Security Reference Documentation](https://docs.spring.io/spring-security/reference/) — Spring Security 공식 레퍼런스
