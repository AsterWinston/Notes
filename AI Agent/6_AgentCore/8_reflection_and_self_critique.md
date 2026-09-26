# 1. 为什么需要反思
结论先行：Agent 的第一次尝试往往不完美，而**反思（Reflection）是提升成功率最划算的手段之一**——它让 Agent 从失败中提取教训并改进，无需重新训练模型。

## 1.1 反思的定义与定位
### 1.1.1 什么是反思（Reflection）
反思指 Agent 产生输出后，主动评估、归因，并据此修正后续行为的过程，把"一次性生成"变成"可迭代改进"。完整反思包含四个动作：

1. 评估（Evaluate）：判断当前结果好坏。
2. 归因（Attribute）：定位问题出在哪一步、为什么。
3. 抽象（Abstract）：把具体失败提炼为可复用教训。
4. 应用（Apply）：在下一轮尝试中利用该教训。

### 1.1.2 反思与相关概念的区别
| 概念 | 关注点 | 时机 | 是否改变行为 |
|------|--------|------|--------------|
| 反思 | 从失败中学习 | 单次任务内 | 是，影响下一轮 |
| 重试 | 换随机性再试 | 单次任务内 | 否，只是重复 |
| 微调 | 更新模型参数 | 离线训练 | 是，永久改变 |
| 提示优化 | 改系统提示词 | 离线开发 | 是，跨任务 |
| 评测 | 度量质量 | 离线 | 否，只观测 |

> 关键区别：重试是"再赌一次"，反思是"带着教训再赌一次"。

## 1.2 反思的价值与代价
### 1.2.1 收益来源
- 纠错：修正算术、格式、事实性错误。
- 补全：发现遗漏的约束或步骤。
- 收敛：多轮迭代逼近正确答案。
- 复用：反思记忆可跨尝试、跨相似任务复用。

### 1.2.2 成本与风险
| 维度 | 代价 |
|------|------|
| Token | 每轮评估 + 反思 + 重生成，约为单次的 2~4 倍 |
| 延迟 | 串行多轮，端到端延迟线性上升 |
| 质量 | 自评偏差可能导致越改越差 |
| 复杂度 | 需评估器、终止条件、版本管理 |

## 1.3 反思循环的整体流程
### 1.3.1 闭环流程图
```
执行 → 评估 → 发现不足 → 反思原因 → 改进重试
  ↑                                    │
  └────────────────────────────────────┘
```

### 1.3.2 与 Agent 主循环的集成
| 位置 | 说明 | 适用 |
|------|------|------|
| 单步后 | 每个工具调用后自检 | 高风险操作 |
| 子任务后 | 每个子目标完成后复盘 | 长任务分解 |
| 整体后 | 整个任务结束后复盘 | 一次性交付 |

> 生产上建议"整体后反思为主、子任务后反思为辅"，避免每步反思导致成本爆炸。

# 2. 三种反思层次
结论先行：反思越靠近"策略层"，改进价值越大，但越难自动化；生产上通常以结果反思打底，逐步向过程、策略层扩展。

## 2.1 结果反思（Outcome Reflection）
### 2.1.1 定义与示例
结果反思只关心"输出对不对"，不关心过程。

| 项目 | 内容 |
|------|------|
| 内容 | 输出是否正确 |
| 例子 | "答案算错了" |
| 自动化难度 | 低 |
| 典型评估器 | 规则校验、单元测试 |

### 2.1.2 实现要点
```java
public record OutcomeReflection(boolean ok, String reason) {}

public OutcomeReflection reflectOutcome(String expected, String actual) {
    if (expected.equals(actual)) {
        return new OutcomeReflection(true, "结果一致");
    }
    return new OutcomeReflection(false,
            "期望 " + expected + "，实际 " + actual);
}
```

## 2.2 过程反思（Process Reflection）
结论先行：过程反思关注"步骤是否合理"，能发现"答案碰巧对了但方法错"的隐患。

```java
public record Step(String action, String observation) {}

public String reflectProcess(List<Step> trace) {
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < trace.size(); i++) {
        Step s = trace.get(i);
        if (s.observation().contains("未找到")) {
            sb.append("第 ").append(i + 1).append(" 步 ")
              .append(s.action()).append(" 无效，前置步骤可能缺失\n");
        }
    }
    return sb.isEmpty() ? "过程无明显问题" : sb.toString();
}
```

## 2.3 策略反思（Strategy Reflection）
结论先行：策略反思处于最高层，审视"方法是否最优"，往往需要跨任务的经验积累。

