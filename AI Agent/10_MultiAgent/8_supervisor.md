# 1. 概述与定位

## 1.1 什么是 Supervisor 模式

结论先行：Supervisor（主管，又称 Orchestrator 编排者）模式引入一个中心化的主管 Agent，由它读取用户任务、查看已完成工作，决定"下一步交给哪个子 Agent"或"任务已完成"，并在结束时把各子 Agent 的结果汇总成最终答复。

机制拆解：

1. 主管持有全局视图：任务目标、可用 Agent 清单、历史结果。
2. 每轮只做一次决策，产出结构化的 `SupervisorDecision`。
3. 子 Agent 无状态、无全局意识，只按指令执行并返回文本结果。
4. 结果回填到 history，进入下一轮决策，直到 FINISH 或触达步数上限。

```
              ┌──────────────┐
              │  Supervisor  │  ← 决定下一步
              └──────┬───────┘
        ┌────────────┼────────────┐
        ▼            ▼            ▼
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ Agent A  │  │ Agent B  │  │ Agent C  │
   └────┬────┘  └────┬────┘  └────┬────┘
        └────────────┼────────────┘
                     ▼
              ┌──────────────┐
              │  Supervisor  │  ← 汇总或继续
              └──────────────┘
```

边界与坑：主管不是"更聪明的 Agent"，而是一个受限的路由器；一旦让主管也去执行具体任务，就会退化成单 Agent，并丧失可审计性。

生产建议：主管必须使用较强模型，子 Agent 可按任务难度分层选型。

## 1.2 与其他多智能体模式的对比

结论先行：Supervisor 是"中心化编排"，与 Swarm（去中心化交接）、Hierarchical（多层主管）、Pipeline（固定流水线）在控制权与可追踪性上差异明显。

| 模式 | 决策者 | 拓扑 | 可追踪性 | 循环风险 | 典型场景 |
|------|--------|------|----------|----------|----------|
| Supervisor | 中心主管 | 星型 | 高 | 低 | 可分解的复杂任务 |
| Swarm | 各 Agent | 网状 | 低 | 高 | 开放、探索型任务 |
| Hierarchical | 多层主管 | 树型 | 中 | 低 | 超大规模任务 |
| Pipeline | 固定流程 | 链型 | 高 | 无 | 步骤固定的流程 |

机制拆解：控制权集中在单点，因此可追踪性与循环可控性最好，代价是单点瓶颈与每轮额外的模型调用。

边界与坑：表中"循环风险低"是相对的——若提示词让子 Agent 反复重试，仍会空转。

生产建议：任务可分解、需要审计时优先选 Supervisor；探索型任务再考虑 Swarm。

## 1.3 适用与不适用场景

### 1.3.1 适用场景

结论先行：任务能被分解为若干相对独立、职责清晰的子任务时，Supervisor 最合适。

- 需要多种专业能力（检索、计算、写作、代码执行）协同。
- 需要审计每一步"为什么这么走"。
- 子任务之间存在依赖顺序，需要动态决定。

### 1.3.2 不适用场景

结论先行：任务高度串行且步骤固定、或子任务间需大量自由协商时，Supervisor 收益低。

- 单一能力即可完成：直接用单 Agent。
- 需要 Agent 之间频繁双向协商：考虑群聊/辩论模式。
- 极端开放、边界模糊：考虑 Swarm。

边界与坑：为"看起来更高级"而强上 Supervisor，会引入无谓延迟与成本。

生产建议：先用单 Agent 基线，确认瓶颈确实来自"需要多种能力"后再升级。

# 2. 结构与运行机制

## 2.1 拓扑结构

结论先行：Supervisor 采用星型拓扑，所有子 Agent 只与主管通信，彼此不直连，从根本上避免了 O(n²) 的通信爆炸。

机制拆解：

1. 主管节点保存 `Map<String, Agent>` 注册表。
2. 每个子 Agent 对外只暴露 `description()` 与 `handle(instruction)`。
3. 消息只沿"主管→子 Agent→主管"两条边流动。

```
        ┌───────────────────────────┐
        │        Supervisor         │
        └───┬───────┬───────┬───────┘
            │       │       │
        ┌───▼─┐ ┌───▼─┐ ┌───▼─┐
        │  A  │ │  B  │ │  C  │
        └─────┘ └─────┘ └─────┘
   子 Agent 之间不直接通信（星型）
```

边界与坑：星型拓扑要求主管能完整描述每个子 Agent 的能力，描述过粗会导致路由错误。

生产建议：为每个 Agent 维护稳定的名称、描述与输入输出契约。

## 2.2 单轮生命周期

### 2.2.1 决策阶段

结论先行：决策阶段把"任务 + Agent 清单 + 历史"压缩成一次模型调用，产出下一动作。

