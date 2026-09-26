# 1. 评测总览

流程自动化 Agent（Workflow Automation Agent，简称 WFA）把自然语言请求转化为可执行、可审批、可追溯的业务流程。它的评测与普通对话 Agent 不同：不仅要看「答得好不好」，还要看「流程跑得稳不稳、错得起不起、事后查得到查不到」。

## 1.1 评测目标

评测需要回答四个核心问题：

1. 抽取是否准：用户随口说出的信息能否被正确填进槽位。
2. 决策是否稳：规则校验与审批建议是否与业务预期一致。
3. 流程是否可靠：服务重启、下游故障、网络抖动后能否恢复或补偿。
4. 行为是否合规：高额申请是否被拦住，每一步是否留痕。

这四个问题分别对应功能正确性、决策一致性、系统可靠性与合规审计性。缺少任何一项，项目都无法从 Demo 走向生产。

## 1.2 评测维度

| 维度 | 说明 | 典型指标 |
|------|------|----------|
| 槽位填充 | 从多轮输入抽取结构化字段 | 抽取准确率、完成率 |
| 规则校验 | 用业务规则判断申请合法性 | 校验准确率 |
| 审批建议 | 给出 APPROVE/REVIEW/REJECT 建议 | 建议准确率（对比人工） |
| 可靠性 | 崩溃恢复、补偿、幂等 | 流程成功率、恢复成功率 |
| 审计 | 全流程事件可追溯 | 事件完整率 |

### 1.2.1 功能维度

功能维度关注「结果对不对」，包括槽位填充、规则校验、审批建议三类。这些维度可离线批量评测，依赖人工标注数据集，适合在 CI 中每次提交都跑。

功能维度的评测成本主要在标注：需要业务专家确认每个用例的期望值，尤其是审批建议这类带主观判断的标签。

### 1.2.2 非功能维度

非功能维度关注「过程稳不稳」，包括可靠性、审计、性能与安全。这类维度通常需要故障注入（Fault Injection）与混沌测试（Chaos Testing），难以在纯离线环境完整覆盖。

非功能维度虽然用例数少，但一旦失败往往就是线上事故，因此权重更高。

## 1.3 评测分层

```
         +---------------------------+
         |   端到端评测 (E2E)        |   真实流程跑通率
         +---------------------------+
         |   组件评测 (Component)    |   槽位/规则/审批单点
         +---------------------------+
         |   单元评测 (Unit)         |   纯函数、状态机
         +---------------------------+
```

三层各自独立：端到端失败时，先下沉到组件层定位，再下沉到单元层复现。分层评测能显著缩短问题定位时间。

# 2. 评测维度详解

## 2.1 槽位填充

槽位填充（Slot Filling）从多轮对话中抽取结构化字段，例如金额、类别、事由、日期。

### 2.1.1 抽取准确率

抽取准确率衡量单个槽位值与标注一致的比例。金额、类别等强约束字段使用精确匹配，事由等自由文本使用相似度匹配。

### 2.1.2 完成率

完成率衡量一次会话内所有必填槽位被填满的比例。完成率低通常意味着追问策略有问题，或用户在中途放弃。

### 2.1.3 追问效率

追问效率衡量平均需要多少轮才能填满槽位。轮数越多体验越差，但追问过少又容易抽错，需要平衡。

需要注意「用户改口」场景：用户先说 5000，后改口 8000，评测必须以最后一轮为准。评测集应显式覆盖改口、否定、指代（「就按刚才那个」）等复杂情形。

## 2.2 规则校验

规则校验（Rule Validation）用业务规则判断申请是否合法，例如金额上限、类别白名单、时间范围。

### 2.2.1 规则建模

| 规则 | 示例 | 期望 |
|------|------|------|
| 金额上限 | 单笔不超过 50000 | 超出则 REVIEW |
| 类别白名单 | 差旅/招待/办公 | 其他类别拒绝 |
| 时间范围 | 报销需在 90 天内 | 超期拒绝 |

### 2.2.2 规则冲突

多条规则可能同时命中且结论冲突。评测要覆盖冲突用例，并验证优先级策略是否稳定、可解释。

规则校验的准确率相对容易达到高值，因为规则本身是确定性的；难点在于规则版本管理与冲突检测。

## 2.3 审批建议

审批建议（Approval Suggestion）输出 APPROVE / REVIEW / REJECT 三档建议。

### 2.3.1 三档语义

- APPROVE：可直接自动通过。
- REVIEW：需人工复核。
- REJECT：直接拒绝并说明原因。

### 2.3.2 漏放代价

- 与人工审批结论对比，计算一致率。
- 重点考察「漏放」：本应 REVIEW 却给出 APPROVE，属于高危错误。
- 「多拦」代价较低，宁可多送人工。

