# 1. OAuth2 客户端概述

OAuth2 客户端（Client）是代表资源拥有者请求受保护资源的应用。客户端实现的复杂度取决于客户端类型、安全级别和使用的授权模式。

## 1.1 客户端类型与安全分类

```
                    OAuth2 客户端分类
                         │
          ┌──────────────┴──────────────┐
          │                             │
    机密客户端                      公开客户端
    (Confidential)                (Public)
          │                             │
    ┌─────┴─────┐              ┌───────┴────────┐
    │           │              │                 │
  Web 应用   M2M 服务       SPA/浏览器        Native App
  (有后端)  (密钥安全)    (JS 可读源码)     (可反编译)
```

| 类型 | 示例 | 密钥安全 | client_secret | 推荐的授权模式 |
|------|------|----------|---------------|---------------|
| 机密 Web | Spring Boot, Django, Express | 能安全保存 | 使用 | Auth Code + client_secret |
| 机密 M2M | 微服务、定时任务 | 能安全保存 | 使用 | Client Credentials |
| 公开 SPA | React, Vue, Angular | 无法保存 | 不使用 | Auth Code + PKCE |
| 公开 Native | iOS, Android, 桌面 App | 无法保存 | 不使用 | Auth Code + PKCE |

---

# 2. 机密客户端（Web 应用）实现

## 2.1 整体架构

```
┌─────────┐   Browser     ┌─────────────┐   Backend    ┌─────────────┐
│ Browser │<─────────────>│  Web Server  │<───────────>│ Auth Server │
│         │  Session Cookie│ (Backend)   │ API Call    │             │
└─────────┘               └──────┬──────┘             └─────────────┘
                                 │
                                 │ Token Storage
                                 ▼
                          ┌─────────────┐
                          │  Database   │
                          │  / Redis    │
                          └─────────────┘
```

关键设计决策：
1. Access Token **永远不传到浏览器**
2. 使用 Session Cookie（HttpOnly, Secure, SameSite=Lax）维护用户会话
3. 后端代理所有 API 请求，自动附加 Access Token
4. Refresh Token 安全存储在后端数据库（加密）

## 2.2 核心实现流程

### 2.2.1 步骤一：发起授权请求

```java
// 生成 state 和 code_verifier，存储到 session
@GetMapping("/login/oauth2")
public String initiateOAuth2(HttpSession session) {
    String state = generateRandomString(32);
    String codeVerifier = generateCodeVerifier();
    String codeChallenge = sha256Base64(codeVerifier);
    
    session.setAttribute("oauth2_state", state);
    session.setAttribute("oauth2_code_verifier", codeVerifier);
    
    String authUrl = UriComponentsBuilder
        .fromHttpUrl(authorizationEndpoint)
        .queryParam("response_type", "code")
        .queryParam("client_id", clientId)
        .queryParam("redirect_uri", redirectUri)
        .queryParam("scope", "openid profile email")
        .queryParam("state", state)
        .queryParam("code_challenge", codeChallenge)
        .queryParam("code_challenge_method", "S256")
        .build()
        .toUriString();
    
    return "redirect:" + authUrl;
}
```

### 2.2.2 步骤二：处理回调

```java
@GetMapping("/callback")
public String handleCallback(
        @RequestParam("code") String code,
        @RequestParam("state") String state,
        HttpSession session) {
    
    // 1. 验证 state（防 CSRF）
    String savedState = (String) session.getAttribute("oauth2_state");
    if (!Objects.equals(state, savedState)) {
        throw new SecurityException("State mismatch - possible CSRF attack");
    }
    
    // 2. 用 code 换取 Token
    String codeVerifier = (String) session.getAttribute("oauth2_code_verifier");
    TokenResponse tokenResponse = exchangeCodeForToken(code, codeVerifier);
    
    // 3. 验证 ID Token
    IDTokenClaims claims = verifyIdToken(tokenResponse.getIdToken());
    
    // 4. 查找或创建用户
    String sub = claims.getSubject();
    User user = userService.findOrCreateBySub(sub, claims);
    
    // 5. 存储 Token 到数据库（加密）
    tokenService.storeTokens(user.getId(), tokenResponse);
    
    // 6. 清除 OAuth2 临时 session 数据
    session.removeAttribute("oauth2_state");
    session.removeAttribute("oauth2_code_verifier");
    
    // 7. 建立用户会话
    session.setAttribute("user_id", user.getId());
    
    return "redirect:/dashboard";
}
```

### 2.2.3 步骤三：用 code 换 Token

