# 1. 接口设计

## 1.1 统一响应体包装

不要直接返回异构的服务端原始 JSON，通过自定义 Decoder 剥离统一响应体：

```java
// 服务端统一返回
{
  "code": 200,
  "message": "success",
  "data": { "id": 1, "name": "John" }
}

// ❌ 错误：Feign 接口暴露了包装结构
@GetMapping("/users/{id}")
ApiResponse<User> getUser(@PathVariable Long id);

// ✓ 正确：Decoder 剥离包装，Feign 接口只关心业务对象
@GetMapping("/users/{id}")
User getUser(@PathVariable Long id);
```

说明一点以免误读：`ApiResponse<User>` 在技术上**能**正确解码（泛型不丢，`20_response-mapping.md` §1.1），这里否掉它是**设计**理由——每个调用点被迫拆壳判 code，错误处理散落。拆壳 Decoder 的完整实现与 void/非壳回退等边界在 `20_response-mapping.md` §8 与 `6_encoder-decoder.md` §7.1，落地按那两处为准；别忘了拆壳后"业务码失败"进了熔断统计的口径问题（`14_circuit-breaker.md` §3.2）。

## 1.2 避免循环依赖

```java
// ❌ 危险：服务 A 调 B，B 调 A
// order-service → user-service → order-service

// ✓ 方案 1：消息队列解耦
// ✓ 方案 2：提取公共服务
// ✓ 方案 3：考虑非 Feign 的异步 HTTP 调用
```

循环调用依赖的微观形态要认清：Feign 代理是懒构建的（`21_source-dynamic-proxy.md` §3.1），**构造期互相注入不报错**，风险全在运行时——A 故障时请求打到 B、B 的降级/重试又回打 A，形成放大环路；更隐蔽的是**降级路径造环**：A 的 fallback 调 B、B 的 fallback 调 A，平时零流量没人测。判据与治理：依赖图上有向无环是架构审查项；确需互相感知的数据用事件同步（方案 1）而不是同步互查；每条 Feign 链路把"超时×重试×降级去向"写成一行预算，评审时看环上的乘积是否小于任一环的超时。

## 1.3 接口粒度适中

```java
// ❌ 过于细粒度（N+1 问题）
User user = userClient.getUser(userId);
Order order = orderClient.getOrder(orderId);
Product product = productClient.getProduct(order.getProductId());

// ✓ 批量查询
UserDto dto = userClient.getUserWithOrders(userId);

// ✓ 或使用 CompletableFuture 并行调用
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> userClient.getUser(userId));
CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(() -> orderClient.getOrder(orderId));
CompletableFuture.allOf(userFuture, orderFuture).join();
```

三个方案的优先级是 **批量接口 > 并行拼装 > 串行**，但并行方案的坑比收益更容易被忽视：`supplyAsync` 默认 commonPool（为 CPU 密集设计，阻塞 IO 会饿死它，务必传业务隔离线程池）；ThreadLocal 上下文（认证/traceId）在工作线程丢失，拦截器静默少带头（`7_interceptor.md` §5.1——先 TaskDecorator 搬运再上线）；并行后**总 RT = max(两腿)+开销**，预算与熔断口径都要按"一次入口消耗 N 路并发"重算。批量接口则是契约级改造：ids 上限、空集合返回、部分失败语义都要定义，别把 N+1 变成"一次 500 个 id 的慢查询"。

# 2. 配置管理

## 2.1 配置类隔离

```java
// ❌ 错误：配置类被全局扫描到
@Configuration
public class UserFeignConfig {
    @Bean
    public RequestInterceptor authInterceptor() { ... }
}

// ✓ 正确：不标记 @Configuration，单独指定
public class UserFeignConfig {
    @Bean
    public RequestInterceptor authInterceptor() { ... }
}

// 在 @FeignClient 中引用
@FeignClient(name = "user-service", configuration = UserFeignConfig.class)
```

