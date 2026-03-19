---
title: "[인증과 보안] 1편 — 인증 기초: 세션, 토큰, 그 선택의 기준"
date: 2026-03-17T22:07:00+09:00
draft: false
tags: ["인증", "세션", "JWT", "토큰", "보안", "서버"]
series: ["인증과 보안"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 4
summary: "세션 기반 인증 동작 원리와 분산 환경 한계, JWT 구조(Header.Payload.Signature)와 서명 검증, Access Token/Refresh Token 전략, 토큰 탈취 대응, 세션 vs JWT 실전 선택 기준까지"
---

로그인은 단순해 보인다. 아이디와 비밀번호를 입력하면 들어가지는 것.

그런데 서버 엔지니어링 관점에서 보면, 그 단순한 행위 뒤에 "이 사람이 누구인지를 어떻게 계속 기억할 것인가"라는 상태 관리 문제가 숨어 있다. HTTP는 본질적으로 무상태(stateless) 프로토콜이다. 요청과 요청 사이에는 아무것도 기억하지 않는다.

그래서 우리는 매 요청마다 사용자를 식별하는 장치가 필요하다. 그 장치가 바로 세션과 토큰이다.

이 두 방식은 서로 다른 철학에서 출발하며, 각각의 트레이드오프가 있다. 이 글에서는 두 방식의 동작 원리를 깊이 파고들고, 실무에서 어떤 기준으로 선택해야 하는지를 명확히 정리한다.

---

## 세션 기반 인증: 서버가 기억하는 방식

### 세션의 동작 원리

세션 기반 인증은 서버가 인증 상태를 직접 보관하는 방식이다. 흐름을 단계별로 보면 이렇다.

```
클라이언트                          서버
   |                                  |
   | --- POST /login (id/pw) -------> |
   |                                  |  1. 자격증명 검증
   |                                  |  2. 세션 생성 (session_id = "abc123")
   |                                  |  3. 세션 저장소에 저장
   |                                  |     { "abc123": { userId: 42, role: "USER" } }
   | <-- Set-Cookie: JSESSIONID=abc123|
   |                                  |
   | --- GET /api/profile ----------> |
   |     Cookie: JSESSIONID=abc123    |  4. 쿠키에서 session_id 추출
   |                                  |  5. 세션 저장소 조회 -> userId=42
   |                                  |  6. 요청 처리
   | <-- 200 OK (프로필 데이터) ------|
```

핵심은 **서버가 상태를 가진다**는 점이다. 클라이언트가 가진 것은 세션 ID뿐이며, 이 ID는 실제 데이터에 대한 참조(reference)에 불과하다. 세션 ID 자체는 아무 의미도 없는 랜덤 문자열이다. 의미는 서버 저장소에 있다.

### Spring Session 구현 예시

Spring Boot에서 세션을 사용하는 전형적인 패턴을 보자.

```java
@RestController
@RequestMapping("/api")
public class AuthController {

    private final UserService userService;

    @PostMapping("/login")
    public ResponseEntity<Void> login(
            @RequestBody LoginRequest request,
            HttpSession session) {

        User user = userService.authenticate(request.getEmail(), request.getPassword());
        // 인증 실패 시 userService에서 예외 throw

        // 세션에 사용자 정보 저장
        session.setAttribute("userId", user.getId());
        session.setAttribute("role", user.getRole());
        session.setMaxInactiveInterval(30 * 60); // 30분 비활성 타임아웃

        return ResponseEntity.ok().build();
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(HttpSession session) {
        session.invalidate(); // 세션 완전 삭제
        return ResponseEntity.ok().build();
    }

    @GetMapping("/profile")
    public ResponseEntity<UserProfile> profile(HttpSession session) {
        Long userId = (Long) session.getAttribute("userId");
        if (userId == null) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        return ResponseEntity.ok(userService.getProfile(userId));
    }
}
```

실무에서는 이 방식보다 Spring Security의 `SecurityContext`를 활용하는 것이 일반적이다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)              // 동시 세션 1개로 제한
                .maxSessionsPreventsLogin(false) // 새 로그인 시 기존 세션 만료
            )
            .formLogin(form -> form
                .loginProcessingUrl("/api/login")
                .successHandler(customSuccessHandler())
            );
        return http.build();
    }
}
```

### 세션 저장소: 메모리에서 Redis까지

기본적으로 서블릿 컨테이너는 세션을 JVM 힙 메모리에 저장한다. 단일 서버 환경에서는 이것으로 충분하지만, 프로덕션 환경에서는 여러 문제가 생긴다.

**메모리 세션의 문제:**
- 서버 재시작 시 모든 세션 소멸
- 세션 객체가 GC 대상이 되지 않으면 메모리 누수
- 가비지 컬렉션 압력 증가

이를 해결하기 위해 Redis 같은 외부 저장소를 쓴다. Spring Session을 사용하면 코드 변경 없이 저장소를 교체할 수 있다.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  session:
    store-type: redis
    redis:
      flush-mode: on-save
      namespace: "myapp:session"
  data:
    redis:
      host: redis-host
      port: 6379
```

