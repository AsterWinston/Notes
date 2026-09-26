# 1. 核心思想

## 1.1 定义与直觉

### 1.1.1 一句话定义

**自洽性（Self-Consistency）** 是一种解码（Decoding）策略：对同一问题采样多次，取出现次数最多的答案。

结论先行：它用"多条推理路径的一致性"替代"单条路径的可靠性"。

### 1.1.2 与单次推理的区别

| 维度 | 单次推理 | Self-Consistency |
|------|----------|------------------|
| 采样次数 | 1 | N |
| 决策依据 | 单条路径 | 多数表决 |
| 稳定性 | 受随机性影响 | 更稳定 |
| 成本 | 1x | Nx |

> 直觉：一次可能"运气好"，也可能"运气差"；多次取众数，就是"平均掉运气"。

## 1.2 一个可复现的例子

### 1.2.1 采样与投票

```
问题 → 采样 N 次（temperature 提高）→ 得到 N 个答案 → 多数表决
```

```
问题：一个班 40 人，60% 是女生，女生有多少人？
采样1：24
采样2：24
采样3：24
采样4：16   ← 少数
采样5：24
多数表决 → 24（正确）
```

### 1.2.2 结果解读

- 正确路径被多次独立复现，说明它更"稳"。
- 错误路径各不相同，难以形成多数。
- 结论：投票能过滤随机错误，但无法过滤"系统性一致错误"。

## 1.3 适用前提

结论先行：Self-Consistency 只在"答案空间可枚举"时才成立。

```
若答案为自由文本（如一段摘要），无法比较"是否相同"，
投票就失去意义。
```

# 2. 为什么有效

## 2.1 错误分布的直觉

```
单次推理可能走错路
多次推理：
- 正确路径更容易重复出现
- 错误路径往往各不相同
- 多数表决能过滤随机错误
```

### 2.1.1 正确路径更集中

当模型"大致会做"时，正确推理路径的生成概率最高，多次采样中会反复出现。

### 2.1.2 错误路径更分散

错误来自各种随机偏差，彼此不一致，因此难以累积成多数。

## 2.2 与 CoT 的关系

| 组合 | 效果 |
|------|------|
| 仅 Self-Consistency | 对短答案有效 |
| CoT + Self-Consistency | 效果最佳 |
| 仅 CoT | 单链，可能走偏 |

> Self-Consistency 通常与 CoT 联合使用：每条链先推理，再对答案投票。

## 2.3 概率视角

```
设单次正确率为 p，独立采样 N 次：
- 只要正确路径的出现概率高于任何单条错误路径，
  多数表决的正确率随 N 增大而提升。
- 但若所有错误都指向同一错误答案，投票反而放大错误。
```

边界与坑：模型若存在"系统性偏差"，多次采样只会重复同一错误。

| 情形 | 投票效果 |
|------|----------|
| 正确路径概率最高 | 提升 |
| 错误路径更集中 | 可能放大错误 |
| 路径彼此独立 | 提升明显 |

生产建议：投票前先确认任务不存在强系统性偏差，必要时引入外部工具验证。

# 3. 实现

## 3.1 基础实现

### 3.1.1 采样循环

```java
public String selfConsistent(String question, int samples) {
    List<String> answers = IntStream.range(0, samples)
            .mapToObj(i -> model.chat(ChatRequest.builder()
                    .messages(UserMessage.from(question))
                    .options(ChatOptions.builder()
                            .temperature(0.7)          // 提高多样性
                            .build())
                    .build()))
            .map(r -> extractAnswer(r))
            .toList();

    return mostCommon(answers);
}
```

答案抽取辅助方法：

```java
private String extractAnswer(String raw) {
    // 约定模型输出 "答案：X"
    int idx = raw.lastIndexOf("答案：");
    String ans = (idx >= 0) ? raw.substring(idx + 3) : raw;
    return ans.trim().toLowerCase();   // 归一化，便于比较
}
```

### 3.1.2 多数表决

```java
private String mostCommon(List<String> answers) {
    return answers.stream()
            .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
            .entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElseThrow();
}
```

## 3.2 Java 完整示例（Spring AI）

### 3.2.1 并发采样

