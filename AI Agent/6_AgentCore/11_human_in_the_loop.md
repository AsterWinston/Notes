# 1. 为什么需要人工介入

结论先行：Agent 并非总是可靠，高风险动作、模糊意图、价值判断与异常兜底四类场景必须由人把关。

## 1.1 四类必须介入的场景

### 1.1.1 场景清单

Agent 并非总是可靠，某些场景必须由人把关：

```
高风险动作：转账、删除、对外发送、生产变更
模糊意图：需求不明确，需要澄清
价值判断：涉及伦理、品牌、法律
异常兜底：Agent 无法处理时转人工
```

### 1.1.2 风险与价值判断

| 场景 | 核心问题 | 介入方式 |
|------|----------|----------|
| 高风险动作 | 做错了不可逆 | 事前审批 |
| 模糊意图 | 做之前就错了 | 澄清 |
| 价值判断 | 无法用规则量化 | 人工决策 |
| 异常兜底 | 做不下去 | 转人工 |

## 1.2 人在回路的定位

### 1.2.1 自动化与人工的边界

人在回路（Human-in-the-Loop，HITL）不是"Agent 失败后的补丁"，而是**系统设计的一部分**：把不可自动化的决策显式交给人类，同时保证 Agent 仍能在获批后自主推进。

```
全自动 ────────── 人在回路 ────────── 全人工
（高风险不可用）  （可控自动化的甜点）  （效率低）
```

### 1.2.2 边界与坑

- 把所有环节都设人工：退化为"人工工作流"，失去 Agent 价值。
- 完全不设人工：高风险动作一旦出错无法挽回。
- 人工介入无记录：无法审计，也无法优化审批策略。

# 2. 人工介入的四种形态

结论先行：人工介入分为事前审批、事中干预、事后审核、兜底转人工，按介入时机与自动化程度选择。

## 2.1 四种形态对比

### 2.1.1 形态表

| 形态 | 说明 | 例子 |
|------|------|------|
| 事前审批 | 执行前人工确认 | 大额转账 |
| 事中干预 | 执行中可暂停/修改 | 长任务中途调整 |
| 事后审核 | 先执行后复核 | 内容生成后审 |
| 兜底转人工 | Agent 处理不了转人 | 客服升级 |

### 2.1.2 选择依据

| 维度 | 事前审批 | 事中干预 | 事后审核 | 兜底转人工 |
|------|----------|----------|----------|------------|
| 介入时机 | 执行前 | 执行中 | 执行后 | 任意 |
| 可逆性要求 | 需不可逆 | 需可暂停 | 需可撤销 | 无 |
| 延迟影响 | 高 | 中 | 低 | 中 |
| 适用风险 | 高 | 中高 | 中低 | 任意 |

## 2.2 事前审批

在动作发生前拦截，是高风险场景的默认选择，见第 3 节。

## 2.3 事中干预

长任务运行期间允许人工暂停、修改计划后再继续，见第 5 节。

## 2.4 事后审核与兜底转人工

事后审核用于"先做后审、错了可撤"的场景；兜底转人工用于 Agent 明确无法处理的场景，见第 7 节。

## 2.5 边界与坑

- 事后审核不能用于不可逆动作。
- 事中干预要求任务本身可暂停，否则干预无处落脚。
- 兜底转人工需要完整上下文交接，否则人工要重新问一遍。

# 3. 事前审批实现

结论先行：事前审批的核心是一个"审批闸门"，按工具名与参数判断是否需要挂起等待人工确认。

## 3.1 审批闸门

### 3.1.1 判定逻辑

```java
public class ApprovalGate {

    private final Set<String> highRiskTools = Set.of(
            "transferMoney", "deleteData", "sendEmail");

    public boolean requiresApproval(String toolName, Map<String, Object> args) {
        if (!highRiskTools.contains(toolName)) return false;
        // 金额阈值：小额可自动放行
        if ("transferMoney".equals(toolName)) {
            BigDecimal amount = new BigDecimal(args.get("amount").toString());
            return amount.compareTo(BigDecimal.valueOf(1000)) > 0;
        }
        return true;
    }
}
```

### 3.1.2 完整 Java 示例

```java
public StepResult stepWithGate(AgentState state) {
    ToolCall call = decideNextTool(state);

    if (approvalGate.requiresApproval(call.name(), call.args())) {
        ApprovalRequest req = approvalService.createRequest(
                state.taskId(), call, state.goal());        // 含理由与影响
        stateStore.save(state.taskId(), state.withStatus(AgentStatus.WAITING_HUMAN));
        notifier.notifyApprover(req);                        // 通知审批人
        return StepResult.suspended();
    }
    return execute(call);
}
```

## 3.2 执行流程

### 3.2.1 流程图

执行流程：

```
Agent 决定调用高风险工具
        │
        ▼
  挂起，创建审批单（含动作、参数、理由）
        │
        ▼
  通知审批人（页面/IM/邮件）
        │
   ┌────┴────┐
   ▼         ▼
 批准      拒绝
   │         │
   ▼         ▼
 执行    返回拒绝原因给 Agent
```

