---
title: "[인증과 보안] 3편 — 권한 관리: 인가(Authorization) 설계"
date: 2026-03-17T22:05:00+09:00
draft: false
tags: ["인가", "RBAC", "ABAC", "Spring Security", "보안", "서버"]
series: ["인증과 보안"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 4
summary: "인증 vs 인가 명확히 구분, RBAC 설계와 역할 계층, ABAC 개념, 리소스 소유권 검증과 수평적 권한 상승 방지, Spring Security 필터 체인과 커스텀 인가 로직까지"
---

로그인에 성공했다고 해서 모든 문이 열리는 건 아니다. 인증(Authentication)은 "당신이 누구인지"를 확인하는 과정이고, 인가(Authorization)는 "당신이 무엇을 할 수 있는지"를 결정하는 과정이다. 많은 시스템이 인증에는 공을 들이면서 인가를 대충 처리한다. 결과는 예측 가능하다. 일반 사용자가 타인의 주문 내역을 조회하거나, 만료된 계정이 관리자 기능에 접근하거나, 공격자가 URL 파라미터 하나를 바꿔 권한 밖의 데이터를 열람한다. 이 글은 인가 시스템을 제대로 설계하는 방법을 다룬다. RBAC부터 ABAC까지, 그리고 Spring Security로 이를 실제 코드로 구현하는 것까지.

---

## 인증과 인가, 무엇이 다른가

### 개념의 경계

두 개념을 혼용하는 개발자가 생각보다 많다. 명확하게 구분하자.

**인증(Authentication)**: 신원 확인. "나는 user@example.com 이다"라는 주장이 사실인지 검증한다. 비밀번호, OTP, 생체 인식, 인증서 등이 수단이 된다.

**인가(Authorization)**: 권한 확인. 인증된 주체가 특정 자원(resource)에 대해 특정 행위(action)를 수행할 수 있는지 결정한다. "이 사용자가 이 데이터를 읽을 수 있는가?"

순서가 중요하다. 인증 없이 인가는 불가능하다. 누구인지 모르면 무엇을 허용할지 결정할 수 없다. 반대로 인증에 성공했다고 모든 리소스에 접근 가능한 것도 아니다.

```
요청 흐름:
클라이언트 요청
  → 인증 필터: "이 토큰은 유효한가? 이 사용자는 존재하는가?"
  → 인가 필터: "이 사용자는 이 URL에 접근할 수 있는가?"
  → 컨트롤러: "이 사용자는 이 특정 리소스를 조작할 수 있는가?"
```

### 흔한 실수 패턴

**실수 1: 인증만 검사하고 인가를 생략**

```java
// 나쁜 예시 - 로그인 여부만 확인
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId, @AuthenticationPrincipal User user) {
    if (user == null) {
        throw new UnauthorizedException();
    }
    // 이 사용자가 orderId의 소유자인지 검사하지 않음!
    return orderRepository.findById(orderId).orElseThrow();
}
```

로그인한 사용자라면 누구든 `orderId`만 알면 타인의 주문을 볼 수 있다.

**실수 2: 프론트엔드 UI 숨김으로 인가 처리**

버튼을 숨기는 건 UX다. 서버 API에 인가 로직이 없으면 API를 직접 호출하는 공격자를 막을 수 없다.

**실수 3: 역할 체크를 비즈니스 로직 안에 흩어 놓기**

```java
// 나쁜 예시 - 역할 체크가 비즈니스 로직과 뒤섞임
public void deletePost(Long postId, User user) {
    Post post = postRepository.findById(postId).orElseThrow();
    if (!user.getRole().equals("ADMIN") && !post.getAuthorId().equals(user.getId())) {
        throw new ForbiddenException();
    }
    postRepository.delete(post);
}
```

조건이 복잡해질수록 버그가 생긴다. 인가 로직은 별도 계층으로 분리해야 한다.

---

## RBAC: 역할 기반 접근 제어

### 핵심 개념

RBAC(Role-Based Access Control)는 권한을 사용자에게 직접 부여하지 않고, 역할(Role)에 부여한다. 사용자는 역할을 부여받고, 역할을 통해 권한을 간접적으로 얻는다.

```
사용자 ─── 역할 ─── 권한
user1 ─── ADMIN ─── READ_USER
                 ─── WRITE_USER
                 ─── DELETE_USER
user2 ─── MEMBER ─── READ_POST
                  ─── WRITE_POST
```

장점은 관리 단순화다. 사용자가 100명이고 권한이 50개라면, 사용자-권한 조합은 이론상 5000개다. 역할이 5개라면 역할-권한 조합은 250개로 줄고, 사용자-역할 조합은 100개 이하가 된다.

### 역할 계층 설계

단순 RBAC에서 역할 간 계층(Hierarchy)을 도입하면 상위 역할이 하위 역할의 권한을 자동으로 상속한다.

```
SUPER_ADMIN
    └── ADMIN
            └── MODERATOR
                    └── MEMBER
                            └── GUEST
```

Spring Security에서 역할 계층을 설정하는 방법:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public RoleHierarchy roleHierarchy() {
        RoleHierarchyImpl hierarchy = new RoleHierarchyImpl();
        hierarchy.setHierarchy(
            "ROLE_SUPER_ADMIN > ROLE_ADMIN\n" +
            "ROLE_ADMIN > ROLE_MODERATOR\n" +
            "ROLE_MODERATOR > ROLE_MEMBER\n" +
            "ROLE_MEMBER > ROLE_GUEST"
        );
        return hierarchy;
    }

    @Bean
    public DefaultWebSecurityExpressionHandler expressionHandler(RoleHierarchy roleHierarchy) {
        DefaultWebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
        handler.setRoleHierarchy(roleHierarchy);
        return handler;
    }
}
```

이제 `ROLE_ADMIN`을 가진 사용자는 `ROLE_MEMBER`의 모든 권한을 자동으로 가진다.

### DB 스키마 설계

실무에서 RBAC는 보통 이런 스키마로 구현한다.

```sql
-- 사용자 테이블
CREATE TABLE users (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    email       VARCHAR(255) UNIQUE NOT NULL,
    password    VARCHAR(255) NOT NULL,
    created_at  DATETIME NOT NULL
);

