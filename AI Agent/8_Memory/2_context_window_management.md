# 1. 上下文窗口是稀缺资源

**结论先行**：上下文窗口（Context Window）是模型单次请求能处理的 Token 上限，它是记忆系统最硬的约束。所有记忆策略最终都要回答同一个问题：在有限窗口内放什么。

## 1.1 窗口的构成

```text
上下文窗口 = System + 工具定义 + 记忆 + 检索 + 问题 + 输出预留
```

### 1.1.1 固定部分

System 提示（System Prompt）与工具定义（Tool Definition）通常每次请求都一样，属于固定开销。它们不可裁剪，必须优先满足。

### 1.1.2 可变部分

检索上下文（Retrieval）、对话历史（History）、长期记忆（Long-term Memory）随请求变化，是预算调度的主要对象。

### 1.1.3 输出预留

模型生成的回答也要占用窗口。若不预留，输入占满窗口后输出会被截断，甚至直接报错。

### 1.1.4 Token 计数

预算是按 Token 计算的，因此必须先有 Token 估算器。常见做法有三种：调用厂商的分词接口、使用本地分词器（如 jtokkit）、或按字符数粗略估算。

```java
public interface TokenEstimator {
    int estimate(String text);
}

public class HeuristicTokenEstimator implements TokenEstimator {
    @Override
    public int estimate(String text) {
        if (text == null || text.isEmpty()) {
            return 0;
        }
        // 粗略启发式：中文约 1 字 1 token，英文约 4 字符 1 token
        int cjk = 0;
        int other = 0;
        for (int i = 0; i < text.length(); i++) {
            if (text.charAt(i) > 0x2E80) {
                cjk++;
            } else {
                other++;
            }
        }
        return cjk + (int) Math.ceil(other / 4.0);
    }
}
```

> 启发式估算有 10%~30% 误差，生产环境应使用与模型一致的分词器，并留安全余量。

## 1.2 超限的后果

| 后果 | 表现 |
|------|------|
| 报错 | 请求被拒绝（400/422） |
| 截断 | 内容被静默裁掉，语义不完整 |
| 成本飙升 | 输入 Token 计费随长度线性增长 |
| 延迟增加 | 长上下文推理更慢 |

**边界与坑**：不同厂商对超限的处理不同，有的直接报错，有的静默截断，有的丢弃中间部分。不能依赖"它会自动处理"。

## 1.3 生产建议

- 在应用层显式做 Token 预算，不依赖模型侧兜底。
- 为固定部分与可变部分分别设定上限。
- 输出预留不得低于窗口的 15%~25%。

# 2. 预算分配

**结论先行**：把窗口当作一笔预算，按用途切分为若干桶（Bucket），并为输出强制预留。最常见的错误是忘记预留输出。

## 2.1 静态预算

```text
窗口 128K 的分配示例：
├── System 提示         5K    （固定）
├── 工具定义            8K    （固定）
├── 检索上下文          40K   （可变）
├── 对话历史            35K   （可变）
├── 长期记忆            10K   （按需）
├── 当前问题            2K
└── 输出预留            28K   ← 必须预留！
```

## 2.2 预算代码实现

```java
public record ContextBudget(
        int total,
        int system,
        int tools,
        int retrieval,
        int history,
        int longTerm,
        int question,
        int outputReserve
) {
    public static ContextBudget standard(int windowSize) {
        return new ContextBudget(
                windowSize,
                (int) (windowSize * 0.04),
                (int) (windowSize * 0.06),
                (int) (windowSize * 0.30),
                (int) (windowSize * 0.27),
                (int) (windowSize * 0.08),
                (int) (windowSize * 0.03),
                (int) (windowSize * 0.22));
    }

    public int variableBudget() {
        return retrieval + history + longTerm;
    }

    public int fixedBudget() {
        return system + tools;
    }
}
```

不同窗口规模下的比例参考：

| 窗口 | System | 工具 | 检索 | 历史 | 长期 | 问题 | 输出预留 |
|------|--------|------|------|------|------|------|----------|
| 8K | 0.5K | 0.5K | 2K | 2K | 0.5K | 0.5K | 2K |
| 32K | 1.5K | 2K | 9K | 9K | 2.5K | 1K | 7K |
| 128K | 5K | 8K | 40K | 35K | 10K | 2K | 28K |

## 2.3 预算校验

预算切分后应做自洽校验，避免配置错误导致输出被挤占。

