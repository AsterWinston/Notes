# 1. 手写 ReAct Agent

## 1.1 题目与考点

### 1.1.1 题目描述

实现一个 ReAct（Reasoning + Acting，推理与行动）循环：模型可以调用工具，工具结果回填后继续推理，直到模型不再调用工具或达到最大步数。

### 1.1.2 考点

- 循环与终止条件；
- 工具执行与结果回填；
- 消息序列的正确构造。

## 1.2 参考实现

### 1.2.1 核心代码

```java
public String reactLoop(String question, List<Tool> tools, int maxSteps) {
    List<Message> messages = new ArrayList<>();
    messages.add(SystemMessage.from("你可以使用工具完成任务。"));
    messages.add(UserMessage.from(question));

    for (int step = 0; step < maxSteps; step++) {
        ChatResponse response = chatClient.prompt()
                .messages(messages)
                .toolCallbacks(toCallbacks(tools))
                .call()
                .chatResponse();

        AiMessage ai = response.getResult().getOutput();
        messages.add(ai);

        // 无工具调用 → 最终答案
        if (!ai.hasToolCalls()) {
            return ai.getText();
        }
        // 执行工具并回填
        for (ToolCall call : ai.getToolCalls()) {
            String result = executeTool(call);
            messages.add(new ToolResponseMessage(List.of(
                    new ToolResponse(call.id(), call.name(), result))));
        }
    }
    return "达到最大步数";
}
```

### 1.2.2 关键点解析

- 循环以"是否还有工具调用"为终止条件，而非固定轮次；
- 每次必须把 `AiMessage` 与 `ToolResponseMessage` 成对加入历史，且 `ToolResponse` 的 `id` 要与 `ToolCall.id()` 对应；
- `maxSteps` 是兜底，防止死循环与成本失控。

## 1.3 追问与生产视角

### 1.3.1 常见追问

**追问**：达到最大步数怎么办？返回已获得的部分结果并提示未完成，或降级为直接作答。

**追问**：工具抛异常怎么处理？捕获异常后把错误信息作为工具结果回填，让模型自行纠错或换工具。

### 1.3.2 生产视角

生产环境还要加：单步超时、工具幂等、调用去重、步数与成本上限、全链路 Trace，以及危险工具的审批与护栏。

# 2. 手写 RAG 流程

## 2.1 题目与考点

### 2.1.1 题目描述

实现一个最小可用的 RAG（Retrieval-Augmented Generation，检索增强生成）流程：向量化查询、检索、组装上下文、强约束生成。

### 2.1.2 考点

- 检索、上下文组装、强约束提示；
- 资料不足时的处理策略。

## 2.2 参考实现

### 2.2.1 核心代码

```java
public String ragAnswer(String question) {
    // ① 向量化查询
    Embedding queryEmb = embeddingModel.embed(question).content();

    // ② 检索
    List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                    .query(question)
                    .topK(5)
                    .similarityThreshold(0.7)
                    .build());

    // ③ 组装上下文
    String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n---\n"));

    // ④ 生成
    return chatClient.prompt()
            .system("只依据资料回答，资料不足时说明。")
            .user("资料：\n%s\n\n问题：%s".formatted(context, question))
            .call().content();
}
```

### 2.2.2 关键点解析

- `topK` 控制召回数量，`similarityThreshold` 过滤低相关片段；
- 上下文用分隔符拼接，便于模型区分不同资料；
- 系统提示强约束"只依据资料"，并声明资料不足时的行为。

## 2.3 追问与生产视角

### 2.3.1 常见追问

**追问**：检索为空怎么办？直接拒答或走兜底话术，不要让模型自由发挥。

**追问**：如何加引用？给每个片段编号，要求模型在答案中标注编号并做引用校验。

### 2.3.2 生产视角

生产中要加混合检索与重排、租户过滤、结果缓存、超时降级；上下文要按 Token 预算裁剪，避免超长。

# 3. 手写幂等工具

## 3.1 题目与考点

### 3.1.1 题目描述

实现一个幂等的创建订单工具：重试时不得重复下单。

### 3.1.2 考点

- 幂等键、缓存结果、TTL；
- 重试窗口与并发安全。

## 3.2 参考实现

### 3.2.1 核心代码

```java
@Tool("创建订单（幂等）")
public ToolResult createOrder(
        @ToolParam(description = "幂等键，重试时必须相同") String idempotencyKey,
        @ToolParam(description = "商品编码") String sku) {

    // ① 检查是否已执行
    String cached = redis.opsForValue().get("order:" + idempotencyKey);
    if (cached != null) {
        return ToolResult.ok(cached);
    }
    // ② 执行
    String orderId = orderService.create(sku);

    // ③ 记录结果（覆盖重试窗口）
    redis.opsForValue().set("order:" + idempotencyKey, orderId,
            Duration.ofHours(24));

    return ToolResult.ok(orderId);
}
```

### 3.2.2 关键点解析

- 幂等键由调用方生成并透传，重试时必须相同；
- 结果缓存要覆盖整个重试窗口，TTL 应大于最大重试时长；
- 并发场景下还需用 `setIfAbsent` 加锁或唯一索引兜底。