```java
private TokenResponse exchangeCodeForToken(String code, String codeVerifier) {
    HttpHeaders headers = new HttpHeaders();
    headers.setBasicAuth(clientId, clientSecret);
    headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
    
    String body = UriComponentsBuilder.newInstance()
        .queryParam("grant_type", "authorization_code")
        .queryParam("code", code)
        .queryParam("redirect_uri", redirectUri)
        .queryParam("code_verifier", codeVerifier)
        .build()
        .getQuery();
    
    HttpEntity<String> request = new HttpEntity<>(body, headers);
    
    ResponseEntity<TokenResponse> response = restTemplate.postForEntity(
        tokenEndpoint, request, TokenResponse.class);
    
    if (!response.getStatusCode().is2xxSuccessful()) {
        throw new OAuth2Exception("Failed to exchange code for token");
    }
    
    return response.getBody();
}
```

## 2.3 Token 管理与自动刷新

```java
// 获取有效的 Access Token（自动处理刷新）
public String getValidAccessToken(String userId) {
    TokenRecord record = tokenRepository.findByUserId(userId);
    
    // 检查 Access Token 是否即将过期（提前 60 秒刷新）
    if (record.getAccessTokenExpiresAt().minusSeconds(60).isBefore(Instant.now())) {
        if (record.getRefreshToken() != null) {
            // 使用 Refresh Token 获取新 Token
            TokenResponse newTokens = refreshAccessToken(record.getRefreshToken());
            record = updateTokens(userId, newTokens);
        } else {
            throw new OAuth2Exception("No Refresh Token available and Access Token expired");
        }
    }
    
    return record.getAccessToken();
}

// 刷新 Access Token
private TokenResponse refreshAccessToken(String refreshToken) {
    HttpHeaders headers = new HttpHeaders();
    headers.setBasicAuth(clientId, clientSecret);
    headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
    
    String body = UriComponentsBuilder.newInstance()
        .queryParam("grant_type", "refresh_token")
        .queryParam("refresh_token", refreshToken)
        .build()
        .getQuery();
    
    HttpEntity<String> request = new HttpEntity<>(body, headers);
    ResponseEntity<TokenResponse> response = restTemplate.postForEntity(
        tokenEndpoint, request, TokenResponse.class);
    
    return response.getBody();
}
```

### 2.3.1 Token 存储表设计

```sql
CREATE TABLE oauth2_tokens (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    access_token TEXT NOT NULL,
    access_token_expires_at TIMESTAMP NOT NULL,
    refresh_token TEXT,
    refresh_token_expires_at TIMESTAMP,
    scope VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_user_id (user_id),
    INDEX idx_access_token_expires (access_token_expires_at)
);
```

> **重要**：Token 在数据库中应使用 AES-256-GCM 加密存储。未加密存储 Token 是严重的安全漏洞。

---

# 3. 公开客户端（SPA）实现

## 3.1 SPA 的安全挑战

```
SPA 的核心困境：
  - 代码在浏览器中执行 → 无法安全存储密钥
  - JS 可访问 localStorage/sessionStorage → Token 易被盗取
  - XSS 漏洞 → Token 完全暴露
  - 第三方 JS 库 → 供应链攻击风险
```

## 3.2 推荐方案：BFF（Backend For Frontend）模式

```
                    BFF 架构
                    
┌──────────┐     ┌──────────────────┐     ┌─────────────┐
│          │     │    BFF Server     │     │             │
│   SPA    │◄───>│  (Thin Backend)   │◄───>│ Auth Server │
│ (React/  │     │                   │     │             │
│  Vue/    │     │ ┌───────────────┐ │     └─────────────┘
│  Angular)│     │ │ Session Store │ │
│          │     │ │ Token Store   │ │
└──────────┘     │ └───────────────┘ │
                 │                   │
  Session        └──────────────────┘   OAuth2
  Cookie                                Protocol
  (HttpOnly,                            (Backend)
  Secure,
  SameSite)
```

```
BFF 模式的优势：

1. Access Token 不出浏览器 — SPA 只持有 HttpOnly Session Cookie
2. Token 刷新在 BFF 完成 — 前端无感知
3. 符合机密客户端安全模型 — 可使用 client_secret
4. 统一 API 网关 — BFF 可充当 API Proxy

实现要点：
- BFF 是轻量级后端（如 express, Spring Boot Thin）
- BFF 与 SPA 同源（Same Origin），避免 CORS 复杂性
- BFF 管理 OAuth2 完整流程（code 换 token, 刷新, 存储）
- SPA 只需要知道 "已登录" 或 "未登录"
```

## 3.3 SPA 直接 OAuth2（无 BFF）—— 需谨慎

如果无法部署 BFF，SPA 直接处理 OAuth2 需要额外安全措施：

