# 1. 为什么需要工作流引擎

**结论先行**：当 Agent 处理长流程（跨越数小时、需要补偿与可视化）时，手写状态机已不够用，工作流引擎（Workflow Engine）能提供持久化、补偿、重试、定时等成熟能力。

## 1.1 长流程的挑战

Agent 处理长流程时会遇到工作流领域的经典问题：

| 问题 | 说明 |
|------|------|
| 长事务 | 流程跨越数小时甚至数天 |
| 补偿 | 中间步骤失败需回滚已完成步骤 |
| 定时/延迟 | 等待审批、定时触发 |
| 可视化 | 运营需要看到流程进度 |
| 重试与超时 | 每个节点独立的重试策略 |

这些问题单独看都不难，难在它们同时出现：一个流程既要等待人工审批（定时），又要在失败时回滚（补偿），还要能被运营看到进度（可视化）。

## 1.2 何时引入引擎

> 当 Agent 流程变长、需要补偿与可视化时，引入工作流引擎比手写状态机更划算。

判断标准可以用下面这张表：

| 信号 | 是否引入引擎 |
|------|--------------|
| 流程只有 2-3 步、无补偿 | 否，状态机足够 |
| 需要等待外部事件（人工/定时） | 是 |
| 中间步骤失败需回滚 | 是 |
| 需要给运营看流程图 | 是 |
| 流程版本频繁变更 | 是 |

**边界与坑**：引擎是重量级组件，引入后带来部署、运维、学习成本。对短流程"上引擎"是过度设计，反而降低迭代速度。

**生产建议**：先用状态机实现，当补偿、定时、可视化三者出现两个以上时，再迁移到工作流引擎。

一个"长流程"的典型时间线：

```
T+0s    受理
T+2s    Agent 风险判断
T+5s    进入人工复核（挂起）
T+6h    审批人处理（等待中，进程不占用资源）
T+6h5s  继续执行
T+6h10s 完成并通知
```

关键点：从 T+5s 到 T+6h 期间，流程处于"挂起"状态，引擎持久化上下文并释放线程。手写状态机若不做持久化，这段时间的进程崩溃就会丢失流程。

# 2. 与 Agent 的结合方式

**结论先行**：Agent 与工作流有三种结合方式，生产中最常见的是"Agent 作为工作流的一个节点"——用工作流保证确定性与可审计，把"需要判断"的部分交给 Agent。

## 2.1 三种结合模式

```
方式一：Agent 作为工作流的一个节点
  工作流：受理 → [Agent 智能处理] → 复核 → 归档

方式二：工作流作为 Agent 的执行引擎
  Agent 规划 → 生成工作流定义 → 引擎执行

方式三：Agent 调用工作流
  Agent 决定发起一个长流程，工作流负责后续编排
```

| 方式 | 控制权 | 适用 |
|------|--------|------|
| 方式一 | 工作流主导 | 流程确定、局部智能 |
| 方式二 | Agent 主导 | 流程动态生成 |
| 方式三 | 混合 | Agent 触发长流程 |

## 2.2 职责边界

> 生产中最常见的是**方式一**：用工作流保证确定性与可审计，把"需要判断"的部分交给 Agent。

### 2.2.1 为什么方式一最稳

工作流负责"路径确定"的部分（分支、重试、超时、补偿），Agent 负责"语义不确定"的部分（判断、生成、工具选择）。二者边界清晰，便于分别测试与治理。

### 2.2.2 方式二的代价

让 Agent 生成工作流定义（如 BPMN 或代码），灵活性最高，但流程不可预先审计、难以测试，生产环境慎用。

### 2.2.3 方式三的定位

方式三是方式一与方式二的折中：Agent 只决定"是否发起"，具体编排仍由预定义工作流完成。

**边界与坑**：让 Agent 直接修改流程定义会导致"流程不可复现"；把确定性分支也交给 Agent 会引入不必要的随机性。

**生产建议**：默认方式一；确需动态编排时，让 Agent 在受限模板内选择，而非自由生成。

方式一的伪代码骨架：

```java
// 工作流节点：Agent 判断
public RiskLevel assess(Order order) {
    // Agent 只负责"判断"，不负责"流程"
    return agent.judgeRisk(order);
}

// 工作流节点：确定性分支（由引擎执行）
// if riskLevel == HIGH -> 人工复核
// else                 -> 自动通过
```

# 3. 主流引擎

**结论先行**：选引擎看团队背景与场景——BPMN 建模选 Flowable/Camunda，代码即流程选 Temporal，轻量状态机选 Spring StateMachine，LLM 编排选 LangGraph4j。

## 3.1 引擎对比