同一条纪律还有两个变体入口要堵：`@ComponentScan` 扫到、以及把它注册进主容器的任何写法——主容器一旦有组件 Bean，会被**所有**客户端子上下文看见（`4_configuration.md` §5.1/§6.1）。CI 时代的机械防法：ArchUnit 一条规则"Feign 配置类不得带 @Configuration/@Component 且不得位于被扫描包"，比 CR 口头约定可靠。

## 2.2 超时配置原则

```text
connectTimeout < circuitBreaker.timeoutDuration < readTimeout
  (3s)        < (5s)                            < (10s)
```

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            connect-timeout: 3000    # 连接超时：3s
            read-timeout: 10000      # 读取超时：10s

resilience4j:
  timelimiter:
    configs:
      default:
        timeout-duration: 5s         # 熔断器超时：5s（介于两者之间）
```

不等式是**方向感**而不是铁律：它成立的前提是"希望超时以 TimeLimiter 的 `TimeoutException` 口径统一暴露"（`14_circuit-breaker.md` §7.1 也给了反方向选择——read-timeout 更小、让网络异常先行）。无论选哪个方向，**异常统计名单必须跟着口径配**（只配 TimeoutException 而实际先响的是 IOException，失败率统计就漏计）。还有一条更高优先级的不等式常被本图掩盖：`本层总预算(超时×重试乘积) ≤ 调用方预算 × 0.7`（`9_retry.md` §9.1）——三层都满足局部不等式、全局仍然超支的链路，故障时会在最外层统一爆雷。

## 2.3 不同环境不同配置

```yaml
# application-dev.yml
spring.cloud.openfeign.client.config.default.logger-level: FULL

# application-prod.yml
spring.cloud.openfeign.client.config.default.logger-level: BASIC
```

更灵活的组合是"双开关预置"：生产把 `logger-level` 常备 FULL、`logging.level` 停 INFO（零输出零开销），故障期经 actuator 对特定接口包热开 DEBUG——FULL 秒级生效、影响面精确（`10_logging.md` §7）。yml profile 路线的重启成本与 actuator 路线的权限收敛（/loggers 端点不能公网可达）都要在运维侧过一遍。

# 3. 异常处理

## 3.1 捕获 FeignException

```java
// ✓ 全局异常处理器
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(FeignException.class)
    public ResponseEntity<?> handleFeignException(FeignException e) {
        log.error("Feign call failed: {}", e.getMessage());
        if (e.status() == -1) {
            // 连接失败（网络不通、超时等）
            return ResponseEntity.status(503)
                    .body(Map.of("message", "Service unavailable"));
        }
        return ResponseEntity.status(e.status())
                .body(Map.of("message", "Upstream error"));
    }
}
```

示例里 `return ResponseEntity.status(e.status())` 的透传要慎用：上游 500 原样转给入口，会把本服务的告警口径污染成"我方服务端故障"；对外响应码应由**本层语义**决定（上游故障包装成本服务的 502/503 + 内部错误码，`8_error-handling.md` §6.1）。同时 `log.error` 打全量 `e.getMessage()` 在 FULL 拼接体下可能带出敏感 body，错误日志也讲脱敏。

## 3.2 ErrorDecoder 中不要吞掉异常

```java
// ❌ 错误：在 ErrorDecoder 中捕获了所有异常返回 null
public class BadErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        try {
            // ...
        } catch (Exception e) {
            return null;  // ❌ 这会导致调用方拿到 null，误以为成功
        }
    }
}

// ✓ 正确：解析失败回退到默认处理
public class GoodErrorDecoder implements ErrorDecoder {
    private final ErrorDecoder defaultDecoder = new ErrorDecoder.Default();

    @Override
    public Exception decode(String methodKey, Response response) {
        try {
            return parseBusinessError(response);
        } catch (Exception e) {
            return defaultDecoder.decode(methodKey, response);
        }
    }
}
```

回退分支有一个隐蔽前置条件：body 流**只能读一次**，`parseBusinessError` 若已消费流，Default 再读会拿空——回退实现要么先缓存字节再 `response.toBuilder().body(bytes)` 重建（`8_error-handling.md` §3.1 的纪律），要么解析逻辑本就基于缓存串。`decode` 返回 null 的后果比"误以为成功"更糟：Feign 直接 NPE 在框架内部（它要求非 null 的 Exception），栈里全是 Feign 类名更难归因。

# 4. Feign 客户端线程安全

## 4.1 Feign 客户端是线程安全的

```java
// ✓ 可以安全地在多个线程中共享同一个 Feign 客户端
@Service
public class OrderService {
    private final UserClient userClient;  // 单例，线程安全

