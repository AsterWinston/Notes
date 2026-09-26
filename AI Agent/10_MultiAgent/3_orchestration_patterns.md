# 1. 模式总览

**结论先行**：编排（Orchestration）解决的核心问题是"下一步交给谁"。六种常见模式——Supervisor、Swarm、流水线、并行、分层、辩论——分别对应不同的决策者与拓扑。选错模式，后面所有优化都是逆水行舟。

**原理**：编排 = 控制流 + 数据流。控制流决定"谁先谁后、谁听谁的"，数据流决定"上一步的产物如何传给下一步"。模式差异本质上是这两条流的不同组织方式。

| 模式 | 决策者 | 特点 |
|------|--------|------|
| Supervisor | 主管 Agent | 中心化分发 |
| Swarm | Agent 间自主 | 去中心化交接 |
| 流水线 | 预定义顺序 | 顺序处理 |
| 并行 | 预定义 | 同时处理 |
| 分层 | 多层主管 | 组织结构 |
| 辩论 | 规则 | 多视角讨论 |

## 1.1 六种编排模式

**结论先行**：先分清"谁决策"和"是否并行"，模式就基本确定了。

### 1.1.1 中心化模式

**原理**：由单一 Supervisor 决定每一步的执行者。控制流集中，天然可审计、可设步数上限。

**边界与坑**：Supervisor 是单点，既是瓶颈也是强模型成本的集中点。

### 1.1.2 去中心化模式

**原理**：Agent 之间自主交接（handoff，控制权转移），没有全局调度者。灵活但难追踪。

**边界与坑**：容易形成 A→B→A 的循环，必须有全局步数上限兜底。

### 1.1.3 静态与动态组合

**原理**：流水线、并行是静态编排（流程预先写死）；Supervisor、Swarm 是动态编排（运行时决定）。分层与辩论可视为两者的混合。

## 1.2 决策者与拓扑

**结论先行**：决策者在哪里，故障就在哪里集中。

```
Supervisor（中心化）        Swarm（去中心化）

     ┌────────┐                  A ──▶ B
     │ 主管   │                  ▲      │
     └───┬────┘                  │      ▼
    ┌────┼────┐                  D ◀── C
    ▼    ▼    ▼
    A    B    C
```

| 模式 | 决策者 | 拓扑 | 故障集中点 |
|------|--------|------|------------|
| Supervisor | 主管 | 星型 | 主管 |
| Swarm | 各 Agent | 网状 | 无（但易失控） |
| 流水线 | 无（预定义） | 链式 | 任一环节 |
| 并行 | 无（预定义） | 扇出/扇入 | 汇总节点 |
| 分层 | 多层主管 | 树型 | 各层主管 |
| 辩论 | 规则 | 环形 | 轮次上限 |

# 2. Supervisor（主管模式）

**结论先行**：Supervisor 是最常用的多 Agent 模式。主管负责决策"下一步交给谁"，Worker 只负责执行，最后主管汇总。

**原理**：主管持有全局任务描述与历史记录，每一步输出一个"下一个 Agent 名称"或"DONE"。这等价于把 ReAct（推理-行动）循环的"行动"限定为"调用某个 Agent"。

## 2.1 机制

**结论先行**：Supervisor 的机制可拆成"决策循环"与"汇总"两个阶段。

### 2.1.1 决策循环

**原理**：每轮循环做三件事——主管决策、Worker 执行、结果入历史。历史是主管判断的依据，也是上下文增长的主要来源。

```
┌──────────────┐
│  Supervisor  │  ← 决策：下一步交给谁
└──────┬───────┘
  ┌────┼────┐
  ▼    ▼    ▼
 A    B    C
```

### 2.1.2 汇总

**原理**：当主管判定 DONE，用 `synthesize` 把历史中的产物合并为最终答案。

**边界与坑**：汇总若只是拼接历史，会把中间过程暴露给用户。应当"重新生成"而非"复制粘贴"。

## 2.2 完整实现