이렇게 하면 세션이 Redis에 저장되고, 애플리케이션은 HTTP 요청마다 Redis에서 세션을 조회한다.

---

## 분산 환경의 세션: 어디서 무너지는가

### Sticky Session의 함정

서버가 한 대일 때는 세션이 잘 동작한다. 하지만 로드밸런서 뒤에 서버 3대를 두는 순간 문제가 생긴다.

```
          로드밸런서
         /    |    \
     서버A  서버B  서버C
      [S1]  [S2]  [S3]
```

사용자가 서버A에서 로그인했다면, 그 세션(S1)은 서버A의 메모리에만 있다. 다음 요청이 서버B로 라우팅되면, 서버B는 S1을 모르기 때문에 401 Unauthorized를 반환한다.

가장 흔히 보이는 임시방편이 **Sticky Session(세션 고정)**이다. 로드밸런서가 특정 클라이언트의 요청을 항상 같은 서버로 보내는 방식이다. Nginx에서는 이렇게 설정한다.

```nginx
upstream backend {
    ip_hash;  # IP 기반 고정
    server server-a:8080;
    server server-b:8080;
    server server-c:8080;
}
```

이 방식의 문제는 명백하다.

1. **불균등한 부하**: 특정 서버에 트래픽이 집중될 수 있다
2. **고가용성 파괴**: 서버A가 죽으면 그 서버의 세션을 가진 모든 사용자가 강제 로그아웃된다
3. **스케일아웃 제약**: 새 서버를 추가해도 기존 사용자는 옛 서버에 묶인다
4. **배포 어려움**: 롤링 배포 시 세션이 있는 서버를 내리기 어렵다

### Redis 세션 클러스터링: 올바른 해법

분산 환경에서 세션을 제대로 다루려면 모든 서버가 공유하는 외부 세션 저장소가 필요하다.

```
          로드밸런서
         /    |    \
     서버A  서버B  서버C
          \   |   /
          Redis 클러스터
          [세션 저장소]
```

이 구조에서는 어느 서버에 요청이 가든 Redis에서 동일한 세션을 읽는다. 서버A가 죽어도 세션은 Redis에 살아있으니 서버B나 C가 이어받으면 된다.

**하지만 이 구조도 비용이 있다.** 모든 인증된 요청마다 Redis 네트워크 호출이 발생한다. 세션 조회, 경우에 따라 세션 업데이트(마지막 접근 시간 갱신), 이 두 번의 I/O가 요청 처리 지연에 더해진다. 트래픽이 많은 서비스에서 Redis 세션 서버 자체가 병목이 될 수 있다.

