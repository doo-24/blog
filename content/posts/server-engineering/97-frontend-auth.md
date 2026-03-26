---
title: "[Frontend 통합] 5편 — 인증 플로우 통합: OAuth 리다이렉트부터 토큰 관리까지"
date: 2026-03-17T12:03:00+09:00
draft: false
tags: ["OAuth", "PKCE", "토큰 관리", "Silent Refresh", "서버"]
series: ["Frontend 통합"]
series_group: "서버 엔지니어링"
series_group_order: 1
series_index: 14
summary: "OAuth Authorization Code+PKCE 프론트엔드 구현 흐름, 토큰 저장 위치(httpOnly 쿠키 vs 메모리 vs localStorage), Silent Refresh와 Refresh Token Rotation, BFF 토큰 프록시 패턴까지"
---

서버 엔지니어 입장에서 인증은 백엔드 영역처럼 느껴진다.
토큰 발급, 검증, 세션 관리 — 이 모든 것이 서버에서 일어나기 때문이다.

하지만 실제로 인증 버그의 절반 이상은 프론트엔드와 서버의 경계에서 발생한다.
리다이렉트 URI 불일치, 토큰 저장 방식 오류, Silent Refresh 타이밍 문제가 그 예다.

이 편에서는 OAuth Authorization Code + PKCE 플로우를 프론트엔드가 어떻게 구현하는지, 서버가 어디에 맞춰 설계해야 하는지를 함께 살펴본다.

---

## 1. OAuth Authorization Code + PKCE — 전체 흐름

### 1-1. 왜 PKCE인가

Authorization Code 플로우의 취약점은 `code` 탈취다.
공격자가 리다이렉트 URI를 가로채 `code`를 확보하면 Access Token을 발급받을 수 있다.

PKCE(Proof Key for Code Exchange)는 이를 막는다.
클라이언트가 랜덤 문자열(`code_verifier`)을 생성하고, 그 해시(`code_challenge`)를 Authorization 요청에 포함시킨다.
서버는 Token 요청 시 `code_verifier`를 받아 검증한다 — `code`를 탈취해도 `code_verifier` 없이는 토큰을 받을 수 없다. 자물쇠(challenge)는 미리 걸어두고, 열쇠(verifier)는 나중에 제시하는 구조다. 공격자는 자물쇠만 보고 열쇠를 만들 수 없다.

퍼블릭 클라이언트(SPA, 모바일 앱)에서는 PKCE가 사실상 필수다.

### 1-2. 단계별 흐름

**Step 1 — code_verifier, code_challenge 생성 (프론트엔드)**

```javascript
// 브라우저 Web Crypto API 사용
async function generatePKCE() {
  const verifier = generateRandomString(64);
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const digest = await crypto.subtle.digest("SHA-256", data);
  const challenge = base64URLEncode(digest);
  return { verifier, challenge };
}

function generateRandomString(length) {
  const array = new Uint8Array(length);
  crypto.getRandomValues(array);
  return Array.from(array, (b) => b.toString(36)).join("").slice(0, length);
}

function base64URLEncode(buffer) {
  return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=/g, "");
}
```

`code_verifier`는 나중에 사용해야 하므로 `sessionStorage`에 임시 저장한다.
`localStorage`는 XSS에 노출되므로 피한다.

**Step 2 — Authorization 요청 (브라우저 리다이렉트)**

