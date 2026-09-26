
# 1. 概述

SSO 涉及浏览器、认证中心和业务系统三方协作，故障可能发生在任何一个环节。本章系统性梳理常见故障场景、排查方法和解决策略。

# 2. 故障排查方法论

```
SSO 故障排查四步法：

① 定位故障环节
   浏览器端？网络层？认证中心？业务系统？
   → 查看浏览器 Network 面板 / 302 重定向链

② 检查日志和监控
   认证中心日志 / 业务系统日志 / Nginx 日志
   → 查看错误码和异常堆栈

③ 复现与隔离
   能否稳定复现？是否影响所有用户？
   → 缩小排查范围

④ 修复与验证
   修复代码/配置 → 回归测试 → 上线验证
```

# 3. 常见故障场景

## 3.1 无限重定向

```
现象：浏览器地址栏不断在业务系统和认证中心之间跳转

可能原因：
1. Cookie 未正确设置（Domain/Path/SameSite 不匹配）
2. Session 每请求都重新创建
3. CAS ST 验证后未创建局部 Session
4. OAuth2 callback 处理异常导致再次跳转登录

排查步骤：
① 浏览器 F12 → Network → 勾选 "Preserve log"
② 观察 302 重定向链
③ 检查每次跳转的 Cookie Set-Cookie/Request
④ 确认回调用 ticket/code 参数是否存在

解决：
// CAS: 检查 ServiceProperties 配置
serviceProperties.setService("https://myapp.com/login/cas"); // 必须与 CAS 注册的 Service 匹配

// Cookie 配置
cookie.setPath("/");
cookie.setDomain(".example.com");
cookie.setSameSite("Lax"); // SSO 必须用 Lax，不能用 Strict
```

## 3.2 "未收到有效的 CAS Ticket"

```
现象：CAS 验证返回 "ticket not recognized" 或 "INVALID_TICKET"

原因分析：
1. ST 已过期（默认 10 秒）→ 时间不同步
2. ST 已被使用过一次 → 重复提交
3. Service URL 与 ST 签发时不匹配

排查：
① 确认 CAS Server 和 Client 时间差 < 1 分钟
② 检查 ST 是否在 URL 中被多次传递（如 JS 重复请求）
③ 确认 service 参数与 CAS Server 中注册的完全一致

解决：
# CAS Server 增加 ST 超时
st.timeToKillInSeconds=30  # 从 10 秒增加到 30 秒

# 检查时间同步
ntpdate -u ntp.example.com
```

## 3.3 JWT 签名验证失败

```
现象：401 Unauthorized，"Invalid JWT signature"

原因分析：
1. 签名密钥不匹配（签发和验证使用不同密钥）
2. 密钥轮换后未更新验证方的公钥
3. JWT 被篡改
4. Header 中的 alg 和实际签名算法不一致
5. kid 找不到对应的密钥

排查：
① 用 jwt.io 手动解析 Token，检查 Header 和 Payload
② 对比签发方和验证方的密钥配置
③ 检查 JWKS 端点是否返回正确的公钥
④ 检查密钥轮换后的 kid 匹配

解决：
// 签发方和验证方使用同一对非对称密钥
// 签发方持有私钥
Jwts.builder().signWith(privateKey).compact();

// 验证方使用对应公钥
Jwts.parser().verifyWith(publicKey).build().parseSignedClaims(token);
```

## 3.4 Cookie 未正常传递（跨域问题）

```
现象：用户已在认证中心登录，但业务系统仍重定向到登录页

原因分析：
1. Cookie Domain 不匹配（跨域）
2. SameSite=Strict 导致 302 重定向时不携带 Cookie
3. Cookie Secure=true 但使用 HTTP
4. 浏览器禁用第三方 Cookie

排查：
① Chrome F12 → Application → Cookies → 检查 Cookie 属性
② 观察 Set-Cookie 响应头
③ 检查 SameSite 属性值

解决：
// SameSite=Lax（而非 Strict）
cookie.setAttribute("SameSite", "Lax");

// 跨域场景使用 OAuth2/OIDC 标准流程而非 Cookie 共享

// 检查 HTTPS 配置
```

## 3.5 OAuth2 redirect_uri 不匹配

```
现象：授权服务器返回 "invalid redirect_uri" 或 "redirect_uri mismatch"

原因分析：
1. 请求中的 redirect_uri 与注册的不完全匹配
2. 协议不匹配（http vs https）
3. 端口号不匹配
4. 尾部斜杠不匹配
5. 编码问题

排查：
① 精确对比请求 URL 和注册的 redirect_uri
② 检查 URL 编码后的结果
③ 确认是否启用通配符 redirect_uri（不推荐）

解决：
// ✅ 正确：精确匹配
注册: https://myapp.com/login/oauth2/code/google
请求: https://myapp.com/login/oauth2/code/google

// ❌ 错误：尾部斜杠不匹配
注册: https://myapp.com/login/oauth2/code/google
请求: https://myapp.com/login/oauth2/code/google/

// ❌ 错误：协议不匹配
注册: https://myapp.com/callback
请求: http://myapp.com/callback
```

## 3.6 认证后返回 403

