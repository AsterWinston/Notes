
# 1. 概述

无状态 JWT 认证是一种不依赖服务端 Session 的认证模式。所有用户身份信息编码在 JWT Token 中，服务端仅需验证签名即可确认用户身份。这种模式天然适合微服务架构和水平扩展场景。

```
┌──────────────────────────────────────────────────────────────────┐
│              无状态 JWT 认证 vs 有状态 Session                    │
│                                                                   │
│  有状态 Session:                                                  │
│  ┌──────┐   SessionID   ┌──────────┐     ┌──────────────┐       │
│  │Client│──────────────►│  Service │────►│ Redis/内存   │       │
│  └──────┘               └──────────┘     │ Session 存储 │       │
│                              │           └──────────────┘       │
│                        每次请求查 Session                        │
│                                                                   │
│  无状态 JWT:                                                      │
│  ┌──────┐   Bearer JWT   ┌──────────┐                           │
│  │Client│──────────────►│  Service │    不需要查任何存储！       │
│  └──────┘               └──────────┘                           │
│                        仅验证 JWT 签名                            │
│                        用户信息在 JWT Payload 中                  │
└──────────────────────────────────────────────────────────────────┘
```

# 2. 完整实现方案

## 2.1 系统架构

```
                    ┌─────────────────────────┐
                    │    认证服务 (Auth)       │
                    │  /login → 返回 JWT      │
                    │  /refresh → 刷新 Token   │
                    │  /logout → 加入黑名单    │
                    └───────────┬─────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │Service A │    │Service B │    │Service C │
        │ JWT Filter│   │ JWT Filter│   │ JWT Filter│
        │ 验签即可  │   │ 验签即可  │   │ 验签即可  │
        └──────────┘    └──────────┘    └──────────┘
```

## 2.2 认证服务实现

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private JwtService jwtService;

    @Autowired
    private TokenBlacklistService blacklistService;

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 登录 → 返回双 Token
     */
    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request) {
        // 1. 认证
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(), request.getPassword()));

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();

        // 2. 生成 Token
        String accessToken = jwtService.createAccessToken(userDetails);
        String refreshToken = jwtService.createRefreshToken(userDetails);

        // 3. Refresh Token 存 Redis（用于吊销和刷新）
        redisTemplate.opsForValue().set(
            "refresh_token:" + refreshToken,
            userDetails.getUsername(),
            Duration.ofDays(7));

        // 4. 记录登录事件
        logLoginEvent(userDetails.getUsername(), request);

        return ResponseEntity.ok(new LoginResponse(accessToken, refreshToken));
    }

    /**
     * 刷新 Access Token
     */
    @PostMapping("/refresh")
    public ResponseEntity<LoginResponse> refresh(@RequestBody RefreshRequest request) {
        String refreshToken = request.getRefreshToken();

        // 1. 验证 Refresh Token 格式和签名
        Claims claims = jwtService.parseToken(refreshToken);
        String username = claims.getSubject();

        // 2. 检查 Refresh Token 是否在 Redis 中（未被吊销）
        String storedUser = redisTemplate.opsForValue()
            .get("refresh_token:" + refreshToken);
        if (storedUser == null || !storedUser.equals(username)) {
            return ResponseEntity.status(401)
                .body(new LoginResponse(null, null, "Refresh Token 已失效"));
        }

        // 3. 撤销旧 Refresh Token
        redisTemplate.delete("refresh_token:" + refreshToken);

        // 4. 加载用户信息并生成新 Token
        UserDetails userDetails = userDetailsService.loadUserByUsername(username);

        String newAccessToken = jwtService.createAccessToken(userDetails);
        String newRefreshToken = jwtService.createRefreshToken(userDetails);

        // 5. 存储新 Refresh Token
        redisTemplate.opsForValue().set(
            "refresh_token:" + newRefreshToken,
            username,
            Duration.ofDays(7));

        return ResponseEntity.ok(new LoginResponse(newAccessToken, newRefreshToken));
    }

    /**
     * 登出
     */
    @PostMapping("/logout")
    public ResponseEntity<Void> logout(
            @RequestHeader("Authorization") String authHeader,
            @RequestBody(required = false) LogoutRequest request) {

        // 1. 将 Access Token 加入黑名单（jti + 过期时间）
        String accessToken = authHeader.replace("Bearer ", "");
        Claims claims = jwtService.parseToken(accessToken);
        String jti = claims.getId();
        Date expiration = claims.getExpiration();
        long ttl = expiration.getTime() - System.currentTimeMillis();

        if (ttl > 0) {
            redisTemplate.opsForValue().set(
                "token_blacklist:" + jti,
                "revoked",
                Duration.ofMillis(ttl));
        }

        // 2. 删除 Refresh Token
        if (request != null && request.getRefreshToken() != null) {
            redisTemplate.delete("refresh_token:" + request.getRefreshToken());
        }

        return ResponseEntity.ok().build();
    }
}
```

## 2.3 JWT 服务实现

```java
@Service
public class JwtService {

