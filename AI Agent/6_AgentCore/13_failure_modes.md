# 1. 为什么失败模式重要

生产事故中，**大部分 Agent 失败不是"模型不够强"，而是控制流、工具、上下文设计缺陷**。认识失败模式才能有针对性地防御。

## 1.1 失败不等于模型弱

结论先行：换更强的模型往往只能把失败率从 30% 降到 25%，而修好控制流可以把它降到 3%。原因在于：

- 模型的随机性无法根除，只能约束；
- 多步任务的失败概率是**乘性累积**的，单步 95% 成功率在 20 步后只剩 36%；
- 绝大多数失败有**结构性根因**，与模型强弱无关。

## 1.2 失败模式的价值

| 价值 | 说明 |
|------|------|
| 预判 | 上线前知道会在哪里坏 |
| 检测 | 用运行时信号识别失败 |
| 兜底 | 失败后仍有可接受的输出 |
| 回流 | 失败案例沉淀为评测集 |

## 1.3 本文结构

先建立分类（第 2 节），再逐类拆解机制与对策（第 3~8 节），最后给出检测、兜底与案例分析方法（第 9~10 节）。

# 2. 失败模式分类

## 2.1 六类总览

```
┌──────────────────────────────────────────────┐
│ ① 循环类   死循环、重复调用、反复重试          │
│ ② 推理类   幻觉、目标漂移、错误决策            │
│ ③ 工具类   误选工具、参数错误、越权            │
│ ④ 上下文类 上下文爆炸、信息丢失、噪声干扰      │
│ ⑤ 系统类   超时、并发冲突、状态不一致          │
│ ⑥ 成本类   成本失控、Token 爆炸               │
└──────────────────────────────────────────────┘
```

| 类别 | 一句话根因 | 首要对策 |
|------|-----------|----------|
| 循环类 | 缺少终止与去重 | 最大步数 + 调用签名去重 |
| 推理类 | 模型在信息不足时仍输出 | 约束依据 + 验证 |
| 工具类 | 描述与权限边界模糊 | 明确边界 + 鉴权 |
| 上下文类 | 信息组织方式不当 | 窗口 + 压缩 + 重排 |
| 系统类 | 缺少超时/幂等/隔离 | 分层超时 + 状态外置 |
| 成本类 | 缺少预算与路由 | 预算 + 模型分层 + 缓存 |

## 2.2 失败的三层根因

### 2.2.1 控制流层

终止条件缺失、循环无上限、重试无退避、并发无串行化。

### 2.2.2 上下文层

关键信息未前置、历史全量堆积、工具结果未裁剪、噪声未过滤。

### 2.2.3 工具层

描述重叠、参数无校验、越权未拦截、结果不可解析、写操作不幂等。

> 三层根因相互叠加：一次失败通常同时命中多个类别，定位时要按层排查。

## 2.3 失败模式的叠加与优先级

| 叠加组合 | 典型后果 | 优先修哪层 |
|----------|----------|-----------|
| 循环 + 成本 | 短时间内烧掉大量 Token | 控制流层（先加步数上限） |
| 工具 + 系统 | 重试放大故障、雪崩 | 系统层（先加熔断与幂等） |
| 上下文 + 推理 | 信息丢失导致幻觉 | 上下文层（先做关键信息前置） |
| 推理 + 工具 | 越权执行高风险动作 | 工具层（先加鉴权与审批） |

**优先级原则**：先修**会造成不可逆损失**的（越权、重复副作用），再修**只损失成本**的（循环、Token 爆炸）。

# 3. ① 循环类失败

## 3.1 死循环

### 3.1.1 表现

```
步1: getOrder("A123") → "不存在"
步2: getOrder("A123") → "不存在"
步3: getOrder("A123") → "不存在"   ← 无限重复
```

### 3.1.2 机制

模型把"失败"理解为"重试即可"，而工具返回的确定性错误（如"不存在"）并不会因重试而改变，于是形成固定点循环。

### 3.1.3 对策

- **最大步数**：硬上限（如 15 步），超出即终止；
- **调用签名去重**：相同 `工具名 + 参数哈希` 连续出现即判定卡死；
- **提示约束**：明确"同一确定性错误不得重复尝试超过 2 次"。

## 3.2 反复重试无效操作

```
工具失败 → 重试 → 失败 → 重试 → ...（无退避、无策略变更）
```