    // 多线程环境下安全使用
    public void processOrders(List<Order> orders) {
        orders.parallelStream().forEach(order -> {
            User user = userClient.getUser(order.getUserId());  // 安全
        });
    }
}
```

线程安全的来源是**结构性的**：代理与方法处理器不可变、组件无状态、重试状态经 `Retryer.clone()` 每次调用独享（`9_retry.md` §2.1）、body 每次物化（`23_source-request-pipeline.md` §3.1）——所以"共享单例"是设计承诺而非运气。两个例外仍要盯：自定义组件里的可变字段（§4.2）与 parallelStream 的公共池问题（§1.3，阻塞 IO 用它等于拿 FJ 池赌并发），"安全"指代理本身，不指你选的线程模型。

## 4.2 Interceptor 的线程安全

```java
// ❌ 危险：拦截器持有可变状态
@Component
public class UnsafeInterceptor implements RequestInterceptor {
    private String currentToken;  // 线程不安全！

    @Override
    public void apply(RequestTemplate template) {
        template.header("Authorization", currentToken);
    }
}

// ✓ 正确：通过 ThreadLocal 或请求上下文传递
@Component
public class SafeInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String token = AuthContextHolder.getToken();  // ThreadLocal
        if (token != null) {
            template.header("Authorization", "Bearer " + token);
        }
    }
}
```

ThreadLocal 版是"相对正确"：同线程 Web 请求安全，但**池化线程上是脏的**（上一个任务的值残留——比 null 更危险的错法），`@Async`/虚拟线程/消费线程上则直接丢失（`7_interceptor.md` §5.1 的三对策）。更稳的上下文通道是 Micrometer Tracing 的 `ThreadLocalAccessor`/`TaskDecorator` 注册体系；最稳的仍是把凭证作为显式参数，让编译器替你追线程。

# 5. 性能陷阱

## 5.1 避免在循环中重复调用

```java
// ❌ 错误：N 次远程调用
List<User> users = new ArrayList<>();
for (Long id : userIds) {
    users.add(userClient.getUser(id));  // N 次 HTTP 调用
}

// ✓ 正确：批量接口
@GetMapping("/users")
List<User> getUsersByIds(@RequestParam("ids") List<Long> ids);

// ✓ 或并发调用
List<CompletableFuture<User>> futures = userIds.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> userClient.getUser(id)))
    .collect(Collectors.toList());
List<User> users = futures.stream()
    .map(CompletableFuture::join)
    .collect(Collectors.toList());
```

循环调用的伤害是三段式的：RT 线性累加（N×RT）、N 个请求的池与配额占用、以及**故障率相乘**（单次 1% 失败、循环 50 次后"至少一败"概率过半）。批量版要把契约一次定完整：ids 上限与超限行为、返回顺序与缺失 id 的语义（Map 形态比 List 更诚实）。并发版的 join 没有超时——`futures.get(总预算, MILLISECONDS)` 收口，否则一个卡死的下游把并行方案的收益全数退成线程占用（§1.3 的池与上下文三坑同样适用）。

## 5.2 不要用 Feign 传输大文件

```java
// ❌ 不适合：
@GetMapping("/files/{id}")
byte[] downloadLargeFile(@PathVariable String id);  // 100MB 全部加载到内存

// ✓ 使用 feign.Response：
@GetMapping("/files/{id}")
feign.Response download(@PathVariable String id);   // 流式读取

