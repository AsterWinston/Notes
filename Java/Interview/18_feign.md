本篇覆盖 OpenFeign 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。四个主题块：定位与用法、核心原理、生产配置与性能、降级治理与演进。调用方式选型全景见 15_spring_cloud.md（2.2 各题），负载均衡与熔断体系见 15_spring_cloud.md（3.2 各题），Sentinel 深水区见 20_sentinel.md，网关侧调用约束见 19_spring_cloud_gateway.md。

# 1. 定位与用法

## 1.1 血统与定位

### 1.1.1 Feign、OpenFeign、Spring Cloud OpenFeign 是什么关系？

**三个名字三代人**（这题答"两个东西是一回事"直接出局）：

| 名字 | 本体 | 状态（2026） |
|------|------|-------------|
| Feign（Netflix） | 2012 年 Netflix 开源的声明式 HTTP 客户端库，注解是自家的 `@RequestLine/@Param` | 2016 年后停更，代码捐给社区 |
| **OpenFeign / Feign 社区版**（`io.github.openfeign:feign-core`） | 社区接管的继续维护线：核心仍是"注解 → 动态代理 → HTTP"，扩展了契约（Contract）机制 | 活跃，版本随 feign-core 演进 |
| **Spring Cloud OpenFeign**（`spring-cloud-starter-openfeign`） | Spring Cloud 对 feign-core 的**封装适配层**：`@FeignClient`、Spring MVC 注解支持、与 LoadBalancer/CircuitBreaker/Nacos 注册的集成、starter 自动装配 | Spring Cloud 列车现役组件（2025.1 起对齐 Boot 4） |

**日常口语的"OpenFeign"通常指第三行**（Spring Cloud 官方文档也自称 Spring Cloud OpenFeign，历史章节里写"Feign"）——严格辨析：`@FeignClient` 这个注解是 **Spring Cloud 的**，不是 Feign 核心的；Netflix Feign 时代没有这个注解。

**Spring Cloud OpenFeign 在 Feign 之上做了什么**（对比旧"Netflix 全家桶"记忆点）：1）`SpringMvcContract` 替换默认契约，复用 `@GetMapping/@RequestParam` 等 MVC 注解（1.2.1）；2）Encoder/Decoder 桥接 Spring 的 `HttpMessageConverters`（Jackson 等随 Boot 全局配置生效）；3）与 Spring Cloud LoadBalancer 深度集成（服务名 → 实例解析，2.2.3）；4）熔断抽象接入（fallback/FallbackFactory，4.1.1）；5）`@EnableFeignClients` 扫描注册 + 每客户端子容器（2.1.2）；6）配置属性体系与自动装配。

**答题收口**：Ribbon/Hystrix 退场后，Spring Cloud OpenFeign 活下来并且默认搭配的是 LoadBalancer + CircuitBreaker/Sentinel——"声明式调用"这个抽象经受住了组件换代，因为它解决的是**写 HTTP 模板代码的无聊**，不绑定任何一代治理件。

### 1.1.2 @FeignClient 的关键属性有哪些？各解决什么问题？

```java
@FeignClient(
    name = "order-service",          // 服务名（经注册中心+LB 解析）；value 同义
    url = "${order.direct-url:}",    // 直连地址，配了就不走注册中心（联调/固定对端用）
    path = "/api",                   // 所有方法的公共路径前缀
    contextId = "orderReadClient",   // 客户端身份 ID：Bean 名与子容器名（同名多客户端必配，否则启动冲突）
    configuration = OrderFeignConfig.class,  // 该客户端专属配置类（不进全局，2.1.2）
    fallback = OrderFallback.class,          // 降级实现（需熔断器支撑，4.1.1）
    fallbackFactory = OrderFallbackFactory.class,  // 降级工厂（可拿异常，二选一）
    primary = false                  // 同接口多实现时是否优先注入
)
public interface OrderClient { ... }
```

**逐个的实战考点**：
1. **name vs url**：url 存在时跳过服务发现——生产慎用（等于把 LB/摘除能力关掉）；灰度直连排障时是利器（"绕开注册中心验证是不是节点问题"，15_spring_cloud.md 4.2.2 排障闭环的一环）；
2. **contextId**：两个接口 `name` 相同（同服务拆读写接口，常见于按业务域组织 client 包）必须各给 contextId，否则 `bean already defined` 启动报错——高频新手坑；它还决定日志里的客户端身份与子容器隔离粒度（2.1.2）；
3. **configuration**：指向一个**不带 @Configuration 也能用**的配置类（Feign 推荐不加，防止被组件扫描意外全局化，2.1.2）；
4. **fallback 与 fallbackFactory 同配时 fallbackFactory 生效**（且都要有熔断集成才触发，4.1.1）；
5. 5.x/较新版本另有 `qualifiers`、`dismiss404`（404 返回 null 不抛异常，聚合查询场景省心）等冷门属性——报得出一两个即显读过源码文档。

**追问**："`name` 能写服务上下文路径吗"——不能，name 只用于发现；接口级前缀用 path，服务带 context-path 时把前缀写进 path（如 `path="/order-app/api"`），这是 Spring Cloud 体系"服务名 ≠ URL 前缀"的常见混淆点。

### 1.1.3 OpenFeign 和 RestClient、WebClient、@HttpExchange 什么关系？怎么选？

**先分层**（调用方式全景与代际口径见 15_spring_cloud.md 2.2.2，此处只讲 Feign 的位置）：

| 层 | 件 | 形态 |
|----|----|------|
| 命令式 API | RestClient（Boot 3.1+，替代 RestTemplate 的正解） | 链式手写请求 |
| 响应式 API | WebClient | Mono/Flux、非阻塞 |
| **声明式接口** | **Spring Cloud OpenFeign**（动态代理 + 运行时注解解析） | 接口 + 注解 |
| 声明式接口（Spring 原生） | **@HttpExchange**（Spring 6，HTTP Interface，无 Feign 依赖） | 接口 + 注解，底层由 RestClient/WebClient 执行 |

