# 1. 核心思想

**结论先行**：ToT（Tree of Thoughts，思维树）把线性推理改造成树状搜索，用多条候选路径与剪枝回溯，换取复杂难题更高的求解率。

## 1.1 一句话定义

ToT = 把推理组织成树，探索多条路径并回溯。

| 概念 | 英文 | 含义 |
|------|------|------|
| 节点 | Node / Thought | 一段中间推理或候选方案 |
| 层 | Layer / Depth | 搜索深度，即推理步数 |
| 分支因子 | Breadth | 每层生成的候选数量 |
| 剪枝 | Prune | 按评分保留 top-k，丢弃低分分支 |

```
               问题
                |
      +---------+---------+
      v         v         v
    思路A     思路B     思路C      <- 生成多个候选
      |         |
   +--+--+   +--+--+
   v     v   v     v
  A1    A2  B1    B2              <- 继续扩展
      |
      v
   评估选择最优路径                  <- 剪枝
```

## 1.2 机制拆解

**结论先行**：ToT 的核心不是"一次想得更长"，而是"多次想、比较后再选"。

### 1.2.1 思维节点的粒度

一个"思维"（thought）是推理过程中的一个连贯中间态，其粒度直接决定搜索空间大小。

| 粒度 | 示例 | 影响 |
|------|------|------|
| 粗（一步到位） | 整个解题方案 | 分支少、探索弱 |
| 中（一句一步） | "先排序，再双指针" | 平衡，工程常用 |
| 细（一词一步） | 单个算式 | 分支爆炸、成本高 |

工程实践中，粒度取"一句可独立评估的推理片段"最稳妥。

### 1.2.2 四类操作

ToT 把一次大推理拆成四类可独立优化的操作：

```
(1) 生成（Generate）：产生多个候选思路
(2) 评估（Evaluate）：给每个思路打分
(3) 选择（Select）：保留最优的几个
(4) 扩展（Expand）：继续向下展开
```

## 1.3 与 CoT 的区别

**结论先行**：CoT 是"一条路走到黑"，ToT 是"多路探索并择优"。

| 维度 | CoT | ToT |
|------|-----|-----|
| 结构 | 线性链 | 树状 |
| 路径 | 单条 | 多条 |
| 回溯 | 无 | 有 |
| 中间评估 | 无 | 有 |
| 剪枝 | 无 | 有 |
| 调用次数 | 1 | breadth × depth 量级 |
| 成本 / 延迟 | 低 | 高 |
| 适用 | 常规推理 | 需要探索的难题 |

**边界与坑**：若单链 CoT 已能稳定答对，引入 ToT 只会放大成本与延迟，不提升正确率。

**生产建议**：先用 CoT 建立基线；只在"单链推理反复失败"的题型上才启用 ToT。

# 2. 四个步骤详解

**结论先行**：生成决定探索广度，评估决定剪枝质量，选择决定资源分配，扩展决定搜索深度。

## 2.1 生成（Generate）

**结论先行**：生成的质量上限决定整棵树的解空间覆盖度。

### 2.1.1 采样式生成

同一提示下用较高温度多次采样，得到多样候选。优点是实现简单；缺点是候选同质化。

### 2.1.2 提议式生成

在提示中显式要求"给出 n 个不同方向"，一次调用返回多个候选，成本更低、多样性更好。

```java
public interface ThoughtGenerator {

    @UserMessage("""
            针对下面的问题，生成 {{n}} 个不同的解题思路。
            每个思路用一句话概括，不要展开细节，思路之间差异明显。

            问题：{{problem}}
            当前进展：{{progress}}

            只输出 JSON 数组：["思路1","思路2"]
            """)
    List<String> generate(@V("problem") String problem,
                          @V("progress") String progress,
                          @V("n") int n);
}
```

**边界与坑**：一次返回多条时，模型容易"凑数"，把同一思路换皮重复。

**生产建议**：提示中要求"差异明显"，并对候选做文本或语义去重。

## 2.2 评估（Evaluate）

**结论先行**：评估器是 ToT 的"裁判"，其可靠性直接决定剪枝是否正确。

### 2.2.1 独立打分

对每个候选单独打分，输出 0~1 的连续值或 1~10 的离散值。

```java
public interface ThoughtEvaluator {

    @UserMessage("""
            评估这个思路能否导向正确答案，给 0~1 分。
            0 = 完全错误，1 = 一定正确。

            问题：{{problem}}
            思路：{{thought}}

            只输出分数（如 0.8）。
            """)
    double evaluate(@V("problem") String problem,
                    @V("thought") String thought);
}
```

