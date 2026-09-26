
# 1. 概述

OAuth 2.0 定义了多种授权模式（Grant Type）以适应不同的客户端场景。虽然 OAuth 2.1 已将授权码模式（+ PKCE）提升为唯一推荐的流程，但理解其他模式对于维护旧系统、调试和面试仍有重要意义。本章系统讲解各授权模式的流程、安全风险与适用场景。

# 2. 授权模式总览

| 授权模式 | Grant Type | OAuth 2.0 | OAuth 2.1 | 安全级别 | 主要风险 |
|---------|-----------|-----------|-----------|---------|---------|
| **授权码模式** | authorization_code | ✅ 推荐 | ✅ 唯一 | 高 | 需 PKCE |
| **隐式模式** | implicit | ✅ 已过时 | ❌ 移除 | 低 | Token 在 URL 中 |
| **密码模式** | password | ✅ 已过时 | ❌ 移除 | 极低 | 凭据暴露给客户端 |
| **客户端凭证** | client_credentials | ✅ | ✅ | 高 | 无用户上下文 |
| **设备码模式** | urn:ietf:params:oauth:grant-type:device_code | ✅ | ✅ | 中 | 用户确认时效 |
| **刷新令牌** | refresh_token | ✅ | ✅ | — | Token 泄露风险 |

# 3. 客户端凭证模式（Client Credentials Grant）

## 3.1 适用场景

```
适用：服务端到服务端通信（Machine-to-Machine, M2M）
不适用：需要用户身份的端到端认证

示例场景：
- 后端微服务 A 访问微服务 B 的 API
- CI/CD 流水线调用自动化 API
- 定时任务访问系统 API
```

## 3.2 流程

```
客户端 (Service A)                    授权服务器
        │                                │
        │ POST /token                    │
        │ grant_type=client_credentials  │
        │ &scope=read                    │
        │ Authorization: Basic           │
        │  $(base64(client_id:secret))  │
        │───────────────────────────────►│
        │                                │ 验证 client 凭据
        │                                │ 签发 Access Token
        │                                │ (无用户上下文，无 Refresh Token)
        │ {                              │
        │   access_token: "xxx",         │
        │   token_type: "Bearer",        │
        │   expires_in: 3600             │
        │ }                              │
        │◄───────────────────────────────│
        │                                │
        │ 使用 access_token 访问资源      │
        │ Authorization: Bearer xxx      │
```

## 3.3 Java 实现

```java
// 客户端凭证模式 Token 获取
public String getClientCredentialsToken() {
    RestTemplate restTemplate = new RestTemplate();

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
    headers.setBasicAuth("service-a", "service-a-secret");  // client_id:client_secret

    MultiValueMap<String, String> body = new LinkedMultiValueMap<>();
    body.add("grant_type", "client_credentials");
    body.add("scope", "read write");

    HttpEntity<MultiValueMap<String, String>> request =
        new HttpEntity<>(body, headers);

    ResponseEntity<TokenResponse> response = restTemplate.postForEntity(
        "https://auth-server.com/oauth2/token", request, TokenResponse.class);

    return response.getBody().getAccessToken();
}

// Spring Security OAuth2 Client 配置
// application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          my-service:
            client-id: service-a
            client-secret: service-a-secret
            authorization-grant-type: client_credentials
            scope: read,write
        provider:
          my-service:
            token-uri: https://auth-server.com/oauth2/token
```

## 3.4 安全注意事项

```
风险点：
1. client_secret 仅适用于后端应用（SPA/移动端无法保密）
2. 返回的 Token 不带用户信息（sub 是 client_id）
3. 授权粒度粗（同一个 client 下所有实例共享权限）

防御：
1. 使用 private_key_jwt 替代 client_secret
2. 结合 Scope 实现细粒度权限
3. 短期 Token + 定期轮换
```

# 4. 隐式模式（Implicit Grant）—— 已废弃

## 4.1 为什么被废弃

```
隐式模式设计初衷：纯前端 SPA 无法保护 client_secret

流程（隐患标注 ❌）：
GET /authorize?
    response_type=token         ← 直接返回 token
    &client_id=spa-app
    &redirect_uri=https://spa.com/callback

302 → https://spa.com/callback#access_token=eyJhbGci...&expires_in=3600

❌ Token 出现在 URL fragment 中
❌ 可能通过 Referer header 泄露
❌ 浏览器历史记录中可见
❌ 前端 JavaScript 可读取 fragment（XSS 风险）
❌ 无 Refresh Token（用户频繁重授权）
```

