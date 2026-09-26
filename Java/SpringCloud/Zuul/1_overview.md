
# 1. 什么是 API 网关

API 网关（API Gateway）是微服务架构中的关键组件，位于客户端与后端服务之间，承担所有外部请求的入口角色。它的核心职责包括：

- **路由转发**：将请求路由到正确的后端服务
- **统一认证**：在入口处进行身份验证和授权
- **限流熔断**：保护后端服务免受突发流量冲击
- **协议转换**：HTTP/HTTPS、HTTP/2、WebSocket 等协议适配
- **日志监控**：统一收集请求日志和调用链数据

在微服务架构中，如果没有网关，客户端需要直接与各个微服务通信，导致以下问题：

```
客户端 ──→ 服务A
客户端 ──→ 服务B
客户端 ──→ 服务C    ← 紧耦合、鉴权分散、跨域问题
```

引入网关后：

```
客户端 ──→ 网关 ──→ 服务A
                 ──→ 服务B          ← 统一入口、集中鉴权、路由解耦
                 ──→ 服务C
```

# 2. Zuul 简介

Zuul 是 Netflix 开源的 JVM 边缘服务（Edge Service），核心定位是**所有请求的前门**（The Front Door）。Spring Cloud 对其进行了封装集成，使其成为 Spring Cloud 生态中最早的一代 API 网关解决方案。

## 2.1 发展历程

| 版本 | 描述 |
|------|------|
| Zuul 1.x | 基于 Servlet 2.5 的阻塞 I/O 模型，使用 Tomcat 或 Jetty 作为容器 |
| Zuul 2.x | 基于 Netty 的异步非阻塞 I/O 模型，支持 HTTP/2 和 WebSocket |

## 2.2 Zuul 的核心能力

- **动态路由**：基于服务注册中心（Eureka/Consul/Nacos）动态发现后端服务
- **过滤器链机制**：pre → route → post 三道防线，支持自定义扩展
- **压力测试**：内置过滤器可注入测试流量，逐步增加对集群的负载
- **金丝雀发布**：通过过滤器实现流量按比例分流至新版本服务
- **多区域弹性**：跨 AWS Region 的请求路由，将请求转移到其他区域
- **请求洞察**：记录所有请求的详细统计信息

# 3. Zuul 1.x vs Zuul 2.x

## 3.1 架构差异

```
Zuul 1.x 架构（阻塞模型）：

  请求 → Servlet 容器（Tomcat）→ ZuulServlet
                                    │
                            ┌───────┼───────┐
                            ▼       ▼       ▼
                          pre    route    post
                         Filter  Filter  Filter
                            │       │       │
                            ▼       ▼       ▼
                        HTTP Client（阻塞 I/O）

Zuul 2.x 架构（异步模型）：

  请求 → Netty Server → NettyRequestHandler
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
      InboundFilters   EndpointFilters   OutboundFilters
            │               │               │
            ▼               ▼               ▼
      Netty Client（异步非阻塞 I/O）
```

## 3.2 关键区别

| 维度 | Zuul 1.x | Zuul 2.x |
|------|----------|----------|
| I/O 模型 | 阻塞 I/O（BIO） | 非阻塞 I/O（NIO） |
| 运行容器 | Tomcat/Jetty（Servlet 容器） | Netty |
| 过滤模型 | pre/route/post/error | inbound/endpoint/outbound |
| 连接管理 | 每个请求占用一个线程 | 事件驱动，线程复用 |
| HTTP/2 | 不支持 | 原生支持 |
| WebSocket | 不支持 | 支持 |
| 背压（Backpressure） | 无 | 有（RxJava） |
| 生产就绪 | ✅ 久经考验 | ⚠️ 2018 年开源，生态较新 |
| Spring Cloud 集成 | Spring Cloud Netflix Zuul | 未正式集成（需自行适配） |

# 4. Zuul 1.x 的线程模型与阻塞问题

理解 Zuul 1.x 的阻塞特性至关重要。因为 Zuul 1.x 基于 Servlet 2.5 规范，每个请求都会绑定一个 Tomcat 工作线程：

```java
// ZuulServlet.service() 方法简化逻辑
public void service(ServletRequest servletRequest, ServletResponse servletResponse) {
    // 1. 初始化 RequestContext（基于 ThreadLocal）
    init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);

    try {
        // 2. 执行 pre 过滤器
        preRoute();

        // 3. 执行 route 过滤器（此处会发起阻塞的 HTTP 调用）
        route();

        // 4. 执行 post 过滤器
        postRoute();
    } catch (Throwable e) {
        // 5. 执行 error 过滤器
        error(e);
    } finally {
        // 6. 清理 RequestContext，防止内存泄漏
        RequestContext.getCurrentContext().unset();
    }
}
```

当 route 阶段通过 HttpClient 发起对后端服务的调用时，当前 Tomcat 线程会**完全阻塞**直到收到响应。这意味着：

- 连接数 = 线程数，高并发时需要大量线程
- 线程切换引入额外开销
- 后端服务响应慢时，网关线程会被大量占满，导致雪崩

# 5. Zuul 1.x 与 Spring Cloud Gateway 对比

Spring Cloud Gateway 是 Spring 官方推出的第二代网关，基于 Spring 5 WebFlux 和 Project Reactor 构建：

| 维度 | Zuul 1.x | Spring Cloud Gateway |
|------|----------|---------------------|
| I/O 模型 | 阻塞 I/O | 非阻塞 I/O（Netty） |
| 编程范式 | Servlet（命令式） | WebFlux（响应式） |
| 路由定义 | 配置文件 + RouteLocator | RouteLocator + Predicate + Filter |
| 过滤器 | ZuulFilter（pre/route/post/error） | GatewayFilter + GlobalFilter |
| 限流 | 需自定义实现 | 内置 RequestRateLimiter |
| 熔断 | Hystrix（已停更） | Resilience4j |
| 动态路由 | 需配合 Archaius | 内置 Actuator 端点 |
| Spring Boot 版本 | 2.3.x 后停止更新 | 持续更新（Spring Boot 3.x 已支持） |
| 社区活跃度 | Netflix OSS 全线进维护模式 | Spring 官方主推 |

综合来看，对于新项目，Spring Cloud Gateway 是更优选择。但 Zuul 1.x 作为经典实现，其设计思想和源码架构仍有较高的学习价值。

# 6. 为什么还要学习 Zuul

尽管 Spring Cloud Netflix 已进入维护模式，学习 Zuul 仍有以下价值：

1. **大量存量项目**：许多生产系统仍运行在 Spring Cloud Hoxton 及之前版本，深入理解 Zuul 有助于维护和调优
2. **设计模式借鉴**：Zuul 的过滤器链模式是责任链模式的经典应用，理解它对设计类似中间件系统有帮助
3. **迁移基础**：理解 Zuul 的行为才能在向 Gateway 迁移时做对等的功能映射
4. **面试考察**：Zuul 与 Gateway 的对比、核心源码实现是大厂面试中的高频话题