```java
// 세션 조회 비용을 줄이기 위한 캐싱 전략 예시
@Service
public class CachedSessionService {

    private final RedisTemplate<String, SessionData> redisTemplate;
    private final Cache<String, SessionData> localCache;

    public CachedSessionService(RedisTemplate<String, SessionData> redisTemplate) {
        this.redisTemplate = redisTemplate;
        // Caffeine 로컬 캐시: TTL 30초, 최대 10,000 항목
        this.localCache = Caffeine.newBuilder()
            .expireAfterWrite(30, TimeUnit.SECONDS)
            .maximumSize(10_000)
            .build();
    }

    public SessionData getSession(String sessionId) {
        // L1: 로컬 캐시 확인
        SessionData cached = localCache.getIfPresent(sessionId);
        if (cached != null) return cached;

        // L2: Redis 조회
        SessionData session = redisTemplate.opsForValue().get("session:" + sessionId);
        if (session != null) {
            localCache.put(sessionId, session);
        }
        return session;
    }
}
```

이 로컬 캐시 전략은 읽기 비용을 줄이지만, 세션 무효화(로그아웃, 강제 탈퇴)가 즉시 반영되지 않을 수 있다. 보안 요구사항과 성능 요구사항 사이의 트레이드오프다.

---

## JWT: 서버가 기억하지 않는 방식

### JWT의 철학

JWT(JSON Web Token)는 세션과 정반대 발상이다. **서버가 아무것도 기억하지 않는다.** 대신 토큰 자체에 사용자 정보를 담고, 그 무결성을 암호학적 서명으로 보증한다.

서버는 토큰을 받을 때마다 서명을 검증하고, 검증이 통과되면 토큰 안의 데이터를 신뢰한다.

```
클라이언트                          서버
   |                                  |
   | --- POST /login (id/pw) -------> |
   |                                  |  1. 자격증명 검증
   |                                  |  2. JWT 생성 (서명 포함)
   |                                  |  3. 저장소에 아무것도 저장 안 함
   | <-- 200 OK { access_token: ... } |
   |                                  |
   | --- GET /api/profile ----------> |
   |     Authorization: Bearer <JWT>  |  4. JWT 서명 검증
   |                                  |  5. Payload에서 userId 추출
   |                                  |  6. 요청 처리 (DB 세션 조회 없음)
   | <-- 200 OK (프로필 데이터) ------|
```

### JWT 구조 해부

JWT는 점(`.`)으로 구분된 세 부분으로 이루어진다.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJ1c2VySWQiOjQyLCJyb2xlIjoiVVNFUiIsImlhdCI6MTcwMDAwMDAwMCwiZXhwIjoxNzAwMDAzNjAwfQ
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

각 부분은 Base64URL로 인코딩되어 있다. 디코딩하면 이렇다.

**Header (알고리즘 선언):**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload (클레임):**
```json
{
  "userId": 42,
  "role": "USER",
  "iat": 1700000000,
  "exp": 1700003600
}
```

표준 클레임(Registered Claims):
- `iss` (Issuer): 토큰 발급자
- `sub` (Subject): 토큰 주체 (보통 사용자 ID)
- `aud` (Audience): 토큰 수신자
- `exp` (Expiration): 만료 시각 (Unix timestamp)
- `iat` (Issued At): 발급 시각
- `jti` (JWT ID): 토큰 고유 식별자 (replay attack 방지용)

**Signature (서명):**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

### 서명 검증 원리

HMAC-SHA256 방식(대칭키)과 RS256 방식(비대칭키)의 차이를 이해하는 것이 중요하다.

**HMAC-SHA256 (HS256):**
```
서명 생성: HMAC-SHA256(header.payload, secretKey)
서명 검증: HMAC-SHA256(header.payload, secretKey) == 수신한 서명?
```

서명 생성과 검증에 같은 키를 사용한다. 따라서 이 키를 아는 서버만 검증할 수 있고, 키가 노출되면 누구나 유효한 토큰을 위조할 수 있다. 단일 서비스나 내부 마이크로서비스 간 통신에 적합하다.

**RS256 (비대칭키):**
```
서명 생성: RSA-Sign(header.payload, privateKey)    <- 발급 서버만 소유
서명 검증: RSA-Verify(header.payload, publicKey)   <- 누구나 가능
```

비대칭키 방식에서는 서명 검증에 공개키만 있으면 된다. 공개키는 말 그대로 공개해도 된다. 이 방식은 마이크로서비스 아키텍처에서 강력하다. 각 서비스가 인증 서버의 공개키를 가지고 자체적으로 토큰을 검증할 수 있다. 인증 서버에 네트워크 호출 없이.