    @Value("${jwt.access-token-expiration:900000}")  // 15分钟（毫秒）
    private long accessTokenExpiration;

    @Value("${jwt.refresh-token-expiration:604800000}") // 7天（毫秒）
    private long refreshTokenExpiration;

    private final RSAPrivateKey privateKey;
    private final RSAPublicKey publicKey;

    public JwtService(
            @Value("${jwt.private-key}") RSAPrivateKey privateKey,
            @Value("${jwt.public-key}") RSAPublicKey publicKey) {
        this.privateKey = privateKey;
        this.publicKey = publicKey;
    }

    /**
     * 生成 Access Token
     */
    public String createAccessToken(UserDetails userDetails) {
        return buildToken(userDetails, accessTokenExpiration, "access");
    }

    /**
     * 生成 Refresh Token
     */
    public String createRefreshToken(UserDetails userDetails) {
        return buildToken(userDetails, refreshTokenExpiration, "refresh");
    }

    private String buildToken(UserDetails userDetails, long expiration, String tokenType) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);

        return Jwts.builder()
            .issuer("https://auth.example.com")
            .subject(userDetails.getUsername())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).toList())
            .claim("type", tokenType)
            .claim("scope", "read write")
            .issuedAt(now)
            .expiration(expiryDate)
            .id(UUID.randomUUID().toString())  // jti
            .header()
                .keyId("signing-key-2024")      // kid
                .and()
            .signWith(privateKey)
            .compact();
    }

    /**
     * 验证并解析 Token
     */
    public Claims parseToken(String token) {
        return Jwts.parser()
            .verifyWith(publicKey)
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    /**
     * 从 Token 获取用户名
     */
    public String getUsername(String token) {
        return parseToken(token).getSubject();
    }

    /**
     * 验证 Token 是否有效（未过期 + 未被黑名单）
     */
    public boolean isTokenValid(String token, UserDetails userDetails) {
        try {
            Claims claims = parseToken(token);

            String username = claims.getSubject();
            Date expiration = claims.getExpiration();

            return username.equals(userDetails.getUsername())
                && expiration.after(new Date())
                && !isTokenBlacklisted(claims.getId());
        } catch (JwtException e) {
            return false;
        }
    }

    private boolean isTokenBlacklisted(String jti) {
        // 检查 Redis 黑名单
        return redisTemplate.hasKey("token_blacklist:" + jti);
    }
}
```

## 2.4 JWT 认证过滤器

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private JwtService jwtService;

    @Autowired
    private UserDetailsService userDetailsService;

    private static final List<String> EXCLUDED_PATHS = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/api/auth/refresh",
        "/api/public/"
    );

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {

        String path = request.getServletPath();

        // 跳过白名单路径
        if (EXCLUDED_PATHS.stream().anyMatch(path::startsWith)) {
            filterChain.doFilter(request, response);
            return;
        }

        // 从请求头提取 Token
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            sendUnauthorized(response, "缺少认证令牌");
            return;
        }

        String jwt = authHeader.substring(7);
        String username;

        try {
            Claims claims = jwtService.parseToken(jwt);
            username = claims.getSubject();

            // 验证 Token 类型
            String tokenType = claims.get("type", String.class);
            if (!"access".equals(tokenType)) {
                sendUnauthorized(response, "不允许使用 Refresh Token 访问资源");
                return;
            }

        } catch (JwtException e) {
            sendUnauthorized(response, "令牌无效或已过期");
            return;
        }

        // 加载用户并验证
        if (username != null
                && SecurityContextHolder.getContext().getAuthentication() == null) {

            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            if (jwtService.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }

    private void sendUnauthorized(HttpServletResponse response, String message)
            throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.getWriter().write(
            String.format("{\"code\":401,\"message\":\"%s\"}", message));
    }
}
```

