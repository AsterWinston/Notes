# 1. 实现总览

结论先行：多 Agent 系统的实现核心是"主管决策 + 子 Agent 执行 + 共享状态 + 终止控制"四件事，其余都是围绕这四件事的加固。边界不清则表现为"能跑但一上量就死循环、成本失控、无法调试"。

## 1.1 目标与范围

本篇聚焦可运行的最小实现（Minimum Viable Implementation，MVI）：

- 用 Supervisor（主管）模式协调多个专业 Agent（智能体）完成复合任务。
- 每步都有可观测的协作 Trace（轨迹），可回放"谁、在什么指令下、产出了什么"。
- 内置终止控制（步数、成本、循环、停滞），保证系统一定会停，并通过 SSE（Server-Sent Events）暴露进度。

范围之外：评测与灰度（见 4_evaluation.md）、工具权限与沙箱、检索后端选型。

## 1.2 架构分层

### 1.2.1 组件职责

| 组件 | 职责 | 关键约束 |
|------|------|----------|
| Supervisor | 决策委托谁、何时结束 | 强模型、结构化输出 |
| Agent 注册表 | 按名字解析子 Agent | 名字唯一、描述清晰 |
| Orchestrator | 驱动主循环、组装结果 | 无业务逻辑，只做编排 |
| SharedState | 跨 Agent 共享数据 | 可序列化、键命名空间化 |
| LoopDetector | 检测重复委托 | 签名归一化 |
| TraceStore | 持久化每轮记录 | 异步写入、可查询 |
| TerminationPolicy | 判定是否停止 | 组合多种策略 |

### 1.2.2 请求流转

```text
Client -> Orchestrator
              |
        loop: decide -> dispatch -> record
              |            |          |
              v            v          v
        Supervisor      Agents   SharedState
              \            |          /
               +--> TraceStore --> EventBus -> SSE
```

流程：调用 Supervisor 得到决策（委托或结束）；若委托则解析 Agent 并执行、产出写入 SharedState；每轮写 Trace 并发事件；满足终止条件则合成（synthesize）或返回终止结果。

## 1.3 数据模型

### 1.3.1 核心类型

用 Java `record`（记录类型，不可变数据载体）表达数据。

```java
public record AgentRequest(String instruction, Map<String, Object> meta) {
    public AgentRequest(String instruction) { this(instruction, Map.of()); }
}

public record AgentResult(boolean success, String output, String error) {
    public static AgentResult ok(String output) { return new AgentResult(true, output, null); }
    public static AgentResult fail(String error) { return new AgentResult(false, null, error); }
}

public record AgentTurn(String agent, String instruction, String output,
                        boolean success, long durationMs, Instant timestamp) {
    public static AgentTurn of(String agent, String instruction, String output) {
        return new AgentTurn(agent, instruction, output, true,
                System.currentTimeMillis(), Instant.now());
    }
    public static AgentTurn error(String message) {
        return new AgentTurn("_system", message, null, false, 0, Instant.now());
    }
}

public record SupervisorDecision(String action, String instruction, String reason) {
    public boolean isFinish() { return "FINISH".equalsIgnoreCase(action); }
}
```

### 1.3.2 共享状态

SharedState 是共享白板。值必须可序列化，键用 `agentName.key` 前缀避免覆盖。

```java
public class SharedState {
    private final Map<String, Object> data = new ConcurrentHashMap<>();
    public void put(String key, Object value) { data.put(key, value); }
    @SuppressWarnings("unchecked")
    public <T> T get(String key, Class<T> type) { return (T) data.get(key); }
    public Map<String, Object> snapshot() { return Map.copyOf(data); }
}
```

# 2. Supervisor 实现

结论先行：Supervisor 的产出质量直接决定系统质量。重点是"强模型 + 明确规则 + 结构化输出 + 规则兜底"。

## 2.1 决策接口

### 2.1.1 决策提示词

