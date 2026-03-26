---
title: "[인증과 보안] 2편 — OAuth 2.0과 OpenID Connect: 위임 인증의 원리"
date: 2026-03-17T22:06:00+09:00
draft: false
tags: ["OAuth", "OIDC", "소셜 로그인", "PKCE", "보안", "서버"]
series: ["인증과 보안"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 4
summary: "OAuth 2.0 역할(Resource Owner, Client, Authorization Server), 인가 코드 플로우 단계별 분해, PKCE, Client Credentials·Device Flow 등 그란트 타입별 용도, OpenID Connect와 소셜 로그인 구현까지"
---

"구글 계정으로 로그인"을 클릭할 때 내부에서 무슨 일이 벌어지는지 설명할 수 있는가?

단순히 "OAuth 쓰면 돼"라고 답한다면, 보안 취약점이 어디에 숨어 있는지 파악하지 못한 상태로 운영하는 것이다.

Authorization Code Flow와 Implicit Flow의 차이, PKCE가 왜 존재하는지, ID Token이 Access Token과 어떻게 다른지 — 이 질문들에 자신 있게 답하지 못한다면, 오늘 이 글을 끝까지 읽어야 한다.

OAuth 2.0과 OpenID Connect는 현대 인증 아키텍처의 근간이다.

잘못 이해하면 사용자 계정 탈취부터 CSRF 취약점까지 심각한 보안 사고로 이어진다.

---

## 1. OAuth 2.0의 등장 배경과 핵심 문제 의식

OAuth 이전의 세상을 상상해보자. 사용자가 서드파티 앱에게 Gmail 연락처에 접근하도록 허용하려면, 구글 계정의 **아이디와 비밀번호를 직접 넘겨야** 했다. 앱이 해당 자격증명을 어떻게 사용할지, 얼마나 오래 보관할지 통제할 방법이 없었다.

OAuth 2.0이 해결하려 한 문제는 하나다: **비밀번호 없이 특정 리소스에 대한 제한된 접근 권한을 위임**하는 것. "내 구글 드라이브에서 파일 읽기 권한만" 혹은 "내 트위터 타임라인 읽기 권한만"을 앱에 줄 수 있다면, 비밀번호를 노출하지 않아도 된다.

> **중요한 전제**: OAuth 2.0은 **인가(Authorization)** 프로토콜이지 **인증(Authentication)** 프로토콜이 아니다. 이 구분을 흐릿하게 이해하는 순간, 심각한 보안 취약점이 생긴다. OpenID Connect가 왜 탄생했는지는 이 맥락에서 이해해야 한다.

---

## 2. OAuth 2.0의 네 가지 역할

RFC 6749 스펙은 OAuth 2.0의 참여자를 명확하게 네 역할로 정의한다.

### Resource Owner (자원 소유자)

보호된 자원을 소유한 주체. 일반적으로 **최종 사용자(End User)**다. 구글 드라이브의 파일은 사용자 소유이므로, 사용자가 Resource Owner다. 사용자가 직접 개입할 수 없는 서버 간 통신(Client Credentials Flow)에서는 Resource Owner가 Client와 동일한 주체가 되기도 한다.

### Resource Server (자원 서버)

보호된 자원을 실제로 호스팅하는 서버. Google Drive API, GitHub API, Kakao API 등이 여기에 해당한다. Access Token을 검증하고 요청을 처리한다. Authorization Server와 같은 서버일 수도 있고, 별도 서버일 수도 있다. 실무에서는 대부분 분리되어 있다.

### Client (클라이언트)

Resource Owner를 대신해 보호된 자원에 접근하려는 애플리케이션. "클라이언트"라는 단어가 혼란을 줄 수 있는데, 여기서의 Client는 브라우저나 모바일 앱이 될 수도 있고, **백엔드 서버**가 될 수도 있다. OAuth에서의 Client는 자원 접근을 요청하는 쪽이라는 의미다.

Client는 두 종류로 나뉜다:
- **Confidential Client**: 비밀 키(Client Secret)를 안전하게 보관할 수 있는 서버 사이드 앱
- **Public Client**: 비밀 키를 안전하게 보관할 수 없는 SPA, 모바일 앱

이 구분이 중요한 이유는, Public Client에서는 Client Secret을 사용할 수 없기 때문에 PKCE 같은 별도 보안 메커니즘이 필요하기 때문이다.

### Authorization Server (인가 서버)

Resource Owner를 인증하고, 그 동의를 받아 Client에게 토큰을 발급하는 서버. 구글의 `accounts.google.com`, 카카오의 `kauth.kakao.com` 등이 이에 해당한다. Authorization Server는 두 가지 엔드포인트를 반드시 노출한다:
- **Authorization Endpoint** (`/oauth/authorize`): 사용자 동의 화면 제공
- **Token Endpoint** (`/oauth/token`): 토큰 발급

```
┌─────────────────────────────────────────────────────────────┐
│                    OAuth 2.0 역할 관계도                      │
│                                                             │
│  ┌──────────────┐    ①동의 요청    ┌──────────────────┐    │
│  │   Resource   │◄──────────────►│  Authorization   │    │
│  │    Owner     │    ②동의 승인    │     Server       │    │
│  └──────────────┘                └──────────────────┘    │
│         │                               │                  │
│         │                        ③Authorization           │
│         │                            Code 발급             │
│         │                               │                  │
│         ▼                               ▼                  │
│  ┌──────────────┐    ④토큰 요청    ┌──────────────────┐    │
│  │    Client    │───────────────►│  Authorization   │    │
│  │  (우리 앱)   │◄───────────────│     Server       │    │
│  └──────────────┘    ⑤토큰 발급   └──────────────────┘    │
│         │                                                   │
│         │ ⑥API 호출 (Bearer Token)                         │
│         ▼                                                   │
│  ┌──────────────┐                                          │
│  │   Resource   │                                          │
│  │    Server    │                                          │
│  └──────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. 인가 코드 플로우 단계별 분해

Authorization Code Flow는 OAuth 2.0에서 가장 안전하고 널리 사용되는 그란트 타입이다. 단계를 하나씩 뜯어보자.

### Step 1: Authorization Request 구성

사용자가 "구글 계정으로 로그인" 버튼을 클릭하면, Client는 다음 URL로 사용자를 리다이렉트한다.

```
https://accounts.google.com/o/oauth2/v2/auth
  ?response_type=code
  &client_id=YOUR_CLIENT_ID
  &redirect_uri=https://yourapp.com/callback
  &scope=openid%20email%20profile
  &state=xK9m2nP4qR7s
```

각 파라미터의 의미:
- `response_type=code`: 인가 코드를 원한다는 의미. `token`이면 Implicit Flow
- `client_id`: Authorization Server에 등록된 Client 식별자
- `redirect_uri`: 인가 코드를 받을 URL. 사전에 등록된 URI와 **정확히 일치**해야 한다
- `scope`: 요청하는 권한의 범위
- `state`: CSRF 방지용 랜덤 값. **반드시 사용해야 한다**

`state` 파라미터를 생략하거나 고정값을 사용하는 것은 CSRF 공격에 무방비하게 노출되는 것과 같다. 반드시 세션마다 암호학적으로 안전한 랜덤 값을 생성해야 한다.

```java
// Spring Boot에서 state 생성 예시
@GetMapping("/oauth/login")
public String initiateLogin(HttpSession session) {
    String state = generateSecureState();
    session.setAttribute("oauth_state", state);

    String authorizationUri = UriComponentsBuilder
        .fromUriString("https://accounts.google.com/o/oauth2/v2/auth")
        .queryParam("response_type", "code")
        .queryParam("client_id", clientId)
        .queryParam("redirect_uri", redirectUri)
        .queryParam("scope", "openid email profile")
        .queryParam("state", state)
        .build()
        .toUriString();

    return "redirect:" + authorizationUri;
}

private String generateSecureState() {
    byte[] bytes = new byte[32];
    new SecureRandom().nextBytes(bytes);
    return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
}
```

### Step 2: 사용자 인증과 동의

Authorization Server가 사용자를 인증하고 동의 화면을 보여준다. 이 단계는 **전적으로 Authorization Server의 영역**이다. Client는 이 과정에서 사용자의 자격증명을 볼 수 없다. 이것이 OAuth의 핵심 가치다.

### Step 3: Authorization Code 수신

사용자가 동의하면, Authorization Server는 Client의 `redirect_uri`로 인가 코드를 담아 리다이렉트한다.

```
https://yourapp.com/callback
  ?code=4/P7q7W91a-oMsCeLvIaQm6bTrgtp7
  &state=xK9m2nP4qR7s
```

이 시점에서 반드시 `state` 값을 검증해야 한다.

```java
@GetMapping("/oauth/callback")
public String handleCallback(
    @RequestParam String code,
    @RequestParam String state,
    HttpSession session
) {
    // state 검증: CSRF 방지의 핵심
    String savedState = (String) session.getAttribute("oauth_state");
    if (!state.equals(savedState)) {
        throw new SecurityException("State mismatch - possible CSRF attack");
    }
    session.removeAttribute("oauth_state");

    // 이제 code를 사용해 토큰 요청
    return exchangeCodeForToken(code);
}
```

### Step 4: Authorization Code를 Token으로 교환

인가 코드는 수명이 매우 짧다 (보통 10분 이내). Client는 이 코드를 즉시 Token Endpoint로 POST 요청을 보내 토큰으로 교환해야 한다. **이 요청은 백채널(Back Channel), 즉 서버 간 직접 통신으로 이루어진다.**

```java
@Service
public class OAuthTokenService {

    private final RestTemplate restTemplate;

    public TokenResponse exchangeCodeForToken(String code) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("grant_type", "authorization_code");
        params.add("code", code);
        params.add("redirect_uri", redirectUri);
        params.add("client_id", clientId);
        params.add("client_secret", clientSecret); // Confidential Client만 사용

        HttpEntity<MultiValueMap<String, String>> request =
            new HttpEntity<>(params, headers);

        return restTemplate.postForObject(
            "https://oauth2.googleapis.com/token",
            request,
            TokenResponse.class
        );
    }
}
```

Authorization Server는 Access Token, Refresh Token, (OpenID Connect인 경우) ID Token을 응답한다.

```json
{
  "access_token": "ya29.a0AfH6SMC...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "1//0gLTuXNe...",
  "scope": "openid email profile",
  "id_token": "eyJhbGciOiJSUzI1NiJ9..."
}
```

### 왜 두 단계(코드 → 토큰)로 나누는가?

이 질문이 핵심이다. 인가 코드를 직접 토큰으로 사용하면 안 되는 이유:

1. **URL에 노출**: 인가 코드는 브라우저 URL에 나타난다. 브라우저 히스토리, 서버 로그, Referer 헤더에 기록될 수 있다
2. **백채널 교환**: 토큰 교환은 서버 to 서버로 이루어지므로 브라우저에 노출되지 않는다
3. **클라이언트 인증**: 토큰 요청 시 `client_secret`으로 Client를 인증할 수 있다 (Confidential Client의 경우)

인가 코드 자체는 탈취되더라도 `client_secret` 없이는 토큰으로 교환할 수 없다. 이중 보안이다.

---

## 4. PKCE: Public Client를 위한 보안 강화

### PKCE가 필요한 이유

모바일 앱이나 SPA는 `client_secret`을 안전하게 보관할 수 없다. 앱 바이너리를 역공학하면 비밀 키를 추출할 수 있다. 이런 Public Client에서 인가 코드 플로우를 사용하면, 인가 코드가 탈취될 경우 아무 방어 수단이 없다.

**Authorization Code Interception Attack**: 악성 앱이 합법적인 앱의 `redirect_uri`와 동일한 URL 스킴을 등록해두면, 인가 코드를 가로챌 수 있다.

PKCE(Proof Key for Code Exchange, RFC 7636)는 이 문제를 해결하기 위한 확장이다.

### PKCE 동작 원리

```
┌─────────────────────────────────────────────────────────────┐
│                     PKCE 플로우                              │
│                                                             │
│  Client                           Authorization Server      │
│    │                                       │               │
│    │ ①code_verifier 생성 (랜덤 43-128자)   │               │
│    │   code_challenge = SHA256(code_verifier)               │
│    │                                       │               │
│    │──② Authorization Request ────────────►│               │
│    │   (code_challenge, code_challenge_method=S256)         │
│    │                                       │               │
│    │   [사용자 인증 및 동의]                │               │
│    │                                       │               │
│    │◄──③ Authorization Code ───────────────│               │
│    │                                       │               │
│    │──④ Token Request ─────────────────────►│               │
│    │   (code + code_verifier)              │               │
│    │                                       │               │
│    │   [서버: SHA256(code_verifier) == code_challenge 검증] │
│    │                                       │               │
│    │◄──⑤ Access Token ─────────────────────│               │
└─────────────────────────────────────────────────────────────┘
```

공격자가 인가 코드를 탈취해도 `code_verifier`를 모르면 토큰으로 교환할 수 없다. `code_verifier`는 메모리에만 존재하며, 외부에 노출되지 않는다.

```java
// Spring Security OAuth2 Client는 PKCE를 자동으로 처리하지만,
// 직접 구현할 경우:
public class PkceGenerator {

