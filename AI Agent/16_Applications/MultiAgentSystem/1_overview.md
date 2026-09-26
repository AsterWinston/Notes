# 1. 项目背景

**结论先行**：复杂任务（如市场分析、项目管理）需要多种专业能力协作，单个 Agent 难以同时兼顾检索、分析、写作与校验。多智能体系统（Multi-Agent System，MAS）通过角色分工与主管编排，让产出质量优于单 Agent。

## 1.1 问题陈述

### 1.1.1 单 Agent 的局限

| 局限 | 表现 | 后果 |
|------|------|------|
| 上下文拥挤 | 一个提示词塞入所有工具与指令 | 指令互相干扰，遗漏 |
| 角色冲突 | 既要严谨分析又要流畅写作 | 风格摇摆 |
| 难以并行 | 串行执行 | 延迟高 |
| 难以审计 | 只有一条消息链 | 出问题无法定位 |

### 1.1.2 多 Agent 的机会

| 机会 | 说明 |
|------|------|
| 专业化 | 每个 Agent 专注一种能力 |
| 可并行 | 独立子任务并发执行 |
| 可审计 | 每次委托与结果单独记录 |
| 可替换 | 单个 Agent 升级不影响整体 |

### 1.1.3 单 Agent 与多 Agent 示意

```
单 Agent：task → [一个模型 + 全部工具] → answer

多 Agent：task → [Supervisor]
                    ├→ [检索] → 证据
                    ├→ [分析] → 结论
                    └→ [写作] → 初稿
                    → [汇总] → answer
```

## 1.2 定义与术语

| 术语 | 英文 | 含义 |
|------|------|------|
| 主管 | Supervisor | 负责分解、委托、汇总的协调者 |
| 子 Agent | Worker Agent | 执行具体子任务的专家 |
| 共享状态 | Shared State | 跨 Agent 传递的键值上下文 |
| 交接 | Handoff | 从主管到子 Agent 的委托 |
| 轨迹 | Trace | 完整的协作决策与结果记录 |
| 轮次 | Turn | 一次"委托—执行"的完整往返 |

## 1.3 价值主张

- 质量：多视角、多步骤产出更完整。
- 效率：可并行的子任务缩短端到端时间。
- 可控：显式编排优于"黑盒"自主循环。

**边界与坑**：多 Agent 并非总是更好，任务简单时反而增加成本与延迟。

**生产建议**：先用单 Agent 建立基线，再评估多 Agent 的增量收益。

# 2. 目标用户

**结论先行**：目标用户是那些任务复杂、需要多专业协作、且对过程可追溯有要求的角色。

## 2.1 用户画像

| 用户 | 场景 | 关键诉求 | 成功标准 |
|------|------|----------|----------|
| 分析师 | 综合研究报告 | 多来源、有引用 | 报告可直接使用 |
| 项目经理 | 项目规划 | 步骤清晰、可执行 | 计划可落地 |
| 内容团队 | 内容生产流水线 | 多环节、可复用 | 缩短出稿时间 |
| 决策者 | 多视角决策支持 | 观点全面、有对比 | 支持决策 |

## 2.2 典型场景

```java
public record Scenario(
        String id,
        String name,
        List<String> requiredAgents,
        Duration sla,
        BigDecimal costBudget) {}

Scenario market = new Scenario(
        "market-report",
        "AI Agent 市场格局分析",
        List.of("retriever", "analyzer", "writer", "critic"),
        Duration.ofMinutes(5),
        new BigDecimal("2.00"));
```

| 场景 | 子 Agent | SLA | 预算 |
|------|----------|-----|------|
| 市场分析 | retriever/analyzer/writer | 5 分钟 | 2 元 |
| 项目规划 | retriever/planner/critic | 3 分钟 | 1.5 元 |
| 内容流水线 | retriever/writer/editor | 2 分钟 | 1 元 |

## 2.3 使用旅程

```
① 用户提交任务
② 系统分解并展示计划
③ 子 Agent 执行，进度可见
④ 主管汇总产出
⑤ 用户查看引用与轨迹
```

**边界与坑**：用户期望"一键出结果"，但多 Agent 过程较长，需明确进度反馈。

**生产建议**：提供流式进度与中间产物预览。

# 3. 功能范围

**结论先行**：核心是"分解—协作—共享—整合—记录—终止"，同时明确非目标以避免范围蔓延。

## 3.1 核心功能

```
核心功能：
├── 任务分解与分配
├── 多 Agent 协作（Supervisor）
├── 共享状态管理
├── 结果整合
├── 协作轨迹记录
└── 终止与降级
```

| 功能 | 说明 |
|------|------|
| 任务分解 | 把大任务拆成可委托的子任务 |
| 协作 | 主管按能力选择子 Agent |
| 共享状态 | 中间产物集中管理 |
| 结果整合 | 汇总为一致答案 |
| 轨迹记录 | 全过程可回放 |
| 终止降级 | 防止失控 |

