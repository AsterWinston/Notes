# 1. 设计原则

## 1.1 八条核心原则

结论先行：多 Agent 设计有八条核心原则，覆盖复杂度、职责、拓扑、状态、可观测、终止、成本与评测。

| 原则 | 说明 |
|------|------|
| 从简到繁 | 工作流 → 单 Agent → 多 Agent |
| 职责清晰 | 每个 Agent 边界明确 |
| 星型拓扑 | 优先 Supervisor，避免网状 |
| 状态外置 | 共享状态持久化 |
| 全程可观测 | 协作链完整记录 |
| 终止明确 | 步数 + 循环检测 |
| 成本可控 | 限制 Agent 数与轮数 |
| 评测驱动 | 对比单 Agent 基线 |

机制拆解：八条原则可归为三组——结构（职责、拓扑、状态）、控制（终止、成本）、反馈（可观测、评测），"从简到繁"是总纲。

边界与坑：原则之间存在张力，例如"星型拓扑"提升可控性但可能造成主管瓶颈。

生产建议：把原则作为评审清单而非教条，按任务特征权衡取舍。

## 1.2 原则背后的权衡

结论先行：每条原则都在"能力"与"复杂度"之间做取舍，理解取舍才能正确应用。

| 原则 | 收益 | 代价 |
|------|------|------|
| 从简到繁 | 避免过度设计 | 可能需要重构 |
| 职责清晰 | 减少重叠 | 边界设计成本 |
| 星型拓扑 | 通信可控 | 主管瓶颈 |
| 状态外置 | 可恢复、可审计 | 存储与一致性成本 |
| 全程可观测 | 可调试 | 存储与脱敏成本 |
| 终止明确 | 防死循环 | 可能提前截断 |
| 成本可控 | 预算安全 | 可能牺牲质量 |
| 评测驱动 | 决策有据 | 评测本身有成本 |

机制拆解：每条原则的代价都可被预算与护栏吸收，关键是显式承认代价而非假装没有。

生产建议：设计文档中显式写出每项选择的"已知代价"。

## 1.3 从简到繁的演进路径

结论先行：正确顺序是工作流 → 单 Agent → 多 Agent，逐级验证后再升级。

```
工作流（确定性编排）→ 单 Agent（ReAct + 工具）→ 多 Agent（分工协作）
```

机制拆解：每一级升级都应有"上一级失败"的量化证据，而非主观感觉；直接从工作流跳到多 Agent 会掩盖"其实工作流就够"的事实。

生产建议：每级保留回退能力，多 Agent 失败时降级为单 Agent。

# 2. 设计检查清单

## 2.1 立项检查清单

结论先行：立项阶段先回答"是否需要多 Agent"，避免无效投入。

- [ ] 是否真的需要多 Agent？（对比单 Agent）
- [ ] 每个 Agent 职责是否单一清晰？
- [ ] 编排模式是否合适（Supervisor/Swarm/流水线）？
- [ ] 是否有步数上限与循环检测？
- [ ] 交接时传递的是摘要还是全量历史？
- [ ] 共享状态是否外置并加锁？
- [ ] 是否有完整的协作 Trace？
- [ ] 是否限制了 Agent 数与轮数？
- [ ] 是否对比过单 Agent 基线？
- [ ] 是否有终止与降级策略？

机制拆解：前三条属"是否做"，后七条属"如何做"，回答"是否做"为否时其余免谈。

边界与坑：清单被当作走过场，逐项打勾却无证据支撑。

生产建议：每项附上证据链接（评测报告、Trace 样例）。

## 2.2 实现与上线检查清单

结论先行：实现阶段聚焦接口、状态、错误与测试；上线阶段确认监控、告警、降级与预算就位。

| 类别 | 检查点 |
|------|--------|
| 接口 | Agent 描述可供主管选择 |
| 状态 | 单一写者或加锁 |
| 交接 | 结构化上下文，非纯文本 |
| 错误 | 每个 Agent 有超时与重试 |
| 测试 | 单元 + 集成 + 回归 |
| 日志 | traceId 贯穿 |