**Feign 与 @HttpExchange 的正面竞争（2026 高频新考点）**：两者抽象几乎同形（接口注解生成代理），差异在**归属与耦合**——@HttpExchange 是 Spring Framework 原生（无需引 Feign 依赖、无子容器魔法、序列化/拦截器全用 Spring 自己的 ClientHttpRequestInterceptor 体系），但它**不含服务发现与负载均衡语义**（URI 写服务名要靠 RestClient + LoadBalancer 底座自己拼）；Feign 的价值在 Spring Cloud 全家桶体验：`@FeignClient(name="order-service")` 一行接上 Nacos + LoadBalancer + CircuitBreaker + Sentinel 资源（4.1.2）。**选型口径**：Spring Cloud 体系内继续用 Feign（生态集成是既得利益）；轻量/新项目/不想背 Feign 概念 → @HttpExchange + RestClient/LB 是官方指路的演进方向（"像调本地方法"这件事，Spring 决定自己做了）。

**易错**：把"声明式"当 Feign 专利——REST 注解风格源自 JAX-RS/Spring MVC 注解复用，@HttpExchange 同样"声明式"；答题报出"声明式是模式，Feign 只是 Java 侧最普及的一种实现"即超纲。

## 1.2 请求模型

### 1.2.1 Contract 是干什么的？为什么 Feign 能用 Spring MVC 注解？

**职责**：Contract（契约）= **"把接口上的注解翻译成请求元数据"的解析器**——接口方法上的注解 → `MethodMetadata`（HTTP 方法、URI 模板、参数位置映射：path/query/header/body、produces/consumes）。Feign 默认 `Contract.Default` 认自家 `@RequestLine/@Param/@Headers`；Spring Cloud 装配时替换为 **`SpringMvcContract`**，认 `@GetMapping/@PostMapping/@RequestParam/@PathVariable/@RequestHeader/@RequestBody`——这就是"用 MVC 注解写远程调用"的全部原理：**注解只是原料，Contract 才是翻译官**。

**时机与性能**：MethodMetadata 在**客户端构建期解析一次、终身缓存**（每个方法一个 handler）——运行期不重复反射解析注解；方法级注解改动要重启生效（动态化没有，改的是模板缓存的前提）。

**@RequestParam 传对象 / @SpringQueryMap**：GET 把 POJO 摊平成 query——旧写法 `@SpringQueryMap`（Spring Cloud 私有注解），**2023.0 起官方改为直接在对象参数上标 `@RequestParam`**（对象字段自动展开为查询参数，并支持自定义 Converter）——旧教程与新写法的分界，报得出即时效感。`@SpringQueryMap` 已废弃但短期仍可用（过渡双支持）。

**扩展位**：自定义 Contract 继承 `SpringMvcContract` 加解析规则（如支持 JAX-RS 注解、复用服务端接口定义）是过去常见做法；现在更常见的复用姿势是**服务端 Controller 与 Feign 接口共享同一个 @HttpExchange 标注的 API 接口**（Spring 6 风格，契约单一来源，11_spring_security.md 里服务端接口即调用方契约）——Contract 定制从"常用"退为"兜底"。

**易错**：GET 请求带 `@RequestBody`——Feign 允许发出（模板构建不拦），但语义反常且中间层/服务端普遍不读 GET body（Servlet 容器默认不解析），**4_java_web.md 的 GET body 三不管地带**照搬适用；接口方法**必须恰好一个 body 参数**（两个 @RequestBody 启动即报错）。

### 1.2.2 Feign 接口怎么写？文件上传下载、动态 URL 这些边角场景怎么处理？

**基本式**（接口即文档，返回值直接映射 Decoder）：

```java
@FeignClient(name = "product-service", path = "/api/products")
public interface ProductClient {
    @GetMapping("/{id}")
    Product get(@PathVariable("id") Long id);                    // 注解必须显式写 value（未开 -parameters 时）

    @PostMapping
    Long create(@RequestBody ProductCreateCmd cmd);

    @GetMapping("/search")
    List<Product> search(@RequestParam("kw") String kw,          // query
                         @RequestHeader("X-Tenant") String tenant);

    @DeleteMapping("/{id}")
    void delete(@PathVariable("id") Long id);                    // 204/空体返回 void/null 都兼容
}
```

**四个高频边角**：
1. **文件上传（multipart）**：默认不工作——需要 `feign-form` 系列适配（`io.github.openfeign.form:feign-form-spring`，较新版本 Spring Cloud 已内置支持）+ 方法签名 `@RequestPart("file") MultipartFile file` + `consumes = MULTIPART_FORM_DATA_VALUE`，且自定义 `Encoder` 委托 `SpringFormEncoder` 包装默认编码器；**大文件走 Feign 是反模式**（内存翻倍：全量缓冲后发送）——直传对象存储正解（25_minio.md）；
2. **文件下载/二进制**：返回 `Response`（feign 原生类型，拿 InputStream 自己处理）或 `byte[]` + `produces = APPLICATION_OCTET_STREAM_VALUE`；**返回 Resource/InputStream 时 Decoder 默认不支持**——要么自定义 Decoder，要么用 byte[]（小文件）；流式要警惕连接占用（响应没读完连接不回收）；
3. **动态 URL/Target 复用**：`url` 支持占位符按配置切换；同一接口指向多租户不同端点可用 `Targeter`/`FeignClientBuilder` 运行期构建实例（较少用，报得出名字即可）；
4. **PATCH 方法**：早期 Feign 不支持（RequestMethod 枚举缺项，`Invalid HTTP method: PATCH` 老报错），现代版本已支持——遇到即版本太老。