```javascript
class OAuth2Client {
  constructor() {
    // Token 只存在内存中（不用 localStorage）
    this.accessToken = null;
    this.refreshToken = null;
  }
  
  // 发起授权
  async initiateLogin() {
    const codeVerifier = this.generateCodeVerifier();
    const codeChallenge = await this.sha256(codeVerifier);
    const state = this.generateRandomState();
    
    // 存到 sessionStorage（仅 state 和 verifier，不是 Token）
    sessionStorage.setItem('oauth_state', state);
    sessionStorage.setItem('code_verifier', codeVerifier);
    
    const params = new URLSearchParams({
      response_type: 'code',
      client_id: this.clientId,
      redirect_uri: this.redirectUri,
      scope: 'openid profile email',
      state: state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256'
    });
    
    window.location.href = `${this.authEndpoint}?${params}`;
  }
  
  // 处理回调
  async handleCallback() {
    const params = new URLSearchParams(window.location.search);
    const code = params.get('code');
    const state = params.get('state');
    
    // 验证 state
    const savedState = sessionStorage.getItem('oauth_state');
    if (state !== savedState) {
      throw new Error('State mismatch - possible CSRF attack');
    }
    
    // 用 code 换 token（POST 请求）
    const codeVerifier = sessionStorage.getItem('code_verifier');
    const tokens = await this.exchangeCodeForToken(code, codeVerifier);
    
    // 清除临时数据
    sessionStorage.removeItem('oauth_state');
    sessionStorage.removeItem('code_verifier');
    
    // 只存到内存！
    this.accessToken = tokens.access_token;
    this.refreshToken = tokens.refresh_token;
    
    // 清除 URL 中的 code 参数（防止浏览器历史泄露）
    window.history.replaceState({}, document.title, '/');
  }
  
  // 自动刷新 Access Token（Service Worker 实现更安全）
  async getValidAccessToken() {
    if (this.isTokenExpiring()) {
      const tokens = await this.refreshTokens(this.refreshToken);
      this.accessToken = tokens.access_token;
      this.refreshToken = tokens.refresh_token;
    }
    return this.accessToken;
  }
}
```

> SPA 直接 OAuth2 的局限性：
> - 页面刷新后 Token 丢失（需静默刷新）
> - XSS 攻击下内存中的 Token 也能被窃取
> - Refresh Token 存在 JS 内存中有泄露风险

---

# 4. 原生移动应用实现

## 4.1 AppAuth 模式

移动端 OAuth2 推荐使用 **AppAuth** 模式（通过系统浏览器完成授权，而非应用内 WebView）。

```
为什么不推荐 WebView？

1. 安全考虑：
   - 用户无法验证 URL（无地址栏）
   - App 可以通过 JS 注入窃取密码
   - 无法使用系统级密码管理器
   
2. 用户体验：
   - 无法利用已登录的浏览器会话（SSO 失效）
   - 无法使用系统级自动填充
   
3. AppAuth 模式（推荐）：
   - 启动系统浏览器（Chrome Custom Tabs / SFSafariViewController）
   - 通过 custom URL scheme 或 App Links / Universal Links 回调
   - App 永远不碰用户密码
```

### 4.1.1 AppAuth 回调方式

```
方式一：Custom URL Scheme（不推荐）
  myapp://callback?code=xxx
  问题：多 App 可注册同一 scheme

方式二：App Links（Android）/ Universal Links（iOS）（推荐）
  https://myapp.example.com/callback?code=xxx
  优势：由系统验证域名归属，唯一且安全

方式三：Claimed HTTPS（RFC 8252）
  本地 HTTP 服务器监听 127.0.0.1 端口
  只在首次授权时使用，不用于生产
```

## 4.2 Android 实现要点

```java
// 使用 AppAuth 库
// https://github.com/openid/AppAuth-Android

AuthorizationServiceConfiguration serviceConfig =
    new AuthorizationServiceConfiguration(
        Uri.parse("https://auth.example.com/authorize"),
        Uri.parse("https://auth.example.com/token")
    );

// 生成 PKCE
String codeVerifier = CodeVerifierUtil.generateRandomCodeVerifier();
String codeChallenge = CodeVerifierUtil.deriveCodeVerifierChallenge(codeVerifier);

AuthorizationRequest.Builder builder = new AuthorizationRequest.Builder(
    serviceConfig,
    clientId,
    ResponseTypeValues.CODE,
    Uri.parse("https://myapp.example.com/callback")
);

AuthorizationRequest request = builder
    .setScope("openid profile email")
    .setCodeVerifier(codeVerifier, codeChallenge, "S256")
    .build();

// 启动授权
AuthorizationService authService = new AuthorizationService(this);
authService.performAuthorizationRequest(
    request,
    PendingIntent.getActivity(this, 0, new Intent(), 0)
);
```

## 4.3 Token 安全存储

