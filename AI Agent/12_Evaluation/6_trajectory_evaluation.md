# 1. 什么是轨迹评测

结论先行：轨迹评测（trajectory evaluation）是对 Agent 从目标到结果的完整动作序列做质量评估，它回答"过程是否正确"，与只看结果的评测互补。

**轨迹（Trajectory）** = Agent 从目标到结果的完整动作序列。

```
目标：查订单 A123 物流并催单

轨迹：
① queryOrder("A123")        → 已发货，运单 SF123
② queryLogistics("SF123")   → 运输中，3 天未更新
③ urgeOrder("SF123")        → 催单成功
④ 生成最终答案

评测：步骤是否必要？顺序是否合理？有无冗余？
```

## 1.1 轨迹的定义

### 1.1.1 步骤结构

轨迹是状态转移序列：每个元素记录"决策、动作、观察、代价"。它既是评测对象，也是问题定位与人工评审的依据。

```java
public record Step(
        int index,
        StepType type,        // LLM 决策 / 工具调用
        String toolName,      // 工具名（LLM 决策步为 null）
        Map<String, Object> args,
        String observation,   // 工具返回或模型输出
        long latencyMs,
        long tokens,
        BigDecimal cost
) {
    public String signature() {
        return toolName + ":" + normalize(args);
    }
}
```

## 1.2 轨迹要素

### 1.2.1 要素表

| 要素 | 说明 | 用途 |
|------|------|------|
| 步骤序列 | 有序动作列表 | 顺序与必要性 |
| 工具与参数 | 调用了什么 | 工具/参数准确率 |
| 观察结果 | 工具返回 | 判断是否有效 |
| 代价 | 耗时/Token/成本 | 效率评测 |
| 终止原因 | 正常/超限/失败 | 终止合理性 |

边界与坑：轨迹必须完整记录，任何一步丢失都会导致必要性、循环等指标失真。

生产建议：轨迹以结构化 JSON 落库，附带 `traceId` 与 `taskId` 便于关联与回放。

# 2. 为什么评测轨迹

结论先行：结果正确不等于过程正确；只看结果会漏掉"碰巧对"与"越权对"两类高危情况。

```
结果正确 ≠ 过程正确
```

| 情况 | 结果 | 过程 | 评价 |
|------|:----:|:----:|------|
| 正常 | ✅ | ✅ | 好 |
| 碰巧对 | ✅ | ❌ | 危险（不稳定） |
| 走了弯路 | ✅ | ⚠️ | 低效 |
| 对但越权 | ✅ | ❌ | 不可接受 |

> 只看结果会漏掉"碰巧对"与"越权对"。

## 2.1 结果与过程的正交

### 2.1.1 四象限

结果与过程是两个独立维度，组合出四象限：

```
            过程正确
              │
   低效对 ────┼──── 理想
              │
   ───────────┼─────────── 结果正确
              │
   越权对 ────┼──── 碰巧对
              │
            过程错误
```

## 2.2 只看结果的盲区

### 2.2.1 盲区表

| 盲区 | 表现 | 后果 |
|------|------|------|
| 碰巧对 | 靠猜测命中 | 换输入即失败 |
| 越权对 | 用高危动作达成 | 合规风险 |
| 低效对 | 绕远路完成 | 成本与延迟高 |
| 隐藏错误 | 中间步骤错但自纠 | 稳定性差 |

边界与坑：轨迹评测需要 `expectedActions` 等标注，建标成本高于结果评测，需权衡投入。

生产建议：结果评测做全量，轨迹评测对关键任务与高危动作做重点覆盖。

# 3. 轨迹评测维度

结论先行：轨迹评测有七个维度——必要性、顺序、工具选择、参数、冗余、循环、终止。

| 维度 | 说明 |
|------|------|
| 步骤必要性 | 是否有无效步骤 |
| 顺序合理性 | 依赖是否满足 |
| 工具选择 | 是否用了正确的工具 |
| 参数正确性 | 参数是否正确 |
| 冗余度 | 是否重复调用 |
| 循环 | 是否陷入循环 |
| 终止 | 是否及时终止 |

## 3.1 七维度对照

### 3.1.1 判定方式