```java
public class SupervisorOrchestrator {

    private static final int MAX_STEPS = 12;

    private final Supervisor supervisor;
    private final AgentRegistry registry;

    public SupervisorOrchestrator(Supervisor supervisor, AgentRegistry registry) {
        this.supervisor = supervisor;
        this.registry = registry;
    }

    public String supervise(String task) {
        List<String> history = new ArrayList<>();

        for (int step = 0; step < MAX_STEPS; step++) {
            // ① 主管决策：返回 Agent 名或 DONE
            String next = supervisor.decide(task, history);

            if ("DONE".equals(next)) {
                return supervisor.synthesize(task, history);
            }

            // ② 执行：Worker 只关心任务与历史
            Agent worker = registry.get(next);
            if (worker == null) {
                history.add("system: 未知 Agent " + next + "，已忽略");
                continue;
            }
            String result = worker.handle(task, history);
            history.add(next + ": " + result);
        }
        return "达到最大步数仍未完成，请人工介入";
    }
}
```

| 优点 | 缺点 |
|------|------|
| 流程清晰可控 | 主管可能成为瓶颈 |
| 易调试 | 主管需强模型 |
| 可审计 | 中心化 |

## 2.3 边界与坑

**结论先行**：Supervisor 的坑集中在"主管决策质量"与"历史膨胀"两处。

| 坑 | 现象 | 对策 |
|----|------|------|
| 主管幻觉 | 决策出不存在或错误的 Agent | 决策结果做白名单校验 |
| 历史膨胀 | 每轮全量历史导致 Token 暴涨 | 只保留最近 N 轮 + 摘要 |
| 主管不收敛 | 反复调用同一 Agent | 检测重复动作并强制 DONE |
| 汇总失真 | 汇总时丢失关键结论 | 汇总提示词显式要求保留约束 |
| 瓶颈 | 主管串行等待 | 主管用低延迟模型，Worker 并行 |

**生产建议**：给主管的决策输出加 JSON Schema 约束，并用白名单校验；对"同一 Agent 连续调用超过 2 次"设熔断。

# 3. Swarm（群体模式）

**结论先行**：Swarm 把"下一步交给谁"的决定权下放给每个 Agent。适合流程开放、无法预先编排的任务，但必须用全局步数上限兜底。

**原理**：每个 Agent 在完成自己的部分后，可以返回一个"下一个 Agent"（handoff，交接）。控制权在 Agent 之间流转，没有全局调度者。

## 3.1 机制

```
Agent 间自主交接：

  Agent A ──"这个问题需要 B"──▶ Agent B ──▶ Agent C
```

| 组成 | 作用 |
|------|------|
| current | 当前持有控制权的 Agent |
| input | 传给当前 Agent 的输入 |
| nextAgent | Agent 声明的下一个执行者 |
| isFinal | 是否终结 |

**边界与坑**：Agent 声明 `nextAgent` 时可能指向自己，形成自循环。

## 3.2 完整实现

```java
public record AgentResult(String output, String nextAgent, boolean isFinal) {}

public class SwarmOrchestrator {

    private static final int MAX_STEPS = 15;

    private final AgentRegistry registry;

    public SwarmOrchestrator(AgentRegistry registry) {
        this.registry = registry;
    }

    public String swarm(String task, String startAgent) {
        String current = startAgent;
        String input = task;
        Set<String> visited = new HashSet<>();

        for (int step = 0; step < MAX_STEPS; step++) {
            if (!visited.add(current + "#" + step)) {
                return "检测到重复交接，终止：" + current;
            }

            AgentResult result = registry.get(current).handle(input);

            if (result.isFinal()) {
                return result.output();
            }
            // 交接给下一个 Agent
            current = result.nextAgent();
            input = result.output();
        }
        return "达到最大步数仍未完成，请人工介入";
    }
}
```

| 优点 | 缺点 |
|------|------|
| 去中心化、灵活 | 难追踪 |
| 无单点瓶颈 | 易循环 |
| 适合开放任务 | 难调试 |

