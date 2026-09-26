# 1. 优化的基本原则

**结论先行**：提示优化（Prompt Optimization）本质是工程迭代，必须以评测数据驱动，而非凭直觉改写。

## 1.1 三条铁律

```
没有评测，就没有优化。
用数据迭代，而非凭感觉。
一次只改一个变量。
```

| 铁律 | 反例 | 后果 |
|------|------|------|
| 评测驱动 | 凭感觉改措辞 | 无法判断好坏 |
| 单变量 | 同时改提示与模型 | 无法归因 |
| 有基线 | 不记录当前得分 | 无法比较 |

## 1.2 评测驱动

**结论先行**：先定义指标，再谈优化；指标定义不清，优化就是无头苍蝇。

### 1.2.1 指标定义

| 指标 | 含义 | 用途 |
|------|------|------|
| 准确率 | 正确回答占比 | 主指标 |
| 格式合规率 | 可解析比例 | 工程指标 |
| 平均成本 | 每请求 token 费用 | 成本约束 |
| P95 延迟 | 尾部耗时 | 体验约束 |
| 安全率 | 未触发违规比例 | 底线 |

### 1.2.2 指标权衡

```
准确率 ↑ 但成本 ↑↑  → 需按业务定取舍
```

## 1.3 单变量原则

**结论先行**：一次只改一个变量，否则无法知道是哪个改动带来收益。

```java
// ❌ 同时改：措辞 + 示例 + 模型 + 温度
// ✅ 一次只改一项，其余保持不变
```

## 1.4 边界与坑

- **测试集泄漏**：把测试样例写进提示，得分虚高。
- **指标单一**：只看准确率，忽略成本与延迟。
- **样本太少**：10 条样例的差异多为噪声。

**生产建议**：评测集至少 50 条，覆盖各类失败模式与边界输入。

# 2. 手动优化流程

**结论先行**：手动优化是一条固定流水线，核心是"归因—修改—验证"闭环。

## 2.1 六步流程

### 2.1.1 流程图

```
① 建评测集（50+ 条）
② 跑基线，记录指标
③ 分析失败案例，归类原因
④ 针对性修改提示
⑤ 重新评测，对比基线
⑥ 通过则保留，否则回退
```

## 2.2 Java 实现

```java
public void optimizeIteratively(List<EvalCase> cases) {
    PromptTemplate current = loadPrompt("v1");
    double baseline = evaluate(cases, current);

    for (String candidate : generateCandidates(current)) {
        double score = evaluate(cases, PromptTemplate.from(candidate));

        if (score > baseline + 0.02) {          // 显著提升才采纳
            current = PromptTemplate.from(candidate);
            baseline = score;
            log.info("采纳新版本，得分 {} → {}", baseline, score);
        }
    }
    savePrompt("v2", current);
}
```

## 2.3 采纳阈值

**结论先行**：提升幅度必须超过噪声阈值才采纳，避免追逐随机波动。

| 提升幅度 | 决策 |
|----------|------|
| > 5% | 直接采纳 |
| 2%~5% | 扩大评测集验证 |
| < 2% | 视为噪声，不采纳 |

## 2.4 边界与坑

- **回退缺失**：改了没变好却不回退，版本越改越差。
- **评测集固定**：长期只用一套集，会过拟合。
- **忽略方差**：单次评测波动大，应多次取均值。

**生产建议**：把采纳阈值、评测集版本、提示版本一并写入实验记录。

# 3. 失败归因

**结论先行**：优化前先分类失败原因，不同失败对应不同提示对策。

## 3.1 失败类型与对策

| 失败类型 | 提示层面的对策 |
|----------|----------------|
| 格式错误 | 明确输出格式 + 结构化输出 |
| 答非所问 | 明确任务 |
| 遗漏信息 | 提供更完整上下文 |
| 编造内容 | 强化"只依据资料" |
| 推理错误 | 加 CoT |
| 分类错误 | 加示例 + 枚举 |

## 3.2 归因代码

```java
public void analyzeFailures(List<FailureCase> failures) {
    Map<String, Long> categories = failures.stream()
            .collect(Collectors.groupingBy(
                    this::categorize, Collectors.counting()));

    categories.entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .forEach(e -> log.info("{}: {}", e.getKey(), e.getValue()));
    // 针对最多的失败类型优化
}
```

## 3.3 归因维度

### 3.3.1 分层归因

| 层次 | 问题 | 对策方向 |
|------|------|----------|
| 输入层 | 上下文缺失/噪声 | 改检索与组装 |
| 指令层 | 任务不明确 | 改提示措辞 |
| 示例层 | 示例不足/误导 | 增删示例 |
| 模型层 | 能力不足 | 换模型或拆任务 |
| 输出层 | 解析失败 | 结构化输出 |