### Spring Boot JWT 구현

```java
// JWT 유틸리티 클래스
@Component
public class JwtTokenProvider {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration.access:3600}")  // 기본 1시간
    private long accessTokenExpiration;

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateAccessToken(Long userId, String role) {
        return Jwts.builder()
            .subject(userId.toString())
            .claim("role", role)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + accessTokenExpiration * 1000))
            .signWith(getSigningKey())
            .compact();
    }

    public Claims validateAndParseClaims(String token) {
        try {
            return Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
        } catch (ExpiredJwtException e) {
            throw new TokenExpiredException("토큰이 만료되었습니다");
        } catch (JwtException e) {
            throw new InvalidTokenException("유효하지 않은 토큰입니다");
        }
    }
}
```

```java
// JWT 인증 필터
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider tokenProvider;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null) {
            try {
                Claims claims = tokenProvider.validateAndParseClaims(token);
                Long userId = Long.parseLong(claims.getSubject());
                String role = claims.get("role", String.class);

                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userId,
                        null,
                        List.of(new SimpleGrantedAuthority("ROLE_" + role))
                    );
                SecurityContextHolder.getContext().setAuthentication(authentication);

            } catch (TokenExpiredException | InvalidTokenException e) {
                // 토큰 오류 시 SecurityContext를 비워두면
                // 이후 인가 필터에서 401 처리
            }
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (StringUtils.hasText(header) && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}
```

---

## Access Token / Refresh Token 전략

### 왜 두 개의 토큰이 필요한가

JWT의 근본적인 약점이 있다. **토큰은 만료되기 전까지 서버가 막을 방법이 없다.** 세션은 서버에서 삭제하면 즉시 무효화되지만, JWT는 서버가 상태를 가지지 않으므로 "이 토큰은 더 이상 유효하지 않다"는 사실을 전달할 방법이 없다.

그래서 Access Token은 유효기간을 짧게 (15분 ~ 1시간) 가져간다. 짧은 유효기간은 토큰이 탈취되더라도 피해를 시간으로 제한한다. 하지만 그러면 15분마다 재로그인해야 하는 UX 문제가 생긴다.

이 딜레마를 해결하는 것이 Refresh Token이다.

```
Access Token  : 유효기간 짧음 (15분~1시간)  / 모든 API 요청에 사용
Refresh Token : 유효기간 김  (7일~30일)     / Access Token 재발급에만 사용
```

### Refresh Token 흐름

```
클라이언트                              서버
   |                                      |
   | --- POST /auth/login --------------> |
   | <-- { access_token, refresh_token } -|
   |                                      |
   |   ... 15분 경과 ...                  |
   |                                      |
   | --- GET /api/data ----------------> |
   |     Authorization: Bearer <expired> |  access_token 만료 -> 401 반환
   | <-- 401 Unauthorized --------------- |
   |                                      |
   | --- POST /auth/refresh -----------> |
   |     { refresh_token: "..." }        |  refresh_token 검증
   | <-- { access_token: "새것" } ------- |
   |                                      |
   | --- GET /api/data ----------------> |
   |     Authorization: Bearer <새것>    |  정상 처리
   | <-- 200 OK ------------------------- |
```

### Refresh Token 보안 강화: Rotation

Refresh Token은 유효기간이 길기 때문에 탈취되면 더 위험하다. 이를 완화하는 기법이 **Refresh Token Rotation**이다. Refresh Token을 사용할 때마다 새 Refresh Token을 발급하고 기존 것을 무효화한다.

