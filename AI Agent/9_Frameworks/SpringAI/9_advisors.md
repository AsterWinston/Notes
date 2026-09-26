# 1. 什么是 Advisor

结论先行：Advisor 是 Spring AI 的拦截器链（类似 Servlet Filter / Spring AOP），负责记忆、RAG、日志、安全、统计等所有横切关注点。

## 1.1 核心定位

Advisor 是 Spring AI 的**拦截器链**机制，类似 Servlet Filter 或 Spring AOP：

```
ChatClient.prompt()
      │
      ▼
┌──────────────────────────────────────────────┐
│  Advisor Chain（按 order 排序）                │
│  ① LoggingAdvisor                             │
│  ② ChatMemoryAdvisor                          │
│  ③ QuestionAnswerAdvisor (RAG)                │
│  ④ SafeGuardAdvisor                           │
│        │                                      │
│        ▼                                      │
│     ChatModel.call()                          │
│        │                                      │
│        ▲ 响应反向穿过各 Advisor                 │
└──────────────────────────────────────────────┘
```

## 1.2 Advisor 能做什么

Advisor 可以：

- 在请求前**修改 Prompt**（注入记忆、检索结果、系统提示）；
- 在响应后**修改 ChatResponse**（过滤、加工、记录）；
- 实现横切关注点：日志、记忆、RAG、安全、限流、可观测。

### 1.2.1 与 AOP 的异同

| 维度 | Spring AOP | Advisor |
|------|-----------|---------|
| 切点 | 方法匹配 | 固定 ChatClient 流程 |
| 顺序 | `@Order` | `getOrder()` |
| 对象 | 任意 Bean | ChatClientRequest/Response |

### 1.2.2 请求与响应模型

Advisor 操作的是不可变的 `ChatClientRequest` / `ChatClientResponse`，修改需通过 `mutate()` 产生新对象。

## 1.3 与 Filter/AOP 的边界

结论先行：Advisor 只作用于 `ChatClient` 调用，不覆盖 HTTP 层与任意方法。

### 1.3.1 职责边界

| 层 | 负责 |
|----|------|
| Servlet Filter | 鉴权、限流、日志（HTTP 级） |
| Advisor | 提示增强、模型响应加工 |
| AOP | 任意 Bean 方法 |

### 1.3.2 不要混用

把 HTTP 鉴权逻辑塞进 Advisor 会导致非 HTTP 调用（如批处理）绕过安全，应在 Filter 层完成。

# 2. Advisor 接口

结论先行：所有 Advisor 都实现 `Ordered`，并按同步/流式分为 `CallAdvisor` 与 `StreamAdvisor`。

## 2.1 接口定义

```java
public interface Advisor extends Ordered {
    String getName();
}
```

两个核心子接口：

| 接口 | 作用 |
|------|------|
| `CallAdvisor` | 拦截同步调用 |
| `StreamAdvisor` | 拦截流式调用 |

## 2.2 核心方法

核心方法（概念）：

```java
ChatClientResponse adviseCall(
        ChatClientRequest request,
        CallAdvisorChain chain);
```

`chain.nextCall(request)` 表示"交给下一个 Advisor（或最终模型）"。

### 2.2.1 责任链语义

```
adviseCall(req)
   ├─ 前置逻辑
   ├─ chain.nextCall(req)  → 下一环/模型
   └─ 后置逻辑（可修改 response）
```

### 2.2.2 不可变对象

`ChatClientRequest` 不可变，修改需 `request.mutate().prompt(...).build()`，这避免了链上相互污染。

## 2.3 链的终止与短路

结论先行：Advisor 可以选择不调用 `chain.nextCall()`，从而短路整条链。

### 2.3.1 短路场景

- 安全拦截命中，直接返回兜底响应；
- 缓存命中，直接返回历史结果；
- 限流超限，快速失败。

### 2.3.2 短路示例

```java
@Override
public ChatClientResponse adviseCall(ChatClientRequest request,
                                     CallAdvisorChain chain) {
    if (isBlocked(request)) {
        return ChatClientResponse.builder()
                .chatResponse(blockedResponse())
                .build();
    }
    return chain.nextCall(request);
}
```

> 短路后不会再有后续 Advisor 与模型调用，务必确认这是预期行为。

# 3. 内置 Advisor

结论先行：常用能力开箱即用，但要注意各自的 starter 依赖。

## 3.1 内置清单

| Advisor | 作用 | 引入 |
|---------|------|------|
| `MessageChatMemoryAdvisor` | 注入对话历史 | `spring-ai-starter-model-chat-memory-repository-*` |
| `PromptChatMemoryAdvisor` | 把历史拼进系统提示 | 同上 |
| `QuestionAnswerAdvisor` | RAG：检索并注入上下文 | `spring-ai-advisors-vector-store` |
| `SimpleLoggerAdvisor` | 打印请求/响应 | 核心 |
| `SafeGuardAdvisor` | 敏感词拦截 | 核心 |
| `ReReadingAdvisor` | 重读提示技巧 | 核心 |
| `ToolCallingAdvisor` | 工具调用循环（2.0 起显式化） | 核心 |