**易错**：接口方法签名带 `HttpServletRequest` 之类 Servlet 类型（Feign 不认，按 body 处理直接乱）；**返回 `Page`/复杂泛型**（`Page<Product>` 是 Jackson 反序列化黑洞，无默认构造器 + 泛型擦除）——DTO 转换在客户端接口层做一次是纪律，服务端分页模型别直接透传（8_spring_mvc.md 返回体设计同理）。

# 2. 核心原理

## 2.1 启动与代理

### 2.1.1 OpenFeign 的注册与动态代理原理？从注解到可注入的 Bean 经历了什么？

**三段生命周期**：

**① 注册期**：`@EnableFeignClients` → `@Import` 注册器（`FeignClientsRegistrar`，新版改名 `FeignClientRegistrar`，骨架不变）：
- 扫描（默认主类包，可 `basePackages`）+ 解析每个 `@FeignClient` 接口；
- 不是直接创建实例，而是为每个接口注册一个 **`FeignClientFactoryBean` 的 BeanDefinition**（`definition.setTargetType(接口)`，contextId 定 Bean 名）；
- 另支持 `@FeignClient` 标在类上的 `clients` 显式列表、以及 **`spring.cloud.openfeign.feign-clients.registrations` 声明式注册**（Boot 3.2+ 免注解配置化，新版本加分项）。

**② 创建期**（容器按类型注入该接口时，FactoryBean.getObject() 执行）：
- 经 `FeClientFactory`（NamedContextFactory）拿到**该客户端专属子容器**里的组件（Contract/Encoder/Decoder/Client/Retryer/配置——2.1.2）；
- `Feign.Builder` 组装 **`Contract.parseAndValidateMetadata()`：逐方法解析注解成 MethodMetadata**（一次性，缓存）；
- `Targeter.target()` 产出 **JDK 动态代理**（接口无实现类，只能 JDK Proxy），InvocationHandler 为 **`ReflectiveFeign.FeignInvocationHandler`**：`method → MethodHandler`（`SynchronousMethodHandler`，每方法一个）的映射表。

**③ 调用期**：代理拦截 → 对应 handler 执行请求流水线（2.2.1 的十步详拆）。

**必讲细节（追问高发）**：
- **为什么用 JDK 动态代理而不是 CGLIB**：目标是纯接口，JDK Proxy 零依赖够用；CGLIB 是"没有接口"才被迫的选择（11_spring_security.md 里 AOP 选型同款论证）；
- **default 方法**：接口 default 方法 Feign **不会代理成 HTTP 调用**（`Object` 方法与 default 方法走本地执行）——"想在 client 接口里包一层重试/组合逻辑"的合理姿势；
- **懒加载**：代理对象在**首次注入**时才创建（FactoryBean 语义），启动期不校验目标服务可用性——"URL 写错启动不报错、调用才炸"是设计内行为，接口冒烟测试要覆盖。

### 2.1.2 每个 @FeignClient 的子容器是怎么回事？全局配置和专属配置怎么隔离？

**机制**：Spring Cloud 用 **`NamedContextFactory`（FeignClientContextFactory/LoadBalancer 同款家族）** 给每个 contextId 建一个 **Spring 子 ApplicationContext**：父容器 = 应用上下文（可见全局 Bean），子容器 = 该客户端专属组件集（Contract、Client、Encoder、Logger.Level、ErrorDecoder、RequestInterceptor 各一套）。`configuration = XxxConfig.class` 指定的类**注册进子容器**——A 客户端的自定义 Decoder 影响不到 B。

**由此推出两条使用纪律（这题真正的考点）**：
1. **配置类别标 @Configuration 且别放 @ComponentScan 范围**——标了/扫了就被**父容器**收编，所有客户端共享（子容器按类型查找时又能在父级兜底找到），"专属配置"变全局污染，隔离静默失效——**Feign 第一大经典坑**，配置类普通 class 只交给 `configuration` 属性引用即可；
2. **优先级**：子容器显式 Bean > 子容器自动装配默认 > 父容器全局 Bean（查不到才向上找）。`@Autowired @Qualifier` 到具体客户端要配 contextId（Bean 名规则：`contextId + "FeignClient"`）。

**全局化诉求的正解**：想"所有客户端都加一个拦截器/统一超时"——把 Bean 放**父容器**（@Component 的 RequestInterceptor 全局生效；超时走配置属性 default 段 3.1.1），而不是往每个 configuration 里塞。**易混**：`RequestInterceptor` 放父容器 = 全局；`Logger.Level`/`ErrorDecoder` 同理——**子容器优先取自己的，没有才落到全局**，这个查找方向决定了"全局兜底 + 个别覆盖"的治理模型。

## 2.2 调用链路

### 2.2.1 一次 OpenFeign 调用的完整流程？

**十步主线**（背骨架、讲每步"谁负责"，细节见各专题题）：

```text
① orderClient.getOrder(1001L)                    —— 调用方按接口编程
② JDK 代理 → FeignInvocationHandler.invoke()      —— 按 Method 路由到 MethodHandler
③ SynchronousMethodHandler：取缓存的 MethodMetadata，
   用参数值填充 → 生成 RequestTemplate            —— URL 模板 {id} → /api/products/1001
④ 遍历 RequestInterceptor.apply(template)         —— 注入 token/traceId/灰度头（4.2.1）
⑤ Encoder：@RequestBody 对象 → JSON 字节           —— SpringEncoder/HttpMessageConverters（2.2.2）
⑥ 解析目标实例：Client 拿到 http://order-service/... 
   经 LoadBalancer 选实例改写为 http://10.0.0.7:8080/... （2.2.3）
⑦ 底层 HTTP 客户端发请求（URLConnection/OkHttp/HC5，3.1.2）
⑧ 非 2xx → ErrorDecoder.decode() 翻译成异常抛出    —— 可携带 RetryableException 触发重试（2.2.2/3.1.1）
⑨ 2xx → Decoder：JSON → 返回值类型                 —— 泛型信息来自方法签名（2.2.2）
⑩ 返回；期间 Logger 按级别记录、熔断/统计切面在此层观察成败（4.1）
```