### 2.2.2 投票对比

让模型在若干候选中选出最优（或两两比较），比绝对打分更稳定。

| 方式 | 优点 | 缺点 |
|------|------|------|
| 独立打分 | 可并行、实现简单 | 分数不可比、易漂移 |
| 两两比较 | 更稳定、符合人类偏好 | 调用次数 O(n^2) |
| 投票选择 | 一次调用选 top-k | 结果粒度较粗 |

**边界与坑**：模型自评分数往往偏高且分布集中，直接取绝对值不可靠。

**生产建议**：对同批候选做归一化（softmax 或 min-max），或改用排名/比较式评估。

## 2.3 选择（Select）

**结论先行**：选择就是在每层保留 top-k，控制下一层的搜索宽度。

```java
public List<String> evaluateAndSelect(String problem,
                                      List<String> thoughts, int keep) {
    return thoughts.stream()
            .sorted(Comparator.comparingDouble(
                    (String t) -> -evaluator.evaluate(problem, t)))
            .limit(keep)
            .toList();
}
```

**边界与坑**：`sorted` 的比较器会对每个元素重复求值，造成重复的模型调用。

**生产建议**：先算一次分数并缓存，再排序，避免重复调用。

## 2.4 扩展（Expand）

**结论先行**：扩展把选中的思维当作新的上下文，继续生成下一层候选。

```java
public List<Node> expand(Node node, int breadth) {
    List<String> thoughts = generator.generate(
            node.state(), node.state(), breadth);
    return thoughts.stream()
            .map(t -> new Node(UUID.randomUUID().toString(), t,
                    node.id(), 0.0, node.depth() + 1))
            .toList();
}
```

**边界与坑**：扩展时不带父节点状态，模型会丢失上下文，重复生成已否定的方向。

**生产建议**：把"已走过的路径 + 已排除的思路"一并放进提示。

# 3. Java 实现

**结论先行**：把生成器、评估器、搜索策略解耦，才能独立替换与单测。

## 3.1 数据结构

### 3.1.1 Node 与结果类型

```java
public record Node(
        String id,
        String state,        // 当前累积的推理状态
        String parentId,
        double score,
        int depth) {
}
```

### 3.1.2 生成器与评估器接口

```java
public interface ThoughtGenerator {
    List<String> generate(String problem, String progress, int n);
}

public interface ThoughtEvaluator {
    double evaluate(String problem, String thought);
}
```

**生产建议**：定义为函数式接口，便于用 lambda 注入桩实现做单测。

## 3.2 评估器实现

**结论先行**：评估器应返回可比较的数值，并对异常做兜底。

```java
public class LlmThoughtEvaluator implements ThoughtEvaluator {

    private final ChatModel model;

    public LlmThoughtEvaluator(ChatModel model) {
        this.model = model;
    }

    @Override
    public double evaluate(String problem, String thought) {
        String prompt = """
                评估这个思路能否导向正确答案，给 0~1 分。

                问题：%s
                思路：%s

                只输出分数（如 0.8）。
                """.formatted(problem, thought);
        try {
            String raw = model.chat(prompt).trim();
            return Math.max(0.0, Math.min(1.0, Double.parseDouble(raw)));
        } catch (Exception e) {
            return 0.5;   // 解析失败给中性分，避免误剪
        }
    }
}
```

**边界与坑**：模型可能返回 "0.8 分"、"评分：0.8" 等非纯数字，直接解析会抛异常。

**生产建议**：用正则抽取第一个浮点数，并对结果做区间裁剪。

## 3.3 主循环

**结论先行**：主循环按层推进，每层生成、评估、剪枝，直至达到深度上限。

```java
public String treeOfThoughts(String problem, int depth, int breadth) {
    List<String> thoughts = generateThoughts(problem, breadth);

    for (int d = 1; d < depth; d++) {
        List<String> next = new ArrayList<>();

        for (String thought : thoughts) {
            next.addAll(generateThoughts(
                    problem + "\n当前思路：" + thought, breadth));
        }
        thoughts = evaluateAndSelect(problem, next, breadth);   // 剪枝
    }
    return selectBest(thoughts);
}
```

## 3.4 搜索策略

### 3.4.1 BFS 与 Beam Search

**结论先行**：BFS/Beam 每层保留 top-k，适合"步骤数确定"的问题。

