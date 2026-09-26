# 1. Session 不生效 / 仍是 Tomcat 原生 Session

## 1.1 问题表现

- Session 数据没有存储到 Redis/DB
- 多实例之间 Session 不同步
- `request.getSession()` 返回的是 Tomcat 原生 HttpSession

## 1.2 排查步骤

```bash
# 1. 检查依赖是否引入
mvn dependency:tree | grep spring-session

# 2. 检查是否误排除了自动配置
# 查看启动日志是否有 SessionRepositoryFilter 的注册信息

# 3. 检查 spring-session-data-redis 版本是否与 Spring Boot 版本匹配
```

## 1.3 常见原因与解决

| 原因 | 解决方案 |
|------|----------|
| 没引入存储后端依赖（如 `spring-session-data-redis`） | 添加对应 Maven 依赖 |
| 排除了 `SessionAutoConfiguration` | 移除排除配置 |
| 自定义了 `SessionRepositoryFilter` 但配置不正确 | 确保 Bean 命名和注册方式正确 |
| WebFlux 环境下使用了 Servlet 的配置 | 改用 `@EnableRedisWebSession` |

```java
// 验证 Spring Session 是否生效
@RestController
public class SessionCheckController {

    @Autowired(required = false)
    private RedisSessionRepository redisRepository;

    @GetMapping("/check/session-repository")
    public String checkRepository() {
        if (redisRepository != null) {
            return "Spring Session (Redis) 已生效: "
                + redisRepository.getClass().getName();
        }
        return "Spring Session 可能未配置";
    }
}
```

---

# 2. Redis 连接问题

## 2.1 连接超时

```
错误: Unable to connect to Redis; nested exception is
io.lettuce.core.RedisConnectionException

解决方案：
1. 检查 Redis 是否正在运行: redis-cli PING
2. 检查 host/port 配置是否正确
3. 检查防火墙是否放行 6379 端口
4. 检查 Redis bind 配置 (redis.conf: bind 0.0.0.0)
```

## 2.2 Redis 密码认证失败

```
错误: NOAUTH Authentication required

解决方案：
spring.data.redis.password: your-password
```

## 2.3 集群模式配置错误

```
错误: MOVED 重定向处理失败

常见原因：
- Lettuce 未正确配置 ClusterTopologyRefresh
- Spring Boot 版本与 Lettuce 版本不兼容
- 集群节点列表不全

解决方案：
spring.redis.cluster.nodes: node1:6379,node2:6379,node3:6379
spring.redis.cluster.max-redirects: 3
```

---

# 3. Session 序列化问题

## 3.1 反序列化失败

```
错误: Cannot deserialize; nested exception is
org.springframework.core.serializer.support.SerializationFailedException

常见场景：
- 类结构变更（新增/删除字段）
- 类路径变更（包名改名）
- 跨版本部署（新旧版本读取同一份 Session 数据）
```

**解决方案**：

```java
// 1. 使用 JSON 序列化替代 JDK
@Bean
public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
    return new GenericJackson2JsonRedisSerializer();
}

// 2. 设置 FAIL_ON_UNKNOWN_PROPERTIES = false
ObjectMapper mapper = new ObjectMapper();
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
```

## 3.2 序列化循环引用栈溢出

```
错误: java.lang.StackOverflowError at
com.fasterxml.jackson.databind.ser.BeanSerializer.serialize

原因：对象之间存在循环引用

解决方案：
- 使用 @JsonIgnore 打断循环
- 使用 @JsonManagedReference / @JsonBackReference
- 使用 @JsonIdentityInfo 标记引用
```

---

# 4. 并发会话控制问题

## 4.1 并发控制不生效

```java
// 确认 SessionRegistry Bean 是否存在
@Autowired(required = false)
private SessionRegistry sessionRegistry;
// 如果为 null，说明未自动配置

// 手动创建
@Bean
public SessionRegistry sessionRegistry(
        FindByIndexNameSessionRepository<?> repository) {
    return new SpringSessionBackedSessionRepository<>(repository);
}
```

## 4.2 用户被频繁踢下线

**原因**：多个请求同时触发 Session 创建（如 AJAX 请求加载多个资源）

**解决方案**：

```java
// 注册自定义 SessionAuthenticationStrategy
@Bean
public SessionAuthenticationStrategy sessionAuthenticationStrategy() {
    ConcurrentSessionControlAuthenticationStrategy strategy =
        new ConcurrentSessionControlAuthenticationStrategy(sessionRegistry());
    strategy.setMaximumSessions(5);    // 适当增加上限
    strategy.setExceptionIfMaximumExceeded(false);
    return new CompositeSessionAuthenticationStrategy(Arrays.asList(
        new ConcurrentSessionControlAuthenticationStrategy(sessionRegistry()),
        new SessionFixationProtectionStrategy(),
        new RegisterSessionAuthenticationStrategy(sessionRegistry())
    ));
}
```

---

# 5. Session 过期问题

## 5.1 Session 过早过期

**排查**：

```bash
# Redis 查看 Session 的 TTL
redis-cli TTL "spring:session:sessions:abc123"

# 检查过期时间是否与预期一致
redis-cli HGET "spring:session:sessions:abc123" maxInactiveInterval
```

**原因**：

