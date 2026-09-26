# 1. 概述与定位

## 1.1 什么是 Swarm 模式

结论先行：Swarm（蜂群）模式让多个 Agent 彼此自主交接（handoff，交接），没有中心主管；当前 Agent 判断"这个问题该由谁处理"，直接把控制权与消息交给下一个 Agent，直到某个 Agent 决定自己给出最终答复。

机制拆解：

1. 系统只有一个入口 Agent 和一个 Agent 注册表。
2. 每个 Agent 接收 `(input, context)`，返回一个 `Handoff`。
3. `Handoff.nextAgent == null` 表示"我来收尾"，流程结束。
4. 否则控制权转移到下一个 Agent，消息与上下文随之传递。

```
Agent 间自主交接，无中心主管：

  Agent A ──"这个问题需要 B"──▶ Agent B ──▶ Agent C
```

```
用户 ──▶ Agent A
            │ 判断：需要 B 的专业能力
            ▼
         Agent B
            │ 判断：需要 C
            ▼
         Agent C
            │ 判断：可以回答
            ▼
          输出
```

边界与坑：Swarm 的"灵活"来自每个 Agent 都有决策权，但这也意味着没有任何单点能保证全局目标被达成。

生产建议：仅在任务开放、边界模糊时使用；否则优先 Supervisor。

## 1.2 与 Supervisor 的定位差异

结论先行：Supervisor 是中心化编排，Swarm 是去中心化交接，二者在决策者、可追踪性、循环风险上正好相反。

| 维度 | Supervisor | Swarm |
|------|-----------|-------|
| 决策者 | 主管 | 各 Agent |
| 中心化 | 是 | 否 |
| 可追踪 | 易 | 难 |
| 循环风险 | 低 | 高 |
| 单点瓶颈 | 有 | 无 |
| 适用 | 可分解任务 | 开放任务 |

机制拆解：Supervisor 每轮多一次"主管调用"，换来全局视图；Swarm 省掉这次调用，但每个 Agent 都要自行判断去向。

边界与坑：Swarm 的"无单点瓶颈"是双刃剑——出问题时也没有单点能兜底。

生产建议：需要审计与严格控制的场景不要用 Swarm，或与 Supervisor 结合使用。

## 1.3 适用与不适用场景

### 1.3.1 适用场景

结论先行：任务步骤无法预先确定、需要 Agent 依据内容自行决定后续处理者时，Swarm 更自然。

- 客服分流：订单、退款、物流问题互相转接。
- 开放式问答：不知道需要哪种专家，逐层定位。
- 探索型任务：路径取决于中间结果。

### 1.3.2 不适用场景

结论先行：任务需要强审计、强预算控制、或步骤固定时，Swarm 不合适。

- 金融、医疗等合规要求高的场景。
- 成本敏感、需要精确预算的场景。
- 步骤固定的流水线任务。

边界与坑：把本可固定流程的任务做成 Swarm，只会增加不确定性与调试成本。

生产建议：先用规则或 Pipeline 兜底，确认确需动态交接再引入 Swarm。

# 2. 结构与交接机制

## 2.1 拓扑结构

结论先行：Swarm 是网状拓扑，任意 Agent 之间都可能发生交接，通信对数量理论上随 Agent 数平方增长。

机制拆解：

1. 每个 Agent 持有"可交接对象"的知识（名称或能力描述）。
2. 交接是单向的：交出控制权后，原 Agent 不再参与本轮。
3. 因此同一时刻只有一个 Agent 活跃，避免了并发写冲突。

```
        ┌─────┐        ┌─────┐
        │  A  │───────▶│  B  │
        └──┬──┘        └──┬──┘
           │  ▲           │
           ▼  │           ▼
        ┌─────┐        ┌─────┐
        │  C  │◀──────▶│  D  │
        └─────┘        └─────┘
     任意节点可指向任意节点（网状）
```

边界与坑：网状不等于"每个 Agent 都知道所有 Agent"，通常只暴露有限的可交接集合。

生产建议：用白名单限制每个 Agent 可交接的对象，收敛搜索空间。

## 2.2 交接协议

结论先行：交接协议的核心是一个不可变的 `Handoff` 记录，字段为下一个 Agent、消息、上下文。