| 上线项 | 检查点 |
|--------|--------|
| 监控 | 轮数、成本、失败率 |
| 告警 | 阈值与通知渠道 |
| 降级 | 部分结果或转人工 |
| 预算 | 按任务设上限并熔断 |
| 回滚 | 可切回单 Agent |

机制拆解：实现阶段的多数缺陷来自隐式契约——Agent 之间未明说的假设。

生产建议：灰度发布，先小流量验证再全量。

# 3. Agent 设计

## 3.1 好 Agent 的五个特征

结论先行：一个好 Agent 应满足单一职责、描述清晰、输入输出明确、自带工具与权限、可独立测试。

机制拆解：②是"可被编排"的前提，主管依赖描述做路由决策；⑤是"可维护"的前提。

边界与坑：描述含糊会让主管误选，导致路由错误与重复劳动。

生产建议：描述中同时写"负责什么"与"不负责什么"。

## 3.2 Agent 接口定义

结论先行：用最小接口暴露能力，用不可变记录承载请求与结果。

```java
public interface Agent {
    String name();
    String description();                  // 供主管判断何时调用
    AgentResult handle(AgentRequest request);
}

public record AgentRequest(
        String taskId, String instruction, Map<String, Object> context) {}

public record AgentResult(
        String agentName, String output, boolean success,
        List<String> citations,            // 事实来源
        Map<String, Object> meta) {}       // token、耗时
```

机制拆解：接口只保留 `name`/`description`/`handle` 三要素，其余能力通过工具与上下文注入，保持正交。

边界与坑：把状态放进 Agent 实例会导致并发问题，应通过 `AgentRequest` 传入。

生产建议：`AgentResult` 携带 `citations`，为后续事实校验提供依据。

## 3.3 Agent 注册与描述

结论先行：用注册表统一管理 Agent，主管按描述检索候选；描述质量决定路由质量。

```java
private final Map<String, Agent> agents = new ConcurrentHashMap<>();

public void register(Agent agent) {
    agents.put(agent.name(), agent);
}

public Agent get(String name) {
    Agent a = agents.get(name);
    if (a == null) {
        throw new IllegalArgumentException("未知 Agent: " + name);
    }
    return a;
}
```

机制拆解：正例 + 反例让主管的路由判断更准确，减少推诿与误派；描述过长会占用主管上下文。

生产建议：把描述作为可测试资产，用路由用例回归验证。

# 4. 编排选择

## 4.1 模式选择决策树

结论先行：按任务特征选择编排模式，优先选最简单的可用模式。

```
任务可分解、需协调 → Supervisor
固定顺序 → 流水线
独立子任务 → 并行
开放任务 → Swarm（慎用）
大型组织 → 分层
多视角决策 → 辩论
```

机制拆解：Supervisor 适合需要动态路由的场景，流水线适合确定性阶段，并行适合无依赖子任务。

边界与坑：Swarm 的自主交接最难控制，易陷入死循环，仅在充分护栏下使用。

生产建议：默认 Supervisor，除非有明确理由选择其他模式。

## 4.2 模式对比表

结论先行：六种模式在可控性、成本、延迟与适用场景上差异明显。

| 模式 | 可控性 | 成本 | 延迟 | 典型场景 |
|------|:------:|:----:|:----:|----------|
| Supervisor | 高 | 中 | 中 | 客服分流、任务分发 |
| 流水线 | 高 | 低 | 低 | 文档处理、ETL |
| 并行 | 中 | 中 | 低 | 多源检索、批量分析 |
| Swarm | 低 | 高 | 高 | 开放式探索 |
| 分层 | 中 | 高 | 高 | 大型组织式协作 |
| 辩论 | 中 | 高 | 高 | 多视角决策 |

机制拆解：可控性大致与通信自由度成反比，自由度越高越难预测。

生产建议：用 A/B 测试验证模式选择，而非仅凭直觉。

## 4.3 编排器实现示例

结论先行：一个最小 Supervisor 编排器，含步数上限与委托记录。

