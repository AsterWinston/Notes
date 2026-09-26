# 1. 整体架构

**结论先行**：系统采用 Supervisor（主管）模式，主管负责分解、决策、委托与汇总，子 Agent 各司其职，所有中间结果写入共享状态，全过程记录 Trace（轨迹）。

## 1.1 架构总览

### 1.1.1 组件

```
                 用户任务
                    │
                    ▼
            ┌───────────────┐
            │  Supervisor   │  ← 决策 + 汇总
            └───────┬───────┘
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ 检索 Agent│  │ 分析 Agent│  │ 写作 Agent│
└──────────┘  └──────────┘  └──────────┘
      │             │             │
      └─────────────┼─────────────┘
                    ▼
            ┌───────────────┐
            │  共享状态      │
            └───────────────┘
```

| 组件 | 职责 |
|------|------|
| Supervisor | 决策、委托、汇总 |
| Agent | 执行单一专长子任务 |
| SharedState | 跨 Agent 上下文 |
| TraceStore | 记录与回放 |
| TerminationPolicy | 终止与降级 |

### 1.1.2 数据流

```
task → supervisor.decide → handoff → agent.handle
     → state.put → trace.record → supervisor.decide ...
     → finish → synthesize → result
```

## 1.2 部署视图

| 组件 | 部署形态 | 说明 |
|------|----------|------|
| 编排服务 | 无状态 Pod | 可水平扩展 |
| 共享状态 | Redis/DB | 会话级隔离 |
| 向量检索 | 独立服务 | 检索 Agent 调用 |
| Trace 存储 | 关系库/对象存储 | 追加写 |

```yaml
mas:
  max-steps: 8
  max-cost: 2.00
  timeout: 300s
  max-concurrency: 3
```

## 1.3 设计原则

- 显式编排优于隐式自主。
- 状态集中管理，Agent 无副作用。
- 一切可回放：输入、决策、输出全记录。

**边界与坑**：共享状态若无隔离，多任务并发会串数据。

**生产建议**：以 `taskId` 为状态命名空间，任务结束即回收。

# 2. Agent 定义

**结论先行**：Agent 是"输入请求 + 共享状态 → 结构化结果"的纯逻辑单元，接口统一以便主管调度。

## 2.1 Agent 接口

### 2.1.1 接口契约

```java
public interface Agent {
    String name();
    String description();
    AgentResult handle(AgentRequest request, SharedState state);
}
```

| 方法 | 作用 |
|------|------|
| name | 唯一标识，主管据此路由 |
| description | 供主管理解能力 |
| handle | 执行并返回结果 |

### 2.1.2 结果模型

```java
public record AgentResult(
        boolean success,
        String output,
        Map<String, Object> artifacts,
        Usage usage) {

    public static AgentResult ok(String output) {
        return new AgentResult(true, output, Map.of(), Usage.zero());
    }

    public static AgentResult fail(String reason) {
        return new AgentResult(false, reason, Map.of(), Usage.zero());
    }
}
```

## 2.2 内置 Agent

```java
public class RetrievalAgent implements Agent {
    public String name() { return "retriever"; }
    public String description() { return "检索企业知识库与联网信息"; }

    public AgentResult handle(AgentRequest req, SharedState state) {
        List<Document> docs = retriever.retrieve(req.instruction());
        state.put("documents", docs);
        return AgentResult.ok(summarize(docs));
    }
}
```

| Agent | 职责 | 输出键 |
|-------|------|--------|
| retriever | 检索 | documents |
| analyzer | 分析 | findings |
| writer | 写作 | draft |
| critic | 校验 | review |

## 2.3 自定义 Agent

实现 `Agent` 接口并注册即可，主管通过 `description` 自动感知新能力。

**边界与坑**：Agent 不应直接读写其他 Agent 的内部字段，只能通过共享状态约定键通信。

**生产建议**：为每个 Agent 定义输入输出契约文档与超时。

# 3. Supervisor 编排

**结论先行**：编排是一个"决策—委托—记录"的循环，直到主管判定完成或触发终止。

## 3.1 编排循环

### 3.1.1 决策

主管根据任务、可用 Agent 与历史轮次，输出下一步动作或"完成"。

### 3.1.2 委托