提示词约束三件事：一次只委托一个、指令可执行、信息足够时结束。

```java
@Service
@RequiredArgsConstructor
public class SupervisorAgent {

    private final ChatClient chatClient;

    public SupervisorDecision decide(String agents, String task, String history) {
        return chatClient.prompt()
                .system("""
                        你是任务主管，协调多个专业 Agent 完成用户任务。
                        规则：
                        1. 每次只委托一个 Agent；
                        2. 委托时给出明确、可执行的指令；
                        3. 信息足够时输出 FINISH；
                        4. 不要重复委托相同工作；
                        5. 不要臆造不存在的 Agent 名称。
                        输出 JSON：
                        {"action":"agentName|FINISH","instruction":"","reason":""}
                        """)
                .user("""
                        可用 Agent：
                        %s

                        用户任务：%s

                        已完成：
                        %s
                        """.formatted(agents, task, history))
                .call()
                .entity(SupervisorDecision.class);
    }

    public String synthesize(String task, String history) {
        return chatClient.prompt()
                .system("整合各 Agent 的工作成果，输出最终答案。")
                .user("任务：%s\n\n工作记录：\n%s".formatted(task, history))
                .call().content();
    }
}
```

### 2.1.2 结构化输出

`entity(...)` 依赖模型 JSON 能力。生产中做两层校验：语法层（能否解析）与语义层（action 是否在注册表内）。做法是包一层 `decideSafe`：最多重试一次，若仍解析失败或 action 不在 `knownAgents` 中，则返回 `new SupervisorDecision("FINISH", "", "fallback")` 强制收敛。

| action 取值 | 含义 | 后续动作 |
|-------------|------|----------|
| 某个 Agent 名 | 委托该 Agent | dispatch |
| FINISH | 信息足够 | synthesize |
| 空/未知 | 非法决策 | 记错误轮，兜底 |

## 2.2 兜底与重试

- 解析失败：重试一次，仍失败则强制 FINISH。
- 未知 Agent：记录错误轮，让主管重新决策。
- 连续 N 次错误：降级为单 Agent 直接回答。

## 2.3 边界与坑

- 坑一：把"委托谁"和"做什么"混在一起，子 Agent 无法独立执行。指令必须自包含。
- 坑二：历史过长导致主管分心。传入的应是摘要而非全文。
- 反例：history 拼接每个 Agent 的完整输出，10 轮后提示词膨胀到几万 token。
- 生产建议：history 做滚动窗口 + 摘要压缩，保留最近 3 轮全文，更早的只留一句结论。

# 3. 子 Agent 实现

结论先行：子 Agent 应是"薄执行器"——输入明确指令，输出结果并写共享状态，不参与编排决策。

## 3.1 接口契约

```java
public interface Agent {
    String name();
    String description();
    AgentResult handle(AgentRequest req, SharedState state);
}
```

请求只含一条指令与可选元数据；结果要么成功带输出，要么失败带错误。失败不抛异常穿透，而转成可记录的失败轮。

## 3.2 检索 Agent

```java
@Component
@RequiredArgsConstructor
public class RetrievalAgent implements Agent {

    private final ContentRetriever retriever;

    @Override
    public String name() { return "retriever"; }

    @Override
    public String description() { return "检索企业知识库与联网信息"; }

    @Override
    public AgentResult handle(AgentRequest req, SharedState state) {
        try {
            List<Document> docs = retriever.retrieve(req.instruction());
            state.put("retriever.documents", docs);
            String summary = docs.stream()
                    .map(Document::getText)
                    .collect(Collectors.joining("\n---\n"));
            return AgentResult.ok(truncate(summary, 3000));
        } catch (Exception e) {
            return AgentResult.fail("检索失败：" + e.getMessage());
        }
    }
}
```

## 3.3 分析 Agent