### 3.2.2 审批单模型

```java
public record ApprovalRequest(
        String id,
        String taskId,
        String toolName,
        Map<String, Object> args,
        String reason,        // 为什么需要做这件事
        String impact,        // 影响范围
        Instant createdAt,
        ApprovalStatus status
) {}
```

## 3.3 边界与坑

- 审批闸门放在工具外部：绕过工具的直接调用不受保护，应放在工具内部或统一入口。
- 阈值硬编码：业务调整后需改代码，建议配置化。
- 审批单缺少影响说明：审批人无法判断，只能盲目批准。

# 4. 状态挂起与恢复

结论先行：审批期间任务必须挂起并持久化状态，批准回调后从挂起点恢复执行。

## 4.1 挂起机制

### 4.1.1 状态枚举

```java
public enum AgentStatus { EXECUTING, WAITING_HUMAN, COMPLETED, FAILED }
```

### 4.1.2 挂起与恢复 Java 示例

```java
public StepResult step(AgentState state) {
    ToolCall call = decideNextTool(state);

    if (approvalGate.requiresApproval(call.name(), call.args())) {
        approvalService.createRequest(state.taskId(), call);   // 创建审批单
        state = state.withStatus(AgentStatus.WAITING_HUMAN);   // 挂起
        stateStore.save(state.taskId(), state);
        return StepResult.suspended();
    }
    return execute(call);
}

// 审批回调
@PostMapping("/approvals/{id}/approve")
public void onApproved(@PathVariable String id) {
    ApprovalRequest req = approvalService.approve(id);
    AgentState state = stateStore.load(req.taskId());
    state = state.withStatus(AgentStatus.EXECUTING);
    stateStore.save(req.taskId(), state);
    agentRunner.resumeAsync(req.taskId());          // 恢复执行
}
```

> **挂起必须持久化状态**，否则服务重启后无法恢复。

## 4.2 恢复的幂等性

### 4.2.1 重复执行风险

恢复时若重新执行"已批准"的工具调用，可能产生重复副作用；必须借助幂等键或"已执行记录"跳过。

```java
public void onApproved(String id) {
    ApprovalRequest req = approvalService.approve(id);
    if (idempotency.isExecuted(req.toolName(), req.args())) {
        return;                                     // 已执行，直接忽略
    }
    agentRunner.resumeAsync(req.taskId());
}
```

### 4.2.2 边界与坑

- 挂起状态未持久化：重启后任务永久卡在 WAITING_HUMAN。
- 审批超时无处理：审批单长期悬挂，需超时策略（自动拒绝或升级）。
- 并发批准：同一审批单被多次点击，需幂等与状态校验。

# 5. 事中干预

结论先行：长任务应支持暂停与人工修改计划，通过暂停标志与计划替换实现。

## 5.1 暂停与恢复

### 5.1.1 Java 示例

长任务应支持暂停与人工修改：

```java
public class InterruptibleAgent {

    private final Map<String, Boolean> pauseFlags = new ConcurrentHashMap<>();

    public void pause(String taskId) {
        pauseFlags.put(taskId, true);
    }

    public AgentState step(AgentState state) {
        if (Boolean.TRUE.equals(pauseFlags.get(state.taskId()))) {
            return state.withStatus(AgentStatus.PAUSED);   // 暂停
        }
        return doStep(state);
    }

    // 人工修改计划后继续
    public void updatePlan(String taskId, List<SubTask> newPlan) {
        AgentState state = stateStore.load(taskId);
        stateStore.save(taskId, state.withPlan(newPlan));
        pauseFlags.remove(taskId);
    }
}
```

### 5.1.2 计划修改

人工修改计划后应记录修改人、时间与原因，便于审计与复盘。

## 5.2 边界与坑

- 暂停标志只在内存：多实例部署时需放到共享存储。
- 暂停点不明确：若任务在工具执行中途暂停，需等当前步骤完成再停。
- 修改计划不校验：新计划可能违反原有约束，需做一致性校验。

# 6. 澄清式交互

结论先行：意图不明确时，Agent 应主动询问而非猜测；对高风险任务"先问清楚"比"猜着做"更安全。

## 6.1 主动澄清

### 6.1.1 Java 示例

当意图不明确时，Agent 应主动询问而非猜测：

```java
interface Clarifier {
    @UserMessage("""
            用户请求：{{it}}
            如果需要更多信息才能执行，输出需要澄清的问题；
            如果信息足够，输出"信息充足"。
            """)
    String checkClarity(String request);
}

public String handle(String request) {
    String clarity = clarifier.checkClarity(request);
    if (!clarity.contains("信息充足")) {
        return clarity;          // 返回问题，等待用户补充
    }
    return agent.run(request);
}
```

### 6.1.2 澄清与追问