机制拆解：主管看到的是被裁剪过的历史摘要，而非全部原始消息，避免上下文无限膨胀。

### 2.2.2 委托阶段

结论先行：委托阶段把决策中的 `action` 解析为具体 Agent，并传入 `instruction`。

边界与坑：必须处理"未知 Agent"的情况，否则会抛异常中断整个流程。

### 2.2.3 收集阶段

结论先行：收集阶段把子 Agent 的结果以 `agent: result` 的形式追加到 history。

机制拆解：结果既供下一轮决策参考，也是最终汇总的素材。

生产建议：为每条历史记录带上轮次编号与耗时，便于回溯。

## 2.3 终止条件与结果汇总

结论先行：终止有两个触发点——主管主动返回 FINISH，或步数达到上限；两种情况都进入汇总。

| 终止条件 | 触发方 | 处理 |
|----------|--------|------|
| 主管判定完成 | Supervisor | 正常汇总 |
| 达到步数上限 | 编排器 | 强制汇总并标注未完成 |
| 未知 Agent | 编排器 | 记录错误并继续 |
| 子 Agent 异常 | 编排器 | 记录错误并继续 |

边界与坑：达到上限时若直接返回"达到最大步数"，用户拿不到任何有用结果；应尽力汇总已有进展。

生产建议：汇总结果里显式标注是否完整、缺失了哪些步骤。

# 3. 决策模型

## 3.1 决策数据结构

结论先行：用不可变 record 承载决策，字段固定为动作、指令、理由三部分。

```java
public record SupervisorDecision(
        String action,          // 委托给谁 / FINISH
        String instruction,     // 给该 Agent 的指令
        String reason           // 决策理由，便于审计
) {}
```

机制拆解：`action` 既可能是 Agent 名称，也可能是特殊值 `FINISH`；用字符串而非枚举便于模型直接产出。

边界与坑：模型可能输出大小写不一致或带空格的名称，需要归一化。

生产建议：把 `FINISH` 等保留字集中定义为常量，避免散落魔法字符串。

## 3.2 决策提示词设计

结论先行：提示词要让模型明确"可选动作集合"和"输出格式"，并禁止它直接答题。

```java
interface Supervisor {

    @UserMessage("""
            你是任务主管，负责协调多个专业 Agent 完成用户任务。

            可用 Agent：
            {{agents}}

            用户任务：{{task}}
            已完成的工作：
            {{history}}

            决定下一步：
            - 委托给某个 Agent（输出其名称与指令）
            - 或任务完成（输出 FINISH）

            只输出 JSON，不要解释：
            {"action":"agentName","instruction":"...","reason":"..."}
            """)
    SupervisorDecision decide(@V("agents") String agents,
                              @V("task") String task,
                              @V("history") String history);
}
```

机制拆解：把"可用 Agent 清单"动态注入，模型才能在合法集合内选择。

边界与坑：若历史为空字符串，模型可能误判任务未开始；应给出占位提示如"（暂无）"。

生产建议：在提示词中显式要求"若信息已足够，输出 FINISH"，减少多余轮次。

## 3.3 决策输出校验与容错

结论先行：模型输出不可全信，必须做解析校验，失败时回退到默认动作。

```java
public SupervisorDecision safeDecide(String agents, String task,
                                     String history) {
    try {
        SupervisorDecision d = supervisor.decide(agents, task, history);
        String action = normalize(d.action());
        if (!"FINISH".equals(action) && !agentNames.contains(action)) {
            return new SupervisorDecision("FINISH",
                    "未知动作，提前结束", "fallback");
        }
        return new SupervisorDecision(action, d.instruction(), d.reason());
    } catch (Exception e) {
        return new SupervisorDecision("FINISH",
                "决策解析失败，提前结束", "exception:" + e.getMessage());
    }
}

private String normalize(String s) {
    return s == null ? "FINISH" : s.trim().toUpperCase();
}
```

边界与坑：连续多次解析失败意味着主管模型或提示词有问题，应触发告警而非静默降级。

生产建议：统计"回退率"，作为主管模型质量的监控指标。

# 4. 完整 Java 实现

## 4.1 核心接口

结论先行：先定义 `Agent` 与 `Synthesizer` 两个接口，再实现编排器。

```java
public interface Agent {
    String name();
    String description();
    String handle(String instruction);
}

public interface Synthesizer {
    String synthesize(String task, List<String> history);
}
```

机制拆解：`Agent` 只暴露能力描述与执行入口，保持无状态；`Synthesizer` 独立于主管，便于替换汇总策略。

边界与坑：若让 `Agent.handle` 返回结构化对象，需额外定义契约；初版用纯文本最省事。

生产建议：为 `handle` 增加超时与异常包装，避免单点卡死整条流程。

## 4.2 编排器

