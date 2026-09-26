# 1. SK 的 Agent 抽象

**结论先行**：Semantic Kernel（简称 SK）把 Agent（智能体）抽象为"**模型 + 指令 + 插件 + 线程**"四要素的组合，并提供单 Agent 与多 Agent 两类入口。

```
Agent = 模型 + 指令 + 插件 + 线程（会话状态）
```

## 1.1 一句话定义与边界

Agent 是"**能自主调用工具、并在多轮对话中保持状态的模型封装**"。与单纯的 Chat Completion（对话补全）相比，Agent 多了两件事：**工具调用**与**会话记忆**。

边界：SK 的 Agent 不做长期记忆、不做任务持久化调度，这些需要应用层补齐。

## 1.2 机制拆解

### 1.2.1 四要素

| 要素 | 作用 | 对应字段 |
|------|------|----------|
| 模型 | 推理与生成 | `ChatCompletionService` |
| 指令 | 角色与行为约束 | `instructions` |
| 插件 | 可调用的工具集 | `kernel` / `plugins` |
| 线程 | 会话状态容器 | `AgentThread` |

### 1.2.2 一次调用的时序

```
调用方            Agent            模型           插件
  │ invokeAsync ──▶ │               │              │
  │                 │ 组装提示 ─────▶│              │
  │                 │◀── 工具调用请求 │              │
  │                 │ 执行函数 ─────────────────────▶│
  │                 │◀───────────────────── 函数结果 │
  │                 │ 回填结果 ─────▶│              │
  │◀── 最终消息 ────│◀── 生成回复 ───│              │
```

关键点：**工具调用对调用方透明**，Agent 内部完成"请求工具 → 执行 → 回填"的循环。

## 1.3 概念对照表

| 概念 | 说明 |
|------|------|
| `ChatCompletionAgent` | 基础对话 Agent |
| `AgentThread` | 会话状态容器 |
| `AgentGroupChat` | 多 Agent 群聊 |

补充概念：

| 概念 | 说明 |
|------|------|
| `ChatMessageContent` | 单条消息 |
| `AuthorRole` | 消息角色（USER/ASSISTANT/...） |
| `AgentGroupChatSettings` | 群聊设置 |

## 1.4 边界与坑

- Agent 不是"万能编排器"，复杂流程仍需 Graph。
- 线程默认在内存，进程重启即丢。
- 工具调用次数不设上限会导致无限循环。

## 1.5 生产建议

- 明确 Agent 的职责边界，一个 Agent 只做一件事。
- 线程状态外置到 Redis（远程字典服务）或数据库。
- 为工具调用设步数上限。

# 2. 基础 Agent

**结论先行**：`ChatCompletionAgent` 是最小可用 Agent，通过 Builder（建造者）装配名称、指令、Kernel 与参数。

## 2.1 构建 Agent

### 2.1.1 Builder 各字段

| 字段 | 必填 | 说明 |
|------|:----:|------|
| `name` | 否 | Agent 名称，多 Agent 时用于区分 |
| `instructions` | 是 | 系统指令 |
| `kernel` | 是 | 提供模型与插件 |
| `arguments` | 否 | 默认参数与执行设置 |

### 2.1.2 指令设计

指令即系统提示（System Prompt），建议包含：

1. 角色定位。
2. 输出格式约束。
3. 可用工具的说明。
4. 拒绝策略（越界请求如何处理）。

## 2.2 基础 Agent 示例

```java
ChatCompletionAgent agent = ChatCompletionAgent.builder()
        .name("Assistant")
        .instructions("你是一个专业的中文助手。")
        .kernel(kernel)
        .arguments(new KernelArguments(settings))
        .build();

AgentThread thread = new ChatHistoryAgentThread();

// 调用
ChatMessageContent response = agent.invokeAsync(
        "你好", thread).block().get(0);
```

## 2.3 调用与返回

### 2.3.1 invokeAsync

`invokeAsync` 返回一个异步集合（`Mono<List<ChatMessageContent>>` 语义），因为一次调用可能产生多条消息（含工具调用消息）。

