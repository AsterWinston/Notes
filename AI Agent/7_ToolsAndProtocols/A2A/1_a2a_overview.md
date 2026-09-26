# 1. A2A 是什么

**A2A（Agent-to-Agent Protocol）** 是用于 **Agent 之间通信与协作** 的开放协议，由 Google 提出并捐赠给 Linux 基金会。它定义了一套标准的发现、任务、消息与产出模型，让不同框架、不同厂商构建的 Agent 能互相调用。

## 1.1 定义与由来

A2A 的目标不是"让模型更聪明"，而是"让 Agent 能互相找到并协作"。它解决的问题非常具体：

- 一个 Agent 如何发现另一个 Agent 会做什么？
- 一个长任务如何跨 Agent 传递并同步状态？
- 不同组织之间的 Agent 如何安全地互操作？

## 1.2 与其他协议的边界

```
MCP：Agent ↔ 工具
A2A：Agent ↔ Agent
```

> 类比：MCP 是"接外设"，A2A 是"Agent 之间的通用语言"。

## 1.3 一个最小类比

把 A2A 想象成"公司之间的商务合作"：

| 现实世界 | A2A 对应 |
|----------|----------|
| 公司官网上的业务介绍 | Agent Card |
| 一份正式委托单 | Task |
| 沟通往来邮件 | Message |
| 最终交付的成果 | Artifact |
| 项目当前进度 | TaskState |

# 2. 解决的问题

A2A 并非凭空出现，它对应着一批真实存在的集成痛点。

## 2.1 问题清单与方案映射

| 问题 | A2A 的方案 |
|------|------------|
| Agent 由不同框架/厂商构建 | 标准协议通信 |
| 无法互相调用 | 通过 Agent Card 发现能力 |
| 任务状态无法同步 | 标准任务模型 |
| 长任务无法协作 | 异步任务 + 状态查询 |
| 多 Agent 系统难集成 | 标准化互操作 |

## 2.2 为什么不能直接用 HTTP/RPC

有人会问：既然都是 HTTP，为什么还要 A2A？因为裸 RPC 缺少 Agent 协作必需的语义。

| 维度 | 裸 HTTP/RPC | A2A |
|------|-------------|-----|
| 能力发现 | 需人工读文档 | Agent Card 自动发现 |
| 任务语义 | 无 | Task 生命周期 |
| 长任务 | 同步阻塞或自造轮子 | 原生异步 + 推送 |
| 中间态 | 无 | WORKING / INPUT_REQUIRED |
| 产出结构 | 自定义 | Artifact / Part 标准结构 |
| 互操作性 | 每个接口都不同 | 统一协议 |

## 2.3 反例：什么时候不需要 A2A

- 同一个进程内的两个 Agent：直接方法调用或图编排即可。
- Agent 调用一个无状态工具：用 MCP 或直接 `@Tool`。
- 同一框架内的多 Agent：用框架自带的编排能力。

> 引入 A2A 是有成本的（网络、鉴权、状态持久化）。跨边界时才值得。

# 3. 核心概念

A2A 的抽象不多，但每个都承担明确职责。

## 3.1 概念总览

| 概念 | 说明 |
|------|------|
| **Agent Card** | Agent 的"名片"，描述能力与端点 |
| **Task** | 一次任务（有状态、可查询） |
| **Message** | 任务中的消息 |
| **Artifact** | 任务产出（结果） |
| **Part** | 消息/产出的内容片段（文本/文件/数据） |

## 3.2 概念之间的关系

```
Agent Card（发现）
      │
      ▼
Task（一次协作的容器）
  ├── Message（输入/中间对话）
  │     └── Part（text / file / data）
  ├── TaskStatus（状态 + 时间戳）
  └── Artifact（输出）
        └── Part（text / file / data）
```

### 3.2.1 Task 与 Context

- **Task**：一次具体请求，有唯一 `id`，有状态机。
- **Context**：一组相关 Task 的上下文，用于多轮协作；同一 Context 下的 Task 可共享历史。

### 3.2.2 Message、Part 与 Artifact

