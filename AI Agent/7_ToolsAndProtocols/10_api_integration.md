# 1. 把 REST API 封装为工具

## 1.1 核心原则

**结论先行**：把 REST（Representational State Transfer，表述性状态转移）API 封装为工具时，必须遵循"一个业务动作对应一个工具"，绝不能把通用的 HTTP（HyperText Transfer Protocol，超文本传输协议）调用能力直接暴露给模型。

### 1.1.1 反例与正例

| 反例（危险） | 正例（推荐） |
|--------------|--------------|
| `callApi(url, method, body)` | `createTicket(title, desc)` |
| `executeSql(sql)` | `queryOrdersByDate(start, end)` |
| `httpGet(path, params)` | `getUserProfile(userId)` |

反例的问题在于：参数过于自由、语义模糊、权限无法收敛，模型可以组合出任意请求，甚至访问内网地址。

### 1.1.2 为什么通用 HTTP 工具危险

- 权限无法收敛：工具本身不携带业务语义，鉴权只能靠 URL 拼接，极易越权。
- 提示词无法约束：模型看到 `url` 参数就会尝试构造任意地址，包括内网元数据地址。
- 审计无意义：日志里只有一条 URL，无法还原"用户想做什么业务动作"。
- 错误不可解释：下游返回的原始错误无法映射为模型能理解并自我纠正的建议。

## 1.2 封装的分层模型

**结论先行**：一次工具调用应穿越"模型层 → 工具层 → 客户端层 → 远端服务"四层，每层职责单一、边界清晰。

### 1.2.1 分层职责

各层职责与禁止事项：

| 层 | 职责 | 不应做 |
|----|------|--------|
| 模型层 | 选择工具、组织参数 | 拼接 URL、处理协议 |
| 工具层 | 校验、鉴权、幂等、裁剪 | 处理连接细节 |
| 客户端层 | 超时、重试、连接池 | 业务语义判断 |
| 远端服务 | 业务实现 | 无条件信任调用方 |

### 1.2.2 与直接调用 HTTP 的差异

| 维度 | 通用 HTTP 工具 | 业务动作工具 |
|------|----------------|--------------|
| 参数 | 任意 url/method/body | 强类型业务字段 |
| 鉴权 | 依赖 URL | 服务端按动作鉴权 |
| 幂等 | 无 | 幂等键 |
| 可观测 | 差 | 按动作打点 |
| 模型友好度 | 低 | 高 |

## 1.3 边界与生产建议

**边界与坑**：

- 工具粒度要适中：太细导致模型选择困难，太粗导致参数爆炸。
- 工具名与描述是模型选择的主要依据，必须描述"做什么"而不是"怎么调"。

**生产建议**：

- 单领域工具数量控制在 20 个以内，超过时按领域拆分 Agent。
- 工具描述统一模板：动作 + 对象 + 约束，例如"创建客服工单（标题必填）"。

# 2. 封装模式

## 2.1 Spring AI 风格封装

**结论先行**：Spring AI 通过 `@Tool` 注解把普通 Java 方法注册为工具，参数用 `@ToolParam` 描述，模型据此生成调用参数。

### 2.1.1 @Tool 与 @ToolParam

| 注解 | 作用 | 关键属性 |
|------|------|----------|
| `@Tool` | 标记方法为工具 | `name`、`description` |
| `@ToolParam` | 描述参数 | `description`、`required` |

### 2.1.2 完整示例

```java
@Component
@RequiredArgsConstructor
public class TicketTools {

    private final TicketApiClient apiClient;

    @Tool("创建客服工单")
    public ToolResult createTicket(
            @ToolParam(description = "工单标题") String title,
            @ToolParam(description = "问题描述") String description,
            @ToolParam(description = "优先级：LOW/MEDIUM/HIGH", required = false)
            String priority) {

        try {
            TicketResponse response = apiClient.create(CreateTicketRequest.builder()
                    .title(title)
                    .description(description)
                    .priority(priority == null ? "MEDIUM" : priority)
                    .build());

            return ToolResult.ok(Map.of(
                    "ticketId", response.id(),
                    "status", response.status()));
        } catch (FeignException e) {
            return ToolResult.fail("创建工单失败：" + e.getMessage());
        }
    }
}
```

## 2.2 LangChain4j 风格封装

**结论先行**：LangChain4j 用 `@Tool` 描述方法、`@P` 描述参数，风格与 Spring AI 接近，但需显式声明工具所在的接口或对象。

### 2.2.1 注解差异

```java
public interface TicketAssistant {

    @Tool("创建客服工单，返回工单号与状态")
    ToolResult createTicket(
            @P("工单标题") String title,
            @P("问题描述") String description,
            @P("优先级 LOW/MEDIUM/HIGH") String priority);
}
```

