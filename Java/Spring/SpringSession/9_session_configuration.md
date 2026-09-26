# 1. Session 超时配置

## 1.1 配置方式

**方式一：application.yml**

```yaml
spring:
  session:
    timeout: 30m                                    # 推荐：使用 Duration 格式
    # timeout: 1800s                                # 也支持秒
    # timeout: 1800                                 # 不推荐：默认当作秒
```

**方式二：@EnableRedisHttpSession / @EnableJdbcHttpSession 注解**

```java
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class SessionConfig {
}
```

**方式三：Server 端配置**

```yaml
server:
  servlet:
    session:
      timeout: 30m
```

**优先级**：`spring.session.timeout` > `server.servlet.session.timeout` > 注解中的 `maxInactiveIntervalInSeconds` > 代码中 `setMaxInactiveInterval()`

## 1.2 超时时间选择建议

| 应用类型 | 推荐超时 | 原因 |
|----------|----------|------|
| **金融/银行** | 5-15 分钟 | 安全要求高 |
| **政务系统** | 15-30 分钟 | 安全 + 体验平衡 |
| **企业管理系统** | 30-60 分钟 | 日常工作需要 |
| **社交/论坛** | 60-120 分钟 | 降低重新登录频率 |
| **电商** | 30 分钟 | 结账过程不能过长 |
| **内容浏览** | 60-120 分钟（记住我） | 阅读体验 |

## 1.3 动态修改 Session 超时

```java
@RestController
public class SessionController {

    @GetMapping("/session/extend")
    public String extendSession(HttpSession session) {
        // 延长 Session 超时时间
        session.setMaxInactiveInterval(3600); // 60 分钟
        return "Session 超时已延长至 60 分钟, ID: " + session.getId();
    }
}
```

---

# 2. Cookie 配置

## 2.1 完整 Cookie 配置

```yaml
server:
  servlet:
    session:
      cookie:
        name: MYSESSION              # Cookie 名称
        domain: .example.com         # 域名（支持子域名共享）
        path: /                      # 路径
        http-only: true             # 禁止 JS 访问（防 XSS）
        secure: true                # 仅 HTTPS（生产环境必须）
        same-site: Lax              # SameSite 策略
        max-age: 7d                 # Cookie 有效期
```

## 2.2 Cookie 属性详解

| 属性 | 说明 | 推荐值 |
|------|------|--------|
| **name** | Cookie 名称 | 避免使用默认 `SESSION`，可自定义防冲突 |
| **domain** | 作用域名 | 如 `.example.com` 支持子域共享 |
| **path** | 作用路径 | 默认 `/`，一般不需要改 |
| **httpOnly** | 禁止 JavaScript 读取 | 生产必须 `true`（防 XSS） |
| **secure** | 仅 HTTPS 传输 | 生产必须 `true` |
| **sameSite** | 跨站请求限制 | `Lax` 安全与兼容平衡，`Strict` 最安全 |
| **maxAge** | 浏览器端保存时间 | 使用 Duration，如 `7d`、`30m` |

## 2.3 SameSite 策略

| 值 | 效果 |
|----|------|
| **Strict** | 完全禁止跨站携带 Cookie（最安全，但可能影响体验） |
| **Lax** | 允许顶级导航（如从 Email 点链接进入）携带 Cookie（推荐） |
| **None** | 允许跨站携带（必须配合 `Secure`，用于第三方 Cookie） |

## 2.4 编程式 Cookie 配置

```java
@Configuration
public class SessionConfig {

    @Bean
    public HttpSessionIdResolver httpSessionIdResolver() {
        CookieHttpSessionIdResolver resolver =
            new CookieHttpSessionIdResolver();

        DefaultCookieSerializer cookieSerializer =
            new DefaultCookieSerializer();
        cookieSerializer.setCookieName("MYSESSION");
        cookieSerializer.setUseHttpOnlyCookie(true);     // HttpOnly
        cookieSerializer.setUseSecureCookie(true);       // Secure
        cookieSerializer.setSameSite("Lax");              // SameSite
        cookieSerializer.setDomainNamePattern("^.+?\\.(\\w+\\.[a-z]+)$");
        cookieSerializer.setCookieMaxAge(3600);           // 秒

        resolver.setCookieSerializer(cookieSerializer);
        return resolver;
    }
}
```

---

# 3. Redis 专属配置

## 3.1 完整配置项