### 2.3.2 结果结构

| 字段 | 说明 |
|------|------|
| `getContent()` | 文本内容 |
| `getAuthorRole()` | 角色 |
| `getModelId()` | 模型标识 |
| `getMetadata()` | 扩展元数据 |

## 2.4 完整示例

```java
public class BasicAgentDemo {

    public static void main(String[] args) {
        ChatCompletionService chat = OpenAIChatCompletion.builder()
                .withOpenAIAsyncClient(client)
                .withModelId("gpt-4o")
                .build();

        Kernel kernel = Kernel.builder()
                .withAIService(ChatCompletionService.class, chat)
                .withPlugin(KernelPluginFactory.createFromObject(
                        new WeatherPlugin(), "weather"))
                .build();

        ChatCompletionAgent agent = ChatCompletionAgent.builder()
                .name("WeatherAssistant")
                .instructions("""
                        你是天气助手。用户询问天气时调用 weather 插件。
                        不确定城市时先向用户澄清。
                        """)
                .kernel(kernel)
                .build();

        AgentThread thread = new ChatHistoryAgentThread();

        ChatMessageContent r1 = agent.invokeAsync(
                "北京今天天气怎么样？", thread).block().get(0);
        System.out.println(r1.getContent());

        ChatMessageContent r2 = agent.invokeAsync(
                "那明天呢？", thread).block().get(0);
        System.out.println(r2.getContent());
    }
}
```

## 2.5 边界与坑

- 指令过长会挤占上下文预算。
- 未指定 `name` 时多 Agent 场景难以区分消息来源。
- 工具抛出的异常若未处理，会中断整个 Agent 调用。

## 2.6 生产建议

- 把指令作为可配置项，支持热更新。
- 对工具异常统一包装为可读文本返回给模型。
- 记录每次调用的 token 用量。

# 3. 多 Agent 群聊

**结论先行**：`AgentGroupChat` 让多个 Agent 共享一个会话，由**选择策略**决定谁发言、**终止策略**决定何时结束。

## 3.1 组成要素

| 要素 | 作用 |
|------|------|
| Agents | 参与群聊的 Agent 列表 |
| SelectionStrategy | 选择下一个发言者 |
| TerminationStrategy | 判断是否终止 |
| ChatHistory | 共享消息历史 |

## 3.2 构建 AgentGroupChat

### 3.2.1 执行设置

```java
AgentGroupChat chat = new AgentGroupChat(writerAgent, reviewerAgent);

chat.setExecutionSettings(
        KernelArguments.builder()
                .withExecutionSettings(new AgentGroupChatSettings() {
                    @Override
                    public SelectionStrategy getSelectionStrategy() {
                        return new SequentialSelectionStrategy();
                    }
                    @Override
                    public TerminationStrategy getTerminationStrategy() {
                        return new ApprovalTerminationStrategy();
                    }
                })
                .build());
```

### 3.2.2 添加消息与执行

```java
chat.addChatMessage(new ChatMessageContent(AuthorRole.USER, "写一篇关于 AI 的文章"));

// 执行直到终止
chat.invokeAsync().block();
```

## 3.3 完整示例

```java
public class GroupChatDemo {

    public static void main(String[] args) {
        Kernel kernel = buildKernel();

        ChatCompletionAgent writer = ChatCompletionAgent.builder()
                .name("Writer")
                .instructions("你是撰稿人。完成初稿后以 [DRAFT] 结尾。")
                .kernel(kernel)
                .build();

        ChatCompletionAgent reviewer = ChatCompletionAgent.builder()
                .name("Reviewer")
                .instructions("你是审稿人。满意时回复 [APPROVED]，否则给出修改意见。")
                .kernel(kernel)
                .build();

        AgentGroupChat chat = new AgentGroupChat(writer, reviewer);
        chat.setExecutionSettings(KernelArguments.builder()
                .withExecutionSettings(AgentGroupChatSettings.builder()
                        .withSelectionStrategy(new SequentialSelectionStrategy())
                        .withTerminationStrategy(new ApprovalTerminationStrategy())
                        .build())
                .build());

        chat.addChatMessage(new ChatMessageContent(
                AuthorRole.USER, "写一篇 200 字关于 AI 的短文"));
        chat.invokeAsync().block();

        chat.getChatHistory().forEach(m ->
                System.out.println(m.getAuthorRole() + ": " + m.getContent()));
    }
}
```