结论先行：编排器是主循环，负责决策、委托、收集、终止。

```java
public class SupervisorOrchestrator {

    private final Supervisor supervisor;
    private final Synthesizer synthesizer;
    private final Map<String, Agent> agents;

    public SupervisorOrchestrator(Supervisor supervisor,
                                  Synthesizer synthesizer,
                                  List<Agent> agentList) {
        this.supervisor = supervisor;
        this.synthesizer = synthesizer;
        this.agents = agentList.stream()
                .collect(Collectors.toMap(Agent::name, a -> a));
    }

    public String run(String task, int maxSteps) {
        List<String> history = new ArrayList<>();
        for (int step = 0; step < maxSteps; step++) {
            SupervisorDecision decision = supervisor.decide(
                    describeAgents(), task, joinHistory(history));

            if ("FINISH".equals(decision.action())) {
                return synthesize(task, history);
            }

            Agent agent = agents.get(decision.action());
            if (agent == null) {
                history.add("错误：未知 Agent " + decision.action());
                continue;
            }

            String result;
            try {
                result = agent.handle(decision.instruction());
            } catch (Exception e) {
                result = "执行失败：" + e.getMessage();
            }
            history.add("%s: %s".formatted(decision.action(), result));
        }
        return synthesize(task, history);
    }

    private String synthesize(String task, List<String> history) {
        return synthesizer.synthesize(task, history);
    }

    private String joinHistory(List<String> history) {
        return history.isEmpty() ? "（暂无）" : String.join("\n", history);
    }
}
```

机制拆解：主循环结构固定，变化点只有"决策"和"执行"两处，便于替换与测试。

边界与坑：`Collectors.toMap` 遇到重名 Agent 会抛异常，应在注册时校验名称唯一。

生产建议：把 `maxSteps` 做成可配置项，并按任务类型给不同上限。

## 4.3 Agent 注册与描述

结论先行：主管能正确路由的前提，是 Agent 描述既简洁又能区分职责。

```java
public String describeAgents() {
    return agents.entrySet().stream()
            .map(e -> "- %s：%s".formatted(
                    e.getKey(), e.getValue().description()))
            .collect(Collectors.joining("\n"));
}
```

```
- retriever：检索企业知识库
- analyzer：分析数据并给出洞察
- writer：撰写报告
- calculator：执行数学计算
```

机制拆解：描述文本会进入主管提示词，直接决定路由质量。

边界与坑：描述里若出现重叠职责（如 analyzer 和 writer 都写"总结"），主管会摇摆。

生产建议：为每个描述补一句"不负责什么"，进一步划清边界。

## 4.4 汇总器与可运行示例

结论先行：汇总器把散落的历史组织成面向用户的最终答复。

```java
public class LlmSynthesizer implements Synthesizer {

    private final ChatModel model;

    public LlmSynthesizer(ChatModel model) {
        this.model = model;
    }

    @Override
    public String synthesize(String task, List<String> history) {
        String prompt = """
                用户任务：%s
                各 Agent 的工作记录：
                %s

                请综合以上记录，直接给出面向用户的最终答复。
                """.formatted(task, String.join("\n", history));
        return model.chat(prompt);
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        List<Agent> agents = List.of(
                new RetrieverAgent(), new AnalyzerAgent(),
                new WriterAgent(), new CalculatorAgent());
        var orchestrator = new SupervisorOrchestrator(
                new LlmSupervisor(), new LlmSynthesizer(model), agents);
        String answer = orchestrator.run(
                "统计上季度各区域销售额并撰写摘要", 8);
        System.out.println(answer);
    }
}
```

边界与坑：汇总同样是一次模型调用，若历史很长会挤占上下文；应对历史做摘要压缩。

生产建议：汇总提示词里要求"注明数据来源与不确定项"。

# 5. 关键优化

## 5.1 规则预筛

结论先行：明显的意图无需调用主管模型，直接用规则路由，可显著降本降延迟。

```java
public Optional<String> preRoute(String task) {
    String t = task.toLowerCase();
    if (t.contains("计算") || t.contains("求和")) {
        return Optional.of("calculator");
    }
    if (t.contains("检索") || t.contains("查一下")) {
        return Optional.of("retriever");
    }
    return Optional.empty();
}
```

机制拆解：预筛命中则跳过决策调用；未命中再走模型主管。

边界与坑：规则过强会误伤，需保证未命中时能平滑回落到模型决策。

生产建议：对规则命中率与准确率分别埋点，持续校准关键词。

## 5.2 决策缓存

结论先行：相似的任务前缀往往产生相同决策，可缓存决策结果复用。

| 缓存键 | 值 | 失效策略 |
|--------|-----|----------|
| 任务指纹 + 历史指纹 | SupervisorDecision | TTL + 容量上限 |
| 任务指纹 | 预筛动作 | 规则变更时清空 |