```
现象：认证成功，但访问资源时返回 403

可能原因：
1. 用户权限不足
2. Token 中的权限与请求的权限不匹配
3. 权限提取逻辑有误（roles 字段名不匹配）
4. CSRF Token 缺失或无效

排查：
① 检查 Authentication.getAuthorities() 是否正确
② 检查 SecurityFilterChain 的授权规则
③ 检查 Token 中的 roles 字段名

解决：
// 自定义权限提取器
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
    converter.setAuthoritiesClaimName("roles");  // 确认字段名
    converter.setAuthorityPrefix("ROLE_");       // 确认前缀

    JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
    jwtConverter.setJwtGrantedAuthoritiesConverter(converter);
    return jwtConverter;
}
```

## 3.7 Session 丢失（集群切换）

```
现象：用户登录后刷新页面或访问其他节点，变为未登录

原因分析：
1. Session 未共享（使用本地 Session）
2. Redis 连接失败
3. Session 序列化失败
4. Sticky Session 没有正确配置

排查：
① 查看 Redis 是否存储了 Session 数据
② 检查 Spring Session 依赖和配置
③ 检查 Nginx 是否有 ip_hash 配置

解决：
# 使用 Spring Session + Redis
spring.session.store-type=redis
spring.session.redis.namespace=myapp:session

# Nginx 如果未使用 Redis 共享 Session，加 ip_hash
upstream backend {
    ip_hash;
    server app1:8080;
    server app2:8080;
}
```

## 3.8 SLO 不生效

```
现象：用户在一个系统登出后，访问其他系统仍然显示已登录

原因分析：
1. SLO 通知未发送到所有系统
2. SLO 回调 URL 不可达
3. 浏览器可能使用缓存的页面
4. iframe 静默刷新导致 Token 自动续期

排查：
① 检查 CAS/OIDC 的 SLO 日志
② 确认各系统的 SLO 端点可达
③ 检查浏览器是否缓存了已登录的页面状态

解决：
// CAS: 确保 Service Registered 中配置了 logoutUrl
{
    "logoutType": "BACK_CHANNEL",
    "logoutUrl": "https://myapp.com/logout/cas"
}

// OIDC: 确保启用了 RP-Initiated Logout
// 或实现 Back-Channel Logout 端点

// 通用：实现 MQ 广播登出
```

## 3.9 Token 频繁过期

```
现象：用户频繁被踢出，需要重新登录

原因分析：
1. Token TTL 设置太短
2. Refresh Token 未被正确使用
3. 时钟偏差导致 Token 提前失效
4. Token 黑名单误杀

排查：
① 检查 Token 签发时间和过期时间
② 确认客户端是否正确刷新 Token
③ 检查服务器时间同步

解决：
// 增加 Access Token TTL
String token = Jwts.builder()
    .expiration(new Date(System.currentTimeMillis() + 900_000)) // 15 分钟
    .compact();

// 正确实现 Refresh Token 刷新逻辑
// Token 允许的时钟偏差
NimbusJwtDecoder decoder = NimbusJwtDecoder
    .withJwkSetUri(jwkSetUri)
    .build();
// NimbusJwtDecoder 默认允许 60 秒时钟偏差
```

## 3.10 高并发下性能下降

```
现象：用户量大时登录变慢或超时

原因分析：
1. Token 验证每次查数据库
2. Redis 连接池不足
3. JWT 验签没有缓存公钥
4. 数据库连接池不够
5. 线程池配置过小

排查：
① 查看各组件 CPU/内存使用率
② 检查数据库慢查询
③ 检查 Redis 延迟和连接数
④ 查看 JVM GC 日志

解决：
- 使用多级缓存（Caffeine + Redis）
- 增加 Redis 连接池大小
- 缓存 JWKS 公钥 10 分钟
- 使用读写分离
- 优化数据库索引
- 异步化非关键操作
```

# 4. 调试工具与技巧

## 4.1 浏览器调试

```
Chrome DevTools:
1. F12 → Application → Cookies → 查看 Cookie 属性
2. Network → 勾选 "Preserve log" → 跟踪 302 跳转链
3. 查看 Authorization 请求头中的 Token 格式
4. Console → document.cookie（仅非 HttpOnly Cookie）

curl 命令：
# 跟踪重定向
curl -v -L https://myapp.com/protected

# 查看响应头
curl -I https://sso.example.com/login

# 携带 Cookie
curl -v --cookie "TGC=xxx" https://myapp.com/protected
```

## 4.2 JWT 调试

```bash
# 解码 JWT（不验证签名）
echo "eyJhbGciOi..." | cut -d'.' -f2 | base64 -d | jq .

# 使用 jwt-cli 工具
jwt decode eyJhbGciOi...
```

## 4.3 SAML 调试

```bash
# SAML Request/Response 解码
# SAML 消息通常经过 Deflate + Base64 编码
echo "SAMLResponse值" | base64 -d | python3 -c "
import sys,zlib
data = sys.stdin.buffer.read()
try:
    # 先尝试 raw inflate（无 header）
    print(zlib.decompress(data, -15).decode())
except:
    print(data.decode())
"
```

# 5. 总结

| 常见问题 | 首要检查 |
|---------|---------|
| 无限重定向 | Cookie SameSite / Domain 配置 |
| Ticket/Token 无效 | 时间同步 / 密钥一致性 |
| Cookie 不传递 | SameSite / Secure / Domain 匹配 |
| 403 权限不足 | Token 权限字段 / 授权规则 |
| Session 丢失 | Redis 连接 / Sticky Session |
| SLO 不生效 | SLO 端点可达性 / 配置正确性 |
| Token 频繁过期 | TTL 配置 / Refresh Token 刷新逻辑 |
| 性能问题 | 缓存 / 连接池 / 索引 |