## 3.3 边界与坑

| 坑 | 现象 | 对策 |
|----|------|------|
| 无限循环 | A→B→A 反复 | 全局步数上限 + 访问记录 |
| 责任扩散 | 无人对最终结果负责 | 指定终结 Agent |
| 交接信息丢失 | 下游不知上游约束 | 传递 HandoffContext |
| 无全局视角 | 无法判断是否该停 | 周期性"裁判" Agent |
| 难调试 | 链路长且非线性 | 每步记录 Trace |

**生产建议**：Swarm 只用于"专家各自领域清晰、交接语义明确"的场景；否则退回 Supervisor。

# 4. 流水线

**结论先行**：流水线是静态编排的最简形式——顺序执行，前一个的输出是后一个的输入。可预测、易测试，但不适合需要回退的任务。

**原理**：把任务拆成固定阶段，每个阶段由一个 Agent 负责。控制流是一条直线。

## 4.1 机制

```
Agent A → Agent B → Agent C
（顺序执行，前一个的输出是后一个的输入）
```

| 阶段 | 职责 | 输入 | 输出 |
|------|------|------|------|
| 检索 | 收集资料 | 问题 | 文档列表 |
| 分析 | 提炼要点 | 文档列表 | 分析结论 |
| 写作 | 成文 | 分析结论 | 草稿 |

**边界与坑**：直线流程中，任一阶段出错都会污染后续所有阶段，且无法回头。

## 4.2 实现

```java
public class PipelineOrchestrator {

    private final List<Agent> stages;

    public PipelineOrchestrator(Agent research, Agent analyze, Agent write) {
        this.stages = List.of(research, analyze, write);
    }

    public String pipeline(String input) {
        String result = input;
        for (Agent agent : stages) {
            result = agent.handle(result);
            if (result == null || result.isBlank()) {
                throw new PipelineStageException(
                        "阶段 " + agent.name() + " 产出为空");
            }
        }
        return result;
    }
}
```

## 4.3 边界与坑

| 坑 | 现象 | 对策 |
|----|------|------|
| 错误传播 | 早期错误污染全链路 | 每阶段做校验 |
| 无回退 | 后期发现问题无法重来 | 关键阶段加重试/重生成 |
| 阶段耦合 | 输出格式变化影响下游 | 定义阶段间契约 |
| 延迟叠加 | 串行链路延迟高 | 可并行的阶段改并行 |

**生产建议**：为每个阶段定义明确的输入输出契约，并在阶段边界做 Schema 校验。

# 5. 并行

**结论先行**：当子任务相互独立时，并行能把延迟从"求和"降到"取最大值"。关键在汇总（aggregation）的质量。

**原理**：把输入同时发给多个 Agent，各自独立处理，最后统一汇总。

## 5.1 机制

```
         ┌──▶ Agent A ──┐
输入 ────┼──▶ Agent B ──┼───▶ 汇总
         └──▶ Agent C ──┘
```

| 要素 | 说明 |
|------|------|
| 扇出 | 输入分发给多个 Agent |
| 并行度 | 同时执行的 Agent 数 |
| 汇总 | 合并多个结果 |

**边界与坑**：并行度不等于越大越好——受限于线程池、模型 QPS（每秒查询数）与成本预算。

## 5.2 实现

**结论先行**：JDK 17 用 `CompletableFuture`，JDK 21 可用虚拟线程。

### 5.2.1 CompletableFuture

```java
public String parallel(String input) {
    List<CompletableFuture<String>> futures = agents.stream()
            .map(a -> CompletableFuture.supplyAsync(() -> a.handle(input), executor))
            .toList();

    List<String> results = futures.stream()
            .map(CompletableFuture::join).toList();

    return aggregator.aggregate(input, results);
}
```

### 5.2.2 虚拟线程

