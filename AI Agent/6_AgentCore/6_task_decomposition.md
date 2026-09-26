# 1. 任务分解是什么

## 1.1 定义

任务分解（Task Decomposition）是把一个复杂目标拆成**若干可独立执行、可验证的子任务**的过程。它是规划的基础。

```
"帮我准备一份季度经营分析报告"
        │
        ▼  分解
┌─────────────────────────────────────────┐
│ 1. 取数：销售、成本、库存                  │
│ 2. 计算：同比/环比、毛利率                 │
│ 3. 分析：异常点归因                        │
│ 4. 撰写：结论与建议                        │
│ 5. 排版：输出报告                          │
└─────────────────────────────────────────┘
```

从工程角度看，分解的本质是**把一次"大而模糊"的模型调用，转化为多次"小而确定"的调用或工具动作**。每一次调用都有清晰的输入、输出与判定标准，从而使整条链路可测试、可重试、可观测。

## 1.2 为什么重要

- **降低单步复杂度**：每步只需模型做一件简单的事，成功率更高；
- **便于并行**：无依赖的子任务可并发，缩短总耗时；
- **便于校验**：每个子任务有明确完成标准，可逐项验收；
- **便于调度**：子任务可映射到不同工具/Agent。

再补充两点常被忽略的价值：

- **便于成本核算**：粒度清晰的子任务可以分别统计 Token 与耗时，定位昂贵环节；
- **便于降级**：某个子任务失败时，可只降级该子任务，而不必放弃整个目标。

## 1.3 与规划的关系

```
任务分解：把目标拆成"做什么"（子任务集合）
规划    ：把子任务排成"按什么顺序做"（执行序列）
```

分解是"横向切开"，规划是"纵向排序"。二者常在同一轮 LLM 调用中一起产出。

需要强调的是：分解结果是一个**集合**（无序），规划结果是一个**序列或偏序**（有序）。集合去重、排序、并行层级划分都发生在规划阶段，因此分解本身不关心先后顺序。

## 1.4 分解的质量度量

判断一次分解好不好，可以用几个可量化指标：

| 指标 | 含义 | 期望 |
|------|------|------|
| 子任务数 | 分解后的步骤数量 | 3~10 个，随复杂度线性增长 |
| 平均粒度 | 单步对应的工具/动作数 | 接近 1 |
| 依赖密度 | 边数 / 节点数 | 越低越易并行 |
| 可执行率 | 能映射到工具的子任务占比 | 接近 100% |
| 重复率 | 重复 action+参数 的占比 | 接近 0 |
| 最大深度 | 递归层数 | 2~3 |

> 度量不是为了追求完美，而是为了在"拆得太粗"和"拆得太细"之间找到平衡点。

# 2. 分解的四个原则

## 2.1 原子性

每个子任务只做一件事。反例："查询并分析销售数据"——查询与分析应拆开。

判定方法：如果一个子任务需要**两种不同能力**（如检索 + 计算）或**两个不同工具**，通常说明它还不够原子。

## 2.2 可验证

有明确的完成标准（成功/失败/产出物）。反例："优化一下体验"——无法判定完成。

可验证的三种常见形态：

- **断言型**：结果满足某条件（如 `status == 200`）；
- **产物型**：产出了某文件/记录；
- **评估型**：由评估器（LLM 或规则）打分超过阈值。

## 2.3 低耦合

子任务之间依赖尽量少，减少串行等待与级联失败。

低耦合不等于零依赖。数据流依赖（B 需要 A 的输出）通常是**必要**的；需要消除的是**控制流耦合**（B 需要知道 A 的内部状态、重试次数等）。

## 2.4 可执行

子任务能映射到具体工具/动作。反例："理解用户意图"——若无对应能力则无法执行。

> 关键实践：分解时把**可用工具清单**一并提供给模型，让它在能力边界内拆分，而不是天马行空地拆。

## 2.5 原则的冲突与取舍

| 冲突 | 取舍 |
|------|------|
| 原子 vs 步数 | 拆太细导致步数爆炸 → 合并同类 |
| 低耦合 vs 数据依赖 | 强依赖时保留串行，不强行并行 |
| 可验证 vs 开放性 | 开放任务用"产出物存在"作为弱验证 |

实际工程中，四条原则的优先级通常为：**可执行 > 可验证 > 原子性 > 低耦合**。因为一个不可执行的子任务无论多"原子"都没有意义。