```java
public static void validate(ContextBudget budget) {
    int allocated = budget.system() + budget.tools() + budget.retrieval()
            + budget.history() + budget.longTerm() + budget.question()
            + budget.outputReserve();
    if (allocated > budget.total()) {
        throw new IllegalStateException(
                "预算超配：" + allocated + " > " + budget.total());
    }
    if (budget.outputReserve() < budget.total() * 0.15) {
        throw new IllegalStateException("输出预留不足，至少 15%");
    }
}
```

## 2.4 边界与生产建议

- 各桶比例应按业务实测调优，不要照搬示例。
- 预留部分要区分"输出"与"下一轮输入增长"。
- 预算配置应集中管理，支持热更新。

# 3. 消息裁剪

**结论先行**：当消息历史超出预算时，从最旧的消息开始丢弃，优先保留最新内容。这是最简单也最常用的裁剪策略。

## 3.1 从新到旧裁剪

```java
public List<Message> trimToFit(List<Message> messages, int budget) {
    int used = 0;
    LinkedList<Message> result = new LinkedList<>();

    // 从最新往旧保留
    for (int i = messages.size() - 1; i >= 0; i--) {
        Message msg = messages.get(i);
        int tokens = TokenEstimator.estimate(msg.text());

        if (used + tokens > budget) break;

        result.addFirst(msg);
        used += tokens;
    }
    return result;
}
```

## 3.2 完整裁剪实现

简单裁剪会产生一个问题：工具调用（Tool Call）与工具结果（Tool Result）成对出现，若只保留其一，模型会报错或困惑。

```java
public List<Message> trimSafely(List<Message> messages, int budget) {
    int used = 0;
    LinkedList<Message> result = new LinkedList<>();
    for (int i = messages.size() - 1; i >= 0; i--) {
        Message msg = messages.get(i);
        int tokens = TokenEstimator.estimate(msg.text());
        if (used + tokens > budget) {
            break;
        }
        result.addFirst(msg);
        used += tokens;
        // 若保留的是工具结果，必须把其请求也一并保留
        if (msg instanceof ToolExecutionResultMessage && i > 0
                && messages.get(i - 1) instanceof AiMessage ai
                && ai.hasToolCalls()) {
            int pairTokens = TokenEstimator.estimate(ai.text());
            if (used + pairTokens <= budget) {
                result.addFirst(ai);
                used += pairTokens;
                i--;
            }
        }
    }
    return result;
}
```

## 3.3 裁剪的边界情况

| 情况 | 风险 | 处理 |
|------|------|------|
| 单条消息超预算 | 裁剪后为空 | 截断该消息或拒绝 |
| 工具对断裂 | 模型报错 | 成对保留 |
| System 被裁 | 规则丢失 | 永不裁剪 |
| 最新问题被裁 | 无法回答 | 永不裁剪 |

## 3.4 裁剪策略对比

| 策略 | 粒度 | 优点 | 缺点 |
|------|------|------|------|
| 按条裁剪 | 单条消息 | 实现简单 | 易破坏工具对 |
| 按轮裁剪 | 一问一答 | 语义完整 | 粒度较粗 |
| 按 Token | 精确预算 | 利用率高 | 需分词器 |
| 按摘要 | 压缩早期 | 省 Token | 丢失细节 |

## 3.5 边界与生产建议

- 裁剪以"轮"为单位比以"条"为单位更安全。
- 保留被裁消息的摘要，避免信息完全丢失。
- 裁剪后做一次 Token 复核，防止估算偏差。

# 4. 裁剪优先级

**结论先行**：裁剪不是无差别丢弃，而是按优先级：System > 当前问题 > 最近对话 > 高相关检索 > 早期对话。

## 4.1 优先级顺序

```text
必须保留（按顺序）：
① System 提示（规则不可丢）
② 当前问题
③ 最近的对话（至少最近 1~2 轮）
④ 高相关检索片段

可裁剪：
⑤ 早期对话
⑥ 低相关检索片段
⑦ 长期记忆（按需）
```

## 4.2 组装实现

```java
public List<Message> assemble(ContextBudget budget, ContextParts parts) {
    List<Message> result = new ArrayList<>();

    // ① 必留
    result.add(SystemMessage.from(parts.system()));
    result.add(UserMessage.from(parts.question()));

    // ② 检索（按分数排序，取到预算）
    result.addAll(selectTop(parts.retrievalDocs(), budget.retrieval()));

    // ③ 历史（从近到远，取到预算）
    result.addAll(recentWithin(parts.history(), budget.history()));

    // ④ 长期记忆（按需）
    if (budget.longTerm() > 0) {
        result.addAll(selectRelevant(parts.longTermMemory(), budget.longTerm()));
    }
    return result;
}
```

