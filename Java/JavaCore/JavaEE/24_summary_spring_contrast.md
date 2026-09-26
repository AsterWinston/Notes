
# 1. 全规范速查总表

| 领域 | Jakarta EE 规范 | Spring 等价 | 实战地位 |
|------|----------------|-------------|----------|
| Web | Servlet | DispatcherServlet（底层） | ⭐⭐⭐⭐⭐ |
| Web | JSP/EL/JSTL | Thymeleaf | ⭐ |
| Web | JSF | Spring MVC / 前后端分离 | ⭐ |
| Web | WebSocket | Spring WebSocket/STOMP | ⭐⭐⭐ |
| 组件 | CDI | Spring IoC | ⭐⭐⭐⭐⭐ |
| 组件 | Jakarta DI | `@Autowired` | ⭐⭐⭐⭐⭐ |
| 组件 | Interceptors | Spring AOP | ⭐⭐⭐⭐ |
| 组件 | EJB | Spring 声明式事务等 | ⭐⭐ |
| 数据 | JDBC + DataSource | JdbcTemplate/HikariCP | ⭐⭐⭐⭐ |
| 数据 | JPA | Spring Data JPA（基于 JPA） | ⭐⭐⭐⭐ |
| 数据 | Bean Validation | Spring 校验（复用） | ⭐⭐⭐⭐ |
| 事务 | JTA | Spring `@Transactional` | ⭐⭐⭐⭐ |
| 集成 | JNDI | `@Value`/配置中心 | ⭐⭐ |
| 集成 | JMS | Spring JMS/RabbitTemplate | ⭐⭐⭐ |
| 集成 | JavaMail | JavaMailSender | ⭐⭐⭐ |
| 集成 | JCA | Spring 事务资源管理 | ⭐ |
| Web服务 | JAX-WS | Spring Web Services | ⭐ |
| Web服务 | JAX-RS | Spring MVC `@RestController` | ⭐⭐⭐ |
| 数据格式 | JAXB/JAXP | Jackson | ⭐ |
| 数据格式 | JSON-P/JSON-B | Jackson | ⭐⭐⭐ |
| 安全 | Jakarta Security | Spring Security | ⭐⭐⭐⭐ |
| 并发 | Jakarta Concurrency | `@Async`/`@Scheduled` | ⭐⭐ |
| 批处理 | Jakarta Batch | Spring Batch | ⭐⭐⭐ |
| 远程 | RMI | gRPC/Dubbo/HTTP | ⭐ |
| 部署 | EAR/WAR | 可执行 JAR/Docker | ⭐⭐⭐ |

# 2. 核心思想对比

## 2.1 两者共通的底层思想

```
依赖注入（DI）      ：CDI @Inject  ≈ Spring @Autowired
面向切面（AOP）     ：Interceptors  ≈ Spring AOP
声明式事务          ：@TransactionAttribute ≈ @Transactional
声明式安全          ：@RolesAllowed  ≈ @PreAuthorize
容器管理生命周期    ：Managed Bean  ≈ Spring Bean
```

- Spring 的很多设计**借鉴并统一了** JavaEE 的规范思想。
- JavaEE 通过**规范 + 应用服务器**实现；Spring 通过**框架 + 内嵌容器**实现，更轻量、更灵活。

## 2.2 核心差异

| 维度 | JavaEE（Jakarta EE） | Spring |
|------|----------------------|--------|
| 定位 | 平台规范 | 框架 |
| 实现方式 | 规范 + 服务器实现 | 框架 + IoC 容器 |
| 部署 | 需应用服务器 | 可内嵌（可执行 JAR） |
| 配置 | 注解 + XML 描述符 | 注解 + YAML（自动配置） |
| 灵活性 | 标准化优先 | 约定 + 可定制 |
| 生态 | 相对封闭 | 极其丰富 |
| 演进 | Eclipse 基金会 | Pivotal/VMware 社区 |

# 3. 学习路线回顾

```
第 1 步：Web 层      Servlet/JSP/EL/JSTL       → JavaWeb 笔记（实操）
第 2 步：组件模型    CDI/DI/拦截器/托管Bean    → 理解 IoC/AOP 本质
第 3 步：数据与事务  JDBC/JPA/JTA              → 理解持久化与事务边界
第 4 步：集成能力    JNDI/JMS/JavaMail/JCA     → 理解资源解耦与消息
第 5 步：Web 服务    JAX-WS/JAX-RS             → 理解 SOAP 与 REST
第 6 步：安全/并发   Security/Concurrency/Batch→ 理解声明式能力
第 7 步：部署        EAR/WAR                   → 理解企业应用形态
```

## 3.1 高频考点（面试）

| 主题 | 要点 |
|------|------|
| Servlet | 生命周期、转发 vs 重定向、异步 |
| JSP | 内置对象、四大作用域、与 Servlet 关系 |
| Filter/Listener | 执行链、8 种监听器 |
| JPA | 实体状态、JPQL、N+1、事务与持久化上下文 |
| JTA/事务 | REQUIRED 传播、回滚规则、XA/2PC |
| CDI | 作用域、限定符、事件、客户端代理 |
| JavaEE 迁移 | javax → jakarta、版本兼容 |
| 应用服务器 | Tomcat vs 应用服务器 |

# 4. JavaEE 与 Spring 的共存形态

```
形态一：纯 JavaEE     → 应用服务器（GlassFish/WildFly）+ EJB/JSF/JPA
形态二：纯 Spring     → Spring Boot + 内嵌 Tomcat + Spring 全家桶
形态三：混合（存量）  → 应用服务器 + Spring（通过 jndi/事务管理器集成）
```

- **新项目**：几乎全部选择 Spring Boot（或 Quarkus/Micronaut 等现代框架）。
- **存量企业系统**：大量 JavaEE/EJB 代码仍在金融、电信、政府运行。
- **学习价值**：理解 JavaEE 规范有助于：
  - 读源码（Spring 大量兼容 `jakarta.*` 标准）。
  - 处理存量系统迁移。
  - 理解"标准规范 vs 框架实现"的演进逻辑。

# 5. JavaEE 的现代演化

## 5.1 Jakarta EE 未来方向

- **云原生友好**：更轻量的配置、对容器/微服务优化。
- **Java 版本同步**：Jakarta EE 11 支持 Java 21。
- **与框架共存**：Quarkus（GraalVM）、Micronaut 等基于 JavaEE 规范的现代实现。

## 5.2 参考框架

| 框架 | 特点 |
|------|------|
| Quarkus | GraalVM 原生编译、云原生、基于 CDI/Servlet 等标准 |
| Micronaut | 编译期 DI、启动快 |
| Helidon | Oracle 云原生框架 |

# 6. 结语

JavaEE（Jakarta EE）定义了 Java 企业级开发的**标准底座**。虽然现代开发以 Spring 生态为主，但 JavaEE 的规范思想（依赖注入、容器管理生命周期、声明式事务/安全/消息）贯穿了整个 Java 服务端技术栈。掌握规范本质，再看 Spring 的封装与实现，会事半功倍。

> 相关延伸阅读：`JavaCore/JavaWeb`（Web 层实操）、`DataMiddleware/Hibernate`（JPA 实现）、`Messaging/`（消息中间件）、`Spring/`（框架实现）、`Interview/4_java_web.md` 与 `Interview/5_tomcat.md`（面试整理）。