```java
public record SupervisorOrchestrator(
        AgentRegistry registry, Supervisor supervisor, int maxSteps) {

    public String run(String task) {
        List<String> history = new ArrayList<>();
        for (int step = 0; step < maxSteps; step++) {
            SupervisorDecision d = supervisor.decide(task, history);
            if (d.finalAnswer() != null) {
                return d.finalAnswer();
            }
            Agent agent = registry.get(d.agentName());
            history.add("[%s] %s".formatted(agent.name(),
                    agent.handle(new AgentRequest(task,
                            d.instruction(), Map.of())).output()));
        }
        return "达到最大步数，基于已有信息给出结论：\n"
                + String.join("\n", history);
    }
}

interface Supervisor {
    SupervisorDecision decide(String task, List<String> history);
}

record SupervisorDecision(String agentName, String instruction,
                          String finalAnswer) {}
```

机制拆解：循环内先让主管决策，若返回 `finalAnswer` 则终止；否则委托给指定 Agent 并记录。

边界与坑：主管可能反复委托同一 Agent，需额外做循环检测。

生产建议：把 `history` 压缩后再喂给主管，控制上下文增长。

# 5. 上下文管理

## 5.1 交接上下文的正确姿势

结论先行：交接应传"摘要 + 关键数据 + 约束"，而非全量对话历史。

```java
// 错误：传递全量历史
handoff.setContext(fullConversationHistory);

// 正确：传递摘要 + 关键数据
handoff.setContext(new HandoffContext(
        task, summarize(history),
        extractKeyData(history), constraints));
```

机制拆解：全量历史会随轮次线性膨胀，且包含大量无关内容，挤占上下文并抬高成本；摘要则可能丢失细节，故关键数据必须以结构化字段单独传递。

生产建议：为摘要设长度上限，并保留原始数据引用以便追溯。

## 5.2 摘要与结构化数据

结论先行：用不可变记录区分目标、摘要、数据与约束，避免语义混淆。

```java
public record HandoffContext(
        String task,                    // 原始目标，只读
        String summary,                 // 可滚动更新的摘要
        Map<String, Object> data,       // 精确关键数据
        List<String> constraints        // 硬约束
) {
    public HandoffContext merge(Map<String, Object> delta) {
        var merged = new HashMap<>(data);
        merged.putAll(delta);
        return new HandoffContext(task, summary, merged, constraints);
    }
}
```

机制拆解：`task` 全程不变以锚定目标；`data` 用 `merge` 合并而非替换，防止上游数据丢失。

边界与坑：键冲突会覆盖，需为不同 Agent 的数据加命名空间。

生产建议：`constraints` 在每轮重新注入，防止漂移。

## 5.3 上下文预算控制

结论先行：为每次调用设定上下文预算，超限时按优先级裁剪。

| 内容 | 优先级 | 超限处理 |
|------|:------:|----------|
| 原始目标 | 最高 | 永不裁剪 |
| 硬约束 | 高 | 永不裁剪 |
| 关键数据 | 高 | 保留字段名，截断值 |
| 近期对话 | 中 | 保留最近 k 轮 |
| 早期对话 | 低 | 摘要或丢弃 |

机制拆解：按"对结果的影响"排序，优先保留不可替代的信息。

生产建议：记录每次裁剪量，观察是否频繁触发。

# 6. 成本控制

## 6.1 六类控本手段

结论先行：限制、分层、缓存、预筛、预算与并行度控制共同构成成本护栏。

### 6.1.1 规模与单价

限制 Agent 数（≤5）；限制轮数（≤10）；模型分层，简单 Agent 用小模型。

### 6.1.2 复用与熔断

缓存复用中间结果；规则预筛把明显意图直接路由；按任务设预算上限。

机制拆解：限制管规模上限，分层降单价，缓存减重复，预筛免调用，预算做熔断。

边界与坑：缓存对非确定性输出收益有限，且可能返回过期结果。

生产建议：为缓存设置 TTL 与失效策略。

## 6.2 预算与熔断实现

