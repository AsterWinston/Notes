# 1. 概述

Sentinel 作为 Spring Cloud Alibaba 的核心组件，通过 `spring-cloud-starter-alibaba-sentinel` 提供**零配置集成**。主要能力包括：

- **自动配置**：自动启用 Sentinel 拦截器
- **注解支持**：`@SentinelResource` 透明拦截
- **RestTemplate 支持**：`@SentinelRestTemplate` 为 HTTP 调用添加防护
- **动态数据源**：与 Nacos/Apollo 等配置中心集成
- **Actuator 端点**：暴露 Sentinel 运行指标

## 1.1 "零配置"的真实覆盖面

starter 自动装配的范围要划清：它管**Web 入口资源、注解切面、RestTemplate/Feign/WebClient 适配、数据源接线、Dashboard 心跳**；不管**Dubbo/gRPC/Sentinel 集群 token server**（各需自己的 adapter 依赖，§2 表外）。且"零配置"≠"零认知"——集成层替你决定了资源命名、context 归一、被拦响应形态这三件你迟早要排查的事，本篇的主体就是这三件事。

# 2. 依赖与自动配置

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

**自动配置类**：

| 配置类 | 职责 |
|--------|------|
| `SentinelWebAutoConfiguration` | 注册 SentinelWebInterceptor，拦截所有 Spring MVC 请求 |
| `SentinelWebFluxAutoConfiguration` | WebFlux 环境的 Sentinel 集成 |
| `SentinelFeignAutoConfiguration` | Feign 集成 |
| `SentinelAutoConfiguration` | 核心 Bean 初始化 |
| `SentinelEndpointAutoConfiguration` | Actuator 端点暴露 |

## 2.1 表要按版本纪元拆成两代

Boot 2.x（SCA 2021.x，javax 系）与 Boot 3.x（SCA 2022/2023，jakarta 系）的 Web 集成**不是同一套实现**：老一代 `SentinelWebAutoConfiguration` 注册的是 **Servlet Filter**（`CommonServletFilter`，来自 sentinel-web-servlet adapter，资源名=原始 URI）；Boot 3 一代换成了 **Spring MVC 拦截器**（`SentinelWebInterceptor`，来自 sentinel-spring-webmvc-v6x-adapter，能拿到 HandlerMapping 的匹配模板）。这个代际差异直接决定 §3 的资源名质量——排查"资源名为什么长这样"先确认自己在哪一代。其余三行跨代稳定；`SentinelAutoConfiguration` 的核心产出是 `SentinelResourceAspect` 等 Bean 接线与 `project.name` 设定（应用名兜底链：`spring.application.name` → 未设则 `csp.sentinel.app.name`，都没有时 Dashboard 里显示成令人困惑的默认名，容器环境务必显式设）。

# 3. 拦截机制

## 3.1 Spring MVC 自动拦截

`SentinelWebInterceptor` 自动将所有 Spring MVC 请求作为 Sentinel 资源保护：

```text
HTTP 请求 → DispatcherServlet
  → SentinelWebInterceptor.preHandle()
    → SphU.entry("GET:/api/user/{id}")
      → SlotChain 执行
        → 通过 → Controller 执行
        → 拒绝 → SentinelBlockException → 返回 429
```

**资源名生成规则**：`{HTTP_METHOD}:{URL_PATTERN}`

```java
// 自动生成资源名示例
GET:/api/user/{id}
POST:/api/order
DELETE:/api/product/{id}
```

## 3.2 【纠偏】默认资源名不带方法前缀；模板名是新一代的红利