# 3. 常见分解策略

## 3.1 按阶段分解

```
收集信息 → 处理信息 → 分析 → 输出
```

适合：研究、报告、数据管道。

### 3.1.1 阶段划分的粒度

阶段不宜过多（3~5 个），否则调度开销与协调成本上升；每个阶段内部再按需细化。

### 3.1.2 阶段分解的 Java 实现

```java
public record Stage(String name, List<String> steps) {}

public List<Stage> decomposeByStage(String goal) {
    return List.of(
            new Stage("收集", List.of("确定数据源", "拉取原始数据")),
            new Stage("处理", List.of("清洗", "对齐口径")),
            new Stage("分析", List.of("计算指标", "归因异常")),
            new Stage("输出", List.of("撰写结论", "生成报告"))
    );
}
```

### 3.1.3 阶段分解的边界

阶段分解要求任务本身**天然可分阶段**。对于"一问一答"的简单任务强行分阶段，只会增加开销。反例：把"翻译一句话"拆成"理解 → 翻译 → 润色"，其中"理解"与"润色"都不产生独立价值。

## 3.2 按数据分片分解

```
把大任务按数据分片：
  处理 1 月数据
  处理 2 月数据
  ...
  合并结果
```

适合：可并行的大批量处理。分片之间必须**相互独立**；若合并时需要跨分片排序或去重，需在 Reduce 阶段显式处理。

## 3.3 按能力分解

```
查询类任务 → 交给检索能力
计算类任务 → 交给代码执行能力
写作类任务 → 交给生成能力
```

适合：能力边界清晰的系统。

### 3.3.1 能力路由

按能力分解的关键是"能力路由"：根据子任务类型选择执行器。

```java
public String route(String subTask) {
    if (looksLikeQuery(subTask))  return retriever.handle(subTask);
    if (looksLikeCompute(subTask)) return codeRunner.handle(subTask);
    return generator.handle(subTask);
}
```

> 边界：当子任务跨能力（既要查又要算）时，应**先拆再路由**，不要在一个执行器里混合多种能力。

## 3.4 递归分解

### 3.4.1 实现

```java
public List<String> decompose(String task, int depth) {
    if (depth == 0 || isAtomic(task)) {
        return List.of(task);
    }
    List<String> subTasks = llmDecompose(task);
    return subTasks.stream()
            .flatMap(t -> decompose(t, depth - 1).stream())
            .toList();
}
```

### 3.4.2 深度控制

> 递归必须有深度上限，否则会无限拆解。

- 设 `maxDepth`（建议 2~3）；
- 设"原子任务"判定（能直接映射到工具即原子）；
- 每层记录来源，便于回溯。

### 3.4.3 递归分解的边界

递归分解的常见陷阱是"**越拆越碎**"：每层都觉得自己不够原子，最终产生几十个无意义的微步骤。防御手段：

- 深度上限 + 子任务数上限双约束；
- 相邻两层若子任务集合高度相似，判定为"无进展"，立即停止；
- 记录父任务，一旦发现语义重复则合并。

# 4. 依赖识别与拓扑排序

## 4.1 依赖图

子任务之间常有依赖，需排出执行顺序：

```
     取数
    ┌──┴──┐
    ▼     ▼
  计算   计算       ← 可并行
  销售   成本
    └──┬──┘
       ▼
     归因分析
       ▼
     撰写报告
```

依赖图的三个要素：

- **节点**：一个子任务；
- **有向边** `A → B`：表示 B 依赖 A（A 必须先完成）；
- **入度**：指向节点的边数，入度为 0 的节点是当前可执行节点。

## 4.2 拓扑排序

```java
public record SubTask(String id, String action, Set<String> dependsOn) {}

public List<SubTask> topoSort(List<SubTask> tasks) {
    // Kahn 算法：按依赖关系排序，识别可并行层级
}
```

拓扑排序把子任务排成"层级"：同一层内无依赖，可并行。

### 4.2.1 Kahn 算法