- **Message** 是交互的载体，`role` 区分 `user`（客户端）与 `agent`。
- **Part** 是内容的最小单元，同一 Message 可混合多种 Part。
- **Artifact** 是任务产物，与 Message 的区别在于它是"交付物"而非"对话"。

# 4. Agent Card

Agent Card 是 A2A 的入口：客户端先取名片，才知道该发什么任务、发到哪里。

## 4.1 字段详解

```json
{
  "name": "Travel Agent",
  "description": "提供旅行规划服务",
  "url": "https://travel-agent.example.com/a2a",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "plan-trip",
      "name": "旅行规划",
      "description": "根据目的地与预算规划行程",
      "examples": ["帮我规划一个 3 天的东京行程"]
    }
  ],
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text", "file"]
}
```

| 字段 | 含义 | 作用 |
|------|------|------|
| `name` | Agent 名称 | 人类可读标识 |
| `url` | 端点地址 | 任务发送目标 |
| `version` | 协议/服务版本 | 兼容性判断 |
| `capabilities.streaming` | 是否支持 SSE | 决定能否流式调用 |
| `capabilities.pushNotifications` | 是否支持推送 | 决定能否用 Webhook |
| `skills` | 技能列表 | 能力发现的核心 |
| `defaultInputModes` | 默认输入类型 | 客户端据此构造 Part |
| `defaultOutputModes` | 默认输出类型 | 客户端据此解析 Artifact |

## 4.2 发现机制

> Agent Card 发布在 `/.well-known/agent-card.json`，供其他 Agent 发现。

```
客户端                         服务端 Agent
   │  GET /.well-known/agent-card.json
   │──────────────────────────────────▶
   │  AgentCard (JSON)
   │◀──────────────────────────────────
   │  选择 skill → 构造 Task
   │──────────────────────────────────▶
```

## 4.3 Java 建模示例

### 4.3.1 记录类型定义

```java
import java.util.List;

public record AgentCard(
        String name,
        String description,
        String url,
        String version,
        Capabilities capabilities,
        List<Skill> skills,
        List<String> defaultInputModes,
        List<String> defaultOutputModes) {

    public record Capabilities(boolean streaming, boolean pushNotifications) {
    }

    public record Skill(String id, String name, String description, List<String> examples) {
    }
}
```

### 4.3.2 解析与校验

```java
import com.fasterxml.jackson.databind.ObjectMapper;

public final class AgentCardLoader {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    public static AgentCard load(String json) throws Exception {
        AgentCard card = MAPPER.readValue(json, AgentCard.class);
        if (card.url() == null || card.skills() == null || card.skills().isEmpty()) {
            throw new IllegalArgumentException("Agent Card 缺少 url 或 skills");
        }
        return card;
    }

    public static boolean supportsStreaming(AgentCard card) {
        return card.capabilities() != null && card.capabilities().streaming();
    }
}
```

# 5. 任务模型

Task 是 A2A 的核心：它把"一次协作"变成一个可查询、可恢复的状态对象。

## 5.1 状态机

```
Task 状态：
SUBMITTED → WORKING → COMPLETED
                    ├──▶ FAILED
                    ├──▶ CANCELED
                    └──▶ INPUT_REQUIRED（需要更多输入）
```

```java
public enum TaskState {
    SUBMITTED,        // 已提交
    WORKING,          // 处理中
    INPUT_REQUIRED,   // 需要输入
    COMPLETED,        // 完成
    FAILED,           // 失败
    CANCELED,         // 已取消
    REJECTED          // 被拒绝
}
```

## 5.2 状态迁移规则

| 当前状态 | 可迁移到 | 说明 |
|----------|----------|------|
| SUBMITTED | WORKING / REJECTED | 接收或被拒 |
| WORKING | COMPLETED / FAILED / CANCELED / INPUT_REQUIRED | 执行中的分支 |
| INPUT_REQUIRED | WORKING / CANCELED | 补充输入后继续 |
| COMPLETED | 终态 | 不可再变 |
| FAILED | 终态 | 可携带错误信息 |
| CANCELED | 终态 | 客户端主动取消 |
| REJECTED | 终态 | 服务端拒绝受理 |