```java
public String reflectStrategy(String task, List<String> pastFailures) {
    return model.chat("""
            你是流程优化专家。请从以下失败记录中提炼可复用的
            策略级教训（不要停留在具体操作）：
            任务：%s
            失败记录：
            %s
            """.formatted(task, String.join("\n", pastFailures)));
}
```

## 2.4 三层对比与选择
| 层次 | 内容 | 例子 | 价值 | 自动化难度 |
|------|------|------|------|------------|
| 结果反思 | 输出是否正确 | "答案算错了" | 低 | 低 |
| 过程反思 | 步骤是否合理 | "不该先查物流，应先查订单" | 中 | 中 |
| 策略反思 | 方法是否最优 | "应该先确认用户身份再操作" | 高 | 高 |

> 越往上层反思，改进价值越大，但越难自动化。

选择建议：有验证器用结果反思；有多步轨迹用过程反思；长期同类任务、追求泛化用策略反思。

# 3. 基础实现：生成-批判-修正
结论先行：生成-批判-修正（Generate-Critique-Revise）是反思的最小可行形态，用一对 Generator/Critic 交替循环即可实现。

## 3.1 模式概述
```
生成 generate → 批判 critique → 修正 revise
     ↑                              │
     └────────── 未通过 ────────────┘
```

| 角色 | 职责 | 是否可用同一模型 |
|------|------|------------------|
| Generator | 产出初稿 | 是 |
| Critic | 找问题、给建议 | 建议独立 |
| Reviser | 据意见改写 | 可与 Generator 同 |

## 3.2 完整 Java 实现
### 3.2.1 接口定义
```java
interface Generator {
    String generate(String task);
}

interface Critic {
    @UserMessage("""
            严格评估下面的回答，指出具体问题并给出改进建议。
            任务：{{task}}
            回答：{{answer}}

            输出格式：
            问题：...
            建议：...
            """)
    String critique(@V("task") String task, @V("answer") String answer);
}
```

### 3.2.2 编排逻辑
```java
public String refine(String task, int rounds) {
    String answer = generator.generate(task);

    for (int i = 0; i < rounds; i++) {
        String critique = critic.critique(task, answer);
        if (critique.contains("无问题")) break;

        answer = generator.generate("""
                任务：%s
                上一版回答：%s
                评审意见：%s
                请据此改进。
                """.formatted(task, answer, critique));
    }
    return answer;
}
```

### 3.2.3 运行示例
| 轮次 | 动作 | 结果 |
|------|------|------|
| 0 | generate | 初稿 |
| 1 | critique | "缺少边界处理" |
| 1 | revise | 补上边界处理 |
| 2 | critique | "无问题" → 退出 |

## 3.3 边界与坑
- 死循环：Critic 永远不说"无问题" → 必须有轮数上限。
- 空泛批判：只输出"不够好" → 强制"问题 + 建议"格式。
- 同质化：Generator 与 Critic 同模型时互相附和 → 用不同温度或模型。
- 上下文丢失：改写时未携带原始约束 → 每轮重申任务。

## 3.4 生产建议
- 轮数默认 2，最多 3。
- Critic 用更强模型，Generator 可用更便宜模型。
- 每轮保存中间版本，便于回退与审计。

# 4. Reflexion：带记忆的反思
结论先行：Reflexion 的关键是**把反思结果存入记忆**，供后续尝试使用，让 Agent "越试越聪明"。

## 4.1 核心思想
| 维度 | 生成-批判-修正 | Reflexion |
|------|----------------|-----------|
| 记忆 | 无，只传上一版 | 累积反思 |
| 反思内容 | 针对当前稿 | 提炼为教训 |
| 跨尝试 | 否 | 是 |
| 上下文压力 | 低 | 高 |

## 4.2 完整 Java 实现
### 4.2.1 数据结构
```java
public record Evaluation(boolean success, String feedback) {}
public record Reflection(int attempt, String lesson) {}
```

### 4.2.2 主循环
```java
public class ReflexionAgent {

    private final ChatModel model;
    private final List<String> reflections = new ArrayList<>();

    public String solve(String task, int maxAttempts) {
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            String attemptResult = attempt(task);

            Evaluation eval = evaluate(task, attemptResult);
            if (eval.success()) {
                return attemptResult;
            }
            // 生成反思并累积
            String reflection = reflect(task, attemptResult, eval.feedback());
            reflections.add(reflection);
        }
        return "多次尝试仍未成功";
    }

    private String attempt(String task) {
        String prompt = task;
        if (!reflections.isEmpty()) {
            prompt += "\n\n过往失败的经验教训：\n"
                    + String.join("\n", reflections);
        }
        return model.chat(prompt);
    }
}
```