### 2.2.2 两种框架的取舍

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 生态 | Spring 原生 | 独立，可脱离 Spring |
| 工具注册 | Bean 扫描 | 接口/对象显式绑定 |
| 适用 | Spring Boot 项目 | 任意 Java 项目 |

## 2.3 边界与生产建议

**边界与坑**：

- 不要把 `Map` 作为唯一返回类型，字段拼写错误无法在编译期发现。
- 返回对象必须可被 JSON（JavaScript Object Notation）序列化，避免循环引用。

**生产建议**：

- 为每个工具定义专属的响应记录类型，明确字段含义与单位。
- 工具方法保持无状态，依赖通过构造器注入。

# 3. HTTP 客户端选择

## 3.1 主流客户端对比

**结论先行**：Spring 6.1+ 项目优先用 `RestClient`，响应式场景用 `WebClient`，微服务声明式调用用 `Feign`，底层定制用 `OkHttp` 或 JDK `HttpClient`。

| 客户端 | 特点 | 适用场景 |
|--------|------|----------|
| `RestClient`（Spring 6.1+） | 现代、流畅 API、同步 | 大多数 Spring Boot 应用 |
| `WebClient` | 响应式、非阻塞 | 高并发、Reactor 栈 |
| `Feign` | 声明式、适合微服务 | 服务间调用 |
| `OkHttp` / `HttpClient` | 底层灵活 | 需要精细控制 |

## 3.2 RestClient 配置示例

### 3.2.1 基础 Bean

```java
@Bean
public RestClient ticketApiClient(@Value("${api.ticket.url}") String baseUrl) {
    return RestClient.builder()
            .baseUrl(baseUrl)
            .defaultHeader("Authorization", "Bearer " + apiToken())
            .requestFactory(clientHttpRequestFactory())
            .build();
}
```

## 3.3 边界与生产建议

**边界与坑**：

- `WebClient` 在阻塞上下文中调用会导致线程饥饿，需谨慎混用。
- 客户端 Bean 应单例复用，内部持有连接池；每次新建会耗尽端口。

**生产建议**：

- 统一在客户端层设置默认头、超时、重试，避免散落在业务代码。
- 为每个下游服务配置独立连接池，隔离故障。

# 4. 超时、重试与熔断

## 4.1 超时配置

**结论先行**：任何 HTTP 调用都必须设置连接超时、读取超时与连接请求超时，缺一不可。

### 4.1.1 三类超时

| 超时 | 含义 | 建议值 |
|------|------|--------|
| connectTimeout | 建立 TCP 连接 | 1–2 秒 |
| responseTimeout | 等待响应数据 | 3–5 秒 |
| connectionRequestTimeout | 从池取连接 | 1 秒 |

### 4.1.2 完整工厂配置

```java
@Bean
public ClientHttpRequestFactory clientHttpRequestFactory() {
    HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();

    RequestConfig config = RequestConfig.custom()
            .setConnectTimeout(2_000)
            .setResponseTimeout(5_000)
            .setConnectionRequestTimeout(1_000)
            .build();

    factory.setHttpClient(HttpClients.custom()
            .setDefaultRequestConfig(config)
            .build());
    return factory;
}
```

> 详见 `14_ProductionEngineering/Reliability/1_tool_timeout.md`。

## 4.2 重试策略

**结论先行**：只有幂等的读操作才能自动重试，写操作重试必须依赖幂等键，否则会重复下单。

### 4.2.1 只读可重试

| 方法 | 可重试 | 说明 |
|------|:------:|------|
| GET | 是 | 天然幂等 |
| HEAD | 是 | 天然幂等 |
| PUT | 条件 | 幂等语义 |
| POST | 否（除非幂等键） | 可能重复创建 |
| DELETE | 条件 | 幂等语义 |

### 4.2.2 指数退避

```java
RetryPolicy<Object> policy = RetryPolicy.builder()
        .handle(SocketTimeoutException.class, FeignException.class)
        .withBackoff(Duration.ofMillis(200), Duration.ofSeconds(2), 2.0)
        .withMaxRetries(3)
        .build();
```

## 4.3 边界与生产建议

**边界与坑**：

- 重试会放大流量，必须配合退避与抖动，否则形成重试风暴。
- 熔断半开状态只放行少量请求探测，不能全量放开。

**生产建议**：

- 超时时间应小于上游整体预算，形成超时级联递减。
- 熔断与重试指标接入监控，异常时告警。

# 5. 参数校验

## 5.1 为什么必须服务端校验

**结论先行**：Schema（模式）只是给模型的提示，模型可能生成越界或非法参数，服务端必须再校验一遍。

