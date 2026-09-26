# 1. 一句话区分

MCP 与 A2A 是当下 Agent 生态中最常被同时提及的两个协议。它们看似相近，实际解决的是完全不同的问题。

## 1.1 定位口诀

```
MCP：让 Agent 用上工具（Agent → 工具）
A2A：让 Agent 之间协作（Agent → Agent）
```

## 1.2 为什么容易混淆

- 两者都是"开放协议"，都基于 HTTP/SSE；
- 都由大厂推动，都在讲"互操作"；
- 名字都抽象，初学时容易只记住"协议"二字。

但它们的**连接对象**和**抽象模型**完全不同，混用会导致架构错位。

## 1.3 一张图记住

```
        A2A（水平、平级）
   ┌──────────┐         ┌──────────┐
   │ Agent A  │◀───────▶│ Agent B  │
   └────┬─────┘         └────┬─────┘
        │ MCP                │ MCP（垂直、向下）
        ▼                    ▼
   ┌─────────┐          ┌─────────┐
   │  Tools  │          │  Tools  │
   └─────────┘          └─────────┘
```

# 2. 对比表

## 2.1 总体对比

| 维度 | MCP | A2A |
|------|-----|-----|
| 连接对象 | Agent ↔ 工具/资源 | Agent ↔ Agent |
| 方向 | 垂直（向下） | 水平（平级） |
| 核心抽象 | Tool / Resource / Prompt | Task / Message / Artifact |
| 交互 | 调用-返回 | 任务-状态-产出 |
| 有状态 | 通常无状态 | 有状态（任务） |
| 长任务 | 不适合 | 原生支持 |
| 提出方 | Anthropic | Google |
| 发现机制 | 工具列表 | Agent Card |

## 2.2 抽象模型对比

| 概念层 | MCP 概念 | A2A 概念 | 差异 |
|--------|----------|----------|------|
| 能力描述 | Tool Schema | Agent Card / Skill | 一个是函数签名，一个是服务名片 |
| 输入 | Arguments | Message / Part | 结构化参数 vs 多模态消息 |
| 输出 | Result | Artifact / Part | 单次返回 vs 交付物 |
| 生命周期 | 无 | TaskState | 无状态 vs 有状态 |
| 提示 | Prompt 模板 | 无 | MCP 独有 |

### 2.2.1 能力描述的差异

- MCP 用 **Tool Schema** 描述一个函数：名字、参数、返回。
- A2A 用 **Agent Card / Skill** 描述一个服务：会做什么、支持什么模态。
- 前者是"函数级"，后者是"服务级"；粒度不同，导致发现方式不同。

### 2.2.2 输入输出的差异

| 对比项 | MCP | A2A |
|--------|-----|-----|
| 输入形态 | 结构化 Arguments | 多模态 Message/Part |
| 输出形态 | 单次 Result | 可多轮的 Artifact |
| 是否可多轮 | 否（每次独立） | 是（同一 Task 内） |
| 是否可中断追问 | 否 | 是（INPUT_REQUIRED） |

> 结论：MCP 是"函数调用"的标准化，A2A 是"委派任务"的标准化。

## 2.3 通信与状态对比

| 维度 | MCP | A2A |
|------|-----|-----|
| 传输 | stdio / Streamable HTTP | HTTP / SSE |
| 典型耗时 | 毫秒~秒 | 秒~分钟~小时 |
| 中间状态 | 无 | WORKING / INPUT_REQUIRED |
| 断线恢复 | 重发请求 | 查询 Task 续跑 |
| 取消语义 | 无标准 | CANCELED 状态 |
| 人机协同 | 无 | INPUT_REQUIRED |

# 3. 互补关系

二者不是竞争关系，而是分别占据架构的不同层。

## 3.1 分层架构