**三个"谁在哪一步"追问点**：超时发生在 ⑦（connect/read 两段，3.1.1）；重试的入口有两个——Feign Retryer ⑧ 之后（默认 NEVER_RETRY）与 LoadBalancer 层 ⑥↔⑦（实例级换机重试，2.2.3）；熔断统计 ⑩ 包在整次调用外（Sentinel 把 ②~⑩ 当一个资源，4.1.2）。

**易错**：把 ⑥ 说成"先选实例再构建模板"——现代实现是**模板先构建（服务名 URI）、执行时改写**（LB 感知的 Client 装饰器负责），顺序说反说明背的是 Ribbon 时代旧文（旧版才是 Target 构造期带 LB 选择）。

### 2.2.2 Encoder/Decoder/ErrorDecoder 怎么协作？异常最终以什么形态抛给调用方？

**编解码桥接 Spring**：默认 `SpringEncoder/SpringDecoder` 内部就是 `ObjectFactory<HttpMessageConverters>`——**Boot 全局配置的 Jackson（含 JavaTimeModule、自定义 Module、命名策略）自动对 Feign 生效**，这是 Spring Cloud OpenFeign 体验丝滑的关键（换 Jackson↔Fastjson2 改全局 HttpMessageConverter 即可，不必动 Feign）；按 Content-Type 选择 converter（响应 `application/json` 走 MappingJackson2HttpMessageConverter）。**坑位**：请求没带 Content-Type 时 Encoder 挑转换器可能落空（`No encoder` 报错）——POST JSON 显式 `contentType = "application/json"` 最稳；`Page`/多态类型反序列化失败属 Jackson 通用问题（1.2.2 收口句）。

**ErrorDecoder 的位置与语义**：响应**状态码非 2xx**（且不是 RetryableException 场景）时进入 `decode(methodKey, response)`——返回一个 **Exception 即抛出**、返回 null 继续默认流程（默认 ErrorDecoder 抛 `FeignException` 家族：`FeignException.BadRequest/InternalServerError...`，带状态码与响应体）。自定义姿势：按状态码翻译成**业务异常**（404 → `ResourceNotFoundException`），让调用方 catch 业务语义而不是 parse `FeignException.contentUTF8()`。

**最终异常形态清单**（这题的区分度在"把异常家族列全"）：
- 网络/超时：底层客户端抛出后被包成 `RetryableException`（连接失败/IO 异常默认视为可重试信号）或 `FeignException`；**SocketTimeoutException 常被包装**——catch 时要看 cause 链（熔断统计能否识别"慢调用/超时"取决于包装层次，4.1.2 埋钩子）；
- 4xx/5xx：`FeignException`（或你的 ErrorDecoder 产物）；
- 解码失败：`DecodeException`（JSON 格式不符/字段类型不匹配——**响应结构变更的爆炸点**，14_canal.md 3.3.2 同款"下游先兼容"纪律）；
- 全部上述异常**继续向上**，被熔断层观察到（计次/降级触发），若配了 fallback 则被替换为降级返回（4.1.1）。

**追问**："Feign 超时了调用方拿到什么"——默认 `FeignException`（cause=SocketTimeoutException，read timeout）或底层客户端的超时异常（HC5 抛 `ConnectTimeoutException/SocketTimeoutException` 的包装）——**"把超时和 5xx 分开统计"是熔断阈值调准的前提**（Resilience4j 按异常类型分类、Sentinel 按异常比例/慢调用比例，20_sentinel.md）。

### 2.2.3 Feign 与 LoadBalancer 的集成发生在哪一步？换实例重试是怎么回事？

**集成点**：Feign 核心不感知"服务名"——Spring Cloud 在**子容器装配 Client 时给出"负载均衡感知"的 Client 装饰器**（`org.springframework.cloud.openfeign.loadbalancer` 包一族：按底层客户端匹配 BlockingLoadBalancer 版 HttpClient/OkHttp/Default 实现，由自定义 `Targeter` 挂接）：执行时拿 RequestTemplate 里的 `http://order-service/...`，调 **`BlockingLoadBalancerClient.execute()`** 选实例、改写真实 ip:port 后交给真实 HTTP 客户端——**实例选择发生在"发请求前一刻"**（2.2.1 ⑥），与命令式 `@LoadBalanced RestClient` 同一底座（拦截器式改写，15_spring_cloud.md 3.2.2 的 supplier 缓存与算法全适用）。

**两层重试、别混为一谈（本题最深考点）**：
1. **Feign 层 Retryer**：作用于"拿到可重试响应之后"（429/Retry-After、RetryableException），默认 `Retryer.NEVER_RETRY`——**OpenFeign 默认不重试**（与直觉相反的送命点），开启要显式配 `Retryer.Default(period,maxPeriod,maxAttempts)` Bean；且重试**不保证换实例**（同一 handler 重放，LB 层每次重新选才有换机语义）；
2. **LoadBalancer 层 RetryPolicy**：Spring Cloud LoadBalancer 自带重试策略（默认关闭；配 `spring.cloud.loadbalancer.retry.enabled=true` + `RetryPolicy` Bean 如 `NextAvailableServiceInstanceRetryPolicy`）——**针对幂等请求（GET）失败自动换下一个实例**，这才是"换机重试"的正解；
3. 叠加顺序：一次 Feign 调用内可能 LB 层先换机 N 次、Feign 层再按 Retryer 重放——**两层都开 = 重试次数乘法**，下游故障时放大流量（20_sentinel.md 的重试风暴论证同款；13_rocketmq.md 3.2.1 一个思想）。