### 4.2.3 记忆压缩
反思累积会让上下文膨胀，需要压缩：

```java
private String compress(List<String> reflections, int maxChars) {
    String joined = String.join("\n", reflections);
    if (joined.length() <= maxChars) return joined;

    return model.chat("""
            把下列经验教训压缩为不超过 5 条要点，保留可操作信息：
            %s
            """.formatted(joined));
}
```

## 4.3 与基础实现对比
| 场景 | 更适合 |
|------|--------|
| 单次可改好 | 生成-批判-修正 |
| 需要多次试错 | Reflexion |
| 上下文敏感 | 生成-批判-修正 |
| 同类任务反复 | Reflexion |

## 4.4 边界与坑
- 上下文长度：反思无限累积会超出窗口。
- 教训过拟合：针对某次失败的经验可能误导后续。
- 记忆污染：错误反思被反复复用。

## 4.5 生产建议
- 限制反思条数与总字符数。
- 定期清理低质量反思。
- 把反思记忆写入外部存储，便于审计。

> 反思累积让 Agent "越试越聪明"，但要注意上下文长度。

# 5. 评估器设计
结论先行：反思的前提是"能判断好坏"，评估器（Evaluator）的质量决定了反思的上限。

## 5.1 评估方式总览
| 方式 | 说明 | 适用 |
|------|------|------|
| 规则校验 | 格式、字段、范围 | 结构化输出 |
| 单元测试 | 代码正确性 | 编码 Agent |
| 工具验证 | 能否通过验证器 | 有明确验证的场景 |
| LLM 评审 | 语义质量 | 开放生成 |
| 人工评审 | 最终把关 | 高风险 |

## 5.2 分层评估实现
### 5.2.1 规则校验层
```java
public interface SchemaValidator {
    boolean isValid(String result);
}
```

### 5.2.2 客观验证层
```java
public interface TestRunner {
    TestResult run(String code);
}

public record TestResult(boolean passed, String failureOutput) {}
```

### 5.2.3 LLM 评审层
```java
public record JudgeResult(double score, String reason) {}

public interface LlmJudge {
    JudgeResult judge(String task, String result);
}
```

组合评估：

```java
public Evaluation evaluate(String task, String result) {
    if (!schemaValidator.isValid(result)) {
        return new Evaluation(false, "输出格式不符合要求");
    }
    JudgeResult judge = llmJudge.judge(task, result);
    return new Evaluation(judge.score() >= 0.8, judge.reason());
}
```

## 5.3 评估器元评估
| 指标 | 含义 |
|------|------|
| 准确率 | 与人工判断一致的比例 |
| 假阴性 | 把好结果判为坏（导致无谓重试） |
| 假阳性 | 把坏结果判为好（导致错误通过） |

## 5.4 生产建议
- 先跑规则校验，省 token。
- LLM 评审给出明确评分标准（rubric）。
- 定期用人工标注校准评估器。

# 6. 反思的终止条件
结论先行：反思循环必须有终止条件，否则会无限迭代、成本失控。

## 6.1 终止条件清单
| 条件 | 说明 |
|------|------|
| 评估通过 | 正常结束 |
| 达到最大轮数 | 硬性上限 |
| 反思无新内容 | 说明陷入僵局 |
| 成本超预算 | 成本保护 |

```java
if (reflections.size() >= 2 && isSameAsPrevious(reflection)) {
    return "已多次反思无新改进，停止";
}
```

## 6.2 组合终止策略实现
```java
public final class StopPolicy {

    private final int maxRounds;
    private final long maxTokens;

    public StopPolicy(int maxRounds, long maxTokens) {
        this.maxRounds = maxRounds;
        this.maxTokens = maxTokens;
    }

    public boolean shouldStop(int round, boolean passed,
                              String reflection, String last,
                              long usedTokens) {
        if (passed) return true;
        if (round >= maxRounds) return true;
        if (usedTokens >= maxTokens) return true;
        return reflection != null && reflection.equals(last);
    }
}
```

## 6.3 边界与坑
- 只设轮数不设成本：长提示下 token 仍可能超支。
- "无新内容"判定过松：措辞略变就漏判 → 可用相似度阈值。
- 评估通过即停：可能过早停止，错过更优解 → 设"目标分"而非"及格分"。

# 7. 反思的常见陷阱
结论先行：反思并非免费午餐，最常见的失败模式是自评偏差与成本失控。