两处校准。**其一**：`GET:` 前缀并非默认——由 `spring.cloud.sentinel.http-method-specify` 控制，**默认 false**，即默认资源名就是路径 `/api/user/{id}`；开前缀的动机是"同 URL 不同方法分开限流"（GET 查询与 DELETE 删除共享一名会互相误伤），有此需求就全局开，别事后改名（改了等于换资源，旧规则全部失效）。**其二**（好消息，Boot3 一代）：`SentinelWebInterceptor` 的资源名直接取 **HandlerMapping 匹配到的路径模板**——`/api/user/{id}` 天然收敛，RESTful 数字 ID **不再需要** §3.3 的 UrlCleaner 兜底；而老一代 Filter 按原始 URI 记名（`/api/user/1`、`/api/user/2` 各成一个资源），才必须靠 cleaner 或路径正则归一。两代迁移时资源名会变（`/api/user/{id}` 从"cleaner 改写结果"变成"框架原生模板"），规则跟着名字走，灰度期双名并配。

## 3.3 自定义资源名

```java
@Component
public class MyUrlCleaner implements UrlCleaner {
    @Override
    public String clean(String originUrl) {
        // 统一处理 Restful 风格的 URL
        // /api/user/1 → /api/user/{id}
        // /api/user/2 → /api/user/{id}
        return originUrl.replaceAll("/\\d+", "/{id}")
                        .replaceAll("/[a-f0-9]{32}", "/{uuid}");
    }
}
```

## 3.4 cleaner 的适用面收缩与残余价值

新一代里 cleaner 从"必选"退居"补刀"：模板外的动态段（`/files/{yyyyMMddHHmmss}` 这种不是路由变量的、正则通配路由、遗留系统不走 HandlerMapping 匹配的转发链）仍会漏出原始值，cleaner 是唯一兜底；老一代则照旧全量依赖。两个共性纪律：**cleaner 每请求执行**，正则预编译、别在里面做 IO；**它同时喂 context 名与资源名的清洗**——链路模式（chain）规则的 refResource 对得上清洗后的名字才生效（`8_hotspot_param.md` §4.3 的 web-sync 是另一个开关，别混）。

## 3.5 Context 与 origin 的集成层默认值

Web 适配器每请求 `ContextUtil.enter(contextName, origin)`：context 名 = 归一化后的请求路径（剥 servlet context-path），所以"链路流控的 refResource 填入口 URL"在集成场景就是这条规则（`4_flow_control.md` §3.4）；origin 走 `RequestOriginParser`，**SCA 默认实现读 `x-sentinel-request-origin` 头、无则空串**（`7_authority_rule.md` §2.2 通道②的具体行为）。origin 为空导致来源类规则失明的概率在集成层最高——多数调用方根本不会主动带这个头，要么网关统一注入，要么换成读 `服务名头` 的自定义 parser。异步/错误派发注意：interceptor 形态在 `afterCompletion` 收尾 exit，遇到业务自管的线程切换仍要按 `2_architecture.md` §5.2 处理 Context。

# 4. @SentinelResource 详解

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @GetMapping("/user/{id}")
    @SentinelResource(
        value = "getUser",                     // 资源名
        entryType = EntryType.IN,             // 入口类型（IN=进入/OUT=出口）
        blockHandler = "handleBlock",          // 流控/降级处理方法
        blockHandlerClass = BlockHandler.class,// 指定 blockHandler 所在类
        fallback = "handleFallback",           // 业务异常处理方法
        fallbackClass = FallbackHandler.class, // 指定 fallback 所在类
        defaultFallback = "handleDefault"      // 默认 fallback（单个 Throwable 参数）
    )
    public User getUser(@PathVariable Long id) {
        return userService.getById(id);
    }
}
```

> 注：原稿此处示例写的是 `entryType = EntryType.OUT`，对一个 Controller 入口资源是错误示范（OUT 会把该资源从系统保护与入口统计口径中摘除，§4.3），已改为 IN；属性注释保留原文。

## 4.1 blockHandler vs fallback

| 维度 | blockHandler | fallback |
|------|-------------|----------|
| **触发条件** | BlockException（流控/熔断/系统保护） | 业务异常（Throwable） |
| **参数签名** | 原方法参数 + BlockException | 原方法参数 + Throwable |
| **目的** | 处理流量管控拒绝 | 处理业务异常兜底 |
| **是否必须** | 否（不指定则抛 BlockException） | 否（不指定则抛原异常） |

## 4.2 分离 blockHandler 类

```java
public class UserBlockHandler {