**工程结论**：写操作**宁可不重试 + 消费端幂等**（13_rocketmq.md 3.2.3 同构论证）；只读接口开 LB 层换机重试收益最大；超时预算要按"总耗时 = 单次超时 × (重试次数)"倒推（15_spring_cloud.md 3.2.3 的超时预算链），上游网关的 read timeout 必须大于下游全链重试总和，否则"网关先断、后端还在重试"（19_spring_cloud_gateway.md 超时配置对齐）。

# 3. 生产配置与性能

## 3.1 超时与底层客户端

### 3.1.1 超时怎么配？默认值是多少？为什么说"Feign 超时配置可能不生效"？

**默认值先记**：Feign 核心默认 **connectTimeout=10s、readTimeout=60s**（`Request.Options`）——**生产直接裸跑的默认值是事故配置**（60 秒读超时=调用线程陪着下游死 60 秒，Tomcat 线程池瞬间抽干，5_tomcat.md 4.1.2 排队账）。

**配置属性**（2026 口径：前缀已迁移——Spring Cloud OpenFeign **4.1+ 全部收进 `spring.cloud.openfeign.*`**，老教程的 `feign.client.config.*` 是旧代写法，两套同时存在时以所用版本文档为准）：

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:                    # 全局兜底
            connect-timeout: 2000
            read-timeout: 5000
            logger-level: basic
          product-service:            # 按服务名/contextId 覆盖
            read-timeout: 10000       # 慢接口单独放宽，别动全局
```

**三层取值优先级与"不生效"真相（本题硬核）**：
1. **代码级 `configKey`/RequestOptions Bean > 配置属性 default/具名**；
2. **换底层客户端后，超时语义由客户端层再解释一遍**：Apache HttpClient5 有自己的 `connectTimeout/socketTimeout/responseTimeout`（RequestConfig）与连接池 `time-to-live`，OkHttp 有自己的 `connectTimeout/readTimeout`——**不同实现读取 Feign Options 的路径不同**：HC5 集成里 `responseTimeout`（等价 read）与 Feign readTimeout 谁生效取决于版本装配细节，历史上多起"配了 5s 实际 60s"的事故都出在**混配**——纪律：**超时只在一层定死（推荐 Feign 属性层），另一层显式对齐或不配**，别两层各猜各的；
3. **还有第三层"隐形超时"在 Feign 之外**：连接池**获取连接的等待上限**（HC5 `connectionRequestWaitTimeout`，池满排队——排队时间不计入 read timeout，但会计入调用方总 RT；5_tomcat.md/12_nginx.md 同款"池等待≠网络超时"）、LB 重试叠加（2.2.3）、网关/Sidecar 更外层超时——**排障时"Feign 超时 5s 却 30s 才返回"多半是 排队 × 重试 的账**。

**收口**：超时值不是拍脑袋——自下而上按"该接口 P99 × 安全系数"定，再核对全链预算（入口网关 > 聚合服务 > 叶子服务逐层递减，15_spring_cloud.md 3.2.3）；改超时走配置中心动态下发（4.1.2 的 Nacos 联动），别为改个超时发一次版。

### 3.1.2 默认 HttpURLConnection 有什么问题？OkHttp 和 Apache HttpClient5 怎么选？连接池怎么配？

**默认实现的三宗罪**：无连接池（每请求新建 TCP+TLS，高 QPS 下 TIME_WAIT 堆积、握手 CPU 白烧）、无池化等待语义（并发不受控）、功能弱（高级超时/拦截缺）。**生产第一步就是换客户端**：

```yaml
spring:
  cloud:
    openfeign:
      httpclient:
        hc5:
          enabled: true        # Apache HttpClient5（旧版键：feign.httpclient.hc5.enabled）
      # okhttp:
      #   enabled: true        # OkHttp 三选一