```java
public class SupervisorOrchestrator {

    private final Supervisor supervisor;
    private final Map<String, Agent> agents;
    private final SharedState state;
    private final int maxSteps;

    public String run(String task) {
        List<AgentTurn> turns = new ArrayList<>();

        for (int step = 0; step < maxSteps; step++) {
            SupervisorDecision decision = supervisor.decide(
                    describeAgents(), task, format(turns));

            if (decision.isFinish()) {
                return supervisor.synthesize(task, format(turns));
            }
            Agent agent = agents.get(decision.action());
            AgentResult result = agent.handle(
                    new AgentRequest(decision.instruction()), state);

            turns.add(new AgentTurn(decision.action(),
                    decision.instruction(), result.output()));

            traceStore.recordTurn(task, turns.get(turns.size() - 1));
        }
        return supervisor.synthesize(task, format(turns));
    }
}
```

## 3.2 决策模型

```java
public record SupervisorDecision(
        String action,          // agent 名或 "FINISH"
        String instruction,     // 给子 Agent 的指令
        String reasoning) {

    public boolean isFinish() {
        return "FINISH".equals(action);
    }
}
```

## 3.3 并发与顺序

| 场景 | 策略 |
|------|------|
| 子任务独立 | 并发执行 |
| 有依赖 | 顺序执行 |
| 只读检索 | 可并发 |

**边界与坑**：并发写共享状态会产生竞态，需按 key 隔离或加锁。

**生产建议**：并发度可配置，默认串行以保证可复现。

# 4. 共享状态

**结论先行**：共享状态是 Agent 间唯一的通信媒介，需保证线程安全、可快照、可持久化。

## 4.1 状态模型

```java
public class SharedState {

    private final Map<String, Object> state = new ConcurrentHashMap<>();

    public synchronized void put(String key, Object value) {
        state.put(key, value);
    }

    public Object get(String key) {
        return state.get(key);
    }

    public Map<String, Object> snapshot() {
        return Map.copyOf(state);
    }
}
```

## 4.2 并发安全

`ConcurrentHashMap` 保证单键读写安全，但"读—改—写"复合操作仍需外部同步。

## 4.3 持久化

| 方案 | 特点 |
|------|------|
| 内存 | 快，不可恢复 |
| Redis | 可恢复，需序列化 |
| 数据库 | 可查询，较慢 |

**边界与坑**：`Map.copyOf` 是浅拷贝，嵌套对象仍可能被修改。

**生产建议**：状态值尽量用不可变类型（record、String、List.copyOf）。

# 5. 上下文传递

**结论先行**：交接传"摘要 + 关键数据"，而非全量历史，避免上下文爆炸。

## 5.1 摘要交接

```java
public record AgentRequest(
        String instruction,
        String contextSummary,
        Map<String, Object> data
) {}
```

## 5.2 数据引用

大对象（文档、数据集）以 ID 引用，放在共享状态，避免在消息中复制。

## 5.3 上下文预算

| 项 | 预算 |
|----|------|
| 指令 | ≤ 500 token |
| 摘要 | ≤ 1000 token |
| 关键数据 | 按需 |

**边界与坑**：摘要丢失关键细节会导致下游返工，需保留"结论 + 未决问题"。

**生产建议**：摘要由固定模板生成，控制长度。

# 6. 终止与降级

**结论先行**：终止策略是安全阀，覆盖步数、成本、循环、停滞四种情形。

## 6.1 终止策略

```java
public class TerminationPolicy {

    public boolean shouldStop(int step, BigDecimal cost,
                              List<AgentTurn> turns) {
        if (step >= MAX_STEPS) return true;
        if (cost.compareTo(MAX_COST) > 0) return true;
        if (hasLoop(turns)) return true;
        if (isStagnant(turns)) return true;
        return false;
    }
}
```

## 6.2 循环与停滞检测

```java
boolean hasLoop(List<AgentTurn> turns) {
    Set<String> seen = new HashSet<>();
    for (AgentTurn t : turns) {
        String sig = t.agent() + "|" + hash(t.instruction());
        if (!seen.add(sig)) return true;
    }
    return false;
}

boolean isStagnant(List<AgentTurn> turns) {
    int n = turns.size();
    if (n < 3) return false;
    return turns.get(n - 1).output().equals(turns.get(n - 2).output())
            && turns.get(n - 2).output().equals(turns.get(n - 3).output());
}
```

## 6.3 降级路径

| 触发 | 降级动作 |
|------|----------|
| 超步数 | 用已有结果汇总 |
| 超成本 | 停止委托，直接汇总 |
| 循环 | 跳过重复，换 Agent |
| 子 Agent 失败 | 重试一次后跳过 |

**边界与坑**：降级结果可能不完整，需在响应中标注 `partial=true`。

**生产建议**：降级事件计入监控，分析高频原因。