| 维度 | 判定方式 | 数据来源 |
|------|----------|----------|
| 必要性 | 规则/LLM | 步骤序列 |
| 顺序 | 子序列匹配 | 步骤序列 |
| 工具选择 | 规则 | toolName |
| 参数 | 规则/LLM | args |
| 冗余 | 规则 | signature |
| 循环 | 规则 | signature |
| 终止 | 规则 | 终止状态 |

## 3.2 规则与 LLM 的分工

### 3.2.1 分工边界

规则负责可确定的部分（工具名、签名、循环），LLM 负责语义部分（必要性、是否有更优路径）。

```
规则可判 ──► 工具名、参数格式、重复、循环、步数
LLM 可判 ──► 步骤必要性、路径合理性、替代方案
```

边界与坑：LLM 判官有成本与波动，能规则化的维度不要交给 LLM。

生产建议：优先规则，规则无法覆盖时再用 LLM，并对 LLM 结果抽样人工校验。

# 4. 步骤必要性

结论先行：步骤必要性衡量"实际调用中有多少是期望动作"，用于发现无效或多余的步骤。

## 4.1 必要性计算

### 4.1.1 计算实现

```java
public double stepNecessity(AgentTrace trace, List<String> expectedActions) {
    Set<String> actual = trace.steps().stream()
            .map(Step::toolName)
            .collect(Collectors.toSet());

    // 实际调用中属于期望动作的比例
    long necessary = actual.stream()
            .filter(expectedActions::contains)
            .count();

    return necessary / (double) actual.size();
}
```

| 实际动作 | 是否期望 | 计入必要 |
|----------|:---:|:---:|
| queryOrder | 是 | 是 |
| queryLogistics | 是 | 是 |
| getWeather | 否 | 否 |
| urgeOrder | 是 | 是 |

## 4.2 边界情形

### 4.2.1 常见边界

- 空轨迹：`actual.size()==0` 会除零，需单独处理。
- 合法替代：同一目的可用不同工具，硬编码期望集会误判。
- 去重影响：用 `Set` 会丢掉重复次数信息，冗余需另算。

边界与坑：必要性高不等于过程正确，还需结合顺序与参数判断。

生产建议：期望动作集允许"必需 + 可选"两级，减少对合法替代路径的误伤。

# 5. 顺序合理性

结论先行：顺序合理性检查"依赖是否满足"，通常用期望动作作为子序列与实际序列做匹配。

## 5.1 子序列匹配

### 5.1.1 匹配实现

```java
public boolean isOrderValid(AgentTrace trace, List<String> expectedOrder) {
    List<String> actualOrder = trace.steps().stream()
            .map(Step::toolName)
            .filter(expectedOrder::contains)
            .toList();

    // 检查相对顺序是否一致（子序列匹配）
    int i = 0;
    for (String action : actualOrder) {
        if (action.equals(expectedOrder.get(i))) {
            i++;
            if (i == expectedOrder.size()) return true;
        }
    }
    return false;
}
```

机制：期望序列是实际序列的子序列即视为顺序正确，允许中间插入无关步骤。

## 5.2 依赖图视角

### 5.2.1 拓扑校验

更严谨的做法是把动作建模为有向无环图（DAG），检查拓扑顺序：

```
queryOrder ──► queryLogistics ──► urgeOrder
                    │
                    └──► generateAnswer
```

| 顺序错误 | 示例 | 后果 |
|----------|------|------|
| 缺前置 | 未查询直接催单 | 失败 |
| 逆序 | 先催单后查单 | 逻辑错 |
| 并行误串行 | 可并行却串行 | 低效 |

边界与坑：子序列匹配无法识别"重复满足"，例如同一动作出现两次可能仍被判为正确。

生产建议：对有严格依赖的任务改用 DAG 拓扑校验，而非简单子序列。

# 6. 冗余检测

结论先行：冗余检测用"签名去重"找出重复调用，冗余步数直接反映过程效率。

## 6.1 签名去重

### 6.1.1 去重实现

```java
public int countRedundant(List<Step> steps) {
    Set<String> seen = new HashSet<>();
    int redundant = 0;

    for (Step step : steps) {
        if (!seen.add(step.signature())) {      // 签名重复
            redundant++;
        }
    }
    return redundant;
}
```

