# 1. 为什么 Token 是核心变量

**结论先行**：Token（词元）是 LLM（大语言模型）应用唯一同时决定成本、延迟、容量与限流的资源，必须像管理内存一样对它做预算化（budgeting）。

```
Token 决定：成本 + 延迟 + 上下文容量 + 限流
```

## 1.1 Token 的三重成本

### 1.1.1 计费成本

几乎所有厂商都按 Token 计费，且输入（prompt）与输出（completion）分开计价，输出通常更贵。

| 影响 | 说明 |
|------|------|
| 成本 | 按 Token 计费，输入输出都算，输出更贵 |
| 延迟 | Token 越多，生成越慢（尤其输出） |
| 容量 | 超出上下文窗口会报错或截断 |
| 限流 | 厂商按 TPM（Tokens Per Minute，每分钟 Token 数）限制 |

### 1.1.2 延迟成本

输出 Token 与首字延迟（TTFT）之后的总时长近似线性相关，输入 Token 主要影响预填充（prefill）阶段。

```
总延迟 ≈ 预填充(输入长度) + 输出Token数 / 生成速率
```

### 1.1.3 容量与限流成本

上下文窗口（context window）是硬上限；TPM/RPM（每分钟请求数）是软上限。两者任一被击穿都会导致 429 或截断。

## 1.2 Token 与业务指标的关系

| 业务指标 | 与 Token 的关系 |
|----------|----------------|
| 单次会话成本 | 正比于累计 Token |
| P99 延迟 | 正相关于输出 Token |
| 单位时间吞吐 | 反比于平均 Token |
| 毛利率 | 直接受 Token 单价影响 |

## 1.3 反例：忽视 Token 的典型事故

- 把整篇文档塞进提示，单请求 10 万 Token，成本瞬间失控。
- 忘记预留输出空间，模型刚开始回答就被 `max_tokens` 截断。
- 无租户配额，单个客户刷爆全系统 TPM 配额，所有租户 429。

> **Token 是 LLM 应用最需要被"预算化"的资源。**

# 2. Token 估算

**结论先行**：请求发出前必须估算 Token，精确场景用 tokenizer（分词器），快速判断用粗估公式。

## 2.1 粗估与精确估算

### 2.1.1 粗估规则

经验值：中文约 1 Token/字，英文约 1 Token/4 字符，标点与空白另计。

```java
public final class TokenEstimator {

    private TokenEstimator() {
    }

    // 粗估：中文约 1 token/字，英文约 1 token/4 字符
    public static int estimate(String text) {
        if (text == null || text.isEmpty()) {
            return 0;
        }
        int chinese = countChinese(text);
        int other = text.length() - chinese;
        return chinese + (other / 4) + 1;
    }

    private static int countChinese(String text) {
        int count = 0;
        for (int i = 0; i < text.length(); i++) {
            if (Character.UnicodeScript.of(text.charAt(i))
                    == Character.UnicodeScript.HAN) {
                count++;
            }
        }
        return count;
    }
}
```

### 2.1.2 精确 tokenizer

生产计量与配额必须用模型对应的 tokenizer，避免粗估误差累积。

```java
// 伪代码：不同厂商 tokenizer 接口名不同，统一封装为 count
public static int exact(String text, String model) {
    return tokenizerFor(model).count(text);
}

// 示例：一次编码得到 Token 序列长度
int n = tokenizer.encode(text).size();
```

## 2.2 多模态与特殊内容估算

| 内容类型 | 估算方式 |
|----------|----------|
| 纯文本 | tokenizer 精确计数 |
| 图片 | 按分辨率折算为等效 Token（厂商各异） |
| 工具定义 | 按 JSON Schema 文本计数 |
| 结构化输出 | 预留字段名与括号的额外开销 |

## 2.3 估算的误差边界

- 粗估误差通常 10%~30%，只可用于"是否接近上限"的判断。
- tokenizer 版本升级会改变计数，配额账本要与计费版本对齐。
- 流式响应中无法提前知道最终输出 Token，只能按上限预留。

> 生产建议：**精确计算用 tokenizer，快速判断用粗估**；两条路径都要有，且以 tokenizer 为准。

# 3. 上下文预算分配