```yaml
spring:
  session:
    redis:
      namespace: app:session         # Key 命名空间
      flush-mode: on_save            # on_save | immediate
      save-mode: on_set_attribute    # on_set_attribute | on_get_attribute | always
      cleanup-cron: 0 * * * * *     # 定时清理过期 Session 的 Cron 表达式
      configure-action: notify_keyspace_events      # 配置 Redis 键空间通知
      repository-type: default       # default | indexed | reactive
```

## 3.2 FlushMode 详解

| 模式 | 行为 | 优缺点 |
|------|------|--------|
| **ON_SAVE** | 仅在 `sessionRepository.save()` 调用时持久化 | 性能优，请求结束统一保存 |
| **IMMEDIATE** | 每次 `setAttribute()` / `removeAttribute()` 立即持久化 | 一致性强，性能开销大 |

```java
// ON_SAVE 模式下的保存时机
// 1. SessionRepositoryFilter.onResponseCommitted() 自动调用 save()
// 2. 显式调用 sessionRepository.save(session)
```

## 3.3 SaveMode 详解

| 模式 | 行为 | 性能影响 |
|------|------|----------|
| **ON_SET_ATTRIBUTE** | 仅在 `setAttribute()` 时标记脏数据 | 最小开销（默认） |
| **ON_GET_ATTRIBUTE** | `getAttribute()` 时也跟踪（适用于获取后修改可变对象） | 有一定开销 |
| **ALWAYS** | 每次保存都全量写入所有属性 | 开销最大，但最简单 |

```java
// SaveMode 使用场景对比
// ON_SET_ATTRIBUTE 够用：
session.setAttribute("user", new User("zhangsan"));

// ON_GET_ATTRIBUTE 需要：
User user = session.getAttribute("user");
user.setName("lisi"); // 修改现有对象的属性
// 如果不使用 ON_GET_ATTRIBUTE，此修改不会被持久化！
```

---

# 4. JDBC 专属配置

```yaml
spring:
  session:
    jdbc:
      initialize-schema: always
      table-name: SPRING_SESSION
      schema: classpath:org/springframework/session/jdbc/schema-mysql.sql
      cleanup-cron: 0 * * * * *
```

---

# 5. 多后端共存配置

有时需要在不同环境使用不同后端，或在同一应用中为不同路径使用不同后端：

## 5.1 按 Profile 切换

```yaml
# application-dev.yml（开发环境使用 Redis）
spring:
  session:
    store-type: redis
  data:
    redis:
      host: localhost

---
# application-test.yml（测试环境使用 JDBC/H2）
spring:
  session:
    store-type: jdbc
  datasource:
    url: jdbc:h2:mem:testdb

---
# application-prod.yml（生产环境使用 Redis 集群）
spring:
  session:
    store-type: redis
    redis:
      namespace: prod:session
  data:
    redis:
      cluster:
        nodes: redis1:6379,redis2:6379,redis3:6379
```

## 5.2 多 SessionRepository 共存

```java
@Configuration
public class MultiSessionConfig {

    @Bean
    @Primary
    public RedisSessionRepository primarySessionRepository(
            RedisOperations<String, Object> redis) {
        return new RedisSessionRepository(redis);
    }

    @Bean
    public JdbcSessionRepository secondarySessionRepository(
            JdbcOperations jdbc) {
        JdbcSessionRepository repo = new JdbcSessionRepository(jdbc);
        repo.setTableName("ARCHIVE_SPRING_SESSION");
        return repo;
    }
}
```

---

# 6. WebFlux 配置

## 6.1 WebFlux 环境 Session

```java
@Configuration
@EnableRedisWebSession
public class WebFluxSessionConfig {

    @Bean
    public RedisWebSessionRepository redisWebSessionRepository(
            ReactiveRedisOperations<String, Object> redis) {
        return new RedisWebSessionRepository(redis);
    }

    @Bean
    public WebSessionManager webSessionManager(
            WebSessionRepository repository) {
        DefaultWebSessionManager manager =
            new DefaultWebSessionManager();
        manager.setSessionRepository(repository);
        return manager;
    }
}
```

## 6.2 WebFlux 下的配置差异

| 差异点 | Servlet | WebFlux |
|--------|---------|---------|
| **入口** | `SessionRepositoryFilter` | `WebSessionManager` |
| **Session 接口** | `Session` | `WebSession` |
| **Repository** | `SessionRepository` | `ReactiveSessionRepository` |
| **存储** | 同步 Redis 操作 | 异步 ReactiveRedis 操作 |