## 4.3 边界与生产建议

- 优先级应可配置，不同业务侧重不同。
- 检索片段按分数截断，而非简单截断条数。
- 记录每次裁剪丢弃了什么，便于复盘。

# 5. 位置与顺序

**结论先行**：模型对上下文的注意力分布不均，开头与结尾更容易被记住。关键信息应前置或后置，检索结果居中。

## 5.1 中间迷失现象

```text
模型的注意力分布不均：
开头与结尾的信息更容易被记住（"中间迷失"现象）
```

```text
注意力强度示意：
高 ┤■                                   ■
   │■                                 ■ ■
低 ┤■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
   └──────────────────────────────────────▶
     开头            中间            结尾
```

## 5.2 优化策略

| 策略 | 说明 |
|------|------|
| 关键信息前置 | System、核心约束放开头 |
| 问题后置 | 用户问题放最后 |
| 检索结果居中 | 相关片段放中间 |
| 重复强调 | 关键约束在结尾再提一次 |

## 5.3 模板实现

```java
String prompt = """
        %s        ← System 规则（开头）
        %s        ← 检索上下文（中间）
        %s        ← 历史（中间）
        %s        ← 用户问题（结尾）
        %s        ← 关键约束重申（结尾）
        """.formatted(system, context, history, question, keyReminder);
```

使用消息列表时，同样遵循"前置规则、居中检索、后置问题"的顺序。

```java
public List<Message> positionAware(List<Message> retrieval,
                                   List<Message> history,
                                   String question,
                                   String keyReminder) {
    List<Message> messages = new ArrayList<>();
    // 开头：系统规则与关键约束
    messages.add(SystemMessage.from(SYSTEM_PROMPT));
    messages.add(SystemMessage.from("核心约束：" + keyReminder));
    // 中间：检索与历史
    messages.addAll(retrieval);
    messages.addAll(history);
    // 结尾：问题 + 约束重申
    messages.add(UserMessage.from(question));
    messages.add(SystemMessage.from("再次强调：" + keyReminder));
    return messages;
}
```

## 5.4 边界与生产建议

- "中间迷失"因模型而异，需实测验证。
- 不要为了前置而打乱逻辑顺序，可能引入困惑。
- 结尾重申应简短，避免占用过多预算。

# 6. 动态预算

**结论先行**：固定预算无法适配所有请求。应按问题复杂度与是否需要检索动态分配，把预算给最需要的地方。

## 6.1 复杂度分类

| 请求类型 | 检索需求 | 历史需求 | 输出需求 |
|----------|----------|----------|----------|
| 简单问答 | 低 | 低 | 低 |
| 复杂分析 | 高 | 中 | 高 |
| 多轮对话 | 低 | 高 | 中 |
| 长文档问答 | 高 | 低 | 中 |

## 6.2 动态分配实现

```java
public ContextBudget dynamicBudget(String question, int windowSize) {
    boolean isComplex = complexityClassifier.isComplex(question);
    boolean needRetrieval = needsRetrieval(question);

    if (!needRetrieval) {
        // 不需要检索 → 把预算给历史与输出
        return new ContextBudget(windowSize, 5_000, 8_000, 0, 60_000, 10_000, 2_000, 43_000);
    }
    if (isComplex) {
        // 复杂问题 → 给检索更多
        return new ContextBudget(windowSize, 5_000, 8_000, 60_000, 20_000, 10_000, 2_000, 23_000);
    }
    return ContextBudget.standard(windowSize);
}
```

复杂度分类器可以是规则，也可以是小模型。规则版示例：

```java
public boolean isComplex(String question) {
    if (question.length() > COMPLEX_LENGTH) {
        return true;
    }
    if (question.contains("分析") || question.contains("对比")
            || question.contains("为什么")) {
        return true;
    }
    return false;
}

public boolean needsRetrieval(String question) {
    return question.contains("文档") || question.contains("资料")
            || question.contains("根据") || question.contains("查一下");
}
```

## 6.3 边界与生产建议

- 分类器本身有误差，应为误判保留回退路径。
- 动态调整要有上下界，防止某桶被压到 0。
- 记录每类请求的实际预算使用，用于持续调优。

