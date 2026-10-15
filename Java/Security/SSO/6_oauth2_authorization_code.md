
# 1. 概述

**OAuth 2.0 授权码模式（Authorization Code Grant）** 是 OAuth 2.0 中最安全、最常用的授权流程。它是实现"使用微信/Google 登录第三方网站"这类功能的底层协议。2022 年发布的 OAuth 2.1 草案中，授权码模式被提升为**唯一推荐**的前端参与授权流程。

```
┌──────────────────────────────────────────────────────────────────┐
│               OAuth 2.0 授权码模式核心角色                        │
│                                                                   │
│  ┌──────────────┐                                                │
│  │ 资源所有者     │  ← 用户（User）                               │
│  │ (Resource     │     "允许这个应用访问我的 Google 照片"         │
│  │  Owner)       │                                                │
│  └──────┬───────┘                                                │
│         │                                                         │
│         │ 用户操作浏览器                                          │
│         ▼                                                         │
│  ┌──────────────┐     ┌──────────────────┐    ┌──────────────┐   │
│  │ 客户端        │────►│  授权服务器       │    │  资源服务器   │   │
│  │ (Client)     │     │  (Auth Server)   │    │  (Resource)  │   │
│  │ 第三方应用    │◄────│  签发 Token       │    │  受保护数据   │   │
│  └──────┬───────┘     └──────────────────┘    └──────┬───────┘   │
│         │                                            │            │
│         │ 携带 Access Token 访问资源                 │            │
│         └────────────────────────────────────────────┘            │
└──────────────────────────────────────────────────────────────────┘
```

# 2. 为什么授权码模式更安全

## 2.1 核心安全设计

```
              ┌─── 前端 Channel ───┐        ┌─── 后端 Channel ───┐
              │   (不安全)          │        │   (安全)            │
              │                     │        │                     │
浏览器 ────►  授权服务器             │        │                     │
  │                                │        │                     │
  │ ① GET /authorize               │        │                     │
  │────────────────────────────────►│       │                     │
  │                                │        │                     │
  │ ② 302 ?code=xxx&state=zzz      │        │                     │
  │◄────────────────────────────────│       │                     │
  │                                │        │                     │
  │ ③ GET 回调 ?code=xxx           │        │                     │
  │────────────────────► 客户端     │        │                     │
  │                                │        │ ④ POST /token       │
  │                                │        │────────────────────►│
  │                                │        │  code + client_     │
  │                                │        │  secret             │
  │                                │        │                     │
  │                                │        │ ⑤ {access_token}    │
  │                                │        │◄────────────────────│
  │                                │        │                     │
  │ ⑥ 客户端持有 Access Token                               │     │
  │◄────────────────────│          │        │                     │
  │                                │        │                     │
  │   授权码泄露给攻击者 → 无 client_secret → 无法换取 Token       │
```

**三层安全防护：**
1. **授权码（code）**：仅通过前端浏览器传递，短生命周期（通常 1~10 分钟）
2. **client_secret**：仅在后端持有，不经过浏览器
3. **PKCE**：即使 code 被截获，攻击者也无法使用（见 5）

## 2.2 与其他模式的安全对比

| 攻击向量 | 授权码模式 | 隐式模式 | 密码模式 |
|---------|-----------|---------|---------|
| **Token 从 URL 泄露** | ✅ 安全（Token 不经过 URL） | ❌ Token 在 URL 中 | ✅（Token 不在 URL） |
| **Token 从 Referer 泄露** | ✅ 安全 | ❌ Referer 可能泄露 Token | ✅ |
| **授权码窃取** | ✅ PKCE 防护 | 不适用 | 不适用 |
| **client_secret 暴露** | ✅ 仅后端持有 | ❌ 无法保密 | ⚠️ 可能暴露 |
| **CSRF (state 参数)** | ✅ state 防护 | ✅ state | — |
| **浏览器历史记录** | ✅ 安全（仅 code 在 URL） | ❌ Token 在 URL | ✅ |

