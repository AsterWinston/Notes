# 1. JWT 认证流程概述

JWT 认证的核心思想是**无状态认证**：服务端不保存会话信息，客户端持有包含用户信息的自包含令牌。本章深入讲解 JWT 在身份认证中的完整交互流程、令牌生命周期管理、双令牌模式（Access Token + Refresh Token）的设计与实现。

---

# 2. 基本认证流程

## 2.1 一次完整的 JWT 认证交互

```
┌──────────┐                          ┌──────────┐                    ┌──────────┐
│  Client  │                          │  Server  │                    │ Database │
│ (Browser)│                          │          │                    │          │
└────┬─────┘                          └────┬─────┘                    └────┬─────┘
     │                                     │                               │
     │  1. POST /login {user, password}    │                               │
     │────────────────────────────────────>│                               │
     │                                     │  2. 查询用户凭证              │
     │                                     │─────────────────────────────>│
     │                                     │  3. 返回用户信息              │
     │                                     │<─────────────────────────────│
     │                                     │                               │
     │                                     │  4. 验证密码                  │
     │                                     │  5. 构建 Claims              │
     │                                     │  6. 生成 JWT（签名）         │
     │                                     │                               │
     │  7. 200 OK {access_token, refresh_token}                           │
     │<────────────────────────────────────│                               │
     │                                     │                               │
     │  8. 存储 JWT（内存/HttpOnly Cookie）│                               │
     │                                     │                               │
     │  9. GET /api/users                  │                               │
     │     Authorization: Bearer <jwt>     │                               │
     │────────────────────────────────────>│                               │
     │                                     │  10. 提取 JWT                │
     │                                     │  11. 验证签名                │
     │                                     │  12. 检查 exp/iss/aud        │
     │                                     │  13. 提取用户信息            │
     │                                     │                               │
     │  14. 200 OK (user data)             │                               │
     │<────────────────────────────────────│                               │
```

## 2.2 各步骤详解

### 2.2.1 步骤 1-3：凭证验证

```java
@PostMapping("/login")
public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
    // 1. 查找用户
    User user = userService.findByUsername(request.getUsername())
        .orElseThrow(() -> new AuthenticationException("Invalid credentials"));
    
    // 2. 验证密码（使用 BCrypt 等单向哈希）
    if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
        throw new AuthenticationException("Invalid credentials");
    }
    
    // 3. 检查账户状态（锁定、过期、禁用等）
    if (!user.isEnabled()) {
        throw new AuthenticationException("Account disabled");
    }
    
    // 4. 签发 JWT
    return ResponseEntity.ok(issueTokens(user));
}
```

### 2.2.2 步骤 4-6：JWT 签发

```java
public AuthResponse issueTokens(User user) {
    long now = System.currentTimeMillis();
    
    // Access Token: 短有效期
    String accessToken = Jwts.builder()
        .setIssuer("https://auth.example.com")
        .setSubject(user.getId().toString())
        .setAudience("api.example.com")
        .setIssuedAt(new Date(now))
        .setExpiration(new Date(now + accessTokenExpiration))
        .setId(UUID.randomUUID().toString())
        .claim("roles", user.getRoles())
        .claim("email", user.getEmail())
        .signWith(privateKey, SignatureAlgorithm.RS256)
        .compact();
    
    // Refresh Token: 长有效期
    String refreshToken = Jwts.builder()
        .setIssuer("https://auth.example.com")
        .setSubject(user.getId().toString())
        .setIssuedAt(new Date(now))
        .setExpiration(new Date(now + refreshTokenExpiration))
        .setId(UUID.randomUUID().toString())
        .claim("type", "refresh")
        .signWith(refreshSigningKey, SignatureAlgorithm.HS256)
        .compact();
    
    return new AuthResponse(accessToken, refreshToken, accessTokenExpiration / 1000);
}
```