```java
@Service
public class SelfConsistencyService {

    private final ChatClient chatClient;
    private final ExecutorService pool = Executors.newFixedThreadPool(8);

    public SelfConsistencyService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String vote(String question, int samples) {
        List<CompletableFuture<String>> futures = IntStream.range(0, samples)
                .mapToObj(i -> CompletableFuture.supplyAsync(
                        () -> sampleOnce(question), pool))
                .toList();

        List<String> answers = futures.stream()
                .map(CompletableFuture::join)
                .toList();
        return mostCommon(answers);
    }

    private String sampleOnce(String question) {
        String raw = chatClient.prompt()
                .user(question)
                .options(ChatOptions.builder().temperature(0.7).build())
                .call()
                .content();
        return extractAnswer(raw);
    }
}
```

### 3.2.2 投票聚合

```
采样并发执行 → 收集答案 → 归一化（去空格/大小写）→ 计数 → 取众数
```

```java
private String mostCommon(List<String> answers) {
    return answers.stream()
            .collect(Collectors.groupingBy(
                    a -> a.replaceAll("\\s+", "").toLowerCase(),
                    Collectors.counting()))
            .entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElseThrow();
}
```

## 3.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 答案格式不一 | "24" 与 "24 人" | 归一化 |
| 并发过高 | 触发限流 | 信号量限流 |
| 异常吞没 | 单次失败影响整体 | 容错与重试 |

# 4. 与 CoT 结合

## 4.1 组合方式

```
CoT + Self-Consistency：
① 每次采样都带 CoT（分步推理）
② 对最终答案投票（不是对推理过程投票）
```

### 4.1.1 对答案投票

不同推理路径可能得到相同答案，投票应在"答案"层面进行。

### 4.1.2 不对推理投票

推理文本几乎不会完全一致，对推理投票没有意义。

## 4.2 Java 实现

```java
interface CotSolver {
    @UserMessage("""
            逐步推理后给出最终答案，格式：
            推理：...
            答案：X
            问题：{{it}}
            """)
    String solve(String question);
}

public String solveConsistent(String question, int samples) {
    List<String> answers = IntStream.range(0, samples)
            .mapToObj(i -> extractFinalAnswer(solver.solve(question)))
            .toList();
    return mostCommon(answers);
}
```

> **对答案投票，而非对推理投票**——不同推理路径可能得到相同答案。

## 4.3 常见误区

| 误区 | 问题 |
|------|------|
| 对推理文本投票 | 文本难一致，投票无意义 |
| 只投答案不归一化 | 同一答案被拆成多类 |
| temperature 过低 | 所有链几乎相同，投票退化 |

# 5. 适用场景

## 5.1 适合

| 适合 | 原因 |
|------|------|
| 有唯一正确答案的推理题 | 可投票 |
| 数学计算 | 答案可枚举 |
| 逻辑判断 | 答案离散 |
| 分类（边界模糊时） | 类别有限 |
| 答案可枚举 | 众数有意义 |

> 判断标准：能否为"答案相同"定义一个可靠的判定函数。若能，才适合投票。

## 5.2 不适合

| 不适合 | 原因 |
|--------|------|
| 开放生成 | 无唯一答案 |
| 创意写作 | 需要多样性 |
| 摘要 | 答案连续 |
| 需要多样性 | 投票抹平差异 |

## 5.3 决策清单

```
答案是否可枚举？
  否 → 不适用
  是 → 单次是否已足够稳定？
          是 → 不用（浪费成本）
          否 → 采样 3~5 次投票
```

> 关键前提：**答案空间可枚举**，才能投票。

# 6. 采样参数

## 6.1 temperature 与 top-p

### 6.1.1 temperature

```java
// 提高 temperature 增加多样性
ChatOptions.builder()
        .temperature(0.7)          // 0.5~1.0
        .build();
```

temperature 过低会导致所有采样几乎相同，投票退化为单次推理。

### 6.1.2 top-p

| 参数 | 建议 |
|------|------|
| temperature | 0.5~1.0（太低则答案都一样） |
| top-p | 0.9~0.95（配合 temperature） |
| 采样数 | 3~10（越多越准，成本越高） |
| 答案抽取 | 结构化输出便于提取 |

## 6.2 采样数

| 采样数 | 相对成本 | 收益 |
|:------:|:--------:|------|
| 1 | 1x | 基线 |
| 3 | 3x | 明显提升 |
| 5 | 5x | 小幅提升 |
| 10 | 10x | 边际递减 |

## 6.3 答案抽取

```
要求模型输出固定格式：
  推理：...
  答案：X
再用正则或结构化输出提取 X。
```

| 方式 | 可靠性 | 成本 |
|------|:------:|:----:|
| 正则提取 | 中 | 低 |
| 结构化输出 | 高 | 低 |
| 二次模型抽取 | 高 | 高 |