```java
@Component
@RequiredArgsConstructor
public class AnalysisAgent implements Agent {

    private final ChatClient chatClient;

    @Override
    public String name() { return "analyzer"; }

    @Override
    public String description() { return "分析数据并给出洞察"; }

    @Override
    public AgentResult handle(AgentRequest req, SharedState state) {
        String analysis = chatClient.prompt()
                .system("基于提供的数据给出分析，输出关键洞察，不要编造数据。")
                .user(req.instruction())
                .call().content();
        state.put("analyzer.analysis", analysis);
        return AgentResult.ok(analysis);
    }
}
```

## 3.4 工具型 Agent

工具型 Agent 封装副作用操作（写文件、调 API），必须带权限校验与幂等键。

```java
@Component
@RequiredArgsConstructor
public class WriterAgent implements Agent {

    private final PermissionChecker permissionChecker;

    @Override
    public String name() { return "writer"; }

    @Override
    public String description() { return "将结果写入指定文件或系统"; }

    @Override
    public AgentResult handle(AgentRequest req, SharedState state) {
        if (!permissionChecker.allowed("writer", req.meta())) {
            return AgentResult.fail("无写权限");
        }
        String key = String.valueOf(req.meta().get("idempotencyKey"));
        return AgentResult.ok(write(req.instruction(), key));
    }
}
```

## 3.5 边界与坑

| 坑 | 表现 | 应对 |
|----|------|------|
| Agent 越权决策 | 自行调用其他 Agent | 只暴露 handle |
| 输出过长 | 下轮提示词爆炸 | 统一 truncate + 摘要 |
| 异常穿透 | 整个任务崩溃 | handle 内捕获转 fail |
| 副作用重复 | 重试导致重复写入 | 幂等键 |
| 描述含糊 | 主管选错 Agent | 描述含"何时用我" |

# 4. 编排器

结论先行：编排器是"无业务逻辑的循环泵"，只负责决策、委托、记录、终止四步。

## 4.1 主循环

### 4.1.1 决策阶段

每轮先决策，输入是可用 Agent 列表、原任务、已完成历史。

### 4.1.2 委托阶段

解析 Agent、执行、结果追加进历史。未知 Agent 记为错误轮，不中断循环。

### 4.1.3 收敛阶段

决策为 FINISH 时调用 synthesize 汇总所有轮次产出最终答案。

```java
@Service
@RequiredArgsConstructor
public class MultiAgentOrchestrator {

    private static final int MAX_STEPS = 12;

    private final SupervisorAgent supervisor;
    private final Map<String, Agent> agents;      // Spring 注入
    private final TraceStore traceStore;
    private final CostTracker costTracker;
    private final TerminationPolicy terminationPolicy;

    public MultiAgentResult run(String taskId, String task) {
        SharedState state = new SharedState();
        List<AgentTurn> turns = new ArrayList<>();

        for (int step = 0; step < MAX_STEPS; step++) {
            SupervisorDecision decision = supervisor.decide(
                    describeAgents(), task, format(turns));
            if (decision.isFinish()) {
                String result = supervisor.synthesize(task, format(turns));
                return MultiAgentResult.success(result, turns.size());
            }
            if (terminationPolicy.shouldStop(step,
                    costTracker.costOf(taskId), turns)) {
                return MultiAgentResult.terminated(turns.size());
            }
            Agent agent = agents.get(decision.action());
            if (agent == null) {
                turns.add(AgentTurn.error("未知 Agent: " + decision.action()));
                continue;
            }
            long start = System.currentTimeMillis();
            AgentResult result = agent.handle(
                    new AgentRequest(decision.instruction()), state);
            AgentTurn turn = result.success()
                    ? new AgentTurn(decision.action(), decision.instruction(),
                            result.output(), true,
                            System.currentTimeMillis() - start, Instant.now())
                    : AgentTurn.error(result.error());
            turns.add(turn);
            traceStore.recordTurn(taskId, turn);
        }
        return MultiAgentResult.maxSteps(turns.size());
    }
}
```