# 3. Token 安全存储

## 3.1 前端存储方案对比

| 方案 | XSS 风险 | CSRF 风险 | 跨域 | 推荐度 |
|------|---------|----------|------|--------|
| **localStorage** | 高（JS 可读） | 低 | 不自动发送 | ❌ 不推荐 |
| **sessionStorage** | 高 | 低 | 不自动发送 | ❌ 不推荐 |
| **Cookie (HttpOnly + Secure)** | 低（JS 不可读） | 需防护 | 受同源策略限制 | ⚠️ 可以用 |
| **In-Memory Variable** | 低 | 低 | 不自动发送 | ✅ 推荐（SPA） |
| **BFF 模式** | 最低 | 低 | 灵活 | ✅✅ 最推荐 |

## 3.2 BFF（Backend For Frontend）模式

```
┌──────────────────────────────────────────────────────────────┐
│                      BFF 架构                                 │
│                                                               │
│  ┌──────────┐   Cookie(SSO_SESSION)   ┌──────────┐          │
│  │  Browser │◄──────────────────────►│   BFF    │          │
│  │  (SPA)   │                         │  (层)    │          │
│  └──────────┘                         │          │          │
│                                       │ Session  │          │
│                                       │ 存 Token │          │
│                                       └────┬─────┘          │
│                                            │                 │
│                                Bearer Token│                 │
│                                            ▼                 │
│                              ┌──────────────────────┐       │
│                              │    微服务 A / B / C  │       │
│                              └──────────────────────┘       │
│                                                               │
│  关键点：                                                     │
│  1. Browser 只与 BFF 交互，使用 HttpOnly Cookie               │
│  2. BFF 持有 access_token + refresh_token                    │
│  3. BFF 代表用户调用后端微服务（携带 Bearer Token）            │
│  4. 后端微服务之间也是 Bearer Token 传递                      │
│  5. Browser 永远不接触 Token 原文                             │
└──────────────────────────────────────────────────────────────┘
```

```java
/**
 * BFF 层实现
 */
@RestController
@RequestMapping("/api/bff")
public class BffController {

    @Autowired
    private TokenStore tokenStore;  // Redis 或其他

    /**
     * 登录 → Token 存在 BFF 的 Session 中
     */
    @PostMapping("/login")
    public String login(@RequestBody LoginRequest request,
                        HttpSession session) {
        // 调用认证服务获取 Token
        TokenResponse tokens = authClient.login(request.getUsername(),
            request.getPassword());

        // 存储到 BFF Session（不返回给前端）
        session.setAttribute("ACCESS_TOKEN", tokens.getAccessToken());
        session.setAttribute("REFRESH_TOKEN", tokens.getRefreshToken());

        return "登录成功";
    }

    /**
     * 代理请求到后端服务
     */
    @GetMapping("/users/**")
    public ResponseEntity<?> proxyUserApi(HttpSession session,
            HttpServletRequest request) {
        String accessToken = (String) session.getAttribute("ACCESS_TOKEN");

        // 使用 BFF 持有的 Token 调用后端
        String result = userApiClient.getUserInfo(accessToken);
        return ResponseEntity.ok(result);
    }
}
```