```java
@Service
@Transactional
@RequiredArgsConstructor
public class TokenRefreshService {

    private final RefreshTokenRepository refreshTokenRepository;
    private final JwtTokenProvider tokenProvider;

    public TokenPair refreshTokens(String oldRefreshToken) {
        // 1. 데이터베이스에서 refresh token 조회
        RefreshToken storedToken = refreshTokenRepository
            .findByToken(oldRefreshToken)
            .orElseThrow(() -> new InvalidTokenException("존재하지 않는 refresh token"));

        // 2. 이미 사용된 토큰인지 확인 (rotation 핵심)
        if (storedToken.isUsed()) {
            // 같은 refresh token이 두 번 사용되었다는 것은
            // 토큰 탈취 후 재사용 시도일 가능성이 높다
            // -> 해당 사용자의 모든 refresh token 무효화
            refreshTokenRepository.deleteAllByUserId(storedToken.getUserId());
            throw new TokenReuseDetectedException("토큰 재사용 감지: 보안상 모든 세션을 종료합니다");
        }

        // 3. 만료 확인
        if (storedToken.isExpired()) {
            refreshTokenRepository.delete(storedToken);
            throw new TokenExpiredException("refresh token이 만료되었습니다");
        }

        // 4. 기존 토큰 사용됨 처리
        storedToken.markAsUsed();
        refreshTokenRepository.save(storedToken);

        // 5. 새 토큰 쌍 발급
        Long userId = storedToken.getUserId();
        String newAccessToken = tokenProvider.generateAccessToken(userId, storedToken.getRole());
        String newRefreshToken = generateAndSaveRefreshToken(userId, storedToken.getRole());

        return new TokenPair(newAccessToken, newRefreshToken);
    }

    private String generateAndSaveRefreshToken(Long userId, String role) {
        String token = UUID.randomUUID().toString(); // 또는 별도 JWT
        RefreshToken refreshToken = RefreshToken.builder()
            .token(token)
            .userId(userId)
            .role(role)
            .expiresAt(Instant.now().plus(30, ChronoUnit.DAYS))
            .used(false)
            .build();
        refreshTokenRepository.save(refreshToken);
        return token;
    }
}
```

Refresh Token을 데이터베이스에 저장한다는 점이 중요하다. "JWT는 서버가 아무것도 저장하지 않는다"는 명제가 Refresh Token 앞에서 무너진다. Refresh Token 자체는 JWT일 필요가 없으며, 오히려 단순한 랜덤 문자열(opaque token)로 만들고 서버 DB에서 관리하는 것이 더 통제하기 쉽다.

### Refresh Token 저장 위치

클라이언트에서 Refresh Token을 어디에 저장할지도 중요한 보안 결정이다.

```
저장 위치          | XSS 취약 | CSRF 취약 | 비고
-------------------+----------+-----------+--------------------------------
localStorage       |   위험   |   안전    | JS에서 직접 접근 가능 -> XSS로 탈취
httpOnly 쿠키      |   안전   |   위험    | JS 접근 불가, CSRF 토큰으로 방어 필요
메모리(JS 변수)    |   안전   |   안전    | 페이지 새로고침 시 소멸 -> UX 불편
```

현재 권장되는 방식은 **httpOnly + Secure + SameSite=Strict 쿠키**에 Refresh Token을 저장하고, CSRF 방어를 추가하는 것이다.

```java
@PostMapping("/auth/login")
public ResponseEntity<AccessTokenResponse> login(
        @RequestBody LoginRequest request,
        HttpServletResponse response) {

    TokenPair tokens = authService.login(request);

    // Refresh Token을 httpOnly 쿠키에 저장
    ResponseCookie refreshCookie = ResponseCookie.from("refresh_token", tokens.getRefreshToken())
        .httpOnly(true)    // JS 접근 차단
        .secure(true)      // HTTPS에서만 전송
        .sameSite("Strict") // 동일 사이트에서만 전송 (CSRF 방어)
        .path("/auth/refresh") // refresh 엔드포인트에서만 쿠키 전송
        .maxAge(Duration.ofDays(30))
        .build();

    response.addHeader(HttpHeaders.SET_COOKIE, refreshCookie.toString());

    // Access Token은 응답 바디로 (클라이언트 메모리에 보관)
    return ResponseEntity.ok(new AccessTokenResponse(tokens.getAccessToken()));
}
```

---

## 토큰 탈취 대응: 현실적인 위협과 방어

### 어디서 어떻게 탈취되는가