```java
public record Handoff(
        String nextAgent,          // 下一个 Agent（null 表示结束）
        String message,            // 传递的消息
        Map<String, Object> context
) {}
```

```java
interface Agent {
    Handoff handle(String input, Map<String, Object> context);
}
```

机制拆解：`nextAgent == null` 是唯一的终止信号；`message` 承载"给下一个 Agent 的输入"；`context` 承载跨 Agent 的结构化数据。

边界与坑：若 Agent 既返回 `nextAgent` 又返回终答，语义会混乱；约定"有 nextAgent 时 message 是转交说明"。

生产建议：在协议层校验 `nextAgent` 必须在注册表内，非法值直接拒绝。

## 2.3 上下文传递

### 2.3.1 显式上下文

结论先行：把关键数据放进 `context` 的显式键值，避免依赖自然语言复述。

```java
Map<String, Object> ctx = new HashMap<>();
ctx.put("orderId", "A-1001");
ctx.put("amount", 299.0);
return new Handoff("refund-agent", "请处理退款", ctx);
```

机制拆解：显式上下文可被下一个 Agent 直接读取，减少信息损失。

边界与坑：上下文键名不统一会造成混乱，需要预先约定 schema。

### 2.3.2 隐式上下文

结论先行：仅靠 `message` 文本传递信息，最简单但最容易丢信息。

边界与坑：多跳之后，最初的关键约束往往在复述中丢失。

生产建议：至少把"原始目标"和"硬约束"放进显式上下文，其余可走文本。

# 3. 完整 Java 实现

## 3.1 核心接口

结论先行：先定义 `Agent` 与 `Handoff`，再实现编排器与具体 Agent。

```java
public interface Agent {
    String name();
    Handoff handle(String input, Map<String, Object> context);
}
```

机制拆解：`handle` 是唯一入口，返回 `Handoff` 统一表达"继续"或"结束"。

边界与坑：接口不暴露 `description`，导致可交接集合只能硬编码；可额外加 `canHandle` 方法。

生产建议：为 Agent 增加 `Set<String> peers()` 声明可交接对象。

## 3.2 编排器

结论先行：编排器是一个带循环检测的 while 循环，负责推进控制权并兜底终止。

```java
public class SwarmOrchestrator {

    private final Map<String, Agent> agents;
    private final String startAgent;
    private final Logger log = LoggerFactory.getLogger(getClass());

    public SwarmOrchestrator(List<Agent> agentList, String startAgent) {
        this.agents = agentList.stream()
                .collect(Collectors.toMap(Agent::name, a -> a));
        this.startAgent = startAgent;
    }

    public String run(String task, int maxSteps) {
        String current = startAgent;
        String input = task;
        Map<String, Object> context = new HashMap<>();
        List<String> path = new ArrayList<>();

        for (int step = 0; step < maxSteps; step++) {
            Agent agent = agents.get(current);
            if (agent == null) {
                return "未知 Agent：" + current;
            }
            path.add(current);

            Handoff handoff = agent.handle(input, context);

            if (handoff.nextAgent() == null) {
                return handoff.message();          // 结束
            }
            if (!agents.containsKey(handoff.nextAgent())) {
                return "非法交接目标：" + handoff.nextAgent();
            }
            if (path.contains(handoff.nextAgent())) {
                log.warn("检测到回环：{}，路径 {}", handoff.nextAgent(), path);
            }

            current = handoff.nextAgent();
            input = handoff.message();
            context.putAll(handoff.context());
        }
        return "达到最大步数";
    }
}
```

机制拆解：`path` 记录访问顺序用于回环检测；`context` 逐跳合并。

边界与坑：`context.putAll` 会覆盖同名键，需明确"后来者覆盖"还是"保留先到者"。

生产建议：把 `path` 一并写进 Trace，便于事后还原交接链。

## 3.3 具体 Agent 示例

结论先行：每个 Agent 的职责是"能处理就处理，不能处理就精确转交"。

```java
@Component
public class OrderAgent implements Agent {

    private final OrderService orderService;

    public OrderAgent(OrderService orderService) {
        this.orderService = orderService;
    }

    @Override
    public String name() {
        return "order-agent";
    }

    @Override
    public Handoff handle(String input, Map<String, Object> context) {
        if (input.contains("退款")) {
            return new Handoff("refund-agent",
                    "用户需要退款：" + input, context);
        }
        if (input.contains("物流")) {
            return new Handoff("logistics-agent",
                    "用户询问物流：" + input, context);
        }
        String result = orderService.query(input);
        return new Handoff(null, result, context);   // 结束
    }
}
```