**结论先行**：把上下文窗口当成一份固定预算，按角色切分，且必须为输出预留空间。

## 3.1 预算模型

```
上下文窗口（如 128K）
├── System 提示         5%     （固定）
├── 工具定义            10%    （固定）
├── 检索上下文（RAG）   40%    （可变）
├── 对话历史            30%    （可变）
├── 当前问题            5%
└── 输出预留            10%    ← 必须预留！
```

## 3.2 预算的代码实现

### 3.2.1 record 实现

```java
public record ContextBudget(
        int total,
        int system,
        int tools,
        int retrieval,
        int history,
        int question,
        int outputReserve
) {

    public static ContextBudget of(int total) {
        return new ContextBudget(
                total,
                (int) (total * 0.05),
                (int) (total * 0.10),
                (int) (total * 0.40),
                (int) (total * 0.30),
                (int) (total * 0.05),
                (int) (total * 0.10));
    }

    public int inputBudget() {
        return total - outputReserve;
    }
}
```

### 3.2.2 动态调整

固定比例只是起点；当工具定义变多时，应从检索比例中借用额度。

```java
// 工具数量变化时重新分配：工具多则压缩检索
public ContextBudget rebalance(int toolTokens) {
    int base = total - outputReserve;
    int fixed = system + toolTokens + question;
    int variable = Math.max(0, base - fixed);
    int retrievalShare = (int) (variable * 0.57); // 剩余中约 57% 给检索
    int historyShare = variable - retrievalShare;
    return new ContextBudget(total, system, toolTokens,
            retrievalShare, historyShare, question, outputReserve);
}
```

## 3.3 输出预留的坑

| 坑 | 后果 |
|----|------|
| 不预留输出 | 响应被截断，JSON 不完整 |
| 预留过小 | 长答案中途停止 |
| 预留过大 | 输入被无谓压缩，召回质量下降 |

> **最常见的错误：忘记为输出预留空间**，导致响应被截断。

# 4. 上下文裁剪

**结论先行**：裁剪的目标不是"塞下更多"，而是"在预算内保留信息密度最高的内容"。

## 4.1 裁剪策略

### 4.1.1 从新到旧

```java
public List<Message> trimToBudget(List<Message> messages, int maxTokens) {
    int total = 0;
    LinkedList<Message> result = new LinkedList<>();

    // 从最新往旧保留
    for (int i = messages.size() - 1; i >= 0; i--) {
        Message msg = messages.get(i);
        int tokens = TokenEstimator.estimate(msg.text());

        if (total + tokens > maxTokens) {
            break;
        }

        result.addFirst(msg);
        total += tokens;
    }
    return result;
}
```

### 4.1.2 摘要压缩

早期对话不必直接丢弃，可先摘要成短文本再放入。

```java
// 伪代码：把早期历史压缩为一段摘要
String summary = summarizer.summarize(earlyMessages);
List<Message> compact = List.of(system, user("历史摘要：" + summary),
        recentMessage);
```

### 4.1.3 滑窗

保留最近 N 轮 + 固定摘要槽位，形成可预测的 Token 占用。

```
[System] [摘要槽] [最近第3轮] [最近第2轮] [最近第1轮] [当前问题]
```

## 4.2 裁剪优先级

```
保留：System 提示 > 当前问题 > 最近对话 > 检索结果 > 早期对话
```

| 优先级 | 内容 | 理由 |
|:------:|------|------|
| 1 | System 提示 | 决定行为与安全 |
| 2 | 当前问题 | 没有它无法回答 |
| 3 | 最近对话 | 指代消解依赖 |
| 4 | 检索结果 | 可再召回 |
| 5 | 早期对话 | 可摘要 |

## 4.3 裁剪的质量风险

- 裁剪掉指代对象会导致"它/那个"无法解析。
- 摘要本身消耗一次模型调用，需计入总成本。
- 裁剪后应做回归评测，确认答案质量未下降。

> 生产建议：裁剪顺序固定、可配置，并对每次裁剪记录"丢了多少 Token、丢了哪类内容"。

# 5. 检索结果的 Token 控制

**结论先行**：检索片段应按预算截断而非整段丢弃，并优先保留高排名的完整片段。

## 5.1 片段级预算