## 3.4 边界与坑

- 群聊消息历史会快速增长，需截断。
- 选择策略若一直不选到"能终止"的 Agent，会死循环。
- 多个 Agent 共享 Kernel 时注意插件的线程安全。

## 3.5 生产建议

- 群聊务必配置最大轮数。
- 消息历史做窗口截断或摘要压缩。
- 为每个 Agent 单独记录 token 成本。

# 4. 协作模式

**结论先行**：协作由两类策略构成——**选择策略**（谁来说）与**终止策略**（何时停）。

## 4.1 三种模式

| 模式 | 说明 |
|------|------|
| 顺序 | 按固定顺序轮流 |
| 选择 | 按规则/模型选择下一个发言者 |
| 终止 | 满足条件时结束 |

## 4.2 顺序模式

### 4.2.1 机制

按 Agent 注册顺序循环，直到终止策略触发。优点是确定、可复现。

### 4.2.2 示例

```java
public class SequentialSelectionStrategy extends SelectionStrategy {
    @Override
    public Agent selectNext(List<Agent> agents, List<ChatMessageContent> history) {
        int idx = history.size() % agents.size();
        return agents.get(idx);
    }
}
```

## 4.3 选择模式

### 4.3.1 机制

由模型阅读历史后决定下一位发言者。适合角色分工不固定的场景。

### 4.3.2 示例

```java
// 伪代码：由模型选择下一个发言者
public Agent selectNext(List<Agent> agents, List<ChatMessageContent> history) {
    String prompt = "根据对话历史，从 " + names(agents)
            + " 中选择下一个最合适的发言者，只返回名字。";
    String name = chat.complete(prompt + render(history));
    return findByName(agents, name);
}
```

## 4.4 终止模式

### 4.4.1 机制

终止策略检查最后一条消息或整体状态，返回 `true` 结束群聊。

### 4.4.2 示例

```java
// 终止策略：审查者批准即结束
public class ApprovalTerminationStrategy extends TerminationStrategy {
    @Override
    protected boolean shouldTerminate(AgentMessage message) {
        return message.getContent().contains("[APPROVED]");
    }
}
```

## 4.5 边界与坑

- 选择策略过于"聪明"会导致行为不可预测。
- 终止条件依赖关键词，模型换个说法就失效。
- 顺序模式可能与任务逻辑不符（顺序错）。

## 4.6 生产建议

- 终止条件用**结构化标记**（如 JSON 字段）而非自然语言。
- 同时设置"关键词终止 + 最大轮数终止"双保险。
- 选择策略优先用规则，其次才用模型。

## 4.7 策略组合矩阵

| 选择策略 | 终止策略 | 适用场景 |
|----------|----------|----------|
| 顺序 | 关键词 | 写-审稿循环 |
| 顺序 | 轮数 | 固定轮流讨论 |
| 模型选择 | 关键词 | 角色不固定的协作 |
| 模型选择 | 轮数 | 头脑风暴 |

# 5. 线程与状态

**结论先行**：`AgentThread` 是会话状态的载体，**状态默认在内存**，生产环境必须外置持久化。

## 5.1 机制

### 5.1.1 线程类型

| 类型 | 说明 |
|------|------|
| `ChatHistoryAgentThread` | 基于内存消息历史 |
| 自定义线程 | 对接外部存储 |

### 5.1.2 状态生命周期

```
创建线程 → 多轮调用（追加消息）→ 序列化 → 存储
                                        │
                              下次请求 ← 反序列化
```

## 5.2 多轮对话示例

```java
// 线程维护会话状态
AgentThread thread = new ChatHistoryAgentThread();

agent.invokeAsync("我叫张三", thread).block();
agent.invokeAsync("我叫什么？", thread).block();   // → 张三
```