## 3.3 追问与生产视角

### 3.3.1 常见追问

**追问**：并发同时到达怎么办？用 `setIfAbsent` 抢锁，未抢到者等待或返回处理中。

**追问**：执行成功但写缓存失败？用唯一索引在数据库层兜底，保证最终幂等。

### 3.3.2 生产视角

幂等键要与业务请求绑定，缓存与数据库双保险；失败结果是否缓存取决于业务是否允许重试。

# 4. 手写限流器（令牌桶）

## 4.1 题目与考点

### 4.1.1 题目描述

实现一个线程安全的令牌桶限流器，支持突发流量与匀速补充。

### 4.1.2 考点

- 令牌桶原理、线程安全。

## 4.2 参考实现

### 4.2.1 核心代码

```java
public class TokenBucket {
    private final double capacity;
    private final double refillRate;
    private double tokens;
    private long lastRefill;

    public TokenBucket(double capacity, double refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = capacity;
        this.lastRefill = System.nanoTime();
    }

    public synchronized boolean tryAcquire(int permits) {
        refill();
        if (tokens >= permits) {
            tokens -= permits;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefill) / 1e9;
        tokens = Math.min(capacity, tokens + elapsed * refillRate);
        lastRefill = now;
    }
}
```

### 4.2.2 关键点解析

- 桶容量决定突发上限，补充速率决定平均速率；
- 惰性补充：每次请求时按时间差补令牌，无需定时任务；
- `synchronized` 保证多线程下令牌计数正确。

## 4.3 追问与生产视角

### 4.3.1 常见追问

**追问**：分布式怎么限流？用 Redis + Lua 脚本原子执行，或接入网关限流。

**追问**：令牌桶与漏桶区别？令牌桶允许突发，漏桶输出恒定速率。

### 4.3.2 生产视角

生产中常按 RPM（每分钟请求数）与 TPM（每分钟 Token 数）双维度限流，并对不同租户分配配额。

# 5. 手写重试（指数退避 + 抖动）

## 5.1 题目与考点

### 5.1.1 题目描述

实现带指数退避与抖动的重试：仅对可重试异常重试，避免重试风暴。

### 5.1.2 考点

- 可重试判断、指数退避、抖动。

## 5.2 参考实现

### 5.2.1 核心代码

```java
public <T> T retry(int maxAttempts, Supplier<T> action) {
    RuntimeException last = null;

    for (int attempt = 0; attempt < maxAttempts; attempt++) {
        try {
            return action.get();
        } catch (Exception e) {
            last = new RuntimeException(e);
            if (!isRetryable(e) || attempt == maxAttempts - 1) {
                throw last;
            }
            sleep(backoff(attempt));
        }
    }
    throw last;
}

private Duration backoff(int attempt) {
    long exp = 500L * (1L << attempt);
    long capped = Math.min(exp, 5000);
    double jitter = 0.8 + Math.random() * 0.4;
    return Duration.ofMillis((long) (capped * jitter));
}
```

### 5.2.2 关键点解析

- `isRetryable` 区分可重试（超时、限流）与不可重试（参数错误）；
- 指数退避让重试间隔递增，上限封顶；
- 抖动避免大量请求同时重试造成雪崩。

## 5.3 追问与生产视角

### 5.3.1 常见追问

**追问**：重试会放大流量吗？会，所以要限次、退避、抖动，并配合熔断。

**追问**：写操作能重试吗？需幂等，否则会重复写入。

### 5.3.2 生产视角

重试要设总预算（次数与总时长），并区分下游故障类型；关键链路建议用现成库（如 Resilience4j）。

# 6. 手写 RRF 融合

## 6.1 题目与考点

### 6.1.1 题目描述

实现 RRF（Reciprocal Rank Fusion，倒数排名融合），把多路检索结果融合为一个排名。

### 6.1.2 考点

- RRF 公式、只依赖排名。

## 6.2 参考实现

### 6.2.1 核心代码

```java
public List<Document> rrfFuse(List<List<Document>> rankedLists, int k) {
    Map<String, Double> scores = new HashMap<>();
    Map<String, Document> docs = new HashMap<>();

    for (List<Document> list : rankedLists) {
        for (int rank = 0; rank < list.size(); rank++) {
            Document doc = list.get(rank);
            String id = doc.getId();
            scores.merge(id, 1.0 / (k + rank + 1), Double::sum);
            docs.putIfAbsent(id, doc);
        }
    }
    return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .map(e -> docs.get(e.getKey()))
            .toList();
}
```

### 6.2.2 关键点解析

- 分数为 `1 / (k + rank + 1)`，只依赖排名，不依赖原始分数；
- `k` 平滑低排名文档的影响，常用 60；
- 同一文档出现在多路结果中，分数累加，排名更靠前。

## 6.3 追问与生产视角

### 6.3.1 常见追问

**追问**：为什么不直接加权分数？不同检索器分数量纲不同，RRF 更稳健。