```
Android Token 存储最佳实践：

1. Access Token → EncryptedSharedPreferences
2. Refresh Token → Android Keystore（硬件级加密）
3. 加密实现：
   
   MasterKey masterKey = new MasterKey.Builder(context)
       .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
       .build();

   EncryptedSharedPreferences sharedPreferences = EncryptedSharedPreferences.create(
       context,
       "secure_oauth_tokens",
       masterKey,
       EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
       EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
   );
```

---

# 5. M2M 客户端实现（Client Credentials）

## 5.1 典型架构

```java
// 服务 A 请求 Token 访问服务 B

public class ServiceAClient {
    private final RestTemplate restTemplate;
    private CachedToken cachedToken;
    
    // 获取 Token（带缓存和自动刷新）
    public String getAccessToken() {
        if (cachedToken == null || cachedToken.isExpiringSoon()) {
            cachedToken = requestNewToken();
        }
        return cachedToken.getAccessToken();
    }
    
    private CachedToken requestNewToken() {
        HttpHeaders headers = new HttpHeaders();
        headers.setBasicAuth(clientId, clientSecret);
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        
        String body = "grant_type=client_credentials&scope=read";
        HttpEntity<String> request = new HttpEntity<>(body, headers);
        
        ResponseEntity<TokenResponse> response = restTemplate.postForEntity(
            tokenEndpoint, request, TokenResponse.class);
        
        TokenResponse token = response.getBody();
        return new CachedToken(
            token.getAccessToken(),
            Instant.now().plusSeconds(token.getExpiresIn())
        );
    }
}

class CachedToken {
    private final String accessToken;
    private final Instant expiresAt;
    
    boolean isExpiringSoon() {
        return Instant.now().plusSeconds(60).isAfter(expiresAt);
    }
}
```

## 5.2 M2M Token 缓存策略

```
M2M Token 缓存的最佳实践：

1. 使用 Guava Cache / Caffeine
   - 基于时间的过期策略
   - 自动异步刷新
   
2. 分布式缓存（Redis）
   - key: oauth2:token:{client_id}
   - value: access_token
   - TTL: expires_in - 60（提前 60 秒过期）
   
3. 熔断与降级
   - 授权服务器不可用时，已有 Token 继续使用
   - 兜底：使用本地缓存的最后可用 Token

Java 示例（Caffeine）：
LoadingCache<String, String> tokenCache = Caffeine.newBuilder()
    .expireAfterWrite(tokenExpiresIn - 60, TimeUnit.SECONDS)
    .build(key -> requestNewToken());
```

---

# 6. 客户端注册规范

## 6.1 客户端元数据

```
注册时需要提供的信息：

核心信息：
  - client_name: "My App"
  - application_type: "web" | "native"  （OIDC）
  - redirect_uris: ["https://..."]
  - grant_types: ["authorization_code", "refresh_token"]
  - response_types: ["code"]

安全配置：
  - token_endpoint_auth_method: "client_secret_basic" | "private_key_jwt" | "none"
  - tls_client_auth_subject_dn: （mTLS 证书的 DN）
  - jwks_uri: （用于 private_key_jwt 认证的 JWKS 端点）
  - request_object_signing_alg: "RS256"

功能配置：
  - scope: "openid profile email"
  - post_logout_redirect_uris: ["https://..."]
  - backchannel_logout_uri: "https://..."
```

## 6.2 客户端认证方式

```
方式一：client_secret_basic（最常用）
  Authorization: Basic base64(client_id:client_secret)
  
方式二：client_secret_post
  POST body: client_id=xxx&client_secret=xxx
  不推荐 — Body 参数可能被中间件日志记录

方式三：private_key_jwt（高级，推荐）
  客户端用私钥签名 JWT 作为认证凭证
  无需传输 client_secret
  client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
  client_assertion=eyJhbGciOiJSUzI1NiIs...

方式四：mTLS（双向 TLS）
  客户端证书认证
  最安全，但部署复杂度高
```

---

# 7. 客户端常见陷阱与解决方案

| 陷阱 | 问题 | 解决方案 |
|------|------|----------|
| redirect_uri 不匹配 | 授权服务器拒绝请求 | 确保完整的 redirect_uri 注册（含路径） |
| state 参数丢失 | CSRF 风险 | 在 session/cache 中持久化 state，回调时验证 |
| Token 存在 localStorage | XSS 可窃取 | 使用 BFF 模式或仅存内存 |
| Access Token 过期未处理 | API 调用失败 | 实现自动刷新机制（提前 60 秒刷新） |
| Refresh Token 未轮换 | 安全问题 | 每次刷新后更新 Refresh Token |
| 移动端使用 WebView | 密码窃取风险 | 使用系统浏览器 + AppAuth |
| 日志中记录 Token | Token 泄露 | 配置日志脱敏，禁止记录 Authorization Header |
| HTTP 传输 | Token 被嗅探 | 强制 HTTPS，配置 HSTS |