# 3. 授权码模式详细流程

## 3.1 第一步：请求授权（Authorization Request）

```
用户点击"使用微信登录"
       │
       ▼
浏览器 GET 请求授权服务器：
GET /authorize?
    response_type=code          ← 必须，表示使用授权码模式
    &client_id=my-app           ← 客户端标识符
    &redirect_uri=https://myapp.com/callback  ← 授权成功后的回调地址
    &scope=openid%20profile%20email           ← 请求的权限范围
    &state=随机字符串_防CSRF                   ← 必须，防CSRF
    &code_challenge=PKCE_Challenge             ← PKCE，推荐
    &code_challenge_method=S256                ← PKCE 方法
```

## 3.2 第二步：用户授权

```
授权服务器处理：
1. 验证 client_id 是否已注册
2. 验证 redirect_uri 是否匹配注册的
3. 展示授权页面给用户：
   "应用 'My App' 请求访问您的：公开信息、邮箱地址"
   [ 同意 ]  [ 拒绝 ]
4. 用户点击"同意"
5. 生成一次性授权码：code = random_base64_string
6. 302 重定向到 redirect_uri：
   https://myapp.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=原state
```

## 3.3 第三步：换取 Token（Token Request）

```java
// 客户端后端代码（Java）
@GetMapping("/callback")
public String callback(@RequestParam String code,
                       @RequestParam String state,
                       HttpSession session) {

    // 1. 验证 state（防 CSRF）
    String savedState = (String) session.getAttribute("OAUTH_STATE");
    if (!state.equals(savedState)) {
        throw new SecurityException("Invalid state - possible CSRF attack");
    }

    // 2. 用 code 换取 Token（后端到后端，安全信道）
    RestTemplate restTemplate = new RestTemplate();

    // 构建请求参数
    MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
    params.add("grant_type", "authorization_code");
    params.add("code", code);
    params.add("redirect_uri", "https://myapp.com/callback");
    params.add("client_id", "my-app");
    params.add("client_secret", "client-secret-xxx");

    // 添加 PKCE code_verifier
    String codeVerifier = (String) session.getAttribute("CODE_VERIFIER");
    params.add("code_verifier", codeVerifier);

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
    headers.setBasicAuth("my-app", "client-secret-xxx"); // 也可以用 Basic Auth

    HttpEntity<MultiValueMap<String, String>> entity =
        new HttpEntity<>(params, headers);

    // POST https://auth-server.com/token
    ResponseEntity<TokenResponse> response = restTemplate.postForEntity(
        "https://auth-server.com/token", entity, TokenResponse.class);

    // 3. 处理响应
    TokenResponse tokenResponse = response.getBody();
    // tokenResponse.accessToken: "eyJhbGciOi..."
    // tokenResponse.refreshToken: "8x7d93..."
    // tokenResponse.idToken: "eyJ..." (如果是OIDC)
    // tokenResponse.expiresIn: 3600
    // tokenResponse.tokenType: "Bearer"

    // 4. 存储 Token
    session.setAttribute("ACCESS_TOKEN", tokenResponse.getAccessToken());
    session.setAttribute("REFRESH_TOKEN", tokenResponse.getRefreshToken());

    return "redirect:/home";
}
```

## 3.4 第四步：使用 Token 访问资源

```java
// 携带 Access Token 访问受保护资源
public UserInfo getUserInfo(String accessToken) {
    RestTemplate restTemplate = new RestTemplate();

    HttpHeaders headers = new HttpHeaders();
    headers.setBearerAuth(accessToken);  // Authorization: Bearer eyJhbGci...

    HttpEntity<Void> entity = new HttpEntity<>(headers);

    ResponseEntity<UserInfo> response = restTemplate.exchange(
        "https://auth-server.com/userinfo",  // OIDC UserInfo 端点
        HttpMethod.GET,
        entity,
        UserInfo.class
    );

    return response.getBody();
}
```

## 3.5 Token 响应结构