```java
// Beam Search：每层保留 top-k
public String beamSearch(String problem, int depth, int beamWidth) {
    List<Node> beam = List.of(new Node("root", problem, null, 0.0, 0));

    for (int d = 0; d < depth; d++) {
        List<Node> candidates = new ArrayList<>();

        for (Node node : beam) {
            for (String thought : generateThoughts(node, beamWidth)) {
                double score = evaluator.evaluate(problem, thought);
                candidates.add(new Node(UUID.randomUUID().toString(),
                        thought, node.id(), score, node.depth() + 1));
            }
        }
        beam = candidates.stream()
                .sorted(Comparator.comparingDouble(Node::score).reversed())
                .limit(beamWidth)
                .toList();
    }
    return selectBest(beam);
}
```

### 3.4.2 DFS 与回溯

**结论先行**：DFS 深入一条路径，低于阈值即剪枝并回溯，适合"步数不定"的问题。

```java
public Optional<Node> dfs(Node node, int maxDepth, double threshold) {
    if (node.depth() >= maxDepth) {
        return Optional.of(node);
    }
    for (String child : generator.generate(node.state(), node.state(), 3)) {
        double score = evaluator.evaluate(node.state(), child);
        if (score < threshold) {
            continue;                       // 剪枝：低分不深入
        }
        Node next = new Node(UUID.randomUUID().toString(),
                child, node.id(), score, node.depth() + 1);
        Optional<Node> result = dfs(next, maxDepth, threshold);
        if (result.isPresent()) {
            return result;                  // 找到可行解即返回
        }
    }
    return Optional.empty();
}
```

### 3.4.3 策略对比

| 策略 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| BFS | 每层保留 top-k | 稳定、易并行 | 内存随宽度增长 |
| DFS | 深入一条，回溯 | 省内存、可能早停 | 易陷局部最优 |
| Beam Search | 每层保留 top-k 继续 | 平衡广度与深度 | 贪心，非全局最优 |

## 3.5 框架封装

**结论先行**：Spring AI 用 `entity()` 映射返回类型；LangChain4j 用声明式接口绑定模板。

```java
// Spring AI：ChatClient 直接映射为 List（泛型映射依赖框架版本）
List<String> thoughts = chatClient.prompt()
        .user("针对问题生成 3 个不同思路，只输出 JSON 数组：" + problem)
        .call()
        .entity(new ParameterizedTypeReference<List<String>>() {});
```

```java
// LangChain4j：AiServices 声明式接口
public interface TotAssistant {

    @SystemMessage("你是擅长分步推理的助手，只输出要求的 JSON。")
    @UserMessage("""
            针对问题生成 {{n}} 个不同思路，只输出 JSON 数组。
            问题：{{problem}}
            当前进展：{{progress}}
            """)
    List<String> generate(@V("problem") String problem,
                          @V("progress") String progress,
                          @V("n") int n);
}

// 装配
TotAssistant assistant = AiServices.create(TotAssistant.class, chatModel);
```

**生产建议**：把"生成""评估"分别封装为独立服务，便于替换模型、独立限流与重试。

# 4. 成本与预算

**结论先行**：ToT 成本随宽度与深度指数增长，必须在入口处做硬预算。

## 4.1 成本模型

```
成本 ≈ 生成次数 × 每次调用成本
生成次数 ≈ breadth^depth（指数增长）

depth=3, breadth=3 → 27 次生成 + 评估
```

**边界与坑**：成本是指数级的，depth 与 breadth 各加 1，调用量可能翻数倍。

## 4.2 参数建议

| 参数 | 建议 | 说明 |
|------|------|------|
| depth | 2~3 | 再深收益递减 |
| breadth | 2~3 | 宽度优先于深度 |
| 总调用 | 控制在 20 次内 | 含评估调用 |
| 阈值 | 0.5~0.7 | 低于阈值剪枝 |

## 4.3 预算与并发

**结论先行**：候选生成彼此独立，可并行；评估调用应批量或缓存。

**边界与坑**：并行会放大瞬时速率，容易触发服务端限流（rate limit）。

**生产建议**：并行度不超过 4；用信号量（Semaphore）限流；失败按指数退避重试。

# 5. 边界与反例

**结论先行**：ToT 不是万能药，误用会让成本翻倍而正确率不变。

## 5.1 不适合的场景