```java
public String parallelVirtual(String input) throws InterruptedException {
    List<String> results;
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        List<StructuredTaskScope.Subtask<String>> tasks = agents.stream()
                .map(a -> scope.fork(() -> a.handle(input)))
                .toList();
        scope.join().throwIfFailed();
        results = tasks.stream()
                .map(StructuredTaskScope.Subtask::get)
                .toList();
    }
    return aggregator.aggregate(input, results);
}
```

**边界与坑**：`join()` 是阻塞等待；若某个 Future 抛异常且未处理，会以 `CompletionException` 形式抛出，需统一捕获。

## 5.3 边界与坑

| 坑 | 现象 | 对策 |
|----|------|------|
| 汇总不当 | 简单拼接导致内容冲突 | 用模型做去重与裁决 |
| 成本翻倍 | N 个 Agent 同时调用 | 设并行度上限与预算 |
| 线程耗尽 | 共享线程池被占满 | 独立线程池 + 隔离 |
| 异常吞没 | 一个失败整体失败 | 允许部分成功 + 降级 |
| 顺序丢失 | 结果顺序不稳定 | 按 Agent 名排序 |

**生产建议**：并行任务应支持"部分成功"——失败的分支记录后继续汇总，而不是整体失败。

# 6. 分层

**结论先行**：分层用多层主管管理大量 Agent。适合大型任务，但通信与协调成本随层数上升，通常不超过三层。

**原理**：顶层主管把任务拆给中层主管，中层再拆给执行 Agent，结果逐层汇总。

## 6.1 机制

```
        ┌──────────────┐
        │  顶层主管     │
        └──┬───────┬───┘
           ▼       ▼
     ┌─────────┐ ┌─────────┐
     │ 中层主管 │ │ 中层主管 │
     └──┬───┬──┘ └──┬───┬──┘
        ▼   ▼       ▼   ▼
      Agent Agent Agent Agent
```

| 层级 | 职责 | 关注粒度 |
|------|------|----------|
| 顶层 | 任务分解与总汇总 | 目标 |
| 中层 | 子任务分解与子汇总 | 子目标 |
| 执行层 | 具体执行 | 动作 |

**边界与坑**：每多一层，信息就多一次有损汇总，误差逐层放大。

## 6.2 实现

```java
public class HierarchicalOrchestrator {

    private final Supervisor topSupervisor;
    private final Map<String, SupervisorOrchestrator> middleLayers;

    public String run(String task) {
        // 顶层拆解为子任务
        List<String> subtasks = topSupervisor.decompose(task);
        List<String> partial = new ArrayList<>();

        for (String subtask : subtasks) {
            SupervisorOrchestrator middle = middleLayers.get(route(subtask));
            partial.add(middle.supervise(subtask));
        }
        // 顶层汇总
        return topSupervisor.synthesize(task, partial);
    }

    private String route(String subtask) {
        return topSupervisor.route(subtask);
    }
}
```

## 6.3 边界与坑

| 坑 | 现象 | 对策 |
|----|------|------|
| 汇总失真 | 多层摘要丢失细节 | 关键结论原样上抛 |
| 成本爆炸 | 每层都调用强模型 | 下层用便宜模型 |
| 协调开销 | 层数越多越慢 | 控制在 2–3 层 |
| 责任不清 | 出问题难定位层级 | 每层独立 Trace |

**生产建议**：分层仅用于"确实存在组织层级"的任务；否则扁平化更高效。

# 7. 辩论

**结论先行**：辩论通过"提出—批评—修正"的多轮循环提升结论质量，适合需要多视角、可证伪判断的任务。

**原理**：提案者给出方案，批评者指出问题，提案者据此修正；若干轮后由裁判裁决。

## 7.1 机制

```
Agent A 提方案 → Agent B 批评 → Agent A 修正 → ... → 结论
```

| 角色 | 职责 |
|------|------|
| 提案者 | 提出并修正方案 |
| 批评者 | 指出漏洞与反例 |
| 裁判 | 评估并给出最终结论 |

**边界与坑**：轮次过多会导致"为改而改"，甚至越改越差；必须设轮次上限与收敛判据。

## 7.2 实现