### 2.2.3 步骤 9-13：JWT 验证（服务端过滤器）

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) {
        String header = request.getHeader("Authorization");
        
        if (header == null || !header.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }
        
        String token = header.substring(7);
        
        try {
            Claims claims = Jwts.parserBuilder()
                .setSigningKey(publicKey)
                .requireIssuer("https://auth.example.com")
                .requireAudience("api.example.com")
                .build()
                .parseClaimsJws(token)
                .getBody();
            
            // 构建认证对象
            String userId = claims.getSubject();
            List<String> roles = claims.get("roles", List.class);
            
            UsernamePasswordAuthenticationToken auth = 
                new UsernamePasswordAuthenticationToken(userId, null, 
                    roles.stream().map(SimpleGrantedAuthority::new).toList());
            
            SecurityContextHolder.getContext().setAuthentication(auth);
            
        } catch (JwtException e) {
            SecurityContextHolder.clearContext();
        }
        
        chain.doFilter(request, response);
    }
}
```

---

# 3. Access Token + Refresh Token 双令牌模式

## 3.1 为什么需要双令牌

单令牌模式的问题：

| 问题 | 描述 |
|------|------|
| 安全性 vs 体验的矛盾 | 长有效期不安全，短有效期频繁要求重新登录 |
| 权限变更延迟 | JWT 签发后，用户权限变更需等到令牌过期才生效 |
| 吊销困难 | JWT 无法主动吊销，只能等过期 |

**解决方案：双令牌模式**：

```
Access Token（短有效期 5-30 分钟）：
  - 用于访问 API
  - 包含用户信息
  - 有效期短，即使泄漏影响也有限

Refresh Token（长有效期 7-30 天）：
  - 仅用于刷新 Access Token
  - 不包含敏感信息
  - 服务端存储，可主动吊销
```

## 3.2 双令牌流程

```
客户端                              认证服务                      资源服务
  │                                    │                            │
  │  1. 登录 (user/pass)               │                            │
  │───────────────────────────────────>│                            │
  │  2. {access_token, refresh_token}  │                            │
  │<───────────────────────────────────│                            │
  │                                    │                            │
  │  3. API调用 + access_token         │                            │
  │────────────────────────────────────────────────────────────────>│
  │  4. 200 OK (data)                  │                            │
  │<────────────────────────────────────────────────────────────────│
  │                                    │                            │
  │  ... access_token 过期 ...         │                            │
  │                                    │                            │
  │  5. API调用 + 过期access_token     │                            │
  │────────────────────────────────────────────────────────────────>│
  │  6. 401 Unauthorized               │                            │
  │<────────────────────────────────────────────────────────────────│
  │                                    │                            │
  │  7. POST /refresh {refresh_token}  │                            │
  │───────────────────────────────────>│                            │
  │                                    │ 验证 refresh_token         │
  │                                    │ 检查是否被吊销             │
  │  8. {new_access_token, new_refresh_token}                     │
  │<───────────────────────────────────│                            │
  │                                    │                            │
  │  9. 使用新 access_token 重试       │                            │
  │────────────────────────────────────────────────────────────────>│
  │  10. 200 OK (data)                  │                            │
  │<────────────────────────────────────────────────────────────────│
```

## 3.3 Refresh Token 设计

### 3.3.1 Refresh Token 的类型选择

| 方案 | 描述 | 优缺点 |
|------|------|--------|
| JWT Refresh Token | 自包含，类似于 JWT | 优点：无需服务端存储；缺点：无法主动吊销 |
| Opaque Refresh Token | 随机字符串，服务端存储 | 优点：可主动吊销；缺点：需要数据库查询 |
| 混合方案 | JWT + 版本号 | 优点：验证快速，可批量吊销 |

### 3.3.2 混合方案实现（推荐）

```java
@Entity
public class UserTokenVersion {
    @Id
    private Long userId;
    
    private long tokenVersion; // 令牌版本号
    
    private Instant updatedAt;
}

// 签发 Refresh Token 时携带版本号
public String issueRefreshToken(User user) {
    long version = tokenVersionService.getVersion(user.getId());
    
    return Jwts.builder()
        .setSubject(user.getId().toString())
        .claim("ver", version)       // 版本号
        .claim("type", "refresh")
        .setExpiration(new Date(now + refreshTokenExpiration))
        .signWith(refreshKey)
        .compact();
}