## 4.2 替代方案

| 旧方案（隐式模式） | 新方案 |
|-------------------|--------|
| SPA 直接获取 Token | **BFF 模式**：SPA → BFF（后端） → 授权服务器（授权码 + PKCE） |
| SPA 通过 iframe 静默刷新 | Refresh Token Rotation + 后端管理 |
| token 存 localStorage | BFF 的 HttpOnly Cookie |

# 5. 密码模式（Resource Owner Password Credentials Grant）

## 5.1 流程

```
客户端直接收集用户的用户名和密码，代为请求 Token

POST /token
    grant_type=password
    &username=zhangsan
    &password=P@ssw0rd123
    &scope=read write
    &client_id=my-app
    &client_secret=xxx

→ {access_token, refresh_token, ...}
```

## 5.2 为什么被废弃

```
❌ 核心问题：客户端接触用户的原始密码！
   - 第三方应用直接看到用户密码
   - 无法区分是用户还是客户端伪造
   - 用户无法撤销对某个应用的授权（改了密码所有应用都失效）

❌ 破坏了 OAuth 的授权模型：
   - OAuth = 用户授权应用访问资源，无需给密码
   - 密码模式 = 用户把密码给了应用 → 退化为共享密码

✅ 唯一合理场景：第一方应用（自家应用）
   如：官方的 iOS/Android 客户端
   但即使是第一方应用，也推荐用授权码 + PKCE
```

## 5.3 安全考虑

```java
// 如果必须使用密码模式（仅第一方应用），加强防护：

// 1. 强制 MFA（多因素认证）
// 2. 设备绑定（Device Binding）
// 3. 异常登录检测（异地、新设备）
// 4. 限制 scope（不允许 admin 范围）
// 5. 实现 Token 撤销功能

@PostMapping("/token")
public TokenResponse passwordGrant(
        @RequestParam("grant_type") String grantType,
        @RequestParam("username") String username,
        @RequestParam("password") String password,
        @RequestParam("client_id") String clientId,
        @RequestParam("client_secret") String clientSecret) {

    // ❌ 这是密码模式的原生实现 —— 已不推荐
    // ✅ 请改用授权码模式 + PKCE
    throw new UnsupportedGrantTypeException("password grant is deprecated");
}
```

# 6. 设备码模式（Device Code Grant）

## 6.1 适用场景

```
适用：输入受限设备（没有浏览器或键盘体验差）
"""
- 智能电视 (Smart TV)
- 打印机
- IoT 设备
- CLI 工具 (如 GitHub CLI)
- 游戏主机
"""

### 1.6.2. 流程详解

```
设备                     授权服务器                用户手机上浏览器
 │                           │                           │
 │ 1. POST /device/code     │                           │
 │    client_id=xxx         │                           │
 │──────────────────────────►│                           │
 │                           │ 2. 生成设备码和用户码     │
 │ 3. 返回：                │                           │
 │  {                       │                           │
 │    device_code: "GMMhm...", │                        │
 │    user_code: "WDJB-MJHT",  │                        │
 │    verification_uri:       │                         │
 │      "https://auth.com/device",                       │
 │    verification_uri_       │                         │
 │      complete:             │                         │
 │      "https://auth.com/device?user_code=WDJB-MJHT",  │
 │    expires_in: 600,        │                         │
 │    interval: 5             │                         │
 │  }                       │                           │
 │◄──────────────────────────│                           │
 │                           │                           │
 │ 4. 展示 user_code         │                           │
 │    "请在浏览器访问          │                           │
 │     https://auth.com/device│                           │
 │     并输入代码: WDJB-MJHT" │                           │
 │                           │                           │
 │                           │   5. 用户访问 URL 并输入  │
 │                           │      代码 WDJB-MJHT       │
 │                           │◄──────────────────────────│
 │                           │                           │
 │                           │   6. 用户登录并确认授权    │
 │                           │◄──────────────────────────│
 │                           │                           │
 │ 7. 轮询 POST /token      │                           │
 │    grant_type=            │                           │
 │      urn:ietf:params:     │                           │
 │      oauth:grant-type:    │                           │
 │      device_code          │                           │
 │    &device_code=GMMhm...  │                           │
 │    &client_id=xxx         │                           │
 │    (每 interval 秒一次)   │                           │
 │──────────────────────────►│                           │
 │                           │                           │
 │                           │ 8. 用户已授权 → 返回Token │
 │ 9. {access_token,...}    │                           │
 │◄──────────────────────────│                           │