```java
public List<SubTask> topoSort(List<SubTask> tasks) {
    Map<String, Integer> indegree = new HashMap<>();
    Map<String, List<String>> adj = new HashMap<>();
    // 构建入度与邻接表...
    Queue<String> q = new LinkedList<>();
    indegree.forEach((id, d) -> { if (d == 0) q.add(id); });

    List<SubTask> sorted = new ArrayList<>();
    while (!q.isEmpty()) {
        String id = q.poll();
        sorted.add(byId(tasks, id));
        for (String next : adj.getOrDefault(id, List.of())) {
            if (indegree.merge(next, -1, Integer::sum) == 0) q.add(next);
        }
    }
    if (sorted.size() != tasks.size()) throw new InvalidPlanException("存在环");
    return sorted;
}
```

## 4.3 环检测

若依赖图存在环，则无法执行。需在调度前检测：

```java
if (hasCycle(tasks)) {
    throw new InvalidPlanException("计划存在循环依赖");
}
```

> 环检测是"计划可执行性"的第一道防线。

### 4.3.1 环的处理策略

检测到环后，不应直接崩溃，而应尝试修复：

| 策略 | 说明 | 适用 |
|------|------|------|
| 报错终止 | 直接拒绝执行 | 强正确性要求 |
| 请求模型重规划 | 把环反馈给 LLM 重新分解 | 自动 Agent |
| 删除弱依赖边 | 断开最"可疑"的一条边 | 依赖可降级 |
| 人工介入 | 转人工审核 | 高风险任务 |

> 修复后必须**重新做一次环检测**，直到无环或达到重试上限。

# 5. 并行执行

## 5.1 识别可并行层

拓扑排序后，同一层的子任务彼此无依赖，可并发执行。

```
第 0 层: 取数
第 1 层: 计算销售  计算成本
第 2 层: 归因分析
第 3 层: 撰写报告
```

## 5.2 并行实现

```java
List<CompletableFuture<Result>> futures = independentTasks.stream()
        .map(t -> CompletableFuture.supplyAsync(() -> executor.execute(t), pool))
        .toList();

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();

List<Result> results = futures.stream().map(CompletableFuture::join).toList();
```

### 5.2.1 失败处理

并行中任一子任务失败，应**部分成功可继续**：收集成功结果，失败的进入重试或标记降级，而非整体回滚（除非有强一致性要求）。

```java
public List<Outcome> runParallel(List<SubTask> tasks) {
    return tasks.parallelStream().map(t -> {
        try {
            return Outcome.ok(t.id(), executor.execute(t));
        } catch (Exception e) {
            return Outcome.failed(t.id(), e.getMessage());
        }
    }).toList();
}
```

## 5.3 并发控制

> 并行能显著缩短长任务耗时，但要注意下游服务的并发承载能力。

- 设置并发上限（信号量/线程池大小）；
- 对下游限流，避免压垮；
- 单任务超时 + 整体超时。

```java
Semaphore gate = new Semaphore(8);     // 最多 8 个并发

public Result guardedExecute(SubTask t) {
    gate.acquireUninterruptibly();
    try {
        return executor.execute(t);
    } finally {
        gate.release();
    }
}
```

> 边界：并行度并非越高越好。当子任务共享同一 LLM 配额或下游 API 时，过高并行会触发限流，反而降低吞吐。

# 6. 分解的输出格式

## 6.1 结构化记录

```java
public record SubTask(
        String id,
        String action,
        String tool,
        List<String> dependsOn,
        String expectedOutput) {}

interface Decomposer {
    @UserMessage("""
            把任务拆解为子任务，输出 JSON：
            任务：{{it}}

            要求：
            - 每个子任务可独立执行
            - 标明依赖（dependsOn 填子任务 id）
            - 标明期望输出
            """)
    List<SubTask> decompose(String task);
}
```

## 6.2 校验

结构化输出让分解结果可被程序校验：

- 依赖 id 是否存在；
- 是否存在环；
- 每个子任务是否有对应工具。

### 6.2.1 校验清单

| 校验项 | 规则 |
|--------|------|
| 依赖存在 | dependsOn 的 id 必须在集合中 |
| 无环 | 拓扑排序能覆盖全部节点 |
| 可执行 | 每个 tool 在可用工具清单内 |
| 无重复 | action+参数 唯一 |

### 6.2.2 校验的 Java 实现

```java
public void validate(List<SubTask> tasks, Set<String> availableTools) {
    Set<String> ids = tasks.stream().map(SubTask::id).collect(Collectors.toSet());

    for (SubTask t : tasks) {
        if (!availableTools.contains(t.tool())) {
            throw new InvalidPlanException("未知工具: " + t.tool());
        }
        for (String dep : t.dependsOn()) {
            if (!ids.contains(dep)) {
                throw new InvalidPlanException("依赖不存在: " + dep);
            }
        }
    }
    topoSort(tasks);        // 顺带做环检测
}
```

