# 1. 为什么需要线上 A/B

```
离线评测：固定评测集，不能反映真实用户行为
线上 A/B：真实流量，验证真实效果
```

## 1.1 离线评测的局限

- 评测集是构造的，与真实分布存在偏差。
- 无法反映用户真实行为（追问、放弃、点踩）。
- 无法衡量业务指标（转化、留存、成本）。

## 1.2 线上 A/B 的价值

### 1.2.1 与离线评测对比

| 维度 | 离线评测 | 线上 A/B |
|------|----------|----------|
| 数据 | 构造 | 真实 |
| 指标 | 质量分数 | 业务指标 |
| 反馈 | 快 | 慢 |
| 成本 | 低 | 高 |
| 真实性 | 有限 | 高 |

> 离线保证不退化，线上验证真实效果，两者互补而非替代。

## 1.3 适用与不适用场景

| 场景 | 是否适合 A/B | 说明 |
|------|--------------|------|
| Prompt 微调 | 适合 | 可随机分流 |
| 模型替换 | 适合 | 关注质量与成本 |
| UI 大改 | 谨慎 | 学习效应强 |
| 低频高价值 | 不适合 | 样本量不足 |
| 安全修复 | 不适合 | 应直接全量 |

# 2. 实验设计

## 2.1 六步流程

```
① 提出假设：新版 Prompt 能提升满意度
② 定义指标：满意度、任务完成率、成本
③ 分流：随机分配到 A（对照）与 B（实验）
④ 收集数据：固定周期
⑤ 统计检验：判断差异是否显著
⑥ 决策：上线 / 放弃 / 继续
```

## 2.2 假设与指标设计

### 2.2.1 主指标与护栏指标

| 类型 | 作用 | 示例 |
|------|------|------|
| 主指标（OEC） | 决定成败 | 任务完成率 |
| 护栏指标 | 防负向 | 延迟、成本、安全 |
| 诊断指标 | 解释原因 | 追问率、转人工率 |

> 假设要可证伪：「B 的任务完成率比 A 高至少 3%」，而不是「B 更好」。

## 2.3 分流单元与随机化

| 分流单元 | 优点 | 缺点 |
|----------|------|------|
| 用户 ID | 体验一致 | 需登录 |
| 会话 ID | 无需登录 | 同用户可能跨组 |
| 设备 ID | 稳定 | 多设备不一致 |

# 3. 分流实现

## 3.1 路由算法

### 3.1.1 稳定哈希

同一用户在同一实验里必须始终进同一组，否则体验割裂、指标污染。

### 3.1.2 分桶

```
bucket = hash(userId + experimentName) % 100
treatment 若 bucket < treatmentPercentage
```

## 3.2 路由实现代码

```java
public class ExperimentRouter {

    public Variant route(String experimentName, String userId) {
        ExperimentConfig config = configStore.get(experimentName);
        if (!config.enabled()) {
            return Variant.control(config.controlVersion());
        }
        // 稳定哈希：同一用户始终分到同一组
        int hash = Math.abs((userId + experimentName).hashCode());
        int bucket = hash % 100;
        return bucket < config.treatmentPercentage()
                ? Variant.treatment(config.treatmentVersion())
                : Variant.control(config.controlVersion());
    }
}
```

> `String.hashCode()` 在 Java 各版本间稳定，但与 Python 等语言不一致；需跨语言一致时改用 MurmurHash3。

## 3.3 分流要求与校验

| 要求 | 说明 |
|------|------|
| 随机性 | 均匀分配 |
| 稳定性 | 同一用户始终同组 |
| 可追溯 | 记录用户所属组 |
| 可关闭 | 一键停止实验 |

### 3.3.1 均匀性检验

```java
public boolean isBalanced(Map<String, Long> groupCounts) {
    long total = groupCounts.values().stream().mapToLong(Long::longValue).sum();
    for (long count : groupCounts.values()) {
        double ratio = count / (double) total;
        if (ratio < 0.45 || ratio > 0.55) {
            return false;
        }
    }
    return true;
}
```

# 4. 指标定义

## 4.1 指标分类

| 类别 | 指标 | 方向 |
|------|------|------|
| 业务 | 任务完成率、满意度 | 越高越好 |
| 质量 | 点赞率、转人工率 | 点赞高、转人工低 |
| 效率 | P50/P99 延迟、成本 | 越低越好 |
| 安全 | 护栏触发率、错误率 | 越低越好 |

## 4.2 指标数据结构

```java
public record ExperimentMetrics(
        // 业务指标
        double taskCompletionRate,     // 任务完成率
        double userSatisfaction,       // 满意度（点赞/点踩）
        double followUpRate,           // 追问率（越低越好）
        // 质量指标
        double thumbsUpRate,           // 点赞率
        double escalationRate,         // 转人工率
        // 效率指标
        long p50LatencyMs,
        long p99LatencyMs,
        BigDecimal avgCost,
        // 安全指标
        double guardrailTriggerRate,
        double errorRate
) {}
```

## 4.3 指标计算公式