```javascript
async function startLogin() {
  const { verifier, challenge } = await generatePKCE();
  const state = generateRandomString(32);

  // state와 verifier를 세션에 저장 (CSRF 방어용 state 포함)
  sessionStorage.setItem("oauth_state", state);
  sessionStorage.setItem("pkce_verifier", verifier);

  const params = new URLSearchParams({
    response_type: "code",
    client_id: "my-spa-client",
    redirect_uri: "https://app.example.com/callback",
    scope: "openid profile email",
    state: state,
    code_challenge: challenge,
    code_challenge_method: "S256",
  });

  window.location.href = `https://auth.example.com/oauth/authorize?${params}`;
}
```

**Step 3 — 콜백 처리 (프론트엔드 /callback 라우트)**

```javascript
async function handleCallback() {
  const params = new URLSearchParams(window.location.search);
  const code = params.get("code");
  const state = params.get("state");
  const error = params.get("error");

  if (error) {
    throw new Error(`OAuth error: ${error}`);
  }

  // state 검증 — CSRF 방어
  const savedState = sessionStorage.getItem("oauth_state");
  if (state !== savedState) {
    throw new Error("State mismatch — possible CSRF attack");
  }

  const verifier = sessionStorage.getItem("pkce_verifier");
  sessionStorage.removeItem("oauth_state");
  sessionStorage.removeItem("pkce_verifier");

  // code를 백엔드로 전달하여 토큰 교환
  const response = await fetch("/api/auth/token", {
    method: "POST",
    credentials: "include", // 쿠키 포함
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ code, verifier }),
  });

  if (!response.ok) throw new Error("Token exchange failed");
}
```

**Step 4 — 토큰 교환 (서버사이드)**

백엔드가 `code`와 `verifier`를 받아 Authorization Server로 토큰 요청을 보낸다.

```http
POST /oauth/token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=AUTH_CODE_HERE
&redirect_uri=https://app.example.com/callback
&client_id=my-spa-client
&code_verifier=VERIFIER_HERE
```

응답으로 `access_token`, `refresh_token`, `id_token`을 받는다.
이 토큰을 어디에 저장할지가 다음 핵심 주제다.

---

## 2. 토큰 저장 위치 — 세 가지 전략 비교

### 2-1. localStorage

가장 단순한 방법이지만 가장 위험하다.

```javascript
// 이렇게 하지 말 것
localStorage.setItem("access_token", token);
```

`localStorage`는 동일 오리진의 모든 JavaScript에서 접근 가능하다.
XSS 공격 한 번으로 토큰 전체가 탈취된다.

토큰이 탈취되면 공격자는 서버에 정상 요청을 보낼 수 있다 — 서버 입장에서 구분할 방법이 없다.

**결론: Access Token과 Refresh Token 모두 localStorage에 두지 않는다.**

### 2-2. 메모리 (JavaScript 변수)

```javascript
// 모듈 레벨 변수
let accessToken = null;

export function setAccessToken(token) {
  accessToken = token;
}

export function getAccessToken() {
  return accessToken;
}
```

XSS로 직접 접근하기 어렵고, 탭이나 창을 닫으면 사라진다.
Access Token처럼 수명이 짧은 토큰(5~15분)에 적합하다.

단점은 새로고침 시 사라진다는 것이다.
사용자가 F5를 누를 때마다 Silent Refresh나 재로그인이 필요하다.

**결론: Access Token 저장에 권장. 단, 새로고침 처리 로직이 필요하다.**

### 2-3. httpOnly 쿠키

서버가 Set-Cookie 헤더로 발급하고, JavaScript에서는 접근할 수 없다.
XSS 공격으로 토큰을 읽을 수 없다는 것이 핵심 장점이다.

```http
HTTP/1.1 200 OK
Set-Cookie: refresh_token=eyJ...; HttpOnly; Secure; SameSite=Strict; Path=/api/auth; Max-Age=2592000
```

`HttpOnly` — JavaScript 접근 차단.
`Secure` — HTTPS에서만 전송.
`SameSite=Strict` — 크로스사이트 요청 시 쿠키 미포함 (CSRF 방어).
`Path=/api/auth` — 특정 경로에만 쿠키 전송 (범위 최소화).

CSRF가 남아있는 위협이지만, `SameSite=Strict`와 CSRF 토큰 조합으로 방어한다.

**결론: Refresh Token은 반드시 httpOnly 쿠키. Access Token도 httpOnly 쿠키 사용 가능(성능 트레이드오프 있음).**

### 2-4. 저장 전략 요약

| 토큰 | 권장 저장소 | 이유 |
|------|------------|------|
| Access Token | 메모리 | 수명 짧음, XSS 노출 최소화 |
| Refresh Token | httpOnly 쿠키 | 수명 길어 탈취 위험 높음, JS 접근 차단 |
| CSRF Token | localStorage 또는 커스텀 헤더 | httpOnly 쿠키와 함께 CSRF 방어 |

---

## 3. Silent Refresh — 사용자 모르게 토큰 갱신하기

### 3-1. 왜 Silent Refresh가 필요한가

Access Token 수명을 5분으로 설정했다.
사용자가 계속 사용 중인데 5분마다 로그인 화면이 나타나면 안 된다.

만료 전에 자동으로 새 Access Token을 발급받는 것이 Silent Refresh다. Access Token 수명을 짧게 유지하면 탈취 시 피해를 최소화할 수 있고, Silent Refresh가 그 짧은 수명을 사용자 경험 저하 없이 보완한다.

### 3-2. 타이머 기반 구현

```javascript
const TOKEN_REFRESH_MARGIN = 60 * 1000; // 만료 60초 전에 갱신

