
# 1. 概述

Feign 是一个**声明式 HTTP 客户端**，受 Retrofit、JAX-RS 2.0 和 WebSocket 的启发。它的核心理念是：**通过编写 Java 接口并添加注解即可定义 HTTP 客户端，无需手动构造请求**。

OpenFeign 是 Feign 的开源社区维护版本（Feign 原由 Netflix 开发，后停止维护），Spring Cloud OpenFeign 则是在 OpenFeign 基础上与 Spring 生态（Spring MVC 注解、Ribbon/LoadBalancer、服务发现等）深度集成的产物。

# 2. 声明式调用 vs 命令式调用

**命令式调用（RestTemplate）**：

```java
String url = "http://user-service/api/users/" + userId;
User user = restTemplate.getForObject(url, User.class);
```

需要手动拼接 URL、处理参数、选择 HTTP 方法。

**声明式调用（Feign）**：

```java
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable("id") Long id);
}
```

只需定义接口，Feign 自动生成实现。

# 3. 核心优势

| 特性 | 说明 |
|------|------|
| **声明式** | 注解驱动，代码即文档，接口即契约 |
| **可插拔** | Encoder、Decoder、Client、Contract 等均可替换 |
| **集成性强** | 无缝集成 Spring MVC 注解、负载均衡、服务发现、熔断 |
| **HTTP 抽象** | 屏蔽底层 HTTP 通信细节（HttpURLConnection / Apache HttpClient / OkHttp） |
| **拦截器链** | 请求/响应拦截，支持鉴权、日志、追踪等横切逻辑 |

# 4. 发展历史

```
Netflix Feign (9.x, 停止维护)
    → OpenFeign (9.x → 10.x → 11.x → 12.x+)
        → Spring Cloud OpenFeign (2.x → 3.x → 4.x)
```

- **Netflix Feign 9.x**：原始版本，仅支持 Feign 原生注解
- **OpenFeign 10.x+**：社区接管，引入 `RequestInterceptor` 增强，支持 Spring MVC 注解（通过 `SpringMvcContract`）
- **Spring Cloud OpenFeign**：将 OpenFeign 封装为 Spring Boot Starter，自动配置，集成 Ribbon → Spring Cloud LoadBalancer，Hystrix → Resilience4j

# 5. 与其他 HTTP 客户端对比

| 维度 | OpenFeign | RestTemplate | WebClient | Apache HttpClient | OkHttp |
|------|-----------|--------------|-----------|-------------------|--------|
| **编程模型** | 声明式接口 | 命令式模板 | 响应式/命令式 | 命令式底层 | 命令式底层 |
| **Spring 集成** | 深度融合 | 原生支持 | 原生支持 | 需手动集成 | 需手动集成 |
| **负载均衡** | 内置支持 | @LoadBalanced | @LoadBalanced | 无 | 无 |
| **熔断** | 集成 Resilience4j | 集成 Resilience4j | 集成 Resilience4j | 手动 | 手动 |
| **响应式** | 不原生支持 | 不原生支持 | 原生支持 | 无 | 无 |
| **连接池** | 可配置（底层换 HttpClient） | 可配置 | Reactor Netty | 内置 | 内置 |

# 6. 架构全景

```
┌─────────────────────────────────────────────────────────────────┐
│                      业务代码 (@FeignClient 接口)                  │
├─────────────────────────────────────────────────────────────────┤
│  JDK 动态代理 (ReflectiveFeign)                                   │
│       │                                                          │
│       ▼                                                          │
│  Contract                    ← 解析注解 → MethodMetadata           │
│  (SpringMvcContract)                                             │
│       │                                                          │
│       ▼                                                          │
│  MethodHandler               ← 封装一次调用                        │
│  (SynchronousMethodHandler)                                      │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────┬──────────┬──────────┬──────────┐                   │
│  │Interceptor│ Encoder  │  Client  │ Decoder  │                   │
│  │ (请求拦截) │ (编码请求体) │ (执行HTTP) │ (解码响应体) │                   │
│  └──────────┴──────────┴──────────┴──────────┘                   │
│                     │                                             │
│                     ▼                                             │
│  ┌─────────────────────────────────────────┐                     │
│  │ Spring Cloud LoadBalancer / 服务发现      │                     │
│  └─────────────────────────────────────────┘                     │
│                     │                                             │
│                     ▼                                             │
│  ┌─────────────────────────────────────────┐                     │
│  │ 底层 HTTP 引擎 (HttpURLConnection /      │                     │
│  │ Apache HttpClient / OkHttp)              │                     │
│  └─────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

# 7. 核心接口一览

| 接口 | 作用 | 默认实现 |
|------|------|----------|
| `Contract` | 解析接口方法注解，生成 `MethodMetadata` | `SpringMvcContract`（Spring Cloud） |
| `Encoder` | 将请求体对象编码为 HTTP body | `SpringEncoder` |
| `Decoder` | 将 HTTP 响应体解码为对象 | `SpringDecoder` |
| `Client` | 执行 HTTP 请求 | `Client.Default`（JRE），可替换为 Apache/OkHttp |
| `RequestInterceptor` | 请求发送前拦截，修改 RequestTemplate | 无默认，需自定义 |
| `ErrorDecoder` | 将 HTTP 错误响应解码为异常 | `ErrorDecoder.Default` |
| `Retryer` | 控制重试策略 | `Retryer.NEVER_RETRY`（默认不重试） |
| `Logger` | 日志记录 | `Slf4jLogger` |

# 8. 依赖关系

```
spring-cloud-starter-openfeign
    ├── spring-cloud-openfeign-core
    │       ├── io.github.openfeign:feign-core
    │       ├── io.github.openfeign:feign-httpclient (可选)
    │       └── io.github.openfeign:feign-okhttp (可选)
    ├── spring-cloud-starter-loadbalancer (Hoxton.SR1+ 默认，替代 Ribbon)
    └── (可选) spring-cloud-starter-circuitbreaker-resilience4j
```
