# 1. 核心概念问题

## 1.1 什么是 Spring Session？有什么作用？

**回答要点**：

- Spring Session 是 Spring 生态中用于管理 HTTP Session 的框架
- 核心目标：将 Session 从应用服务器内存中解耦，实现外部化存储
- 支持 Redis、JDBC、Hazelcast、MongoDB 等多种后端存储
- 透明替换：通过 `SessionRepositoryFilter` 替换原生 `HttpSession`，业务代码无需修改

## 1.2 什么场景下需要 Spring Session？

**回答要点**：

| 场景 | 说明 |
|------|------|
| 分布式部署 / 微服务 | 多实例共享 Session |
| 弹性伸缩 (K8s) | Pod 不能依赖本地内存 |
| Session 高可用 | 实例宕机不丢失 Session |
| 前后端分离 | Header 方式传递 Session ID |
| 统一 Session 管理 | 查询、统计、强制下线 |

---

# 2. 架构与原理问题

## 2.1 Spring Session 的核心架构是什么？

**回答要点**：

```
三层架构：
1. 适配层：SessionRepositoryFilter（Servlet）/ WebSessionManager（WebFlux）
2. 核心层：SessionRepository 接口、Session 接口、HttpSessionIdResolver
3. 存储层：Redis / JDBC / Hazelcast / MongoDB 实现
```

## 2.2 SessionRepositoryFilter 的工作原理？

**回答要点**：

- 包装 `HttpServletRequest` 和 `HttpServletResponse`
- 重写 `getSession()`，从存储加载而非容器创建
- 在响应提交时（`onResponseCommitted()`）持久化 Session
- 新 Session 自动写入 Cookie，失效 Session 自动清除 Cookie

## 2.3 如何在请求中识别和传递 Session？

**回答要点**：

- **Cookie 方式**（默认）：Cookie: `SESSION=uuid`
- **Header 方式**（REST API）：`X-Auth-Token: uuid`
- `HttpSessionIdResolver` 接口抽象解析逻辑
- 可自定义混合策略

---

# 3. 存储后端问题

## 3.1 Spring Session 支持哪些存储后端？

**回答要点**：

| 后端 | 实现类 | 适用场景 |
|------|--------|----------|
| Redis | `RedisSessionRepository` | 高性能、高并发（推荐） |
| JDBC | `JdbcSessionRepository` | 不想引入额外存储 |
| Hazelcast | `HazelcastSessionRepository` | 嵌入式、零运维 |
| MongoDB | `MongoSessionRepository` | 已有 MongoDB 技术栈 |
| HashMap | `MapSessionRepository` | 测试、单机 |

## 3.2 Redis 中 Session 的数据结构是什么？

**回答要点**：

- **Session 数据**：HASH (`spring:session:sessions:{id}`)，字段包括 `creationTime`、`lastAccessedTime`、`maxInactiveInterval`、`sessionAttr:xxx`
- **过期控制**：STRING (`spring:session:sessions:expires:{id}`)，利用 TTL 自动过期
- **用户索引**：SET (`spring:session:index:PRINCIPAL_NAME_INDEX_NAME:{username}`)，存储 Session ID 集合

**为什么需要两个 Key？**

Redis 的 HASH 不支持对单个 Field 设置 TTL，所以用专门的 STRING Key 配合 TTL 实现过期。

## 3.3 JDBC 存储和 Redis 存储的区别？

| 维度 | JDBC | Redis |
|------|------|-------|
| 读性能 | ms 级 | μs 级 |
| 过期机制 | 定时任务清理 | TTL 自动过期 |
| 运维 | 低（复用 DB） | 中（独立 Redis） |
| 适用规模 | 中小 | 大 |

---

# 4. 序列化问题

## 4.1 Spring Session 默认使用什么序列化？有什么问题？

**回答要点**：

- 默认使用 JDK 序列化 (`JdkSerializationRedisSerializer`)
- 问题：二进制格式不可读、不安全（反序列化漏洞）、类变更敏感、跨语言不兼容
- 推荐使用 Jackson JSON 序列化