## 3.2 选择建议

### 3.2.1 按需引入

不要一次性挂载所有 Advisor，按业务需要最小化，减少不可见的副作用。

### 3.2.2 依赖与版本

不同版本 Advisor 的包名与构造方式有差异，升级时重点回归。

## 3.3 内置 Advisor 的协作

结论先行：内置 Advisor 可以叠加，但要明确谁先谁后。

### 3.3.1 记忆 + RAG

记忆注入历史，RAG 注入知识，二者顺序影响模型看到的上下文结构。

### 3.3.2 日志 + 统计

日志看内容，统计看数量，通常日志在前以便记录原始请求。

# 4. 注册 Advisor

结论先行：注册分全局默认与单次调用，顺序由 `order` 决定。

## 4.1 注册方式

### 4.1.1 默认注册

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(
                new SimpleLoggerAdvisor(),
                MessageChatMemoryAdvisor.builder(memory).build()
        )
        .build();
```

### 4.1.2 单次注册

```java
chatClient.prompt()
        .user("...")
        .advisors(new SafeGuardAdvisor(List.of("违规词")))
        .call()
        .content();
```

## 4.2 顺序控制

### 4.2.1 显式 order

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(
                Advisor.builder(SimpleLoggerAdvisor::new).order(0).build(),
                Advisor.builder(() -> MessageChatMemoryAdvisor.builder(memory).build())
                        .order(100).build()
        )
        .build();
```

> 顺序很重要：日志应最先（看到原始请求）或最后（看到最终请求）；记忆注入应在检索之前。

### 4.2.2 合并规则

单次 `.advisors(...)` 与全局 `defaultAdvisors(...)` 合并后统一按 order 排序执行。

# 5. 自定义 Advisor

结论先行：自定义 Advisor 是接入企业治理（限流、审计、脱敏）的最佳切面。

## 5.1 日志 Advisor

```java
public class LoggingAdvisor implements CallAdvisor {

    private static final Logger log = LoggerFactory.getLogger(LoggingAdvisor.class);

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        long start = System.currentTimeMillis();
        log.info("请求消息数: {}", request.prompt().getInstructions().size());

        ChatClientResponse response = chain.nextCall(request);

        long cost = System.currentTimeMillis() - start;
        log.info("耗时 {} ms，响应: {}",
                cost,
                response.chatResponse().getResult().getOutput().getText());
        return response;
    }

    @Override
    public String getName() {
        return "LoggingAdvisor";
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

## 5.2 Token 统计 Advisor

```java
public class TokenUsageAdvisor implements CallAdvisor {

    private final MeterRegistry meterRegistry;

    public TokenUsageAdvisor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);
        Usage usage = response.chatResponse().getMetadata().getUsage();

        meterRegistry.counter("ai.tokens.prompt").increment(usage.getPromptTokens());
        meterRegistry.counter("ai.tokens.completion").increment(usage.getCompletionTokens());
        return response;
    }

    @Override
    public String getName() { return "TokenUsageAdvisor"; }

    @Override
    public int getOrder() { return 200; }
}
```

## 5.3 敏感信息脱敏 Advisor

```java
public class PiiMaskingAdvisor implements CallAdvisor {

    private static final Pattern PHONE =
            Pattern.compile("1[3-9]\\d{9}");

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        // 请求侧脱敏
        Prompt masked = maskPrompt(request.prompt());
        ChatClientResponse response = chain.nextCall(
                request.mutate().prompt(masked).build());

        // 响应侧脱敏
        String text = response.chatResponse().getResult().getOutput().getText();
        String maskedText = PHONE.matcher(text).replaceAll("****");
        // 构造新响应（示意）
        return response;
    }

    @Override
    public String getName() { return "PiiMaskingAdvisor"; }

    @Override
    public int getOrder() { return 50; }
}
```

## 5.4 限流 Advisor

### 5.4.1 令牌桶示意

```java
public class RateLimitAdvisor implements CallAdvisor {

    private final RateLimiter limiter;

    public RateLimitAdvisor(RateLimiter limiter) {
        this.limiter = limiter;
    }

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        if (!limiter.tryAcquire()) {
            throw new IllegalStateException("请求过于频繁");
        }
        return chain.nextCall(request);
    }

    @Override
    public String getName() { return "RateLimitAdvisor"; }

    @Override
    public int getOrder() { return 20; }
}
```

### 5.4.2 降级策略

超限时应快速失败并返回兜底话术，而不是排队拖垮线程池。

## 5.5 审计 Advisor

结论先行：审计 Advisor 负责把"谁在何时问了什么、模型答了什么"落库，满足合规要求。

### 5.5.1 记录内容

| 字段 | 说明 |
|------|------|
| 调用者 | 用户/租户 ID |
| 请求摘要 | 脱敏后的提示 |
| 响应摘要 | 脱敏后的回答 |
| 耗时/用量 | 性能与成本 |
| 时间戳 | 审计追溯 |

### 5.5.2 异步落库

审计写入应异步化，避免阻塞主链路；失败时降级为本地日志。

# 6. RAG Advisor：QuestionAnswerAdvisor

结论先行：`QuestionAnswerAdvisor` 把"检索 + 注入"封装成一个 Advisor，一行接入 RAG。

## 6.1 基本用法

```java
ChatClient client = ChatClient.builder(chatModel)
        .defaultAdvisors(QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(SearchRequest.builder()
                        .topK(5)
                        .similarityThreshold(0.7)
                        .build())
                .build())
        .build();