```
┌──────────────────────────────────────────────────┐
│  Agent A（Supervisor）                            │
│  ┌────────────┐  A2A   ┌────────────┐            │
│  │ 子 Agent 1 │◀──────▶│ 子 Agent 2 │            │
│  └─────┬──────┘        └──────┬─────┘            │
└────────┼──────────────────────┼──────────────────┘
         │ MCP                  │ MCP
         ▼                      ▼
   ┌──────────┐          ┌──────────┐
   │ 文件工具  │          │ 数据库    │
   └──────────┘          └──────────┘
```

> **A2A 让 Agent 协作，MCP 让每个 Agent 用上工具。**

### 3.1.1 垂直连接与水平连接

```
垂直（MCP）：Agent 依赖工具，方向向下，调用-返回。
水平（A2A）：Agent 协作 Agent，方向平级，任务-状态。
```

- 垂直连接通常是"能力增强"：给 Agent 加手和眼。
- 水平连接通常是"能力组合"：把多个 Agent 组织成团队。

### 3.1.2 一个 Agent 的两条边

任何参与协作的 Agent，通常同时有两条边：

| 边 | 协议 | 作用 |
|----|------|------|
| 向下 | MCP | 接入自己的工具 |
| 向右 | A2A | 与其他 Agent 协作 |

## 3.2 数据流

```
用户请求
  │
  ▼
Supervisor（本地推理）
  │  A2A Task
  ├──────────────▶ 子 Agent 1
  │                    │ MCP call
  │                    ▼
  │                文件系统
  │  Artifact
  │◀──────────────
  ▼
汇总并返回用户
```

## 3.3 Java 中的组合

```java
public final class SupervisorAgent {

    private final A2AClient remoteAgents;   // 跨 Agent：A2A
    private final McpClient localTools;     // 接工具：MCP

    public SupervisorAgent(A2AClient remoteAgents, McpClient localTools) {
        this.remoteAgents = remoteAgents;
        this.localTools = localTools;
    }

    public String handle(String userRequest) {
        // 1) 本地能力能解决就直接用工具
        if (isLocalTask(userRequest)) {
            return localTools.callTool("search_docs", userRequest);
        }
        // 2) 需要跨 Agent 协作则走 A2A
        Task task = remoteAgents.sendMessage(
                new MessageSendParams(Message.user(userRequest), null, null));
        return waitForArtifact(task.id());
    }

    private boolean isLocalTask(String req) {
        return req.contains("查文档");
    }

    private String waitForArtifact(String taskId) {
        // 轮询或订阅推送，返回 Artifact 内容
        throw new UnsupportedOperationException("伪代码：等待任务产出");
    }
}
```

# 4. 典型组合场景

## 4.1 旅行预订场景

```
用户："帮我订一张明天去东京的机票并安排行程"

Supervisor Agent
   │
   ├── A2A → 机票 Agent（负责查询与预订机票）
   │            └── MCP → 航司 API
   │
   └── A2A → 行程 Agent（负责规划行程）
                └── MCP → 地图 API、天气 API
```

## 4.2 代码仓库维护场景

```
用户："修复 issue #42 并提 PR"

Supervisor Agent
   │
   ├── A2A → 代码 Agent
   │            └── MCP → Git Server、文件系统
   │
   └── A2A → 测试 Agent
                └── MCP → CI API、日志系统
```

- 代码 Agent 与测试 Agent 是不同职责的 Agent；
- 各自通过 MCP 接入所需工具；
- Supervisor 通过 A2A 协调二者。

## 4.3 端到端时序

```
用户      Supervisor    机票Agent     航司API
 │            │             │            │
 │─订机票+行程▶│             │            │
 │            │─A2A Task───▶│            │
 │            │             │─MCP call──▶│
 │            │             │◀──价格─────│
 │            │◀─Artifact───│            │
 │            │─A2A Task───▶行程Agent    │
 │            │◀─Artifact───│            │
 │◀─汇总结果──│             │            │
```

# 5. 何时用哪个

## 5.1 选择清单