| 引擎 | 特点 |
|------|------|
| Flowable | 开源 BPMN，成熟 |
| Camunda | 功能强，商业支持 |
| Temporal | 代码即工作流，适合开发者 |
| Spring StateMachine | 轻量状态机 |
| LangGraph4j | LLM 场景图编排 |
| Spring AI Alibaba Graph | 国内 LLM 编排 |

| 引擎 | 建模方式 | 长流程 | 补偿 | 学习曲线 |
|------|----------|--------|------|----------|
| Flowable | BPMN XML | 支持 | 支持 | 中 |
| Camunda | BPMN XML | 支持 | 支持 | 中 |
| Temporal | Java 代码 | 强 | 支持 | 中 |
| Spring StateMachine | Java 代码 | 弱 | 无 | 低 |
| LangGraph4j | 图结构 | 中 | 无 | 低 |

## 3.2 选型建议

### 3.2.1 按团队背景

有 BPM（业务流程管理）经验的团队选 Flowable/Camunda；纯研发团队选 Temporal，避免学习 BPMN。

### 3.2.2 按场景

需要给业务人员画流程图的选 BPMN；纯代码驱动、追求开发效率的选 Temporal。

### 3.2.3 LLM 场景

如果流程本身是"图"（节点是 LLM 调用、边是条件），LangGraph4j 更贴近语义，但缺乏企业级补偿与审计。

**边界与坑**：同时引入多个引擎会造成两套持久化与监控体系，运维复杂；BPMN 的 XML 与代码分离容易导致"流程改了但代码没改"。

**生产建议**：全公司统一一种主引擎；LLM 编排可作为补充，但长事务与补偿仍交给主引擎。

选型时还应评估部署形态：

| 引擎 | 部署 | 持久化 |
|------|------|--------|
| Flowable | 内嵌/独立 | 关系库 |
| Camunda | 独立 | 关系库 |
| Temporal | 独立集群 | 自带 |
| Spring StateMachine | 内嵌 | 无 |
| LangGraph4j | 内嵌 | 可选 |

# 4. Flowable 集成

**结论先行**：Flowable 用 BPMN 定义流程，Agent 节点通过 Java Delegate 接入。它的优势是流程可视化与成熟的企业级能力。

## 4.1 流程启动

```java
// 定义流程：受理 → Agent 处理 → 人工复核 → 完成
@RestController
@RequiredArgsConstructor
public class WorkflowController {

    private final RuntimeService runtimeService;
    private final TaskService taskService;

    @PostMapping("/workflow/start")
    public String start(@RequestBody OrderRequest request) {
        Map<String, Object> vars = Map.of(
                "orderId", request.orderId(),
                "status", "PENDING");

        ProcessInstance instance =
                runtimeService.startProcessInstanceByKey(
                        "orderProcess", vars);
        return instance.getId();
    }
}
```

## 4.2 Agent 节点

Agent 节点用 Java Delegate 实现：

```java
@Component("agentTaskDelegate")
@RequiredArgsConstructor
public class AgentTaskDelegate implements JavaDelegate {

    private final Assistant assistant;

    @Override
    public void execute(DelegateExecution execution) {
        String orderId = (String) execution.getVariable("orderId");

        // 调用 Agent 完成判断
        String decision = assistant.chat(
                "评估订单 %s 的风险等级，输出 LOW/MEDIUM/HIGH"
                        .formatted(orderId));

        execution.setVariable("riskLevel", decision);
    }
}
```

### 4.2.1 变量传递

`execution.setVariable` 写入流程变量，后续节点与网关可读取。变量名要稳定，避免与流程定义耦合过紧。

### 4.2.2 异常与重试

Delegate 抛异常会触发流程的错误边界事件（error boundary event），可配置重试或转人工。

```java
try {
    execution.setVariable("riskLevel", assess(orderId));
} catch (Exception e) {
    execution.setVariable("error", e.getMessage());
    throw new BpmnError("AGENT_FAILED", "Agent 处理失败");
}
```

### 4.2.3 幂等

流程可能因重试重复执行 Delegate，需用 `execution.getProcessInstanceId() + activityId` 做幂等键。

**边界与坑**：Delegate 中做长耗时 LLM 调用会占用流程引擎线程；应在 Delegate 中投递异步任务，用"等待信号"挂起流程。

**生产建议**：Delegate 保持轻量，重活交给异步任务；对 Agent 调用设置超时并映射为流程错误。

# 5. Temporal

**结论先行**：Temporal 用普通 Java 代码描述流程，由引擎负责持久化与重放，最适合开发者，且天然支持长流程与人工审批。

## 5.1 代码即工作流