```java
public String buildContext(List<Document> docs, int maxTokens) {
    StringBuilder sb = new StringBuilder();
    int used = 0;

    for (Document doc : docs) {
        String text = doc.getText();
        int tokens = TokenEstimator.estimate(text);

        if (used + tokens > maxTokens) {
            // 超出预算：截断而非丢弃
            int remaining = maxTokens - used;
            text = truncate(text, remaining);
            if (text.isEmpty()) {
                break;
            }
            tokens = TokenEstimator.estimate(text);
        }
        sb.append(text).append("\n---\n");
        used += tokens;
    }
    return sb.toString();
}
```

## 5.2 截断 vs 丢弃

| 策略 | 优点 | 缺点 |
|------|------|------|
| 整段丢弃 | 片段语义完整 | 浪费剩余预算 |
| 末段截断 | 用满预算 | 语义可能不完整 |
| 句子级截断 | 语义相对完整 | 实现稍复杂 |

## 5.3 边界与生产建议

- 截断要落在句子边界，避免半个句子误导模型。
- 每个片段保留来源标记（文档 ID），便于引用与审计。
- 剩余预算不足一个最小片段时，直接停止追加。

> 生产建议：先按重排分数排序，再按预算填充；高分段优先拿满，低分段才被截断。

# 6. 每任务 Token 预算

**结论先行**：Agent 任务可能多轮循环，必须给"整任务"设 Token 上限，防止失控。

## 6.1 预算对象设计

```java
public class TokenBudget {

    private final int maxTokensPerTask;
    private final AtomicInteger used = new AtomicInteger();

    public TokenBudget(int maxTokensPerTask) {
        this.maxTokensPerTask = maxTokensPerTask;
    }

    public boolean tryConsume(int tokens) {
        int current = used.addAndGet(tokens);
        if (current > maxTokensPerTask) {
            throw new BudgetExceededException(
                    "任务 Token 预算已耗尽: %d/%d"
                            .formatted(current, maxTokensPerTask));
        }
        return true;
    }

    public int remaining() {
        return Math.max(0, maxTokensPerTask - used.get());
    }
}
```

## 6.2 Agent 循环中的预算检查

```java
while (state.status() == AgentStatus.EXECUTING) {
    if (budget.remaining() < MIN_STEP_TOKENS) {
        log.warn("Token 预算不足，提前结束");
        return partialResult(state);
    }
    state = step(state);
}
```

## 6.3 并发下的预算一致性

- 多个子任务共享同一预算时，用 `AtomicInteger` 或分布式计数器保证原子性。
- 每个子任务应再设"子预算"，避免单步吃光全任务额度。
- 预算耗尽应返回"部分结果 + 原因"，而不是抛异常中断用户。

> 生产建议：预算粒度分三层——任务级、步骤级、单次调用级，逐层收敛。

# 7. 租户与用户配额

**结论先行**：Token 配额要按用户、租户、系统三层设置，并在请求前预检、请求后结算。

## 7.1 多级配额

```java
public record TokenQuota(
        long dailyTokens,
        long monthlyTokens,
        BigDecimal dailyCost
) {
}
```

| 维度 | 配额示例 |
|------|----------|
| 用户 | 10 万 token/天 |
| 租户 | 1000 万 token/天 |
| 系统 | 按成本预算 |

## 7.2 配额检查与原子性

```java
public void checkQuota(String tenantId, int estimatedTokens) {
    long usedToday = quotaService.usedToday(tenantId);

    if (usedToday + estimatedTokens > quota(tenantId).dailyTokens()) {
        throw new QuotaExceededException("今日 Token 配额已用尽");
    }
}
```

```java
// 原子预扣：先扣减，失败再回滚，避免并发超卖
boolean reserved = quotaStore.tryReserve(tenantId, estimatedTokens);
if (!reserved) {
    throw new QuotaExceededException("配额不足");
}
try {
    Usage usage = callModel(...);
    quotaStore.adjust(tenantId, estimatedTokens, actualTokens(usage));
} catch (Exception e) {
    quotaStore.release(tenantId, estimatedTokens);
    throw e;
}
```

## 7.3 配额耗尽策略

| 策略 | 适用 |
|------|------|
| 拒绝新请求 | 硬配额 |
| 降级小模型 | 软配额 |
| 排队延后 | 离线任务 |
| 通知升级 | SaaS 付费引导 |

