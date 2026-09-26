# 1. 压缩的核心权衡

上下文压缩（context compression）是在不损害任务质量的前提下，
减少送入模型的 Token 数量。

## 1.1 为什么要压缩

结论先行：模型上下文窗口（context window）有限且昂贵，
长会话、多工具调用会让上下文迅速膨胀，压缩是必选项。

### 1.1.1 核心公式

```
压缩率 ↑  →  Token ↓  →  信息损失 ↑  →  质量 ↓
```

目标：**在质量可接受的前提下，最大化压缩率**。

### 1.1.2 不压缩的代价

| 不压缩的后果 | 说明 |
|--------------|------|
| 超出窗口 | 请求被拒或截断 |
| 成本上升 | 每 Token 计费 |
| 延迟上升 | 输入越长越慢 |
| 质量下降 | 噪声稀释关键信息 |

## 1.2 压缩率与信息损失的权衡

结论先行：压缩不是"压得越狠越好"，
而是在质量约束下找压缩率的上限。

| 压缩率 | 典型手段 | 风险 |
|:------:|----------|------|
| 低（10%~30%） | 去重、去寒暄 | 几乎无损 |
| 中（30%~60%） | 关键句、结构化 | 轻微损失 |
| 高（60%~90%） | 摘要、语义压缩 | 易丢细节 |
| 极高（>90%） | 一句话题要 | 只保主线 |

## 1.3 压缩的时机

结论先行：压缩应在"触发阈值"处异步进行，
而非每轮同步压缩，否则延迟不可接受。

```
消息累积 ──▶ 估算 Token ──▶ 超软限？──▶ 异步压缩 ──▶ 替换上下文
                                │否
                                └──▶ 直接使用
```

# 2. 五种压缩技术

## 2.1 技术总览

结论先行：五类技术从"简单粗暴"到"语义等价"依次递进，
可靠性与成本同步上升。

### 2.1.1 技术清单

| 技术 | 压缩率 | 信息损失 | 成本 |
|------|:------:|:--------:|:----:|
| 截断 | 高 | 高 | 无 |
| 摘要 | 中高 | 中 | 中 |
| 结构化抽取 | 中 | 低 | 中 |
| 关键句选择 | 中 | 中 | 低 |
| 语义压缩 | 中高 | 中 | 高 |

## 2.2 技术对比与选型

| 技术 | 保留原文 | 需模型 | 可验证性 | 适用 |
|------|:--------:|:------:|:--------:|------|
| 截断 | 是（部分） | 否 | 高 | 兜底 |
| 摘要 | 否 | 是 | 低 | 一般对话 |
| 结构化抽取 | 否 | 是 | 高 | 任务型 |
| 关键句选择 | 是 | 是 | 中 | 信息密集 |
| 语义压缩 | 否 | 是 | 低 | 长叙事 |

选型建议：任务型 Agent 优先"结构化抽取"，
对话型优先"关键句 + 摘要"，极端预算用"截断"兜底。

选择决策树：

```
需要保留精确数字/ID？
 ├─ 是 ──▶ 结构化抽取（+ 正则兜底）
 └─ 否 ──▶ 内容信息密度高？
            ├─ 是 ──▶ 关键句选择
            └─ 否 ──▶ 叙事为主？
                       ├─ 是 ──▶ 语义压缩 / 摘要
                       └─ 否 ──▶ 截断（兜底）
```

生产建议：多技术组合往往优于单一技术——
先用选择性保留去掉寒暄与冗余，再对剩余内容做结构化抽取。

# 3. 结构化抽取

## 3.1 为什么推荐结构化抽取

结论先行：结构化抽取（structured extraction）把对话抽取为
"事实、决策、实体、未解决项"，比自由摘要更可靠，不会丢关键变量。

```
自由摘要："讨论了退款，用户同意等 30 天。"
        ← 可能丢掉订单号、金额

结构化抽取：
  facts:     ["退款时限 30 天"]
  entities:  {orderId: ORD-001, amount: 100}
  decisions: ["按 30 天处理"]
        ← 关键变量被强制保留
```