```

它会自动：**用户提问 → 向量检索 → 把命中文档注入 Prompt → 调用模型**。

## 6.2 参数调优

### 6.2.1 topK 与阈值

| 参数 | 作用 | 建议 |
|------|------|------|
| `topK` | 召回数量 | 3~8 |
| `similarityThreshold` | 相似度下限 | 0.6~0.8 |

### 6.2.2 与记忆的顺序

RAG 与记忆同时存在时，记忆注入在前、RAG 检索在后，避免检索结果被历史干扰。

> 详见 `10_rag.md`。

## 6.3 多租户 RAG

结论先行：多租户下检索必须带租户过滤，否则会跨租户泄露。

### 6.3.1 过滤条件

在 `SearchRequest` 中附加租户过滤表达式，只召回当前租户的文档。

### 6.3.2 与 Advisor 的结合

可通过 Advisor 参数传入 `tenantId`，在检索前动态构造过滤条件。

# 7. 流式 Advisor

结论先行：流式场景必须实现 `StreamAdvisor`，否则拦截会被跳过。

## 7.1 基本实现

```java
public class StreamLoggingAdvisor implements StreamAdvisor {

    @Override
    public Flux<ChatClientResponse> adviseStream(ChatClientRequest request,
                                                 StreamAdvisorChain chain) {
        return chain.nextStream(request)
                .doOnNext(r -> System.out.print(
                        r.chatResponse().getResult().getOutput().getText()));
    }

    @Override
    public String getName() { return "StreamLoggingAdvisor"; }

    @Override
    public int getOrder() { return 0; }
}
```

## 7.2 同时支持同步与流式

### 7.2.1 实现两个接口

若同一能力需覆盖两种调用方式，可实现 `CallAdvisor` + `StreamAdvisor`。

### 7.2.2 流式脱敏难点

流式响应是增量 token，脱敏需跨 chunk 缓冲，注意边界截断问题。

# 8. Advisor 的典型组合顺序

结论先行：安全靠前、上下文注入居中、统计靠后。

## 8.1 参考顺序

```
order 0    LoggingAdvisor          记录原始请求
order 50   GuardrailAdvisor        输入校验/拦截
order 100  MessageChatMemoryAdvisor 注入历史
order 150  QuestionAnswerAdvisor   注入 RAG 上下文
order 200  TokenUsageAdvisor       统计用量
order 300  (ChatModel)             实际调用
```

> 具体顺序需按业务调整，但原则是：**安全校验靠前，上下文注入居中，统计靠后**。

## 8.2 顺序错误的后果

### 8.2.1 记忆晚于统计

统计 Advisor 若在记忆注入前执行，会少算历史 Token。

### 8.2.2 安全晚于记忆

安全校验若在记忆注入后执行，可能漏拦历史中的敏感内容。

# 9. 生产注意事项

结论先行：Advisor 虽轻，但顺序、异常与性能三处最容易出事。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| Advisor 抛异常导致整体失败 | 内部 try-catch，降级放行 |
| 顺序错误导致记忆/RAG 失效 | 显式设置 order，写测试 |
| 日志泄露敏感信息 | 脱敏后再记录 |
| 流式拦截遗漏 | 同时实现 `StreamAdvisor` |
| 性能开销 | 避免在 Advisor 里做重 IO |

## 9.2 测试与观测

### 9.2.1 顺序断言测试

用单测断言各 Advisor 的 `getOrder()` 值，防止重构时被改乱。

### 9.2.2 每环耗时

在每个 Advisor 记录进入/离开时间，定位链路瓶颈。

## 9.3 幂等与副作用

结论先行：Advisor 本身应尽量无副作用；若必须写状态，务必幂等。

### 9.3.1 常见副作用

| 副作用 | 风险 | 建议 |
|--------|------|------|
| 计数 | 重试重复计数 | 去重键 |
| 缓存写 | 脏数据 | 版本号 |
| 审计 | 重复记录 | 幂等键 |

### 9.3.2 重试语义

流式调用可能因网络重连触发重放，Advisor 内的副作用需能识别重复请求。

# 10. 小结

## 10.1 要点回顾

- Advisor = Spring AI 的 AOP，负责所有横切能力。
- 记忆、RAG、日志、安全、统计都通过 Advisor 实现。
- 顺序（order）决定行为，务必显式声明并测试。
- 自定义 Advisor 是接入企业治理（限流、审计、脱敏）的最佳切面。

## 10.2 延伸阅读

- 记忆 Advisor：`8_chat_memory.md`
- RAG：`10_rag.md`
- 可观测：`13_Observability/3_agent_trace_and_spans.md`