**对策**：限制重试次数 + 指数退避 + 失败后改变策略。

```java
record RetryPolicy(int maxAttempts, long baseBackoffMs) {
    boolean shouldRetry(int attempt, Throwable cause) {
        return attempt < maxAttempts && !(cause instanceof BusinessException);
    }
    long backoff(int attempt) {
        return baseBackoffMs * (1L << attempt);   // 指数退避
    }
}
```

## 3.3 边界与反例

- **反例**：把所有重试都设成 0，导致瞬时网络抖动直接失败。正确做法是区分**可重试错误**（超时、5xx）与**不可重试错误**（参数非法、业务拒绝）。
- **边界**：去重窗口太短会误伤正常重复（如分页查询），应结合参数与时间窗。

## 3.4 循环防护实现

```java
public class LoopGuard {
    private final int maxSteps;
    private final Deque<String> recentSignatures = new ArrayDeque<>();

    public LoopGuard(int maxSteps) {
        this.maxSteps = maxSteps;
    }

    public void check(int step, String signature) throws LoopDetectedException {
        if (step > maxSteps) {
            throw new LoopDetectedException("超过最大步数 " + maxSteps);
        }
        long same = recentSignatures.stream().filter(signature::equals).count();
        if (same >= 2) {
            throw new LoopDetectedException("重复调用：" + signature);
        }
        recentSignatures.addLast(signature);
        if (recentSignatures.size() > 20) {
            recentSignatures.removeFirst();
        }
    }
}
```

四道闸必须同时具备：**步数上限、签名去重、Token 预算、时间上限**。任一缺失都可能被绕过。

# 4. ② 推理类失败

## 4.1 幻觉（Hallucination）

### 4.1.1 表现

| 表现 | 例子 |
|------|------|
| 编造事实 | 虚构不存在的政策条款 |
| 编造工具 | 调用不存在的工具 |
| 编造引用 | 伪造来源链接 |

### 4.1.2 机制

模型被训练为"尽量给出答案"，在信息不足时倾向用**最可能的文本**填补空白，而非承认未知。

### 4.1.3 对策

- 要求"只依据给定资料，不足则拒答"；
- 工具集固定 + 白名单校验；
- 引用必须可验证；
- 关键结论用工具/代码验证。

## 4.2 目标漂移

多步之后忘记原始目标，偏离任务。

**对策**：每轮重申目标；定期用"目标检查"提示自检。

```java
String check = """
        原始目标：%s
        当前进展：%s
        问题：当前动作是否仍在服务于原始目标？若否，说明偏离并纠正。
        """.formatted(goal, progress);
```

## 4.3 过早终止

信息不足就给出答案。

**对策**：明确"完成标准"；要求列出"已完成/未完成"清单。

## 4.4 过度自信

模型对错误结论高度自信。

**对策**：要求给出置信度；低置信度转人工。

## 4.5 边界与反例

- **反例**：为了防幻觉而要求"任何情况都必须引用"，导致简单常识问题也被迫检索，成本飙升。
- **边界**：置信度是模型的**自评**，与真实正确率相关性有限，只能作为辅助信号，不能单独作为放行依据。

# 5. ③ 工具类失败

## 5.1 误选工具

| 失败 | 原因 | 对策 |
|------|------|------|
| 误选工具 | 描述不清、工具重叠 | 明确边界、工具分组 |
| 参数错误 | Schema 不明确 | 类型化参数 + 校验 |
| 越权调用 | 未做权限控制 | 工具级鉴权 + 动态挂载 |
| 忽略结果 | 结果过长/不结构化 | 精简结构化返回 |
| 重复副作用 | 重试 + 非幂等 | 幂等键 + 去重 |

## 5.2 参数错误

模型可能给出格式正确但语义错误的参数（如日期 `2024-13-01`）。对策是在工具入口做**强校验并返回可读错误**：

```java
@Tool("查询指定日期的订单")
String queryOrders(@P("日期，格式 yyyy-MM-dd") String date) {
    if (!DatePattern.isValid(date)) {
        return "参数错误：date 必须为 yyyy-MM-dd，例如 2024-06-01";
    }
    // ...
}
```

## 5.3 越权调用

工具级鉴权 + 动态挂载，缺一不可：

```java
public List<Object> toolsFor(User user, String scene) {
    return allTools.stream()
            .filter(t -> permissionService.canUse(user, t, scene))
            .toList();
}
```