```
taskCompletionRate = 完成任务会话数 / 总会话数
userSatisfaction   = (点赞 - 点踩) / (点赞 + 点踩)
followUpRate       = 有追问的会话数 / 总会话数
escalationRate     = 转人工会话数 / 总会话数
p99LatencyMs       = 延迟的 99 分位数
```

## 4.4 指标设计注意事项

- 主指标只能有一个，避免「指标打架」。
- 比率指标要设最小样本量，否则方差过大。
- 成本按会话而非按请求统计，防止长会话被低估。
- 护栏指标一旦恶化，即使主指标提升也应否决。

# 5. 数据收集

## 5.1 埋点与记录

| 数据 | 用途 | 注意 |
|------|------|------|
| 分组 | 归因 | 必须落库 |
| 反馈 | 满意度 | 区分主动/被动 |
| 延迟 | 效率 | 分位数而非均值 |
| 成本 | 经济性 | 含重试与缓存 |

## 5.2 收集实现

```java
@Component
public class ExperimentTracker {

    public void recordInteraction(String experiment, Variant variant,
                                  String userId, InteractionResult result) {
        experimentStore.record(new Interaction(
                experiment,
                variant.name(),
                userId,
                result.taskCompleted(),
                result.userFeedback(),      // +1 / -1 / 0
                result.latencyMs(),
                result.cost(),
                Instant.now()));
    }
}
```

## 5.3 数据质量

| 问题 | 后果 | 应对 |
|------|------|------|
| 分组未记录 | 无法归因 | 强制落库 |
| 反馈稀疏 | 满意度噪声大 | 结合行为指标 |
| 时区不一致 | 周期对齐错 | 统一 UTC |
| 重复计数 | 指标虚高 | 幂等键去重 |

# 6. 统计显著性

```
核心问题：A 与 B 的差异是真实的，还是随机波动？
```

## 6.1 假设检验基础

### 6.1.1 原假设与备择假设

- 原假设 H0：A 与 B 无差异。
- 备择假设 H1：A 与 B 有差异。
- p 值：H0 成立时，观察到当前或更极端结果的概率。

### 6.1.2 第一类与第二类错误

| 错误 | 含义 | 概率 | 代价 |
|------|------|------|------|
| 第一类 | 无差异误判为有 | α（0.05） | 错误上线 |
| 第二类 | 有差异误判为无 | β（0.20） | 错失优化 |
| 功效 | 1 - β | 0.80 | 越高越灵敏 |

## 6.2 样本量估算

```
n ≈ 16 × σ² / Δ²

σ：指标标准差
Δ：期望检测的最小差异
```

```java
public int requiredSampleSize(double stdDev, double minDetectableEffect) {
    return (int) Math.ceil(16 * Math.pow(stdDev, 2)
            / Math.pow(minDetectableEffect, 2));
}
```

> 想检测更小的差异，样本量按平方增长：Δ 减半，n 增至 4 倍。

## 6.3 显著性检验

### 6.3.1 双比例 z 检验

```java
public SignificanceResult test(double controlRate, int controlN,
                               double treatmentRate, int treatmentN) {
    // 双比例 z 检验
    double p1 = controlRate, p2 = treatmentRate;
    double p = (p1 * controlN + p2 * treatmentN) / (controlN + treatmentN);
    double se = Math.sqrt(p * (1 - p) * (1.0 / controlN + 1.0 / treatmentN));
    double z = (p2 - p1) / se;
    double pValue = 2 * (1 - normalCdf(Math.abs(z)));
    return new SignificanceResult(z, pValue, pValue < 0.05);
}
```

### 6.3.2 连续指标 t 检验

延迟、成本等连续指标用 Welch's t 检验（不假设方差相等）：

```java
public SignificanceResult welchT(double meanA, double varA, int nA,
                                 double meanB, double varB, int nB) {
    double se = Math.sqrt(varA / nA + varB / nB);
    double t = (meanB - meanA) / se;
    double df = Math.pow(varA / nA + varB / nB, 2)
            / (Math.pow(varA / nA, 2) / (nA - 1)
             + Math.pow(varB / nB, 2) / (nB - 1));
    double pValue = 2 * (1 - studentTCdf(Math.abs(t), df));
    return new SignificanceResult(t, pValue, pValue < 0.05);
}
```

## 6.4 置信区间与效应量

```
95% CI = (p2 - p1) ± 1.96 × SE
效应量 Cohen's h = 2 × arcsin(√p2) - 2 × arcsin(√p1)
```

| 指标 | 说明 |
|------|------|
| p 值 < 0.05 | 差异显著（95% 置信） |
| 置信区间 | 差异的范围 |
| 效应量 | 差异的大小 |

> 显著不等于重要：p 值小但效应量微弱时，收益可能不抵工程成本。

# 7. 实验陷阱

## 7.1 统计类陷阱

| 陷阱 | 说明 | 避免 |
|------|------|------|
| 样本不足 | 结论不可靠 | 先算样本量 |
| 过早下结论 | 看到差异就停 | 固定周期 |
| 多重比较 | 看太多指标 | 预设主指标 |
| 幸存者偏差 | 只统计完成的 | 用全量 |
| 多次偷看后停止 | 假阳性膨胀 | 固定停止规则 |