```java
// Schema 声明 required=false，不代表参数一定合法
// 模型仍可能传入超长字符串或非法枚举
```

## 5.2 校验的四个层次

### 5.2.1 校验示例

```java
// 必填与长度
if (title == null || title.isBlank()) {
    return ToolResult.fail("标题不能为空");
}
if (title.length() > 200) {
    return ToolResult.fail("标题过长（最多 200 字）");
}
// 枚举与范围
if (priority != null && !Set.of("LOW", "MEDIUM", "HIGH").contains(priority)) {
    return ToolResult.fail("优先级取值不合法");
}
// 业务校验
if (!orderService.exists(orderId)) {
    return ToolResult.fail("订单不存在，无法创建关联工单");
}
```

## 5.3 完整示例

```java
@Tool("创建工单")
public ToolResult createTicket(String title, String description, String priority) {
    // ① 必填校验
    if (title == null || title.isBlank()) {
        return ToolResult.fail("标题不能为空");
    }
    // ② 长度校验
    if (title.length() > 200) {
        return ToolResult.fail("标题过长（最多 200 字）");
    }
    if (description != null && description.length() > 5000) {
        return ToolResult.fail("描述过长（最多 5000 字）");
    }
    // ③ 枚举校验
    if (priority != null && !Set.of("LOW", "MEDIUM", "HIGH").contains(priority)) {
        return ToolResult.fail("优先级取值不合法");
    }
    // ④ 业务校验
    return doCreate(title, description, priority);
}
```

## 5.4 边界与生产建议

**边界与坑**：

- 用正则校验时注意灾难性回溯（ReDoS），避免超长输入拖垮 CPU。
- 校验顺序应为"便宜的先做"，先格式后业务，减少无谓的下游调用。

**生产建议**：

- 把校验逻辑抽成独立方法，便于复用与单测。
- 校验失败返回结构化错误，附带修正建议给模型。

# 6. 鉴权与密钥管理

## 6.1 凭证类型

**结论先行**：凭证分为用户级与服务级，代用户操作必须使用用户级 Token（令牌），系统操作使用服务级 Token。

用户级 Token 携带用户身份，下游据此做数据权限过滤；服务级 Token 代表系统身份，权限大、风险高，必须严格限制使用范围。

| 场景 | 凭证 | 风险 |
|------|------|------|
| 代用户操作 | 用户级 Token（OAuth） | 中 |
| 系统级操作 | 服务级 Token | 高 |
| 第三方 API | API Key（密钥管理） | 中 |

## 6.2 密钥管理实现

```java
@Component
public class ApiCredentialProvider {

    private final SecretManager secretManager;

    // 用户级凭证（代用户调用）
    public String userToken(UserContext user) {
        return secretManager.get(
                "api/%s/%s/token".formatted(user.tenantId(), user.userId()));
    }

    // 服务级凭证（系统调用）
    public String serviceToken() {
        return secretManager.get("api/service/token");
    }
}
```

## 6.3 边界与生产建议

**边界与坑**：

- 严禁把密钥写入代码、配置仓库或日志；一旦泄露必须立即轮换。
- 用户级 Token 过期后需要刷新，工具层要处理刷新失败场景。

**生产建议**：

- 使用密钥管理服务（如 Vault）动态下发，短有效期。
- 对凭证使用做审计，异常调用可追溯。

# 7. 幂等性

## 7.1 幂等的本质

**结论先行**：幂等指同一请求执行多次与执行一次效果相同，是重试安全的前提，写操作必须支持。

## 7.2 幂等键设计

幂等键应由调用方生成并全局唯一，常用 UUID（Universally Unique Identifier，通用唯一识别码）或业务唯一标识。

| 环节 | 做法 |
|------|------|
| 传递 | 放在请求头或请求体 |
| 存储 | 下游记录键与结果 |
| 去重 | 键已存在则返回首次结果 |

## 7.3 代码示例

```java
@Tool("创建订单（幂等）")
public ToolResult createOrder(
        @ToolParam(description = "幂等键，重试时必须相同") String idempotencyKey,
        @ToolParam(description = "商品编码") String sku) {

    // 传给下游 API，由其保证幂等
    return apiClient.create(CreateOrderRequest.builder()
            .idempotencyKey(idempotencyKey)
            .sku(sku)
            .build());
}
```

> 写操作必须幂等，详见 `14_ProductionEngineering/Reliability/3_idempotency.md`。

## 7.4 边界与生产建议

**边界与坑**：

- 幂等键必须由模型在重试时原样复用，提示词中要明确要求。
- 幂等窗口有有效期，超过窗口的重复请求仍可能重复执行。

