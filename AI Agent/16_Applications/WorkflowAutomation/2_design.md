# 1. 整体架构

## 1.1 分层架构

结论先行：系统分三层——Agent 层负责"理解与判断"，工作流引擎层负责"编排与持久化"，业务系统层负责"落地执行"。

```text
用户（自然语言）
   │
   ▼
┌──────────────────────────┐
│  Agent 层                 │
│  ├── 意图识别（发起/查询） │
│  ├── 槽位填充             │
│  └── 规则校验             │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│  工作流引擎               │
│  ├── 流程编排             │
│  ├── 状态持久化           │
│  ├── 任务调度             │
│  └── 补偿                 │
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────┐
│  业务系统                 │
│  审批 / 报销 / 工单       │
└──────────────────────────┘
```

### 1.1.1 Agent 层

- 输入：用户自然语言 + 会话状态。
- 输出：结构化意图、槽位、校验结论、审批建议。
- 特性：无状态（状态存于外部存储），可水平扩展。

### 1.1.2 工作流引擎层

- 输入：流程定义 + 业务事件。
- 输出：流程实例状态流转。
- 特性：有状态、持久化、支持补偿与重试。

### 1.1.3 业务系统层

- 审批系统、报销系统、工单系统。
- 通过 API 被引擎调用，不感知 Agent 存在。

## 1.2 数据流与调用时序

```text
用户        Agent       引擎       业务系统
 │  发起     │           │           │
 │──────────▶│  意图     │           │
 │           │──────────▶│  创建实例 │
 │  追问     │◀──────────│           │
 │◀──────────│  槽位不足 │           │
 │  补充     │           │           │
 │──────────▶│  校验     │           │
 │           │──────────▶│  调用服务 │
 │           │           │──────────▶│
 │  结果     │◀──────────│◀──────────│
 │◀──────────│           │           │
```

调用原则：

- Agent 与引擎通过 API 解耦，不共享内存。
- 引擎调用业务系统必须幂等。
- 所有跨层调用带 traceId（链路追踪标识）。

# 2. 职责划分

## 2.1 职责矩阵

结论：确定的部分给引擎，判断的部分给 Agent。

| 职责 | 归属 |
|------|------|
| 自然语言理解 | Agent |
| 槽位填充 | Agent |
| 规则校验 | Agent（LLM）+ 代码 |
| 流程编排 | 工作流引擎 |
| 状态持久化 | 工作流引擎 |
| 补偿 | 工作流引擎 |
| 审批 | 人 / Agent 建议 |

> **确定的部分给引擎，判断的部分给 Agent。**

## 2.2 边界原则

### 2.2.1 给引擎的部分

- 需要"可重放、可恢复"的逻辑。
- 有明确状态机的流程。
- 需要补偿的分布式操作。

### 2.2.2 给 Agent 的部分

- 输入是非结构化自然语言。
- 规则模糊、需经验判断。
- 输出是建议而非最终决策。

# 3. 槽位填充设计

## 3.1 槽位模型

### 3.1.1 槽位定义

| 槽位 | 类型 | 必填 | 示例 |
|------|------|------|------|
| amount | number | 是 | 5000 |
| category | enum | 是 | 差旅/办公/招待 |
| reason | string | 是 | 上海出差 |
| date | date | 是 | 2026-09-20 |

### 3.1.2 槽位状态

每个槽位有三种状态：未填充、已填充、待确认（低置信度）。

```text
UNFILLED ──抽取──▶ FILLED
   ▲                  │
   └──追问/清空───────┘
              │低置信度
              ▼
          TO_CONFIRM
```

## 3.2 填充流程

结论：抽取 → 合并（不覆盖已填）→ 检查缺失 → 追问或完成。

```text
用户输入 ─▶ 抽取 ─▶ 合并 ─▶ 缺失检查 ─┬─ 有缺失 ─▶ 追问
                                      └─ 无缺失 ─▶ 完成
```

## 3.3 代码示例

```java
public class SlotFillingAgent {

    private static final List<String> REQUIRED = List.of(
            "amount", "category", "reason", "date");

    public SlotFillResult fill(String sessionId, String userInput) {
        Map<String, Object> slots = loadSlots(sessionId);

        // 抽取本轮信息
        Map<String, Object> extracted = extractor.extract(userInput);
        slots.putAll(extracted);

        // 检查缺失
        List<String> missing = REQUIRED.stream()
                .filter(s -> !slots.containsKey(s))
                .toList();

        if (!missing.isEmpty()) {
            saveSlots(sessionId, slots);
            return SlotFillResult.needMore(
                    "还需要提供：" + describe(missing));
        }
        return SlotFillResult.complete(slots);
    }
}
```