    public static PkceData generate() throws NoSuchAlgorithmException {
        // code_verifier: 암호학적으로 안전한 랜덤 문자열
        byte[] verifierBytes = new byte[32];
        new SecureRandom().nextBytes(verifierBytes);
        String codeVerifier = Base64.getUrlEncoder()
            .withoutPadding()
            .encodeToString(verifierBytes);

        // code_challenge: SHA-256(code_verifier)의 Base64URL 인코딩
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] challengeBytes = digest.digest(codeVerifier.getBytes(StandardCharsets.UTF_8));
        String codeChallenge = Base64.getUrlEncoder()
            .withoutPadding()
            .encodeToString(challengeBytes);

        return new PkceData(codeVerifier, codeChallenge);
    }

    public record PkceData(String codeVerifier, String codeChallenge) {}
}
```

```java
// Authorization Request에 PKCE 파라미터 추가
PkceData pkce = PkceGenerator.generate();
session.setAttribute("pkce_verifier", pkce.codeVerifier());

String authUri = UriComponentsBuilder
    .fromUriString(authorizationEndpoint)
    .queryParam("code_challenge", pkce.codeChallenge())
    .queryParam("code_challenge_method", "S256")
    // ... 나머지 파라미터
    .build().toUriString();
```

현재 업계 표준은 **Public Client는 항상 PKCE를 사용**하는 것이며, Confidential Client도 PKCE를 함께 사용하도록 권장하는 추세다(OAuth 2.0 Security BCP).

---

## 5. 그란트 타입별 용도와 선택 기준

OAuth 2.0은 다양한 시나리오를 위해 여러 그란트 타입을 정의한다. 각 타입의 존재 이유와 적합한 사용 사례를 이해해야 한다.

### Authorization Code Grant (with PKCE)

**언제 사용**: 사용자가 직접 개입하는 모든 시나리오
- 웹 앱 (Confidential Client)
- 모바일 앱, SPA (Public Client, PKCE 필수)
- 가장 안전한 방식이므로 가능하면 항상 이 방식을 선택

### Client Credentials Grant

**언제 사용**: 사용자(Resource Owner)가 없는 **서버 간 통신**

마이크로서비스 A가 마이크로서비스 B의 API를 호출할 때, 특정 사용자의 이름으로 호출하는 것이 아니라 서비스 자체의 권한으로 호출한다.

```java
@Service
public class ServiceToServiceTokenClient {