## 6.2 冗余分类

### 6.2.1 分类与处置

| 冗余类型 | 说明 |
|----------|------|
| 完全相同 | 同工具同参数 |
| 语义重复 | 不同工具达到同一目的 |
| 无效重试 | 失败后无变化地重试 |

| 类型 | 可否规则检测 | 处置 |
|------|:---:|------|
| 完全相同 | 是 | 去重缓存 |
| 语义重复 | 否（需 LLM） | 合并工具 |
| 无效重试 | 部分 | 退避 + 上限 |

边界与坑：参数规范化（大小写、空格、字段顺序）不做会导致漏检"实际相同"的调用。

生产建议：对查询类工具加结果缓存，相同签名短时间内直接复用。

# 7. 循环检测

结论先行：循环是 Agent 最危险的失效模式，必须用"窗口重复"规则实时检测并中断。

## 7.1 窗口重复检测

### 7.1.1 检测实现

```java
public LoopDetection detectLoop(List<Step> steps, int window, int repeat) {
    for (int i = window * (repeat - 1); i < steps.size(); i++) {
        boolean loop = true;
        for (int j = 1; j < repeat; j++) {
            if (!steps.get(i).signature()
                    .equals(steps.get(i - window * j).signature())) {
                loop = false;
                break;
            }
        }
        if (loop) {
            return new LoopDetection(true, i, steps.get(i).signature());
        }
    }
    return LoopDetection.none();
}
```

## 7.2 参数与阈值

### 7.2.1 窗口取值

| window | repeat | 含义 |
|:---:|:---:|------|
| 1 | 3 | 连续 3 次相同调用 |
| 2 | 3 | ABABAB 式循环 |
| 3 | 2 | ABCABC 式循环 |

边界与坑：`window` 过小会误伤合法重试，过大则漏检；应结合业务设阈值并记录中断原因。

生产建议：检测到循环立即中断并降级返回，同时上报监控告警。

# 8. 轨迹评分

结论先行：轨迹评分把多维指标加权为综合分，并对循环等严重问题施加惩罚。

## 8.1 评分构成

### 8.1.1 加权公式

```java
public record TrajectoryScore(
        double necessity,        // 步骤必要性
        double orderCorrectness, // 顺序正确性
        double toolAccuracy,     // 工具选择准确率
        double paramAccuracy,    // 参数准确率
        int redundantSteps,
        boolean hasLoop,
        double overall           // 综合分
) {}

public TrajectoryScore score(AgentTrace trace, EvalCase expected) {
    double necessity = stepNecessity(trace, expected.expectedActions());
    double order = isOrderValid(trace, expected.expectedActions()) ? 1.0 : 0.0;
    double toolAcc = toolAccuracy(trace, expected);
    double paramAcc = paramAccuracy(trace, expected);
    int redundant = countRedundant(trace.steps());
    boolean loop = detectLoop(trace.steps(), 1, 3).detected();

    double overall = necessity * 0.25 + order * 0.25
            + toolAcc * 0.25 + paramAcc * 0.25;
    if (loop) overall *= 0.5;                 // 循环重罚

    return new TrajectoryScore(necessity, order, toolAcc, paramAcc,
            redundant, loop, overall);
}
```

## 8.2 惩罚项

### 8.2.1 惩罚规则

| 问题 | 惩罚方式 | 理由 |
|------|----------|------|
| 循环 | 乘 0.5 | 严重失效 |
| 冗余多 | 线性扣分 | 低效 |
| 越权 | 置 0 | 红线 |
| 未终止 | 置 0 | 不可用 |

边界与坑：乘性惩罚会随多次触发快速趋零，注意惩罚叠加的上限与可解释性。

生产建议：综合分用于排序优化，红线问题单独门禁，不依赖综合分兜底。

# 9. 用 LLM 评测轨迹

结论先行：规则无法覆盖"路径是否合理、是否有更优方案"等语义问题，需引入 LLM 判官（LLM-as-a-Judge）。

## 9.1 判官提示

### 9.1.1 提示模板