## 3.2 非目标

```
非目标：
├── 无限自主
├── 无人工监督
```

## 3.3 功能优先级

| 功能 | 优先级 | 里程碑 |
|------|:------:|:------:|
| 任务分解 | P0 | P0 |
| Supervisor 协作 | P0 | P0 |
| 共享状态 | P1 | P1 |
| 结果整合 | P1 | P1 |
| 轨迹记录 | P1 | P3 |
| 终止降级 | P0 | P1 |

**边界与坑**：不加限制的"自主"会导致成本失控与死循环。

**生产建议**：默认开启步数与成本上限。

# 4. 非功能需求

**结论先行**：多 Agent 的价值必须以"优于单 Agent 基线"为前提，同时受可控、可观测、可靠三项约束。

## 4.1 质量与可控

| 维度 | 要求 | 度量 |
|------|------|------|
| 质量 | 优于单 Agent 基线 | 评测集胜率 |
| 可控 | 步数、成本上限 | 超限率 = 0 |

### 4.1.1 上限的落地

```java
public final class Limits {
    public static final int MAX_STEPS = 8;
    public static final BigDecimal MAX_COST = new BigDecimal("2.00");

    public static void assertWithin(int steps, BigDecimal cost) {
        if (steps > MAX_STEPS) throw new BudgetExceeded("steps");
        if (cost.compareTo(MAX_COST) > 0) throw new BudgetExceeded("cost");
    }
}
```

## 4.2 可观测与可靠

| 维度 | 要求 |
|------|------|
| 可观测 | 完整协作 Trace |
| 可靠 | 终止与降级 |

## 4.3 性能与成本

| 维度 | 目标 |
|------|------|
| 端到端延迟 | < 5 分钟 |
| 单任务成本 | < 2 元 |
| 可用性 | 99.5% |

```java
@ConfigurationProperties(prefix = "mas")
public record MasProperties(
        int maxSteps,
        BigDecimal maxCost,
        Duration timeout,
        int maxConcurrency) {}
```

**边界与坑**：质量提升与成本上升通常正相关，需定义"性价比"阈值。

**生产建议**：为每个任务记录步数、Token、成本，纳入看板。

# 5. 验收标准

**结论先行**：验收既看结果，也看过程，还看安全边界。

## 5.1 功能验收

- [ ] 任务能被正确分解
- [ ] 子 Agent 被正确调用
- [ ] 结果可整合为最终答案

## 5.2 质量验收

- [ ] 质量优于单 Agent 基线
- [ ] 步数与成本在上限内
- [ ] 无死循环

## 5.3 安全验收

- [ ] 协作过程完整可追溯
- [ ] 有终止与降级机制
- [ ] 无越权数据访问

**边界与坑**：基线对比要用同一评测集与评审标准。

**生产建议**：验收用例固化为回归集。

# 6. 技术选型

**结论先行**：编排用图/状态机框架，模型按角色分级，状态可持久化，观测全链路。

## 6.1 编排框架

| 组件 | 选择 | 说明 |
|------|------|------|
| 编排 | LangGraph4j / Spring AI Alibaba Graph | 支持节点、边、条件分支 |
| 状态机 | 自研或框架 | 显式流转 |

| 维度 | 图编排 | 自研循环 |
|------|--------|----------|
| 可观测 | 强 | 需自建 |
| 灵活性 | 中 | 高 |
| 上手成本 | 低 | 高 |

## 6.2 模型与状态

| 组件 | 选择 |
|------|------|
| 模型 | 主管用强模型，子 Agent 按需 |
| 状态 | 共享状态 + 持久化 |

## 6.3 观测与存储

| 组件 | 选择 |
|------|------|
| 观测 | 全链路 Trace |
| 存储 | 轨迹持久化到数据库 |

```java
@Configuration
public class MasConfig {

    @Bean
    public Supervisor supervisor(ChatClient strong) {
        return new LlmSupervisor(strong, "gpt-4o");
    }

    @Bean
    public Map<String, Agent> agents(Retriever r, Analyzer a, Writer w) {
        return Map.of(
                "retriever", new RetrievalAgent(r),
                "analyzer", new AnalyzerAgent(a),
                "writer", new WriterAgent(w));
    }
}
```

**边界与坑**：框架抽象会隐藏 Token 消耗与调用次数，需自行埋点。

**生产建议**：把编排定义与业务逻辑解耦，便于替换框架。

# 7. 里程碑

**结论先行**：按"最小可用 → 完整流程 → 持久化 → 观测评测 → 对比优化"逐步推进。

## 7.1 阶段划分