```

### 1.6.3. Java 实现

```java
public class DeviceCodeFlow {

    private static final String DEVICE_AUTH_URL = "https://auth-server.com/device/code";
    private static final String TOKEN_URL = "https://auth-server.com/token";

    /**
     * 第一步：发起设备码请求
     */
    public DeviceCodeResponse requestDeviceCode() {
        RestTemplate restTemplate = new RestTemplate();

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("client_id", "tv-app");
        params.add("scope", "openid profile");

        DeviceCodeResponse response = restTemplate.postForObject(
            DEVICE_AUTH_URL, params, DeviceCodeResponse.class);

        // 展示给用户
        System.out.println("请访问: " + response.getVerificationUri());
        System.out.println("输入代码: " + response.getUserCode());

        return response;
    }

    /**
     * 第二步：轮询获取 Token（直到用户授权或超时）
     */
    public TokenResponse pollForToken(DeviceCodeResponse deviceCode) {
        RestTemplate restTemplate = new RestTemplate();
        long intervalInMs = deviceCode.getInterval() * 1000L;
        long deadline = System.currentTimeMillis()
            + deviceCode.getExpiresIn() * 1000L;

        while (System.currentTimeMillis() < deadline) {
            try {
                MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
                params.add("grant_type",
                    "urn:ietf:params:oauth:grant-type:device_code");
                params.add("device_code", deviceCode.getDeviceCode());
                params.add("client_id", "tv-app");

                ResponseEntity<TokenResponse> response = restTemplate.postForEntity(
                    TOKEN_URL, params, TokenResponse.class);

                if (response.getStatusCode() == HttpStatus.OK) {
                    return response.getBody();  // 用户已授权！
                }
            } catch (HttpClientErrorException e) {
                if ("authorization_pending".equals(getErrorCode(e))) {
                    // 用户还没授权，等待后继续轮询
                    Thread.sleep(intervalInMs);
                    continue;
                } else if ("slow_down".equals(getErrorCode(e))) {
                    // 服务器要求降低轮询频率
                    intervalInMs += 5000;
                    Thread.sleep(intervalInMs);
                    continue;
                } else {
                    throw e;  // 其他错误
                }
            }
        }

        throw new RuntimeException("Device code expired");
    }
}

@Data
class DeviceCodeResponse {
    private String deviceCode;
    private String userCode;
    private String verificationUri;
    private String verificationUriComplete;
    private int expiresIn;    // 秒
    private int interval;     // 轮询间隔（秒）
}
```

### 1.6.4. 错误响应

```json
// 用户尚未授权
{
    "error": "authorization_pending",
    "error_description": "The user has not yet completed authorization"
}

// 轮询过快
{
    "error": "slow_down",
    "error_description": "The polling interval is too frequent"
}

// 用户拒绝授权
{
    "error": "access_denied",
    "error_description": "The user denied the authorization request"
}

// 设备码过期
{
    "error": "expired_token",
    "error_description": "The device_code has expired"
}
```

## 1.7. 刷新令牌模式（Refresh Token Grant）

### 1.7.1. 流程

```
POST /token
    grant_type=refresh_token
    &refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
    &client_id=my-app
    &client_secret=xxx

→ {
    access_token: "new_access_token...",
    refresh_token: "new_refresh_token...", // 可选（Token Rotation）
    token_type: "Bearer",
    expires_in: 3600
}
```

### 1.7.2. 自动刷新实现（Spring）

```java
@Component
public class TokenRefresher {

    @Autowired
    private RestTemplate restTemplate;

    /**
     * 带自动刷新的 API 调用
     */
    public <T> T callWithAutoRefresh(String url, Class<T> responseType) {
        try {
            return doApiCall(url, responseType);
        } catch (HttpClientErrorException e) {
            if (e.getStatusCode() == HttpStatus.UNAUTHORIZED) {
                // Token 过期，尝试刷新
                refreshToken();
                // 重试
                return doApiCall(url, responseType);
            }
            throw e;
        }
    }