**生产建议**：

- 下游未支持幂等时，在工具层建立"键 → 结果"的本地缓存。
- 幂等记录设置合理 TTL（Time To Live，存活时间）。

# 8. 错误映射

## 8.1 错误映射的目标

**结论先行**：把 HTTP 状态码与异常翻译为模型可理解的错误码与建议，而不是把原始堆栈抛给模型。

## 8.2 HTTP 状态码到工具错误

### 8.2.1 映射规则

4xx 多为客户端问题，通常不可重试，应引导模型修改参数或告知用户；5xx 多为下游故障，可重试或切换工具，需配合熔断。

| 状态码 | 含义 | 可重试 | 建议 |
|--------|------|:------:|------|
| 400 | 参数错误 | 否 | 修正参数 |
| 401/403 | 无权限 | 否 | 换方式或告知用户 |
| 404 | 不存在 | 否 | 确认标识 |
| 429 | 限流 | 是 | 稍后重试 |
| 500/502/503 | 下游故障 | 是 | 稍后重试或换工具 |

## 8.3 完整映射示例

```java
private ToolResult mapError(Exception e) {
    if (e instanceof FeignException feign) {
        return switch (feign.status()) {
            case 400 -> ToolResult.fail("请求参数错误");
            case 401, 403 -> ToolResult.fail("无权限访问该接口");
            case 404 -> ToolResult.fail("资源不存在");
            case 429 -> ToolResult.fail("调用过于频繁，请稍后重试");
            case 500, 502, 503 -> ToolResult.fail("下游服务暂时不可用");
            default -> ToolResult.fail("调用失败：" + feign.getMessage());
        };
    }
    if (e instanceof SocketTimeoutException) {
        return ToolResult.fail("调用超时，请稍后重试");
    }
    return ToolResult.fail("未知错误：" + e.getMessage());
}
```

## 8.4 边界与生产建议

**边界与坑**：

- 不要直接把 `e.getMessage()` 原样返回，可能包含 SQL、路径等内部细节。
- 下游返回 200 但业务码表示失败时，也要映射，不能只看 HTTP 状态。

**生产建议**：

- 定义统一错误码枚举，工具层与监控共用。
- 保留原始异常用于日志，返回给模型的是脱敏信息。

# 9. 结果裁剪与 Token 控制

## 9.1 为什么要裁剪

**结论先行**：API 返回可能包含大量无关字段，直接回传会浪费 Token（词元）、稀释关键信息，必须裁剪为关键字段。

## 9.2 裁剪策略

只保留模型完成任务所需的字段，其余丢弃。

| 策略 | 说明 |
|------|------|
| 分页 | 限制返回条数，提供游标 |
| 摘要 | 长文本截断并标注 |
| 聚合 | 返回统计值而非明细 |

## 9.3 代码示例

```java
// API 返回可能很大，只保留关键字段
private ToolResult toToolResult(OrderResponse response) {
    return ToolResult.ok(Map.of(
            "orderId", response.id(),
            "status", response.status(),
            "amount", response.amount(),
            "createdAt", response.createdAt()));
    // 不返回完整的 response（可能含大量无关字段）
}
```

## 9.4 边界与生产建议

**边界与坑**：

- 裁剪过度会导致模型缺少必要上下文，反复追问，反而更费 Token。
- 截断长文本要给出明确标记，避免模型误以为内容完整。

**生产建议**：

- 为每个工具设定返回体积预算，超限时截断并提示。
- 大结果集提供"继续获取"的分页工具。

# 10. 生产注意事项与小结

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 暴露通用 HTTP 工具 | 按业务动作封装 |
| 无超时 | 必设连接/读取超时 |
| 无重试 | 只读可重试 |
| 写操作非幂等 | 幂等键 |
| 密钥硬编码 | 密钥管理 |
| 无参数校验 | 服务端校验 |
| 返回过大 | 裁剪字段 |
| 错误不友好 | 映射为结构化错误 |

## 10.2 上线检查表

- 每个工具是否对应唯一业务动作，且名称与描述清晰。
- 是否配置连接、读取、请求三类超时。
- 写操作是否支持幂等键，重试是否只针对幂等操作。
- 凭证是否来自密钥管理服务，是否支持轮换。
- 参数是否在服务端完成校验。
- 错误是否映射为结构化错误并附带建议。
- 返回结果是否裁剪且设有体积上限。

## 10.3 小结

- 把 API 按业务动作封装为具体工具，而非通用 HTTP 工具。
- 超时、重试、幂等、鉴权是必备。
- 参数必须服务端校验。
- 错误映射为结构化信息，让模型能调整。
- 返回结果裁剪为关键字段。
