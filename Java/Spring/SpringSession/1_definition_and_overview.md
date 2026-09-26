# 1. 定义与作用

Spring Session 是 Spring 生态中用于**管理用户会话（Session）信息**的框架，核心目标是**将 HTTP Session 从应用服务器的内存中解耦**，实现会话的外部化存储，从而支持分布式和云原生场景。

**一句话概括**：Spring Session 让你不再依赖 `HttpSession` 的默认实现（如 Tomcat 的内存 Session），而是将 Session 数据存储到 Redis、JDBC 数据库、Hazelcast、MongoDB 等外部存储中。

> 版本说明：本文档基于 **Spring Session 3.x + Spring Boot 3.x**，JDK 17+。

---

## 1.1 为什么需要 Spring Session

### 1.1.1 传统 Session 的局限

在传统的 Java Web 应用中，Session 管理存在以下痛点：

| 问题 | 说明 |
|------|------|
| **单点故障** | Session 存储在单个服务器内存中，服务器宕机则 Session 全部丢失 |
| **无法水平扩展** | 多个应用实例无法共享 Session，负载均衡需要配置 Sticky Session（黏性会话） |
| **内存压力大** | 大量用户 Session 占用 JVM 堆内存，可能导致 Full GC 和应用性能下降 |
| **重启丢失** | 应用重启时所有 Session 都会丢失，影响用户体验 |
| **容器绑定** | Session 实现与 Servlet 容器（Tomcat/Jetty/Undertow）紧耦合 |
| **难以管理** | 无法统一对 Session 进行查询、统计、分析、清理等操作 |
| **跨域/跨应用** | 多应用共享 Session 困难，需要复杂的 Session 复制机制 |

### 1.1.2 Spring Session 提供的解决方案

```
传统架构：                          Spring Session 架构：
┌─────────────┐                    ┌─────────────┐
│  浏览器      │                    │  浏览器      │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       ▼                                  ▼
┌─────────────┐                    ┌─────────────┐
│  负载均衡    │                    │  负载均衡    │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
  ┌────┴────┐                        ┌────┴────┐
  ▼         ▼                        ▼         ▼
┌────┐   ┌────┐                  ┌────┐   ┌────┐
│App1│   │App2│                  │App1│   │App2│
│ ┌─┐│   │ ┌─┐│                  │    │   │    │
│ │S││   │ │S││                  └──┬─┘   └──┬─┘
│ └─┘│   │ └─┘│                     │       │
└────┘   └────┘                     └───┬───┘
 Session 独立存储                         │
                                   ┌──────▼──────┐
                                   │   Redis/DB  │
                                   │  (共享Session)│
                                   └─────────────┘
```

**核心优势**：

| 优势 | 说明 |
|------|------|
| **无状态应用** | 应用服务器不再持有 Session，可随意扩缩容 |
| **高可用** | 外部存储（如 Redis 集群）保障 Session 数据不丢失 |
| **跨实例共享** | 多个应用实例天然共享 Session 数据 |
| **容器无关** | 不依赖特定 Servlet 容器 |
| **统一管理** | 可在外部存储中统一查询、清理、分析 Session |
| **RESTful 支持** | 支持无状态 REST API 的 Session/Token 管理 |

---

## 1.2 Spring Session 核心能力

### 1.2.1 核心特性概览

| 特性 | 说明 |
|------|------|
| **多种后端存储** | Redis、JDBC、Hazelcast、MongoDB 等 |
| **透明的 Session 替代** | 通过 Filter 替换原生 `HttpSession`，业务代码无需修改 |
| **REST API 支持** | 通过 `HttpSessionIdResolver` 支持 Header 或 Cookie 中的 Session ID |
| **Session 事件** | `SessionCreatedEvent`、`SessionDeletedEvent`、`SessionExpiredEvent` 等 |
| **集群支持** | 通过 Redis Pub/Sub 或 Hazelcast Topic 同步 Session 变更事件 |
| **Spring Security 集成** | 无缝集成 Spring Security 的并发会话控制、Session 注册表等 |
| **可扩展序列化** | 支持 JDK、JSON、Kryo、Protobuf 等多种序列化方式 |
| **Session 查找** | 支持通过 Session ID 或用户信息查找和管理 Session |

### 1.2.2 版本演进

| 版本 | 主要特性 |
|------|----------|
| **1.x** | 初始版本，支持 Redis 存储，通过 `springSessionRepositoryFilter` 替换原生 Session |
| **2.x** | 支持 Spring Framework 5.x，引入 Reactor 响应式编程支持 (Spring WebFlux) |
| **3.x** | 支持 Spring Boot 3.x / Spring Framework 6.x，JDK 17+，Jakarta EE 迁移，新增 `RedisIndexedSessionRepository` 改进 |

---

## 1.3 适用场景

### 1.3.1 典型应用场景