```json
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "8x7d93f2-1a4b-4c5d-8e9f-0a1b2c3d4e5f",
    "scope": "openid profile email",
    "id_token": "eyJhbGciOiJSUzI1NiJ9.eyJzdWIi...（OIDC）"
}
```

# 4. OAuth 2.0 权限范围（Scope）

## 4.1 Scope 设计原则

```
Scope = 权限粒度 + 用户可理解性

好的 Scope 设计：
- openid      → OIDC 认证
- profile     → 获取用户基本信息（姓名、头像）
- email       → 获取邮箱地址
- phone       → 获取手机号
- read:orders → 只读订单
- write:orders → 可修改订单

不好的 Scope 设计：
- admin       → 太大，含义模糊
- all         → 不安全
- system     → 含义不清
```

## 4.2 Scope 校验流程

```java
// 授权服务器端 Scope 校验
public boolean validateScope(String requestedScope, String registeredScope) {
    Set<String> requested = new HashSet<>(Arrays.asList(requestedScope.split(" ")));
    Set<String> registered = new HashSet<>(Arrays.asList(registeredScope.split(" ")));

    // 请求的 scope 必须是已注册的子集
    if (!registered.containsAll(requested)) {
        Set<String> invalid = new HashSet<>(requested);
        invalid.removeAll(registered);
        throw new InvalidScopeException("Invalid scopes: " + invalid);
    }
    return true;
}
```

# 5. PKCE（Proof Key for Code Exchange）

## 5.1 为什么需要 PKCE

授权码在浏览器 302 重定向中传递，可能被以下方式截获：
- 浏览器历史记录
- Referer header 泄露
- 恶意浏览器插件
- 移动端自定义 URL Scheme 劫持

PKCE 确保：即使授权码被截获，攻击者也无法凭它换取 Token。

## 5.2 PKCE 原理

```
┌─────────────────── PKCE 原理 ───────────────────┐
│                                                    │
│  客户端生成两个随机值：                             │
│                                                    │
│  ① code_verifier (高熵随机字符串, 43-128字符)     │
│     示例: dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
│                                                    │
│  ② code_challenge = SHA256(code_verifier)           │
│     → Base64URL-Encode → 传给授权服务器             │
│                                                    │
│  步骤1: 授权请求时发送 code_challenge               │
│  步骤2: 换 Token 时发送 code_verifier               │
│  步骤3: 授权服务器验证:                              │
│         SHA256(code_verifier) == code_challenge ?   │
│                                                    │
│  攻击者截获了 code，但没有 code_verifier            │
│  → 无法计算出正确的 code_challenge                  │
│  → 无法换取 Token ✓                                │
└────────────────────────────────────────────────────┘
```

## 5.3 Java PKCE 实现

```java
public class PkceUtil {

    /**
     * 生成 Code Verifier
     * 使用 Base64URL 编码的 32 字节随机数 → 43 字符
     */
    public static String generateCodeVerifier() {
        SecureRandom secureRandom = new SecureRandom();
        byte[] codeVerifier = new byte[32];
        secureRandom.nextBytes(codeVerifier);
        return Base64.getUrlEncoder().withoutPadding()
            .encodeToString(codeVerifier);
    }

    /**
     * 生成 Code Challenge (S256)
     */
    public static String generateCodeChallenge(String codeVerifier) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            byte[] digest = md.digest(codeVerifier.getBytes(StandardCharsets.US_ASCII));
            return Base64.getUrlEncoder().withoutPadding()
                .encodeToString(digest);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 验证 PKCE
     */
    public static boolean verifyCodeChallenge(String codeVerifier, String codeChallenge) {
        String computedChallenge = generateCodeChallenge(codeVerifier);
        return MessageDigest.isEqual(
            computedChallenge.getBytes(),
            codeChallenge.getBytes()
        );
    }
}
```

## 5.4 PKCE 在 Web 应用中的使用