    // 方法必须为 static
    public static User handleGetUserBlock(Long id, BlockException ex) {
        log.warn("User query blocked: id={}", id);
        return new User(id, "Blocked", 0);
    }

    public static User handleCreateUserBlock(User user, BlockException ex) {
        log.warn("User creation blocked");
        return null;
    }
}

@SentinelResource(value = "getUser",
    blockHandler = "handleGetUserBlock",
    blockHandlerClass = UserBlockHandler.class)
public User getUser(Long id) { ... }
```

## 4.3 切面实现的三条硬限制

注解生效依赖 Spring AOP，签名规则细节（`3_quick_start.md` §4.4）之外，实现层三条会真实咬人：**① 同类自调用不拦截**（代理语义，内部 `this.method()` 绕过切面——把被调方法拆到另一 Bean 或改用 API 式）；**② private/final 方法不可用**（CGLIB/JDK 代理的通用限制）；**③ 属性名写错不报错**——`blockHandler = "handleBolck"` 拼错后按"未配置"降级为抛原异常，静默到线上第一次限流才暴露，**每条降级方法上线前手动触发一次 Block**（构造超阈值流量或临时规则 count=0）。entryType 的语义账：IN 才参与系统保护与入口统计口径（`6_system_protection.md` §1.1 的 OUT 短路就在消费这个标记），给"出向调用"标 IN 会把出站流量掺进 machine-root 入口统计，系统规则的 qps/avgRt 读数随之失真。

# 5. @SentinelRestTemplate

为 RestTemplate 添加 Sentinel 保护：

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @SentinelRestTemplate(
        blockHandler = "handleBlock",
        blockHandlerClass = RestBlockHandler.class,
        fallback = "handleFallback",
        fallbackClass = RestFallbackHandler.class
    )
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

public class RestBlockHandler {
    public static SentinelClientHttpResponse handleBlock(
            HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution, BlockException ex) {
        return new SentinelClientHttpResponse("Blocked by Sentinel: " + ex.getMessage());
    }
}
```

## 5.1 RestTemplate 保护的四条边界

**① 粒度是 Bean**：注解标在 @Bean 方法上、生效于该 RestTemplate 实例的全部请求——没走这个 Bean 的调用（new 出来的、别的库自己的 HTTP 客户端）零保护；多模板实例各自加注解，且**不要求 `spring.cloud.sentinel.filter.enabled`**，与被拦的 Web 入口是两个方向。**② 资源名**：形如 `POST:http://host/path`（方法+URL，host 为字面域名）——同一外部服务的多路径是多个资源，规则按 host 前缀批量管理时注意没有通配能力，高基数 URL（带 path 变量）会资源爆炸，给这类调用手动包 `SphU.entry` 收敛资源名更稳。**③ 出向调用记 OUT**（§4.3 口径的另一半）。**④ 被拦的返回形态**：`SentinelClientHttpResponse` 是一个**真实的 200 响应壳**（示例文案里塞了 Blocked 字样但状态码默认 200）——调用方 `getForObject` 会拿到这串文本继续跑，务必在 blockHandler 里给可判别的错误结构（或自定义状态码），这比 Feign 路线抛异常更容易被吞掉故障。WebClient 有对应的 `@SentinelWebClient`/操作符方案，Feign 走的是完全另一条链路（`13_feign_integration.md`）。

# 6. Actuator 端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: sentinel
  endpoint:
    sentinel:
      enabled: true