```

**二选一对照**：

| 维度 | Apache HttpClient5 | OkHttp |
|------|--------------------|--------|
| 定位 | 企业级全能：细粒度超时三段（connect/socket/response）、认证/重试策略、成熟监控生态 | 轻量现代：**连接池+路由管理省心**、原生 HTTP/2、WebSocket |
| HTTP/2 | 支持（需 TLS/ALPN，实践少用） | 默认协商，内网 h2c 场景更顺 |
| 配置复杂度 | 高（Manager/RequestConfig 多个对象互相覆盖，3.1.1 的混配坑大户） | 低（Builder 一气呵成） |
| 依赖 | Apache 系（多数企业已有） | 独立三方 |
| 常见口径 | **默认推荐**：与 Spring/Boot 生态默认客户端一致（RestClient 的 HC5 支持同源）、可控项多 | 追求简洁/HTTP2 或已因其他组件引入 OkHttp 时 |

**HC5 连接池基线**（数值按容量定标，结构要会写）：

```java
@Bean
public ApacheHttpClient5BuilderCustomizer<PoolingHttpClientConnectionManagerBuilder> poolCustomizer() {
    return b -> b.setDefaultMaxPerRoute(50)      // 每目标服务并发上限——默认仅 5！不换客户端也先改这个
                 .setMaxConnTotal(200)           // 总上限
                 .setDefaultConnectionConfig(c -> c.setTimeToLive(TimeValue.ofMinutes(5))
                                                  .setConnectTimeout(TimeValue.ofSeconds(2)));
}
```

**三个必讲联动**：
1. **`maxPerRoute` 是真实并发闸门**：200 总容量、默认 perRoute=5 → 调 3 个服务每个最多 5 并发，其余线程在池里排队（排队时间不在 readTimeout 内，3.1.1 第 3 层）——"Feign 偶发 30s 超时但下游 P99 只有 200ms"头号根因；
2. **KeepAlive 与两端对齐**：客户端连接空闲回收必须**早于**服务端（Tomcat `keepAliveTimeout` 5_tomcat.md 4.1.3 / Nginx `keepalive_timeout` 12_nginx.md 2.2.3）——否则拿到"对端已悄悄关闭"的死连接，症状=**偶发、重试即好的 502/IOException**（12_nginx.md 3.1.2/5.2.2 的 keepalive 竞态在应用侧的原样复刻）；HC5 可配 `validateAfterInactivity` 借出前探活兜底；
3. **池指标进监控**：leased/pending/available 连接数（Micrometer 绑定 HC5 PoolStats）——没有观测的池调参=蒙眼。

**易错**：`httpclient.enabled`（HC4）与 `hc5.enabled` 同时为 true 的优先级、以及"换了客户端但没建自定义 Manager Bean——其实 starter 会按 `spring.cloud.openfeign.httpclient.*` 属性建默认池（max-conn-total=200、max-conn-per-route=50，较新版本）"——属性可用则优先属性，复杂策略才上 Bean；两代配置键混报是面试现场最常见的露馅点，表态"按版本文档对表"最稳。

## 3.2 日志、压缩与吞吐

### 3.2.1 Feign 日志级别有哪些？为什么配了 FULL 还是看不到日志？

**四级**（`Logger.Level`，记录的是 Feign 视角的请求/响应）：

| 级别 | 内容 |
|------|------|
| NONE | 无（默认） |
| BASIC | 方法、URL、响应码、耗时 |
| HEADERS | BASIC + 请求/响应头 |
| FULL | HEADERS + 请求/响应体（排障神器、生产噪音与敏感信息源） |

**配置两处**：级别（`spring.cloud.openfeign.client.config.default.logger-level: full` 或 configuration 里 `Logger.Level` Bean，属性优先级更高）+ **输出开关**。

**"看不到日志"的双重门槛（这题的存在理由）**：Feign 内部 Logger 走 slf4j，**且把日志挂在"Feign 接口类名"这个 logger 上、级别 DEBUG**——必须同时满足：① Level 配到 BASIC/FULL；② `logging.level.com.example.client.OrderClient=DEBUG`（或包级 DEBUG）——只配一个的现场排查能浪费半小时；老实现还有"子容器里没有 Logger Bean"的装配差异坑，统一按"属性 + logging.level 双查"处置成肌肉记忆。

**生产纪律**：FULL 只开在**专用排障窗口或独立低流量客户端**（configuration 指给单一 contextId，2.1.2 隔离的用武之地）；FULL 常开的两笔账——大 body 序列化拖 RT + **请求头里的 Authorization/请求体里的 PII 进日志系统**（合规事故，27_redis.md、30_linux.md 日志治理同理）；要常态观测"调用成功与否/耗时"交给 **Micrometer/Sentinel 指标（4.1.2）与访问日志**，不是 Feign Logger——"日志是给人看的调试件，指标才是给系统用的观测件"。

### 3.2.2 Gzip 压缩怎么开？哪些场景值得？

```yaml
spring:
  cloud:
    openfeign:
      compression:
        request:
          enabled: true
          mime-types: application/json
          min-request-size: 2048      # 小于 2KB 不压（压缩本身烧 CPU）
        response:
          enabled: true               # 本质是"请求头带 Accept-Encoding: gzip"