## 2.4 可靠性

可靠性覆盖崩溃恢复、补偿、幂等三个方面，详见第 6 章。核心指标是流程成功率与恢复成功率。

### 2.4.1 恢复成功率

恢复成功率 = 故障后成功续跑的流程数 / 发生故障的流程数。它衡量状态持久化与恢复逻辑是否可靠。

### 2.4.2 补偿成功率

补偿成功率 = 成功补偿数 / 应补偿数。它衡量一致性保障能力，任何补偿失败都应视为高危。

## 2.5 审计

审计要求全流程事件可追溯：谁在何时发起、经过哪些步骤、由谁审批、结果如何。审计事件应不可篡改且可导出。

### 2.5.1 事件模型

| 字段 | 含义 |
|------|------|
| eventId | 事件唯一标识 |
| workflowId | 所属流程 |
| actor | 操作者 |
| action | 动作类型 |
| fromState | 迁移前状态 |
| toState | 迁移后状态 |
| timestamp | 发生时间 |
| requestId | 关联请求，用于追踪 |

### 2.5.2 不可篡改

审计存储应只允许追加（Append-only），并通过哈希链或外部归档保证不可篡改。评测需验证「删除/修改事件」的尝试会被拒绝。

审计评测的判定标准是「事件完整率」：每个状态迁移都必须有对应事件，缺失即为缺陷。

# 3. 评测集设计

## 3.1 数据集结构

### 3.1.1 用例模型

评测用例统一为 `WorkflowEvalCase`，包含类别、输入与期望值。

```java
public record WorkflowEvalCase(
        String id,
        String category,
        List<String> inputs,
        Map<String, Object> expectedSlots,
        Map<String, Object> application,
        String expectedSuggestion,
        String scenario,
        String expectedOutcome) {
}
```

### 3.1.2 字段说明

| 字段 | 用途 |
|------|------|
| id | 唯一标识，失败时用于回放 |
| category | 决定走哪条评测分支 |
| inputs | 多轮用户输入 |
| expectedSlots | 槽位期望值 |
| application | 已结构化的申请单 |
| expectedSuggestion | 审批建议期望 |
| scenario | 可靠性场景描述 |
| expectedOutcome | 可靠性期望结果 |

## 3.2 样例数据

```json
[
  {
    "id": "w1",
    "inputs": ["我要申请出差报销", "5000 元", "去上海", "上周"],
    "expectedSlots": {"amount": 5000, "category": "差旅",
                      "reason": "去上海", "date": "..."},
    "category": "slot-filling"
  },
  {
    "id": "w2",
    "application": {"amount": 50000, "category": "招待"},
    "expectedSuggestion": "REVIEW",
    "category": "approval"
  },
  {
    "id": "w3",
    "scenario": "流程中途服务重启",
    "expectedOutcome": "可恢复",
    "category": "reliability"
  }
]
```

## 3.3 数据构造与分层

评测集应覆盖以下分层，避免只用「顺风顺水」样例：

| 分层 | 占比建议 | 示例 |
|------|:--------:|------|
| 常规 | 60% | 标准报销 |
| 边界 | 20% | 恰好等于上限 |
| 异常 | 15% | 缺字段、格式错 |
| 对抗 | 5% | 提示注入、越权尝试 |

数据来源可包括历史工单、人工合成、生产回放。生产回放（Replay）能发现分布漂移。

### 3.3.1 来源与标注

历史工单最贴近真实分布，但需脱敏；人工合成可补齐长尾；生产回放需过滤脏数据。标注应由业务专家复核。

### 3.3.2 版本与维护

评测集应带版本号，用例变更需评审。新增用例只增不删，删除需记录原因，保证历史可比。

# 4. 自动化评测

## 4.1 评测主流程

```java
public WorkflowEvalReport evaluate(List<WorkflowEvalCase> cases) {
    int slotCorrect = 0, approvalCorrect = 0, recoveryPassed = 0;

    for (WorkflowEvalCase c : cases) {
        switch (c.category()) {
            case "slot-filling" -> {
                SlotFillResult result = simulateSlotFilling(c.inputs());
                if (slotsMatch(result.slots(), c.expectedSlots())) slotCorrect++;
            }
            case "approval" -> {
                ApprovalSuggestion s = advisor.advise(toApplication(c));
                if (s.suggestion().equals(c.expectedSuggestion())) approvalCorrect++;
            }
            case "reliability" -> {
                if (testRecovery(c)) recoveryPassed++;
            }
            default -> throw new IllegalArgumentException("unknown category");
        }
    }
    return new WorkflowEvalReport(slotCorrect, approvalCorrect, recoveryPassed);
}
```