| 适合 | 不适合 |
|------|--------|
| 需要试错的难题 | 常规问答 |
| 有多种解法的规划 | 简单推理 |
| 24 点游戏类 | 事实查询 |
| 复杂决策 | 延迟敏感 |

## 5.2 常见失败模式

| 失败模式 | 表现 | 对策 |
|----------|------|------|
| 候选同质 | 多条思路几乎一样 | 提示要求差异 + 去重 |
| 评估失真 | 分数普遍偏高 | 用比较式评估 |
| 过度剪枝 | 好思路被早期误杀 | 提高阈值下限、保留探索 |

## 5.3 评估器陷阱

**结论先行**：让"生成者"自评，容易产生自我肯定偏差。

```java
// 反例：同一个模型既生成又评估，且提示不设边界
String best = model.chat("这些思路哪个最好？" + candidates);

// 正例：评估提示要求给出可验证的判据
String judge = model.chat("""
        按以下三个判据逐项打分，再给总分：
        1) 是否可行；2) 是否可验证；3) 是否覆盖边界情况。
        候选：%s
        """.formatted(candidates));
```

**生产建议**：生成与评估尽量用不同提示或不同模型；评估结果记录到日志用于回归。

# 6. 简化版 ToT

**结论先行**：多数业务不需要完整树搜索，"生成多个 + 选一个"就能拿到大部分收益。

## 6.1 生成 3 选 1

不必完整实现 ToT，可用简化策略：

```java
// 生成 3 个方案 → 评估 → 选最好的执行
public String simplifiedToT(String problem) {
    List<String> candidates = generateThoughts(problem, 3);

    // 让模型直接选最好的
    String best = model.chat("""
            下面是针对问题的 3 个方案，选出最好的并说明理由：

            问题：%s

            方案1：%s
            方案2：%s
            方案3：%s

            输出：最佳方案编号 + 理由
            """.formatted(problem, candidates.get(0),
                    candidates.get(1), candidates.get(2)));

    return execute(problem, best);
}
```

## 6.2 自一致性投票

**结论先行**：对同一问题多次采样，取出现次数最多的答案，用多数表决抵消随机错误。

```java
public String selfConsistency(String problem, int samples) {
    Map<String, Long> votes = new HashMap<>();
    for (int i = 0; i < samples; i++) {
        String answer = normalize(model.chat(problem, /*temperature=*/ 0.7));
        votes.merge(answer, 1L, Long::sum);
    }
    return votes.entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElseThrow();
}
```

## 6.3 简化版与完整版对比

| 维度 | 简化版（3 选 1） | 自一致性 | 完整 ToT |
|------|:----------------:|:--------:|:--------:|
| 调用次数 | 4 | n | breadth^depth |
| 实现复杂度 | 低 | 低 | 高 |
| 收益 | 约 80% | 中 | 高 |
| 适用 | 多数业务 | 有确定答案 | 难题探索 |

> 简化版能用 20% 的成本获得 80% 的收益。

# 7. 生产注意事项

**结论先行**：ToT 上生产的关键是"可预算、可观测、可评测"。

## 7.1 风险与对策

| 风险 | 建议 |
|------|------|
| 成本指数增长 | 限制 depth/breadth |
| 用于简单任务 | 按需使用 |
| 无剪枝 | 每层评估选择 |
| 延迟高 | 异步或简化 |
| 无评测 | 验证是否真提升 |

## 7.2 可观测性与评测

**结论先行**：ToT 必须记录搜索轨迹，并用对照实验证明其相对 CoT 的增益。

**生产建议**：记录 `requestId`、总调用数、每层 top-k 分数与剪枝原因；超预算请求降级为 CoT。

| 评测方法 | 说明 |
|----------|------|
| A/B 对照 | 同题分别跑 CoT 与 ToT，比较正确率与成本 |
| 固定测试集 | 用难题集（如 24 点、规划题）回归 |
| 成本核算 | 记录 token 与调用次数 |
| 人工抽检 | 检查剪枝是否误杀 |

**生产建议**：以"单位正确率的成本"为北极星指标，而不是只看正确率。

# 8. 小结

## 8.1 要点回顾

- ToT 把推理组织成树，探索多路径并剪枝。
- 四步：生成、评估、选择、扩展。
- 搜索策略：BFS、DFS、Beam Search。

## 8.2 落地清单

- 成本随 depth/breadth 指数增长，需严格限制。
- 简化版（生成 3 选 1）与自一致性性价比高。
- 上生产要可预算、可观测、可评测。