# 7. 超限处理

**结论先行**：超限应优雅降级而非直接失败：能裁剪则裁剪，能精简则精简，实在不行才拒绝并给出明确提示。

## 7.1 三类超限场景

| 场景 | 处理 |
|------|------|
| 输入超限 | 裁剪历史/检索 |
| 输出超限 | 提高 maxTokens / 要求精简 |
| 无法裁剪 | 拒绝并提示 |

## 7.2 优雅降级实现

```java
public String handle(String question, List<Message> history) {
    int estimated = estimateTotal(question, history);

    if (estimated > MAX_INPUT_TOKENS) {
        // 先裁剪
        history = trimToFit(history, MAX_INPUT_TOKENS - estimateQuestion(question));

        if (estimateTotal(question, history) > MAX_INPUT_TOKENS) {
            return "对话过长，请开启新会话或精简问题。";
        }
    }
    return chat(question, history);
}
```

## 7.3 输出超限的处理

输出超限通常发生在长文本生成场景，处理方式与输入超限不同。

```java
public String chatWithOutputGuard(String question, List<Message> history) {
    ChatOptions options = ChatOptions.builder()
            .maxTokens(OUTPUT_RESERVE)
            .build();
    try {
        return chatClient.prompt()
                .messages(history)
                .options(options)
                .call()
                .content();
    } catch (OutputLimitExceededException ex) {
        // 降级：要求模型精简或分点输出
        return chatClient.prompt()
                .messages(history)
                .options(ChatOptions.builder().maxTokens(OUTPUT_RESERVE).build())
                .user("请用不超过 200 字分点总结。")
                .call()
                .content();
    }
}
```

## 7.4 边界与生产建议

- 降级路径要有明确用户提示，而非静默失败。
- 输出超限可通过"要求分点、限制字数"缓解。
- 拒绝时保留原始输入，便于用户复制重试。

# 8. 监控

**结论先行**：上下文管理必须可观测，否则预算不足、截断、超限等问题会在生产环境中静默累积。

## 8.1 关键指标

| 指标 | 告警 |
|------|------|
| 平均上下文 Token | 持续上升需排查 |
| 截断比例 | 高说明预算不足 |
| 超限次数 | > 0 需关注 |
| 输出预留利用率 | 长期偏低说明预留过多 |

## 8.2 埋点实现

```java
registry.summary("context.tokens.used").record(totalTokens);
registry.summary("context.tokens.truncated").record(truncatedTokens);
registry.counter("context.overflow").increment();      // 超限次数
registry.summary("context.output.reserve.used").record(outputTokens);
```

## 8.3 告警策略

```text
截断比例 > 5%      ──▶ 警告：检查预算配置
超限次数 > 0/小时  ──▶ 严重：检查裁剪逻辑
平均 Token 上升   ──▶ 提示：检查记忆增长与遗忘
```

## 8.4 生产建议

- 指标按租户、模型、接口维度打标签。
- 保留超限请求的样本用于离线分析。
- 建立预算配置的变更审计。

# 9. 生产注意事项

**结论先行**：上下文管理的风险集中在"忘记预留、裁剪无优先级、忽略位置、固定预算、无监控、直接报错"六类。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 忘记预留输出 | 预留 20%+ |
| 裁剪无优先级 | System 与问题必留 |
| 忽略位置效应 | 关键信息前置/后置 |
| 固定预算 | 按复杂度动态分配 |
| 无监控 | 监控 Token 与截断 |
| 超限直接报错 | 优雅降级 |

## 9.2 实施清单

- 建立统一 Token 估算器，统一口径。
- 预算配置化并可热更新。
- 裁剪逻辑覆盖工具调用成对约束。
- 接入监控与告警。
- 定期复盘截断与超限样本。
- 为不同模型维护独立的窗口与预算配置。
- 上线前用真实流量做 Token 压测，验证裁剪与降级路径。

# 10. 小结

## 10.1 核心结论

- 上下文窗口是稀缺资源，必须预算化。
- 分配要考虑固定部分、可变部分与输出预留。
- 裁剪按优先级：System > 问题 > 最近对话 > 检索。
- 注意"中间迷失"，关键信息前置/后置。

## 10.2 实践要点

- 超限要优雅处理，并持续监控。
- 动态预算让资源流向最需要的部分。
- 工具调用消息必须成对保留。