    private String refreshToken() {
        String refreshToken = tokenStore.getRefreshToken();

        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("grant_type", "refresh_token");
        params.add("refresh_token", refreshToken);
        params.add("client_id", "my-app");
        params.add("client_secret", "xxx());

        TokenResponse newTokens = restTemplate.postForObject(
            "https://auth-server.com/token", params, TokenResponse.class);

        tokenStore.saveTokens(
            newTokens.getAccessToken(),
            newTokens.getRefreshToken()
        );

        return newTokens.getAccessToken();
    }
}
```

## 1.8. Bearer Token 与 DPoP

### 1.8.1. Bearer Token 的问题

```
Bearer Token = "持有即可以使用"

问题：
- 任何人获得 Token 就可以使用（不验证 Token 持有者）
- Token 从 HTTP Header 泄露 → 攻击者可冒用
- 无法绑定到特定客户端

攻击场景：
1. 日志泄露（请求头被记录到日志系统）
2. MITM（中间人攻击）
3. 浏览器插件窃取
4. 第三方 Script 窃取（如广告 SDK）
```

### 1.8.2. DPoP（Demonstration of Proof-of-Possession）

```java
// DPoP = 证明 Token 持有者确实是合法客户端

// 每个请求携带 DPoP Proof JWT：
// 1. 客户端生成公私钥对
// 2. 每次请求创建一个 DPoP JWT
// 3. DPoP JWT 包含：请求的 URL、HTTP 方法、时间戳

POST /resource
Authorization: DPoP eyJhbGciOi...（Access Token）
DPoP: eyJhbGciOi...（DPoP Proof JWT）

// DPoP Proof JWT Payload:
{
    "jti": "unique-nonce",      // 防重放
    "htm": "POST",              // HTTP 方法
    "htu": "https://api.example.com/resource",  // 请求 URL
    "iat": 1712345678,          // 签发时间
    "ath": "base64(sha256(access_token))"  // 绑定到特定的 Access Token
}

// 资源服务器验证：
// 1. DPoP JWT 的签名（使用客户端公钥）
// 2. htm 匹配实际 HTTP 方法
// 3. htu 匹配实际请求 URL
// 4. iat 在允许的时间窗口内
// 5. ath 匹配实际的 Access Token 哈希
// → 即使 Access Token 泄露，攻击者无法创建有效的 DPoP 证明
```

## 1.9. 授权模式选择决策

```
┌──────────────────────────────────────────────────────────────┐
│                   OAuth 2.0 授权模式选择                       │
│                                                               │
│  Q1: 是否有用户参与？                                         │
│   ├── 否 → 客户端凭证模式（M2M）                              │
│   └── 是 → Q2                                                 │
│                                                               │
│  Q2: 客户端是否有后端（能保护 client_secret）？               │
│   ├── 是 → 授权码模式 + PKCE + client_secret_basic/post      │
│   └── 否（SPA） → Q3                                         │
│                                                               │
│  Q3: SPA 是否有 BFF（Backend For Frontend）？                 │
│   ├── 是 → 授权码模式 + PKCE（BFF 代理）                     │
│   └── 否 → 授权码模式 + PKCE + 无 client_secret              │
│                                                               │
│  Q4: 输入受限设备？                                           │
│   ├── 是 → 设备码模式                                         │
│   └── 否 → 继续 Q2/Q3                                        │
│                                                               │
│  ⚠️ 永远不要使用：                                            │
│   - 隐式模式 (implicit)                                       │
│   - 密码模式 (password)                                       │
│   这两个在 OAuth 2.1 中已被移除                               │
└──────────────────────────────────────────────────────────────┘
```

## 1.10. 总结

| 模式 | 核心风险 | 替代方案 |
|------|---------|---------|
| **隐式模式** | Token 暴露在浏览器 URL/Referer | 授权码 + PKCE + BFF |
| **密码模式** | 客户端接触用户原始密码 | 授权码 + PKCE |
| **客户端凭证** | 无用户身份，权限粗 | ✅ 合理场景下安全使用 |
| **设备码模式** | 用户码被偷看，Token 被盗 | ✅ 短有效期 + HTTPS |
| **Refresh Token** | Token 泄露，无法吊销 | Token Rotation + 泄露检测 |