结论先行：用可累计的预算对象在每次调用前检查，超限立即熔断。

```java
import java.util.concurrent.atomic.AtomicLong;

public class BudgetGuard {

    private final long maxTokens;
    private final AtomicLong used = new AtomicLong();

    public BudgetGuard(long maxTokens) {
        this.maxTokens = maxTokens;
    }

    public void charge(long tokens) {
        if (used.addAndGet(tokens) > maxTokens) {
            throw new BudgetExceededException("超出预算: " + used.get());
        }
    }
}
```

机制拆解：`charge` 在每次模型调用后累计，超限抛 `BudgetExceededException` 触发上层降级。

边界与坑：并发调用需同步，否则可能突破上限。

生产建议：熔断后返回部分结果而非直接报错，改善体验。

# 7. 可观测

## 7.1 Trace 结构

结论先行：用父子 span 还原协作链，每次交接是一个 span。

```
Trace: 用户任务
├── Span: Supervisor 决策 1
├── Span: Agent A（输入、输出、耗时、Token）
├── Span: Supervisor 决策 2
├── Span: Agent B
└── Span: 汇总
```

机制拆解：父子关系天然对应"主管 → Agent"的委托结构，可重建为树。

边界与坑：只记日志不记关联 ID，事后无法拼出完整链路。

生产建议：接入现有 APM 体系，避免自建孤岛。

## 7.2 指标、日志与埋点

结论先行：指标看趋势，日志看细节，二者共享 traceId 才能关联；埋点应收敛到统一入口。

| 类型 | 内容 | 用途 |
|------|------|------|
| 指标 | 轮数、成本、失败率、延迟 | 监控告警 |
| 日志 | 输入输出、决策理由 | 排障归因 |
| Trace | 调用链 | 性能与链路分析 |

机制拆解：`traceStore` 负责明细，`registry` 负责聚合指标，二者共享 `agentName` 标签以便关联。

边界与坑：高基数标签（如 taskId）会拖垮指标系统，应仅用于日志而非指标。

生产建议：记录入口只保留一处，避免埋点分散导致口径不一致。

# 8. 测试

## 8.1 测试金字塔

结论先行：多 Agent 测试分单元、集成、回归三层，越往上越贴近真实但越慢。

| 层次 | 对象 | 特点 |
|------|------|------|
| 单元 | 单 Agent、编排逻辑 | Mock 依赖，快 |
| 集成 | 多 Agent 协作 | 真实模型，慢 |
| 回归 | 固定用例集 | 防提示词退化 |

机制拆解：单元测试用 Mock 保证确定性，集成测试验证协作，回归测试守护整体质量。

生产建议：CI 中跑单元，夜间跑集成与回归。

## 8.2 单元测试示例

结论先行：用 Mock 验证主管的路由与终止行为。

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.never;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

import java.util.List;
import java.util.Map;
import org.junit.jupiter.api.Test;

class SupervisorOrchestratorTest {