| 需求 | 选择 |
|------|------|
| 让 Agent 读文件/查数据库 | MCP |
| 调用一个远程 Agent 的能力 | A2A |
| 多 Agent 分工协作 | A2A |
| 接入第三方工具生态 | MCP |
| 跨组织 Agent 互操作 | A2A |

### 5.1.1 判断"工具"还是"Agent"

| 特征 | 工具 | Agent |
|------|------|-------|
| 状态 | 无状态 | 有状态 |
| 步骤 | 单步 | 多步 |
| 时长 | 毫秒~秒 | 秒~小时 |
| 决策 | 无 | 有自主决策 |
| 协议 | MCP | A2A |

### 5.1.2 一个模糊地带的处理

"查询订单状态"这类能力既可做成工具，也可做成 Agent：

- 若只是按订单号查一次 → 工具（MCP）。
- 若需要多轮澄清、跨系统汇总 → Agent（A2A）。

## 5.2 决策表

| 问题 | 是 | 否 |
|------|----|----|
| 目标是"工具"还是"Agent"？ | 工具→MCP | Agent→A2A |
| 是否需要跨进程/跨团队？ | 用协议 | 用本地调用 |
| 任务是否长时、有中间态？ | A2A | MCP 即可 |
| 是否需要能力发现？ | Agent Card(A2A) | 工具列表(MCP) |

## 5.3 常见误用

| 误用 | 后果 | 正解 |
|------|------|------|
| 用 A2A 包一个无状态工具 | 过度设计 | 用 MCP |
| 用 MCP 做长任务编排 | 无状态、难恢复 | 用 A2A |
| 同进程还用 A2A | 网络开销 | 直接调用 |
| 全部自研协议 | 生态割裂 | 用标准协议 |

# 6. 不重叠的部分

## 6.1 能力矩阵

| 能力 | 只有 MCP | 只有 A2A |
|------|:--------:|:--------:|
| 工具发现与调用 | 是 | |
| 资源读取 | 是 | |
| 提示模板 | 是 | |
| 任务生命周期 | | 是 |
| Agent 能力发现 | | 是 |
| 长任务与状态同步 | | 是 |
| 人机协同（INPUT_REQUIRED） | | 是 |

## 6.2 能力解释

### 6.2.1 只有 MCP 的能力

- **资源读取**：`Resource` 原语可直接读文件/数据，不必包装成函数。
- **提示模板**：`Prompt` 原语可下发预设提示，统一交互入口。
- **工具发现**：`tools/list` 动态列出能力，无需预注册。

### 6.2.2 只有 A2A 的能力

- **任务生命周期**：TaskState 让长任务可查询、可取消、可恢复。
- **Agent 能力发现**：Agent Card 描述"会做什么"，而非"有哪些函数"。
- **人机协同**：INPUT_REQUIRED 支持任务中途向用户追问。

# 7. 组合使用建议

## 7.1 四条原则

```
① 工具能力用 MCP 暴露
② Agent 能力用 A2A 暴露
③ 内部工具直接用 @Tool（无需 MCP）
④ 内部子 Agent 用框架编排（无需 A2A）
```

> **协议是为"跨边界"服务的**：跨进程、跨团队、跨组织时才需要。

## 7.2 反模式

| 反模式 | 问题 | 建议 |
|--------|------|------|
| 什么都套协议 | 复杂度爆炸 | 先本地，再边界 |
| 协议层层嵌套 | 调试困难 | 限制跳数 |
| 同一边界内混用 | 语义混乱 | 一个边界一种协议 |
| 忽视鉴权 | 安全漏洞 | 两者都要认证 |

## 7.3 Java 落地示例

```java
// 内部工具：直接注解，零协议开销
public final class LocalTools {

    @Tool("计算两个整数之和")
    public int add(int a, int b) {
        return a + b;
    }
}

// 跨进程工具：暴露为 MCP Server
// 跨组织 Agent：暴露为 A2A Agent
// 二者可同时存在于同一个服务中
```

# 8. 选择决策树

## 8.1 决策树