边界与坑：

- `putAll` 会覆盖旧值，多轮修正场景需显式允许覆盖。
- 抽取值为 `null` 或空串时不应视为已填。

# 4. 规则校验设计

## 4.1 双校验机制

结论：确定性规则用代码，模糊规则用 LLM，二者结果合并。

### 4.1.1 代码校验

- 金额阈值、类别限制、查重。
- 优点：快、可测、可解释。
- 缺点：无法覆盖语义合理性。

### 4.1.2 LLM 校验

- 事由是否与类别匹配、是否异常。
- 优点：覆盖长尾。
- 缺点：有延迟、可能不稳定，需约束输出。

## 4.2 决策合并

| 代码结果 | LLM 结果 | 最终 |
|----------|----------|------|
| REJECT | 任意 | REJECT |
| WARN | REJECT | REVIEW |
| WARN | APPROVE | REVIEW |
| PASS | APPROVE | APPROVE |
| PASS | REVIEW | REVIEW |

原则：任一环节拒绝则拒绝；有告警则至少转人工复核。

## 4.3 代码示例

```java
public class RuleValidator {

    public ValidationResult validate(Map<String, Object> form) {
        // ① 代码校验（确定性规则）
        if ((Double) form.get("amount") > 10000) {
            return ValidationResult.warn("金额较大，需主管审批");
        }
        if (isDuplicate(form)) {
            return ValidationResult.reject("检测到重复申请");
        }
        // ② LLM 校验（模糊规则）
        String llmCheck = llm.chat("""
                检查这个申请是否合理：
                %s
                输出：APPROVE / REVIEW / REJECT + 理由
                """.formatted(form));

        return parse(llmCheck);
    }
}
```

边界与坑：

- 强制代码校验先行，LLM 只在确定性通过后调用，节省成本。
- LLM 输出必须解析为枚举，解析失败按 REVIEW 处理。

# 5. 智能审批建议

## 5.1 建议模型

结论：Agent 输出结构化建议，人做最终决策。

| 字段 | 含义 |
|------|------|
| suggestion | APPROVE / REVIEW / REJECT |
| reason | 简短理由 |
| confidence | 置信度 0~1 |

## 5.2 提示词设计

- 明确角色：审批助手，只建议不决策。
- 提供上下文：申请、历史相似案例、相关制度。
- 约束输出：固定 JSON schema。

## 5.3 代码示例

```java
interface ApprovalAdvisor {
    @UserMessage("""
            你是审批助手。基于以下信息给出审批建议：

            申请：{{application}}
            历史记录：{{history}}
            相关制度：{{policies}}

            输出 JSON：
            {"suggestion":"APPROVE|REVIEW|REJECT","reason":"...","confidence":0.0}
            """)
    ApprovalSuggestion advise(String application, String history, String policies);
}
```

> Agent 只给**建议**，最终由人确认。

# 6. 流程与补偿

## 6.1 Saga 模式

结论：长流程用 Saga（补偿事务）保证最终一致，正向执行、逆序补偿。

### 6.1.1 正向步骤

```text
① 扣预算 ─▶ ② 创建审批单 ─▶ ③ 通知
```

### 6.1.2 补偿步骤

```text
③ 取消通知 ◀─ ② 取消审批单 ◀─ ① 释放预算
```

## 6.2 幂等设计

| 操作 | 幂等键 |
|------|--------|
| 扣预算 | workflowId + step |
| 创建审批单 | workflowId |
| 通知 | messageId |

- 每个补偿动作必须可重复执行而不产生副作用。
- 补偿失败进入人工队列，不静默丢弃。

## 6.3 代码示例

