# 1. 技术选型建议

## 1.1 后端存储选择

```
决策树：
需要存储 Session？
  ├── 是 → 选什么存储？
  │    ├── 已有 Redis → Redis（最佳选择）
  │    ├── 已有 MySQL/PostgreSQL → JDBC（中小规模）
  │    ├── 不想引入外部存储 → Hazelcast（嵌入式）
  │    └── 已有 MongoDB → MongoDB
  └── 否 → 使用 JWT（完全无状态）
```

| 场景 | 推荐 | 理由 |
|------|------|------|
| 大多数 Web 应用 | **Redis** | 高性能、成熟、生态好 |
| 中小企业应用 | **JDBC** | 复用已有数据库，降低运维成本 |
| 不想维护外部存储 | **Hazelcast** | 嵌入式，零运维 |
| 高并发微服务 | **Redis 集群** | 水平扩展，万级 QPS |
| 纯 REST API | **JWT + 黑名单** | 无需服务端 Session |
| 混合需求 | **Redis 主 + JDBC 备** | 高可用降级方案 |

## 1.2 序列化选择

```
推荐优先级：
1. Jackson JSON（推荐） — 可读、可调试、足够快
2. Kryo（优化）         — 极高性能场景
3. JDK 序列化（避免）    — 仅用于快速原型或测试
```

---

# 2. 配置最佳实践

## 2.1 环境隔离

```yaml
# application.yml (通用配置)
spring:
  session:
    timeout: 30m
    redis:
      namespace: app:session
      flush-mode: on_save
      save-mode: on_set_attribute

---
# application-dev.yml (开发环境)
spring:
  data:
    redis:
      host: localhost
      port: 6379
server:
  servlet:
    session:
      cookie:
        secure: false        # 开发环境关闭 HTTPS
        http-only: false     # 开发环境允许 JS 访问（调试）
        same-site: Lax

---
# application-prod.yml (生产环境)
spring:
  data:
    redis:
      cluster:
        nodes:
          - redis-cluster-1.prod:6379
          - redis-cluster-2.prod:6379
          - redis-cluster-3.prod:6379
      password: ${REDIS_PASSWORD}
  session:
    timeout: 15m              # 安全要求高，较短超时
    redis:
      namespace: prod:session

server:
  servlet:
    session:
      cookie:
        secure: true          # 强制 HTTPS
        http-only: true       # 禁止 JS 访问
        same-site: Lax
```

## 2.2 配置优先级

```yaml
# 推荐：使用 spring.session.* 统一管理
spring:
  session:
    timeout: 30m           # 设置 Session 超时（自动同步到 Server 端）

# 避免：
server.servlet.session.timeout: 30m  # 不要与上面同时设置不同值
```

## 2.3 使用配置中心

```java
// 不要在代码中硬编码配置
@Configuration
public class SessionConfig {

    @Bean
    public RedisSessionRepository sessionRepository(
            RedisOperations<String, Object> redis,
            @Value("${spring.session.timeout:30m}") Duration timeout,
            @Value("${spring.session.redis.namespace:spring:session}") String namespace) {

        RedisSessionRepository repository = new RedisSessionRepository(redis);
        repository.setDefaultMaxInactiveInterval(timeout);
        repository.setRedisNamespace(namespace);
        return repository;
    }
}
```

---

# 3. 编码最佳实践

## 3.1 Session 属性管理

```java
// ✓ 正确：定义常量管理属性 Key
public class SessionKeys {
    public static final String USER_ID = "app:user:id";
    public static final String USER_NAME = "app:user:name";
    public static final String USER_ROLES = "app:user:roles";
    public static final String LOGIN_TIME = "app:login:time";
    public static final String LOGIN_IP = "app:login:ip";
    public static final String LOCALE = "app:locale";
    public static final String CART_ID = "app:cart:id";
}

// ✓ 使用
session.setAttribute(SessionKeys.USER_ID, userId);
session.setAttribute(SessionKeys.USER_NAME, username);

// ✗ 错误：使用魔法字符串
session.setAttribute("userId", userId);
session.setAttribute("user_name", username);
```