    public String getAccessToken() {
        HttpHeaders headers = new HttpHeaders();
        headers.setBasicAuth(clientId, clientSecret);
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("grant_type", "client_credentials");
        params.add("scope", "inventory:read order:write");

        TokenResponse response = restTemplate.postForObject(
            tokenEndpoint,
            new HttpEntity<>(params, headers),
            TokenResponse.class
        );

        return response.getAccessToken();
    }
}
```

이 플로우에서는 Refresh Token이 발급되지 않는다. 토큰이 만료되면 새로운 Client Credentials Grant 요청을 보내면 된다.

**실무 패턴**: Client Credentials 토큰은 캐시해서 재사용한다. 매 요청마다 토큰을 새로 발급받으면 Authorization Server에 불필요한 부하를 준다.

```java
@Service
public class CachedTokenProvider {

    private volatile String cachedToken;
    private volatile Instant tokenExpiry;
    private final Object lock = new Object();

    public String getToken() {
        if (isTokenValid()) {
            return cachedToken;
        }

        synchronized (lock) {
            // Double-checked locking
            if (isTokenValid()) {
                return cachedToken;
            }
            refreshToken();
        }

        return cachedToken;
    }

    private boolean isTokenValid() {
        return cachedToken != null &&
               Instant.now().isBefore(tokenExpiry.minusSeconds(30)); // 30초 여유
    }

