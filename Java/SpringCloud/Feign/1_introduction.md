# 1. 概述

## 1.1 Feign 是什么

Feign 是一个**声明式 HTTP 客户端**，受 Retrofit、JAX-RS 2.0 和 WebSocket 的启发。它的核心理念是：**通过编写 Java 接口并添加注解即可定义 HTTP 客户端，无需手动构造请求**。

一次 `SphU` 式的命令调用被收敛成一次普通的接口方法调用：URL 拼接、序列化、异常映射、负载均衡这些"HTTP 细节"全部下沉到框架内部，业务代码只面对强类型的方法签名。这也是 Feign 在微服务调用侧成为事实标准的原因——它把"远程调用"伪装成了"本地调用"。

## 1.2 Feign、OpenFeign、Spring Cloud OpenFeign 的关系

OpenFeign 是 Feign 的开源社区维护版本（Feign 原由 Netflix 开发，后停止维护），Spring Cloud OpenFeign 则是在 OpenFeign 基础上与 Spring 生态（Spring MVC 注解、Ribbon/LoadBalancer、服务发现等）深度集成的产物。

三个名字经常混用，实际分工差异很大，排查问题前必须先分清自己在和哪一层打交道：

| 层次 | 产物 | 职责 | 独立可用性 |
|------|------|------|------------|
| feign-core | `io.github.openfeign:feign-core` | 纯 Java 库：代理生成、Contract 解析、编解码、重试、日志 | 可脱离 Spring 独立使用 |
| OpenFeign 注解包 | feign-jaxrs2 / feign-gson 等 | 各种 Contract 的实现和编解码扩展 | 按需引入 |
| Spring Cloud OpenFeign | `spring-cloud-starter-openfeign` | `@FeignClient` 扫描注册、`SpringMvcContract`、自动配置、与 LoadBalancer/服务发现/Resilience4j 集成 | 依赖前两者 |

本文系列讨论的均指第三层（Spring Cloud 环境下的 OpenFeign），源码版本基线为 Spring Cloud OpenFeign 4.x / OpenFeign 12.x 附近。

# 2. 声明式调用 vs 命令式调用

## 2.1 代码形态对比

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

对比的实质是**契约位置**的迁移：命令式调用把 URL、方法、参数散落在每个业务方法体内；Feign 把它们集中到一个接口上，接口即服务契约。调用点从"字符串 + 模板方法"变成"类型化方法"，拼错路径、传错参数的问题在编译期就暴露。

## 2.2 声明式的收益边界

声明式并非在所有场景都是加分项，以下几类命令式/响应式客户端反而更合适：

- **URL 在运行时才知道**（代理转发、动态网关地址）：接口签名无法预先声明；
- **流式与大文件传输**（SSE、分块上传下载）：Feign 的模型是"一次请求一次完整响应"；
- **响应式栈**：Feign 是同步阻塞模型，WebFlux 项目应优先 WebClient（Feign 的 reactive 支持不在官方主线）；
- **精细控制连接生命周期**的底层调用（自定义 socket 行为、双向 TLS 特殊处理）。

在 Spring Cloud 常规"服务 A 同步调用服务 B"的主场景里，Feign 的抽象收益远大于控制损失，因此成为默认选择。

# 3. 核心优势

| 特性 | 说明 |
|------|------|
| **声明式** | 注解驱动，代码即文档，接口即契约 |
| **可插拔** | Encoder、Decoder、Client、Contract 等均可替换 |
| **集成性强** | 无缝集成 Spring MVC 注解、负载均衡、服务发现、熔断 |
| **HTTP 抽象** | 屏蔽底层 HTTP 通信细节（HttpURLConnection / Apache HttpClient / OkHttp） |
| **拦截器链** | 请求/响应拦截，支持鉴权、日志、追踪等横切逻辑 |

其中"可插拔"是 Feign 工程价值的核心：整条调用管线被拆成若干接口化的插槽，每个插槽都有默认实现，又都可以在不影响其他环节的情况下单独替换。本系列中 `5_contract`、`6_encoder-decoder`、`7_interceptor`、`19_http-client-engines` 各篇就是沿这条插槽清单展开的。

# 4. 发展历史

```text
Netflix Feign (9.x, 停止维护)
    → OpenFeign (9.x → 10.x → 11.x → 12.x+)
        → Spring Cloud OpenFeign (2.x → 3.x → 4.x)
```

- **Netflix Feign 9.x**：原始版本，仅支持 Feign 原生注解
- **OpenFeign 10.x+**：社区接管，引入 `RequestInterceptor` 增强，支持 Spring MVC 注解（通过 `SpringMvcContract`）
- **Spring Cloud OpenFeign**：将 OpenFeign 封装为 Spring Boot Starter，自动配置，集成 Ribbon → Spring Cloud LoadBalancer，Hystrix → Resilience4j