```java
@GetMapping("/login")
public String login(HttpSession session) {
    // 1. 生成 PKCE 参数
    String codeVerifier = PkceUtil.generateCodeVerifier();
    String codeChallenge = PkceUtil.generateCodeChallenge(codeVerifier);
    String state = UUID.randomUUID().toString();

    // 2. 存储 code_verifier 和 state 到 session
    session.setAttribute("CODE_VERIFIER", codeVerifier);
    session.setAttribute("OAUTH_STATE", state);

    // 3. 构建授权 URL
    String authUrl = "https://auth-server.com/authorize?"
        + "response_type=code"
        + "&client_id=my-app"
        + "&redirect_uri=https://myapp.com/callback"
        + "&scope=openid+profile+email"
        + "&state=" + state
        + "&code_challenge=" + codeChallenge
        + "&code_challenge_method=S256";

    return "redirect:" + authUrl;
}
```

# 6. Client 认证方式

OAuth 2.0 要求客户端在换取 Token 时证明自己的身份：

## 6.1 客户端认证方法

| 方法 | 说明 | 适用场景 |
|------|------|---------|
| **client_secret_basic** | HTTP Basic Auth: `Authorization: Basic base64(client_id:client_secret)` | 有后端的 Web 应用 |
| **client_secret_post** | 在 POST body 中传递 `client_id` 和 `client_secret` | 同上 |
| **client_secret_jwt** | 使用 HMAC-SHA 签名的 JWT 作为 client_assertion | 需要更强的安全 |
| **private_key_jwt** | 使用 RSA/EC 私钥签名的 JWT 作为 client_assertion | 最高安全级别 |
| **none** | 公开客户端（SPA/移动端），无法保密 | 必须启用 PKCE |

## 6.2 private_key_jwt 示例

```java
public String createClientAssertionJwt(String clientId, String tokenEndpoint) {
    // 使用 RSA 私钥签发 JWT
    PrivateKey privateKey = loadPrivateKey();
    Instant now = Instant.now();

    String jwt = Jwts.builder()
        .issuer(clientId)
        .subject(clientId)
        .audience().add(tokenEndpoint).and()  // aud 必须是 Token 端点
        .issuedAt(Date.from(now))
        .expiration(Date.from(now.plusSeconds(60)))  // 短过期
        .id(UUID.randomUUID().toString())  // jti 防重放
        .signWith(privateKey)  // 使用私钥签名
        .compact();

    return jwt;
}

// 在 Token 请求中使用
MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
params.add("grant_type", "authorization_code");
params.add("code", code);
params.add("redirect_uri", redirectUri);
params.add("client_id", "my-app");
params.add("client_assertion_type",
    "urn:ietf:params:oauth:client-assertion-type:jwt-bearer");
params.add("client_assertion", createClientAssertionJwt(...));
```

# 7. Refresh Token 机制

## 7.1 Refresh Token 流程

```
Access Token 过期
        │
        ▼
POST /token
    grant_type=refresh_token
    &refresh_token=xxx
    &client_id=my-app
    &client_secret=xxx
        │
        ▼
授权服务器验证 refresh_token
  ├─ 有效 → 返回新的 Access Token（可选新的 Refresh Token）
  └─ 无效/过期/已撤销 → 401
```

## 7.2 Refresh Token 轮换（Rotation）