## 3.2 完整实现

比摘要更可靠，不会丢关键变量：

### 3.2.1 压缩接口

```java
public record CompressedContext(
        List<String> facts,             // 事实
        List<String> decisions,         // 决策
        Map<String, String> entities,   // 实体（订单号、金额）
        List<String> openIssues,        // 未解决
        String narrative                // 简短叙述
) {}

interface ContextCompressor {
    @UserMessage("""
            压缩下面的对话，保留所有关键信息。

            必须保留：
            - 所有数字、ID、金额、日期
            - 已达成的结论
            - 用户的约束与偏好
            - 未解决的问题

            对话：
            {{conversation}}

            输出 JSON：
            {
              "facts": [],
              "decisions": [],
              "entities": {},
              "openIssues": [],
              "narrative": ""
            }
            """)
    CompressedContext compress(String conversation);
}
```

### 3.2.2 渲染实现

Spring AI 风格的调用与渲染：

```java
public String compressAndRender(List<Message> messages) {
    CompressedContext ctx = compressor.compress(format(messages));

    StringBuilder sb = new StringBuilder();
    ctx.entities().forEach(
            (k, v) -> sb.append(k).append("=").append(v).append("；"));
    sb.append("\n决策：").append(String.join("；", ctx.decisions()));
    sb.append("\n未解决：").append(String.join("；", ctx.openIssues()));
    sb.append("\n").append(ctx.narrative());
    return sb.toString();
}
```

## 3.3 边界与坑

| 风险 | 后果 | 对策 |
|------|------|------|
| 模型漏抽实体 | 关键信息丢失 | 正则兜底抽取数字/ID |
| 幻觉新增事实 | 错误信息 | 校验抽取结果 |
| JSON 解析失败 | 压缩中断 | 重试 + 降级摘要 |
| 结构过复杂 | 模型难遵循 | 字段精简、给示例 |

生产建议：对数字、订单号、金额等，
用正则从原文再抽取一遍并与模型结果取并集。

# 4. 关键句选择

## 4.1 机制

结论先行：关键句选择（key sentence selection）让模型"挑选"而非"重写"，
因此不会改写失真，但压缩率有限。

用模型选出最关键的句子，而非重写：

### 4.1.1 选择接口

```java
interface KeySentenceSelector {
    @UserMessage("""
            从对话中选出最重要的 10 句话（保持原文），按重要性排序。

            对话：
            {{conversation}}

            输出 JSON 数组：["句子1","句子2"]
            """)
    List<String> select(String conversation);
}
```

## 4.2 优缺点

| 优点 | 缺点 |
|------|------|
| 保留原文，无改写失真 | 压缩率有限 |
| 成本低 | 可能漏掉隐含信息 |

## 4.3 边界与坑

- 句子切分不当会把"条件 + 结论"拆开，导致语义残缺。
- 只选 10 句对长对话不够，应按预算动态决定数量。
- 排序不稳定时，需固定种子或改用打分排序。

# 5. 分层压缩

## 5.1 分层模型

结论先行：不同时间距离的消息用不同压缩强度——
越近越完整，越远越抽象。

```
最近 5 轮：原文
5~20 轮：关键句
20 轮以上：结构化摘要
全部：一句话题要
```

| 层级 | 时间距离 | 形式 | 目的 |
|------|----------|------|------|
| 近期 | 最近 5 轮 | 原文 | 连贯 |
| 中期 | 5~20 轮 | 关键句 | 平衡 |
| 早期 | 20 轮以上 | 结构化摘要 | 降本 |
| 全局 | 全部 | 一句话 | 主线 |

## 5.2 实现

### 5.2.1 分层实现