# 7. 接口设计

**结论先行**：对外只暴露运行与轨迹两个接口，语义清晰、幂等、可回放。

## 7.1 运行接口

```java
POST /api/multiagent/run
{ "task": "分析 AI Agent 市场格局并给出建议" }
→ {
  "status": "COMPLETED",
  "result": "...",
  "steps": 6,
  "agents": ["retriever", "analyzer", "writer"],
  "cost": 1.23
}
```

| 字段 | 说明 |
|------|------|
| status | COMPLETED/PARTIAL/FAILED |
| steps | 实际轮数 |
| cost | 累计成本 |

## 7.2 轨迹接口

```java
GET /api/multiagent/{id}/trace
→ { "turns": [...] }        // 完整协作轨迹
```

## 7.3 错误模型

```java
public record ApiError(
        String code,
        String message,
        String taskId) {}
```

| 错误码 | 含义 |
|--------|------|
| MAS_TIMEOUT | 超时 |
| MAS_BUDGET | 超预算 |
| MAS_AGENT_FAIL | 子 Agent 失败 |

**边界与坑**：长任务用同步接口会超时，应改为异步 + 轮询/回调。

**生产建议**：运行接口支持幂等键，避免重复提交。

# 8. 数据模型

**结论先行**：核心实体是 Task、Turn、AgentResult，按 taskId 关联，支持回放。

## 8.1 核心实体

| 实体 | 关键字段 |
|------|----------|
| Task | id, task_text, status, steps, cost, created_at |
| Turn | id, task_id, seq, agent, instruction, output |
| AgentResult | turn_id, success, artifacts, usage |

## 8.2 表结构

```sql
CREATE TABLE mas_task (
    id          VARCHAR(36) PRIMARY KEY,
    task_text   TEXT NOT NULL,
    status      VARCHAR(16) NOT NULL,
    steps       INT NOT NULL DEFAULT 0,
    cost        DECIMAL(10,4) NOT NULL DEFAULT 0,
    created_at  TIMESTAMP NOT NULL
);

CREATE TABLE mas_turn (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    task_id     VARCHAR(36) NOT NULL,
    seq         INT NOT NULL,
    agent       VARCHAR(64) NOT NULL,
    instruction TEXT,
    output      MEDIUMTEXT,
    INDEX idx_task (task_id, seq)
);
```

## 8.3 索引与容量

| 项 | 建议 |
|----|------|
| 索引 | (task_id, seq) |
| 归档 | 轨迹按月归档 |
| 保留 | 热数据 30 天 |

**边界与坑**：轨迹体积增长快，`MEDIUMTEXT` 也需归档策略。

**生产建议**：大输出存对象存储，库中存引用。

# 9. 关键技术点

**结论先行**：多 Agent 的工程难点集中在循环、成本、上下文、调试、责任与收益六个方面。

## 9.1 问题与方案

| 问题 | 方案 |
|------|------|
| 死循环 | 步数 + 循环检测 |
| 成本高 | 限制 Agent 数与轮数 |
| 上下文爆炸 | 传摘要 |
| 难调试 | 完整 Trace |
| 责任扩散 | 明确职责 |
| 收益不足 | 对比单 Agent |

## 9.2 边界与坑

- 循环检测误判正常重试。
- 并发写状态竞态。
- 摘要丢信息导致返工。

## 9.3 生产注意事项

- 幂等键防重复提交。
- 降级标注 partial。
- 成本实时累计并告警。

**边界与坑**：只记录最终结果而不记录中间决策，出问题无法定位。

**生产建议**：Trace 保留每次决策的 reasoning。

# 10. 演进路线

**结论先行**：从固定流水线演进到自适应编排，逐步引入并行、记忆与自评。

## 10.1 近期

- 稳定 Supervisor + 3 个 Agent。
- 完善终止与降级。

## 10.2 中期

- 并行执行独立子任务。
- 引入 critic 自评与迭代。

## 10.3 长期

| 方向 | 说明 |
|------|------|
| 记忆 | 跨任务经验复用 |
| 自适应 | 动态增删 Agent |
| 成本优化 | 小模型路由 |

**边界与坑**：自适应会降低可预测性，需保留"确定性模式"开关。

**生产建议**：每次演进用评测集验证质量与成本。

# 11. 小结

- 架构：Supervisor + 子 Agent + 共享状态。
- 主管决策委托，子 Agent 各司其职。
- 交接传摘要而非全量历史。
- 终止策略：步数、成本、循环、停滞。
- 完整协作 Trace 是调试基础。