// 验证时检查版本号
public User validateRefreshToken(String token) {
    Claims claims = parseRefreshToken(token);
    long tokenVer = claims.get("ver", Long.class);
    long currentVer = tokenVersionService.getVersion(claims.getSubject());
    
    if (tokenVer != currentVer) {
        throw new TokenRevokedException("Refresh token has been revoked");
    }
    
    return userService.findById(claims.getSubject());
}

// 吊销所有 Refresh Token：递增版本号
public void revokeAllTokens(Long userId) {
    tokenVersionService.incrementVersion(userId);
}
```

## 3.4 Refresh Token 轮换（Rotation）

每次使用 Refresh Token 刷新时，同时签发新的 Refresh Token，使旧的 Refresh Token 失效。

### 3.4.1 轮换目的

- **检测令牌泄漏**：攻击者使用被窃取的 Refresh Token 会使合法用户持有的 Refresh Token 失效，从而暴露泄漏
- **限制泄漏影响**：每个 Refresh Token 只能使用一次

### 3.4.2 轮换实现

```java
public AuthResponse refreshTokens(String refreshToken) {
    // 1. 验证 Refresh Token
    RefreshTokenEntity stored = refreshTokenRepository
        .findByToken(refreshToken)
        .orElseThrow(() -> new InvalidTokenException("Unknown refresh token"));
    
    // 2. 检查是否已被使用（防止重用）
    if (stored.isUsed()) {
        // Refresh Token 被重用 → 可能发生泄漏 → 吊销所有 Token
        refreshTokenRepository.revokeAllForUser(stored.getUserId());
        throw new SecurityException("Token reuse detected, all tokens revoked");
    }
    
    // 3. 标记旧 Token 为已使用
    stored.setUsed(true);
    refreshTokenRepository.save(stored);
    
    // 4. 签发新的 Access Token 和 Refresh Token
    User user = userService.findById(stored.getUserId());
    return issueTokens(user);
}
```

### 3.4.3 Refresh Token 家族（Token Family）

为追踪刷新链，引入 Token Family 概念：

```java
@Entity
public class RefreshTokenEntity {
    @Id
    private String id;
    
    private String family;     // Token 家族 ID（每次登录生成新的家族）
    private String parentId;   // 父 Token ID（上次刷新的 Token）
    private Long userId;
    private boolean used;
    private Instant expiresAt;
    
    // 撤销整个家族
    public static void revokeFamily(String family) {
        // 将家族中所有未使用的 Token 标记为已撤销
    }
}
```

---

# 4. JWT 的客户端存储策略

## 4.1 存储方案对比

| 方案 | 位置 | XSS 防护 | CSRF 防护 | 持久化 | 跨标签页 |
|------|------|----------|-----------|--------|----------|
| `localStorage` | 浏览器存储 | 弱（需自行防护） | 天然免疫 | 持久 | 共享 |
| `sessionStorage` | 浏览器存储 | 弱（需自行防护） | 天然免疫 | 会话级 | 不共享 |
| **HttpOnly Cookie** | 浏览器 Cookie | 强（JS 不可访问） | 需 CSRF 防护 | 可配置 | 共享 |
| **内存变量** | JS 闭包/变量 | 强（无法被 XSS 读取） | 天然免疫 | 不持久 | 不共享 |
| `IndexedDB` | 浏览器存储 | 弱 | 天然免疫 | 持久 | 共享 |
| `Service Worker` | 后台进程 | 较强 | 天然免疫 | 持久 | 共享 |

## 4.2 推荐方案：内存存储 Access Token + HttpOnly Cookie 存储 Refresh Token

```javascript
// 前端（SPA）
class TokenManager {
    // Access Token 存储在内存中（JS 变量/闭包）
    #accessToken = null;
    #refreshPromise = null;
    