-- 역할 테이블
CREATE TABLE roles (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(50) UNIQUE NOT NULL,  -- 'ROLE_ADMIN', 'ROLE_MEMBER'
    description VARCHAR(255)
);

-- 권한 테이블
CREATE TABLE permissions (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(100) UNIQUE NOT NULL, -- 'POST:READ', 'POST:WRITE', 'USER:DELETE'
    description VARCHAR(255)
);

-- 역할-권한 매핑
CREATE TABLE role_permissions (
    role_id       BIGINT NOT NULL REFERENCES roles(id),
    permission_id BIGINT NOT NULL REFERENCES permissions(id),
    PRIMARY KEY (role_id, permission_id)
);

-- 사용자-역할 매핑
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL REFERENCES users(id),
    role_id BIGINT NOT NULL REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);
```

권한 이름 컨벤션으로 `리소스:행위` 형태를 추천한다. `POST:READ`, `USER:WRITE`, `COMMENT:DELETE` 같은 방식이다. 나중에 와일드카드 매칭(`POST:*`)이나 세분화(`POST:READ:OWN` vs `POST:READ:ALL`)도 자연스럽게 확장할 수 있다.

### Java 엔티티 설계

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String password;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();

    // 역할에서 권한 추출
    public Set<String> getAllPermissions() {
        return roles.stream()
            .flatMap(role -> role.getPermissions().stream())
            .map(Permission::getName)
            .collect(Collectors.toSet());
    }
}

@Entity
@Table(name = "roles")
public class Role {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name; // ROLE_ADMIN

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    private Set<Permission> permissions = new HashSet<>();
}

@Entity
@Table(name = "permissions")
public class Permission {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name; // POST:READ
}
```

### UserDetails와 권한 통합

Spring Security는 `UserDetails`의 `getAuthorities()`로 권한을 판단한다. 사용자의 역할과 권한 모두를 `GrantedAuthority`로 노출해야 한다.

```java
public class UserDetailsImpl implements UserDetails {
    private final User user;

    public UserDetailsImpl(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Set<GrantedAuthority> authorities = new HashSet<>();

        // 역할 추가 (ROLE_ 접두사 포함)
        user.getRoles().forEach(role ->
            authorities.add(new SimpleGrantedAuthority(role.getName()))
        );

        // 세부 권한 추가
        user.getAllPermissions().forEach(permission ->
            authorities.add(new SimpleGrantedAuthority(permission))
        );

        return authorities;
    }

    // ... 나머지 UserDetails 메서드
}
```