| 阶段 | 目标 |
|------|------|
| P0 | Supervisor + 2 个子 Agent |
| P1 | 完整协作流程 |
| P2 | 状态持久化 |
| P3 | 观测 + 评测 |
| P4 | 对比基线 + 优化 |

## 7.2 交付物

| 阶段 | 交付物 |
|------|--------|
| P0 | 可跑通的最小闭环 |
| P1 | 终止/降级策略 |
| P2 | 状态与轨迹存储 |
| P3 | 评测报告与 Trace 界面 |
| P4 | 优化后版本 |

## 7.3 退出标准

| 阶段 | 退出标准 |
|------|----------|
| P0 | 2 个子 Agent 协作产出结果 |
| P1 | 完整流程无死循环 |
| P2 | 任务可恢复 |
| P3 | 评测达标 |
| P4 | 优于基线 |

**边界与坑**：过早追求完整功能会拖慢验证，P0 应极简。

**生产建议**：每个阶段设置可量化退出标准。

# 8. 风险

**结论先行**：主要风险是死循环、成本高、难调试、责任扩散、收益不足。

## 8.1 技术风险

| 风险 | 应对 |
|------|------|
| 死循环 | 步数上限 + 循环检测 |
| 难调试 | 完整 Trace |

## 8.2 成本风险

| 风险 | 应对 |
|------|------|
| 成本高 | 限制 Agent 数与轮数 |
| 收益不足 | 对比单 Agent 基线 |

## 8.3 组织风险

| 风险 | 应对 |
|------|------|
| 责任扩散 | 明确职责 |
| 期望过高 | 明确非目标 |

```java
public class RiskGuard {

    public void check(int steps, BigDecimal cost, List<AgentTurn> turns) {
        if (steps >= maxSteps) throw new BudgetExceeded("steps");
        if (cost.compareTo(maxCost) > 0) throw new BudgetExceeded("cost");
        if (loopDetector.hasLoop(turns)) throw new LoopDetected();
    }
}
```

**边界与坑**：循环检测过于敏感会误判正常重试，需区分"重复调用"与"有效重试"。

**生产建议**：为每类风险定义可观测指标与责任人。

# 9. 与单 Agent 的对比

**结论先行**：多 Agent 不是默认选项，只有在任务可分解、需要多能力、且过程需可审计时才更优。

## 9.1 对比维度

| 维度 | 单 Agent | 多 Agent |
|------|----------|----------|
| 质量 | 基线 | 通常更高 |
| 成本 | 低 | 高 |
| 延迟 | 低 | 高（可并行缓解） |
| 可审计 | 弱 | 强 |
| 复杂度 | 低 | 高 |

## 9.2 何时选择多 Agent

- 任务可清晰分解为子任务。
- 子任务需要不同工具或知识。
- 过程需要可追溯与可干预。

## 9.3 决策流程

```
任务是否可分解？
 ├─ 否 → 单 Agent
 └─ 是 → 是否需要多能力？
          ├─ 否 → 单 Agent（多工具）
          └─ 是 → 是否需要审计？
                   ├─ 否 → 单 Agent + 规划
                   └─ 是 → 多 Agent
```

**边界与坑**：为"看起来高级"而引入多 Agent，往往得不偿失。

**生产建议**：用 A/B 对比同一评测集上的质量、成本、延迟。

# 10. 关键设计决策

**结论先行**：以下决策直接影响系统的质量、成本与可维护性，需在 P0 阶段就确定。

## 10.1 编排拓扑

### 10.1.1 集中式 vs 去中心化

| 拓扑 | 优点 | 缺点 |
|------|------|------|
| 集中式（Supervisor） | 可控、易调试 | 主管成瓶颈 |
| 去中心化（Agent 互调） | 灵活 | 难追踪、易失控 |

### 10.1.2 选择建议

早期一律用集中式，等链路稳定后再评估混合式。

## 10.2 状态与记忆

| 方案 | 适用 | 风险 |
|------|------|------|
| 任务内共享状态 | 单次任务协作 | 无 |
| 跨任务长期记忆 | 复用历史经验 | 污染、隐私 |
| 不记忆 | 简单 | 无法积累 |

## 10.3 模型路由

```java
public interface ModelRouter {
    ChatClient route(String agentName);
}

public class TieredRouter implements ModelRouter {
    public ChatClient route(String agentName) {
        return switch (agentName) {
            case "supervisor" -> strong;
            case "writer" -> strong;
            default -> cheap;
        };
    }
}
```

**边界与坑**：全用强模型成本高，全用弱模型质量差，需按角色分级。

**生产建议**：路由策略配置化，支持按任务难度动态调整。

# 11. 小结

- 目标是多 Agent 协作完成复杂任务。
- 核心：分解、协作、共享状态、整合、终止。
- 质量必须优于单 Agent 基线。
- 可控性与可观测性是要求。
- 用 Supervisor 模式 + 完整 Trace。