토큰 탈취 경로를 알아야 방어를 설계할 수 있다.

**1. XSS (Cross-Site Scripting)**
localStorage에 저장된 토큰은 악의적인 스크립트가 `localStorage.getItem('access_token')`으로 즉시 훔칠 수 있다. 광고 SDK, 서드파티 스크립트 중 하나라도 XSS에 취약하면 끝이다.

**2. 중간자 공격 (MITM)**
HTTPS를 사용하지 않으면 네트워크 경로에서 토큰을 가로챌 수 있다. 프로덕션에서 HTTP를 쓰는 일은 없어야 한다.

**3. 로그 노출**
개발자가 디버깅용으로 요청/응답 전체를 로그로 남길 때, Authorization 헤더가 로그 저장소에 그대로 남는다. 로그 수집 시스템이 침해되면 토큰이 대량으로 유출된다.

```java
// 안티패턴: Authorization 헤더를 통째로 로깅
log.debug("Request headers: {}", request.getHeaderNames()); // 위험

// 올바른 방식: 민감한 헤더 마스킹
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        log.debug("Request: {} {} (Authorization: [REDACTED])",
            request.getMethod(), request.getRequestURI());
        filterChain.doFilter(request, response);
    }
}
```

**4. JWT Payload 노출**
흔한 오해가 있다. JWT Payload는 암호화되지 않는다. Base64URL 인코딩은 암호화가 아니라 단순 인코딩이다. 누구나 디코딩할 수 있다. 따라서 민감한 정보를 Payload에 넣으면 안 된다.

```java
// 안티패턴: 민감 정보를 Payload에 포함
Jwts.builder()
    .claim("ssn", user.getSocialSecurityNumber())  // 절대 금지
    .claim("creditCard", user.getCreditCardNumber()) // 절대 금지
    .claim("password", user.getHashedPassword())    // 절대 금지
    .build();

// 올바른 방식: 식별자와 권한 정보만 포함
Jwts.builder()
    .subject(user.getId().toString())
    .claim("role", user.getRole())
    .claim("tier", user.getSubscriptionTier())
    .build();
```

### Access Token 탈취 시 대응

Access Token이 탈취되었을 때, JWT 방식에서는 만료될 때까지 서버가 막을 방법이 없다는 것이 핵심 약점이다. 이를 보완하는 방법들:

**방법 1: 짧은 유효기간**
15분이면 탈취된 토큰의 피해 창이 최대 15분이다. 단순하지만 효과적인 방어다.

**방법 2: 토큰 블랙리스트**
완전한 해결책이지만, JWT의 장점(무상태)을 일부 희생한다.

```java
@Service
@RequiredArgsConstructor
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;

    // 토큰 무효화 (로그아웃, 강제 탈퇴, 비밀번호 변경 등)
    public void blacklist(String token, Instant expiration) {
        Duration ttl = Duration.between(Instant.now(), expiration);
        if (!ttl.isNegative()) {
            // 토큰 만료 시각까지만 블랙리스트 유지
            redisTemplate.opsForValue().set(
                "blacklist:" + hashToken(token),
                "1",
                ttl
            );
        }
    }

    public boolean isBlacklisted(String token) {
        return Boolean.TRUE.equals(
            redisTemplate.hasKey("blacklist:" + hashToken(token))
        );
    }

    private String hashToken(String token) {
        // 토큰 전체를 키로 쓰면 메모리 낭비 -> 해시 사용
        return DigestUtils.sha256Hex(token);
    }
}
```

**방법 3: jti (JWT ID) 기반 무효화**
토큰 전체가 아닌 jti만 블랙리스트에 저장하면 저장 공간을 절약할 수 있다.

```java
public String generateAccessToken(Long userId, String role) {
    String jti = UUID.randomUUID().toString();
    return Jwts.builder()
        .id(jti)  // jti 클레임 설정
        .subject(userId.toString())
        .claim("role", role)
        .expiration(...)
        .signWith(getSigningKey())
        .compact();
}

// 검증 시
public void validateToken(String token) {
    Claims claims = parseToken(token);
    String jti = claims.getId();
    if (blacklistService.isJtiBlacklisted(jti)) {
        throw new InvalidTokenException("무효화된 토큰");
    }
}
```