```

**机制**：请求体压缩（`Content-Encoding: gzip`，服务端 Spring 解压透明）+ 响应压缩（对端按 Accept-Encoding 协商 gzip——**对端得支持**：Tomcat `compressionEnabled/compressionMinSize` 5_tomcat.md、Nginx gzip 12_nginx.md 5.1.2，链路中间层不能再解一次压一次）。

**值不值的三笔账**：内网同机房 JSON 大报文（列表查询、批量提交）带宽敏感才划算；**几 KB 的小请求纯烧两端 CPU**（min-request-size 门槛就是为此）；跨 AZ/公网链路收益放大。**别和 HTTP/2/gRPC 的选型混谈**：压缩解决"体积"，h2 解决"复用与头开销"，gRPC 解决"序列化+多路复用"（15_spring_cloud.md 2.2.1 性能账）——答题分清维度即严谨。响应压缩的隐藏副作用：**流式/边下边处理失效**（整包解压），大结果集改分页。

### 3.2.3 OpenFeign 生产性能优化清单？Feign 有没有真异步？

**清单按收益排序**（每条带"为什么"）：
1. **换客户端 + 池参数**（3.1.2，收益最大头：省掉每请求握手，并发受控）；
2. **超时/重试按接口分级**（3.1.1 + 2.2.3：快接口短超时快失败，慢接口单独放宽；重试只给幂等读）；
3. **连接池 perRoute 与下游实例容量对齐**（perRoute ≥ 单实例可承受并发，防止"我方把对方打死前先把自己池耗干"，5_tomcat.md maxConnections 的镜像账）；
4. **序列化降本**：DTO 瘦身（列表接口只回必要字段——比压缩先想的一步）、全局 ObjectMapper 复用（默认即复用，自定义别每次 new）；
5. **日志 NONE/BASIC**（3.2.1）；
6. **调用点收敛**：循环里逐条 Feign → **批量接口**（一次 POST ids[] 换 N 次 GET）——**RPC 次数是分布式第一成本**，这条比任何参数调优都值钱，答在最后但优先级常在前；
7. 高频只读接口加**客户端缓存**（Caffeine 包一层，27_redis.md 的多级缓存思想；注意失效语义）。

**"Feign 的异步"辨析（追问必考）**：接口方法可声明 `CompletableFuture<T>`/`Future<T>` 返回——**feign-core 只是把同步结果包进 Future，执行仍阻塞当前线程**（伪异步，签名适配而已）；真并行三种正解：① 自己提交业务线程池 `supplyAsync`（线程成本换并行，注意 4.2.1 上下文透传坑）；② 换响应式栈 WebClient（Feign 在 WebFlux 应用里会阻塞 event loop，**官方明确不建议**，19_spring_cloud_gateway.md 同禁）；③ 聚合场景用虚拟线程（Boot 3.2+/Loom：阻塞式 Feign 并发上百路聚合几乎免费——**声明式同步 API 遇到虚拟线程是当代最优雅的解**，1_java.md 虚拟线程章节的落点）。答出"伪异步 + 虚拟线程翻盘"即 2026 满分结构。

**易错**：指望 Feign 自带熔断/限流（它没有，fallback 是熔断层的挂件，4.1.1）；把"性能优化"答成参数堆砌不报"批量与调用次数"这条——MQ/缓存/网关各篇通用的优先级判断。

# 4. 降级治理与演进

## 4.1 熔断降级集成

### 4.1.1 fallback 和 fallbackFactory 有什么区别？"超时也走降级"是天然的还是要配？

**两个挂件定位**：`fallback`（提供接口的降级实现类）与 `fallbackFactory`（按异常生产降级实现的工厂，**能拿到 Throwable**——记录降级原因、按异常类型分流兜底策略的唯一途径；二者同配 factory 生效）。

```java
@Component
public class OrderClientFallbackFactory implements FallbackFactory<OrderClient> {
    @Override
    public OrderClient create(Throwable cause) {
        return id -> {
            if (cause instanceof FeignException.ServiceUnavailable) { /* 服务不可用：返回缓存 */ }
            log.warn("order-service 降级", cause);
            return Order.EMPTY;   // 兜底值：语义要"业务可接受的次优"，不是 null 炸弹
        };
    }
}
```

**关键认知一：fallback 不是 Feign 自己的能力**——`@FeignClient(fallback=...)` 只是**声明**，触发依赖熔断器实现接入：Sentinel（`spring.cloud.openfeign.sentinel.enabled=true`，旧键 `feign.sentinel.enabled`）或 Spring Cloud CircuitBreaker（Resilience4j，`spring.cloud.openfeign.circuitbreaker.enabled=true`）。**什么都不开时配了 fallback 也永不生效**——"降级类写了没反应"的第一排查项（SCA 历史细节：较新版本 Sentinel 适配也走 CircuitBreaker 桥接，属性以版本文档为准）。

**关键认知二：触发范围 ≠ 熔断规则命中**。开了集成后，Feign 调用抛出的**任何异常**（含 4xx/5xx 的 FeignException、超时、解码失败）都会进 fallback——**这是"降级吞掉业务错误"的事故源**：下游返回 400"库存不足"被降级层吞成"默认成功"。正解两条：① ErrorDecoder 把 4xx 翻译成**不被降级接管的业务异常**并原样抛（或 fallbackFactory 里对特定异常 `throw` 重新抛出）；② 降级返回体带**可判别标记**（状态字段/错误码），调用方分支处理——"fallback 返回 null/空集合，上游当正常数据继续算"是教科书级连环错。

**熔断规则与降级的分工复述**（15_spring_cloud.md 3.2.3 框架）：熔断器管**何时快速失败**（异常比例/慢调用比例阈值，20_sentinel.md 深水区），fallback 管**失败了给什么**——资源名口径：Feign 方法在 Sentinel 里的默认资源名是 `GET:http://order-service/orders/{id}`（URL 模板形式，控制台配规则按它圈定，4.1.2 落点）。

### 4.1.2 Feign + Sentinel 的整合原理？和 Resilience4j 路线怎么分工？

**Sentinel 路线（SCA 栈默认）**：开 `sentinel.enabled` 属性后，Feign 客户端创建时**代理外层再包一层 Sentinel 包装**（SentinelFeign.Builder/InvocationHandler 家族）：每次调用以"方法 URL 模板"为资源名进入 `SphU.entry()` 统计——异常数/异常比例/慢调用比例/QPS 全部计入该资源，规则命中抛 `BlockException` → 适配层翻译成降级调用你的 fallback。效果：**所有 Feign 接口自动成为 Sentinel 资源**，控制台对 `GET:http://order-service/...` 直接配流控/熔断规则、看调用拓扑（17_nacos.md 生态联动：规则持久化到 Nacos，重启不丢）。

**Resilience4j 路线**：`circuitbreaker.enabled=true` 走 Spring Cloud CircuitBreaker 抽象——每个 Feign 方法包一层 CircuitBreaker（按配置组 default + 实例名映射），规则用属性/代码声明（`resilience4j.circuitbreaker.*`），降级同样落到 fallback。

**分工口径（选型答题框架）**：能力上两者对 Feign 的挂接点**同构**（都是代理外层 + 异常统计 + fallback 触发），差异在**规则运维模型**——Sentinel：控制台动态改规则 + Nacos 持久化 + 集群限流/热点参数等治理件齐全（国内平台化默认，深水区全在 20_sentinel.md）；Resilience4j：轻量声明式、规则随配置发布（改动走配置中心/发版，无控制台），与 Boot 原生 Micrometer 指标贴合。**同栈纪律二选一**：两套熔断同时包 Feign 会出现"谁先打开谁说了算"的双层统计混乱——面试报"Feign 只接一层熔断，选择取决于治理平台形态"即闭环。

**易错与追问**：① fallback 类忘标 @Component/忘配 circuitbreaker 开关（4.1.1 双坑合集）；② "Sentinel 对 Feign 的熔断统计包含超时吗"——包含（慢调用比例/异常比例都覆盖超时异常，但**前提是超时值本身设得合理**，3.1.1）；③ "降级后 Sentinel 面板上这个资源还有流量吗"——有：被拦截计入 block QPS，这是区分"通过 RT 恶化"与"直接快速失败"的观测位。