    async getAccessToken() {
        if (this.#accessToken && !this.isExpired(this.#accessToken)) {
            return this.#accessToken;
        }
        
        // 防并发刷新
        if (this.#refreshPromise) {
            await this.#refreshPromise;
            return this.#accessToken;
        }
        
        this.#refreshPromise = this.refresh();
        await this.#refreshPromise;
        this.#refreshPromise = null;
        
        return this.#accessToken;
    }
    
    async refresh() {
        // Refresh Token 在 HttpOnly Cookie 中，由浏览器自动发送
        const response = await fetch('/api/auth/refresh', {
            method: 'POST',
            credentials: 'include' // 发送 Cookie
        });
        
        if (!response.ok) {
            // 刷新失败，跳转登录
            window.location.href = '/login';
            return;
        }
        
        const data = await response.json();
        this.#accessToken = data.access_token;
    }
}
```

```java
// 后端：Refresh Token Cookie 配置
@PostMapping("/login")
public ResponseEntity<AuthResponse> login(
        @RequestBody LoginRequest request,
        HttpServletResponse response) {
    
    AuthResponse tokens = authService.authenticate(request);
    
    // Access Token 放在响应体中
    // Refresh Token 放在 HttpOnly Cookie 中
    Cookie refreshCookie = new Cookie("refresh_token", tokens.getRefreshToken());
    refreshCookie.setHttpOnly(true);      // JS 不可访问
    refreshCookie.setSecure(true);         // 仅 HTTPS
    refreshCookie.setPath("/api/auth/refresh"); // 仅刷新接口可见
    refreshCookie.setMaxAge(30 * 24 * 3600); // 30 天
    refreshCookie.setSameSite("Strict");   // 防止 CSRF
    
    response.addCookie(refreshCookie);
    
    return ResponseEntity.ok(new AuthResponse(tokens.getAccessToken()));
}
```

## 4.3 安全最佳实践

```
Access Token:
  - 存储在内存（JavaScript 闭包）
  - 每次页面加载通过 Refresh Token 获取新的
  - 通过 Authorization Header 发送
  - 不存储在 localStorage/sessionStorage（防止 XSS 读取）

Refresh Token:
  - 存储在 HttpOnly、Secure、SameSite Cookie 中
  - 路径限制为 /api/auth/refresh
  - 实现轮换机制
  - 支持服务端吊销
```

---

# 5. 令牌过期处理与无缝刷新

## 5.1 前端拦截器实现

```javascript
// Axios 拦截器：自动刷新过期 Token
let isRefreshing = false;
let failedQueue = [];

const processQueue = (error, token = null) => {
    failedQueue.forEach(prom => {
        if (error) {
            prom.reject(error);
        } else {
            prom.resolve(token);
        }
    });
    failedQueue = [];
};

axios.interceptors.response.use(
    response => response,
    async error => {
        const originalRequest = error.config;
        
        if (error.response?.status === 401 && !originalRequest._retry) {
            if (isRefreshing) {
                // 等待正在进行的刷新
                return new Promise((resolve, reject) => {
                    failedQueue.push({ resolve, reject });
                }).then(token => {
                    originalRequest.headers['Authorization'] = `Bearer ${token}`;
                    return axios(originalRequest);
                });
            }
            
            originalRequest._retry = true;
            isRefreshing = true;
            
            try {
                const { data } = await axios.post('/api/auth/refresh', {}, { 
                    withCredentials: true 
                });
                
                tokenManager.setAccessToken(data.access_token);
                
                // 处理队列中等待的请求
                processQueue(null, data.access_token);
                
                originalRequest.headers['Authorization'] = 
                    `Bearer ${data.access_token}`;
                return axios(originalRequest);
            } catch (refreshError) {
                processQueue(refreshError, null);
                window.location.href = '/login';
                return Promise.reject(refreshError);
            } finally {
                isRefreshing = false;
            }
        }
        
        return Promise.reject(error);
    }
);
```

## 5.2 后端刷新端点

```java
@PostMapping("/refresh")
public ResponseEntity<AuthResponse> refresh(
        @CookieValue(value = "refresh_token", required = false) String refreshToken,
        HttpServletRequest request,
        HttpServletResponse response) {
    
    if (refreshToken == null) {
        throw new AuthenticationException("Refresh token not found");
    }
    
    try {
        AuthResponse tokens = authService.refreshTokens(refreshToken);
        
        // 设置新的 Refresh Token Cookie
        Cookie newCookie = buildRefreshCookie(tokens.getRefreshToken());
        response.addCookie(newCookie);
        
        return ResponseEntity.ok(new AuthResponse(tokens.getAccessToken()));
    } catch (InvalidTokenException e) {
        // 清除 Cookie
        Cookie clearCookie = new Cookie("refresh_token", "");
        clearCookie.setMaxAge(0);
        response.addCookie(clearCookie);
        throw e;
    }
}
```

---

# 6. 登出流程

## 6.1 JWT 登出的挑战

JWT 的无状态特性使登出变得复杂——服务端没有记录哪些 JWT 还"活着"。

## 6.2 登出方案

### 6.2.1 Token 黑名单（最常用）

```java
@Service
public class TokenBlacklistService {
    
