认证解决"你是谁"，授权解决"你能做什么"。本篇用 JWT 在 Gin 中实现一套最小可用的登录与鉴权流程，并解释 Token 过期、刷新、撤销和密钥管理等生产问题。

# 1. 认证与授权基础

## 1.1 会话 vs Token

| 维度 | Session-Cookie | Token（JWT） |
|------|----------------|--------------|
| 状态存储 | 服务端保存会话 | 客户端保存 Token |
| 扩展方式 | 共享 Session 或粘性会话 | 天然适合多服务 |
| 失效控制 | 服务端删除即可 | 无状态时较难立即撤销 |
| 跨域 | 受 Cookie 策略限制 | 放在 Header 中较灵活 |
| 适合场景 | 传统 Web、同域应用 | API、移动端、微服务 |

★ **关键认知**：JWT 不是"更安全的 Session"，而是另一种取舍。它把状态放到客户端，换来扩展性，同时带来撤销困难、Token 泄露等新问题。

## 1.2 JWT 结构

JWT 由三部分组成，用 `.` 连接：

```text
header.payload.signature

┌──────────────┐   ┌──────────────────┐   ┌──────────────┐
│ 算法与类型   │ . │ 业务声明 claims  │ . │ 签名         │
└──────────────┘   └──────────────────┘   └──────────────┘
```

示例 payload：

```json
{
  "sub": "42",
  "role": "user",
  "iss": "gin-demo",
  "exp": 1735689600,
  "iat": 1735686000,
  "jti": "8f14e45f"
}
```

签发与校验流程：

```text
登录成功
  │
  ▼
服务端用密钥签名生成 Token
  │
  ▼
客户端保存 Token，后续放在 Authorization: Bearer <token>
  │
  ▼
服务端验证签名、过期时间和签发者
  │
  ▼
通过则取出 claims，进入业务处理
```

⚠ JWT 的 payload 只是 Base64URL 编码，不是加密。任何人都能解码看到内容，因此不要把密码、身份证号等敏感信息放进去。

# 2. Gin 中实现 JWT

## 2.1 登录签发

使用 `github.com/golang-jwt/jwt/v5`：

```go
type Claims struct {
    UserID int64  `json:"uid"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func SignToken(userID int64, role string) (string, error) {
    now := time.Now()
    claims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            Issuer:    "gin-demo",
            Subject:   strconv.FormatInt(userID, 10),
            IssuedAt:  jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(2 * time.Hour)),
            ID:        uuid.NewString(), // jti，可用于撤销和审计
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(os.Getenv("JWT_SECRET")))
}
```

登录 Handler：

```go
type LoginRequest struct {
    Username string `json:"username" binding:"required"`
    Password string `json:"password" binding:"required"`
}

func Login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"code": "INVALID_ARGUMENT"})
        return
    }

    user, err := userService.FindByUsername(req.Username)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"code": "INVALID_CREDENTIALS"})
        return
    }

    if err := bcrypt.CompareHashAndPassword(
        []byte(user.PasswordHash), []byte(req.Password),
    ); err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"code": "INVALID_CREDENTIALS"})
        return
    }

    token, err := SignToken(user.ID, user.Role)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"code": "TOKEN_SIGN_FAILED"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"token": token})
}
```

⚠ 无论是用户不存在还是密码错误，都应返回同一条错误信息，避免攻击者通过响应差异枚举用户名。密码必须存 bcrypt/argon2 哈希，不能存明文或简单 MD5。

## 2.2 中间件校验

```go
func Auth() gin.HandlerFunc {
    return func(c *gin.Context) {
        header := c.GetHeader("Authorization")
        const prefix = "Bearer "
        if !strings.HasPrefix(header, prefix) {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "code":    "UNAUTHORIZED",
                "message": "请先登录",
            })
            return
        }

        tokenString := strings.TrimPrefix(header, prefix)
        claims := &Claims{}

        token, err := jwt.ParseWithClaims(
            tokenString,
            claims,
            func(token *jwt.Token) (any, error) {
                // 必须显式检查签名算法，防止 alg 混淆攻击
                if token.Method != jwt.SigningMethodHS256 {
                    return nil, fmt.Errorf("unexpected signing method: %v", token.Method)
                }
                return []byte(os.Getenv("JWT_SECRET")), nil
            },
            jwt.WithIssuer("gin-demo"),
            jwt.WithExpirationRequired(),
        )
        if err != nil || !token.Valid {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "code":    "INVALID_TOKEN",
                "message": "登录已失效",
            })
            return
        }

        c.Set("userID", claims.UserID)
        c.Set("role", claims.Role)
        c.Next()
    }
}
```

## 2.3 刷新与登出

无状态 JWT 一旦签发，在过期前通常一直有效。常见方案：

| 方案 | 做法 | 代价 |
|------|------|------|
| 短 Access Token + Refresh Token | Access 15 分钟，Refresh 7 天 | 需要额外存储和刷新接口 |
| 黑名单 | 登出时把 `jti` 加入 Redis，校验时查询 | 每次请求多一次查询 |
| Token 版本号 | 用户表存 tokenVersion，改密码时递增 | 需要查用户状态 |
| 短过期 + 重新登录 | 最简单 | 用户体验较差 |

刷新接口通常这样设计：

```text
POST /auth/refresh
Authorization: Bearer <refresh-token>
  │
  ▼