## 3.4 边界与坑

- **归因偏差**：只看失败样本，忽略"侥幸成功"的样本。
- **分类粗**：一个"格式错误"桶里混了多种根因。
- **不采样**：只统计数量，不留原始输入输出。

**生产建议**：对每个失败类别保留 3~5 条原始样本，供离线复现。

# 4. APE 自动提示工程

**结论先行**：APE（Automatic Prompt Engineering，自动提示工程）用模型生成候选提示，再由评测筛选。

## 4.1 原理

```
任务描述 + 示例 → LLM 生成 N 个候选 → 评测打分 → 选最优
```

## 4.2 Java 实现

```java
interface PromptGenerator {
    @UserMessage("""
            根据以下任务描述，生成 3 个不同的提示词候选。
            每个候选应清晰、具体、可执行。

            任务：{{task}}
            示例输入输出：{{examples}}

            输出 JSON 数组：["提示1","提示2","提示3"]
            """)
    List<String> generate(@V("task") String task,
                          @V("examples") String examples);
}

public PromptTemplate autoOptimize(String task, List<EvalCase> cases) {
    List<String> candidates = generator.generate(task, formatExamples(cases));

    return candidates.stream()
            .map(PromptTemplate::from)
            .max(Comparator.comparingDouble(p -> evaluate(cases, p)))
            .orElseThrow();
}
```

## 4.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 候选同质 | 生成高度相似 | 要求多样化 |
| 过拟合 | 只对评测集最优 | 留出黄金集 |
| 成本 | 生成+评测开销大 | 离线批处理 |

**生产建议**：APE 产出的候选仍需人工审阅，防止生成带偏见或违规措辞。

# 5. DSPy 思想

**结论先行**：DSPy 把提示优化从"手写字符串"变成"声明式编译"，由编译器自动搜索最优提示与示例。

## 5.1 优化即编译

### 5.1.1 四要素

```
① 定义签名（输入 → 输出）
② 定义模块（如何用 LLM）
③ 定义指标（如何评估）
④ 编译器自动搜索最优提示/示例
```

| 要素 | 作用 |
|------|------|
| 签名 Signature | 声明输入输出契约 |
| 模块 Module | 封装调用策略（如 CoT） |
| 指标 Metric | 量化好坏 |
| 优化器 Optimizer | 搜索提示与示例 |

## 5.2 Java 概念映射

```java
// 概念映射到 Java
public interface Signature<I, O> {
    String instruction();            // 指令
    I input();
    O output();
}

// 模块
public class ChainOfThought<I, O> {
    public O forward(I input) {
        // 加 CoT 提示
        throw new UnsupportedOperationException("伪代码");
    }
}

// 优化器
public class BootstrapFewShot {
    public Prompt optimize(Module module, List<EvalCase> trainset) {
        // 自动选择最佳示例
        throw new UnsupportedOperationException("伪代码");
    }
}
```

> Java 生态暂无成熟 DSPy 实现，但可借鉴其"优化即编译"的思想。

## 5.3 与 APE 对比

| 维度 | APE | DSPy |
|------|-----|------|
| 优化对象 | 提示文本 | 提示 + 示例 + 流程 |
| 驱动方式 | 生成候选 | 指标编译搜索 |
| 人工介入 | 较多 | 较少 |
| 可复现性 | 中 | 较高 |

## 5.4 边界与坑

- **指标即目标**：指标定义错了，编译器会优化出错误行为。
- **算力成本**：编译需大量调用，适合离线。
- **框架锁定**：更换框架时迁移成本高。

**生产建议**：先把指标和评测集做扎实，再引入自动优化。

# 6. 示例自动选择

**结论先行**：与其固定几个示例，不如按查询动态检索最相关的示例，提升少样本效果。

## 6.1 检索式示例选择

```
查询 → 在示例池中检索相似示例 → 取 TopK → 注入提示
```

## 6.2 Java 实现

```java
public String selectBestExamples(List<EvalCase> pool, EvalCase query, int k) {
    // ① 检索相似案例
    List<EvalCase> similar = pool.stream()
            .sorted(Comparator.comparingDouble(
                    (EvalCase c) -> -similarity(c, query)))
            .limit(k * 2)
            .toList();

    // ② 从候选中选"最有助于正确回答"的
    return similar.stream().limit(k)
            .map(this::formatExample)
            .collect(Collectors.joining("\n"));
}
```

## 6.3 边界与坑

| 坑 | 说明 | 对策 |
|----|------|------|
| 示例泄漏答案 | 示例与测试过近 | 隔离示例池 |
| 相似但不同 | 语义近、标签异 | 加标签一致性校验 |
| 数量过多 | 占用大量预算 | 控制在 3~5 个 |

