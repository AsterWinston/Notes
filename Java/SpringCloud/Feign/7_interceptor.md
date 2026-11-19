# 1. 概述

## 1.1 定位：请求发出前最后的统一改写点

`RequestInterceptor` 是 Feign 提供的**请求前置拦截**接口，在 `Encoder` 执行前、HTTP 请求发送前被调用，用于对 `RequestTemplate` 进行**统一修改**。

```java
public interface RequestInterceptor {
    void apply(RequestTemplate template);
}
```

它工作在"模板态"而非"报文态"：拿到的是还没序列化的 `RequestTemplate`，改 header、追加 query、动 URL 片段都便宜，但拿不到响应（那是 `Decoder`/`ErrorDecoder` 的领域），也拿不到方法参数原值（参数此时已按 Contract 规则展开进模板）。这个边界决定了拦截器的适用面：**跨请求的横切元信息**（认证、追踪、灰度、签名）归它，业务参数级的加工应回到 Contract/Encoder 层。

# 2. 典型使用场景

| 场景 | 示例 |
|------|------|
| **认证 Token 传递** | 从当前上下文获取 Token，写入 `Authorization` 头 |
| **分布式链路追踪** | 写入 TraceId / SpanId（如 `X-B3-TraceId`） |
| **版本标识** | 写入 API 版本号（`X-Api-Version`） |
| **灰度标记** | 写入灰度染色标记（`X-Gray`） |
| **通用请求头** | 写入 User-Agent、来源服务名等 |
| **请求签名** | 对请求参数签名，写入签名头 |

签名场景值得单独注意：`apply` 时 body 尚未由 Encoder 写入（§8 源码里 `interceptor.apply` 在 `encoder.encode` **之前**），对 body 做签名要改为在拦截器里自己序列化一份，或干脆用自定义 `Contract`/`Capability` 在编码后再签，否则会出现"签名基于空 body"的隐蔽错误。

# 3. 基础示例：Token 传递

```java
@Component
public class AuthRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        String token = getCurrentToken();
        if (StringUtils.hasText(token)) {
            template.header("Authorization", "Bearer " + token);
        }
    }

    private String getCurrentToken() {
        // 从 SecurityContext / ThreadLocal / 请求属性中获取
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth instanceof JwtAuthenticationToken) {
            return ((JwtAuthenticationToken) auth).getToken().getTokenValue();
        }
        return null;
    }
}
```

`template.header(name, value)` 的语义是**追加**：对已存在同名 header 的模板再调一次，会得到两个值（下游按多值解析）。"有则覆盖、无则默认"的写法是先 `template.removeHeader(name)` 再 set，或者用 `!template.headers().containsKey(name)` 做守卫——鉴权头重复是这类拦截器最常见的事故形态。

# 4. 分布式链路追踪示例

```java
@Component
public class TraceRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // Sleuth / Micrometer Tracing 自动注入
        // 这里演示手动添加 TraceId 到 Header
        Span currentSpan = tracer.currentSpan();
        if (currentSpan != null) {
            template.header("X-B3-TraceId", currentSpan.context().traceId());
            template.header("X-B3-SpanId", currentSpan.context().spanId());
        }
    }
}
```

生产项目不要手写这个类：引入 Micrometer Tracing（Boot 3 之前的 Sleuth）后，Feign 的追踪注入由框架自动完成（它还负责跨线程传播上下文），手写版会与之重复注入造成头翻倍。保留此示例的意义在于说明：凡是"框架已有自动版本"的拦截器场景（追踪、租户透传、安全头），先确认 starter 是否已内置，再造轮子。

# 5. 上下文传递（ThreadLocal）

```java
public class UserContext {
    private static final ThreadLocal<Map<String, String>> CONTEXT = new ThreadLocal<>();

    public static void set(String key, String value) {
        CONTEXT.get().put(key, value);
    }

    public static String get(String key) {
        Map<String, String> map = CONTEXT.get();
        return map != null ? map.get(key) : null;
    }

    public static void clear() {
        CONTEXT.remove();
    }
}
```

```java
@Component
public class ContextRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        String userId = UserContext.get("userId");
        if (userId != null) {
            template.header("X-User-Id", userId);
        }
        String tenantId = UserContext.get("tenantId");
        if (tenantId != null) {
            template.header("X-Tenant-Id", tenantId);
        }
    }
}
```

在拦截器/过滤器中设置上下文：

```java
@Component
public class UserContextFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        try {
            UserContext.set("userId", request.getHeader("X-User-Id"));
            UserContext.set("tenantId", request.getHeader("X-Tenant-Id"));
            chain.doFilter(req, res);
        } finally {
            UserContext.clear();
        }
    }
}
```

## 5.1 这套写法的两个失效前提

`UserContext.set` 直接 `CONTEXT.get().put(...)` 在 ThreadLocal 未初始化时 NPE，示例省略了 `withInitial`；更重要的是**线程边界**：拦截器在"发起 Feign 调用的线程"上执行，一旦调用链进入 `@Async`、`CompletableFuture.supplyAsync`、虚拟线程或消息消费线程，ThreadLocal 上下文就断了，表现为"header 偶发缺失且无法复现"。对策三选一：`InheritableThreadLocal`/`ScopedValue`（只覆盖新建子线程）、任务提交处用 `TaskDecorator`/装饰器搬运上下文、或最稳妥——把需要透传的信息显式作为方法参数传递，让上下文不依赖线程。

# 6. 配置注册 Interceptor

## 6.1 方式 1：@Component 全局生效