```

```bash
# 查看 Sentinel 状态
curl http://localhost:8080/actuator/sentinel
```

## 6.1 端点内容与指标的正确来源

`/actuator/sentinel` 是**状态摘要**（版本、appName、若干开关），不是规则/指标接口——别指望它替代 Command Center `/metrics`（`9_dashboard.md` §8.1）或 Dashboard。真正要 Prometheus 抓的曲线走 **feign-micrometer 同款体系之外的 sentinel-core 扩展**：社区方案是注册 `SentinelMetricExtension`/自定义 Slot 打点，或直接周期读 Command Center metrics 转 exporter（`21_logging_monitoring.md` 展开）；actuator 在本集成里的真实贡献是把 `/actuator/health` 等纳入既有监控通道，与 Sentinel 数据是"同管道不同水源"。

# 7. 全局异常处理

```java
@RestControllerAdvice
public class SentinelExceptionHandler {

    @ExceptionHandler(BlockException.class)
    @ResponseStatus(HttpStatus.TOO_MANY_REQUESTS)  // 429
    public Result handleBlockException(BlockException ex) {
        return Result.error(429, "请求过于频繁，请稍后再试");
    }

    @ExceptionHandler(DegradeException.class)
    @ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)  // 503
    public Result handleDegradeException(DegradeException ex) {
        return Result.error(503, "服务暂时不可用");
    }
}
```

## 7.1 被拦异常有两条响应通道，这个 Advice 只盖住一条

Web 入口被限的请求**根本到不了 DispatcherServlet**（Filter/Interceptor 阶段就被拒），异常走的是适配器的 `BlockExceptionHandler` 通道（Bean 形式覆盖，如 `SentinelWebFluxAutoConfiguration` 同款注册；默认实现统一 429 + "Blocked by Sentinel (flow limiting)" 文本——**所有 Block 家族同码**，含 AuthorityException，想给授权 403/降级 503 的区分语义就在这条通道写分支）。§7 代码只接得住**方法内部**抛出的 BlockException（`@SentinelResource` 没配 blockHandler、或 blockHandler 自己再抛）。配了"全局处理器"却对外 429 文案不一致的排查顺序：先分清异常来自哪条通道，再确认改的是对应那条；Feign 调用被熔断的第三方视角另有其路（`13_feign_integration.md`）。

# 8. 最佳实践

1. **资源命名规范**：`{服务名}:{接口}:{方法}`，如 `userService:/api/user:getUser`
2. **blockHandler 与 fallback 分离**：使用独立类管理，保持 Controller 简洁
3. **URL 统一标准化**：通过 `UrlCleaner` 避免 Restful 参数导致资源爆炸
4. **fallback 始终有值**：降级返回应当是有意义的兜底数据，不是 null
5. **配合 Actuator**：开启 Sentinel 端点，集成 Prometheus 监控

## 8.1 第 1 条按现实改写

集成层的资源名不由你命名风格决定（Web 归适配器、Dubbo 归接口签名、RestTemplate 归 URL），强推"服务名:接口:方法"只会制造**规则名与簇点链路对不上**的第 N 类事故。可执行的版本：**Web 资源接受适配器命名**（用 http-method-specify 与 cleaner 管控粒度，§3.2/§3.4）；**注解资源承担"语义名"**（`getUser` 这种动作名，跨服务不复用即可）；评审时唯一硬标准是"规则文件里的每个 resource 都能在目标机器簇点链路找到同名节点"。第 5 条同样纠偏：Prometheus 指标不是 actuator 端点直出的（§6.1）。第 2、4 条补一手：分离类里的降级方法**别依赖实例状态**（static 拿不到注入的 Bean，缓存/开关要走静态入口或参数传入——`fallbackClass` 同款约束），返回 null 的坑在调用方拆箱 NPE，团队级统一 `Result` 包装比逐方法判断快。

## 8.2 相关阅读

Feign 接线在 `13_feign_integration.md`；Gateway 场景资源与异常形态在 `12_gateway_integration.md`；Dubbo 在 `11_dubbo_integration.md`；数据源接线在 `14_dynamic_rules.md`；资源名/Context 机制底层在 `2_architecture.md` §5；被拦响应文案的多通道案例在 `24_troubleshooting.md`。