    private void refreshToken() {
        TokenResponse response = fetchNewToken();
        this.cachedToken = response.getAccessToken();
        this.tokenExpiry = Instant.now().plusSeconds(response.getExpiresIn());
    }
}
```

### Device Authorization Grant (Device Flow)

**언제 사용**: 입력 인터페이스가 제한된 기기 — TV, 게임 콘솔, 스마트 스피커, IoT 기기

키보드로 복잡한 URL을 입력하기 어려운 기기에서 사용자 인증을 가능하게 한다.

```
┌──────────────────────────────────────────────────────────────┐
│                   Device Flow 동작 방식                       │
│                                                              │
│  TV 앱                  Authorization Server     사용자 폰   │
│    │                           │                    │       │
│    │──① Device Auth Request───►│                    │       │
│    │                           │                    │       │
│    │◄──② device_code ──────────│                    │       │
│    │    user_code: WDJB-MJHT   │                    │       │
│    │    verification_uri:      │                    │       │
│    │    google.com/device       │                    │       │
│    │                           │                    │       │
│    │ "폰에서 google.com/device  │                    │       │
│    │  방문 후 WDJB-MJHT 입력"  │◄──③ 코드 입력──────│       │
│    │                           │    및 동의         │       │
│    │──④ Token Polling ─────────►│                    │       │
│    │   (authorization_pending) │                    │       │
│    │──④ Token Polling ─────────►│                    │       │
│    │◄──⑤ Access Token ─────────│                    │       │
└──────────────────────────────────────────────────────────────┘
```

```java
// Device Flow 폴링 구현
public TokenResponse pollForToken(String deviceCode, int interval)
    throws InterruptedException {

    while (true) {
        Thread.sleep(interval * 1000L);

        try {
            return requestToken(deviceCode);
        } catch (OAuth2Exception e) {
            switch (e.getError()) {
                case "authorization_pending":
                    // 사용자가 아직 동의하지 않음, 계속 대기
                    continue;
                case "slow_down":
                    // 폴링 간격을 5초 늘려야 함
                    interval += 5;
                    continue;
                case "expired_token":
                    throw new RuntimeException("Device code expired");
                case "access_denied":
                    throw new RuntimeException("User denied access");
                default:
                    throw e;
            }
        }
    }
}
```

### Refresh Token Grant

Access Token은 수명이 짧다(보통 1시간). Refresh Token을 사용하면 사용자 재인증 없이 새 Access Token을 발급받을 수 있다.

```java
public String refreshAccessToken(String refreshToken) {
    MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
    params.add("grant_type", "refresh_token");
    params.add("refresh_token", refreshToken);
    params.add("client_id", clientId);
    params.add("client_secret", clientSecret);

    TokenResponse response = restTemplate.postForObject(
        tokenEndpoint,
        new HttpEntity<>(params, headers),
        TokenResponse.class
    );

    // 일부 Authorization Server는 Refresh Token도 교체 발급
    if (response.getRefreshToken() != null) {
        updateStoredRefreshToken(response.getRefreshToken());
    }

    return response.getAccessToken();
}
```

**Refresh Token Rotation**: 보안 강화를 위해 Refresh Token을 사용하면 새 Refresh Token을 발급하고 기존 것을 무효화하는 방식. 탈취된 Refresh Token이 사용되면 기존 세션도 무효화되어 탐지가 가능해진다.

### ~~Implicit Grant (폐기)~~

과거에는 SPA를 위해 사용됐지만, **보안 취약점으로 인해 현재는 사용을 권장하지 않는다**. Access Token이 URL fragment에 직접 노출되며, Refresh Token도 없다. PKCE를 사용하는 Authorization Code Flow로 대체해야 한다.

---

## 6. OpenID Connect: 인증 레이어의 추가

### OAuth 2.0만으로는 인증이 안 되는 이유

OAuth 2.0으로 Access Token을 받았다고 해서 "이 사람이 누구인지" 알 수 있는 것은 아니다. Access Token은 단지 "이 클라이언트가 특정 리소스에 접근할 수 있다"는 증명일 뿐이다.

실제로 Access Token으로 `/userinfo` 엔드포인트를 호출해서 사용자 정보를 가져오는 방식을 사용하는 경우가 많다. 하지만 이 방식에는 문제가 있다:
- `/userinfo` API 스펙이 표준화되어 있지 않다 (카카오, 네이버, 구글 모두 다르다)
- 매 요청마다 추가 API 호출이 필요하다
- Access Token 자체가 누가 인증했는지를 보장하지 않는다

OpenID Connect(OIDC)는 OAuth 2.0 위에 **인증 레이어**를 추가한 프로토콜이다. 핵심은 **ID Token**이다.

### ID Token: 사용자의 신원 증명서

ID Token은 JWT(JSON Web Token) 형식의 토큰으로, Authorization Server가 서명한다. 사용자가 **인증되었다는 사실과 신원 정보**를 담는다.

```
// ID Token 페이로드 예시 (디코딩 후)
{
  "iss": "https://accounts.google.com",       // 발급자
  "sub": "110169484474386276334",              // 사용자 고유 식별자
  "aud": "your-client-id",                    // 수신자 (Client ID)
  "exp": 1742300000,                          // 만료 시간
  "iat": 1742296400,                          // 발급 시간
  "nonce": "n-0S6_WzA2Mj",                   // CSRF/재전송 방지
  "email": "user@example.com",
  "email_verified": true,
  "name": "Hong Gildong",
  "picture": "https://lh3.googleusercontent.com/..."
}
```

### ID Token vs Access Token

| 항목 | ID Token | Access Token |
|------|----------|--------------|
| 목적 | 사용자 인증 정보 전달 | 리소스 접근 권한 증명 |
| 수신자 | Client (우리 앱) | Resource Server (API 서버) |
| 형식 | 항상 JWT | JWT 또는 Opaque |
| 검증 방법 | JWT 서명 검증 (로컬) | 인트로스펙션 또는 JWT 서명 검증 |
| Resource Server 전달 | 절대 안 됨 | Bearer 헤더로 전달 |

**중요한 실수**: ID Token을 API 호출의 Bearer Token으로 사용하는 것. ID Token은 Client가 사용하는 인증 정보이지, Resource Server에 전달하는 것이 아니다.

### ID Token 검증

ID Token은 단순히 파싱하는 것으로 끝나서는 안 된다. 반드시 다음을 검증해야 한다.

```java
@Service
public class IdTokenValidator {

