---
title: "[배포와 CI/CD] 4편 — 설정 관리와 환경 분리: 코드와 설정을 분리하는 법"
date: 2026-03-17T20:04:00+09:00
draft: false
tags: ["설정 관리", "12-Factor", "Vault", "Feature Flag", "서버"]
series: ["배포와 CICD"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 6
summary: "12-Factor App 설정 원칙, 환경변수 vs 설정 파일, 환경 분리(dev/staging/prod)와 설정 검증, 시크릿 관리(Vault, AWS Secrets Manager), Feature Flag와 동적 설정 변경까지"
---

프로덕션 장애의 상당수는 코드 버그가 아니라 잘못된 설정에서 비롯된다.

개발 환경에서 잘 돌아가던 애플리케이션이 스테이징에서 데이터베이스 연결에 실패하거나, 운영 배포 직후 API 키가 잘못 주입되어 결제 시스템이 멈추는 사고는 누구나 한 번쯤 겪는다.

설정 관리는 "어디에 값을 넣느냐"의 문제가 아니다.

코드와 설정의 경계를 어디에 그을 것인지, 환경마다 어떻게 격리할 것인지, 비밀 값을 어떻게 안전하게 다룰 것인지, 그리고 배포 없이도 시스템 동작을 어떻게 바꿀 것인지를 설계하는 일이다.

이 글에서는 12-Factor App이 제시하는 원칙부터 Vault, AWS Secrets Manager를 이용한 시크릿 관리, Feature Flag 기반의 동적 설정 변경까지 실전 예제와 함께 살펴본다.

---

## 1. 12-Factor App 설정 원칙

### 설정이란 무엇인가

[The Twelve-Factor App](https://12factor.net/ko/)의 세 번째 원칙은 "설정을 환경(environment)에 저장하라"이다. 여기서 설정의 정의는 명확하다. **배포 환경(dev, staging, prod)마다 달라질 수 있는 모든 것**이 설정이다.

- 데이터베이스 주소, 포트, 자격증명
- 외부 서비스 API 키, 엔드포인트
- 캐시 서버 주소
- 기능 활성화 플래그
- 스레드 풀 크기, 타임아웃 값

반대로 설정이 아닌 것도 있다. 로깅 포맷 지정, 클래스 라우팅 규칙, 언어별 프레임워크 설정처럼 모든 환경에서 동일하게 동작해야 하는 값은 코드(혹은 코드와 함께 버전 관리되는 파일)에 두는 것이 맞다.

### 핵심 원칙: 설정은 코드와 완전히 분리되어야 한다

12-Factor의 원칙을 한 줄로 요약하면 이렇다. 코드베이스는 어떤 자격증명도 노출하지 않은 채 오픈소스로 공개해도 무방해야 한다.

이 기준을 통과하지 못하는 프로젝트는 설정과 코드가 혼재하고 있다는 신호다.

```java
// 안티패턴: 자격증명이 코드에 하드코딩
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://prod-db.internal:5432/myapp")
            .username("app_user")
            .password("supersecret123")  // 절대 금지
            .build();
    }
}
```

```java
// 올바른 패턴: 환경에서 설정을 주입
@Configuration
public class DataSourceConfig {
    @Value("${spring.datasource.url}")
    private String url;

    @Value("${spring.datasource.username}")
    private String username;

    @Value("${spring.datasource.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url(url)
            .username(username)
            .password(password)
            .build();
    }
}
```

---

## 2. 환경변수 vs 설정 파일: 언제 무엇을 쓸까

### 환경변수

환경변수는 12-Factor가 권장하는 기본 설정 전달 방식이다.

OS 수준에서 프로세스에 주입되므로 언어, 프레임워크에 독립적이고 컨테이너 환경(Docker, Kubernetes)에서 자연스럽게 지원된다.

**장점:**
- 런타임 주입이므로 코드베이스에 값이 노출되지 않는다
- 컨테이너 오케스트레이션 도구에서 일급 지원
- 프로세스 격리 경계를 넘어가지 않는다

**단점:**
- 설정 항목이 많아지면 관리가 번거롭다
- 타입 검증이 없다 (모든 값이 문자열)
- 계층적(hierarchical) 구조 표현이 어렵다

```bash
# 환경변수 직접 설정 예시
export DATABASE_URL="jdbc:postgresql://localhost:5432/myapp"
export DATABASE_USERNAME="dev_user"
export DATABASE_PASSWORD="dev_password"
export REDIS_HOST="localhost"
export REDIS_PORT="6379"
export FEATURE_NEW_CHECKOUT="false"
```

### 설정 파일 (YAML / Properties)

Spring Boot를 비롯한 대부분의 프레임워크는 YAML이나 properties 파일을 통한 설정을 지원한다.

설정 파일의 진정한 가치는 **구조와 기본값을 문서화**하는 데 있다.

```yaml
# application.yml - 기본값과 구조 문서화 역할
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/myapp_dev}
    username: ${DATABASE_USERNAME:dev_user}
    password: ${DATABASE_PASSWORD:dev_password}
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      connection-timeout: ${DB_CONN_TIMEOUT:30000}

  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

app:
  feature:
    new-checkout: ${FEATURE_NEW_CHECKOUT:false}
  timeout:
    api-call-ms: ${API_TIMEOUT_MS:5000}
```

`${VAR:default}` 문법으로 환경변수와 기본값을 함께 표현한다.

이 패턴의 핵심은 **파일이 구조와 기본값을 정의하고, 실제 민감한 값은 환경변수로 주입**한다는 점이다.

설정 파일 자체는 버전 관리에 포함해도 안전하다.

### 어느 것을 선택해야 하는가

환경변수와 설정 파일은 상호 배타적인 선택이 아니다. 각각의 강점이 다르므로 조합해서 쓰는 것이 실전 패턴이다.

| 기준 | 환경변수 | 설정 파일 |
|---|---|---|
| 민감한 자격증명 | 적합 | 부적합 (버전 관리 위험) |
| 구조적·계층적 설정 | 번거로움 | 적합 |
| 기본값 문서화 | 어려움 | 적합 |
| 컨테이너 환경 | 자연스러움 | 추가 마운트 필요 |
| 타입 안전성 | 없음 | 프레임워크 지원 |

실전 권장 패턴: **설정 파일로 구조와 기본값을 정의하고, 환경별 민감한 값은 환경변수로 오버라이드한다.**

---

## 3. 환경 분리: dev / staging / prod

### Spring Boot Profile을 이용한 환경 분리

Spring Boot는 Profile 기능으로 환경별 설정 파일을 지원한다.

```
src/main/resources/
├── application.yml           # 공통 설정 (모든 환경)
├── application-dev.yml       # 개발 환경
├── application-staging.yml   # 스테이징 환경
└── application-prod.yml      # 운영 환경
```

```yaml
# application.yml (공통)
spring:
  application:
    name: myapp
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

logging:
  level:
    root: INFO
    com.mycompany: INFO

app:
  cors:
    allowed-origins: ${CORS_ALLOWED_ORIGINS}
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp_dev
    username: dev_user
    password: dev_password
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update

logging:
  level:
    com.mycompany: DEBUG

app:
  cors:
    allowed-origins: "http://localhost:3000"
```

```yaml
# application-prod.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate

logging:
  level:
    com.mycompany: WARN

app:
  cors:
    allowed-origins: ${CORS_ALLOWED_ORIGINS}
```

Profile 활성화는 환경변수로 지정한다:

```bash
SPRING_PROFILES_ACTIVE=prod java -jar myapp.jar
```

### Kubernetes에서의 설정 분리

Kubernetes 환경에서는 ConfigMap과 Secret으로 설정을 분리한다.

```yaml
# configmap-prod.yaml - 비민감 설정
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  SPRING_PROFILES_ACTIVE: "prod"
  DB_POOL_SIZE: "20"
  REDIS_HOST: "redis.production.svc.cluster.local"
  REDIS_PORT: "6379"
  API_TIMEOUT_MS: "3000"
```

```yaml
# secret-prod.yaml - 민감한 자격증명
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
data:
  DATABASE_URL: <base64-encoded-value>
  DATABASE_USERNAME: <base64-encoded-value>
  DATABASE_PASSWORD: <base64-encoded-value>
```

```yaml
# deployment.yaml - Pod에 설정 주입
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
```

### 안티패턴: 환경별 분기 코드

```java
// 안티패턴: 코드에서 환경 분기
@Service
public class PaymentService {
    public void processPayment(Payment payment) {
        String env = System.getenv("APP_ENV");
        if ("prod".equals(env)) {
            // 실제 결제 처리
            realPaymentGateway.process(payment);
        } else {
            // 개발 환경에서 mock
            mockPaymentGateway.process(payment);
        }
    }
}
```

```java
// 올바른 패턴: 인터페이스 추상화 + Profile
public interface PaymentGateway {
    PaymentResult process(Payment payment);
}

@Service
@Profile("prod")
public class RealPaymentGateway implements PaymentGateway {
    public PaymentResult process(Payment payment) { /* 실제 구현 */ }
}

@Service
@Profile("!prod")
public class MockPaymentGateway implements PaymentGateway {
    public PaymentResult process(Payment payment) { /* Mock 구현 */ }
}
```

---

## 4. 설정 검증: 애플리케이션 시작 시 실패를 빠르게

### @ConfigurationProperties로 타입 안전하게

Spring Boot의 `@ConfigurationProperties`를 사용하면 시작 시점에 설정 값을 검증할 수 있다.

```java
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {

    @NotNull
    @Valid
    private Database database = new Database();

    @NotNull
    @Valid
    private Feature feature = new Feature();

    @Valid
    private Timeout timeout = new Timeout();

    @Data
    public static class Database {
        @NotBlank(message = "데이터베이스 URL은 필수입니다")
        private String url;

        @NotBlank
        private String username;

        @NotBlank
        private String password;

        @Min(value = 1, message = "풀 크기는 1 이상이어야 합니다")
        @Max(value = 100, message = "풀 크기는 100 이하여야 합니다")
        private int poolSize = 10;
    }

    @Data
    public static class Feature {
        private boolean newCheckout = false;
        private boolean experimentalSearch = false;
    }

    @Data
    public static class Timeout {
        @Min(100)
        @Max(30000)
        private int apiCallMs = 5000;

        @Min(100)
        private int dbQueryMs = 3000;
    }
}
```

```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

이 방식을 사용하면 필수 설정이 누락되거나 값 범위가 잘못된 경우 애플리케이션이 시작을 거부한다.

"Fail Fast" 원칙을 설정 계층에서 구현하는 것이다.

### 설정 헬스체크

운영 환경에서는 설정이 실제로 동작하는지 확인하는 헬스체크도 중요하다.

```java
@Component
public class ConfigHealthIndicator implements HealthIndicator {

    private final AppProperties appProperties;
    private final DataSource dataSource;

    public ConfigHealthIndicator(AppProperties appProperties, DataSource dataSource) {
        this.appProperties = appProperties;
        this.dataSource = dataSource;
    }

    @Override
    public Health health() {
        try {
            // 실제 DB 연결 확인
            try (var conn = dataSource.getConnection()) {
                if (!conn.isValid(2)) {
                    return Health.down()
                        .withDetail("database", "연결 유효성 검사 실패")
                        .build();
                }
            }
            return Health.up()
                .withDetail("database.poolSize", appProperties.getDatabase().getPoolSize())
                .withDetail("feature.newCheckout", appProperties.getFeature().isNewCheckout())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

---

## 5. 시크릿 관리: Vault와 AWS Secrets Manager

하드코딩과 환경변수 방식 모두 한계가 있다. 환경변수도 프로세스 목록을 조회하거나 로그에서 노출될 위험이 있고, 자격증명 로테이션이 어렵다.

이 문제를 해결하기 위해 전용 시크릿 관리 시스템을 사용한다.

### HashiCorp Vault

Vault는 시크릿의 저장, 접근 제어, 자동 로테이션을 담당하는 시크릿 관리 플랫폼이다.

**Vault 시크릿 저장:**

```bash
# KV (Key-Value) 시크릿 엔진에 시크릿 저장
vault kv put secret/myapp/prod \
    database_url="jdbc:postgresql://prod-db:5432/myapp" \
    database_username="app_user" \
    database_password="$(openssl rand -base64 32)"

# 저장된 시크릿 확인
vault kv get secret/myapp/prod
```

**Spring Boot + Vault 통합:**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml (Spring Cloud Config)
spring:
  cloud:
    vault:
      host: vault.internal.mycompany.com
      port: 8200
      scheme: https
      authentication: KUBERNETES
      kubernetes:
        role: myapp-role
        kubernetes-path: kubernetes
      kv:
        enabled: true
        backend: secret
        default-context: myapp/prod
```

Kubernetes 환경에서 Vault는 서비스 어카운트 토큰으로 인증하므로, 애플리케이션 코드에 Vault 자격증명을 별도로 관리하지 않아도 된다.

**Vault Policy 설정:**

```hcl
# myapp-policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

path "secret/metadata/myapp/*" {
  capabilities = ["list", "read"]
}
```

```bash
# 정책 적용 및 Kubernetes 역할 생성
vault policy write myapp-policy myapp-policy.hcl

vault write auth/kubernetes/role/myapp-role \
    bound_service_account_names=myapp-service-account \
    bound_service_account_namespaces=production \
    policies=myapp-policy \
    ttl=1h
```

### Vault Agent Sidecar (Kubernetes)

애플리케이션 코드 변경 없이 Vault 시크릿을 파일로 주입하는 방식이다.

```yaml
# deployment.yaml with Vault Agent annotation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-role"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/myapp/prod"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/myapp/prod" -}}
          DATABASE_URL={{ .Data.data.database_url }}
          DATABASE_USERNAME={{ .Data.data.database_username }}
          DATABASE_PASSWORD={{ .Data.data.database_password }}
          {{- end }}
    spec:
      serviceAccountName: myapp-service-account
      containers:
        - name: myapp
          image: myapp:latest
          command: ["sh", "-c"]
          args:
            - |
              source /vault/secrets/config
              java -jar /app/myapp.jar
```

### AWS Secrets Manager

AWS 환경에서는 Secrets Manager가 Vault와 유사한 역할을 한다.

```java
// AWS SDK로 시크릿 읽기
@Configuration
public class AwsSecretsConfig {

    @Bean
    public SecretsManagerClient secretsManagerClient() {
        return SecretsManagerClient.builder()
            .region(Region.AP_NORTHEAST_2)
            .credentialsProvider(DefaultCredentialsProvider.create())
            .build();
    }

    @Bean
    public DatabaseCredentials databaseCredentials(SecretsManagerClient client) {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
            .secretId("myapp/prod/database")
            .build();

        GetSecretValueResponse response = client.getSecretValue(request);
        ObjectMapper mapper = new ObjectMapper();

        try {
            return mapper.readValue(response.secretString(), DatabaseCredentials.class);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("시크릿 파싱 실패", e);
        }
    }
}

@Data
public class DatabaseCredentials {
    private String username;
    private String password;
    private String host;
    private int port;
    private String dbname;

    public String toJdbcUrl() {
        return String.format("jdbc:postgresql://%s:%d/%s", host, port, dbname);
    }
}
```

AWS 환경에서는 IAM 역할 기반 인증(IRSA - IAM Roles for Service Accounts)을 사용하면 Kubernetes Pod에 별도 자격증명 없이 Secrets Manager에 접근할 수 있다.

```yaml
# IAM 역할 어노테이션 (EKS)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/myapp-secrets-role
```

### 시크릿 자동 로테이션

Vault의 Dynamic Secrets 기능을 이용하면 데이터베이스 자격증명을 자동으로 생성하고 만료시킬 수 있다.

```bash
# PostgreSQL Dynamic Secrets 설정
vault secrets enable database

vault write database/config/myapp-db \
    plugin_name=postgresql-database-plugin \
    allowed_roles="myapp-role" \
    connection_url="postgresql://{{username}}:{{password}}@prod-db:5432/myapp?sslmode=require" \
    username="vault_admin" \
    password="vault_admin_password"

vault write database/roles/myapp-role \
    db_name=myapp-db \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

이제 애플리케이션이 자격증명을 요청할 때마다 Vault가 새로운 데이터베이스 사용자를 생성하고, TTL이 만료되면 자동으로 삭제한다.

자격증명이 유출되더라도 최대 1시간만 유효하다.

---

## 6. Feature Flag: 배포와 릴리스를 분리하라

### Feature Flag란 무엇인가

Feature Flag(기능 플래그, 혹은 Feature Toggle)는 코드 배포 없이 특정 기능을 켜고 끌 수 있는 메커니즘이다.

이를 통해 **배포(deployment)**와 **릴리스(release)**를 분리할 수 있다.

- 배포: 코드를 프로덕션 서버에 올리는 행위
- 릴리스: 사용자에게 기능을 공개하는 행위

Feature Flag가 없으면 배포와 릴리스가 동시에 일어난다.

Flag가 있으면 코드를 먼저 배포해두고, 검증이 완료된 시점에 Flag를 켜는 방식으로 리스크를 분리할 수 있다.

### 단순 구현: 설정 파일 기반

```java
@ConfigurationProperties(prefix = "app.feature")
@Data
public class FeatureFlags {
    private boolean newCheckout = false;
    private boolean experimentalSearch = false;
    private boolean betaUserDashboard = false;
}

@RestController
@RequestMapping("/checkout")
public class CheckoutController {

    private final FeatureFlags featureFlags;
    private final NewCheckoutService newCheckoutService;
    private final LegacyCheckoutService legacyCheckoutService;

    @PostMapping
    public ResponseEntity<CheckoutResult> checkout(@RequestBody CheckoutRequest request) {
        if (featureFlags.isNewCheckout()) {
            return ResponseEntity.ok(newCheckoutService.process(request));
        }
        return ResponseEntity.ok(legacyCheckoutService.process(request));
    }
}
```

이 방식은 설정 파일 변경 후 재배포가 필요하므로 동적 변경은 불가능하다.

### 동적 Feature Flag: Spring Cloud Config + 실시간 갱신

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: refresh, health, info
```

```java
@RestController
@RequestMapping("/checkout")
@RefreshScope  // /actuator/refresh 호출 시 Bean 재생성
public class CheckoutController {

    @Value("${app.feature.new-checkout:false}")
    private boolean newCheckoutEnabled;

    @PostMapping
    public ResponseEntity<CheckoutResult> checkout(@RequestBody CheckoutRequest request) {
        // newCheckoutEnabled 값이 동적으로 갱신됨
        if (newCheckoutEnabled) {
            return ResponseEntity.ok(newCheckoutService.process(request));
        }
        return ResponseEntity.ok(legacyCheckoutService.process(request));
    }
}
```

Spring Cloud Config Server를 사용하면 Git에 설정을 저장하고, Actuator refresh 엔드포인트를 호출해 재배포 없이 설정을 갱신할 수 있다.

### 전문 Feature Flag 시스템: LaunchDarkly / Unleash

대규모 서비스에서는 전용 Feature Flag 플랫폼을 사용한다.

**Unleash (오픈소스) 통합:**

```xml
<dependency>
    <groupId>io.getunleash</groupId>
    <artifactId>unleash-client-java</artifactId>
    <version>8.0.0</version>
</dependency>
```

```java
@Configuration
public class UnleashConfig {

    @Bean
    public Unleash unleash() {
        UnleashConfig config = UnleashConfig.builder()
            .appName("myapp")
            .instanceId("myapp-prod-1")
            .unleashAPI("https://unleash.internal.mycompany.com/api")
            .apiKey(System.getenv("UNLEASH_API_KEY"))
            .synchronousFetchOnInitialisation(true)
            .build();

        return new DefaultUnleash(config);
    }
}

@Service
public class CheckoutService {

    private final Unleash unleash;

    public CheckoutResult process(CheckoutRequest request, String userId) {
        // 사용자 컨텍스트 기반 점진적 롤아웃
        UnleashContext context = UnleashContext.builder()
            .userId(userId)
            .addProperty("userTier", getUserTier(userId))
            .build();

        if (unleash.isEnabled("new-checkout", context)) {
            return newCheckoutFlow(request);
        }
        return legacyCheckoutFlow(request);
    }
}
```

Unleash는 다음과 같은 롤아웃 전략을 지원한다:

- **gradualRolloutUserId**: 사용자 ID 해시 기반으로 N%에게만 활성화
- **userWithId**: 특정 사용자 목록에만 활성화
- **remoteAddress**: 특정 IP 범위에서만 활성화
- **applicationHostname**: 특정 호스트에서만 활성화

```bash
# Unleash API로 직접 Flag 제어
curl -X PATCH https://unleash.internal/api/admin/features/new-checkout/environments/production/strategies/1 \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
        "name": "gradualRolloutUserId",
        "parameters": {
            "percentage": "10",
            "groupId": "new-checkout"
        }
    }'
```

### Feature Flag 안티패턴

**1. Flag 중첩 지옥**

```java
// 안티패턴: 플래그가 중첩되어 조합 폭발
if (featureFlags.isNewCheckout()) {
    if (featureFlags.isNewPaymentFlow()) {
        if (featureFlags.isNewInventoryCheck()) {
            // 실제 로직
        }
    }
}
```

**2. Flag를 영원히 남겨두기**

Feature Flag는 유효기간이 있어야 한다. 릴리스가 완료된 Flag는 반드시 제거해야 한다.

Flag가 쌓이면 코드를 이해하기 어려워지고 테스트 복잡도가 기하급수적으로 증가한다.

```java
// Flag 제거 계획을 코드에 명시
@Deprecated(since = "2026-04-01", forRemoval = true)
// TODO: new-checkout 100% 롤아웃 완료 후 이 분기 제거 (2026-04-15)
if (unleash.isEnabled("new-checkout", context)) {
    return newCheckoutFlow(request);
}
return legacyCheckoutFlow(request);
```

**3. Flag를 환경 분기용으로 남용**

```java
// 안티패턴: Feature Flag를 환경 분기로 사용
if (featureFlags.isProductionMode()) {
    // 운영 로직
} else {
    // 개발 로직
}
```

환경 분기는 Profile로, 기능 토글은 Feature Flag로 명확히 역할을 나눠야 한다.

---

## 7. 동적 설정 변경: 무중단으로 설정 바꾸기

### Spring Cloud Config Server

```yaml
# config-server application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mycompany/app-config
          search-paths: "{application}/{profile}"
          default-label: main
```

```
config-repo/
├── myapp/
│   ├── default/
│   │   └── application.yml
│   ├── dev/
│   │   └── application.yml
│   └── prod/
│       └── application.yml
```

Config Server가 Git 저장소를 바라보므로, Git에 설정을 커밋하면 `/actuator/refresh`로 애플리케이션에 반영할 수 있다.

Git을 이용하므로 설정 변경 이력과 롤백이 자동으로 지원된다.

### 설정 변경 감사 로그

운영 환경에서 설정 변경은 반드시 추적 가능해야 한다.

```java
@Component
@EventListener
public class ConfigChangeAuditListener {

    private final AuditLogger auditLogger;

    @EventListener(EnvironmentChangeEvent.class)
    public void onConfigChange(EnvironmentChangeEvent event) {
        Set<String> changedKeys = event.getKeys();
        auditLogger.log(AuditEntry.builder()
            .event("CONFIG_CHANGED")
            .changedKeys(changedKeys)
            .timestamp(Instant.now())
            .host(getHostname())
            .build());

        // 민감한 키 변경 시 알림
        if (changedKeys.stream().anyMatch(k -> k.contains("password") || k.contains("secret"))) {
            alertService.sendSecurityAlert("민감한 설정 키가 변경되었습니다: " + changedKeys);
        }
    }
}
```

---

## 8. 설정 관리 체크리스트

프로덕션에 배포하기 전 확인해야 할 항목들이다.

**코드 분리**
- [ ] 소스코드에 자격증명, API 키가 하드코딩되어 있지 않은가
- [ ] `.gitignore`에 `.env` 파일, 로컬 override 파일이 포함되어 있는가
- [ ] 설정 파일을 오픈소스로 공개해도 안전한가

**환경 분리**
- [ ] dev/staging/prod 환경마다 별도 데이터베이스를 사용하는가
- [ ] staging 환경이 prod 데이터에 접근할 수 없는가
- [ ] 환경별 설정 차이가 코드 분기가 아닌 설정으로 표현되어 있는가

**시크릿 관리**
- [ ] 시크릿은 Vault 또는 클라우드 시크릿 매니저에 저장되어 있는가
- [ ] 시크릿 접근에 최소 권한 원칙이 적용되어 있는가
- [ ] 자격증명 로테이션 계획이 있는가

**설정 검증**
- [ ] 필수 설정 누락 시 시작 시점에 실패하는가 (Fail Fast)
- [ ] 설정 값의 타입과 범위가 검증되는가

**Feature Flag**
- [ ] 릴리스 완료된 Flag 제거 계획이 있는가
- [ ] Flag 변경이 감사 로그에 기록되는가

---

## 마무리

설정 관리는 한 번 잘 설계해두면 수많은 장애와 보안 사고를 예방한다.

핵심 원칙을 다시 정리하면 이렇다. 코드와 설정을 분리하고, 환경마다 격리하며, 시크릿은 전용 시스템에서 관리하고, Feature Flag로 배포와 릴리스를 분리하라.

이 네 가지만 제대로 지켜도 설정 관련 장애의 대부분을 막을 수 있다.

특히 Vault의 Dynamic Secrets와 Feature Flag의 점진적 롤아웃은 실전에서 리스크를 크게 줄여주는 기법이다.

설정도 코드처럼 버전 관리하고, 변경은 추적하고, 배포는 자동화하라.

---

## 참고 자료

1. [The Twelve-Factor App - Config](https://12factor.net/ko/config) — 12-Factor App 설정 원칙 원문
2. [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs) — Vault 공식 문서, Dynamic Secrets, Kubernetes 인증
3. [Spring Boot Externalized Configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config) — Spring Boot 외부 설정 공식 가이드
4. [AWS Secrets Manager User Guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) — AWS 시크릿 관리 공식 문서
5. [Unleash Feature Toggle System](https://docs.getunleash.io/) — 오픈소스 Feature Flag 플랫폼 공식 문서
6. [Feature Toggles (Feature Flags) — Martin Fowler](https://martinfowler.com/articles/feature-toggles.html) — Feature Flag 패턴 심층 분석