机制拆解：关键词判断是初版策略，生产可用小模型做意图分类。

边界与坑：关键词"退款"若出现在"我要取消退款"中会误转，需要更细的判定。

生产建议：为每个 Agent 增加"兜底答复"，避免无匹配时卡住。

## 3.4 可运行示例

结论先行：用少量 Agent 串起一条可运行的交接链，验证编排器行为。

```java
public class SwarmMain {
    public static void main(String[] args) {
        List<Agent> agents = List.of(
                new OrderAgent(new OrderService()),
                new RefundAgent(new RefundService()),
                new LogisticsAgent(new LogisticsService()));

        var swarm = new SwarmOrchestrator(agents, "order-agent");
        String answer = swarm.run("我要查询订单 A-1001 的物流", 6);
        System.out.println(answer);
    }
}
```

边界与坑：若入口 Agent 名称写错，会直接返回"未知 Agent"，需在启动时校验。

生产建议：启动阶段做一次"交接图可达性"检查，确保所有 Agent 从入口可达。

# 4. 循环检测与终止

## 4.1 路径去重

结论先行：记录完整路径，一旦下一个目标已在路径中出现，说明可能形成回环。

```java
public boolean isLoop(List<String> path, String next) {
    return path.contains(next);
}
```

机制拆解：路径去重是 O(n) 判断，n 为步数，成本可忽略。

边界与坑：正常的"二次访问"未必是错误（如先查后改），直接中断可能误伤。

生产建议：把回环作为告警而非硬中断，再结合次数阈值决定是否终止。

## 4.2 计数阈值

结论先行：允许有限次重复，超过阈值才判定为死循环。

```java
public boolean isLoop(List<String> path, String next, int threshold) {
    long count = path.stream().filter(next::equals).count();
    return count >= threshold;
}
```

机制拆解：`threshold` 通常取 2 或 3，兼顾容错与止损。

| 阈值 | 行为 | 适用 |
|:----:|------|------|
| 1 | 首次重复即终止 | 强约束场景 |
| 2 | 允许一次重试 | 一般场景 |
| 3 | 允许两次重试 | 宽松探索 |

边界与坑：阈值过大等于没有保护，过小会误杀正常重试。

生产建议：阈值做成配置，按业务容忍度调整。

## 4.3 强制终止

结论先行：无论如何都要有"最后兜底"，在超限时给出结论而非空转。

```java
if (step == maxSteps - 1) {
    // 最后一步强制收尾
    return agent.handle(input, context).message();
}
```

机制拆解：强制收尾可要求当前 Agent 直接作答，或调用一个汇总模型。

边界与坑：强制收尾可能得到低质量答案，但好过没有答案。

生产建议：在返回值中标注"因步数上限强制结束"，让调用方知情。

# 5. 关键优化

## 5.1 交接白名单

结论先行：限制每个 Agent 可交接的对象集合，可显著降低回环与无效跳转。

```java
private static final Map<String, Set<String>> PEERS = Map.of(
        "order-agent", Set.of("refund-agent", "logistics-agent"),
        "refund-agent", Set.of("order-agent"),
        "logistics-agent", Set.of());
```

机制拆解：白名单把"网状"裁剪为"有向稀疏图"，天然抑制循环。

边界与坑：白名单过严会导致"无处可去"，需要在最后一步提供兜底。

生产建议：用白名单 + 兜底答复组合，兼顾可控与可用。

## 5.2 交接预算

结论先行：为每次任务设定最大交接次数与 token 预算，超限即停。

| 预算项 | 建议值 | 说明 |
|--------|:------:|------|
| 最大交接次数 | ≤ 6 | 防止长链 |
| 单 Agent 重试 | ≤ 1 | 防止空转 |
| 总 token | 按任务设 | 熔断降级 |

边界与坑：预算太紧会让复杂任务被迫中断，太松则失去意义。

生产建议：按业务分档，并对超限事件告警。

## 5.3 交接链追踪

结论先行：把完整交接链（谁交给谁、为什么）记录下来，是 Swarm 可维护的前提。