## 5.3 持久化

### 5.3.1 序列化

```java
// 伪代码：把线程历史序列化后存储
List<ChatMessageContent> history = ((ChatHistoryAgentThread) thread).getChatHistory();
String json = objectMapper.writeValueAsString(history);
redis.setex("thread:" + sessionId, 3600, json);
```

### 5.3.2 存储选型

| 存储 | 适用 | 说明 |
|------|------|------|
| 内存 | 单机演示 | 重启即丢 |
| Redis | 在线会话 | TTL 过期 |
| 数据库 | 审计/长期 | 可查询 |

## 5.4 边界与坑

- 历史无限增长会撑爆上下文窗口。
- 多实例部署下内存线程不共享，会话会"丢记忆"。
- 序列化格式变更后旧数据可能反序列化失败。

## 5.5 生产建议

- 按 `sessionId` 存取线程，网关层做粘性或共享存储。
- 历史超过阈值时做摘要压缩。
- 为序列化结构加版本号。

## 5.6 并发与一致性

### 5.6.1 并发写同一会话

| 方案 | 说明 |
|------|------|
| 悲观锁 | 会话级锁，串行处理 |
| 乐观锁 | 版本号冲突则重试 |
| 队列 | 按 sessionId 路由到同一消费者 |

### 5.6.2 示例

```java
// 伪代码：会话级串行
synchronized (lockFor(sessionId)) {
    AgentThread thread = loadThread(sessionId);
    agent.invokeAsync(input, thread).block();
    saveThread(sessionId, thread);
}
```

## 5.7 边界与坑

- 跨实例并发写同一会话会丢消息。
- 长历史 + 大模型 = 超上下文窗口。

# 6. 生产注意事项

**结论先行**：多 Agent 系统上生产，核心是**可终止、可观测、可授权、可持久化**。

## 6.1 风险清单

| 风险 | 建议 |
|------|------|
| 无限对话 | 终止策略 + 步数上限 |
| 成本高 | 限制 Agent 数 |
| 无 Trace | 记录每轮 |
| 状态丢失 | 持久化线程 |
| 权限 | Agent 级授权 |

## 6.2 终止与预算

### 6.2.1 终止策略

```java
public class CompositeTerminationStrategy extends TerminationStrategy {
    @Override
    protected boolean shouldTerminate(AgentMessage message) {
        return message.getContent().contains("[APPROVED]")
                || message.getContent().contains("[ABORT]");
    }
}
```

### 6.2.2 步数上限

```java
// 伪代码：最大轮数保护
if (history.size() > MAX_TURNS) {
    throw new IllegalStateException("群聊超过最大轮数，已强制终止");
}
```

## 6.3 可观测

| 指标 | 说明 |
|------|------|
| 每轮 token | 成本核算 |
| 每轮延迟 | 性能分析 |
| 终止原因 | 策略有效性 |
| 工具调用次数 | 异常检测 |

## 6.4 权限

```java
// 伪代码：Agent 级授权
if (!authz.canUseAgent(currentUser(), agent.getName())) {
    throw new AccessDeniedException("无权使用该 Agent");
}
```

## 6.5 生产建议

- Agent 数量控制在 3~5 个以内。
- 群聊每轮消息全部落库。
- 关键 Agent 加人工确认环节。

## 6.6 灰度与降级

### 6.6.1 灰度

```yaml
# application.yml
sk:
  agents:
    group-chat:
      enabled: true
      max-turns: 8
      rollout-percent: 10      # 仅 10% 流量
```

### 6.6.2 降级

| 故障 | 降级策略 |
|------|----------|
| 模型超时 | 返回缓存答案 |
| 工具不可用 | 跳过该工具并提示 |
| 群聊异常 | 退回单 Agent |

结论：**每个 Agent 都要有降级路径**。

# 7. 小结

- SK 提供 ChatCompletionAgent 与 AgentGroupChat。
- Agent = 模型 + 指令 + 插件 + 线程。
- 多 Agent 用选择策略 + 终止策略。
- 线程维护会话状态，需持久化。
- 必须设终止条件与步数上限。