| 原因 | 解决 |
|------|------|
| `spring.session.timeout` 设置过短 | 增加超时时间 |
| 多个 `server.servlet.session.timeout` 配置冲突 | 统一使用 `spring.session.timeout` |
| 代码中调用了 `setMaxInactiveInterval()` | 检查代码逻辑 |
| Redis 的 expires key 被手动删除 | 检查 Redis 运维操作 |

## 5.2 Session 永不过期

```bash
# 检查 expires key 是否存在
redis-cli EXISTS "spring:session:sessions:expires:abc123"

# 如果 expires key 不存在
# 可能是 RedisConfig 禁用了 expire 或 TTL 设置异常
```

---

# 6. Session 数据不一致

## 6.1 跨实例数据不同步

**原因**：`FlushMode.IMMEDIATE` 未开启，且一个请求在多个实例间切换时

**解决方案**：

- 大多数场景下短暂不一致是可接受的
- 如果需要强一致，设置 `FlushMode.IMMEDIATE`
- 或使用分布式锁保护 Session 写入

## 6.2 Session 属性丢失

**原因**：`SaveMode.ON_SET_ATTRIBUTE` 下修改了可变对象内部的属性

```java
// 问题代码：
User user = session.getAttribute("user");
user.setName("newName");  // 直接修改对象内部属性
// ON_SET_ATTRIBUTE 模式下，此变更不会被持久化！

// 解决方案 1：启用 ON_GET_ATTRIBUTE
// 解决方案 2：重新 setAttribute
session.setAttribute("user", user);

// 解决方案 3：使用不可变对象
record UserInfo(String name, String email) {} // Java 14+
```

---

# 7. 内存与性能问题

## 7.1 Redis 内存飙升

**排查**：

```bash
# 查看 Session Key 数量
redis-cli DBSIZE
redis-cli --scan --pattern "spring:session:*" | wc -l

# 查看每个 Session 的大小
redis-cli --bigkeys

# 查看内存使用
redis-cli INFO memory | grep used_memory_human
```

**原因与解决**：

| 原因 | 解决 |
|------|------|
| Session 超时时间过长 | 缩短超时时间 |
| Session 属性过大 | 控制属性大小，存标识不存大对象 |
| 过期 Key 未及时清理 | 确保 `notify-keyspace-events` 启用 |
| 用户并发 Session 太多 | 限制 `maximumSessions` |

## 7.2 高并发下性能瓶颈

```bash
# 查看 Redis 慢日志
redis-cli SLOWLOG GET 100

# 如果 Session 操作出现在慢日志中：
# 1. 检查是否使用了 JDK 序列化 → 改用 Kryo
# 2. 检查网络延迟 → 就近部署 Redis
# 3. 检查 Redis 是否在单线程处理大 Key
```

---

# 8. 浏览器 Cookie 问题

## 8.1 Cookie 不生效

| 问题 | 原因 | 解决 |
|------|------|------|
| Cookie 不发送 | `secure=true` 但用了 HTTP | 开发环境设置 `secure: false` |
| Cookie 不发送 | `sameSite=Strict` 过于严格 | 改用 `Lax` |
| Cookie 丢失 | `domain` 设置不匹配 | 检查域名配置 |
| Cookie 值被覆盖 | Cookie 名称冲突 | 使用自定义 Cookie 名称 |

## 8.2 跨域 Cookie 问题

前后端分离且不同域名时，Cookie 天然无法跨域：

```yaml
# 同域方案（推荐）
# 使用 Nginx 反向代理，统一域名

# 跨域方案
# 使用 HeaderHttpSessionIdResolver 替代 Cookie
# 前端手动管理 Token
```

---

# 9. JDBC 相关问题

## 9.1 建表失败

```
错误: Table 'SPRING_SESSION' doesn't exist

解决方案：
spring.session.jdbc.initialize-schema: always
# 或手动执行建表 SQL
```

## 9.2 性能问题

```sql
-- 检查是否有慢查询
EXPLAIN SELECT * FROM SPRING_SESSION WHERE SESSION_ID = 'xxx';

-- 检查表大小
SELECT COUNT(*) FROM SPRING_SESSION;
SELECT COUNT(*) FROM SPRING_SESSION_ATTRIBUTES;

-- 清理孤儿属性
DELETE a FROM SPRING_SESSION_ATTRIBUTES a
LEFT JOIN SPRING_SESSION s ON a.SESSION_PRIMARY_ID = s.PRIMARY_ID
WHERE s.PRIMARY_ID IS NULL;
```

---

# 10. 调试技巧

## 10.1 启用 DEBUG 日志

```yaml
logging:
  level:
    org.springframework.session: DEBUG
    org.springframework.session.data.redis: TRACE
    org.springframework.session.web.http: DEBUG
```

## 10.2 自定义 Filter 验证

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SessionDebugFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res,
            FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;

        log.info("请求: {} {}, SessionId={}, Cookies={}",
            request.getMethod(),
            request.getRequestURI(),
            request.getRequestedSessionId(),
            request.getHeader("Cookie")
        );

        chain.doFilter(req, res);

        HttpSession session = request.getSession(false);
        if (session != null) {
            log.info("响应: SessionId={}, Attrs={}",
                session.getId(),
                Collections.list(session.getAttributeNames())
            );
        }
    }
}
```