```java
record Hop(int step, String from, String to, String reason, long costMs) {}
```

机制拆解：`Hop` 序列可还原决策路径，用于事后分析与训练数据积累。

边界与坑：只记结果不记理由，事后无法判断交接是否合理。

生产建议：把 `Hop` 序列写入结构化日志，并接入链路追踪系统。

# 6. 边界与失败模式

## 6.1 踢皮球

结论先行：Agent A 认为该 B 处理，B 认为该 A 处理，形成无限互推。

```
Agent A → "这个应该 B 处理" → Agent B
Agent B → "这个应该 A 处理" → Agent A
        ↓ 无限循环
```

对策：

- 路径去重 + 计数阈值 + 步数上限三重保护。
- 在提示词中写明"什么不该你处理"和"不得直接推诿"。
- 设置"最后兜底 Agent"强制给出结论。

边界与坑：仅靠步数上限会让循环跑满预算，浪费成本。

生产建议：对高频回环对做专项分析，调整职责边界。

## 6.2 上下文丢失

结论先行：多跳传递时，A 的结论到 C 手里可能已经失真或缺失。

机制拆解：每跳都做一次自然语言复述，复述即信息压缩，压缩即损失。

对策：

```java
public record HandoffContext(
        String originalGoal,       // 原始目标，永不改写
        String summary,            // 当前摘要
        Map<String, Object> data,  // 关键数据
        List<String> constraints   // 硬约束
) {}
```

边界与坑：原始目标若被某个 Agent 改写，后续会一路偏航。

生产建议：`originalGoal` 设为只读，任何 Agent 都不得修改。

## 6.3 责任扩散

结论先行：每个 Agent 都认为"这不归我管"，结果是没人处理。

对策：

- 明确每个 Agent 的"负责范围"与"不负责范围"。
- 要求 Agent 在无法处理时给出"最佳判断 + 建议转交对象"，禁止空手转交。
- 设置默认兜底 Agent 承接无人认领的问题。

边界与坑：强制兜底 Agent 会增加其负载，需评估容量。

生产建议：统计各 Agent 的"接收-处理-转交"比例，识别扩散热点。

# 7. 生产注意事项

## 7.1 可观测性

结论先行：Swarm 比 Supervisor 更难追踪，因此对观测的要求更高。

| 字段 | 说明 |
|------|------|
| traceId | 贯穿整条任务 |
| hopIndex | 交接序号 |
| from/to | 交接双方 |
| reason | 交接理由 |
| latencyMs | 单跳耗时 |
| tokens | 单跳消耗 |

生产建议：以交接图形式可视化高频路径，快速发现异常环。

## 7.2 成本控制

结论先行：Swarm 的轮数不可预知，成本天然比固定流程更难控。

- 硬性设置最大交接次数。
- 简单 Agent 用小模型。
- 对重复出现的输入做缓存。
- 超预算时转为直接兜底答复。

边界与坑：不设上限的 Swarm 在极端输入下可能长时间运行。

生产建议：为每次任务设置 wall-clock 超时，与步数上限并行生效。

## 7.3 安全与权限

结论先行：去中心化意味着没有单一关口，权限必须下沉到每个 Agent。

- 每个 Agent 只授予其职责所需的最小工具集。
- 交接目标做白名单校验，防止被诱导跳转到高权限 Agent。
- 对跨 Agent 传递的内容做注入检测。

边界与坑：提示注入可能让某 Agent 把请求转给高权限 Agent，形成提权路径。

生产建议：在交接边界上做权限再校验，而非只信任来源 Agent。

# 8. 小结与落地清单

## 8.1 要点回顾

- Swarm 让 Agent 间自主交接，无中心主管。
- 灵活但难追踪、易循环。
- 必须做循环检测与步数上限。
- 适合开放任务，不适合需严格控制的场景。
- 生产环境慎用，或与 Supervisor 结合。

## 8.2 落地清单

| 项目 | 检查点 |
|------|--------|
| 交接协议 | nextAgent 合法性校验 |
| 终止 | 明确终止条件 + 强制收尾 |
| 循环 | 路径去重 + 阈值 + 步数上限 |
| 上下文 | originalGoal 只读，结构化传递 |
| 可观测 | 完整交接链 Trace |
| 安全 | 白名单 + 最小权限 + 注入检测 |