---

## ABAC: 속성 기반 접근 제어

### RBAC의 한계

RBAC는 역할이 고정되어 있다. "자신이 작성한 글만 수정 가능", "근무 시간에만 접근 가능", "VPN을 통한 접근만 허용" 같은 동적 조건은 역할만으로 표현하기 어렵다.

역할을 무한정 세분화하는 방식으로 해결하려다 보면 `ROLE_MEMBER_WEEKDAY_VPN`, `ROLE_MEMBER_WEEKEND_ONPREMISE` 같은 괴물 역할이 탄생한다.

### ABAC의 접근

ABAC(Attribute-Based Access Control)는 주체(Subject), 자원(Resource), 환경(Environment)의 속성을 기반으로 동적으로 접근 여부를 결정한다.

```
정책 예시:
IF subject.department == resource.department
   AND subject.clearanceLevel >= resource.sensitivityLevel
   AND environment.time BETWEEN '09:00' AND '18:00'
THEN ALLOW
```

구성 요소:
- **주체 속성**: 사용자 역할, 부서, 직급, 위치
- **자원 속성**: 리소스 소유자, 민감도 등급, 카테고리
- **환경 속성**: 접속 시간, IP 주소, 접속 경로(VPN 여부)
- **정책 엔진**: 속성 조합으로 허용/거부 결정

### Spring Security에서 ABAC 구현

Spring Security의 `PermissionEvaluator`를 활용하면 메서드 수준의 ABAC를 구현할 수 있다.

```java
@Component
public class CustomPermissionEvaluator implements PermissionEvaluator {

    private final PostRepository postRepository;
    private final UserContextService userContextService;

    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        if (auth == null || !auth.isAuthenticated()) return false;

        UserDetailsImpl userDetails = (UserDetailsImpl) auth.getPrincipal();

        // 리소스 타입 기반 분기
        if (targetDomainObject instanceof Post post) {
            return evaluatePostPermission(userDetails, post, (String) permission);
        }

        return false;
    }

    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId, String targetType, Object permission) {
        if (auth == null || !auth.isAuthenticated()) return false;

        UserDetailsImpl userDetails = (UserDetailsImpl) auth.getPrincipal();

        if ("Post".equals(targetType)) {
            Post post = postRepository.findById((Long) targetId)
                .orElseThrow(() -> new ResourceNotFoundException("Post not found"));
            return evaluatePostPermission(userDetails, post, (String) permission);
        }

        return false;
    }

    private boolean evaluatePostPermission(UserDetailsImpl userDetails, Post post, String permission) {
        User user = userDetails.getUser();

        return switch (permission) {
            case "READ" -> {
                // 공개 글은 누구나 읽기 가능
                if (post.isPublic()) yield true;
                // 비공개 글은 소유자 또는 관리자만
                yield post.getAuthorId().equals(user.getId())
                    || user.hasRole("ROLE_ADMIN");
            }
            case "WRITE" -> {
                // 소유자 또는 관리자만 수정 가능
                yield post.getAuthorId().equals(user.getId())
                    || user.hasRole("ROLE_ADMIN");
            }
            case "DELETE" -> {
                // 소유자, 모더레이터, 관리자
                yield post.getAuthorId().equals(user.getId())
                    || user.hasAnyRole("ROLE_MODERATOR", "ROLE_ADMIN");
            }
            default -> false;
        };
    }
}
```

설정에 `PermissionEvaluator`를 등록한다.

```java
@Configuration
@EnableMethodSecurity(prePostEnabled = true)
public class MethodSecurityConfig {

    @Bean
    public MethodSecurityExpressionHandler expressionHandler(
            CustomPermissionEvaluator permissionEvaluator,
            RoleHierarchy roleHierarchy) {
        DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler();
        handler.setPermissionEvaluator(permissionEvaluator);
        handler.setRoleHierarchy(roleHierarchy);
        return handler;
    }
}
```

이제 컨트롤러나 서비스에서 `@PreAuthorize`로 선언적으로 인가를 처리할 수 있다.