class TokenManager {
  constructor() {
    this.accessToken = null;
    this.refreshTimer = null;
  }

  setToken(token, expiresIn) {
    this.accessToken = token;
    this.scheduleRefresh(expiresIn);
  }

  scheduleRefresh(expiresIn) {
    if (this.refreshTimer) clearTimeout(this.refreshTimer);
    const delay = expiresIn * 1000 - TOKEN_REFRESH_MARGIN;
    this.refreshTimer = setTimeout(() => this.silentRefresh(), delay);
  }

  async silentRefresh() {
    try {
      // Refresh Token은 httpOnly 쿠키로 자동 포함됨
      const response = await fetch("/api/auth/refresh", {
        method: "POST",
        credentials: "include",
      });

      if (response.status === 401) {
        // Refresh Token도 만료 — 재로그인 유도
        this.handleSessionExpired();
        return;
      }

      const { access_token, expires_in } = await response.json();
      this.setToken(access_token, expires_in);
    } catch (error) {
      console.error("Silent refresh failed:", error);
      this.handleSessionExpired();
    }
  }

  handleSessionExpired() {
    this.accessToken = null;
    // 로그인 페이지로 리다이렉트 또는 이벤트 발행
    window.dispatchEvent(new Event("session-expired"));
  }
}
```

### 3-3. 요청 인터셉터와 토큰 갱신 동기화

여러 요청이 동시에 401을 받으면 갱신 요청이 여러 번 날아간다.
Refresh Token이 Rotation 방식이라면 첫 번째만 성공하고 나머지는 실패한다.

동시 갱신 요청을 막는 큐 패턴이 필요하다.

```javascript
class ApiClient {
  constructor(tokenManager) {
    this.tokenManager = tokenManager;
    this.isRefreshing = false;
    this.failedQueue = [];
  }

  processQueue(error, token = null) {
    this.failedQueue.forEach((prom) => {
      if (error) {
        prom.reject(error);
      } else {
        prom.resolve(token);
      }
    });
    this.failedQueue = [];
  }