| 场景 | 澄清问题示例 |
|------|--------------|
| 目标含糊 | "您指的是退款还是换货？" |
| 缺少参数 | "请提供订单号。" |
| 约束冲突 | "预算与时效冲突，优先哪个？" |
| 风险确认 | "此操作不可逆，确认执行？" |

> 对高风险任务，"先问清楚"比"猜着做"更安全。

## 6.2 边界与坑

- 澄清过度：每一步都问会令用户厌烦，应只在真正影响执行时问。
- 澄清循环：模型反复追问同一问题，需设追问次数上限。
- 澄清不落地：用户补充的信息必须写回状态，否则下一轮又问。

# 7. 兜底转人工

结论先行：当置信度不足、多次失败、明确拒绝、用户要求或命中敏感时，应触发兜底转人工，并交接完整上下文。

## 7.1 触发信号

### 7.1.1 信号表

触发转人工的信号：

| 信号 | 说明 |
|------|------|
| 低置信度 | 模型自评不确定 |
| 多次失败 | 重试仍无法完成 |
| 明确拒绝 | 模型表示无法处理 |
| 用户要求 | 用户主动要求人工 |
| 命中敏感 | 涉及投诉、法律等 |

### 7.1.2 Java 示例

```java
public Response handle(String request) {
    try {
        String result = agent.run(request);

        if (confidence(result) < THRESHOLD || containsUncertainty(result)) {
            return transferToHuman(request, "置信度不足");
        }
        return Response.ok(result);
    } catch (Exception e) {
        return transferToHuman(request, "系统异常：" + e.getMessage());
    }
}
```

## 7.2 转人工的上下文交接

### 7.2.1 交接内容

| 内容 | 用途 |
|------|------|
| 原始请求 | 让人工理解目标 |
| 已执行步骤 | 避免重复劳动 |
| 失败原因 | 快速定位 |
| 候选答案 | 供人工参考 |
| 会话历史 | 完整语境 |

### 7.2.2 边界与坑

- 只转一句话：人工需要从零开始，体验极差。
- 不告知用户已转人工：用户以为系统卡死。
- 转人工后无回写：人工处理结果未回流，Agent 无法学习。

# 8. 审批体验设计

结论先行：审批点过多会失去效率优势，设计应遵循"少而准、阈值化、批量、信息充分、可追溯"。

## 8.1 设计原则

### 8.1.1 原则表

审批点过多会失去效率优势：

| 原则 | 说明 |
|------|------|
| 少而准 | 只在高风险动作设审批 |
| 阈值化 | 小额自动放行，大额审批 |
| 批量审批 | 同类动作合并审批 |
| 信息充分 | 审批单要含动作、参数、理由、影响 |
| 可追溯 | 审批记录进入审计 |

### 8.1.2 阈值化与批量

```java
public boolean requiresApproval(String tool, Map<String, Object> args) {
    return switch (tool) {
        case "transferMoney" -> amountOf(args).compareTo(threshold) > 0;
        case "sendEmail"     -> isExternal(args);      // 仅外发需审批
        default              -> false;
    };
}
```

## 8.2 审批单信息

### 8.2.1 必备字段

| 字段 | 说明 |
|------|------|
| 动作 | 将执行什么工具 |
| 参数 | 具体参数值 |
| 理由 | Agent 为何要做 |
| 影响 | 影响范围与可逆性 |
| 时限 | 超时处理策略 |

### 8.2.2 边界与坑

- 只显示工具名：审批人无法判断风险，等于走过场。
- 无超时策略：审批单永久悬挂，任务卡死。
- 批量合并不同风险：低风险与高风险合并会削弱把关意义。

# 9. 生产注意事项

结论先行：上线前核对"审批数量、挂起持久化、超时策略、信息完整、审计、恢复幂等"六件事。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 审批点过多 | 阈值化 + 只保留高风险 |
| 挂起未持久化 | 状态入库，支持恢复 |
| 审批超时无处理 | 设超时策略（自动拒绝/升级） |
| 审批信息不足 | 展示完整上下文 |
| 无审计 | 审批全流程记录 |
| 恢复后重复执行 | 幂等 + 已执行记录 |

## 9.2 部署检查表

- 高风险工具清单与阈值配置化。
- 审批单包含动作、参数、理由、影响。
- 挂起状态写入持久化存储。
- 审批超时有明确的自动处理策略。
- 审批与执行全过程进入审计日志。

# 10. 小结

## 10.1 核心要点

- 人工介入四形态：事前审批、事中干预、事后审核、兜底转人工。
- 高风险动作必须审批，且审批点要"少而准"。
- 挂起必须持久化状态，否则无法恢复。
- 意图不明时先澄清，不要猜。
- 审批记录必须进入审计。

## 10.2 落地建议

1. 从"高风险工具白名单 + 金额阈值"起步，快速建立审批闸门。
2. 挂起状态与幂等记录一并落库，保证恢复安全。
3. 审批单信息要让人工"一眼可判"。
4. 转人工必须交接完整上下文，而非一句话。