## 4.2 如何自定义序列化器？

**回答要点**：

```java
@Bean
public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
    return new GenericJackson2JsonRedisSerializer();
}
```

---

# 5. 与 Spring Security 集成问题

## 5.1 Spring Session 与 Spring Security 如何协同工作？

**回答要点**：

- Spring Security 将 `SecurityContext` 存储在 Session 中（key 为 `SPRING_SECURITY_CONTEXT`）
- Spring Session 将这个 Session 持久化到外部存储
- `SessionRegistry` 使用 `SpringSessionBackedSessionRegistry`，基于 `FindByIndexNameSessionRepository` 管理并发会话
- Session Fixation 防护利用 `changeSessionId()` 实现

## 5.2 如何在分布式环境下实现并发会话控制？

**回答要点**：

- 使用 `FindByIndexNameSessionRepository` 按用户名索引所有 Session
- `SessionRegistry.getAllSessions()` 查询所有活跃会话
- 超过 `maximumSessions` 限制时，根据策略拒绝或踢出旧会话
- 因为 Session 存储在 Redis/DB 中，所有实例都能查询到最新的会话列表

## 5.3 SessionCreationPolicy 各策略的含义？

| 策略 | 行为 |
|------|------|
| `ALWAYS` | 始终创建 Session |
| `IF_REQUIRED` | 需要时才创建（默认） |
| `NEVER` | 不创建，可用已有的 |
| `STATELESS` | 完全不使用 Session |

---

# 6. 安全相关问题

## 6.1 如何防止 Session Fixation 攻击？

**回答要点**：

- Session Fixation：攻击者诱导用户使用已知 Session ID
- 防护：登录成功后自动更换 Session ID
- Spring Security 自动启用 `SessionFixationProtectionStrategy`
- Spring Session 通过 `changeSessionId()` 实现（替换 ID，保留数据）

## 6.2 Cookie 安全属性有哪些？

| 属性 | 作用 | 推荐值 |
|------|------|--------|
| `HttpOnly` | 禁止 JS 读取 | `true` |
| `Secure` | 仅 HTTPS | `true`（生产） |
| `SameSite` | 跨站请求控制 | `Lax` |

---

# 7. 性能与优化问题

## 7.1 如何优化 Spring Session 的性能？

**回答要点**：

1. 使用高性能序列化（Jackson JSON 或 Kryo）
2. 配置合理的连接池（Lettuce）
3. 使用 `FlushMode.ON_SAVE` 减少 Redis 写入
4. Session 仅存标识，大对象查缓存/DB
5. 避免不必要的 Session 创建
6. Redis 部署在同一网络区域（低延迟）

## 7.2 FlushMode 和 SaveMode 如何选择？

| 配置 | 说明 |
|------|------|
| `FlushMode.ON_SAVE` | 请求结束时统一保存（推荐） |
| `FlushMode.IMMEDIATE` | 立即保存（强一致场景） |
| `SaveMode.ON_SET_ATTRIBUTE` | 仅保存变更属性（推荐） |
| `SaveMode.ALWAYS` | 全量保存（简单但开销大） |

---

# 8. 常见问题排查

## 8.1 Session 不生效怎么办？

1. 检查 `spring-session-data-redis` 等依赖是否引入
2. 检查 Redis 是否正常运行（`redis-cli PING`）
3. 检查是否排除了 `SessionAutoConfiguration`
4. 开启 DEBUG 日志查看 `SessionRepositoryFilter` 是否注册

## 8.2 Redis 内存占用过高怎么办？

1. 缩短 Session 超时时间
2. 控制单 Session 大小（< 5KB）
3. 配置 `maximumSessions` 限制用户并发会话
4. 确保 Redis 键空间通知正常（过期 Key 能及时清理）

---

# 9. 与其他方案对比

## 9.1 Spring Session vs JWT