  async request(url, options = {}) {
    const token = this.tokenManager.accessToken;
    const headers = {
      ...options.headers,
      Authorization: `Bearer ${token}`,
    };

    const response = await fetch(url, { ...options, headers, credentials: "include" });

    if (response.status !== 401) return response;

    // 401 처리 — 갱신 중이면 큐에 대기
    if (this.isRefreshing) {
      return new Promise((resolve, reject) => {
        this.failedQueue.push({ resolve, reject });
      }).then((newToken) => {
        headers.Authorization = `Bearer ${newToken}`;
        return fetch(url, { ...options, headers, credentials: "include" });
      });
    }

    this.isRefreshing = true;

    try {
      const refreshResponse = await fetch("/api/auth/refresh", {
        method: "POST",
        credentials: "include",
      });

      if (!refreshResponse.ok) throw new Error("Refresh failed");

      const { access_token, expires_in } = await refreshResponse.json();
      this.tokenManager.setToken(access_token, expires_in);
      this.processQueue(null, access_token);

      headers.Authorization = `Bearer ${access_token}`;
      return fetch(url, { ...options, headers, credentials: "include" });
    } catch (error) {
      this.processQueue(error);
      this.tokenManager.handleSessionExpired();
      throw error;
    } finally {
      this.isRefreshing = false;
    }
  }
}
```

### 3-4. 새로고침 시 초기화 문제

페이지를 새로 고치면 메모리에 있던 Access Token이 사라진다.
앱 초기화 시점에 자동으로 갱신을 시도해야 한다.

```javascript
async function initializeAuth() {
  try {
    // Refresh Token이 httpOnly 쿠키에 있으면 자동 포함
    const response = await fetch("/api/auth/refresh", {
      method: "POST",
      credentials: "include",
    });

    if (response.ok) {
      const { access_token, expires_in } = await response.json();
      tokenManager.setToken(access_token, expires_in);
      return true; // 로그인 상태
    }
  } catch (error) {
    // Refresh Token 없음 또는 만료
  }
  return false; // 비로그인 상태
}
```

---

## 4. Refresh Token Rotation

### 4-1. 왜 Rotation이 필요한가

Refresh Token은 수명이 길다(보통 30일 이상).
탈취되면 장기간 악용될 수 있다.

Rotation은 Refresh Token을 사용할 때마다 새로운 Refresh Token을 발급하고 기존 것을 무효화한다.
탈취된 토큰이 사용되면, 정상 사용자의 다음 갱신 요청에서 충돌이 감지된다.

### 4-2. 서버 구현 핵심

```python
# FastAPI 예시
@router.post("/auth/refresh")
async def refresh_token(request: Request, db: AsyncSession = Depends(get_db)):
    refresh_token = request.cookies.get("refresh_token")
    if not refresh_token:
        raise HTTPException(status_code=401, detail="No refresh token")

    # DB에서 토큰 조회
    token_record = await db.scalar(
        select(RefreshToken).where(
            RefreshToken.token == refresh_token,
            RefreshToken.revoked == False
        )
    )

    if not token_record:
        # 이미 사용된 토큰 — 토큰 재사용 감지
        # 해당 사용자의 모든 Refresh Token 무효화 (탈취 대응)
        await revoke_all_user_tokens(db, token_record.user_id if token_record else None)
        raise HTTPException(status_code=401, detail="Token reuse detected")

    # 기존 토큰 무효화
    token_record.revoked = True
    await db.commit()

    # 새 토큰 발급
    new_access_token = create_access_token(token_record.user_id)
    new_refresh_token = create_refresh_token(token_record.user_id)

    # 새 Refresh Token DB 저장
    db.add(RefreshToken(
        user_id=token_record.user_id,
        token=new_refresh_token,
        expires_at=datetime.utcnow() + timedelta(days=30)
    ))
    await db.commit()

    response = JSONResponse({"access_token": new_access_token, "expires_in": 300})
    response.set_cookie(
        "refresh_token", new_refresh_token,
        httponly=True, secure=True, samesite="strict",
        path="/api/auth", max_age=2592000
    )
    return response
```

### 4-3. Race Condition 주의

여러 탭에서 동시에 갱신 요청이 오면 Rotation이 충돌한다.
첫 번째 요청이 성공하고 나머지는 이미 무효화된 토큰으로 실패한다.

해결책은 짧은 시간(수 초) 내 재사용은 허용하거나, 패밀리 기반 토큰 관리를 적용하는 것이다.
혹은 앞 절의 클라이언트 큐 패턴으로 갱신 요청을 단일화하면 된다.

---

## 5. BFF 토큰 프록시 패턴

### 5-1. BFF(Backend For Frontend)란

SPA가 Authorization Server와 직접 통신하면 Client Secret을 노출하거나 토큰을 클라이언트에 보관해야 한다.

BFF는 SPA와 백엔드 사이에 위치하는 서버다.
SPA는 BFF에만 요청하고, BFF가 Authorization Server와 통신하며 토큰을 관리한다.

```
[브라우저 SPA]
     |
     | (세션 쿠키만 사용)
     v
[BFF 서버]
     |
     | (Client Secret + Access Token 관리)
     |-----> [Authorization Server]
     |
     | (Bearer Token)
     v