```
场景一：微服务 + 分布式部署
┌─────────────────────────────────────┐
│  Service A    Service B    Service C│
│  (用户服务)   (订单服务)   (商品服务) │
└──────────┬──────────────────────────┘
           │
    ┌──────▼──────┐
    │     Redis   │  ← 统一 Session 存储
    └─────────────┘
```

| 场景 | 具体需求 |
|------|----------|
| **微服务架构** | 多个微服务需要共享用户登录状态 |
| **容器化部署 (K8s)** | Pod 弹性伸缩，不能依赖 Pod 本地内存 |
| **蓝绿发布 / 滚动更新** | 新老版本实例并存，需要共享 Session |
| **前后端分离** | REST API 需要 Token 或 Header 方式传递 Session ID |
| **多活数据中心** | 跨机房 Session 复制与同步 |
| **高并发网站** | 减轻应用服务器内存压力，Session 存于 Redis |
| **移动端** | 移动 APP 与 Web 端共享同一登录态 |

### 1.3.2 不适用场景

| 场景 | 原因 |
|------|------|
| **纯静态网站** | 无需 Session |
| **完全无状态的 JWT 认证** | 无需服务端 Session 存储 |
| **单机小型应用** | 引入外部存储增加了复杂度，收益不明显 |
| **对延迟极度敏感** | 网络往返（如 Redis RTT）会增加几毫秒延迟 |

---

## 1.4 与其他技术的关系

### 1.4.1 Spring Session vs Spring Security

```
Spring Security
  └── 认证（Authentication）
  └── 授权（Authorization）
  └── 并发会话控制 ←── 依赖 Spring Session
  └── Session 注册表  ←── 依赖 Spring Session
  └── 防 Session Fixation ←── 依赖 Session 策略

Spring Session
  └── Session 外部化存储
  └── Session 事件发布
  └── REST API Session 解析
  └── 多会话策略
```

- **Spring Security** 负责**安全**（认证、授权）
- **Spring Session** 负责**会话存储和管理**
- 两者集成后，Spring Security 的 `SessionRegistry` 可以借助 Spring Session 实现分布式场景下的并发会话控制

### 1.4.2 Spring Session vs JWT / OAuth2 Token

| 维度 | Spring Session | JWT / OAuth2 Token |
|------|---------------|---------------------|
| **状态** | 有状态（服务端持有数据） | 无状态（Token 自包含信息） |
| **存储** | 需要外部存储（Redis/DB） | 无需存储（或仅存黑名单） |
| **撤销** | 可立即删除 Session | 需要等到 Token 过期或检查黑名单 |
| **数据量** | Session 可存储大量数据 | Token 不宜过大（HTTP 头限制） |
| **性能** | 有网络 I/O（查 Redis） | 纯 CPU 计算（验签） |
| **安全** | 可随时失效 | 已签发的 Token 难以撤销 |
| **适合场景** | 传统 Web 应用、前后端同域 | 跨域 API、移动端、第三方集成 |

### 1.4.3 Spring Session vs Tomcat Session 复制

| 维度 | Spring Session | Tomcat 集群 Session 复制 |
|------|---------------|---------------------------|
| **实现方式** | 外部存储（Redis/DB） | 多播/广播 Session 到其他节点 |
| **扩展性** | 好（存储独立扩展） | 差（节点越多复制开销越大） |
| **可靠性** | 好（存储高可用） | 一般（复制可能丢失） |
| **性能** | 取决于外部存储 | 复制有网络开销 |
| **耦合度** | 低（与容器解耦） | 高（绑定 Tomcat） |

---

## 1.5 基本术语

| 术语 | 说明 |
|------|------|
| **Session** | 服务器端维护的用户会话对象，用于在多次 HTTP 请求间保持状态 |
| **Session ID** | 会话的唯一标识符，通常通过 Cookie (JSESSIONID) 或 Header 传递 |
| **SessionRepository** | Spring Session 的核心接口，定义 Session 的 CRUD 操作 |
| **SessionRepositoryFilter** | 替换原生 `HttpSession` 的 Servlet Filter |
| **HttpSessionIdResolver** | 解析请求中的 Session ID（从 Cookie 或 Header） |
| **HttpSessionWrapper** | 包装原生 `HttpSession` 实现 Spring Session 行为 |
| **SessionEvent** | Spring Session 发布的会话生命周期事件 |
| **FindByIndexNameSessionRepository** | 扩展接口，支持按用户索引查找 Session |
| **SessionDestroyedEvent** | Session 销毁或过期时发布的事件 |

---

## 1.6 Maven / Gradle 依赖

### 1.6.1 父 POM

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>
```

### 1.6.2 核心依赖

```xml
<!-- Spring Session 核心 -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-core</artifactId>
</dependency>
```

### 1.6.3 各后端存储依赖

```xml
<!-- Redis 后端 -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>

<!-- JDBC 后端 -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-jdbc</artifactId>
</dependency>

<!-- Hazelcast 后端 -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-hazelcast</artifactId>
</dependency>

<!-- MongoDB 后端 (需额外引入 spring-data-mongodb) -->
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-mongodb</artifactId>
</dependency>
```