```
需要让模型调用某个能力？
   │
   ├── 是工具（无状态、单次调用）？
   │      ├── 同进程 → 直接用 @Tool
   │      └── 跨进程/跨团队 → MCP
   │
   └── 是 Agent（有状态、多步、可能长时）？
          ├── 同进程同框架 → 框架编排
          └── 跨进程/跨组织 → A2A
```

## 8.2 边界判断

判断"边界"的四个问题：

| 问题 | 是则跨边界 |
|------|------------|
| 是否跨进程？ | 是 |
| 是否跨团队？ | 是 |
| 是否跨组织/信任域？ | 是 |
| 是否独立部署与升级？ | 是 |

只要有一个"是"，就值得考虑标准协议。

## 8.3 伪代码

```java
public Protocol choose(Target target) {
    if (target.isTool()) {
        return target.isSameProcess() ? Protocol.DIRECT_TOOL : Protocol.MCP;
    }
    // 是 Agent
    if (target.isSameProcess() && target.isSameFramework()) {
        return Protocol.FRAMEWORK_ORCHESTRATION;
    }
    return Protocol.A2A;
}
```

# 9. 生态现状

## 9.1 生态对比

| 协议 | 生态 |
|------|------|
| MCP | 成熟，Server 丰富，主流框架支持 |
| A2A | 发展初期，主要在 Agent 平台间互操作 |

### 9.1.1 MCP 生态

- 官方与社区 Server 数量多，覆盖文件、数据库、Git、浏览器等。
- 主流 Agent 框架普遍内置 MCP Client。
- 工具生态已形成"即插即用"的惯性。

### 9.1.2 A2A 生态

- 处于发展初期，参与者以 Agent 平台与云厂商为主。
- 主要价值在"跨平台 Agent 互操作"。
- 落地案例少于 MCP，规范仍在迭代。

### 9.1.3 组合趋势

- MCP 负责"向下接工具"，A2A 负责"横向接 Agent"。
- 成熟架构常见形态：A2A 编排 + MCP 执行。

> 实践中：**MCP 已广泛落地，A2A 仍在演进**。

## 9.2 选型建议

| 阶段 | 建议 |
|------|------|
| 起步 | 先用 `@Tool`，需要跨进程再上 MCP |
| 成长 | 工具生态统一用 MCP |
| 多 Agent | 先框架编排，跨组织再上 A2A |
| 成熟 | MCP + A2A 组合，统一鉴权与审计 |

# 10. 生产注意事项

## 10.1 架构与复杂度

| 风险 | 建议 |
|------|------|
| 过度使用协议 | 跨边界才用 |
| 混淆两者定位 | 工具用 MCP，Agent 用 A2A |
| 同时引入两套复杂度 | 按需，从简单开始 |

## 10.2 安全

| 风险 | 建议 |
|------|------|
| 无鉴权 | 两者都要鉴权与审计 |
| 权限过大 | 最小权限 + Tool 隔离 |
| 数据外泄 | 出边界前脱敏 |

## 10.3 版本与兼容

| 风险 | 建议 |
|------|------|
| 协议版本漂移 | 锁定协议版本 |
| 能力协商缺失 | 启动时校验能力 |
| 破坏性升级 | 灰度 + 兼容层 |

# 11. 小结

## 11.1 要点回顾

- MCP 连接 Agent 与工具，A2A 连接 Agent 与 Agent。
- 两者互补，可组合使用。
- 协议为"跨边界"服务：跨进程、跨团队、跨组织。
- 同进程场景直接用 `@Tool` 或框架编排即可。
- MCP 已成熟，A2A 仍在演进。

## 11.2 落地检查表

| 检查项 | 通过标准 |
|--------|----------|
| 是否只在跨边界用协议 | 是 |
| 工具是否用 MCP、Agent 是否用 A2A | 是 |
| 是否锁定协议版本 | 是 |
| 是否都有鉴权与审计 | 是 |
| 是否从简单方案起步 | 是 |