> 生产建议：预扣 + 结算 + 回滚三件套，才能在高并发下不超卖配额。

# 8. 超限处理

**结论先行**：所有超限场景都应有明确降级路径，而不是让请求直接失败。

## 8.1 输入超限

| 场景 | 处理 |
|------|------|
| 输入超限 | 裁剪历史与检索结果 |
| 输出超限 | 提高 maxTokens 或要求精简 |
| 任务超预算 | 返回部分结果 |
| 租户超配额 | 拒绝新请求 |
| 成本超预算 | 降级到小模型 |

## 8.2 输出超限

- 输出被截断时，先判断是否 JSON 不完整，再决定重试。
- 结构化输出应预留足够 `max_tokens`，并做解析失败兜底。

## 8.3 降级链路

```java
public String handleWithBudget(String question, UserContext user) {
    try {
        return fullPipeline(question, user);
    } catch (BudgetExceededException e) {
        // 预算不足 → 降级：不用 RAG，直接回答
        return chatClient.prompt()
                .user("简要回答：" + question)
                .call().content();
    }
}
```

```
正常链路：RAG + 工具 + 大模型
   ↓ 预算不足
降级1：RAG + 小模型
   ↓ 仍不足
降级2：无 RAG + 小模型 + 精简提示
   ↓ 仍不足
兜底：缓存答案 / 明确拒绝并说明原因
```

> 生产建议：降级链路要提前设计并演练，不能等到线上 429 才临时补。

# 9. Token 监控

**结论先行**：没有 Token 监控就无法优化，监控要能按模型、租户、接口多维拆解。

## 9.1 指标采集

```java
// 每次调用记录
Usage usage = response.getMetadata().getUsage();
registry.counter("ai.tokens.input", "model", model, "tenant", tenant)
        .increment(usage.getPromptTokens());
registry.counter("ai.tokens.output", "model", model, "tenant", tenant)
        .increment(usage.getCompletionTokens());
```

## 9.2 告警规则

| 指标 | 告警 |
|------|------|
| 单任务 Token | 突增告警（防失控） |
| 租户日 Token | 超 80% 预警 |
| 输入输出比 | 异常比例告警 |
| 平均 Token/请求 | 持续上升需排查 |

## 9.3 归因分析

- 输入 Token 突增：检查检索片段数、历史是否未裁剪。
- 输出 Token 突增：检查提示是否要求啰嗦、是否陷入循环。
- 按接口拆分：定位是哪个功能在消耗 Token。

> 生产建议：监控面板同时展示"Token 量"与"Token 成本"，并与业务量做归一化对比。

# 10. 生产注意事项

**结论先行**：Token 预算化是一项系统工程，需要清单化落地与跨模块协同。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 忘记预留输出 | 必须预留 10%+ |
| 无 Token 估算 | 请求前估算 |
| 无任务预算 | 设每任务上限 |
| 无租户配额 | 按维度设配额 |
| 裁剪无优先级 | System 与问题优先保留 |
| 无监控 | 监控 Token 与成本 |

## 10.2 上线检查表

- [ ] 有 tokenizer 精确计数，且与计费版本一致
- [ ] 输出预留已配置且非零
- [ ] 任务级 / 步骤级预算已生效
- [ ] 租户 / 用户配额已配置且支持预扣
- [ ] 降级链路已实现并演练
- [ ] 监控与告警已接入

## 10.3 与成本 / 并发模块的协同

| 关联模块 | 协同点 |
|----------|--------|
| 成本优化 | Token 量直接换算成本 |
| 并发与批处理 | TPM 限制决定并发上限 |
| 缓存 | 命中可跳过 Token 消耗 |

> 生产建议：Token 预算、成本预算、并发上限三者应共享同一份"资源账本"。

# 11. 小结

- Token 决定成本、延迟与容量，必须预算化。
- 上下文按比例分配，**必须为输出预留**。
- 裁剪按优先级：System > 问题 > 最近对话 > 检索 > 早期。
- 每任务、每用户、每租户都要有 Token 配额。
- 超限要有降级策略，而非直接失败。
- 监控先行：能看见才能优化。