## 5.3 Java 实现

用密封接口建模状态，保证穷尽处理。

```java
public sealed interface TaskStatus
        permits TaskStatus.Done, TaskStatus.Running, TaskStatus.Waiting {

    record Done(boolean success, String detail) implements TaskStatus {
    }

    record Running(int progress) implements TaskStatus {
    }

    record Waiting(String question) implements TaskStatus {
    }
}

public final class TaskStatusMapper {

    public static TaskState toState(TaskStatus status) {
        return switch (status) {
            case TaskStatus.Done d -> d.success() ? TaskState.COMPLETED : TaskState.FAILED;
            case TaskStatus.Running r -> TaskState.WORKING;
            case TaskStatus.Waiting w -> TaskState.INPUT_REQUIRED;
        };
    }
}
```

# 6. 交互方式

A2A 支持三种交互方式，覆盖短任务、长任务与实时反馈。

## 6.1 三种交互模式

| 方式 | 说明 |
|------|------|
| 请求-响应 | 提交任务 → 返回任务 ID → 轮询状态 |
| 流式（SSE） | 任务执行中推送状态与消息 |
| 推送通知 | 完成后回调客户端 webhook |

```
短任务：sendMessage  ──▶ 直接返回 COMPLETED
长任务：sendMessage  ──▶ 返回 taskId ──▶ getTask 轮询
实时  ：sendMessageStreaming ──▶ 事件流
离线  ：sendMessage + pushNotification ──▶ webhook 回调
```

## 6.2 请求-响应与轮询

```java
// ① 提交任务
Task task = client.sendMessage(new MessageSendParams(
        message, taskId, contextId));
// 返回 task.id()，状态 SUBMITTED/WORKING

// ② 查询状态
Task status = client.getTask(task.id());
```

## 6.3 流式与推送

### 6.3.1 SSE 流式

```java
// ③ 或流式接收
client.sendMessageStreaming(params, event -> {
    if (event instanceof TaskStatusUpdateEvent update) {
        System.out.println(update.status().state());
    }
});
```

### 6.3.2 Webhook 推送

当 `capabilities.pushNotifications = true` 时，客户端可注册回调地址：

```java
public record PushNotificationConfig(
        String url,        // 客户端 webhook
        String token,      // 校验令牌
        String taskId) {
}

// 注册推送
client.setTaskPushNotification(new PushNotificationConfig(
        "https://my-app.example.com/a2a/callback",
        "secret-token",
        task.id()));
```

> 服务端在任务状态变化时向 `url` POST 事件；客户端必须校验 `token` 并做幂等处理。

## 6.4 完整 Java 客户端示例

```java
import java.time.Duration;

public final class TravelClientDemo {

    public static void main(String[] args) throws Exception {
        A2AClient client = A2AClient.connect("https://travel-agent.example.com");

        AgentCard card = client.agentCard();
        if (!AgentCardLoader.supportsStreaming(card)) {
            throw new IllegalStateException("目标 Agent 不支持流式");
        }

        Message message = Message.user("帮我规划一个 3 天的东京行程");

        // 流式执行
        client.sendMessageStreaming(new MessageSendParams(message, null, null), event -> {
            switch (event) {
                case TaskStatusUpdateEvent u ->
                        System.out.println("[状态] " + u.status().state());
                case TaskArtifactUpdateEvent a ->
                        System.out.println("[产出] " + a.artifact().name());
                default -> { /* 忽略其他事件 */ }
            }
        });

        // 若为长任务，可轮询兜底
        String taskId = "task-123";
        long deadline = System.currentTimeMillis() + Duration.ofMinutes(5).toMillis();
        while (System.currentTimeMillis() < deadline) {
            Task t = client.getTask(taskId);
            if (t.state() == TaskState.COMPLETED) {
                break;
            }
            Thread.sleep(1000);
        }
    }
}
```

# 7. 与 MCP 的分工