// ✓ 或直接用 OSS/CDN 直传
```

完整链路账在 `18_file-upload-download.md`：byte[] 的堆占用、FULL 日志的 rebuffer 放大、以及请求方向连 body 都是 `byte[]` 的结构性约束（§1.1）。一句话执行版：**文件不进 Feign，进对象存储，Feign 只递 URL**。

# 6. 日志与可观测性

## 6.1 日志敏感信息脱敏

```java
public class SensitiveDataMaskedLogger extends feign.Logger {
    @Override
    protected void log(String configKey, String format, Object... args) {
        String message = String.format(format, args);
        // 脱敏
        message = message.replaceAll("(password=\")[^\"]+", "$1****");
        message = message.replaceAll("(token=)[^& ]+", "$1****");
        log.debug(methodTag(configKey) + message);
    }
}
```

在 `log()` 一处兜住全级别是正确层级（`10_logging.md` §4.1），但正则名单永远追不上字段增长：手机号/身份证/卡号在 **body 行**（FULL 整行 JSON）里不会被这两条命中。工程解是"名单 + 结构"双保险：header 走本类正则，敏感字段多的接口直接限制 `logger-level ≤ HEADERS`；再加一条日志侧兜底（appender 层的脱敏 converter/审计扫描），日志合规不依赖每个 Logger 实现者自觉。

## 6.2 生产环境日志级别

```yaml
# 生产环境推荐
spring.cloud.openfeign.client.config.default.logger-level: BASIC
logging.level.com.example.feign: DEBUG  # 仅记录请求方法和状态码
```

两级开关缺一不可的机制在 `10_logging.md` §3.1/§7：`logger-level` 决定 Feign 产不产生日志行、`logging.level` 决定 SLF4J 放不放行，名字必须挂在**客户端接口包**（Slf4jLogger 按 targetType 命名）。§2.3 的热调模式是它的运营版。

# 7. 版本兼容性

## 7.1 接口向下兼容

```java
// ✓ 新增可选字段 → 不影响老客户端
@PostMapping("/users")
User create(@RequestBody User user);
// User 新增字段：private String nickName;  // 老客户端不传即为 null

// ✓ 新增可选参数
@GetMapping("/users")
List<User> search(@RequestParam(required = false, defaultValue = "1") Integer page);

