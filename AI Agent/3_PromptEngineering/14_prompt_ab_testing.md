# 1. 为什么需要 A/B

## 1.1 直觉不可靠

```
"感觉新 Prompt 更好" ≠ "确实更好"
可能改好了 A 但改坏了 B
```

Prompt 的效果是概率性的。修改者通常只看到自己关心的几个样例，容易产生"确认偏误"：记住了改好的例子，忽略了改坏的例子。

## 1.2 A/B 测试的本质

A/B 测试用**数据**判断哪个版本更好。它的本质是控制变量实验：

- 只改变 Prompt 这一个变量，其余保持一致。
- 用同一批样本或同一批流量对比两个版本。
- 用统计检验判断差异是否显著，而非凭感觉。

| 决策方式 | 依据 | 风险 |
|----------|------|------|
| 拍脑袋 | 个人经验 | 高，易反复 |
| 看几个样例 | 少量案例 | 高，样本偏差 |
| A/B 测试 | 统计指标 | 低，可复现 |

## 1.3 适用与不适用场景

| 场景 | 是否适合 A/B |
|------|--------------|
| 措辞、结构、示例数量调整 | 适合 |
| 更换模型 | 适合 |
| 全新功能上线 | 先离线，再线上 |
| 极低频场景 | 样本不足，谨慎 |
| 合规、安全类改动 | 不适用，直接强制 |

# 2. 两种 A/B

## 2.1 离线 A/B 与线上 A/B

| 类型 | 数据 | 特点 |
|------|------|------|
| 离线 A/B | 固定评测集 | 快、可控 |
| 线上 A/B | 真实流量 | 真实、慢 |

> 先离线筛，再线上验。

## 2.2 先离线后线上的流程

### 2.2.1 离线筛选

在固定评测集上快速对比多个候选版本，淘汰明显更差的方案，把候选收敛到 1 到 2 个。

### 2.2.2 线上验证

把离线胜出的版本以极小流量（如 5%）上线，观察真实业务指标，确认无回归后再逐步放量。

```
离线对比 → 选出候选 → 小流量线上 → 放量 → 全量
   (分钟)                (天)          (天)
```

## 2.3 两者的指标差异

| 维度 | 离线 | 线上 |
|------|------|------|
| 准确率 | 有标注，可算 | 需人工或代理指标 |
| 延迟 | 可测 | 受真实负载影响 |
| 成本 | 可估算 | 真实计费 |
| 用户满意度 | 无 | 点赞 / 追问 |
| 迭代速度 | 快 | 慢 |

# 3. 离线 A/B

## 3.1 对比实验框架

```java
public ComparisonReport compare(PromptTemplate a, PromptTemplate b,
                                List<EvalCase> cases) {
    List<EvalResult> resultsA = evaluate(a, cases);
    List<EvalResult> resultsB = evaluate(b, cases);

    return ComparisonReport.builder()
            .accuracyA(accuracy(resultsA))
            .accuracyB(accuracy(resultsB))
            .latencyA(avgLatency(resultsA))
            .latencyB(avgLatency(resultsB))
            .costA(avgCost(resultsA))
            .costB(avgCost(resultsB))
            .significant(isSignificant(resultsA, resultsB))
            .build();
}
```

关键点：两个版本必须跑同一批 `cases`，且评测过程要可重复（固定随机种子、固定温度参数）。

## 3.2 对比报告模型

### 3.2.1 ComparisonReport

```java
public record ComparisonReport(
        double accuracyA, double accuracyB,
        long latencyA, long latencyB,
        BigDecimal costA, BigDecimal costB,
        boolean significant
) {
    public String verdict() {
        if (!significant) return "无显著差异";
        if (accuracyB > accuracyA) return "B 更优";
        return "A 更优";
    }
}
```

### 3.2.2 verdict 判定

| 条件 | 结论 |
|------|------|
| 不显著 | 无显著差异，维持现状 |
| 显著且 B 准确率更高 | B 更优 |
| 显著且 A 准确率更高 | A 更优 |
| 显著但成本上升明显 | 需权衡，不一定采纳 |

## 3.3 评测集设计要点

| 要点 | 说明 |
|------|------|
| 覆盖边界 | 空输入、超长输入、歧义输入 |
| 分类均衡 | 每类样本量相当 |
| 避免污染 | 评测集不进训练 / 示例 |
| 可复现 | 固定版本、固定参数 |
| 定期更新 | 防止过拟合评测集 |

# 4. 指标对比

## 4.1 核心指标

| 指标 | 说明 |
|------|------|
| 准确率 | 核心质量指标 |
| 延迟 | 用户体验 |
| 成本 | 经济性 |
| 格式合规率 | 结构化输出稳定性 |
| 拒答率 | 无答案时正确拒答 |

## 4.2 指标取舍

多个指标常常相互冲突：更长推理链可能提升准确率，但增加延迟与成本。取舍原则：