    private final JwkSetUriJwtDecoderBuilder decoderBuilder;

    public OidcUser validateAndExtract(String idToken, String expectedNonce) {
        // 1. JWT 서명 검증 (공개 키로)
        JwtDecoder decoder = JwtDecoders.fromIssuerLocation(issuerUri);
        Jwt jwt = decoder.decode(idToken);

        // 2. iss (발급자) 검증
        String iss = jwt.getIssuer().toString();
        if (!issuerUri.equals(iss)) {
            throw new JwtValidationException("Invalid issuer");
        }

        // 3. aud (수신자) 검증 - 반드시 우리 Client ID여야 함
        List<String> aud = jwt.getAudience();
        if (!aud.contains(clientId)) {
            throw new JwtValidationException("Invalid audience");
        }

        // 4. exp (만료 시간) 검증
        if (jwt.getExpiresAt().isBefore(Instant.now())) {
            throw new JwtValidationException("Token expired");
        }

        // 5. nonce 검증 (재전송 공격 방지)
        String tokenNonce = jwt.getClaim("nonce");
        if (!expectedNonce.equals(tokenNonce)) {
            throw new JwtValidationException("Nonce mismatch");
        }

        return buildOidcUser(jwt);
    }
}
```

Spring Security를 사용하면 이 검증들이 자동으로 처리된다. 하지만 내부에서 무슨 일이 벌어지는지 알아야 문제가 생겼을 때 디버깅할 수 있다.

---

## 7. 소셜 로그인 구현 실전 (Spring Boot + Spring Security)

### 의존성 설정

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### application.yml 설정

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - openid
              - email
              - profile
          kakao:
            client-id: ${KAKAO_CLIENT_ID}
            client-secret: ${KAKAO_CLIENT_SECRET}
            client-authentication-method: client_secret_post
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - profile_nickname
              - account_email
        provider:
          kakao:
            authorization-uri: https://kauth.kakao.com/oauth/authorize
            token-uri: https://kauth.kakao.com/oauth/token
            user-info-uri: https://kapi.kakao.com/v2/user/me
            user-name-attribute: id
```