```java
@RestController
@RequestMapping("/api/posts")
public class PostController {

    @GetMapping("/{postId}")
    @PreAuthorize("hasPermission(#postId, 'Post', 'READ')")
    public PostResponse getPost(@PathVariable Long postId) {
        return postService.findById(postId);
    }

    @PutMapping("/{postId}")
    @PreAuthorize("hasPermission(#postId, 'Post', 'WRITE')")
    public PostResponse updatePost(@PathVariable Long postId, @RequestBody PostUpdateRequest request) {
        return postService.update(postId, request);
    }

    @DeleteMapping("/{postId}")
    @PreAuthorize("hasPermission(#postId, 'Post', 'DELETE')")
    public void deletePost(@PathVariable Long postId) {
        postService.delete(postId);
    }
}
```

---

## 리소스 소유권 검증과 수평적 권한 상승 방지

### 수직적 권한 상승 vs 수평적 권한 상승

**수직적 권한 상승(Vertical Privilege Escalation)**: 낮은 권한 사용자가 높은 권한을 획득. 일반 사용자가 관리자 기능에 접근.

**수평적 권한 상승(Horizontal Privilege Escalation)**: 같은 권한 수준에서 타인의 리소스에 접근. 사용자 A가 사용자 B의 개인 정보를 조회.

OWASP Top 10에서 **Broken Access Control**이 1위를 차지한 이유가 여기 있다. 수평적 권한 상승은 역할 체크만으로는 막을 수 없다. 두 사용자 모두 `ROLE_MEMBER`이므로 역할 체크를 통과한다. 추가로 리소스 소유권을 반드시 검증해야 한다.

### 소유권 검증 패턴

**패턴 1: 서비스 계층에서 소유권 검증**

```java
@Service
public class OrderService {

    public OrderResponse getOrder(Long orderId, Long requestUserId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + orderId));

        // 소유권 검증: 요청 사용자가 주문 소유자인가?
        if (!order.getUserId().equals(requestUserId)) {
            // 정보 노출 방지를 위해 "권한 없음"이 아닌 "없는 리소스"로 응답할 수도 있음
            throw new AccessDeniedException("Access denied to order: " + orderId);
        }

        return OrderResponse.from(order);
    }
}
```

**패턴 2: 쿼리 수준에서 소유권 강제**

소유권 조건을 쿼리 자체에 포함시키면 더 안전하다. 해당 사용자의 리소스가 아니면 데이터베이스에서 아예 조회되지 않는다.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // userId를 조건에 포함 — 타인의 orderId를 넣어도 결과가 없음
    Optional<Order> findByIdAndUserId(Long id, Long userId);

    // 페이징 목록 조회도 동일
    Page<Order> findAllByUserId(Long userId, Pageable pageable);
}
```

```java
@Service
public class OrderService {

    public OrderResponse getOrder(Long orderId, Long requestUserId) {
        // 소유권이 일치하지 않으면 empty → ResourceNotFoundException
        Order order = orderRepository.findByIdAndUserId(orderId, requestUserId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + orderId));

        return OrderResponse.from(order);
    }
}
```

이 방식의 장점은 404와 403을 구분하지 않아도 된다는 것이다. 타인의 리소스는 "존재하지 않는 것"처럼 보인다. 공격자가 리소스 존재 여부조차 파악하지 못한다.

**패턴 3: Spring Data JPA의 @Query와 :#{principal.id}**

Spring Security의 SpEL과 Spring Data를 결합하면 현재 인증 사용자를 쿼리에 자동으로 주입할 수 있다.

```java
public interface PostRepository extends JpaRepository<Post, Long> {