## 7.2 工程类陷阱

| 陷阱 | 说明 | 避免 |
|------|------|------|
| 分流不均 | 两组用户分布不同 | 检查分流 |
| 相互干扰 | 同用户参与多实验 | 实验分层 |
| 指标被污染 | 混入其他变更 | 一次只改一个变量 |
| 缓存串组 | A 组命中 B 组缓存 | 缓存键含分组 |
| 配置不一致 | 部分实例未生效 | 配置中心统一推送 |

## 7.3 业务类陷阱

| 陷阱 | 说明 |
|------|------|
| 新奇效应 | 用户因新鲜感短期偏好新版 |
| 学习效应 | 用户适应后效果反转 |
| 季节性 | 节假日流量异常 |
| 局部最优 | 主指标涨、长期留存降 |

> 长周期实验（≥2 周）可缓解新奇效应；必要时看留存等长期指标。

# 8. 实验生命周期

## 8.1 状态机

```java
public enum ExperimentStatus {
    DRAFT, RUNNING, PAUSED, COMPLETED, ROLLED_BACK
}
```

```
DRAFT -> RUNNING -> COMPLETED
           |  ^
           v  |
        PAUSED
           |
           v
       ROLLED_BACK
```

## 8.2 分析实现

```java
public void analyze(String experimentName) {
    ExperimentData data = experimentStore.load(experimentName);
    if (data.totalSamples() < requiredSampleSize()) {
        log.info("样本不足，继续收集: {}/{}",
                data.totalSamples(), requiredSampleSize());
        return;
    }
    SignificanceResult result = test(
            data.controlMetrics().completionRate(), data.controlSamples(),
            data.treatmentMetrics().completionRate(), data.treatmentSamples());
    if (result.significant() && data.treatmentBetter()) {
        log.info("实验成功，建议全量上线");
    } else if (result.significant() && data.controlBetter()) {
        log.info("实验失败，建议回滚");
    } else {
        log.info("差异不显著，建议继续或放弃");
    }
}
```

## 8.3 回滚与开关

- 实验必须有远程开关（Feature Flag），异常时秒级关闭。
- 全量上线前保留一个「快速回滚」窗口。
- 回滚要记录原因与指标快照，供复盘。

# 9. 与灰度发布的区别

## 9.1 目的与指标差异

| 维度 | 灰度发布 | A/B 实验 |
|------|----------|----------|
| 目的 | 验证稳定性 | 验证效果 |
| 指标 | 错误率、延迟 | 业务指标 |
| 决策 | 是否回滚 | 是否全量 |
| 时长 | 短（小时） | 长（天/周） |
| 样本 | 少量 | 需统计功效 |

## 9.2 结合使用

```
① 灰度 1%：验证不崩溃、延迟正常
② 灰度 10%：验证错误率与成本
③ A/B 50%：验证业务指标显著性
④ 全量：确认稳定后放量
```

> 两者可结合：先灰度验稳定性，再 A/B 验效果。

## 9.3 常见误区

| 误区 | 纠正 |
|------|------|
| 用灰度代替 A/B | 灰度不测业务效果 |
| 用 A/B 代替灰度 | A/B 不保证稳定性 |
| 全量后才看指标 | 风险不可控 |

# 10. 生产注意事项

## 10.1 风险与建议

| 风险 | 建议 |
|------|------|
| 样本不足 | 先算样本量 |
| 过早决策 | 固定周期 |
| 指标过多 | 预设主指标 |
| 分流不均 | 检查分组 |
| 实验干扰 | 分层设计 |
| 无回滚 | 保留开关 |
| 成本失控 | 控制实验流量 |

## 10.2 平台化与可观测

### 10.2.1 实验台账

| 字段 | 说明 |
|------|------|
| 实验名/负责人 | 责任到人 |
| 假设/主指标 | 可追溯 |
| 起止时间 | 防止无限期 |
| 流量比例 | 控制影响面 |
| 结论/动作 | 沉淀知识 |

- 实验与 Trace 打通，可直接下钻到失败会话。
- 指标看板与告警联动，异常自动暂停实验。

## 10.3 成本控制

- 限制实验最大流量比例（如 ≤20%）。
- 设置自动停止：达到样本量或显著即停。
- 复用对照组的日志，避免重复采集。

# 11. 小结

## 11.1 核心结论

- 线上 A/B 用真实流量验证真实效果，与离线评测互补。
- 分流要随机、稳定、可追溯、可关闭。
- 指标覆盖业务、质量、效率、安全。
- 必须做统计显著性检验，先算样本量。
- 避免过早决策、多重比较等陷阱。

## 11.2 落地清单

| 项 | 检查 |
|----|------|
| 假设 | 可证伪，含最小可检测差异 |
| 分流 | 稳定哈希，均匀性已校验 |
| 指标 | 一个主指标 + 护栏指标 |
| 样本量 | 实验前已估算 |
| 显著性 | 有检验与置信区间 |
| 开关 | 可秒级回滚 |
| 台账 | 结论已沉淀 |