A2A 与 MCP 经常被拿来比较，但二者是不同层次的协议。

## 7.1 分层视图

```
┌─────────────┐   A2A   ┌─────────────┐
│  Agent A    │◀───────▶│  Agent B    │
└──────┬──────┘         └──────┬──────┘
       │ MCP                   │ MCP
       ▼                       ▼
  ┌─────────┐            ┌─────────┐
  │  Tools  │            │  Tools  │
  └─────────┘            └─────────┘
```

## 7.2 对比

| 协议 | 连接 | 说明 |
|------|------|------|
| MCP | Agent ↔ 工具/资源 | 垂直（向下） |
| A2A | Agent ↔ Agent | 水平（平级） |

## 7.3 协同模式

> 两者互补，不是替代关系。详见 `2_a2a_vs_mcp.md`。

- 一个 Supervisor Agent 用 A2A 调用子 Agent；
- 每个子 Agent 用 MCP 接入自己的工具；
- 边界之内用框架编排，边界之外才用协议。

# 8. 适用场景

## 8.1 典型场景

| 场景 | 说明 |
|------|------|
| 跨组织协作 | 不同公司的 Agent 协作 |
| 多 Agent 系统 | 专业 Agent 分工 |
| 能力复用 | 调用第三方 Agent 能力 |
| 异构框架 | 不同框架构建的 Agent 互操作 |

## 8.2 场景拆解示例

以"企业报销 Agent 调用外部差旅 Agent"为例：

```
报销 Agent（本公司）                   差旅 Agent（服务商）
      │  A2A: Task(skill=book-hotel)
      │──────────────────────────────────▶
      │  状态 WORKING / INPUT_REQUIRED
      │◀──────────────────────────────────
      │  A2A: Message(补充预算)
      │──────────────────────────────────▶
      │  Artifact: 预订单 PDF
      │◀──────────────────────────────────
```

- 双方不需要了解对方内部实现；
- 通过 Agent Card 发现 `book-hotel` 技能；
- 通过 Task 状态同步进度，通过 Artifact 交付结果。

# 9. 不适用场景

## 9.1 不适用清单

| 场景 | 更合适方案 |
|------|------------|
| 单进程内多 Agent | 直接方法调用/图编排 |
| Agent 调用工具 | MCP |
| 同框架内协作 | 框架自带编排 |

## 9.2 反例分析

- **反例一**：同进程两个函数式 Agent 用 A2A 通信 → 平白引入网络与序列化开销。
- **反例二**：把一个查询天气的无状态工具包装成 A2A Agent → 应直接用 MCP 或 `@Tool`。
- **反例三**：所有子 Agent 都在同一框架内，却坚持用 A2A → 框架编排更简单、更易调试。

# 10. 生产注意事项

## 10.1 安全与信任

| 风险 | 建议 |
|------|------|
| 信任外部 Agent | 来源白名单 |
| 无鉴权 | 双向认证 |
| 数据越权 | 按任务最小授权 |

## 10.2 可靠性与可观测性

| 风险 | 建议 |
|------|------|
| 任务状态丢失 | 持久化任务 |
| 长任务超时 | 异步 + 推送 |
| 无审计 | 记录任务全流程 |
| 重复回调 | 幂等 + 去重 |

## 10.3 成本与限流

| 风险 | 建议 |
|------|------|
| 成本不可控 | 预算与限流 |
| 恶意刷任务 | 按来源限流 |
| 无限重试 | 退避 + 上限 |

# 11. 小结

## 11.1 要点回顾

- A2A 是 Agent 之间通信与协作的开放协议。
- 核心概念：Agent Card、Task、Message、Artifact。
- 任务有状态，支持轮询、流式、推送。
- MCP 连接 Agent 与工具，A2A 连接 Agent 与 Agent。
- 适合跨组织、跨框架的 Agent 互操作。

## 11.2 延伸阅读

- 下一节：`2_a2a_vs_mcp.md` 深入对比 A2A 与 MCP。
- 后续：`3_java_sdk.md` 用 Java 实现 A2A Agent 与客户端。