    private final RedisTemplate<String, String> redisTemplate;
    
    // 将 JWT 的 jti 加入黑名单，TTL 与 JWT 剩余有效期一致
    public void blacklist(String jti, Date expiration) {
        long ttl = expiration.getTime() - System.currentTimeMillis();
        redisTemplate.opsForValue()
            .set("blacklist:" + jti, "revoked", ttl, TimeUnit.MILLISECONDS);
    }
    
    public boolean isBlacklisted(String jti) {
        return Boolean.TRUE.equals(
            redisTemplate.hasKey("blacklist:" + jti));
    }
}
```

### 6.2.2 版本号递增（Refresh Token 层）

```java
// 用户修改密码/管理员锁定时 → 递增版本号
public void onPasswordChanged(Long userId) {
    tokenVersionService.incrementVersion(userId);
    // 所有旧的 Refresh Token 和 Access Token 立即失效
    // （Access Token 在下一次刷新时才会检查版本号）
}
```

### 6.2.3 客户端清理

```javascript
function logout() {
    // 清除 Access Token
    tokenManager.clearAccessToken();
    
    // 调用服务端登出（清除 Refresh Token Cookie + 黑名单）
    fetch('/api/auth/logout', { method: 'POST', credentials: 'include' });
    
    // 跳转到登录页
    window.location.href = '/login';
}
```

## 6.3 登出实现的完整流程

```java
@PostMapping("/logout")
public ResponseEntity<Void> logout(
        @RequestHeader("Authorization") String authHeader,
        @CookieValue(value = "refresh_token", required = false) String refreshToken,
        HttpServletResponse response) {
    
    // 1. 将 Access Token 加入黑名单
    if (authHeader != null && authHeader.startsWith("Bearer ")) {
        String token = authHeader.substring(7);
        Claims claims = jwtParser.parseClaimsJws(token).getBody();
        blacklistService.blacklist(claims.getId(), claims.getExpiration());
    }
    
    // 2. 吊销 Refresh Token
    if (refreshToken != null) {
        refreshTokenService.revoke(refreshToken);
    }
    
    // 3. 清除 Cookie
    Cookie clearCookie = new Cookie("refresh_token", "");
    clearCookie.setMaxAge(0);
    clearCookie.setHttpOnly(true);
    clearCookie.setSecure(true);
    clearCookie.setPath("/api/auth/refresh");
    response.addCookie(clearCookie);
    
    return ResponseEntity.noContent().build();
}
```

---

# 7. 多设备登录与会话管理

## 7.1 会话管理需求

| 需求 | 描述 |
|------|------|
| 并发登录控制 | 限制同一用户最大登录设备数 |
| 设备感知 | 知晓用户从哪些设备登录 |
| 远程登出 | 用户可远程注销特定设备 |

## 7.2 实现方案

```java
@Entity
public class UserSession {
    @Id
    private String sessionId;
    private Long userId;
    private String deviceInfo;
    private String ipAddress;
    private Instant createdAt;
    private Instant lastAccessAt;
    private boolean active;
}