```java
public String hierarchicalCompress(List<Message> messages) {
    int n = messages.size();
    StringBuilder sb = new StringBuilder();

    // ① 全部：一句话
    sb.append("【整体】").append(oneLineSummary(messages)).append("\n\n");

    // ② 20 轮以上：结构化摘要
    if (n > 20) {
        CompressedContext ctx = compress(
                format(messages.subList(0, n - 20)));
        sb.append("【早期】").append(render(ctx)).append("\n\n");
    }

    // ③ 5~20 轮：关键句
    if (n > 5) {
        List<String> keys = select(format(
                messages.subList(Math.max(0, n - 20), n - 5)));
        sb.append("【中期】").append(String.join("；", keys)).append("\n\n");
    }

    // ④ 最近 5 轮：原文
    sb.append("【近期】\n").append(format(messages.subList(Math.max(0, n - 5), n)));

    return sb.toString();
}
```

## 5.3 边界与坑

- 分界点（5、20）是经验值，需按平均消息长度调整。
- 边界处可能切断一个"问答对"，应尽量按轮对齐。
- 早期摘要失真后会影响全局，需定期用原文重算。

# 6. 语义压缩

## 6.1 机制

结论先行：语义压缩（semantic compression）把多句合并为等价的紧凑表达，
保持事实不变、只压缩表达。

把多句合并为等价的紧凑表达：

```
原文：
  "用户问报销时限。我说 30 天。用户说太短了。我解释这是公司规定。用户接受了。"

语义压缩：
  "报销时限 30 天（公司规定），用户已接受。"
```

## 6.2 实现

### 6.2.1 压缩接口

```java
interface SemanticCompressor {
    @UserMessage("""
            把下面的对话压缩为等价的紧凑表述，
            保持所有事实、数字、结论不变，只压缩表达。

            对话：{{conversation}}
            压缩后：
            """)
    String compress(String conversation);
}
```

## 6.3 边界与坑

| 风险 | 后果 | 对策 |
|------|------|------|
| 改写引入歧义 | 语义漂移 | 关键句保留原文 |
| 丢失语气/立场 | 误判 | 保留"接受/拒绝"等词 |
| 反复压缩 | 累积失真 | 始终基于原文压缩 |

生产建议：语义压缩只用于"叙事性"内容，
事实、数字、约束一律走结构化抽取。

# 7. 选择性保留

## 7.1 策略表

结论先行：不同内容用不同压缩策略，是最简单有效的降本方式。

| 内容 | 策略 |
|------|------|
| 用户明确约束 | 原文保留 |
| 数字/ID/日期 | 原文保留 |
| 结论/决策 | 结构化保留 |
| 寒暄 | 丢弃 |
| 重复内容 | 去重 |
| 工具详细结果 | 只留关键字段 |

## 7.2 实现

### 7.2.1 选择性实现

```java
public String selectiveCompress(List<Message> messages) {
    StringBuilder sb = new StringBuilder();

    for (Message m : messages) {
        if (isGreeting(m)) continue;                      // 丢弃寒暄
        if (containsCriticalData(m)) {
            sb.append("[原文] ").append(m.text()).append("\n");
        } else if (isToolResult(m)) {
            sb.append("[结果] ").append(extractKeyFields(m)).append("\n");
        } else {
            sb.append("[摘要] ").append(shorten(m)).append("\n");
        }
    }
    return sb.toString();
}
```

| 前缀 | 含义 | 模型解读 |
|------|------|----------|
| [原文] | 逐字保留 | 高可信 |
| [结果] | 工具字段 | 结构化 |
| [摘要] | 压缩改写 | 参考 |

边界与坑：前缀标记会占用 Token，
但对模型理解"哪部分是原文"很有帮助，通常值得。

# 8. 压缩触发与分层

## 8.1 触发条件

结论先行：用"软限/硬限"双阈值控制压缩强度，避免频繁压缩。

| 阈值 | 含义 | 动作 |
|------|------|------|
| SOFT_LIMIT | 软限 | 轻度压缩 |
| HARD_LIMIT | 硬限 | 重度压缩 |

```
Token 量
  │                        ┌──── 硬限：结构化 + 分层
  ├────────────────────────┤
  │        软限：关键句     │
  ├────────────────────────┤
  │        不压缩           │
  └────────────────────────┴────▶
```

## 8.2 自适应实现