落到依赖坐标上的对应关系：Spring Cloud 2020/2021 对应 OpenFeign 10.x/11.x；2022（Spring Cloud OpenFeign 4.0，随 Spring Boot 3）升级到 12.x 并完成了 javax → jakarta 命名空间迁移。老项目升级时最常见的两类破坏性变更就是命名空间与 `spring-cloud-starter-openfeign` 的包路径，细节参见 `25_best-practices.md` 的升级注意。

# 5. 与其他 HTTP 客户端对比

| 维度 | OpenFeign | RestTemplate | WebClient | Apache HttpClient | OkHttp |
|------|-----------|--------------|-----------|-------------------|--------|
| **编程模型** | 声明式接口 | 命令式模板 | 响应式/命令式 | 命令式底层 | 命令式底层 |
| **Spring 集成** | 深度融合 | 原生支持 | 原生支持 | 需手动集成 | 需手动集成 |
| **负载均衡** | 内置支持 | @LoadBalanced | @LoadBalanced | 无 | 无 |
| **熔断** | 集成 Resilience4j | 集成 Resilience4j | 集成 Resilience4j | 手动 | 手动 |
| **响应式** | 不原生支持 | 不原生支持 | 原生支持 | 无 | 无 |
| **连接池** | 可配置（底层换 HttpClient） | 可配置 | Reactor Netty | 内置 | 内置 |

选型判断可以压缩成三问：要不要服务发现与负载均衡（是 → Feign/RestTemplate）；要不要响应式（是 → WebClient）；要不要极致控制底层 HTTP 行为（是 → 直接用 HttpClient/OkHttp，或者保留 Feign 的接口层、把底层 `Client` 换成它们的实现——这正是 Feign "可插拔"给出的第三条路）。

# 6. 架构全景

```text
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

这张图按"生命周期阶段"读最省力，可分为三段：

1. **启动期（构建）**：Contract 把接口注解解析成 `MethodMetadata`，为每个方法生成一个 `MethodHandler`——注解解析只做一次，运行期不再重复；
2. **调用期（执行）**：一次方法调用 = 拦截器改写模板 → Encoder 编码 → Client 执行 HTTP → Decoder/ErrorDecoder 出参或异常；
3. **支撑期（外部依赖）**：Client 之下挂着两件事——"调哪台机器"（LoadBalancer/服务发现）和"怎么走网络"（底层 HTTP 引擎）。

后者的分层也解释了 `19_http-client-engines.md` 的立场：换 HTTP 引擎不动上面任何一层。

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

两个与直觉不符的默认值值得先记住：**默认不重试**（`Retryer.NEVER_RETRY`，OpenFeign 层的重试与熔断层的重试是两套独立机制，容易配置重复）；**默认日志级别 NONE**（配了 `Logger.Level.FULL` 还需把对应包设为 DEBUG 才会输出，见 `10_logging.md`）。

# 8. 依赖关系

```text
spring-cloud-starter-openfeign
    ├── spring-cloud-openfeign-core
    │       ├── io.github.openfeign:feign-core
    │       ├── io.github.openfeign:feign-httpclient (可选)
    │       └── io.github.openfeign:feign-okhttp (可选)
    ├── spring-cloud-starter-loadbalancer (Hoxton.SR1+ 默认，替代 Ribbon)
    └── (可选) spring-cloud-starter-circuitbreaker-resilience4j
```

依赖图里有三个"可选"，对应三种常见故障：不引入 LoadBalancer，`name = "user-service"` 无法解析成真实主机（表现为 UnknownHost）；不引入 feign-httpclient/feign-okhttp，就退回默认的 `Client.Default`（`HttpURLConnection`，无连接池复用细节差异）；不引入 resilience4j 且未开 `circuitbreaker.enabled`，`fallback` 配置会被静默忽略，降级完全不生效——这三条都属于"接口能编译、运行才暴露"的坑，详见 `25_best-practices.md`。

# 9. 系列阅读地图

本篇解决"是什么、为什么、由哪些部件构成"；后续按部件展开：注解与契约（`3_annotations`、`5_contract`）、配置体系（`4_configuration`）、编解码与拦截器（`6`、`7`）、错误/重试/日志/超时（`8`~`11`）、与 LoadBalancer/熔断/服务发现的集成（`13`~`16`）、专项能力（`17`~`20`），以及四篇源码深潜（`21`~`24`，动态代理 → Contract/MethodHandler → 请求管线 → LoadBalancer 集成）。