```java
interface TrajectoryJudge {
    @UserMessage("""
            评估 Agent 的执行轨迹是否合理。

            目标：{{goal}}

            执行轨迹：
            {{trajectory}}

            评估维度：
            1. 步骤是否都是必要的
            2. 顺序是否合理
            3. 是否有更高效的路径
            4. 是否走了弯路

            输出 JSON：
            {"score":0.0,"issues":["..."],"suggestion":""}
            """)
    TrajectoryJudgement judge(@V("goal") String goal,
                              @V("trajectory") String trajectory);
}
```

## 9.2 规则与 LLM 融合

### 9.2.1 融合权重

| 维度 | 规则 | LLM |
|------|:---:|:---:|
| 工具选择 | ✅ | — |
| 参数格式 | ✅ | — |
| 冗余/循环 | ✅ | — |
| 必要性 | 部分 | ✅ |
| 路径合理性 | — | ✅ |

```
最终分 = 规则分 × 0.6 + LLM 分 × 0.4
```

边界与坑：LLM 判官存在位置偏差与自偏好，需固定模型版本并做校准样本。

生产建议：LLM 判官结果与规则冲突时，转人工复核并沉淀为规则。

# 10. 轨迹可视化

结论先行：可视化把轨迹转成可读时间线，是人工评审与问题定位的基础。

## 10.1 文本时间线

### 10.1.1 时间线示例

```
Step 1  [LLM]    决策：调用 queryOrder
        └─ 耗时 1.2s, tokens 850

Step 2  [Tool]   queryOrder("A123")
        └─ 结果：SHIPPED, SF123 | 耗时 120ms

Step 3  [LLM]    决策：调用 queryLogistics
        └─ 耗时 1.1s, tokens 920

Step 4  [Tool]   queryLogistics("SF123")
        └─ 结果：运输中, 3天未更新 | 耗时 340ms

Step 5  [LLM]    决策：调用 urgeOrder
        └─ 耗时 1.3s, tokens 1050

Step 6  [Tool]   urgeOrder("SF123")
        └─ 结果：催单成功 | 耗时 200ms

Step 7  [LLM]    生成最终答案
        └─ 耗时 1.5s, tokens 1200

总计：7 步 | 5.8s | 4340 tokens | $0.008
```

> 可视化轨迹是人工评审与问题定位的基础。

## 10.2 可视化要点

### 10.2.1 展示要点

| 要点 | 说明 |
|------|------|
| 区分 LLM/工具 | 两类步骤视觉区分 |
| 标注耗时 Token | 定位性能瓶颈 |
| 高亮异常 | 失败/重试/循环标红 |
| 折叠长输出 | 保留摘要与原文入口 |

边界与坑：长轨迹一次性展示会淹没关键信息，需支持按步折叠与过滤。

生产建议：把可视化与 traceId 关联，支持从监控告警一键跳转到轨迹详情。

# 11. 生产注意事项

结论先行：轨迹评测的风险集中在"只看结果、无轨迹、无期望动作、循环冗余不度量、人工评审无标准"。

## 11.1 记录与标准

### 11.1.1 记录要求

| 风险 | 建议 |
|------|------|
| 只看结果 | 必须看轨迹 |
| 无轨迹记录 | 保留完整 Trace |
| 无期望动作 | 用例标注 expectedActions |
| 人工评审无标准 | 定义评分维度 |

## 11.2 监控与门禁

### 11.2.1 监控项

| 风险 | 建议 |
|------|------|
| 循环未检测 | 加入检测 |
| 冗余不度量 | 统计冗余步数 |
| 无实时中断 | 超限即中止 |
| 无基线对比 | 保留历史轨迹 |

边界与坑：轨迹含用户数据与工具返回，落库需脱敏与访问控制。

生产建议：循环率、冗余率、平均步数做成监控指标，异常时自动告警并触发评测回归。

# 12. 小结

## 12.1 核心要点

- 轨迹评测关注"过程是否正确"，弥补只看结果的盲区。
- 维度：必要性、顺序、工具选择、参数、冗余、循环、终止。
- 用规则 + LLM 结合评分。

## 12.2 落地清单

- 轨迹可视化是人工评审与定位的基础。
- 循环与越权必须重罚。
- 轨迹需完整记录、脱敏存档，并纳入监控门禁。