---

## 세션 vs JWT: 실전 선택 기준

### 결정 매트릭스

이론보다 실제 상황에 맞는 선택이 중요하다.

```
질문                                    세션 유리   JWT 유리
──────────────────────────────────────────────────────────
즉각적인 토큰 무효화가 필요한가?           ●
(계정 정지, 비밀번호 변경, 의심 활동 감지)

서버 인프라를 최소화하고 싶은가?                       ●
(Redis 같은 공유 저장소 없이)

여러 도메인/서비스가 토큰을 소비하는가?               ●
(마이크로서비스, 서드파티 API)

민감한 세션 데이터를 서버에서만 보관?       ●

모바일 앱, SPA 등 쿠키 없는 클라이언트?               ●

서버 수평 확장이 핵심 요구사항?                        ●
(단, Redis 세션도 가능)

소규모 서비스, 단일 서버?                  ●

완벽한 로그아웃 (모든 디바이스)?            ●
```

### 실제 시나리오별 권장사항

**시나리오 1: 금융/의료 서비스**
```
권장: 세션 (Redis 기반)
이유:
- 계정 정지 시 즉시 차단이 법적/규정 요구사항
- 비밀번호 변경 후 즉시 모든 세션 무효화 필요
- 민감 데이터를 클라이언트 측에 노출하지 말아야 함
- Refresh Token rotation을 써도 15분의 취약 창이 문제
```

**시나리오 2: 마이크로서비스 아키텍처**
```
권장: JWT (RS256)
이유:
- 각 서비스가 중앙 세션 서버 없이 독립적으로 인증 검증
- Auth 서버가 공개키를 JWKS 엔드포인트로 노출
- 서비스 간 네트워크 호출 감소
- 단, 토큰 유효기간을 짧게 유지 (15분 이하)
```

```java
// JWKS 엔드포인트 구현 예시 (Auth 서버)
@GetMapping("/.well-known/jwks.json")
public JwkSet getJwkSet() {
    // 현재 활성 공개키와 rotation 준비 중인 공개키 모두 노출
    return jwkSetProvider.getPublicKeys();
}

// 각 마이크로서비스의 토큰 검증 (Auth 서버 호출 없음)
@Bean
public JwtDecoder jwtDecoder() {
    return NimbusJwtDecoder
        .withJwkSetUri("https://auth.mycompany.com/.well-known/jwks.json")
        .build();
}
```

**시나리오 3: 일반 SaaS 서비스**
```
권장: JWT + Refresh Token Rotation + 선택적 블랙리스트
구성:
- Access Token: 15분, httpOnly 쿠키 or 메모리
- Refresh Token: 7일, httpOnly + Secure + SameSite 쿠키
- Refresh Token DB 저장 (rotation + 재사용 감지)
- 비밀번호 변경/로그아웃 시 Refresh Token 삭제
- 의심 활동 감지 시 jti 블랙리스트 활용
```

**시나리오 4: 소규모 서비스 (MVP, 스타트업 초기)**
```
권장: 서블릿 세션 (단일 서버) 또는 Spring Session + Redis
이유:
- JWT의 복잡성 (Refresh Token rotation, 블랙리스트)이 MVP 단계에 과하다
- Spring Security의 기본 세션 관리가 이미 안전하다
- 필요할 때 JWT로 마이그레이션 가능
```

### 흔한 실수와 안티패턴 정리

**안티패턴 1: 긴 유효기간의 Access Token**
```java
// 잘못된 예: 7일짜리 Access Token
.expiration(new Date(System.currentTimeMillis() + 7 * 24 * 60 * 60 * 1000))

// 옳은 예: 15분~1시간
.expiration(new Date(System.currentTimeMillis() + 15 * 60 * 1000))
```

탈취된 토큰의 피해 기간이 7일이 된다. 이는 사실상 세션의 단점(서버 부하)과 JWT의 단점(무효화 불가)을 동시에 갖는 최악의 선택이다.