- 先设定硬约束（如延迟不超过基线 20%），在约束内优化主指标。
- 主指标只能有一个，避免"什么都想要"。
- 次要指标用于解释，不用于决策。

| 变更方向 | 准确率 | 延迟 | 成本 |
|----------|--------|------|------|
| 增加示例 | 升 | 升 | 升 |
| 增加推理步骤 | 升 | 升 | 升 |
| 精简提示 | 不确定 | 降 | 降 |

## 4.3 回归测试断言

```java
@Test
void compare_prompts() {
    ComparisonReport report = comparator.compare(v1, v2, cases);
    report.print();

    assertTrue(report.accuracyB() >= report.accuracyA() - 0.02,
            "v2 准确率下降");
    assertTrue(report.costB().compareTo(report.costA()) <= 0,
            "v2 成本上升");
}
```

断言应把"允许的容差"写进代码，例如准确率允许下降 2 个百分点，超出即构建失败。

# 5. 分维度对比

## 5.1 分维度聚合

```java
public Map<String, ComparisonReport> compareByCategory(
        PromptTemplate a, PromptTemplate b, List<EvalCase> cases) {

    return cases.stream()
            .collect(Collectors.groupingBy(EvalCase::category))
            .entrySet().stream()
            .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    e -> compare(a, b, e.getValue())));
}
```

## 5.2 典型案例：整体提升某类下降

| 类别 | A 准确率 | B 准确率 | 差异 |
|------|----------|----------|------|
| 常识 | 0.80 | 0.86 | +0.06 |
| 数学 | 0.72 | 0.78 | +0.06 |
| 代码 | 0.75 | 0.60 | -0.15 |
| 合计 | 0.76 | 0.77 | +0.01 |

> **整体提升但某类下降**是常见情况，分维度对比能发现。上表中整体只涨 1 个点，但代码类大跌 15 个点，若只看整体就会误判。

## 5.3 维度选择

分维度可按以下方式切分，通常选择与业务最相关的 2 到 3 个：

- 业务类别：问答、摘要、分类、代码。
- 输入长度：短、中、长。
- 语言：中文、英文、混合。
- 用户分层：新用户、老用户。

# 6. 线上 A/B

## 6.1 流量分流

线上 A/B 需要把真实用户稳定地分到 A、B 两组。分流键通常用用户 ID，保证同一用户始终看到同一版本。

```
                 ┌── A 组（对照）→ v1
用户请求 ─ 哈希 ─┤
                 └── B 组（实验）→ v2
```

## 6.2 实验开关

### 6.2.1 开关与放量

```java
@Service
public class PromptExperiment {

    @Value("${app.prompt.experiment.enabled:false}")
    private boolean enabled;

    @Value("${app.prompt.experiment.treatment-percentage:10}")
    private int treatmentPercentage;

    public String resolvePrompt(String name, UserContext user) {
        if (!enabled) return promptV1.raw(name);

        int hash = Math.abs((user.userId() + name).hashCode());
        boolean inTreatment = (hash % 100) < treatmentPercentage;

        // 记录分组，便于分析
        experimentTracker.record(name, user.userId(),
                inTreatment ? "B" : "A");

        return inTreatment ? promptV2.raw(name) : promptV1.raw(name);
    }
}
```

### 6.2.2 分流一致性

| 分流键 | 稳定性 | 问题 |
|--------|--------|------|
| 随机数 | 无 | 同一用户反复切换 |
| 用户 ID | 高 | 不同实验可能相关 |
| 用户 ID + 实验名 | 高 | 推荐做法 |
| 设备 ID | 中 | 多设备用户不一致 |

## 6.3 快速止损

实验必须能一键关闭：把 `enabled` 置为 `false` 或把 `treatmentPercentage` 置为 `0`，即可让所有流量回到对照组，无需发版。

# 7. 线上指标

## 7.1 业务指标与系统指标

| 指标 | 采集方式 |
|------|----------|
| 点赞率 | 用户反馈 |
| 追问率 | 越低越好 |
| 转人工率 | 越低越好 |
| 任务完成率 | 业务埋点 |
| 平均 Token | 自动采集 |
| 错误率 | 自动采集 |

- 业务指标反映"用户是否满意"。
- 系统指标反映"系统是否健康"。
- 两者都要看，且实验前就应确定主指标。

## 7.2 埋点采集

```java
public void recordInteraction(String experiment, String variant,
                              String userId, InteractionResult result) {
    experimentStore.record(new Interaction(
            experiment, variant, userId,
            result.thumbsUp(),
            result.followUp(),
            result.tokens(),
            result.latencyMs(),
            Instant.now()));
}
```

埋点要同时记录实验名与分组，否则事后无法区分数据来自哪个版本。

## 7.3 指标口径一致性

| 陷阱 | 后果 | 对策 |
|------|------|------|
| A、B 组埋点字段不同 | 无法比较 | 统一埋点 schema |
| 统计窗口不同 | 伪差异 | 固定同窗口 |
| 去重规则不同 | 样本不一致 | 统一去重键 |
| 延迟包含网络抖动 | 噪声大 | 用分位数而非均值 |