### Security 설정

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final CustomOAuth2UserService oauth2UserService;
    private final CustomOidcUserService oidcUserService;
    private final OAuth2AuthenticationSuccessHandler successHandler;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .userInfoEndpoint(userInfo -> userInfo
                    .oidcUserService(oidcUserService)    // OIDC (Google, etc.)
                    .userService(oauth2UserService)      // Non-OIDC (Kakao, etc.)
                )
                .successHandler(successHandler)
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .clearAuthentication(true)
                .deleteCookies("JSESSIONID")
            );

        return http.build();
    }
}
```

### 사용자 정보 처리와 계정 연동

소셜 로그인의 실무적 핵심 과제는 **소셜 계정과 서비스 계정의 연동**이다.

```java
@Service
@RequiredArgsConstructor
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oAuth2User = super.loadUser(userRequest);

        String registrationId = userRequest.getClientRegistration().getRegistrationId();
        OAuthAttributes attributes = OAuthAttributes.of(registrationId, oAuth2User.getAttributes());

        User user = saveOrUpdate(attributes);

        return new DefaultOAuth2User(
            Collections.singleton(new SimpleGrantedAuthority(user.getRole().name())),
            attributes.getAttributes(),
            attributes.getNameAttributeKey()
        );
    }

    private User saveOrUpdate(OAuthAttributes attributes) {
        User user = userRepository
            .findByProviderAndProviderId(attributes.getProvider(), attributes.getProviderId())
            .map(existing -> existing.update(attributes.getName(), attributes.getEmail()))
            .orElse(attributes.toEntity());

        return userRepository.save(user);
    }
}
```

```java
// OAuthAttributes: 각 소셜 제공자마다 다른 응답 구조를 통일
@Builder
@Getter
public class OAuthAttributes {

    private Map<String, Object> attributes;
    private String nameAttributeKey;
    private String name;
    private String email;
    private String picture;
    private String provider;
    private String providerId;

