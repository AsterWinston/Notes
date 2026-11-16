# 1. 槽位抽取

## 1.1 抽取接口

结论先行：槽位抽取是流程自动化的入口，必须做到"宁缺勿造"，缺失字段返回空而非编造。

抽取接口接收自由文本，输出固定 schema 的键值对。底层由 LLM 完成，接口层用声明式注解绑定提示词。

## 1.2 提示词与约束

### 1.2.1 输出 schema

| 字段 | 类型 | 约束 |
|------|------|------|
| amount | number | 正数，单位元 |
| category | enum | 差旅/办公/招待 |
| reason | string | 非空，≤ 200 字 |
| date | string | yyyy-MM-dd |

### 1.2.2 防幻觉

- 明确指令："没有的不要编造"。
- 解析后做类型与范围校验，非法值丢弃。
- 低置信度字段标记 `TO_CONFIRM`，追问确认。

## 1.3 代码示例

```java
interface SlotExtractor {
    @UserMessage("""
            从用户输入中抽取以下字段（没有的不要编造）：
            - amount：金额（数字）
            - category：类别（差旅/办公/招待）
            - reason：事由
            - date：日期（yyyy-MM-dd）

            用户输入：{{it}}

            输出 JSON：{"amount":0,"category":"","reason":"","date":""}
            """)
    Map<String, Object> extract(String input);
}
```

边界与坑：

- 模型可能返回字符串 `"5000"` 而非数字，需统一转换。
- 中文日期（"上周三"）需归一化为 `yyyy-MM-dd`。

# 2. 多轮槽位填充

## 2.1 状态存储

结论：多轮填充依赖外部状态存储，状态在每轮结束后持久化。

| 存储 | 用途 | 特点 |
|------|------|------|
| 数据库 | 权威状态 | 强一致、可审计 |
| Redis | 会话缓存 | 低延迟、可失效 |

## 2.2 合并与缺失检查

- 抽取结果过滤 `null` 与空串后再合并。
- 缺失字段生成自然语言追问。
- 已填字段默认不覆盖，除非用户显式修正。

## 2.3 代码示例

```java
@Service
@RequiredArgsConstructor
public class SlotFillingService {

    private final SlotExtractor extractor;
    private final WorkflowStateStore stateStore;

    private static final List<String> REQUIRED = List.of(
            "amount", "category", "reason", "date");

    public SlotFillResult handle(String workflowId, String input) {
        WorkflowState state = stateStore.load(workflowId);
        Map<String, Object> slots = state.getSlots();

        // 抽取并合并（过滤 null）
        extractor.extract(input).forEach((k, v) -> {
            if (v != null && !isEmpty(v)) slots.put(k, v);
        });

        List<String> missing = REQUIRED.stream()
                .filter(s -> !slots.containsKey(s))
                .toList();

        state.setSlots(slots);
        stateStore.save(workflowId, state);

        if (missing.isEmpty()) {
            return SlotFillResult.complete(slots);
        }
        return SlotFillResult.needMore(
                "还需要提供：" + String.join("、", describeFields(missing)));
    }
}
```

边界与坑：

- 并发输入需加分布式锁，避免丢失更新。
- `save` 必须与状态版本号配合做乐观锁。

# 3. 规则校验

## 3.1 代码校验

### 3.1.1 阈值规则

- 按类别读取限额（`limitFor`）。
- 超额返回 WARN，转主管审批。

### 3.1.2 查重规则

- 同人、同金额、同日期窗口内视为重复。
- 命中直接 REJECT。

## 3.2 LLM 校验

- 仅在代码校验通过后调用，节省成本。
- 输出限定为 APPROVE / REVIEW / REJECT。

## 3.3 代码示例

```java
@Service
@RequiredArgsConstructor
public class ValidationService {

    private final ExpensePolicyService policyService;
    private final ChatClient chatClient;

    public ValidationResult validate(Map<String, Object> form) {
        // ① 代码校验
        double amount = ((Number) form.get("amount")).doubleValue();
        String category = (String) form.get("category");

        Double limit = policyService.limitFor(category);
        if (limit != null && amount > limit) {
            return ValidationResult.warn(
                    "金额 %s 超过 %s 类别标准 %s，需主管审批"
                            .formatted(amount, category, limit));
        }
        // ② 查重
        if (isDuplicate(form)) {
            return ValidationResult.reject("检测到重复申请");
        }
        // ③ LLM 合理性检查
        String check = chatClient.prompt()
                .system("检查申请合理性，输出 APPROVE/REVIEW/REJECT + 简短理由。")
                .user(form.toString())
                .call().content();

        return parseLlmCheck(check);
    }
}
```