```java
// OAuth 2.0 安全最佳实践：Refresh Token Rotation
// 每次使用 Refresh Token，旧 Token 立即失效，返回新的 Refresh Token

@PostMapping("/token")
public TokenResponse refreshToken(@RequestParam String refreshToken) {
    // 1. 验证 Refresh Token
    StoredRefreshToken stored = refreshTokenStore.get(refreshToken);
    if (stored == null || stored.isExpired()) {
        throw new InvalidGrantException("Invalid refresh_token");
    }

    // 2. 使旧 Refresh Token 失效
    refreshTokenStore.remove(refreshToken);

    // 3. 发布新的 Access Token 和 Refresh Token
    String newAccessToken = tokenService.createAccessToken(stored.getUser());
    String newRefreshToken = tokenService.createRefreshToken(stored.getUser());

    // 4. 存储新 Refresh Token，关联旧 Token 链（用于检测泄露）
    refreshTokenStore.put(newRefreshToken,
        new StoredRefreshToken(stored.getUser(), newAccessToken, refreshToken));

    return TokenResponse.builder()
        .accessToken(newAccessToken)
        .refreshToken(newRefreshToken)
        .expiresIn(3600)
        .build();
}

// 检测 Refresh Token 泄露
// 如果发现某个 Refresh Token 被使用，但它的"前身"已经被使用过
// → 说明 Refresh Token 被攻击者盗用 → 全部 token 撤销
public void detectTokenReuse(String currentToken, StoredRefreshToken stored) {
    if (stored.getPreviousToken() != null && stored.isUsed()) {
        // 撤销整个 token 链
        revokeTokenFamily(stored.getTokenFamily());
        throw new SecurityException("Token reuse detected - possible token theft");
    }
    stored.markAsUsed();
}
```

# 8. 常见安全漏洞与防御

## 8.1 CSRF 攻击 → state 参数防御

```
攻击流程：
1. 攻击者用自己的账号获得授权码 code=attacker_code
2. 攻击者诱导受害者访问：https://victim-app.com/callback?code=attacker_code
3. 受害者点击 → victi-app 用 attacker_code 换取攻击者的 Token
4. 受害者"登录"成了攻击者的账号
5. 受害者在 attacker 的账号中进行操作 → 信息泄露

防御：state 参数
GET /authorize?...&state=RANDOM_STRING
回调：/callback?code=xxx&state=RANDOM_STRING
验证：session中的state == 回调中的state ✓
攻击者不知道 session 中的 state → CSRF 失败 ✓
```

## 8.2 重定向 URI 劫持

```
攻击者注册恶意 redirect_uri：
GET /authorize?redirect_uri=https://evil.com&...

防御：
1. 授权服务器严格校验 redirect_uri（精确匹配或白名单）
2. 不允许通配符重定向
3. 忽略 redirect_uri 中的 fragment（#）
```

## 8.3 授权码拦截（PKCE 防御，见 1.5）

## 8.4 Token 泄露防御

```java
// Access Token 安全存储
// ❌ 不要存 localStorage（XSS 可读）
localStorage.setItem("token", accessToken);

// ❌ 不要存 Cookie 且无 HttpOnly（XSS 可读）
document.cookie = "token=" + accessToken;

// ✅ Web 应用：后端 Session 存储，前端不接触 Token
// ✅ SPA：BFF 模式，Token 存在 BFF 的 Session 中
// ✅ 移动端：Keychain (iOS) / Keystore (Android)
```

# 9. OAuth 2.0 vs OAuth 2.1

| 变更项 | OAuth 2.0 | OAuth 2.1 |
|--------|----------|----------|
| **授权模式** | 多种（code/implicit/password/client_credentials） | 仅授权码模式（+ PKCE 强制） |
| **PKCE** | 可选（推荐） | **强制** |
| **隐式模式** | 支持 | **移除** |
| **密码模式** | 支持 | **移除** |
| **Redirect URI** | 要求不严格 | 精确匹配 |
| **Refresh Token** | 无安全要求 | 必须支持发送者约束或一次性 |
| **Bearer Token** | 默认 | 增加 DPoP 支持 |

# 10. 总结

| 要点 | 说明 |
|------|------|
| **授权码模式是最安全的流程** | code 在浏览器，Token 在后端，前端不接触密钥 |
| **PKCE 是必需品** | OAuth 2.1 强制，即使是后端应用也应使用 |
| **state 防 CSRF** | 每次请求生成随机 state，回调时验证 |
| **永不过期的 Refresh Token** | 实现 Token Rotation 检测泄露 |
| **HTTPS 是前提** | 所有通信必须走 HTTPS |
| **redirect_uri 严格校验** | 精确匹配，不允许通配符 |