# 4. Token 过期与续期

## 4.1 前端处理策略

```javascript
// 前端 Axios 拦截器：自动处理 Token 过期
import axios from 'axios';

const apiClient = axios.create({
    baseURL: '/api',
    withCredentials: true  // Cookie 模式
});

// 响应拦截器：401 → 自动刷新 Token
apiClient.interceptors.response.use(
    response => response,
    async error => {
        const originalRequest = error.config;

        // 如果是 401 且未重试过
        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true;

            try {
                // 调用刷新接口（BFF 的 Cookie 中有 Refresh Token）
                await axios.post('/api/auth/refresh');
                // 重试原始请求
                return apiClient(originalRequest);
            } catch (refreshError) {
                // 刷新失败 → 跳转登录
                window.location.href = '/login';
                return Promise.reject(refreshError);
            }
        }

        return Promise.reject(error);
    }
);
```

## 4.2 后端正交刷新

```java
/**
 * 后端自动刷新 Token 的 WebClient 过滤器
 * 无需前端感知 Token 过期
 */
@Component
public class TokenRefreshExchangeFilterFunction
        implements ExchangeFilterFunction {

    @Autowired
    private TokenRefreshService tokenRefreshService;

    @Override
    public Mono<ClientResponse> filter(ClientRequest request,
            ExchangeFunction next) {

        return next.exchange(request)
            .flatMap(response -> {
                if (response.statusCode() == HttpStatus.UNAUTHORIZED) {
                    // Token 过期，尝试刷新
                    return tokenRefreshService.refreshToken()
                        .flatMap(newToken -> {
                            // 用新 Token 重试请求
                            ClientRequest retryRequest = ClientRequest.from(request)
                                .headers(h -> h.setBearerAuth(newToken))
                                .build();
                            return next.exchange(retryRequest);
                        })
                        .switchIfEmpty(Mono.just(response)); // 刷新失败→原样返回
                }
                return Mono.just(response);
            });
    }
}
```

# 5. 性能与安全权衡

```
┌──────────────────────────────────────────────────────────────┐
│              无状态 JWT 性能优化                               │
│                                                               │
│  优化点：                                                      │
│                                                               │
│  1. 公钥缓存：不每次请求 JWKS，缓存公钥 10-30 分钟             │
│  2. Token 黑名单 Bloom Filter：减少 Redis 查询               │
│  3. 本地缓存用户信息：Caffeine 缓存 UserDetails 5 分钟        │
│  4. 合理 Token 大小：控制在 2KB 以内（只放必要 claims）       │
│  5. 异步验签：非阻塞 I/O（WebFlux）                           │
│                                                               │
│  安全加固：                                                    │
│                                                               │
│  1. 使用 RS256/ES256 非对称签名                               │
│  2. Token 黑名单存 Redis（jti → TTL）                        │
│  3. 定期密钥轮换（kid 区分）                                  │
│  4. 监控异常 Token 模式（频繁过期、异常 IP）                   │
└──────────────────────────────────────────────────────────────┘
```

# 6. 总结

| 关键点 | 推荐做法 |
|--------|---------|
| **Token 结构** | Access Token（15min）+ Refresh Token（7d） |
| **签名算法** | RS256 / ES256（非对称） |
| **Token 吊销** | Redis 黑名单（jti） |
| **Refresh Token** | 存 Redis，支持立即吊销 |
| **前端存储** | BFF 模式（HttpOnly Cookie） |
| **Token 传递** | Authorization: Bearer xxx |
| **敏感操作** | 重新认证（或要求短期 Token） |