    public static OAuthAttributes of(String registrationId, Map<String, Object> attributes) {
        return switch (registrationId) {
            case "google" -> ofGoogle(attributes);
            case "kakao" -> ofKakao(attributes);
            case "naver" -> ofNaver(attributes);
            default -> throw new IllegalArgumentException("Unsupported provider: " + registrationId);
        };
    }

    private static OAuthAttributes ofGoogle(Map<String, Object> attributes) {
        return OAuthAttributes.builder()
            .name((String) attributes.get("name"))
            .email((String) attributes.get("email"))
            .picture((String) attributes.get("picture"))
            .provider("google")
            .providerId((String) attributes.get("sub"))
            .nameAttributeKey("sub")
            .attributes(attributes)
            .build();
    }

    @SuppressWarnings("unchecked")
    private static OAuthAttributes ofKakao(Map<String, Object> attributes) {
        Map<String, Object> kakaoAccount = (Map<String, Object>) attributes.get("kakao_account");
        Map<String, Object> profile = (Map<String, Object>) kakaoAccount.get("profile");

        return OAuthAttributes.builder()
            .name((String) profile.get("nickname"))
            .email((String) kakaoAccount.get("email"))
            .picture((String) profile.get("profile_image_url"))
            .provider("kakao")
            .providerId(String.valueOf(attributes.get("id")))
            .nameAttributeKey("id")
            .attributes(attributes)
            .build();
    }
}
```

### 이메일 기반 계정 연동의 함정

"같은 이메일이면 같은 사람"이라고 가정하고 계정을 자동 연동하는 것은 **보안 취약점**이다. 소셜 제공자가 이메일 인증을 강제하지 않을 수 있고, 공격자가 타인의 이메일로 소셜 계정을 생성해 계정 탈취를 시도할 수 있다.

올바른 접근:
1. 이메일이 같아도 **다른 소셜 계정은 별도 계정으로 처리**하거나
2. 연동 시 **사용자 명시적 확인**을 받거나
3. `email_verified: true`인 경우에만 이메일 기반 연동을 허용

---

## 8. Access Token 관리와 Resource Server 구현

### Bearer Token 검증

Resource Server는 모든 요청에서 Access Token을 검증해야 한다.

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain resourceServerFilterChain(HttpSecurity http) throws Exception {
        http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwkSetUri("https://accounts.google.com/.well-known/openid-configuration")
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("scope");
        grantedAuthoritiesConverter.setAuthorityPrefix("SCOPE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return converter;
    }
}
```

### Token Introspection vs JWT 자체 검증

두 방식의 트레이드오프를 이해해야 한다.

**JWT 자체 검증 (로컬 검증)**:
- 장점: Authorization Server 부하 없음, 빠름
- 단점: 토큰 즉시 무효화(revocation)가 불가능. 만료 시간까지 기다려야 함

**Token Introspection (원격 검증)**:
- 장점: 실시간 토큰 유효성 확인, 즉시 무효화 가능
- 단점: 매 요청마다 Authorization Server 호출 오버헤드

실무에서는 Access Token 수명을 짧게(1시간 이하) 유지하고 JWT 자체 검증을 사용하는 것이 일반적이다. 즉각적인 무효화가 필요한 경우(로그아웃, 계정 정지)에는 짧은 수명 토큰 + 블랙리스트 캐시를 조합한다.

---

## 9. 흔한 실수와 안티패턴

### 안티패턴 1: state 파라미터 생략 또는 고정값 사용

```java
// 잘못된 예
String authUri = "https://accounts.google.com/o/oauth2/v2/auth"
    + "?state=myapp";  // 고정값은 CSRF 방지 효과가 없다

// 올바른 예
String state = UUID.randomUUID().toString();
session.setAttribute("oauth_state", state);
```

CSRF 공격으로 공격자가 희생자의 계정을 자신의 소셜 계정에 연결하거나, 희생자를 공격자의 계정으로 로그인시킬 수 있다.

### 안티패턴 2: redirect_uri 검증 미흡

Authorization Server에 `redirect_uri`를 등록할 때 와일드카드나 패턴 매칭을 허용하는 경우:

```
# 위험한 설정 (일부 구형 Authorization Server)
redirect_uri: https://yourapp.com/*

# 안전한 설정
redirect_uri: https://yourapp.com/oauth/callback
```

공격자가 `redirect_uri=https://yourapp.com/evil`을 사용해 인가 코드를 자신의 서버로 리다이렉트할 수 있다.

### 안티패턴 3: Access Token을 프론트엔드에 노출

SPA에서 BFF(Backend for Frontend) 패턴 없이 Access Token을 로컬 스토리지에 저장하는 경우. XSS 공격으로 토큰이 탈취될 수 있다.