[Resource API 서버]
```

### 5-2. BFF 구현 핵심 흐름

BFF는 SPA에게 세션 쿠키만 발급한다.
실제 OAuth 토큰은 BFF 서버 메모리나 Redis에 저장한다.

```python
# BFF의 콜백 처리
@router.post("/bff/auth/callback")
async def bff_callback(
    code: str,
    state: str,
    redis: Redis = Depends(get_redis),
    db: AsyncSession = Depends(get_db)
):
    # Authorization Server로 토큰 교환 (Client Secret은 서버에만 있음)
    token_response = await exchange_code_for_tokens(
        code=code,
        client_id=settings.CLIENT_ID,
        client_secret=settings.CLIENT_SECRET,  # 절대 프론트엔드에 노출 안 됨
        redirect_uri=settings.REDIRECT_URI
    )

    # 세션 생성 및 토큰을 Redis에 저장
    session_id = generate_session_id()
    await redis.setex(
        f"session:{session_id}",
        3600,
        json.dumps({
            "access_token": token_response["access_token"],
            "refresh_token": token_response["refresh_token"],
            "user_id": token_response["sub"]
        })
    )

    # SPA에는 세션 쿠키만 발급
    response = RedirectResponse(url="/dashboard")
    response.set_cookie(
        "session_id", session_id,
        httponly=True, secure=True, samesite="strict",
        max_age=3600
    )
    return response


# BFF의 API 프록시
@router.get("/bff/api/{path:path}")
async def bff_proxy(
    path: str,
    request: Request,
    redis: Redis = Depends(get_redis)
):
    session_id = request.cookies.get("session_id")
    if not session_id:
        raise HTTPException(status_code=401)

    session_data = await redis.get(f"session:{session_id}")
    if not session_data:
        raise HTTPException(status_code=401)

    session = json.loads(session_data)

    # 실제 API 서버로 Bearer Token 포함하여 프록시
    async with httpx.AsyncClient() as client:
        response = await client.request(
            method=request.method,
            url=f"{settings.API_BASE_URL}/{path}",
            headers={"Authorization": f"Bearer {session['access_token']}"},
            content=await request.body()
        )
    return Response(
        content=response.content,
        status_code=response.status_code,
        headers=dict(response.headers)
    )
```

### 5-3. BFF 패턴의 장점

- Client Secret이 서버에만 존재한다.
- Access Token과 Refresh Token이 브라우저에 도달하지 않는다.
- SPA는 세션 쿠키만으로 인증한다 — XSS로 토큰을 탈취할 수 없다.
- Authorization Server 변경 시 BFF만 수정하면 된다.

---

## 6. CSRF/XSS 방어와 토큰 관리의 관계

### 6-1. XSS와 토큰 탈취

XSS(Cross-Site Scripting)는 공격자의 스크립트가 사용자 브라우저에서 실행되는 공격이다.
스크립트가 실행되면 `localStorage`, `sessionStorage`, JavaScript 변수에 있는 모든 것을 읽을 수 있다.

httpOnly 쿠키에 있는 Refresh Token만 XSS로부터 안전하다.
따라서 가장 민감한 토큰(Refresh Token)은 반드시 httpOnly 쿠키에 보관한다.

Content Security Policy(CSP)는 XSS 자체를 막는 방어선이다.

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
```

### 6-2. CSRF와 쿠키 기반 인증

CSRF(Cross-Site Request Forgery)는 다른 사이트에서 사용자의 쿠키를 이용해 요청을 위조하는 공격이다.
httpOnly 쿠키를 사용하면 CSRF가 가능하다 — 쿠키는 자동으로 포함되기 때문이다.

방어 전략은 두 가지를 함께 쓴다.

**SameSite 쿠키:**
```http
Set-Cookie: refresh_token=...; SameSite=Strict; Secure; HttpOnly
```
`Strict`는 크로스사이트 요청에 쿠키를 포함하지 않는다.
서드파티 서비스 연동이 있다면 `Lax`로 완화하고 CSRF 토큰을 추가한다.