```java
@WorkflowInterface
public interface OrderWorkflow {

    @WorkflowMethod
    String process(String orderId);
}

public class OrderWorkflowImpl implements OrderWorkflow {

    private final AgentActivities activities =
            Workflow.newActivityStub(AgentActivities.class,
                    ActivityOptions.newBuilder()
                            .setStartToCloseTimeout(
                                    Duration.ofMinutes(5))
                            .build());

    @Override
    public String process(String orderId) {
        String risk = activities.assessRisk(orderId);

        if ("HIGH".equals(risk)) {
            activities.requestApproval(orderId);        // 等待人工
        }
        String result = activities.execute(orderId);
        activities.notifyUser(orderId, result);
        return result;
    }
}
```

## 5.2 优势与限制

Temporal 的优势：

- **代码即流程**，无需 BPMN 建模；
- 内置重试、超时、补偿；
- 状态由引擎持久化，天然支持长流程；
- 支持"等待信号"（人工审批）。

| 维度 | Temporal | Flowable |
|------|----------|----------|
| 建模 | Java 代码 | BPMN |
| 可视化 | 弱 | 强 |
| 学习曲线 | 低（开发者） | 中 |
| 长流程 | 强 | 支持 |

### 5.2.1 工作流代码的确定性

Workflow 方法必须是确定性的（deterministic）：不能直接调用系统时间、随机数、网络。这些必须封装在 Activity 中。因为引擎会重放代码来恢复状态。

### 5.2.2 版本化

工作流代码变更会影响运行中的实例。Temporal 通过 `Workflow.getVersion` 做兼容，或使用独立任务队列隔离新旧版本。

**边界与坑**：在 Workflow 中直接做 I/O 会破坏确定性，导致重放失败；Activity 必须幂等，因为引擎可能重试。

**生产建议**：Workflow 只做编排，Activity 做副作用；Activity 一律幂等；工作流变更走版本兼容流程。

Activity 接口与幂等实现示意：

```java
public interface AgentActivities {

    @ActivityMethod
    String assessRisk(String orderId);

    @ActivityMethod
    void requestApproval(String orderId);

    @ActivityMethod
    String execute(String orderId);
}

public class AgentActivitiesImpl implements AgentActivities {

    @Override
    public String assessRisk(String orderId) {
        // 用 orderId 作为幂等键，重复调用返回同一结果
        return cache.computeIfAbsent(orderId, id -> agent.judgeRisk(id));
    }
}
```

# 6. 补偿（Saga）

**结论先行**：长事务无法用数据库事务回滚，标准方案是 Saga——每个正向操作配一个补偿操作，失败时反向执行补偿。

## 6.1 补偿模式

```
步骤1：扣款    ✅
步骤2：扣库存  ✅
步骤3：发货    ❌ 失败
        │
        ▼ 补偿
     回滚步骤2（恢复库存）
     回滚步骤1（退款）
```

## 6.2 实现

```java
public class OrderSaga {

    private final Deque<Runnable> compensations = new ArrayDeque<>();

    public void execute() {
        try {
            pay();
            compensations.push(this::refund);

            deductInventory();
            compensations.push(this::restoreInventory);

            ship();
        } catch (Exception e) {
            compensate();                     // 反向执行补偿
            throw e;
        }
    }

    private void compensate() {
        while (!compensations.isEmpty()) {
            try {
                compensations.pop().run();
            } catch (Exception e) {
                log.error("补偿失败，需人工介入", e);
            }
        }
    }
}
```

> 详见 `Reliability/8_saga_and_compensation.md`。

### 6.2.1 补偿的两种编排

| 模式 | 说明 | 适用 |
|------|------|------|
| 编排式（Orchestration） | 中心协调器依次调用 | 流程清晰 |
| 协同式（Choreography） | 各服务监听事件 | 解耦 |

### 6.2.2 补偿必须幂等

补偿操作本身也可能重试，必须幂等。例如"退款"要用退款单号去重。

### 6.2.3 补偿失败

补偿失败不能无限重试，应记录并转人工处理，同时告警。

**边界与坑**：补偿不是"撤销"，而是"业务上的对冲"（如退款而非抹除扣款）；某些操作不可补偿（如已发出的短信），需要设计为可容忍。

**生产建议**：每个正向操作上线时同步设计补偿；补偿失败进死信队列人工兜底。

# 7. 工作流与 Agent 的职责划分

**结论先行**：确定的部分交给工作流，需要判断的部分交给 Agent。这条原则决定了系统是否可测、可审计。

| 职责 | 归属 |
|------|------|
| 流程编排、分支、循环 | 工作流引擎 |
| 状态持久化 | 工作流引擎 |
| 重试、超时、补偿 | 工作流引擎 |
| 需要判断的节点 | Agent |
| 自然语言理解与生成 | Agent |
| 工具调用 | Agent（引擎可包装） |

> 原则：**确定的部分交给工作流，需要判断的部分交给 Agent**。

## 7.1 判断边界的判定

### 7.1.1 可枚举的判定