校验 Refresh Token 类型、过期时间和是否被撤销
  │
  ▼
签发新的 Access Token
```

★ **关键认知**：JWT 的"无状态"是指服务端不保存会话，不代表不能有撤销机制。只要业务需要立即踢下线，就必须引入状态存储。

# 3. 权限控制

## 3.1 角色与权限模型

RBAC（基于角色的访问控制）是最常见的入门模型：

```text
用户 ── 拥有 ──> 角色 ── 拥有 ──> 权限
  │                │                │
 alice           admin         user:read
                 user          user:write
```

简化数据结构：

```go
type User struct {
    ID       int64
    Username string
    Role     string // admin、user
}
```

更复杂的系统会把角色和权限拆成多表，支持一个用户多个角色。

## 3.2 路由级权限

```go
func RequireRole(roles ...string) gin.HandlerFunc {
    allowed := make(map[string]bool, len(roles))
    for _, role := range roles {
        allowed[role] = true
    }

    return func(c *gin.Context) {
        role := c.GetString("role")
        if !allowed[role] {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "code":    "FORBIDDEN",
                "message": "没有访问权限",
            })
            return
        }
        c.Next()
    }
}
```

注册：

```go
api := r.Group("/api/v1")
api.Use(Auth())
{
    api.GET("/profile", getProfile) // 登录即可

    admin := api.Group("/admin")
    admin.Use(RequireRole("admin"))
    {
        admin.GET("/users", listAllUsers)
    }
}
```

# 4. 安全实践

## 4.1 密钥管理

- 密钥至少 32 字节，使用随机值。
- 从环境变量或密钥管理服务读取，禁止硬编码进仓库。
- 不同环境使用不同密钥。
- 支持轮换：新密钥签发，旧密钥在过渡期内仍可校验。
- 不要把密钥写进日志、错误响应或前端代码。

```bash
# 生成一个随机密钥示例
openssl rand -base64 32
```

## 4.2 过期、撤销与重放

- Access Token 过期时间应短，降低泄露后的影响。
- 使用 `jti` 标识每次签发，便于黑名单和审计。
- 对高风险操作增加二次验证，而不是只依赖 Token。
- HTTPS 是底线，否则 Token 在网络中可被截获。
- 客户端存储位置要权衡：`localStorage` 易受 XSS，`HttpOnly Cookie` 易受 CSRF，需配合 SameSite 和 CSRF Token。

# 5. 常见错误

| 错误 | 风险 | 正确做法 |
|------|------|----------|
| 不校验 `alg` | 算法混淆攻击 | 显式限定 HS256 或 RS256 |
| 密钥硬编码 | 泄露后可伪造任意 Token | 环境变量 + 密钥管理 |
| 把敏感信息放 payload | 可被 Base64 解码 | payload 只放必要声明 |
| Token 永久有效 | 泄露后长期可用 | 短过期 + 刷新机制 |
| 401 和 403 混用 | 客户端无法区分登录和权限问题 | 未认证 401，无权限 403 |
| 只在网关校验 | 内网调用绕过鉴权 | 每个服务都要校验或使用可信内网机制 |

# 6. 动手练习

1. 实现 `/auth/login`，校验用户名密码后签发 Access Token。
2. 实现 `/api/v1/profile`，要求登录后返回当前用户信息。
3. 实现 `/api/v1/admin/users`，只允许 `admin` 角色访问。
4. 写测试覆盖：无 Token、Token 格式错误、Token 过期、角色不足。
5. 增加 `jti` 黑名单，验证登出后 Token 立即失效。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 认证 vs 授权 | 认证确认身份，授权确认权限 |
| JWT 结构 | header + payload + signature，payload 不是加密 |
| 签发校验 | 显式校验算法、签名、过期时间和签发者 |
| 刷新撤销 | 短 Access Token + Refresh Token，必要时加黑名单 |
| 权限 | 角色中间件控制路由分组 |
| 安全底线 | HTTPS、强密钥、短过期、敏感信息不进 Token |

下一篇进入测试，讲如何用 `testing` 和 `httptest` 测试 Gin 的 Handler、路由、中间件和依赖替换。