> 校验失败时，把错误信息回灌给模型重新生成，通常 1~2 轮即可得到合法计划。

# 7. 分解的常见问题

## 7.1 问题与对策

| 问题 | 表现 | 对策 |
|------|------|------|
| 粒度过细 | 步骤过多、开销大 | 限制步数、合并同类 |
| 粒度过粗 | 单步仍无法执行 | 递归细化 |
| 依赖遗漏 | 执行顺序错误 | 显式声明 dependsOn + 校验 |
| 重复子任务 | 重复劳动 | 去重（按 action+参数） |
| 不可执行 | 子任务无法映射到工具 | 分解时提供工具清单 |
| 计划有环 | 无法执行 | 拓扑排序前环检测 |

## 7.2 反例分析

**反例一：粒度过粗**

```
目标：分析上月销售并给出建议
分解：["分析销售", "给建议"]      ← "分析销售" 仍无法直接执行
```

修正：把"分析销售"继续拆成"取数 → 计算指标 → 归因"。

**反例二：依赖遗漏**

```
分解：["取数", "计算毛利率", "取成本"]   ← 计算毛利率漏依赖取成本
```

修正：`计算毛利率.dependsOn = [取数, 取成本]`。

**反例三：伪并行**

```
分解：["查询用户", "查询订单"]，但两者都依赖"登录"
```

若"登录"未显式建模，调度器会把二者并行执行，导致鉴权失败。修正：补一个"登录"节点作为共同前置。

# 8. 分解与 Agent 架构

## 8.1 架构映射

| 架构 | 分解方式 |
|------|----------|
| 单 Agent | 内部隐式分解（ReAct 逐步） |
| Planner-Executor | 显式分解为计划 |
| 多 Agent | 分解为子任务并分配给不同 Agent |

> 详见 `12_agent_architectures.md` 与 `10_MultiAgent`。

## 8.2 Planner-Executor 示例

Planner 负责分解与排序，Executor 负责逐步执行，二者解耦：

```java
public class PlannerExecutorAgent {
    private final Decomposer planner;
    private final Executor executor;

    public String run(String goal) {
        List<SubTask> plan = planner.decompose(goal);
        validate(plan, executor.availableTools());

        Map<String, String> results = new HashMap<>();
        for (SubTask t : topoSort(plan)) {
            results.put(t.id(), executor.execute(t, results));
        }
        return results.get(plan.get(plan.size() - 1).id());
    }
}
```

> 边界：Planner-Executor 的计划是"一次成型"的，执行中遇到意外不会自动重规划。需要应对动态环境时，应在每步执行后加"是否需重规划"的判断。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 无限递归分解 | 深度上限 |
| 分解结果不可执行 | 提供工具清单与约束 |
| 并行压垮下游 | 并发上限 + 限流 |
| 无进度反馈 | 分阶段输出进度 |
| 子任务失败无处理 | 失败重试 + 跳过 + 补偿 |
| 计划有环 | 执行前环检测 |

## 9.2 可观测性

分解与执行过程必须可追踪，否则线上问题无从定位：

| 观测项 | 内容 |
|--------|------|
| 分解结果 | 原始计划、每轮修订 |
| 执行轨迹 | 每个子任务的输入、输出、耗时 |
| 依赖状态 | 已完成/进行中/失败节点 |
| 并行指标 | 并发度、排队时间、限流次数 |
| 失败原因 | 异常类型、重试次数、最终处置 |

> 建议把计划与执行轨迹都落库，便于回放与质量分析。

# 10. 小结

## 10.1 关键要点

- 分解原则：原子、可验证、低耦合、可执行。
- 四种策略：按阶段、按数据、按能力、递归。
- 依赖用拓扑排序，无依赖的子任务并行执行。
- 分解结果要结构化，便于校验与调度。
- 粒度和深度都要有上限，且执行前要做环检测。

## 10.2 检查清单

- [ ] 分解时是否提供了可用工具清单。
- [ ] 是否设置了递归深度与子任务数上限。
- [ ] 执行前是否做了环检测。
- [ ] 是否配置了并发上限与超时。
- [ ] 是否对子任务失败做了重试/降级/补偿。