    @Test
    void supervisor_should_delegate_correctly() {
        Agent orderAgent = mock(Agent.class);
        Agent refundAgent = mock(Agent.class);
        when(orderAgent.name()).thenReturn("orderAgent");
        when(refundAgent.name()).thenReturn("refundAgent");
        when(refundAgent.handle(any())).thenReturn(
                new AgentResult("refundAgent", "退款成功", true,
                        List.of(), Map.of()));

        AgentRegistry registry = new AgentRegistry();
        registry.register(orderAgent);
        registry.register(refundAgent);

        Supervisor supervisor = (task, history) -> history.isEmpty()
                ? new SupervisorDecision("refundAgent", "处理退款", null)
                : new SupervisorDecision(null, null, "完成");

        var orchestrator =
                new SupervisorOrchestrator(registry, supervisor, 5);
        assertEquals("完成", orchestrator.run("我要退款"));
        verify(refundAgent).handle(any());
        verify(orderAgent, never()).handle(any());
    }
}
```

机制拆解：第一个测试验证"只调用正确的 Agent"，第二个验证"循环被步数上限截断"。

边界与坑：过度依赖 Mock 会掩盖真实模型行为，需与集成测试配合。

生产建议：为每条护栏（步数、预算、循环）各写一个单元测试。

## 8.3 集成与回归测试

结论先行：集成测试用真实模型验证端到端，回归测试用固定用例防退化。

| 用例类型 | 目的 | 断言方式 |
|----------|------|----------|
| 正常分解 | 验证主流程 | 包含关键要素 |
| 边界输入 | 验证兜底 | 不崩溃、有降级 |
| 诱导循环 | 验证检测生效 | 在步数内终止 |
| 成本超限 | 验证熔断 | 返回部分结果 |

机制拆解：断言"包含关键要素"与"未超预算"，而非逐字匹配，避免非确定性导致误报。

生产建议：每次修改提示词或拓扑后跑全量回归。

# 9. 反模式

## 9.1 反模式总表

结论先行：常见反模式及其正解如下。

| 反模式 | 问题 | 正解 |
|--------|------|------|
| 盲目多 Agent | 复杂度不值 | 先验证收益 |
| 网状通信 | 通信爆炸 | 星型拓扑 |
| 传全量历史 | 上下文爆炸 | 传摘要 |
| 无步数上限 | 死循环 | 上限 + 检测 |
| 职责重叠 | 重复劳动 | 边界清晰 |
| 无 Trace | 无法调试 | 完整记录 |
| 不对比基线 | 不知是否有效 | 对比单 Agent |

机制拆解：反模式多数源于"跳过验证"与"缺少护栏"两类根因。

生产建议：把反模式清单纳入代码评审检查项。

## 9.2 反模式案例拆解

结论先行：以"传全量历史"为例，拆解其如何从方便演变为灾难。

```
初始：2 个 Agent，历史短，无感
   │ 轮次增加 → 历史线性增长 → 上下文膨胀 → 成本上升
   │ 超上下文窗口 → 被迫截断 → 早期信息丢失 → 目标漂移
```

机制拆解：问题在早期被小规模掩盖，随规模增长才暴露，因此必须提前设计。

边界与坑：等到线上出现成本告警才治理，往往需要重构交接协议。

生产建议：设计阶段就采用结构化摘要交接。

## 9.3 从反模式到正解

结论先行：每条反模式都有对应的最小修复动作，可逐步落地。

| 反模式 | 最小修复 | 后续优化 |
|--------|----------|----------|
| 盲目多 Agent | 加基线对比 | 建立升级门槛 |
| 网状通信 | 收敛为星型 | 白名单直连 |
| 传全量历史 | 改为摘要 + 数据 | 上下文预算 |
| 无步数上限 | 加 `maxSteps` | 循环与停滞检测 |
| 职责重叠 | 写清边界 | 路由回归测试 |
| 无 Trace | 加 traceId | 接入 APM |
| 不对比基线 | 跑单 Agent | 纳入 CI |

机制拆解：先做最小修复止血，再做后续优化根治，避免一次性大改。

生产建议：把修复动作排入迭代，明确负责人与验收标准。

# 10. 小结

## 10.1 要点回顾

结论先行：本模块的核心结论可归纳为五点。

- 从简到繁：工作流 → 单 Agent → 多 Agent。
- 职责清晰、星型拓扑、状态外置、全程可观测。
- 传摘要而非全量历史，限制 Agent 数与轮数。
- 必须对比单 Agent 基线。
- 牢记反模式，避免重复踩坑。

## 10.2 落地检查清单

结论先行：按阶段对照检查，确保每一步有验收标准。

| 阶段 | 关键动作 | 验收标准 |
|------|----------|----------|
| 立项 | 对比单 Agent | 有基线数据 |
| 设计 | 职责、拓扑、状态 | 检查清单通过 |
| 实现 | 接口、护栏、Trace | 单测覆盖护栏 |
| 测试 | 单元 + 集成 + 回归 | 全绿 |
| 上线 | 监控、告警、降级 | 灰度通过 |
| 运营 | 复评与调优 | 指标达标 |