```
// 권장 아키텍처: BFF 패턴
[SPA] ──세션 쿠키──► [BFF 서버] ──Access Token──► [Resource Server]
```

BFF가 토큰을 서버 세션에 보관하고, 브라우저와는 HttpOnly 세션 쿠키로만 통신한다.

### 안티패턴 4: Refresh Token을 클라이언트 사이드에 저장

모바일 앱에서 Refresh Token을 평문으로 로컬 스토리지나 SharedPreferences에 저장하는 것. 반드시 Keychain(iOS)이나 Android Keystore를 사용해야 한다.

### 안티패턴 5: scope를 과도하게 요청

```java
// 잘못된 예: 필요하지도 않은 권한까지 요청
scope: "openid email profile contacts calendar drive"

// 올바른 예: 최소 권한 원칙
scope: "openid email"  // 로그인에 필요한 최소한만
```

사용자에게 불필요한 권한 동의를 요청하면 신뢰도가 떨어지고 OAuth 심사에서 거부될 수 있다.

### 안티패턴 6: nonce 없이 ID Token 수락

```java
// nonce를 사용하지 않으면 ID Token 재전송 공격에 취약
// Authorization Request 시:
String nonce = generateSecureNonce();
session.setAttribute("oidc_nonce", nonce);
// nonce 파라미터를 Authorization Request URL에 포함

// Callback 시:
String savedNonce = (String) session.getAttribute("oidc_nonce");
validateIdToken(idToken, savedNonce); // nonce 검증 포함
```

---

## 10. Discovery Document와 동적 설정

OIDC를 지원하는 Authorization Server는 **Discovery Document** (`/.well-known/openid-configuration`)를 통해 모든 엔드포인트와 지원 기능을 자동으로 노출한다.

```bash
$ curl https://accounts.google.com/.well-known/openid-configuration | jq
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "grant_types_supported": ["authorization_code", "refresh_token", ...],
  "id_token_signing_alg_values_supported": ["RS256"],
  ...
}
```

Spring Security는 `issuer-uri`만 설정하면 나머지를 자동으로 가져온다.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.google.com
```

이 설정 하나로 `jwks_uri`를 통한 공개 키 자동 로딩, 토큰 검증 설정이 완료된다.

---

## 정리: 올바른 플로우 선택

```
사용자 개입이 있는가?
├── Yes → Authorization Code Flow
│         ├── 서버 사이드 앱 → Authorization Code (+ client_secret)
│         ├── SPA, 모바일 앱 → Authorization Code + PKCE
│         └── 입력 제한 기기 → Device Authorization Grant
└── No → Client Credentials Grant (서버 간 통신)

인증(Authentication)이 필요한가?
├── Yes → scope에 openid 포함 (OIDC), ID Token 검증
└── No → Access Token만 사용 (OAuth 2.0)
```

OAuth 2.0과 OIDC는 겉보기엔 복잡해 보이지만, 각 컴포넌트의 역할과 책임이 명확하게 분리되어 있다.

Resource Owner는 권한을 위임하고, Authorization Server는 그 위임을 증명하는 토큰을 발급하며, Resource Server는 토큰을 검증하고 자원을 제공한다.

이 흐름을 완전히 이해하면, 어떤 소셜 로그인 제공자든, 어떤 OAuth 기반 API든 일관된 방식으로 통합할 수 있다.

---

## 참고 자료

1. **RFC 6749 - The OAuth 2.0 Authorization Framework** — OAuth 2.0의 공식 스펙. 역할과 플로우 정의의 원전. https://www.rfc-editor.org/rfc/rfc6749

2. **RFC 7636 - Proof Key for Code Exchange (PKCE)** — PKCE 메커니즘의 공식 정의와 보안 근거. https://www.rfc-editor.org/rfc/rfc7636

3. **OpenID Connect Core 1.0** — OIDC 스펙 원문. ID Token 구조와 검증 절차 포함. https://openid.net/specs/openid-connect-core-1_0.html

4. **OAuth 2.0 Security Best Current Practice** — 현재 권장되는 OAuth 2.0 보안 모범 사례. PKCE 강제, Implicit Flow 폐기 등의 근거. https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics

5. **"OAuth 2 in Action"** — Justin Richer, Antonio Sanso 저. OAuth 2.0 프로토콜을 구현 관점에서 가장 깊이 있게 다루는 책. Manning Publications.

6. **Spring Security OAuth2 공식 문서** — Spring Boot 환경에서의 OAuth2 Client, Resource Server 설정 가이드. https://docs.spring.io/spring-security/reference/servlet/oauth2/index.html