// ❌ 删除必填字段或修改返回值结构 → 破坏兼容性
```

Feign 消费端还有一条**调用方侧的镜像纪律**：提供方新增字段，调用方的 DTO 解不解得出（FAIL_ON_UNKNOWN_PROPERTIES 默认在 Spring Boot 关着，自定义 ObjectMapper 常常没关——`20_response-mapping.md` §5 路由到裸 JacksonDecoder 时会因新字段炸老代码）；提供方删字段，调用方 DTO 的对应字段变 null 静默生效。**发布顺序规则**：提供方先行（加字段兼容老消费者），消费者跟进（依赖新字段前先观察一个发布周期）；把 DTO 双向兼容做成契约测试（consumer-driven contract 或最朴素的"老 JSON 灌新 DTO + 新 JSON 灌老 DTO"两个用例）进 CI。

## 7.2 Feign 版本与 Spring Cloud 版本对应

| Spring Cloud | Spring Boot | OpenFeign |
|-------------|-------------|-----------|
| 2023.0.x | 3.2.x | 13.x |
| 2022.0.x | 3.0.x ~ 3.1.x | 12.x |
| 2021.0.x | 2.6.x ~ 2.7.x | 11.x |
| 2020.0.x | 2.4.x ~ 2.5.x | 11.x |
| Hoxton | 2.2.x ~ 2.3.x | 10.x |

升级实操三条：**版本只由 BOM 管**，pom 里给 `feign-*`/`spring-cloud-openfeign-*` 写显式 version 是兼容性事故第一来源（starter 与手挑版本错位后，组件签名不兼容以 NoClassDefFoundError 形态在调用期才爆）；跨大版本升级清单里 Feign 侧的高频破坏点是配置键迁移（`feign.hystrix.*`→`spring.cloud.openfeign.circuitbreaker.*`、2023.0.2+ 引擎键族 `spring.cloud.openfeign.httpclient.*`→`spring.cloud.httpclientfactories.*`）与 `spring-cloud-sleuth`→Micrometer Tracing；一次只升一层（Boot→Cloud→业务组件），每层跑全量集成回归。

# 8. 常见陷阱速查表

| 陷阱 | 现象 | 解决方案 |
|------|------|----------|
| `@PathVariable` 没写 value | `IllegalStateException` | 显式指定 `@PathVariable("id")` |
| `@RequestParam` 没写 value | 编译/运行时报错 | 显式指定 `@RequestParam("name")` |
| 配置类被全局扫描 | 全局配置污染 | 去掉 `@Configuration`，通过 `configuration=` 指定 |
| `readTimeout` < 熔断器超时 | 熔断器不生效 | 熔断器超时应 < `readTimeout` |
| GET 请求带 `@RequestBody` | 默认引擎被 JDK 隐式改成 POST；其他引擎发 GET-with-body、可能被服务端/代理丢弃 body | 改用 `@RequestParam`/`@SpringQueryMap`（`22` §2.2） |
| 循环调用 | StackOverflow / 超时 | 消息队列解耦、提取公共服务 |
| 大结果集不设超时 | 线程堆积 | 设置合理 `read-timeout`，分页查询 |
| `HttpURLConnection` 默认无连接池 | 生产环境性能差 | 切换到 OkHttp 或 Apache HttpClient |
| Fallback 中调远程服务 | 雪崩 | Fallback 逻辑必须本地闭环 |
| `FeignException.status() = -1` | 连接失败 | 单独处理，开启重试 + 熔断 |
| decode404 遇网关路由缺失 | "查无此人"吞掉真 404 故障 | 与提供方确认 404 语义后再开（`8` §7） |
| Feign Retryer 与 R4j Retry 同开 | 请求量放大成乘积 | 一条链路只留一层重试（`9` §6/`13` §10） |
| 拦截器用追加语义写重试值 | 重试后 header/query 越追越多 | apply 先去重或守卫幂等（`7` §8.1） |
| 多个 Encoder/ErrorDecoder Bean | 首次调用才报 NoUniqueBean | 每客户端一类组件只注册一个（`4` §6） |

表已按勘误更新（GET+RequestBody 行的"实际发 POST"只对 JDK 默认引擎成立——`HttpURLConnection.setDoOutput(true)` 隐式转 POST 是其文档行为；Apache/OkHttp 则发出带 body 的 GET，body 常被中间件丢弃，两种都不对）。用法：故障定位时按"现象"列反查，处置完给本表补一行——它应当是团队活文档。

# 9. 上线检查清单

把全系列散落的"验收动作"收拢成一页，CR 与发布评审直接引用：

| # | 检查项 | 依据篇目 |
|---|--------|----------|
| 1 | 每个 Feign 客户端：超时三值有依据（P99/SLA），非默认 60s | `11` §9 |
| 2 | 重试预算乘积 ≤ 调用方预算 70%，注释写进行配置 | `9` §9.1、`13` §10 |
| 3 | 熔断开关链三段齐（依赖/enabled/fallback），且做过一次故障注入验证 fallback 分支 | `14` §2.1、`16` §10.5 |
| 4 | 写接口无自动重试或已配幂等键 | `9` §8.1 |
| 5 | 拦截器全链路过一遍异步线程测试（头没丢、没重复追加） | `7` §5.1/§8.1 |
| 6 | yml 与 @Bean 两入口无同组件重复注册 | `4` §6、`8` §4 |
| 7 | 生产 logger-level/logging.level 按热调模式预置，令牌类头有脱敏 | `10` §7 |
| 8 | 新接口配置键名对照当前 Boot/Cloud 版本（引擎键族迁移） | `4` §8、`25` §7.2 |
| 9 | 压缩逐跳核对：只安排一跳协商，请求压缩确认提供方能解压 | `17` §9/§10 |
| 10 | 文件类接口单独客户端：独立 Encoder/超时/日志档 | `18` §10.6 |

这份清单的价值在"每条都有对应的失败故事"——表内引用就是事故复盘的落点，新人接手一个 Feign 客户端时按 10 条打钩，比读 25 篇更快建立防线。