其中 `describeAgents()` 把注册表中每个 Agent 的 `name` 与 `description` 拼成清单，作为决策提示词的输入。

## 4.2 共享状态与交接

结论先行：交接内容应是"摘要 + 指针"，而非全文。全文放 SharedState，交接只传键名与摘要。

| 交接方式 | 上下文成本 | 信息损失 | 适用 |
|----------|-----------|----------|------|
| 传全文 | 高 | 无 | 轮数很少 |
| 传摘要 | 低 | 有 | 默认 |
| 传键名 | 极低 | 需回查 | 大数据对象 |

## 4.3 终止策略

终止策略是多个条件的并集，任一命中即停。

```java
public class TerminationPolicy {

    private final int maxSteps;
    private final BigDecimal costBudget;
    private final LoopDetector loopDetector;
    private final int loopThreshold;

    public boolean shouldStop(int step, BigDecimal cost, List<AgentTurn> turns) {
        if (step >= maxSteps) return true;
        if (cost.compareTo(costBudget) >= 0) return true;
        if (loopDetector.hasLoop(turns, loopThreshold)) return true;
        return isStagnant(turns);
    }

    private boolean isStagnant(List<AgentTurn> turns) {
        int n = turns.size();
        if (n < 3) return false;
        return similar(turns.get(n - 1).output(), turns.get(n - 2).output())
                && similar(turns.get(n - 2).output(), turns.get(n - 3).output());
    }
}
```

## 4.4 边界与坑

- 坑：MAX_STEPS 设得过大，实际等于没有上限。
- 坑：成本统计在异步场景滞后，可能超预算才停。
- 反例：终止后直接丢弃中间结果，用户什么都拿不到。
- 生产建议：终止时也调一次 synthesize，输出"尽力而为"的答案并标注未完成。

# 5. 循环检测

结论先行：死循环是多 Agent 的头号故障。检测分三层：完全重复、签名重复、语义重复。

## 5.1 签名归一化

把"Agent 名 + 归一化指令"作为签名。归一化即去空白、转小写、去中英文标点，使 `"查资料"` 与 `"查 资料。"` 归为同一签名。

## 5.2 阈值与语义去重

```java
public class LoopDetector {

    public boolean hasLoop(List<AgentTurn> turns, int threshold) {
        Map<String, Integer> counts = new HashMap<>();
        for (AgentTurn turn : turns) {
            String sig = turn.agent() + ":" + normalize(turn.instruction());
            counts.merge(sig, 1, Integer::sum);
            if (counts.get(sig) >= threshold) return true;
        }
        return false;
    }
}
```

签名相同只是表面重复；更隐蔽的是"换句话问同一件事"，可用嵌入（Embedding）相似度检测：对相邻两轮指令求余弦相似度，超过阈值即判为语义循环。阈值经验值：`threshold = 3` 适合多数任务；允许反复检索的研究型任务可放宽到 4。

## 5.3 边界与坑

| 坑 | 说明 | 应对 |
|----|------|------|
| 误判正常重试 | 合理再检索被当循环 | 阈值放宽 + 白名单 |
| 归一化过度 | 不同任务归一成同一签名 | 保留关键实体 |
| 只查相邻轮 | 长距离重复漏检 | 全局计数 |
| 语义检测成本 | 每轮都算嵌入 | 采样检测 |

# 6. 协作 Trace

结论先行：Trace 是调试与信任的基础。没有 Trace 就无法定位"哪一步错了"。

## 6.1 数据结构与渲染

```java
public interface TraceStore {
    void recordTurn(String taskId, AgentTurn turn);
    List<AgentTurn> findByTask(String taskId);
}
```

`renderTrace` 按轮次编号输出 `[agent] instruction` 与截断后的输出，供前端展示与人工回放。建议用追加日志（append-only）存储，便于回放与审计。

## 6.2 可观测性接入

| 维度 | 埋点 |
|------|------|
| 每轮 | agent、耗时、token、成功与否 |
| 每任务 | 总步数、总成本、总耗时、终止原因 |
| 系统 | 死循环率、平均步数、P99 耗时 |