| 维度 | Spring Session | JWT |
|------|---------------|-----|
| 状态 | 有状态 | 无状态 |
| 数据存储 | 服务端（Redis/DB） | 客户端（Token 内） |
| Session 撤销 | 即时删除 | 需等过期或查黑名单 |
| 扩展性 | 依赖存储扩展 | 天然无状态 |
| 数据安全 | 服务端可控 | 客户端可见（需加密） |
| 适用 | 传统 Web、同域 | 跨域 API、第三方集成 |

## 9.2 Spring Session vs Tomcat Session 复制

| 维度 | Spring Session | Tomcat Session 复制 |
|------|---------------|---------------------|
| 架构 | 集中存储 | 对等复制 |
| 扩展性 | 好（存储独立扩展） | 差（节点多开销大） |
| 可靠性 | 高（存储高可用） | 一般 |
| 容器绑定 | 无 | 绑定 Tomcat |

---

# 10. 实践场景题

## 10.1 如何实现"踢人下线"功能？

1. 通过 `FindByIndexNameSessionRepository.findByIndexNameAndIndexValue()` 查找用户所有 Session
2. 调用 `sessionRepository.deleteById(id)` 删除指定 Session
3. 被踢出的用户下次请求时 Session 不存在，需要重新登录

## 10.2 如何实现"同账号多地登录检测与通知"？

1. 监听 `SessionCreatedEvent`，记录 Session ID + IP + 设备
2. 通过索引查找用户已有 Session
3. 如果数量超限，发送 WebSocket/消息通知
4. 提供前端界面让用户选择踢出哪个设备

## 10.3 跨域前后端分离如何传递 Session？

1. 使用 `HeaderHttpSessionIdResolver` 代替 Cookie
2. 前端登录成功获取响应头中的 Token
3. 后续请求在请求头中携带 Token
4. 同域方案更推荐：Nginx 反向代理统一域名

---

# 11. 知识体系总结

```
Spring Session 知识体系
│
├── 基础概念
│   ├── Session / HttpSession
│   ├── Session ID
│   └── Session 生命周期
│
├── 核心接口
│   ├── Session
│   ├── SessionRepository
│   ├── FindByIndexNameSessionRepository
│   ├── HttpSessionIdResolver
│   └── CookieSerializer
│
├── 存储后端
│   ├── Redis（推荐）
│   ├── JDBC
│   ├── Hazelcast
│   └── MongoDB
│
├── 核心机制
│   ├── SessionRepositoryFilter
│   ├── Session 变更追踪
│   ├── 过期处理
│   └── 事件机制
│
├── 安全集成
│   ├── SecurityContext 持久化
│   ├── SessionRegistry
│   ├── 并发会话控制
│   ├── Session Fixation
│   └── CSRF
│
├── 高级特性
│   ├── 序列化自定义
│   ├── 自定义 Session 策略
│   ├── 集群事件同步
│   ├── REST API 支持
│   └── WebFlux 支持
│
├── 运维
│   ├── 性能优化
│   ├── 监控告警
│   ├── 容量规划
│   └── 故障排查
│
└── 最佳实践
    ├── 配置规范
    ├── 编码规范
    ├── 安全规范
    └── 部署规范
```

---

# 12. 快速回顾卡片

| 问题 | 一句话答案 |
|------|-----------|
| Spring Session 做什么？ | 将 HTTP Session 从服务器内存移到外部存储 |
| 核心 Filter？ | `SessionRepositoryFilter`，替换原生 HttpSession |
| 默认存储后端？ | 检测 classpath，Redis 优先 |
| Redis 中几个 Key？ | 3 个：HASH（数据）、STRING（过期）、SET（索引） |
| 如何选序列化？ | 推荐 Jackson JSON，追求极致用 Kryo |
| 如何实现并发控制？ | `SpringSessionBackedSessionRegistry` + `maximumSessions` |
| Cookie vs Header？ | Cookie 默认（同域），Header（前后端分离） |
| 如何自定义 Cookie？ | `DefaultCookieSerializer` 配置 |
| FlushMode 推荐？ | `ON_SAVE`（请求结束统一保存） |
| SaveMode 推荐？ | `ON_SET_ATTRIBUTE`（增量保存） |