```java
@Component
public class AuthRequestInterceptor implements RequestInterceptor {
    // 被 @ComponentScan 扫描到的所有 FeignClient 都会使用此拦截器
}
```

## 6.2 方式 2：针对特定 FeignClient

```java
@FeignClient(name = "user-service",
             configuration = UserFeignConfig.class)
public interface UserClient { }
```

```java
public class UserFeignConfig {
    @Bean
    public RequestInterceptor userAuthInterceptor() {
        return template -> template.header("X-Service-Key", "user-key-123");
    }
}
```

## 6.3 方式 3：YAML 全局配置

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            request-interceptors:
              - com.example.feign.AuthRequestInterceptor
              - com.example.feign.TraceRequestInterceptor
```

yml 方式的拦截器按**无参构造反射实例化**，不能注入任何协作 Bean——需要依赖的版本只能用 §6.1/§6.2；它的定位是"零代码开关"，团队里应约定同一类拦截器只走一种注册渠道，避免同一逻辑 Bean + yml 各注册一次而重复执行。

## 6.4 方式 4：Feign.Builder 手动注入

```java
@Bean
public UserClient userClient() {
    return Feign.builder()
            .requestInterceptor(new AuthRequestInterceptor())
            .target(UserClient.class, "http://localhost:8080");
}
```

这是脱离 Spring 的原生用法（`url` 直连、无负载均衡），适合测试桩或极简集成；在 Spring Cloud 工程里混用会产生"同一个接口两种代理、行为不一致"的分裂状态，仅限显式的特殊场景。

# 7. 多个拦截器的执行顺序

```text
RequestTemplate 创建
    │
    ├── RequestInterceptor A  (先注册的先执行)
    ├── RequestInterceptor B
    ├── RequestInterceptor C
    │
    ▼
Encoder.encode()  (请求体在此之后编码)
    │
    ▼
Client.execute()  → HTTP
```

**控制顺序**：
- `@Component` + `@Order` 控制全局拦截器顺序
- 特定 `@FeignClient(configuration=...)` 中注册的拦截器在全局之后执行

```java
@Component
@Order(1)
public class TraceInterceptor implements RequestInterceptor { }

@Component
@Order(2)
public class AuthInterceptor implements RequestInterceptor { }
```

## 7.1 顺序机制的真实依据

全局与专属拦截器最终会被**合并成同一个列表**（客户端上下文解析时把主上下文的 Bean 一并纳入），列表按 `@Order`/`Ordered` 排序，没写 Order 的排最后——"configuration 注册在全局之后"只是"专属拦截器通常没写 @Order"的表象，不是保证。有顺序依赖时请显式给每个拦截器标 `@Order`，或者把互相依赖的逻辑合并进同一个 Bean 内部按序执行；另外两个拦截器改写同一 header 时，**后执行者看到的已是前者改写后的模板**，追加语义会叠加、覆盖语义以最后一个为准。

# 8. 源码流程

```text
SynchronousMethodHandler.invoke(args)
  │
  ├── 构建 RequestTemplate
  │     template = buildTemplateFromArgs.create(argv)
  │
  ├── 遍历所有 RequestInterceptor
  │     for (RequestInterceptor interceptor : requestInterceptors) {
  │         interceptor.apply(requestTemplate);
  │     }
  │
  ├── encoder.encode(body, bodyType, requestTemplate)
  │
  └── client.execute(request, options)
```

关键源码位置（`SynchronousMethodHandler`）：

```java
RequestTemplate template = buildTemplateFromArgs.create(argv);
Retryer retryer = this.retryer.clone();
while (true) {
    try {
        return executeAndDecode(template, options);
    } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
    }
}
```

其中 `executeAndDecode` 会在内部调用 `targetRequest(template)`，此时遍历 Interceptor：

```java
Request targetRequest(RequestTemplate template) {
    for (RequestInterceptor interceptor : requestInterceptors) {
        interceptor.apply(template);
    }
    return target.apply(template);
}
```

## 8.1 重试循环里的拦截器：每次尝试都会重新执行

对照两段源码能发现一个图上没画出来的事实：`targetRequest` 在 `executeAndDecode` **内部**，而后者位于 `while(true)` 重试循环里——拦截器对每次重试都会再跑一遍，且作用于同一个模板实例（模板只在循环外构建一次）。推论：`apply` 必须幂等。用"追加"语义写重试次数、时间戳这类每次应刷新的值没问题（同名头会越追越多才是灾难），但"append 一段路径/query"的写法在多轮重试后会产生 `?a=1&a=1&a=1` 的畸形请求。签名类逻辑还应注意第二轮重试时时间戳/nonce 已变化，需要整体重算而不是只改时间戳字段。

# 9. 注意事项

1. **不要阻塞 I/O** — Interceptor.apply() 在请求主路径上执行，避免网络调用
2. **注意线程安全** — Interceptor 是单例，不要持有线程不安全的可变状态
3. **FeignClient 级别隔离** — 如果某个 Interceptor 只用于特定 Client，通过 `configuration` 注入，不要用 `@Component`
4. **RequestTemplate 修改 Header 而非直接修改 Request** — 此时 HTTP 请求尚未构建，改 `template.header()` 即可
5. **配合 `@Headers` 使用** — 静态 Header 用 `@Headers`，动态 Header 用 `RequestInterceptor`
6. **重试场景验证幂等** — 拦截器每次尝试都会执行（§8.1），追加式改写要先去重
7. **跨线程调用前确认上下文可得** — ThreadLocal 型拦截器在异步/池化线程上会静默丢头（§5.1），上线前用一条全链路日志验证 `X-User-Id` 是否真的带出去了