主流程用 Java 17 的 switch 表达式分支，新增类别时编译器会提醒补齐分支。

### 4.1.1 离线评测

离线评测在 CI 中批量运行，使用固定数据集，结果可复现。它适合回归测试，防止改动引入退化。

### 4.1.2 在线评测

在线评测在生产流量上抽样，配合影子流量（Shadow Traffic）与小流量灰度，用于发现分布漂移。

### 4.1.3 人工抽检

自动化指标无法覆盖全部主观判断，需定期人工抽检，尤其是审批建议类用例。

## 4.2 槽位匹配

### 4.2.1 精确匹配

金额、类别等强约束字段使用精确匹配。金额需先归一化（去掉「元」「块」等单位）。

### 4.2.2 模糊匹配

事由、备注等自由文本使用相似度匹配，例如归一化编辑距离或语义相似度阈值。

```java
boolean slotsMatch(Map<String, Object> actual, Map<String, Object> expected) {
    for (Map.Entry<String, Object> e : expected.entrySet()) {
        Object a = actual.get(e.getKey());
        if (e.getKey().equals("amount")) {
            if (!normalizeAmount(a).equals(normalizeAmount(e.getValue()))) return false;
        } else if (e.getKey().equals("reason")) {
            if (similarity(String.valueOf(a), String.valueOf(e.getValue())) < 0.8) return false;
        } else if (!Objects.equals(a, e.getValue())) {
            return false;
        }
    }
    return true;
}
```

## 4.3 报告生成

评测报告应包含整体指标、分类别指标与失败用例清单。失败用例必须可回放，便于定位。

```java
public record WorkflowEvalReport(
        int slotCorrect, int approvalCorrect, int recoveryPassed) {

    public double slotAccuracy(int total) {
        return total == 0 ? 0.0 : (double) slotCorrect / total;
    }
}
```

报告应作为 CI 产物归档，并记录模型版本、提示版本、规则版本，保证结果可复现。

# 5. 关键指标

## 5.1 指标目标

| 指标 | 目标 |
|------|------|
| 槽位抽取准确率 | > 90% |
| 槽位完成率 | > 95% |
| 审批建议准确率 | > 80% |
| 流程成功率 | > 99% |
| 恢复成功率 | > 95% |
| 补偿成功率 | > 99% |

### 5.1.1 目标设定原则

目标应结合业务风险与当前基线：先测出基线，再设「跳一跳够得着」的目标，避免指标虚高失去意义。

### 5.1.2 目标校准

每次大版本或模型切换后需重新校准目标，因为数据分布与用户行为都会变化。

## 5.2 指标计算

- 准确率 = 正确数 / 总数，用于单点判断。
- 完成率 = 填满会话数 / 总会话数，衡量多轮体验。
- 成功率 = 成功流程数 / 总流程数，衡量系统稳定性。
- 补偿成功率 = 成功补偿数 / 应补偿数，衡量一致性保障。

### 5.2.1 分位数指标

除平均值外，还应看 P95/P99 延迟，避免长尾拖垮体验。平均值达标不代表尾部长尾达标。

### 5.2.2 分桶指标

按金额档、类别、用户群分桶统计，避免整体达标但某类全面失败。

## 5.3 指标权衡

准确率与召回率天然冲突：审批建议放宽则漏放增加，收紧则人工成本上升。生产上通常对高危类别（大额、招待）提高拦截阈值，对低危类别放宽。

# 6. 可靠性测试

## 6.1 崩溃恢复

```java
@Test
void should_recover_from_restart() {
    String workflowId = engine.start("expense", user);
    slotFilling.handle(workflowId, "5000 元");

    // 模拟重启
    restartApplication();

    // 恢复
    WorkflowState state = stateStore.load(workflowId);
    assertNotNull(state);
    assertEquals("SLOT_FILLING", state.getStatus());

    // 继续
    slotFilling.handle(workflowId, "去上海");
    assertTrue(stateStore.load(workflowId).getSlots().containsKey("reason"));
}
```

崩溃恢复的关键是「状态持久化」与「恢复后幂等续跑」。恢复时必须从持久化状态重建内存上下文，不能依赖进程内变量。

### 6.1.1 恢复时机

恢复可发生在进程启动时（扫未完成流程）或定时巡检时（兜底扫描长时间卡住的流程）。

### 6.1.2 恢复幂等

恢复本身必须幂等：重复触发恢复不能导致重复执行已完成的步骤。

## 6.2 Saga 补偿