**안티패턴 2: alg: none 허용**
JWT 스펙에는 서명 없는 "none" 알고리즘이 있다. 일부 라이브러리 구버전에서 `alg: none`으로 헤더를 조작해 서명 검증을 우회하는 공격이 가능했다.

```java
// 최신 라이브러리는 기본적으로 none을 거부하지만 명시적 방어 권장
Jwts.parser()
    .requireAlgorithm(SignatureAlgorithm.HS256) // 허용 알고리즘 명시
    .verifyWith(signingKey)
    .build();
```

**안티패턴 3: 토큰 검증 없이 Payload 신뢰**
```java
// 절대 금지: 서명 검증 없이 Base64 디코딩만으로 사용
String payload = new String(Base64.decode(token.split("\\.")[1]));
Map<String, Object> claims = objectMapper.readValue(payload, Map.class);
Long userId = (Long) claims.get("userId"); // 조작된 값일 수 있다!

// 올바른 방식: 반드시 검증 후 사용
Claims verifiedClaims = tokenProvider.validateAndParseClaims(token);
Long userId = Long.parseLong(verifiedClaims.getSubject());
```

**안티패턴 4: JWKS 캐싱 없이 매 요청마다 Auth 서버 조회**
```java
// 문제: 매 요청마다 Auth 서버에 공개키 조회 -> Auth 서버가 병목
JwtDecoder jwtDecoder = NimbusJwtDecoder.withJwkSetUri(jwksUri).build();

// 해결: 캐시 설정 (기본적으로 NimbusJwtDecoder는 캐싱하지만 명시적 설정 권장)
NimbusJwtDecoder decoder = NimbusJwtDecoder.withJwkSetUri(jwksUri)
    .cache(Duration.ofHours(1)) // 1시간 캐시
    .build();
```

### 결론: 단순함이 보안이다

인증 시스템을 설계할 때 가장 중요한 원칙은 **필요 이상으로 복잡하게 만들지 않는 것**이다.

Refresh Token rotation, 블랙리스트, JWKS, 로컬 캐싱을 전부 직접 구현하기보다는, 검증된 라이브러리와 프레임워크(Spring Security, Keycloak, Auth0)의 기능을 활용하는 것이 더 안전하다. 직접 구현한 코드일수록 직접 만든 버그가 있다.

세션이 "오래된" 방식이고 JWT가 "최신" 방식이라는 인식은 틀렸다. 각각이 해결하는 문제가 다르고, 트레이드오프가 다르다. 서비스의 규모, 아키텍처, 보안 요구사항을 냉정하게 평가하고 그에 맞는 방식을 선택하는 것이 전문가의 판단이다.

다음 편에서는 OAuth 2.0과 OpenID Connect로 넘어간다. 서드파티 로그인, 소셜 로그인, API 인가의 표준이 되는 이 프로토콜들이 이번 편에서 배운 토큰 개념을 어떻게 확장하는지를 다룬다.

---

## 참고 자료

1. **"The Web Application Hacker's Handbook"** — Dafydd Stuttard, Marcus Pinto. 웹 인증 취약점의 실전 공격과 방어를 깊이 다루는 고전.

2. **RFC 7519 — JSON Web Token (JWT)** — IETF 공식 스펙. JWT 클레임, 서명 알고리즘, 보안 고려사항의 원전.

3. **OWASP Authentication Cheat Sheet** — [https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html). 인증 구현 시 체크리스트로 활용할 수 있는 실전 가이드.

4. **"OAuth 2 in Action"** — Justin Richer, Antonio Sanso. OAuth와 토큰 기반 인증의 전체 생태계를 다루며, JWT 실전 사용 패턴을 상세히 설명한다.

5. **Spring Security Reference Documentation** — [https://docs.spring.io/spring-security/reference/](https://docs.spring.io/spring-security/reference/). Spring 기반 인증 구현의 공식 레퍼런스.

6. **"Security Engineering"** — Ross Anderson (3판). 인증 시스템 설계를 암호학적 기반부터 시스템 아키텍처까지 포괄적으로 다루는 교과서.