# 8. 统计显著性

## 8.1 假设检验

A/B 检验的原假设是"两个版本没有差异"。若观察到的差异在"无差异"假设下出现的概率（p 值）足够小，则拒绝原假设，认为差异显著。

| 概念 | 含义 |
|------|------|
| 原假设 H0 | 两版本无差异 |
| p 值 | 无差异时观察到当前差异的概率 |
| 显著性水平 α | 通常取 0.05 |
| 置信区间 | 差异的合理范围 |

## 8.2 双比例 Z 检验实现

```java
public SignificanceResult test(double rateA, int nA,
                               double rateB, int nB) {
    double p = (rateA * nA + rateB * nB) / (nA + nB);
    double se = Math.sqrt(p * (1 - p) * (1.0 / nA + 1.0 / nB));
    double z = (rateB - rateA) / se;
    double pValue = 2 * (1 - normalCdf(Math.abs(z)));

    return new SignificanceResult(z, pValue, pValue < 0.05);
}
```

```java
public record SignificanceResult(double z, double pValue, boolean significant) {
}
```

| 指标 | 说明 |
|------|------|
| p 值 < 0.05 | 差异显著 |
| 样本量 | 需足够大 |
| 效应量 | 差异大小 |

## 8.3 样本量估算

### 8.3.1 样本量公式

对于双比例检验，单组样本量近似为：

```
n ≈ 16 * p * (1 - p) / δ²
```

其中 `p` 为基准比例，`δ` 为期望检测到的最小差异。

### 8.3.2 最小可检测效应

| 基准率 | 最小差异 | 单组样本量 |
|--------|----------|------------|
| 0.50 | 0.05 | 约 1600 |
| 0.50 | 0.02 | 约 10000 |
| 0.10 | 0.02 | 约 3600 |

**结论**：想检测越小的差异，需要的样本量越大。实验前先算样本量，避免"跑了一天就下结论"。

# 9. 实验陷阱

## 9.1 常见陷阱清单

| 陷阱 | 说明 |
|------|------|
| 样本不足 | 结论不可靠 |
| 过早下结论 | 需固定周期 |
| 多重比较 | 预设主指标 |
| 分流不均 | 检查分组 |
| 相互干扰 | 同用户多实验 |
| 指标污染 | 一次只改一个变量 |

## 9.2 陷阱案例拆解

| 案例 | 问题 | 修正 |
|------|------|------|
| 看了一天就上线 | 样本不足 | 按样本量跑满 |
| 同时改 Prompt 和模型 | 变量混淆 | 一次只改一个 |
| 只报显著的那个指标 | 多重比较 | 预设主指标 |
| A/B 组用户重叠 | 相互干扰 | 实验分层 |
| 周末流量差异 | 时间偏差 | 固定周期对齐 |

## 9.3 规避流程

```
确定主指标 → 算样本量 → 固定实验周期
   → 检查分流均衡 → 只看预设指标 → 显著才采纳
```

# 10. 生产注意事项

## 10.1 风险与对策

| 风险 | 建议 |
|------|------|
| 凭感觉决策 | 用 A/B 数据 |
| 样本不足 | 先算样本量 |
| 只看整体 | 分维度对比 |
| 过早决策 | 固定周期 |
| 无显著性检验 | 做统计检验 |
| 无回滚 | 保留开关 |

## 10.2 实验平台化

实验数量增多后，应把分流、埋点、统计、看板平台化，避免每个实验重复造轮子。

| 能力 | 价值 |
|------|------|
| 统一分流 SDK | 分流逻辑一致 |
| 统一埋点 | 口径一致 |
| 自动显著性计算 | 减少人工错误 |
| 实验看板 | 快速决策 |
| 实验互斥层 | 避免相互干扰 |

## 10.3 反模式

| 反模式 | 问题 | 正确做法 |
|--------|------|----------|
| 实验常开不关 | 流量长期分叉 | 有明确结束条件 |
| 边跑边改 Prompt | 数据不可比 | 实验期间冻结 |
| 用均值看延迟 | 长尾被掩盖 | 用 P95 / P99 |
| 只做线上不做离线 | 浪费流量 | 先离线筛 |
| 无对照组 | 无法归因 | 始终保留 A 组 |

# 11. 小结

## 11.1 核心要点

- A/B 测试用数据判断 Prompt 优劣。
- 离线 A/B 快，线上 A/B 真实，先离线后线上。
- 指标：准确率、延迟、成本、合规率。
- 分维度对比能发现"整体提升但某类下降"。
- 必须做统计显著性检验，避免过早下结论。

## 11.2 落地清单

- 为每个 Prompt 建立固定评测集与回归断言。
- 线上实验用用户 ID 稳定分流，保留一键开关。
- 实验前预设主指标并估算样本量。
- 分维度对比，警惕局部回归。
- 显著且无成本回退时才采纳新版本。