## 5.4 忽略结果与重复副作用

- **忽略结果**：工具返回大段文本，模型只读开头 → 返回结构化摘要；
- **重复副作用**：重试下单导致重复扣款 → 幂等键 + 服务端去重。

## 5.5 完整防护示例

```java
@Tool("创建退款单")
RefundResult createRefund(@P("订单号") String orderId,
                          @P("幂等键") String idempotencyKey) {
    SecurityUtils.requireScene("refund");                  // 鉴权
    if (store.exists(idempotencyKey)) {                    // 幂等
        return store.get(idempotencyKey);
    }
    auditLog.record(currentUser(), "createRefund", orderId);
    RefundResult r = refundService.create(orderId);
    store.save(idempotencyKey, r);
    return r;
}
```

# 6. ④ 上下文类失败

## 6.1 上下文爆炸

| 失败 | 表现 | 对策 |
|------|------|------|
| 上下文爆炸 | 消息越积越多，超 Token 上限 | 窗口 + 压缩 |
| 信息丢失 | 早期关键信息被截断 | 摘要保留关键信息 |
| 噪声干扰 | 检索到大量无关内容 | 提高阈值 + 重排 |
| 位置偏差 | 忽略中间位置信息 | 关键信息前置 |

## 6.2 信息丢失

滑动窗口会丢弃最早的对话，而最早的往往正是**原始目标与约束**。对策：把目标、约束、关键事实抽成**固定摘要区**，永不参与截断。

```
[System]     角色 + 规则
[Pinned]     目标 / 硬约束 / 已确认事实      ← 永不截断
[Summary]    历史压缩摘要                    ← 定期重写
[Recent]     最近 N 轮原始消息               ← 滑动窗口
[Tool]       本轮工具结果（裁剪后）
```

## 6.3 噪声干扰与位置偏差

- **噪声**：检索返回大量低相关文档，稀释信号。对策：相似度阈值 + 重排（Rerank）+ 数量上限；
- **位置偏差**：模型对上下文中间内容注意力最弱。对策：关键指令放首尾。

> 详见 `8_Memory/2_context_window_management.md`。

## 6.4 生产建议

- 为上下文设**硬 Token 上限**，超限强制压缩；
- 压缩后校验关键事实是否仍在；
- 工具结果先裁剪（只留必要字段）再回填。

# 7. ⑤ 系统类失败

## 7.1 超时与依赖不可用

| 失败 | 说明 | 对策 |
|------|------|------|
| 超时 | 模型/工具无响应 | 分层超时 |
| 并发冲突 | 同会话并发写 | 会话级串行/锁 |
| 状态不一致 | 多实例状态不同步 | 状态外置 + 事务 |
| 恢复重复执行 | 崩溃恢复后重放 | 幂等 + 已执行记录 |
| 依赖不可用 | 向量库/模型宕机 | 熔断 + 降级 |

分层超时：**总超时 < 各步超时之和**，且单步超时逐级递减，避免内层长时间占用外层预算。

## 7.2 并发冲突

同一会话的两次请求并发写同一状态，产生覆盖。对策：会话级串行队列或乐观锁（版本号 CAS）。

```java
boolean saveIfVersion(String taskId, AgentState state, long expectedVersion) {
    return store.compareAndSet(taskId, expectedVersion, state);   // 伪代码
}
```

## 7.3 状态不一致与恢复重复

状态放在内存中，多实例各自为政；崩溃恢复后重放已执行的副作用步骤。对策：状态外置 + 已执行步骤记录 + 幂等。

## 7.4 分层超时实现

```java
public record TimeoutPolicy(long totalMs, long llmMs, long toolMs) {
    public static TimeoutPolicy of(long totalMs) {
        return new TimeoutPolicy(totalMs, totalMs / 2, totalMs / 4);
    }
}

public String runWithDeadline(String goal, TimeoutPolicy policy) {
    long deadline = System.nanoTime() + policy.totalMs() * 1_000_000L;
    String answer = model.chat(goal, Duration.ofMillis(policy.llmMs()));   // 伪代码
    if (System.nanoTime() > deadline) {
        throw new TimeoutException("总预算已耗尽");
    }
    return answer;
}
```

> 内层超时必须**小于**外层，否则外层预算会被内层悄悄耗尽。

# 8. ⑥ 成本类失败

## 8.1 Token 爆炸