    @Query("SELECT p FROM Post p WHERE p.id = :id AND p.authorId = :#{principal.id}")
    Optional<Post> findByIdAndCurrentUser(@Param("id") Long id);
}
```

### 인가 체크의 위치

인가 검증을 어느 계층에서 해야 할까? 답은 "여러 계층에서"다.

```
HTTP 계층 (Spring Security Filter Chain)
  → URL 패턴 기반: /admin/** 는 ROLE_ADMIN만
  → 인증 여부: /api/** 는 로그인 필수

메서드 계층 (@PreAuthorize / @PostAuthorize)
  → 역할 기반: @PreAuthorize("hasRole('ADMIN')")
  → 권한 기반: @PreAuthorize("hasAuthority('POST:WRITE')")
  → 동적 인가: @PreAuthorize("hasPermission(#postId, 'Post', 'WRITE')")

서비스/쿼리 계층
  → 소유권 검증: findByIdAndUserId()
  → 비즈니스 규칙 기반 인가
```

계층마다 다른 수준의 인가를 수행한다. URL 계층은 거친 망, 메서드 계층은 중간 망, 서비스/쿼리 계층은 촘촘한 망 역할을 한다.

### 안티패턴: 인가 우회 경로

**안티패턴 1: ID 순서 예측 가능성**

순차 증가 ID(`1, 2, 3, ...`)를 사용하면 공격자가 타인의 리소스 ID를 쉽게 추측한다. UUID나 [ULID](https://github.com/ulid/spec)처럼 예측 불가능한 ID를 사용하면 공격 표면을 줄일 수 있다. 단, 이는 보안의 보완책이지 소유권 검증을 대체하지 않는다.

```java
@Entity
public class Order {
    @Id
    private String id = UUID.randomUUID().toString(); // 예측 불가능한 ID

    private Long userId;
    // ...
}
```

**안티패턴 2: 간접 객체 참조 없이 직접 ID 노출**

Insecure Direct Object Reference(IDOR)는 내부 식별자를 API에 그대로 노출할 때 발생한다. 노출이 불가피하다면 반드시 소유권 검증을 수행해야 한다.

**안티패턴 3: 배치 작업에서 소유권 검증 누락**

```java
// 나쁜 예시 - 배치 삭제에서 소유권 검증 없음
@DeleteMapping("/posts")
public void deletePosts(@RequestBody List<Long> postIds, @AuthenticationPrincipal UserDetailsImpl user) {
    postRepository.deleteAllById(postIds); // 타인의 글도 삭제됨!
}

// 올바른 예시
@DeleteMapping("/posts")
public void deletePosts(@RequestBody List<Long> postIds, @AuthenticationPrincipal UserDetailsImpl user) {
    Long userId = user.getUser().getId();
    // 소유권이 일치하는 것만 삭제
    postRepository.deleteAllByIdInAndUserId(postIds, userId);
}
```

---

## Spring Security 필터 체인과 커스텀 인가 로직

### 필터 체인 구조

Spring Security는 서블릿 필터 체인으로 동작한다. 요청이 들어오면 등록된 필터들이 순서대로 실행되고, 각 필터가 인증/인가 처리를 담당한다.

```
요청
 → SecurityContextPersistenceFilter     (SecurityContext 로딩)
 → UsernamePasswordAuthenticationFilter (폼 로그인)
 → BearerTokenAuthenticationFilter      (JWT 토큰 검증)
 → ExceptionTranslationFilter           (인증/인가 예외 처리)
 → FilterSecurityInterceptor            (URL 기반 인가 결정)
 → 컨트롤러
```

### SecurityFilterChain 설정

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final CustomAccessDeniedHandler accessDeniedHandler;
    private final CustomAuthenticationEntryPoint authenticationEntryPoint;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            // CSRF 비활성화 (JWT 사용 시 세션이 없으므로)
            .csrf(csrf -> csrf.disable())

            // 세션 관리: STATELESS
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // URL 기반 인가 규칙
            .authorizeHttpRequests(auth -> auth
                // 공개 엔드포인트
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/posts/**").permitAll()

                // 역할 기반
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/moderator/**").hasAnyRole("ADMIN", "MODERATOR")

                // 권한 기반
                .requestMatchers(HttpMethod.POST, "/api/posts").hasAuthority("POST:WRITE")
                .requestMatchers(HttpMethod.DELETE, "/api/posts/**").hasAuthority("POST:DELETE")

                // 나머지는 인증 필요
                .anyRequest().authenticated()
            )

            // 예외 처리
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(authenticationEntryPoint)  // 미인증
                .accessDeniedHandler(accessDeniedHandler)             // 인가 실패
            )

            // JWT 필터를 UsernamePasswordAuthenticationFilter 앞에 추가
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)

            .build();
    }
}
```

### JWT 인증 필터 구현

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            String email = jwtTokenProvider.getEmailFromToken(token);

            UserDetails userDetails = userDetailsService.loadUserByUsername(email);

            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,
                    userDetails.getAuthorities()
                );

            authentication.setDetails(
                new WebAuthenticationDetailsSource().buildDetails(request)
            );

            SecurityContextHolder.getContext().setAuthentication(authentication);
        }

        filterChain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### 커스텀 인가 예외 처리

인가 실패 시 응답을 커스텀하는 것은 중요하다. 기본 Spring Security 응답은 HTML을 반환하는데, REST API에서는 JSON이 필요하다.

```java
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    @Override
    public void handle(
            HttpServletRequest request,
            HttpServletResponse response,
            AccessDeniedException accessDeniedException) throws IOException {

        response.setStatus(HttpStatus.FORBIDDEN.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");

        ErrorResponse errorResponse = ErrorResponse.of(
            "ACCESS_DENIED",
            "해당 리소스에 접근할 권한이 없습니다."
        );

        response.getWriter().write(objectMapper.writeValueAsString(errorResponse));
    }
}

@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    @Override
    public void commence(
            HttpServletRequest request,
            HttpServletResponse response,
            AuthenticationException authException) throws IOException {

        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");

        ErrorResponse errorResponse = ErrorResponse.of(
            "UNAUTHORIZED",
            "인증이 필요합니다."
        );

        response.getWriter().write(objectMapper.writeValueAsString(errorResponse));
    }
}
```

### 동적 권한 로딩과 캐시 전략

DB에서 권한을 매 요청마다 조회하면 성능 문제가 생긴다. 캐시를 활용하되, 권한 변경 시 캐시를 무효화하는 전략이 필요하다.

```java
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    @Cacheable(value = "userDetails", key = "#email", unless = "#result == null")
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        User user = userRepository.findByEmailWithRolesAndPermissions(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));

        return new UserDetailsImpl(user);
    }
}

@Service
public class UserRoleService {

    private final UserRepository userRepository;
    private final CacheManager cacheManager;

    @Transactional
    public void assignRole(Long userId, String roleName) {
        User user = userRepository.findById(userId).orElseThrow();
        // ... 역할 부여 로직

        // 권한 변경 후 캐시 무효화
        Cache userDetailsCache = cacheManager.getCache("userDetails");
        if (userDetailsCache != null) {
            userDetailsCache.evict(user.getEmail());
        }
    }
}
```

JWT 기반 시스템에서는 토큰에 포함된 권한 정보가 만료 전까지 고정된다는 문제가 있다. 권한을 즉시 박탈해야 하는 경우:

1. **짧은 토큰 만료 시간**: Access Token을 15분으로 짧게 유지. 권한 변경 후 최대 15분 후 반영.
2. **토큰 블랙리스트**: 박탈된 토큰 ID를 Redis에 저장하고 필터에서 검사.
3. **DB 권한 재확인**: 중요 작업에서는 토큰의 권한 대신 DB를 직접 조회.

```java
// 토큰 블랙리스트 검사
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final TokenBlacklistService tokenBlacklistService;

    @Override
    protected void doFilterInternal(...) {
        String token = extractToken(request);

        if (token != null) {
            // 블랙리스트 확인
            if (tokenBlacklistService.isBlacklisted(token)) {
                response.sendError(HttpStatus.UNAUTHORIZED.value(), "Token has been revoked");
                return;
            }
            // ... 나머지 인증 로직
        }

        filterChain.doFilter(request, response);
    }
}
```

### 메서드 수준 인가의 실전 패턴

`@PreAuthorize`와 `@PostAuthorize`의 차이를 이해하는 것이 중요하다.

- `@PreAuthorize`: 메서드 실행 **전** 인가 검사. 불필요한 DB 쿼리를 막을 수 있다.
- `@PostAuthorize`: 메서드 실행 **후** 반환값을 기준으로 인가 검사. 반환 객체의 속성을 검사할 때 사용.

```java
@Service
public class PostService {

    // 실행 전: 현재 사용자가 POST:WRITE 권한을 가지는가?
    @PreAuthorize("hasAuthority('POST:WRITE')")
    public PostResponse createPost(PostCreateRequest request) {
        // ...
    }

    // 실행 후: 반환된 게시글이 현재 사용자의 것인가?
    @PostAuthorize("returnObject.authorId == authentication.principal.user.id or hasRole('ADMIN')")
    public PostResponse getPrivatePost(Long postId) {
        return postRepository.findById(postId)
            .map(PostResponse::from)
            .orElseThrow();
    }

    // SpEL로 복잡한 조건 표현
    @PreAuthorize(
        "hasRole('ADMIN') or " +
        "(hasAuthority('POST:WRITE') and #request.categoryId == authentication.principal.user.department.categoryId)"
    )
    public PostResponse createCategoryPost(PostCreateRequest request) {
        // ...
    }
}
```

### 인가 로직 테스트

인가 로직은 반드시 자동화된 테스트로 검증해야 한다.

```java
@SpringBootTest
@AutoConfigureMockMvc
class PostControllerAuthTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @WithMockUser(username = "user1@test.com", roles = {"MEMBER"})
    void 일반_사용자는_타인의_비공개_글에_접근할_수_없다() throws Exception {
        // given: user2가 작성한 비공개 글
        Long privatePostId = createPrivatePostByUser2();

        // when: user1이 해당 글에 접근 시도
        mockMvc.perform(get("/api/posts/{postId}", privatePostId))
            // then: 403 또는 404
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(username = "admin@test.com", roles = {"ADMIN"})
    void 관리자는_모든_비공개_글에_접근할_수_있다() throws Exception {
        Long privatePostId = createPrivatePostByUser2();

        mockMvc.perform(get("/api/posts/{postId}", privatePostId))
            .andExpect(status().isOk());
    }

    @Test
    void 미인증_사용자는_보호된_엔드포인트에_접근할_수_없다() throws Exception {
        mockMvc.perform(post("/api/posts")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(username = "user1@test.com", roles = {"MEMBER"})
    void 글_작성자는_자신의_글을_수정할_수_있다() throws Exception {
        Long myPostId = createPostByUser1();

        mockMvc.perform(put("/api/posts/{postId}", myPostId)
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"title\": \"수정된 제목\", \"content\": \"수정된 내용\"}"))
            .andExpect(status().isOk());
    }
}
```

---

## 실전 체크리스트

인가 시스템을 설계하거나 코드 리뷰할 때 다음 항목을 점검하자.

**설계 단계**
- [ ] 인증과 인가 로직이 명확히 분리되어 있는가?
- [ ] 역할 계층이 적절하게 설계되었는가?
- [ ] 모든 API 엔드포인트의 접근 제어 정책이 문서화되어 있는가?

**구현 단계**
- [ ] URL 기반, 메서드 기반, 쿼리 기반 인가를 계층적으로 적용했는가?
- [ ] 리소스 소유권 검증이 쿼리 수준에서 강제되는가?
- [ ] 배치 작업에서도 소유권 검증이 수행되는가?
- [ ] 인가 실패 시 적절한 HTTP 상태 코드(401/403)를 반환하는가?
- [ ] 권한 정보 캐시와 무효화 전략이 있는가?

**테스트 단계**
- [ ] 정상 접근(허용) 케이스 테스트가 존재하는가?
- [ ] 비정상 접근(거부) 케이스 테스트가 존재하는가?
- [ ] 수평적 권한 상승 시도 테스트가 존재하는가?
- [ ] 미인증 접근 테스트가 존재하는가?

---

인가 시스템은 "한 번 만들면 끝"이 아니다. 서비스가 성장하면서 역할이 추가되고, 리소스 유형이 늘어나고, 비즈니스 규칙이 복잡해진다. 처음부터 확장 가능한 구조로 설계하고, 모든 접근 제어 결정을 테스트로 문서화해두는 것이 유일한 방어선이다. 로그인에 성공했다고 모든 문이 열려서는 안 된다. 각 문마다 열쇠가 따로 있어야 한다.

---

## 참고 자료

1. **OWASP Top 10 - A01:2021 Broken Access Control** — https://owasp.org/Top10/A01_2021-Broken_Access_Control/
2. **Spring Security Reference Documentation** — https://docs.spring.io/spring-security/reference/index.html
3. **NIST SP 800-162: Guide to Attribute Based Access Control (ABAC)** — https://nvlpubs.nist.gov/nistpubs/specialpublications/NIST.SP.800-162.pdf
4. **Role-Based Access Controls (NIST)** — Ferraiolo, D.F., Sandhu, R., Gavrila, S., Kuhn, D.R., Chandramouli, R. (2001)
5. **IDOR Prevention Cheat Sheet (OWASP)** — https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html
6. **Spring Security in Action** — Laurentiu Spilca, Manning Publications (2020)