**Double Submit Cookie 패턴:**
```javascript
// 서버가 CSRF 토큰을 일반 쿠키(JS 접근 가능)로 발급
const csrfToken = getCookie("csrf_token");

// 모든 변경 요청에 헤더로 포함
fetch("/api/data", {
  method: "POST",
  credentials: "include",
  headers: {
    "X-CSRF-Token": csrfToken,
    "Content-Type": "application/json",
  },
  body: JSON.stringify(data),
});
```

서버는 쿠키의 CSRF 토큰과 헤더의 토큰을 비교한다.
크로스사이트 요청은 헤더를 설정할 수 없으므로 검증을 통과하지 못한다.

### 6-3. 서버에서 해야 할 일

```python
# CSRF 미들웨어 예시 (FastAPI)
class CSRFMiddleware(BaseHTTPMiddleware):
    SAFE_METHODS = {"GET", "HEAD", "OPTIONS"}

    async def dispatch(self, request: Request, call_next):
        if request.method in self.SAFE_METHODS:
            return await call_next(request)

        # 변경 요청에는 CSRF 검증
        cookie_token = request.cookies.get("csrf_token")
        header_token = request.headers.get("X-CSRF-Token")

        if not cookie_token or not header_token:
            return Response("CSRF token missing", status_code=403)

        if not secrets.compare_digest(cookie_token, header_token):
            return Response("CSRF token mismatch", status_code=403)

        return await call_next(request)
```

`secrets.compare_digest`를 쓰는 이유는 타이밍 공격을 막기 위해서다.
일반 `==` 비교는 문자열 길이에 따라 실행 시간이 달라져 공격에 이용될 수 있다.

---

## 7. 실전 체크리스트

서버 엔지니어가 프론트엔드 팀과 인증을 통합할 때 확인해야 할 항목이다.

**토큰 발급 및 저장:**
- [ ] Access Token 수명이 5~15분 이하인가
- [ ] Refresh Token이 httpOnly, Secure, SameSite 쿠키로 발급되는가
- [ ] Refresh Token Rotation이 적용되어 있는가
- [ ] 재사용 감지 시 해당 사용자 세션 전체를 무효화하는가

**PKCE 및 리다이렉트:**
- [ ] 허용된 redirect_uri 목록이 정확히 등록되어 있는가
- [ ] state 파라미터 검증이 콜백에서 수행되는가
- [ ] code_verifier/code_challenge 검증이 서버에서 이루어지는가

**CSRF/XSS 방어:**
- [ ] CSP 헤더가 설정되어 있는가
- [ ] 변경 요청에 CSRF 토큰 검증이 있는가
- [ ] SameSite 쿠키 정책이 적용되어 있는가

**Silent Refresh:**
- [ ] 프론트엔드가 단일 갱신 요청만 보내도록 큐 처리를 하는가
- [ ] 새로고침 시 초기 토큰 갱신 로직이 있는가
- [ ] Refresh Token 만료 시 사용자에게 재로그인을 유도하는가

---

## 8. 정리

OAuth Authorization Code + PKCE는 프론트엔드와 서버가 함께 구현해야 하는 프로토콜이다.
서버가 토큰 검증과 Rotation을 담당한다면, 프론트엔드는 올바른 저장 위치와 갱신 타이밍을 담당한다.

핵심 원칙은 하나다.
**토큰이 XSS에 노출되지 않게 하려면 httpOnly 쿠키를, CSRF를 막으려면 SameSite + CSRF 토큰을 함께 쓴다.**

BFF 패턴은 이 모든 복잡성을 서버로 흡수한다.
SPA는 세션 쿠키만 다루고, OAuth 토큰의 생명주기는 BFF가 전담한다.

다음 편에서는 웹소켓 연결 시 인증 처리와 실시간 세션 무효화를 다룬다.

---

## 참고 자료

- [RFC 7636 — Proof Key for Code Exchange by OAuth Public Clients](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [OWASP — Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP — HTML5 Security Cheat Sheet (Token Storage)](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html)
- [Okta — Implement the OAuth 2.0 Authorization Code with PKCE Flow](https://developer.okta.com/blog/2019/08/22/okta-authjs-pkce)
- [Auth0 — Token Storage](https://auth0.com/docs/secure/security-guidance/data-security/token-storage)