边界与坑：历史稍有变化就会改变指纹，命中率可能很低，需先测再上。

生产建议：缓存只用于幂等、无副作用的决策，不要缓存含用户隐私的原始文本。

## 5.3 并行委托

结论先行：当多个子任务互不依赖时，主管可一次分派多个 Agent 并行执行。

```java
public List<String> fanOut(List<SupervisorDecision> decisions) {
    return decisions.parallelStream()
            .map(d -> {
                Agent a = agents.get(d.action());
                return "%s: %s".formatted(d.action(), a.handle(d.instruction()));
            })
            .toList();
}
```

机制拆解：并行需要决策支持"一次返回多个动作"，即把 `action` 扩展为动作列表。

边界与坑：并行写共享状态会引发竞争，务必让子 Agent 无共享副作用。

生产建议：并行度受下游限流约束，应设置线程池上限与超时。

# 6. 边界与失败模式

## 6.1 无限循环

结论先行：主管可能反复委托同一 Agent 或来回切换，导致空转。

机制拆解：常见触发是子 Agent 结果不满足主管预期，主管重试但指令不变。

```java
if (step > 2 && lastTwoIdentical(history)) {
    history.add("检测到重复，强制汇总");
    return synthesize(task, history);
}
```

边界与坑：仅靠步数上限能兜底但浪费预算；更早检测重复可省成本。

生产建议：同时使用步数上限、重复检测与超时三重保险。

## 6.2 主管决策错误

结论先行：主管选错 Agent 或给出模糊指令，会让整条链路跑偏。

对策清单：

- 用强模型做主管，并给出高质量 Agent 描述。
- 在提示词中列出"反例"：什么情况不该选某个 Agent。
- 对高风险动作增加规则兜底与人工确认。

边界与坑：一味提高提示词长度会稀释关键信息，需精炼而非堆砌。

生产建议：记录每轮决策理由，离线抽样评估路由准确率。

## 6.3 上下文膨胀与成本

结论先行：轮数越多、历史越长，每轮决策的 token 成本线性上升。

| 轮次 | 历史条目 | 粗略 token 增长 |
|:----:|:--------:|:---------------:|
| 1 | 0 | 基线 |
| 4 | 4 | 约 2 倍 |
| 8 | 8 | 约 3 倍 |

边界与坑：把所有原始输出都塞进历史，会快速触顶上下文窗口。

生产建议：对历史做滚动摘要，只保留结论与关键数据。

# 7. 生产注意事项

## 7.1 可观测性与追踪

结论先行：必须记录每轮的输入摘要、决策、耗时与结果，形成可回放的协作链。

| 字段 | 说明 |
|------|------|
| traceId | 贯穿整条任务 |
| step | 轮次序号 |
| action | 选中 Agent 或 FINISH |
| reason | 主管决策理由 |
| latencyMs | 单轮耗时 |
| tokens | 单轮 token 消耗 |

生产建议：以 traceId 为维度做聚合看板，快速定位异常轮次。

## 7.2 成本与预算

结论先行：为每次任务设置 token 与金额预算，超限即降级。

| 手段 | 说明 |
|------|------|
| 限制轮数 | `maxSteps` 建议 ≤ 10 |
| 限制 Agent 数 | 建议 ≤ 5 |
| 模型分层 | 主管强模型，执行用小模型 |
| 预算熔断 | 超限转为直接汇总 |

生产建议：把预算与 `maxSteps` 一起纳入配置中心，便于灰度调整。

## 7.3 权限与安全

结论先行：子 Agent 可能调用工具，必须做最小权限与输入校验。

- 工具按 Agent 白名单授予，避免越权。
- 对子 Agent 输出做注入检测，防止提示注入扩散。
- 敏感操作需人工确认或二次授权。

边界与坑：中心化主管一旦被提示注入劫持，可能指挥所有子 Agent，风险被放大。

生产建议：在主管与子 Agent 之间增加输出校验层。

# 8. 小结与落地清单

## 8.1 要点回顾

- Supervisor 由主管决定下一步交给谁。
- 每轮：决策 → 委托 → 收集结果 → 再决策。
- 流程清晰可控，是最常用的多 Agent 模式。
- 主管需强模型，可用规则预筛优化。
- 必须设步数上限，并做重复检测与超时。

## 8.2 落地清单

| 项目 | 检查点 |
|------|--------|
| 拓扑 | 星型，子 Agent 不直连 |
| 决策 | 结构化输出 + 校验兜底 |
| 终止 | FINISH、步数上限、重复检测 |
| 汇总 | 标注完整性与不确定项 |
| 可观测 | traceId + 每轮记录 |
| 成本 | 轮数、Agent 数、预算熔断 |