**生产建议**：示例池与评测集物理隔离，防止示例泄漏导致虚高。

# 7. 提示压缩

**结论先行**：压缩能降低 token 成本，但必须验证效果不降，否则得不偿失。

## 7.1 压缩策略

| 策略 | 说明 |
|------|------|
| 去冗余表达 | 删客套与重复 |
| 合并同类约束 | 多条合成一条 |
| 结构化改写 | 段落变列表 |
| 模型压缩 | 用 LLM 改写 |

## 7.2 Java 实现

```java
interface PromptCompressor {
    @UserMessage("""
            压缩下面的提示词，保持语义与效果不变，去除冗余表达。
            只输出压缩后的提示。

            原提示：{{it}}
            """)
    String compress(String prompt);
}
```

> 压缩能降本，但**必须验证效果不降**。

## 7.3 边界与坑

- **丢失关键约束**：压缩把边界条件删掉，行为改变。
- **过度压缩**：提示过短反而降低遵从度。
- **压缩不可逆**：应保留原版以便回退。

**生产建议**：压缩前后跑同一评测集，效果下降超过阈值即回退。

# 8. 优化效果的验证

**结论先行**：优化必须可验证，验证必须同时看效果与成本。

## 8.1 验证维度

| 维度 | 检查点 |
|------|--------|
| 效果 | 准确率不降 |
| 成本 | 平均 token 不升 |
| 稳定性 | 多次运行方差可控 |
| 安全 | 未引入违规输出 |

## 8.2 Java 实现

```java
@Test
void optimization_should_improve() {
    EvalReport before = evaluateWith("v1");
    EvalReport after = evaluateWith("v2");

    assertTrue(after.accuracy() >= before.accuracy(),
            "优化后准确率未提升");
    assertTrue(after.avgCost().compareTo(before.avgCost()) <= 0,
            "优化后成本上升");
}
```

## 8.3 边界与坑

- **只跑一次**：单次结果受随机性影响，应多次取均值。
- **只测效果**：忽略成本上升会侵蚀收益。
- **无对照**：没有基线版本就无法比较。

**生产建议**：把验证用例纳入 CI，提示变更时自动跑评测门禁。

# 9. 常见优化陷阱

**结论先行**：优化最大的风险不是没提升，而是"看似提升实则过拟合或不可归因"。

## 9.1 陷阱表

| 陷阱 | 说明 |
|------|------|
| 过拟合测试集 | 保留未参与优化的测试集 |
| 改多个变量 | 无法归因 |
| 只看平均 | 关注各分类表现 |
| 忽视成本 | 优化也要看成本 |
| 无基线 | 无法判断好坏 |
| 反复微调 | 边际递减 |

## 9.2 过拟合

**结论先行**：反复用同一测试集调参，模型会"记住"测试集而非真正泛化。

```
训练集（可看）  →  调参
验证集（可看）  →  选版本
黄金集（不看）  →  最终验收
```

## 9.3 边际递减

**结论先行**：优化到一定程度后收益递减，应转向数据、检索或模型升级。

| 阶段 | 收益 |
|------|------|
| 初版 → 结构清晰 | 大幅提升 |
| 加示例、加 CoT | 中等提升 |
| 反复措辞微调 | 边际递减 |

**生产建议**：当连续多轮优化提升均低于噪声阈值时，停止并复盘方向。

# 10. 生产注意事项

**结论先行**：提示优化的生产重点是"数据驱动、单变量、防过拟合、版本化"。

## 10.1 风险与对策

| 风险 | 建议 |
|------|------|
| 凭感觉优化 | 用评测数据 |
| 无评测集 | 先建集 |
| 多变量同改 | 一次一个 |
| 过拟合 | 保留黄金集 |
| 忽视成本 | 一并度量 |
| 无版本 | 版本化管理 |

## 10.2 版本化与回归

```java
public record PromptVersion(
        String name,
        String content,
        double accuracy,
        double avgCost,
        String evalSetVersion) {
}
```

## 10.3 落地清单

- 评测集分层：训练 / 验证 / 黄金。
- 每次优化只改一个变量并留实验记录。
- 提示与评测集一起版本化，支持一键回退。
- 效果、成本、安全三线并行验收。

# 11. 小结

## 11.1 要点回顾

- 提示优化必须评测驱动，一次只改一个变量。
- 流程：建集 → 基线 → 归因 → 修改 → 对比。
- APE 用模型生成候选，DSPy 把优化变成编译。

## 11.2 落地清单

- 示例自动选择与提示压缩是常用手段。
- 保留未参与优化的测试集防过拟合。
- 效果与成本一起度量，并做版本化管理。