## 3.2 避免存储大对象

```java
// ✓ 正确：存标识，用时查询
@GetMapping("/profile")
public UserProfile getProfile(HttpSession session) {
    Long userId = (Long) session.getAttribute(SessionKeys.USER_ID);
    return userService.getProfile(userId); // 从缓存或 DB 查询
}

// ✗ 错误：存储完整对象
@GetMapping("/home")
public String home(HttpSession session) {
    UserProfile profile = userService.getFullProfile(userId);
    session.setAttribute("profile", profile); // 可能很大
    return "home";
}
```

## 3.3 线程安全

```java
// ✓ 正确：使用不可变对象或复制后修改
ImmutableUser user = (ImmutableUser) session.getAttribute("user");
ImmutableUser updated = user.withName("newName");
session.setAttribute("user", updated);

// ✗ 错误：直接修改可变对象
User user = (User) session.getAttribute("user");
user.setName("newName");  // 可能不会被持久化（取决于 SaveMode）
```

## 3.4 Session 数据校验

```java
@Component
public class SessionValidator {

    public boolean isValidSession(HttpSession session) {
        if (session == null) return false;

        // 检查必要属性是否存在
        if (session.getAttribute(SessionKeys.USER_ID) == null) return false;

        // 检查 Session 是否即将过期（提前续期）
        if (session.getMaxInactiveInterval() > 0) {
            long remaining = session.getLastAccessedTime()
                + (session.getMaxInactiveInterval() * 1000L)
                - System.currentTimeMillis();

            if (remaining < 300_000) { // 剩余不足 5 分钟
                session.setMaxInactiveInterval(1800); // 续期
            }
        }

        return true;
    }
}
```

---

# 4. 架构最佳实践

## 4.1 Session 与微服务

```
┌──────────────────────────────────────────────┐
│                API Gateway                    │
│  (统一入口，处理 Session 验证和路由)           │
└──────────┬───────────────────────────────────┘
           │
    ┌──────┴──────┐
    │  认证服务     │ ← 管理登录、Session 创建
    │  (Auth)     │
    └──────┬──────┘
           │
    ┌──────┴──────┐
    │  Redis 集群  │ ← 所有服务共享 Session
    └──────┬──────┘
           │
    ┌──────┼──────┬──────────┐
    ▼      ▼      ▼          ▼
  订单   商品   用户         通知
  服务   服务   服务         服务
```

**设计原则**：

| 原则 | 说明 |
|------|------|
| **认证集中化** | 所有登录/注销由认证服务统一处理 |
| **Session 共享** | 通过 Redis 让所有服务读取 Session |
| **最小权限** | 各服务只读自己需要的 Session 属性 |
| **服务隔离** | 使用不同 namespace 隔离不同服务的 Session |

## 4.2 Session 配置统一管理

```java
// 提取统一的 Session 配置为 Starter 或共享模块
@Configuration
public class SessionAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setCookieName("AUTH_TOKEN");
        serializer.setCookiePath("/");
        serializer.setUseHttpOnlyCookie(true);
        serializer.setUseSecureCookie(!isDevProfile());
        serializer.setSameSite("Lax");
        return serializer;
    }

    @Bean
    @ConditionalOnMissingBean
    public HttpSessionIdResolver httpSessionIdResolver(
            CookieSerializer cookieSerializer) {
        CookieHttpSessionIdResolver resolver =
            new CookieHttpSessionIdResolver();
        resolver.setCookieSerializer(cookieSerializer);
        return resolver;
    }
}
```

---

# 5. 安全最佳实践

## 5.1 必须遵守的规则