生产建议：优先用结构化输出（Structured Output）直接返回答案字段，减少解析错误。

# 7. 加权投票

## 7.1 原理

不同采样可以有不同的权重：对推理质量更高的样本赋予更高权重。

```
加权投票 = Σ 权重(样本) 按答案分组求和 → 取最高分
```

## 7.2 Java 实现

```java
public String weightedVote(List<Sample> samples) {
    Map<String, Double> scores = new HashMap<>();

    for (Sample s : samples) {
        // 用模型对推理质量打分，作为权重
        double weight = confidence(s.reasoning());
        scores.merge(s.answer(), weight, Double::sum);
    }
    return scores.entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElseThrow();
}
```

## 7.3 边界

| 边界 | 说明 |
|------|------|
| 权重来源 | 若由模型自评，可能同样有偏 |
| 复杂度 | 需额外评分调用，成本更高 |
| 收益 | 通常仅在平票或边界样本上有用 |

# 8. 成本权衡

## 8.1 成本模型

```
成本 = 单次成本 × 采样数
准确率提升：通常 5%~15%（因任务而异）
```

| 采样数 | 相对成本 | 收益 |
|:------:|:--------:|------|
| 1 | 1x | 基线 |
| 3 | 3x | 明显提升 |
| 5 | 5x | 小幅提升 |
| 10 | 10x | 边际递减 |

## 8.2 性价比拐点

> **3~5 次通常是性价比拐点**。

```
收益
  ^
  |        ______
  |      /        \___
  |    /              \___
  |  /
  +------------------------> 采样数
    1   3   5      10
```

## 8.3 优化手段

| 手段 | 说明 |
|------|------|
| 早停 | 某答案领先多数时提前结束 |
| 并发 | 并行采样降低总延迟 |
| 分级 | 只对高价值问题启用 |
| 缓存 | 相同问题复用结果 |

```java
// 早停：当某答案票数超过剩余采样数时，可提前结束
if (topVotes > remaining) {
    break;
}
```

# 9. 何时不值得

## 9.1 任务层面

| 情况 | 说明 |
|------|------|
| 答案开放 | 无法投票 |
| 任务简单 | 一次就对 |
| 模型已很稳定 | temperature=0 就够 |

## 9.2 系统层面

| 情况 | 说明 |
|------|------|
| 延迟敏感 | 多次采样太慢 |
| 成本敏感 | 成本翻倍 |
| 高并发 | 配额压力大 |

## 9.3 反例

```
问题：把这句话翻译成英文。
→ 答案开放且多样，投票无意义。

问题：今天天气如何？
→ 事实查询，一次即可，多次采样浪费。
```

| 反例类型 | 为什么无效 |
|----------|------------|
| 开放翻译 | 多解，无众数 |
| 事实查询 | 一次即准 |
| 创意文案 | 多样性被抹平 |
| 低价值请求 | 成本不成比例 |

# 10. 生产注意事项

## 10.1 成本与并发

### 10.1.1 并发采样

```java
Semaphore gate = new Semaphore(16);   // 限制并发
```

### 10.1.2 限流

| 风险 | 建议 |
|------|------|
| 成本翻倍 | 只用于高价值任务 |
| 采样数过多 | 3~5 次 |
| 并发过高 | 信号量限流 |

## 10.2 一致性判定

| 风险 | 建议 |
|------|------|
| 用于开放生成 | 不适用 |
| temperature 太低 | 0.5~1.0 |
| 答案抽取困难 | 结构化输出 |
| 平票 | 定义兜底策略（如取首个或降级） |

```java
// 平票兜底：返回首个出现次数最多的答案，并记录告警
```

| 场景 | 兜底策略 |
|------|----------|
| 平票 | 取首次出现者或降级为单次结果 |
| 全部不同 | 视为不稳定，触发人工/工具复核 |
| 无答案 | 返回错误并记录 |

## 10.3 评测与灰度

| 风险 | 建议 |
|------|------|
| 无评测 | 验证是否真提升 |
| 全量上线 | 灰度 + A/B |

```
灰度：比较 1 次 vs N 次的准确率/成本/延迟，达标再放量。
```

# 11. 小结

- Self-Consistency 通过多次采样 + 多数表决提升稳定性。
- 对答案投票，而非对推理投票。
- 与 CoT 结合效果最好。
- 只适用于"答案可枚举"的任务。
- 采样数 3~5 通常性价比最高。
- 需注意成本翻倍与系统性偏差两类风险。