边界与坑：

- `((Number) form.get(...))` 若字段缺失会 NPE，需前置必填校验。
- LLM 输出解析失败时降级为 REVIEW。

# 4. 流程接口实现

## 4.1 启动接口

结论：启动接口完成"识别意图 → 创建实例 → 首轮填充"三步。

## 4.2 输入接口

- 每轮输入调用槽位填充。
- 槽位齐全后触发校验并推进流程。

## 4.3 代码示例

```java
@RestController
@RequestMapping("/api/workflow")
@RequiredArgsConstructor
public class WorkflowController {

    private final WorkflowEngine engine;
    private final SlotFillingService slotFilling;
    private final ValidationService validationService;
    private final IntentClassifier intentClassifier;

    @PostMapping("/start")
    public WorkflowResponse start(@RequestBody StartRequest req,
                                  UserContext user) {
        // ① 识别意图
        String intent = intentClassifier.classify(req.request());

        // ② 创建流程实例
        String workflowId = engine.start(intent, user);

        // ③ 槽位填充
        SlotFillResult fill = slotFilling.handle(workflowId, req.request());

        return WorkflowResponse.from(workflowId, fill);
    }

    @PostMapping("/{id}/input")
    public WorkflowResponse input(@PathVariable String id,
                                  @RequestBody InputRequest req) {
        SlotFillResult fill = slotFilling.handle(id, req.input());

        if (fill.complete()) {
            // 槽位齐了 → 校验 → 进入审批
            ValidationResult validation = validationService.validate(fill.slots());
            engine.proceed(id, validation);
        }
        return WorkflowResponse.from(id, fill);
    }
}
```

边界与坑：

- 幂等：同一 `requestId` 重复提交应返回同一实例。
- 用户身份从 `UserContext` 获取，禁止从请求体传入。

# 5. 审批建议实现

## 5.1 上下文构造

结论：建议质量取决于上下文——申请本身、历史相似案例、相关制度三者缺一不可。

## 5.2 结构化输出

用 `.entity(ApprovalSuggestion.class)` 直接映射为对象，避免手工解析。

## 5.3 代码示例

```java
@Service
@RequiredArgsConstructor
public class ApprovalAdvisorService {

    private final ChatClient chatClient;
    private final HistoryService historyService;

    public ApprovalSuggestion advise(Application app) {
        List<Application> history = historyService.similar(app);

        return chatClient.prompt()
                .system("""
                        你是审批助手，只提供建议，不代替人决策。
                        输出 JSON：
                        {"suggestion":"APPROVE|REVIEW|REJECT","reason":"","confidence":0.0}
                        """)
                .user("""
                        申请：%s
                        历史相似申请：%s
                        """.formatted(app, format(history)))
                .call()
                .entity(ApprovalSuggestion.class);
    }
}
```

边界与坑：

- 历史检索需限流，避免上下文过长。
- 置信度是模型自评，不可作为唯一自动通过依据。

# 6. 工作流引擎集成

## 6.1 委托任务

结论：把 Agent 判断封装为引擎的委托任务（JavaDelegate），由引擎驱动调用。

## 6.2 自动通过策略

- 高置信度 APPROVE 可自动通过，阈值可配置。
- 其余一律等待人工审批。

## 6.3 代码示例

```java
@Component
@RequiredArgsConstructor
public class ApprovalDelegate implements JavaDelegate {

    private final ApprovalAdvisorService advisor;

    @Override
    public void execute(DelegateExecution execution) {
        String workflowId = (String) execution.getVariable("workflowId");
        Application app = (Application) execution.getVariable("application");

        // Agent 给出建议
        ApprovalSuggestion suggestion = advisor.advise(app);

        execution.setVariable("suggestion", suggestion.suggestion());
        execution.setVariable("suggestionReason", suggestion.reason());

        // 高置信度自动通过（可配置）
        if ("APPROVE".equals(suggestion.suggestion())
                && suggestion.confidence() > 0.95) {
            execution.setVariable("autoApproved", true);
        }
        // 否则等待人工审批
    }
}
```