### 8.2.1 双阈值实现

```java
public class AdaptiveCompressor {

    public List<Message> maybeCompress(
            String sessionId, List<Message> messages) {
        int tokens = estimateTokens(messages);

        if (tokens < SOFT_LIMIT) {
            return messages;                              // 未超软限，不压缩
        }

        if (tokens < HARD_LIMIT) {
            // 轻度压缩：关键句选择
            return lightCompress(messages);
        }

        // 重度压缩：结构化 + 分层
        return heavyCompress(messages);
    }

    private static final int SOFT_LIMIT = 6000;
    private static final int HARD_LIMIT = 12000;
}
```

## 8.3 边界与坑

- 阈值应按"模型窗口 − 输出预留 − 系统提示"动态计算，而非硬编码。
- 压缩本身消耗 Token，需评估"压缩收益 > 压缩成本"。
- 压缩后要重新估算 Token，防止压缩结果仍超限。

# 9. 压缩质量验证

## 9.1 验证指标

结论先行：压缩必须验证，核心是"关键实体与决策是否保留"。

| 指标 | 含义 | 目标 |
|------|------|------|
| 实体完整率 | 实体保留比例 | 100% |
| 决策保留率 | 决策保留比例 | 100% |
| 压缩率 | Token 降幅 | 越高越好 |

## 9.2 实现

### 9.2.1 质量评估

```java
public CompressionQuality evaluate(List<Message> original,
                                   List<Message> compressed) {
    CompressedContext before = extractState(original);
    CompressedContext after = extractState(compressed);

    return new CompressionQuality(
            // 实体完整率
            entityPreservation(before.entities(), after.entities()),
            // 决策保留率
            decisionPreservation(before.decisions(), after.decisions()),
            // 压缩率
            1.0 - estimateTokens(compressed)
                    / (double) estimateTokens(original));
}
```

### 9.2.2 单元测试

```java
@Test
void compression_must_preserve_entities() {
    List<Message> compressed = compressor.compress(longConversation);
    CompressedContext state = extractState(compressed);

    assertTrue(state.entities().containsKey("orderId"));
    assertEquals("A123", state.entities().get("orderId"));
}
```

## 9.3 边界与坑

> **关键实体（订单号、金额）丢失是压缩最严重的事故**，必须验证。

- 只测一个样例不够，需构造含多实体的回归集。
- 验证用"抽取对比"，若抽取器本身有漏，会误判。
- 压缩率与保留率需同时看，只追压缩率会牺牲质量。

生产建议：把"实体保留率"设为发布门禁，低于 100% 不允许上线。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 摘要丢关键信息 | 结构化抽取 |
| 压缩成本高 | 小模型 + 异步 |
| 压缩率不足 | 分层压缩 |
| 无验证 | 校验关键实体 |
| 多次压缩失真 | 保留原始快照 |
| 同步压缩增延迟 | 异步 |

## 10.2 生产建议

结论先行：压缩是"用一点成本换大量 Token"的交易，
必须保证"永不丢失关键信息"这条底线。

- 原始快照：压缩前保留完整原文，压缩失败可回退。
- 小模型：压缩用便宜模型（如小参数模型）即可。
- 异步化：在后台线程压缩，用户请求不等压缩完成。
- 可观测：记录压缩率、实体保留率、压缩耗时。

```java
// 异步压缩 + 回退
public CompletableFuture<String> compressAsync(List<Message> messages) {
    return CompletableFuture
            .supplyAsync(() -> compressor.compress(format(messages)), executor)
            .thenApply(this::render)
            .exceptionally(ex -> format(messages));   // 失败回退原文
}
```

# 11. 小结

## 11.1 核心要点

- 压缩是压缩率与信息损失的权衡。
- 五种技术：截断、摘要、结构化抽取、关键句、语义压缩。
- 结构化抽取最可靠，能保证关键信息不丢。

## 11.2 工程要点

- 分层压缩：近期原文、中期关键句、早期摘要。
- 压缩后必须验证关键实体与决策未丢失。