@Service
public class SessionManager {
    
    private static final int MAX_SESSIONS = 5;
    
    public AuthResponse login(User user, String deviceInfo, String ip) {
        // 检查会话数限制
        long activeSessions = sessionRepository.countActiveByUserId(user.getId());
        if (activeSessions >= MAX_SESSIONS) {
            // 删除最旧的会话
            sessionRepository.deleteOldestSession(user.getId());
        }
        
        // 创建新会话
        String sessionId = UUID.randomUUID().toString();
        UserSession session = new UserSession(sessionId, user.getId(), 
            deviceInfo, ip, Instant.now());
        sessionRepository.save(session);
        
        // 签发 JWT 时携带 session_id
        AuthResponse tokens = issueTokens(user, sessionId);
        return tokens;
    }
    
    public void remoteLogout(Long userId, String sessionId) {
        sessionRepository.deactivate(sessionId);
        tokenVersionService.incrementVersion(userId);
    }
}
```

---

# 8. 跨服务 JWT 传递

## 8.1 微服务间令牌传播

```
用户 ──JWT──> API 网关 ──JWT──> 服务 A ──JWT──> 服务 B
```

### 8.1.1 RestTemplate / WebClient 配置

```java
@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    // 拦截器：自动从当前请求上下文传播 JWT
    @Bean
    public ClientHttpRequestInterceptor jwtPropagationInterceptor() {
        return (request, body, execution) -> {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth != null && auth.getCredentials() instanceof String token) {
                request.getHeaders().setBearerAuth(token);
            }
            return execution.execute(request, body);
        };
    }
}
```

### 8.1.2 Spring Cloud Gateway 自动转发

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          filters:
            - TokenRelay= # 自动转发 Bearer token
          predicates:
            - Path=/api/users/**
```

---

# 9. 认证流程中的安全增强

## 9.1 绑定设备指纹

将 JWT 绑定到特定设备，防止令牌在设备间迁移：

```java
// 签发 JWT 时绑定指纹
public String issueToken(User user, String deviceFingerprint) {
    String fingerprintHash = DigestUtils.sha256Hex(deviceFingerprint);
    
    return Jwts.builder()
        .setSubject(user.getId().toString())
        .claim("fp", fingerprintHash)  // 设备指纹哈希
        .signWith(key)
        .compact();
}

// 验证时比对指纹
public Claims validateToken(String token, String deviceFingerprint) {
    Claims claims = parser.parseClaimsJws(token).getBody();
    String tokenFp = claims.get("fp", String.class);
    String currentFp = DigestUtils.sha256Hex(deviceFingerprint);
    
    if (!tokenFp.equals(currentFp)) {
        throw new SecurityException("Token used from different device");
    }
    
    return claims;
}
```

## 9.2 IP 绑定（谨慎使用）

对于高安全场景（如银行），可绑定 IP 地址：

```java
// 签发
.claim("ip", request.getRemoteAddr())

// 验证（注意：移动端 IP 频繁变更可能导致误判）
String tokenIp = claims.get("ip", String.class);
if (!tokenIp.equals(request.getRemoteAddr())) {
    log.warn("Token IP mismatch: expected {}, actual {}", tokenIp, requestIp);
    // 根据安全策略决定是否拒绝
}
```

## 9.3 短时效 + 活跃窗口续期

对于需要长时间操作的系统（如管理后台），可使用滑动有效期：

```javascript
// 用户活跃时自动续期
let lastActivity = Date.now();
const ACTIVITY_WINDOW = 5 * 60 * 1000; // 5 分钟

document.addEventListener('mousemove', () => {
    const now = Date.now();
    if (now - lastActivity > ACTIVITY_WINDOW) {
        // 自动刷新 Token
        tokenManager.refreshAccessToken();
        lastActivity = now;
    }
});
```