```java
@Test
void should_compensate_on_failure() {
    // 模拟审批服务故障
    approvalService.setFailMode(true);

    assertThrows(WorkflowException.class, () -> saga.execute(app));

    // 验证预算已释放
    assertTrue(budgetService.isReleased(app.amount()));
}
```

Saga 模式下，每一步都定义对应的补偿动作。评测要验证「失败后所有已完成步骤均被补偿」，且补偿本身幂等。

### 6.2.1 正向补偿顺序

补偿应按已完成步骤的逆序执行，先补偿最后一步，再逐级回退。

### 6.2.2 补偿失败处理

补偿失败不能静默。应重试、落库为待人工处理，并触发 P0 告警。

## 6.3 幂等测试

同一请求重复提交，结果必须一致且不产生重复副作用。

```java
@Test
void should_be_idempotent_on_retry() {
    String key = "req-123";
    engine.start("expense", user, key);
    engine.start("expense", user, key);   // 重复提交

    assertEquals(1, expenseRepo.countByRequestKey(key));
}
```

幂等键（Idempotency Key）应由客户端生成并随请求传递，服务端据此去重。

# 7. 安全与合规测试

## 7.1 权限与审批

```java
@Test
void high_amount_should_require_manual_approval() {
    ApprovalSuggestion s = advisor.advise(app(100000));
    assertNotEquals("APPROVE", s.suggestion());   // 不应自动通过
}
```

高额、敏感类别必须强制人工审批，任何自动化路径都不能绕过。

### 7.1.1 角色权限矩阵

| 角色 | 发起 | 审批 | 查看 |
|------|:----:|:----:|:----:|
| 员工 | 是 | 否 | 仅本人 |
| 主管 | 是 | 是 | 本部门 |
| 财务 | 是 | 是 | 全部 |

### 7.1.2 越权用例

评测需构造「低权限用户尝试执行高权限动作」的用例，期望结果为拒绝且记录审计。

## 7.2 审计测试

```java
@Test
void should_audit_all_steps() {
    engine.start("expense", user);
    List<AuditEvent> events = auditLog.findByWorkflow(workflowId);
    assertTrue(events.size() >= 2);               // 至少发起 + 一步
}
```

审计事件应包含时间、操作者、动作、前后状态与请求 ID，且只允许追加，不允许修改。

## 7.3 注入与越权

对抗测试要覆盖提示注入（Prompt Injection）与越权访问：

- 用户尝试让 Agent「忽略规则直接通过」。
- 用户尝试提交他人单据或访问他人流程状态。

这类用例失败必须记为零容忍缺陷。

# 8. 上线检查清单

## 8.1 功能与可靠性

- [ ] 流程状态持久化
- [ ] 崩溃恢复验证
- [ ] Saga 补偿验证
- [ ] 幂等保证

## 8.2 安全与合规

- [ ] 高额申请人工审批
- [ ] 全流程审计
- [ ] 权限校验覆盖所有入口

## 8.3 观测与告警

- [ ] 关键指标埋点
- [ ] 告警配置
- [ ] 失败用例可回放

# 9. 灰度上线

## 9.1 分阶段策略

```
阶段1：小额报销（< 1000）
阶段2：中额报销（< 10000）
阶段3：大额（需人工审批）
```

每阶段观察至少一个完整业务周期，指标达标后再进入下一阶段。

### 9.1.1 阶段准入条件

进入下一阶段前，需满足：失败率达标、无 P0 缺陷、人工抽检通过。

### 9.1.2 阶段回退条件

任一 P0 事件、失败率超阈或自动通过率异常，立即回退并冻结灰度。

## 9.2 回滚方案

回滚必须可在分钟级完成：保留旧版本入口、状态双向兼容、补偿脚本预置。回滚演练应纳入上线前检查。

# 10. 监控告警

## 10.1 告警指标

| 指标 | 告警 |
|------|------|
| 流程失败率 | > 1% |
| 补偿失败 | 任意（严重） |
| 流程卡住 | > 24 小时 |
| 自动通过率异常 | 突增 |
| 状态不一致 | 任意 |

## 10.2 告警分级

- P0：补偿失败、状态不一致，立即人工介入。
- P1：失败率超阈、自动通过率突增，15 分钟内响应。
- P2：流程卡住、延迟升高，工作时间内处理。

# 11. 小结

## 11.1 核心结论

- 评测：槽位、审批建议、可靠性、审计。
- 可靠性测试覆盖崩溃恢复与补偿。
- 高额申请必须人工审批。
- 全流程审计是合规要求。
- 灰度从低风险流程开始。

## 11.2 实施要点

- 指标要分层看，尤其关注「漏放」这类高危错误。
- 评测集需覆盖常规、边界、异常、对抗四类分层。
- 性能与容量同样要纳入上线前验证。