```java
public String debate(String question, int rounds) {
    String proposal = proposer.handle(question);
    String previous = null;

    for (int i = 0; i < rounds; i++) {
        String critique = critic.handle(proposal);
        String revised = proposer.revise(proposal, critique);
        if (revised.equals(previous)) {
            break;                     // 收敛：与上一轮相同则提前结束
        }
        previous = proposal;
        proposal = revised;
    }
    return judge.evaluate(question, proposal);
}
```

## 7.3 边界与坑

| 坑 | 现象 | 对策 |
|----|------|------|
| 不收敛 | 反复修改无终点 | 轮次上限 + 收敛判据 |
| 成本高 | 每轮多次强模型调用 | 轮次 2–3 足够 |
| 同质化 | 提案与批评来自同一模型 | 用不同模型/提示词 |
| 裁判偏见 | 裁判偏向某一方 | 裁判独立、给明确评分维度 |

**生产建议**：辩论只用于"高风险决策"或"需要对抗性检验"的场景；日常任务用单 Agent 即可。

# 8. 模式对比与选择

**结论先行**：先按"流程是否可预定义"分静态/动态，再按"是否需中心调度"细分。

## 8.1 横向对比

| 模式 | 决策者 | 可预测性 | 灵活性 | 成本 | 调试难度 |
|------|--------|----------|--------|------|----------|
| Supervisor | 主管 | 中 | 中 | 中 | 低 |
| Swarm | 各 Agent | 低 | 高 | 中高 | 高 |
| 流水线 | 预定义 | 高 | 低 | 低 | 低 |
| 并行 | 预定义 | 高 | 低 | 中 | 低 |
| 分层 | 多层主管 | 中 | 中 | 高 | 中 |
| 辩论 | 规则 | 中 | 中 | 高 | 中 |

## 8.2 组合使用

**结论先行**：真实系统很少只用一种模式，常见组合如下。

| 场景 | 推荐 |
|------|------|
| 任务可分解、需协调 | Supervisor |
| 开放任务、自主协作 | Swarm |
| 固定顺序 | 流水线 |
| 独立子任务 | 并行 |
| 大型组织 | 分层 |
| 需要多视角 | 辩论 |

**生产建议**：以 Supervisor 为主干，局部嵌套流水线/并行；辩论作为"高风险节点的复核器"。

# 9. 生产注意事项

**结论先行**：编排失控的代价是"无限循环 + 成本爆炸 + 无法调试"，三者必须同时治理。

## 9.1 风险与对策

| 风险 | 建议 |
|------|------|
| 无限循环 | 步数上限 |
| 成本失控 | 限制 Agent 数 |
| 难调试 | 统一 Trace |
| 主管瓶颈 | 主管用强模型 |
| 责任扩散 | 明确职责边界 |
| 上下文膨胀 | 传摘要而非全量 |
| 部分失败 | 允许降级与重试 |

## 9.2 上线检查清单

- 所有模式都有全局步数上限。
- 每一步都有 Trace 与 correlationId。
- 每个 Agent 职责边界清晰且唯一。
- 主管决策输出有白名单校验。
- 并行任务支持部分成功。
- 成本有预算与熔断。
- 终止条件可判定且可测试。

**边界与坑**：终止条件最容易写成"模型说完成就完成"，一旦模型幻觉，循环就停不下来。

**生产建议**：终止条件应是"确定性规则 + 模型判断"的组合，规则优先。

# 10. 小结

- 六种编排模式：Supervisor、Swarm、流水线、并行、分层、辩论。
- Supervisor 最常用，流程清晰可控，适合多数场景。
- Swarm 灵活但难追踪，必须设全局步数上限。
- 流水线与并行是静态编排，可预测、易测试。
- 分层适合大型组织，但层数应控制在 2–3 层。
- 辩论适合高风险决策，轮次 2–3 足够。
- 真实系统多为组合模式，以 Supervisor 为主干。
- 必须设步数上限、统一 Trace、明确职责边界。