| 规则 | 说明 |
|------|------|
| **HTTPS 强制** | 生产环境必须使用 HTTPS |
| **HttpOnly Cookie** | 必须设置 `http-only: true` |
| **Secure Cookie** | 必须设置 `secure: true` |
| **SameSite=Lax** | 推荐设置 `same-site: Lax` |
| **禁止存储密码** | Session 中绝不存储明文密码 |
| **Session Fixation** | 保持默认防护 |
| **限制并发会话** | 设置 `maximumSessions` |
| **安全退出** | 注销时必须 `invalidateHttpSession(true)` |

## 5.2 安全配置模板

```java
@Configuration
@EnableWebSecurity
public class SecureSessionSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http)
            throws Exception {
        http
            // 强制 HTTPS
            .requiresChannel(channel -> channel.anyRequest().requiresSecure())

            // 安全响应头
            .headers(headers -> headers
                .contentSecurityPolicy(csp ->
                    csp.policyDirectives("default-src 'self'"))
                .httpStrictTransportSecurity(hsts ->
                    hsts.maxAgeInSeconds(31536000).includeSubDomains(true))
            )

            // Session 管理
            .sessionManagement(session -> session
                .sessionFixation().migrateSession()
                .maximumSessions(2)
                .maxSessionsPreventsLogin(false)
            )

            // 安全退出
            .logout(logout -> logout
                .logoutUrl("/logout")
                .invalidateHttpSession(true)
                .deleteCookies("SESSION", "JSESSIONID", "remember-me")
                .logoutSuccessUrl("/login?logout")
            )

            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());

        return http.build();
    }
}
```

---

# 6. 运维最佳实践

## 6.1 Redis 运维建议

```bash
# 1. 独立的 Redis 实例（与业务缓存分离）
# 避免 Session 操作影响业务缓存性能

# 2. 设置合理的内存上限
maxmemory 2gb
maxmemory-policy volatile-lru

# 3. 监控 Key 数量
# 告警规则：Session Key 数量 > 正常值的 1.5 倍

# 4. 定期检查过期 Key
redis-cli --scan --pattern "spring:session:expirations:*" | wc -l
```

## 6.2 发布与回滚策略

```
发布前检查清单：
□ Session 序列化对象是否兼容新旧版本
□ Cookie 配置是否一致
□ 是否需要提前清理旧 Session
□ 是否有降级方案（Redis 不可用时的处理）

回滚考虑：
□ 回滚后旧版本能否正确反序列化新版本创建的 Session
□ 如果不能，是否接受用户重新登录
```

## 6.3 容量规划

| 指标 | 估算方法 |
|------|----------|
| **活跃 Session 数** | 日活用户数 × 30%（假设 30% 同时在线） |
| **每个 Session 大小** | 通常 2-5KB（需实测） |
| **Redis 内存** | 活跃 Session 数 × 每个 Session 大小 × 3（冗余） |
| **Redis 连接数** | 实例数 × 连接池大小 |
| **QPS** | 总请求数 × Session 操作比例 ≈ 30% |

---

# 7. 不可做的事

| 禁止 | 原因 | 替代方案 |
|------|------|----------|
| 在 Session 中存储大对象 (> 10KB) | 序列化开销大，Redis 内存浪费 | 存 ID，用时查询 |
| 在 Session 中存储敏感数据 | Session 可能泄露 | Token 加密或存服务端 |
| 使用 JDK 默认序列化 | 安全漏洞，不可读 | 使用 Jackson JSON |
| 在 Session 中存储临时计算结果 | 无法保证一致性 | 使用缓存或内存计算 |
| 将 Session 当缓存使用 | Session 语义是会话状态 | 使用 Redis 直接做缓存 |
| 禁用 HttpOnly 或 Secure | 安全风险 | 始终开启 |
| 不设置 Session 超时 | Session 永久堆积 | 设置合理超时 |
| 硬编码 Session 配置 | 环境间不一致 | 使用配置文件 |