## 7.1 陷阱清单
| 陷阱 | 表现 | 对策 |
|------|------|------|
| 自评过宽 | 模型认为"已经很好了" | 用独立 Critic + 严格提示 |
| 自评过严 | 反复否定正确答案 | 设通过阈值 + 轮数上限 |
| 无效反思 | 反思空泛无指导性 | 要求"具体问题 + 具体建议" |
| 反思漂移 | 越改越偏离目标 | 每轮重申原始任务 |
| 成本爆炸 | 无限迭代 | 轮数 + 成本上限 |

## 7.2 陷阱的检测与自动缓解
```java
public boolean isDrifting(String originalTask, String currentAnswer) {
    JudgeResult r = llmJudge.judge(originalTask, currentAnswer);
    return r.score() < 0.5;   // 与原始任务相关性骤降
}

public boolean isVague(String reflection) {
    return reflection.length() < 10 || !reflection.contains("建议");
}
```

## 7.3 生产建议
- 用 A/B 评测验证反思是否真的提升成功率。
- 保留每轮版本，出现漂移时回退。
- 对高风险任务引入人工评审兜底。

# 8. 反思与工具验证
结论先行：最强的反思来自**客观验证**，而非模型自评；能验证的场景应优先用客观验证。

## 8.1 客观验证优先原则
| 验证类型 | 客观性 | 成本 | 优先级 |
|----------|--------|------|--------|
| 编译/测试 | 高 | 低 | 最高 |
| 工具调用结果 | 高 | 中 | 高 |
| LLM 自评 | 低 | 中 | 兜底 |

## 8.2 代码修复循环实现
```java
public String codeWithVerification(String task) {
    String code = coder.write(task);

    for (int i = 0; i < 3; i++) {
        TestResult test = testRunner.run(code);      // 客观验证
        if (test.passed()) return code;

        code = coder.fix(code, test.failureOutput()); // 基于真实错误修复
    }
    return code;
}
```

## 8.3 验证器设计要点
- 快速：单次验证应远快于生成。
- 确定：同一输入结果稳定，不受随机性影响。
- 有信息量：失败时给出可定位的错误信息。

## 8.4 边界与坑
- 测试覆盖不足：通过测试不等于正确。
- 验证器本身有 bug：会误导修复方向。
- 修复引入回归：应跑全量测试而非仅跑单测。

> 能验证的场景，**优先用客观验证代替 LLM 自评**。

# 9. 反思与评测的关系
结论先行：反思是"运行时自我改进"，评测是"离线度量质量"，两者互补。

## 9.1 运行时与离线
| 维度 | 反思 | 评测 |
|------|------|------|
| 时机 | 运行时 | 离线 |
| 目的 | 改进本次 | 度量整体 |
| 输出 | 更好的答案 | 指标报告 |

## 9.2 反思效果度量
```java
public record ReflectionMetrics(int attempts, boolean success, long tokens) {}

public double successRate(List<ReflectionMetrics> data) {
    long ok = data.stream().filter(ReflectionMetrics::success).count();
    return (double) ok / data.size();
}
```

关键问题：开启反思后成功率提升多少、成本增加多少，需要做对照实验。

## 9.3 失败案例回流
- 反思的效果要用评测来验证（是否真的提升了成功率）。
- 反思产生的失败案例可回流为评测集，让评测集持续覆盖真实失败分布。

> 详见 `12_Evaluation`。

# 10. 生产注意事项
结论先行：反思上线前必须解决"有界、可观测、可回退"三件事。

## 10.1 风险清单
| 风险 | 建议 |
|------|------|
| 无轮数上限 | 限制 2~3 轮 |
| 纯模型自评 | 优先客观验证 |
| 反思无记录 | 写入 Trace，便于分析 |
| 成本失控 | 预算告警 |
| 越改越差 | 保留最佳版本，可回退 |

## 10.2 可观测性与 Trace
```java
public record ReflectionTrace(
        String task,
        int round,
        String answer,
        String critique,
        double score,
        long tokens) {}
```

Trace 至少记录：轮次、每轮输入输出、评估分数、token 消耗、终止原因。

## 10.3 部署检查表
- 有最大轮数与 token 预算。
- 评估器包含客观验证层。
- 每轮版本可回退。
- Trace 已接入监控。
- 有离线评测对照实验。

# 11. 小结
## 11.1 核心要点
- 反思 = 执行 → 评估 → 提取教训 → 改进重试。
- Reflexion 把反思结果存入记忆，供后续尝试使用。
- **客观验证优于 LLM 自评**。
- 反思必须有轮数、成本上限，并保留可回退的最佳版本。

## 11.2 落地建议
1. 从结果反思起步，有验证器优先接验证器。
2. 轮数默认 2，Critic 与 Generator 尽量解耦。
3. 用评测证明反思有效，否则只是增加成本。