| 失败 | 原因 | 对策 |
|------|------|------|
| Token 爆炸 | 上下文过长、循环过多 | 预算 + 上限 |
| 模型滥用 | 简单任务用大模型 | 模型分层路由 |
| 无缓存 | 重复调用相同请求 | 结果/语义缓存 |
| 无监控 | 成本不可见 | 按租户/模型统计告警 |

## 8.2 模型滥用与缓存

```java
public ChatModel route(String task, int estTokens) {
    if (task.equals("intent") || estTokens < 500) return smallModel;   // 路由
    return largeModel;
}
```

- **结果缓存**：相同入参直接命中；
- **语义缓存**：近似问题复用答案（需设相似度阈值）。

## 8.3 生产建议

- 每任务设 Token 预算，超预算即降级或终止；
- 按租户/模型维度统计成本并告警；
- 缓存键包含模型版本与提示版本，避免脏命中。

## 8.4 预算实现

```java
public class TokenBudget {
    private final int maxTokensPerTask;
    private int used;

    public TokenBudget(int maxTokensPerTask) {
        this.maxTokensPerTask = maxTokensPerTask;
    }

    public void consume(int tokens) {
        used += tokens;
        if (used > maxTokensPerTask) {
            throw new BudgetExceededException(used, maxTokensPerTask);
        }
    }

    public int remaining() {
        return maxTokensPerTask - used;
    }
}
```

超预算时应**降级**（换小模型 / 返回部分结果）而非静默继续。

# 9. 失败检测与兜底

## 9.1 检测器实现

```java
public class FailureDetector {

    public boolean isStuck(List<ToolCall> history) {
        // 连续 3 次相同调用
        if (history.size() < 3) return false;
        int n = history.size();
        return history.get(n - 1).signature().equals(history.get(n - 2).signature())
                && history.get(n - 2).signature().equals(history.get(n - 3).signature());
    }

    public boolean isHallucinating(String answer, List<String> allowedSources) {
        // 检查引用是否来自允许的来源
        return !allowedSources.stream().anyMatch(answer::contains);
    }
}
```

## 9.2 兜底策略

| 策略 | 说明 |
|------|------|
| 返回部分结果 | 已完成步骤的结果仍有用 |
| 降级到工作流 | 稳定路径兜底 |
| 转人工 | 最终兜底 |
| 明确失败 | 告知用户失败原因，不编造 |

## 9.3 边界与坑

- **反例**：把"部分结果"包装成"完整答案"，等于另一种编造；
- **边界**：检测器本身也会误报（如正常的分页重复调用），需要白名单豁免。

# 10. 失败案例分析框架

## 10.1 五步法

```
1. 复现：能否用相同输入复现？
2. 定位：失败在检索、生成还是工具？
3. 归因：属于哪类失败模式？
4. 修复：改提示、改工具、改控制流？
5. 验证：加评测用例防回归
```

## 10.2 案例示例

以"退款助手重复发起退款"为例：

| 步骤 | 结论 |
|------|------|
| 复现 | 相同工单可复现，第二次请求重复扣款 |
| 定位 | 工具层（createRefund） |
| 归因 | ③ 工具类：重复副作用 |
| 修复 | 加幂等键 + 服务端去重 |
| 验证 | 新增"重复提交"评测用例 |

> 详见 `13_Observability/10_debugging_agents.md`。

# 11. 生产注意事项

## 11.1 风险清单

| 风险 | 建议 |
|------|------|
| 只防不测 | 建失败模式评测集 |
| 无 Trace | 无法定位失败 |
| 无兜底 | 失败时编造答案 |
| 失败静默 | 记录并告警 |
| 不回流 | 失败案例应进入评测集 |

## 11.2 落地建议

- 上线前先跑一遍六类失败模式的"压力清单"；
- 每个失败都应有**检测信号 + 兜底动作**；
- 把线上失败样本自动回流为回归用例。

# 12. 小结

## 12.1 核心要点

- 失败分六类：循环、推理、工具、上下文、系统、成本。
- 死循环、幻觉、目标漂移是最常见的三类。
- 每类失败都有对应的结构性对策。
- 失败必须可检测、可兜底、可回流为评测用例。

## 12.2 关联章节

- 循环控制与终止条件：`4_agent_loop.md`。
- 上下文与记忆管理：`8_Memory/2_context_window_management.md`。
- 失败定位与调试：`13_Observability/10_debugging_agents.md`。