如果分支条件可以枚举（如"金额 > 1 万走人工"），属于工作流；如果条件依赖语义理解（如"这条投诉是否严重"），属于 Agent。

### 7.1.2 混合判定

常见模式是"Agent 给出结构化结论（枚举值），工作流根据结论走分支"。这样既利用了 Agent 的理解能力，又保持了流程的确定性。

```java
String level = agent.classify(complaint);   // 返回枚举字符串
// 工作流根据 level 走确定性分支
```

**边界与坑**：让 Agent 返回自由文本再解析，容易因格式漂移导致分支失败；应要求结构化输出（JSON Schema 约束）。

**生产建议**：Agent 节点统一输出结构化字段，工作流据此分支；对 Agent 输出做校验与兜底。

## 7.2 职责对照示例

| 流程片段 | 实现方 | 理由 |
|----------|--------|------|
| 判断是否需要审批 | Agent | 语义判断 |
| 审批通过后进入发货 | 工作流 | 路径确定 |
| 生成给客户的说明 | Agent | 自然语言生成 |
| 三次重试后转人工 | 工作流 | 策略确定 |
| 选择调用哪个工具 | Agent | 需要推理 |
| 调用工具后的结果校验 | 工作流 | 规则确定 |

# 8. 长流程的进度反馈

**结论先行**：长流程必须提供进度查询，否则用户与运营会陷入"不知道进行到哪一步"的焦虑。进度来自引擎的历史表。

## 8.1 进度查询

```java
@GetMapping("/workflow/{id}/progress")
public Progress progress(@PathVariable String id) {
    HistoricProcessInstance instance = historyService
            .createHistoricProcessInstanceQuery()
            .processInstanceId(id)
            .singleResult();

    List<HistoricActivityInstance> activities = historyService
            .createHistoricActivityInstanceQuery()
            .processInstanceId(id)
            .orderByHistoricActivityInstanceStartTime().asc()
            .list();

    return new Progress(
            instance.getProcessDefinitionKey(),
            activities.stream()
                    .map(HistoricActivityInstance::getActivityName)
                    .toList(),
            activities.stream().filter(a -> a.getEndTime() == null)
                    .map(HistoricActivityInstance::getActivityName)
                    .findFirst().orElse("已完成"));
}
```

## 8.2 可视化

### 8.2.1 已走路径高亮

把历史活动与流程定义中的节点对应，高亮已走路径，即可得到"当前进度图"。

### 8.2.2 预计剩余时间

结合历史同类流程的平均耗时，估算剩余时间，提升用户体验。

| 反馈方式 | 适用 |
|----------|------|
| 文字进度 | 简单、通用 |
| 进度图 | 运营后台 |
| 预计时间 | 用户前台 |

**边界与坑**：历史表数据量会随时间膨胀，直接查询全量活动会变慢；需要定期归档历史数据并分页查询。

**生产建议**：进度查询走只读库或缓存；历史数据按时间分区并定期归档。

进度查询返回结构建议稳定：

```java
public record Progress(
        String processKey,
        List<String> completedSteps,
        String currentStep
) {}
```

`completedSteps` 用于高亮已走路径，`currentStep` 用于提示"当前处理中"。对外接口只暴露这两个字段，避免把引擎内部结构泄露给前端。

# 9. 生产注意事项

## 9.1 风险与对策

| 风险 | 建议 |
|------|------|
| 引擎过重 | 简单流程用状态机即可 |
| 流程与代码不同步 | 流程定义版本化 |
| 补偿不完整 | 每个正向操作都要有补偿 |
| 长流程状态膨胀 | 定期归档历史数据 |
| 无进度反馈 | 提供进度查询接口 |
| 引擎单点 | 集群部署 |

## 9.2 补充建议

- 流程定义变更必须版本化，运行中实例沿用旧版本直至结束；
- 引擎数据库是高价值资产，需独立备份与监控；
- Agent 节点要有超时与降级策略，避免拖垮整个流程。

版本化示意：

```java
// 流程定义带版本号，运行中实例绑定启动时的版本
runtimeService.startProcessInstanceByKey("orderProcess");  // 用最新版
// 已启动实例继续使用其绑定版本，不受新部署影响
```

# 10. 小结

## 10.1 核心要点

- 长流程、补偿、可视化场景适合引入工作流引擎。
- Agent 与工作流结合：确定的部分给工作流，判断的部分给 Agent。
- Flowable/Camunda 适合 BPMN 建模，Temporal 适合代码即流程。
- 补偿（Saga）是长事务回滚的标准方案。
- 简单流程用状态机即可，不必上引擎。

## 10.2 落地建议

- 长流程必须提供进度反馈，并注意历史数据归档；
- Agent 节点应输出结构化结论，供工作流做确定性分支。