建议接入 OpenTelemetry，每个 AgentTurn 作为一个 span，任务作为 trace。

## 6.3 边界与坑

- 坑：Trace 同步写入阻塞主循环。应异步写入或缓冲批量落盘。
- 坑：Trace 记录敏感内容。应脱敏或加密。
- 反例：只记 Agent 名不记指令，无法复现"为什么这么决策"。

# 7. 流式进度

结论先行：长任务必须有进度反馈，否则用户会以为卡死。SSE 是成本最低的方案。

## 7.1 SSE 接口

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> stream(@RequestParam String task) {
    return orchestrator.runStreaming(task)
            .map(event -> ServerSentEvent.<String>builder()
                    .event(event.type())
                    .data(event.data())
                    .build());
}
```

## 7.2 事件模型

| 事件类型 | 时机 | 数据 |
|----------|------|------|
| started | 任务开始 | taskId |
| turn | 每轮结束 | agent + 摘要 |
| done | 任务结束 | taskId |
| error | 异常 | 错误信息 |

事件用不可变记录 `ProgressEvent(String type, String data)` 承载，由 EventBus 推送。

## 7.3 边界与坑

- 坑：代理（Nginx）默认缓冲 SSE，事件不实时。需关闭 `proxy_buffering`。
- 坑：连接超时导致长任务断流。需心跳事件。
- 生产建议：SSE 只做进度，最终结果走单独查询接口，避免断线丢结果。

# 8. 部署与演进

结论先行：多 Agent 的部署风险集中在并发、超时、成本三处，配置必须外置并可热更。

## 8.1 配置项

```yaml
multi-agent: {max-steps: 12, cost-budget: 10.0, loop-threshold: 3,
              agent-timeout-ms: 60000, supervisor-model: gpt-4-class,
              worker-model: gpt-4o-mini}
```

依赖：JDK 17/21（record、文本块）、Spring Boot 3.x（依赖注入、WebFlux）、Spring AI 1.x（ChatClient）、Reactor 3.x（流式）。

## 8.2 生产注意

- 每任务独立 SharedState，禁止跨任务共享。
- 工具调用走统一网关，做限流与审计。
- 长任务放异步执行器，HTTP 线程只负责创建与查询。
- 成本预算按用户/租户隔离；单 Agent 超时即失败，不阻塞链路。
- 提示词版本化，便于回滚。

## 8.3 难点对照

| 难点 | 解决 |
|------|------|
| 死循环 | 循环检测 + 步数上限 |
| 成本高 | 限制 Agent 数与轮数 + 分层模型 |
| 上下文爆炸 | 交接传摘要 |
| 主管决策错误 | 强模型 + 规则兜底 |
| 难调试 | 完整协作 Trace |
| 责任扩散 | 明确 Agent 描述 |
| 终止丢结果 | 终止时也 synthesize |

## 8.4 演进路线

按 P0 单 Agent 基线、P1 Supervisor + 2~3 个子 Agent、P2 循环/成本/停滞终止策略、P3 并行子 Agent（DAG）、P4 动态注册 Agent + 自省 五步递进，每步都可独立上线验证。

# 9. 小结

## 9.1 关键结论

- Supervisor 决策 + 子 Agent 执行 + 共享状态。
- 终止策略：步数、成本、循环、停滞，四者并集。
- 交接传摘要，避免上下文爆炸。
- 完整协作 Trace 是调试基础。
- 流式进度提升长任务体验。

## 9.2 落地清单

- [ ] Agent 描述包含"何时用我"。
- [ ] Supervisor 输出做语法与语义两层校验。
- [ ] 终止时仍合成"尽力而为"答案。
- [ ] Trace 异步写、可脱敏、可回放。
- [ ] 成本预算按租户隔离。
- [ ] SSE 关闭代理缓冲、加心跳。