边界与坑：

- 委托任务内不宜做长耗时操作，LLM 调用需设超时。
- 自动通过策略必须可审计、可回滚。

# 7. 异常补偿实现

## 7.1 补偿注册表

结论：每个步骤注册对应补偿动作，失败时按注册顺序逆序执行。

## 7.2 失败告警

- 补偿失败记录明细并告警。
- 进入人工处理队列，不静默丢弃。

## 7.3 代码示例

```java
@Service
@RequiredArgsConstructor
public class CompensationService {

    private final CompensationRegistry compensationRegistry;
    private final AlertService alertService;

    public void compensate(String workflowId, String step) {
        List<Compensation> compensations = compensationRegistry.forStep(step);

        for (Compensation c : compensations) {
            try {
                c.execute(workflowId);
            } catch (Exception e) {
                log.error("补偿失败，需人工介入: {}", workflowId, e);
                alertService.alert("COMPENSATION_FAILED", workflowId);
            }
        }
    }
}
```

边界与坑：

- 补偿动作必须幂等，允许重复执行。
- 补偿顺序、补偿范围需与正向严格对应。

# 8. 测试与评测

## 8.1 单元测试

| 模块 | 用例 |
|------|------|
| 槽位抽取 | 完整/缺失/幻觉 |
| 规则校验 | 阈值边界/查重 |
| 补偿 | 失败注入 |

## 8.2 评测集

- 覆盖 100+ 真实申请样本。
- 指标：抽取准确率、建议采纳率。

## 8.3 故障演练

- 随机杀进程，验证流程可恢复。
- 注入业务系统超时，验证补偿生效。

# 9. 难点与解决

## 9.1 难点表

| 难点 | 解决 |
|------|------|
| 多轮槽位 | 状态持久化 + 缺失提示 |
| 长流程 | 工作流引擎持久化 |
| 误审批 | 只建议，人确认 |
| 补偿 | Saga + 告警 |
| 审计 | 全流程留痕 |

## 9.2 生产注意事项

- 所有对外调用设超时、重试、熔断。
- LLM 调用记录 prompt 与响应，便于回溯。
- 状态与审计同事务，保证一致。
- 灰度发布，先小流量验证再全量。

# 10. 上线与运维

## 10.1 灰度发布

结论：先小流量灰度，验证指标后再全量，并保留一键回滚能力。

### 10.1.1 发布策略

| 阶段 | 流量 | 观察指标 |
|------|------|----------|
| 内测 | 内部账号 | 功能正确 |
| 灰度 5% | 单一部门 | 抽取准确率 |
| 灰度 30% | 多部门 | 建议采纳率 |
| 全量 | 100% | SLO 达标 |

### 10.1.2 回滚

- 流程定义版本化，回滚即切回旧版本。
- 运行中实例保持在旧版本，避免中断。
- 回滚阈值：错误率 > 1% 立即回滚。

## 10.2 监控与告警

结论：围绕流程、模型、补偿三条线建立监控。

| 监控项 | 指标 | 告警阈值 |
|--------|------|----------|
| 流程 | 卡单数 | > 10 |
| 流程 | 恢复失败 | > 0 |
| 模型 | 调用错误率 | > 5% |
| 模型 | P99 延迟 | > 8 s |
| 补偿 | 失败数 | > 0 |

## 10.3 运维手册

- 常见问题：槽位抽取失败、引擎卡单、补偿失败。
- 处置流程：定位 → 止血 → 恢复 → 复盘。
- 值班要求：关键告警 5 分钟内响应。
- 变更要求：流程定义变更需评审并留档。

# 11. 小结

## 11.1 实现要点

- 槽位填充实现多轮表单。
- 规则校验用代码 + LLM 双校验。
- Agent 只给审批建议，人最终决策。
- 用工作流引擎保证长流程可靠。
- 异常用 Saga 补偿，失败告警。
- 全流程留痕，满足审计与合规。

## 11.2 工程建议

- 所有外部调用设超时与降级。
- 状态与审计同事务写入。
- 灰度发布，指标达标再全量。
- 定期故障演练验证补偿有效。