## 4.2 上下文透传

### 4.2.1 RequestInterceptor 怎么用？跨线程时为什么拿不到请求上下文？

**定位**：请求发出前（2.2.1 ④）修改 RequestTemplate 的统一扩展点——**凭证、链路、灰度三件套的标准落点**：

```java
@Component   // 放父容器 = 全局所有客户端生效（2.1.2 查找规则）
public class PropagationInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 1) 认证透传：把进入本服务的 Authorization 带给下游（11_spring_security.md 4.3.1 服务间信任的一环）
        var attrs = RequestContextHolder.getRequestAttributes();
        if (attrs instanceof ServletRequestAttributes sra) {
            String auth = sra.getRequest().getHeader("Authorization");
            if (auth != null) template.header("Authorization", auth);
        }
        // 2) 链路：W3C traceparent 由 Micrometer Tracing 的 Feign 桥接自动处理（4.2.1 见下），自定义业务 ID 在此打标
        // 3) 灰度：X-Gray 标签透传，供下游按元数据路由（17_nacos.md 1.2.2 全链路灰度的"透传"半段）
        String gray = GrayContext.get();   // ThreadLocal
        if (gray != null) template.header("X-Gray", gray);
    }
}
```

**经典事故：异步线程里 Feign 调用"静默丢头"**。三样东西全是 ThreadLocal/绑定请求线程的——`RequestContextHolder`（Servlet 上下文）、MDC（traceId）、业务灰度 ThreadLocal：一旦调用发生在 `@Async`/自建线程池/虚拟线程并发聚合里，**拦截器在新线程执行、上下文为空，header 悄悄不传**（不报错，下游 401/链路断/灰度漏——三重隐雷）。对策谱系（1_java.md 3.4 ThreadLocal 治理的调用侧应用）：
1. **TaskDecorator 快照搬运**：提交任务时复制 SecurityContext/MDC/自定义上下文（Spring Security 的 `DelegatingSecurityContextExecutor`、MDC 手工 copy；虚拟线程用 `ContextSnapshot` API——ThreadLocal 版与 ScopedValue 版两说）；
2. **参数显式下传**：拦截器无上下文时从方法入参取（`template.request().requestTemplate()` 拿不到业务参数——拦截器签名不暴露方法参数，需要更细就换 `Capability`/自定义 InvocationHandler，工程上少用）；
3. **traceId 别自己搬**：Micrometer Tracing 的 Feign 桥接（observation 自动注入 traceparent，15_spring_cloud.md 4.2.1）+ 其上下文传播装饰器已处理大部分异步场景，自定义只管业务头。

**易错**：拦截器里**发起远程调用取 token**（比如每次 apply 去 auth-service 换凭证）——拦截器在请求线程同步执行，一个隐蔽的串行依赖就此诞生（RT 叠加 + 循环调用风险）；凭证类要么本地签发（JWT 自包含，11_spring_security.md 4.1）、要么缓存预取。多个全局拦截器**执行顺序**按 Bean 注册序/`@Order`——鉴权头覆盖灰度头这类事故靠顺序纪律防。

## 4.3 定位与演进

### 4.3.1 2026 年还该用 OpenFeign 吗？它的固有问题和替代路线？

**先立结论**：Spring Cloud OpenFeign 仍是现役（2025.1 列车在发版，15_spring_cloud.md 2.1.2），**"该不该用"取决于你要不要它那圈集成**——它的问题与替代都围绕这一点展开。

**四个固有代价**（这题的诚实清单）：
1. **HTTP/1.1 + 文本 JSON 的性能天花板**：每次调用完整头开销、无多路复用（同连接复用靠 keep-alive 串行化，并发高就得多连接）、Jackson 反射序列化——内网高频小调用的性能被 gRPC/Dubbo 协议级碾压（序列化体积与 CPU 账见 15_spring_cloud.md 2.1.3/2.2.1）；
2. **抽象的"漏"**：超时三层配置（3.1.1）、伪异步（3.2.3）、上下文透传坑（4.2.1）、异常包装层次——**声明式把简单留给了 happy path，把复杂漏给了排障**；
3. **运行时反射契约**：接口改动要到调用时才炸（1.2.1 无编译期校验），契约漂移靠共享 API jar 或契约测试兜（4.3.1 展开）；
4. **Spring Cloud 绑定**：子容器、属性代际迁移（`feign.*` → `spring.cloud.openfeign.*`）这些"框架税"（升级节奏核对 15_spring_cloud.md 2.1.2）。

**三条替代/演进路线**：
1. **@HttpExchange 原生化**（1.1.3 已展开）：轻量项目与 Spring 系"去 Feign 化"的官方方向——声明式保留、框架税砍半，服务名解析配 `@LoadBalanced RestClient` 补齐；
2. **热点链路下沉 RPC**：Java 内网高频调用换 Dubbo3/gRPC（对外边界仍 REST）——**Feign 不是性能问题本身，是"全用 HTTP 一把梭"的架构问题的一部分**；
3. **平台层接管**：K8s/mesh 化后服务发现与重试/熔断下沉 Sidecar（15_spring_cloud.md 5.1.2/5.2.1），Feign 退化成"写起来顺手的 HTTP 门面"——治理件别再叠三层。

**收口口径**："Spring Cloud 全家桶 + 中等 QPS 内网调用 → OpenFeign 仍是默认解（生态集成值回票价）；高并发核心链路 → RPC/gRPC 专项；新项目求轻 → @HttpExchange——**选型跟着流量画像和平台战略走，不跟情怀走**。"再补一句运维视角：无论哪条路线，**超时、重试、幂等、透传这四件事都躲不掉**——换框架换不掉分布式功课（15_spring_cloud.md 1.1.1 的账单）。