```java
public class ApprovalSaga {

    private final Deque<Runnable> compensations = new ArrayDeque<>();

    public void execute(Application app) {
        try {
            // ① 扣预算
            String budgetId = budgetService.reserve(app.amount());
            compensations.push(() -> budgetService.release(budgetId));

            // ② 创建审批单
            String approvalId = approvalService.create(app);
            compensations.push(() -> approvalService.cancel(approvalId));

            // ③ 通知
            notificationService.notify(app.approver());
        } catch (Exception e) {
            compensate();
            throw new WorkflowException("流程失败，已回滚", e);
        }
    }

    private void compensate() {
        while (!compensations.isEmpty()) {
            try {
                compensations.pop().run();
            } catch (Exception ignore) {
                // 记录并告警，继续补偿其余步骤
            }
        }
    }
}
```

边界与坑：

- 补偿本身可能失败，需记录失败项并告警。
- 补偿顺序必须与正向相反。

# 7. 接口设计

## 7.1 REST 接口

```java
POST /api/workflow/start
{
  "request": "我要申请出差报销 5000 元"
}
→ {
  "workflowId": "...",
  "status": "SLOT_FILLING",
  "missing": ["date", "reason"]
}

POST /api/workflow/{id}/input
{ "input": "上周去上海出差" }
→ { "status": "PENDING_APPROVAL", ... }

GET /api/workflow/{id}
→ { "status": "...", "progress": [...] }
```

## 7.2 状态机

```text
CREATED ─▶ SLOT_FILLING ─▶ VALIDATING ─▶ PENDING_APPROVAL
                 ▲                │              │
                 └── 追问 ────────┘         ┌────┴────┐
                                           ▼         ▼
                                        APPROVED  REJECTED
                                           │
                                           ▼
                                        COMPLETED
   任意状态 ──异常──▶ COMPENSATING ──▶ FAILED
```

## 7.3 错误码

| 错误码 | 含义 | 处理 |
|--------|------|------|
| WF-4001 | 槽位缺失 | 返回追问 |
| WF-4002 | 校验拒绝 | 终止并说明 |
| WF-4090 | 状态冲突 | 幂等返回 |
| WF-5001 | 引擎异常 | 重试/降级 |
| WF-5002 | 补偿失败 | 告警人工 |

# 8. 数据模型

## 8.1 核心实体

| 实体 | 关键字段 |
|------|----------|
| WorkflowInstance | id, status, currentStep |
| SlotData | workflowId, slots(JSON) |
| ApprovalTask | id, workflowId, assignee, decision |
| AuditLog | id, workflowId, action, actor, ts |

## 8.2 表结构

```sql
CREATE TABLE workflow_instance (
    id            VARCHAR(64) PRIMARY KEY,
    biz_type      VARCHAR(32)  NOT NULL,
    status        VARCHAR(32)  NOT NULL,
    current_step  VARCHAR(64),
    slots_json    TEXT,
    created_at    TIMESTAMP    NOT NULL,
    updated_at    TIMESTAMP    NOT NULL
);

CREATE TABLE audit_log (
    id           BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id  VARCHAR(64) NOT NULL,
    action       VARCHAR(64) NOT NULL,
    actor        VARCHAR(64),
    detail       TEXT,
    created_at   TIMESTAMP   NOT NULL
);
```

# 9. 关键技术点

## 9.1 关键点表

| 问题 | 方案 |
|------|------|
| 槽位填充 | LLM 抽取 + 缺失提示 |
| 规则校验 | 代码 + LLM 双校验 |
| 长流程 | 状态持久化 + 恢复 |
| 异常 | Saga 补偿 |
| 误审批 | Agent 只建议，人确认 |
| 审计 | 全流程留痕 |

## 9.2 生产注意事项

- 所有写操作带幂等键，防止重复提交。
- LLM 调用设超时与降级路径。
- 状态变更与审计日志在同一事务写入。
- 流程定义版本化，升级不影响运行中实例。

# 10. 小结

## 10.1 设计要点

- 架构：Agent 层 + 工作流引擎 + 业务系统。
- 确定逻辑给引擎，判断逻辑给 Agent。
- 槽位填充实现多轮表单。
- 校验采用代码 + LLM 双校验并合并决策。
- Agent 只给审批建议，人工确认。
- 长流程用状态持久化 + Saga 补偿，全流程留痕。

## 10.2 设计取舍

| 取舍点 | 选择 | 代价 |
|--------|------|------|
| 判断归属 | Agent 仅建议 | 需人工参与 |
| 一致性 | Saga 最终一致 | 存在短暂中间态 |
| 状态存储 | 关系库强一致 | 写入成本较高 |
| 模型选择 | 中模型 | 复杂场景需回退 |