**追问**：RRF 与加权融合怎么选？分数可比时用加权，否则用 RRF。

### 6.3.2 生产视角

RRF 后常接重排（Reranker）精排；要记录各路来源便于分析召回质量。

# 7. 手写上下文裁剪

## 7.1 题目与考点

### 7.1.1 题目描述

在 Token 预算内裁剪对话历史，优先保留最新消息。

### 7.1.2 考点

- 从新到旧、预算控制。

## 7.2 参考实现

### 7.2.1 核心代码

```java
public List<Message> trimToBudget(List<Message> messages, int maxTokens) {
    int used = 0;
    LinkedList<Message> result = new LinkedList<>();

    // 从最新往旧保留
    for (int i = messages.size() - 1; i >= 0; i--) {
        Message msg = messages.get(i);
        int tokens = estimateTokens(msg.text());
        if (used + tokens > maxTokens) break;

        result.addFirst(msg);
        used += tokens;
    }
    return result;
}
```

### 7.2.2 关键点解析

- 从最新往旧遍历，保证最近上下文优先；
- `estimateTokens` 为估算函数，中文约 1 字 1 Token 量级；
- 用 `addFirst` 维持原有顺序。

## 7.3 追问与生产视角

### 7.3.1 常见追问

**追问**：系统提示会被裁掉吗？不会，系统提示单独保留，只裁剪历史。

**追问**：工具结果太长怎么办？对长结果先摘要或截断再入历史。

### 7.3.2 生产视角

生产常用"摘要 + 最近若干轮 + 检索记忆"的组合，并监控上下文长度分布。

# 8. 手写护栏 Advisor

## 8.1 题目与考点

### 8.1.1 题目描述

实现一个 Advisor（拦截器），在模型调用前后做输入与输出护栏。

### 8.1.2 考点

- 拦截器链、输入输出护栏。

## 8.2 参考实现

### 8.2.1 核心代码

```java
public class GuardrailAdvisor implements CallAdvisor {

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        // 输入护栏
        String input = extractUserText(request);
        if (inputGuardrail.isSuspicious(input)) {
            return rejected("输入包含可疑内容");
        }
        // 调用
        ChatClientResponse response = chain.nextCall(request);

        // 输出护栏
        String output = response.chatResponse().getResult().getOutput().getText();
        if (containsSystemPrompt(output)) {
            return rejected("检测到提示泄露");
        }
        return response;
    }

    @Override
    public String getName() { return "GuardrailAdvisor"; }

    @Override
    public int getOrder() { return 50; }
}
```

### 8.2.2 关键点解析

- `adviseCall` 在调用前后插入逻辑，`chain.nextCall` 触发下一个 Advisor 或模型；
- 输入护栏拦注入，输出护栏拦泄露与违规；
- `getOrder` 控制执行顺序。

## 8.3 追问与生产视角

### 8.3.1 常见追问

**追问**：护栏误伤怎么办？阈值可调，并监控误杀率。

**追问**：护栏能 100% 防注入吗？不能，需与最小权限、输出校验叠加。

### 8.3.2 生产视角

护栏命中率与误杀率都要看板化；规则变更要进 CI 回归。

# 9. 常见考点汇总

## 9.1 考点速查表

| 题目 | 考点 |
|------|------|
| ReAct 循环 | 循环、终止、工具回填 |
| RAG 流程 | 检索、上下文、约束 |
| 幂等 | 幂等键、缓存 |
| 限流 | 令牌桶 |
| 重试 | 退避 + 抖动 |
| RRF | 融合公式 |
| 上下文裁剪 | 预算控制 |
| 护栏 | Advisor |
| 循环检测 | 签名去重 |
| 结构化输出 | Schema + 校验 |

## 9.2 高频变体

### 9.2.1 循环检测

对每次工具调用生成签名（工具名 + 参数哈希），重复出现即判定循环并终止。

### 9.2.2 结构化输出

用 JSON Schema 约束输出，反序列化后校验，失败则重试或降级。

# 10. 答题技巧

## 10.1 五条原则

1. **先讲思路**，再写代码；
2. **写出关键点**（终止、边界、异常）；
3. **说明取舍**（为什么这样写）；
4. **考虑生产**（超时、幂等、监控）；
5. **主动补充**（"实际生产还需要..."）。

## 10.2 表达细节

### 10.2.1 先讲思路

先口述算法与数据结构，得到认可后再落代码，避免方向错误。

### 10.2.2 写出关键点

把终止条件、边界情况、异常处理写清楚，比写全更容易得分。

### 10.2.3 考虑生产

主动补充超时、幂等、限流、Trace 等生产要点。

# 11. 小结

## 11.1 关键结论

- 手写题重点考 ReAct 循环与工具调用。
- 关键：终止条件、幂等、错误处理。
- 限流、重试、RRF 是常见题。
- 先讲思路再写代码，主动说取舍。
- 体现生产意识（超时、监控、边界）。

## 11.2 自测清单

- 能否默写 ReAct 循环与终止条件？
- 能否说清幂等键与 TTL 的作用？
- 能否讲清令牌桶与漏桶的区别？
