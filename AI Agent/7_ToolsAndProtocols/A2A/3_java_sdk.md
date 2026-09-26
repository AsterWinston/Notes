# 1. SDK 概览与总体设计

结论先行：A2A（Agent-to-Agent，智能体间协议）的 Java SDK 把"对外暴露 Agent 能力"与"调用其他 Agent"两件事抽象成五个核心构件——`AgentCard`、`AgentExecutor`、`EventQueue`、`TaskStore`、`A2AClient`。理解这五个构件的职责边界，是写好 A2A 服务端与客户端的前提。

## 1.1 为什么需要 Java SDK

A2A 是一套基于 HTTP + JSON-RPC 2.0 的开放协议。理论上你可以手写 JSON 直接通信，但 SDK 承担了三类重复劳动：

- 协议编解码：请求/响应/事件（Event）的序列化与反序列化。
- 传输适配：轮询、SSE（Server-Sent Events，服务端推送）、推送通知的统一抽象。
- 生命周期管理：任务（Task）的创建、状态流转、持久化与恢复。

因此，SDK 的价值不是"发明新协议"，而是"把协议细节封进类型安全的 Java API"。

## 1.2 核心构件与职责

| 构件 | 作用 | 所在侧 |
|------|------|--------|
| `A2AServer` | 暴露 Agent 能力，处理入站请求 | 服务端 |
| `AgentCard` | 能力描述（能力发现入口） | 双方 |
| `AgentExecutor` | 任务执行逻辑（业务实现点） | 服务端 |
| `EventQueue` | 执行器产出事件的出口 | 服务端 |
| `TaskStore` | 任务持久化与查询 | 服务端 |
| `A2AClient` | 调用其他 Agent | 客户端 |

### 1.2.1 服务端构件

服务端一侧由 `A2AServer` 统一编排：它读取 `AgentCard` 对外发布能力，把入站消息交给 `AgentExecutor` 执行，把执行器产出的事件写入 `EventQueue`，并借助 `TaskStore` 保存任务状态。业务开发者通常只实现 `AgentExecutor` 与 `TaskStore`。

### 1.2.2 客户端构件

客户端一侧由 `A2AClient` 承担：先通过 `AgentCard` 了解远端能力，再发送消息创建任务，最后以轮询、流式或推送方式接收结果。客户端无需感知服务端内部如何执行。

> 具体 API 以官方最新 SDK 为准；本文给出结构化的实现范式，个别方法名以伪代码形式标注。

## 1.3 一次调用的完整时序

```

 Client                     Server
   |                          |
   |-- GET agent-card --------|  1. 能力发现
   |<-- AgentCard ------------|
   |                          |
   |-- sendMessage (POST) --->|  2. 创建任务
   |<-- Task(SUBMITTED) ------|
   |                          |  3. AgentExecutor.execute
   |<-- status: WORKING ------|     enqueue 状态事件
   |<-- artifact: trip-plan --|     enqueue 产出事件
   |<-- status: COMPLETED ----|     enqueue 终态事件
   |                          |
   |-- getTask (可选) ------->|  4. 断线后恢复
   |<-- Task(COMPLETED) ------|

```

## 1.4 边界与生产建议

- 边界：SDK 只负责协议与传输，不负责业务幂等、超时、重试，这些必须由业务层补齐。
- 生产建议：把"能力发现"与"任务调用"分离部署（前者可缓存、可 CDN，后者需鉴权）。

# 2. 定义 Agent Card

结论先行：`AgentCard` 是 A2A 的能力发现入口，等价于"Agent 的名片"。它同时描述基础信息（名称、URL、版本）、协议能力（是否支持流式、推送）与技能（Skills）列表。

## 2.1 结论与作用

`AgentCard` 通常通过 `GET /.well-known/agent-card.json` 暴露，供其他 Agent 在调用前发现能力。它决定了对端可以怎么调用你，因此必须与实际实现严格一致。

## 2.2 字段拆解

| 字段 | 含义 | 备注 |
|------|------|------|
| `name` | Agent 名称 | 人类可读 |
| `description` | 能力描述 | 供对端理解 |
| `url` | A2A 端点 | 调用入口 |
| `version` | 版本号 | 语义化版本 |
| `capabilities` | 协议能力 | streaming / pushNotifications |
| `defaultInputModes` | 默认输入类型 | 如 text |
| `defaultOutputModes` | 默认输出类型 | 如 text |
| `skills` | 技能列表 | 每项含 id/name/examples |

## 2.3 完整示例

```java
AgentCard agentCard = AgentCard.builder()
        .name("Travel Agent")
        .description("提供旅行规划服务")
        .url("https://travel-agent.example.com/a2a")
        .version("1.0.0")
        .capabilities(AgentCapabilities.builder()
                .streaming(true)
                .pushNotifications(true)
                .build())
        .defaultInputModes(List.of("text"))
        .defaultOutputModes(List.of("text"))
        .skills(List.of(
                AgentSkill.builder()
                        .id("plan-trip")
                        .name("旅行规划")
                        .description("根据目的地与预算规划行程")
                        .examples(List.of("帮我规划 3 天的东京行程"))
                        .build()))
        .build();
```

## 2.4 边界与坑

- 声明了 `streaming(true)` 却未实现流式端点，对端会在流式调用时失败。
- `skills` 的 `examples` 不是装饰，很多对端会据此做技能路由，缺失会降低命中率。
- `url` 必须是绝对地址，且与网关暴露的路径一致。

## 2.5 生产建议

- 将 `AgentCard` 作为配置生成，避免手写与实际能力漂移。
- 为 `AgentCard` 增加缓存头，并纳入版本管理以便灰度。

# 3. 实现 Agent 执行器

结论先行：`AgentExecutor` 是唯一的业务实现点。它接收 `RequestContext`，把执行过程以"事件"形式推入 `EventQueue`，由框架负责把事件转换为协议响应或推送。

## 3.1 执行模型

### 3.1.1 事件队列

执行器不直接返回结果，而是向 `EventQueue` 投递事件。常见事件类型：

| 事件 | 含义 |
|------|------|
| `TaskStatusUpdateEvent` | 状态变更（WORKING/COMPLETED/FAILED...） |
| `TaskArtifactUpdateEvent` | 产出物（Artifact）更新 |
| `MessageEvent` | 向用户发送消息 |

### 3.1.2 任务状态机

```

 SUBMITTED ──► WORKING ──► COMPLETED
      │            │
      │            ├────► FAILED
      │            ├────► CANCELED
      │            └────► INPUT_REQUIRED ──► WORKING (补充后)

```

## 3.2 完整执行器示例

```java
public class TravelAgentExecutor implements AgentExecutor {

    private final Assistant assistant;

    @Override
    public void execute(RequestContext context, EventQueue eventQueue) {
        try {
            // ① 标记开始工作
            eventQueue.enqueue(new TaskStatusUpdateEvent(
                    context.getTaskId(), TaskState.WORKING, false));

            // ② 提取用户消息
            String userText = extractText(context.getUserInput());

            // ③ 执行 Agent 逻辑
            String result = assistant.chat(userText);

            // ④ 产出 Artifact
            eventQueue.enqueue(new TaskArtifactUpdateEvent(
                    context.getTaskId(),
                    List.of(Artifact.builder()
                            .name("trip-plan")
                            .parts(List.of(TextPart.from(result)))
                            .build())));

            // ⑤ 标记完成
            eventQueue.enqueue(new TaskStatusUpdateEvent(
                    context.getTaskId(), TaskState.COMPLETED, true));

        } catch (Exception e) {
            eventQueue.enqueue(new TaskStatusUpdateEvent(
                    context.getTaskId(), TaskState.FAILED, true));
        }
    }

    @Override
    public void cancel(RequestContext context, EventQueue eventQueue) {
        // 处理取消
        eventQueue.enqueue(new TaskStatusUpdateEvent(
                context.getTaskId(), TaskState.CANCELED, true));
    }
}
```

## 3.3 取消与幂等

`cancel` 是协作式的：框架通知执行器取消，执行器需自行检查中断标志并尽快停止。若任务已进入终态，再次取消应被幂等忽略。

## 3.4 边界与生产建议

- 坑：在 `execute` 中做同步阻塞的耗时操作会占满线程池，应异步化。
- 坑：`final` 参数为 `true` 表示终态，漏设会导致客户端一直等待。
- 建议：为每个任务设置超时，超时后投递 `FAILED` 终态。

# 4. 启动 Server 与端点

结论先行：通过 Spring 配置装配 `A2AServer`，框架会暴露两个端点：能力发现端点与 JSON-RPC 调用端点。

## 4.1 Spring 配置

```java
@Configuration
public class A2AServerConfig {

    @Bean
    public A2AServer a2aServer(AgentExecutor travelExecutor,
                               TaskStore taskStore) {
        return A2AServer.builder()
                .agentCard(agentCard())
                .agentExecutor(travelExecutor)
                .taskStore(taskStore)              // 持久化任务
                .build();
    }

    // 暴露 Agent Card 与 A2A 端点
    // GET /.well-known/agent-card.json
    // POST /a2a  （JSON-RPC）
}
```

## 4.2 端点与发现

| 端点 | 方法 | 作用 |
|------|------|------|
| `/.well-known/agent-card.json` | GET | 能力发现 |
| `/a2a` | POST | JSON-RPC 调用入口 |

## 4.3 边界与生产建议

- 发现端点应公开；调用端点必须鉴权（见第 8 节）。
- 建议在网关层统一做限流与熔断，避免慢任务拖垮实例。

# 5. 任务持久化

结论先行：任务必须持久化，才能支持长任务、断线重连与故障恢复。`TaskStore` 是持久化抽象，生产环境应落到数据库。

## 5.1 为什么必须持久化

A2A 的任务可能持续数分钟甚至更久。若任务只存在内存，实例重启即丢失，客户端也无法通过 `getTask` 恢复。因此 `TaskStore` 是生产可用性的底线。

## 5.2 JDBC TaskStore 实现

```java
@Component
public class JdbcTaskStore implements TaskStore {

    private final JdbcTemplate jdbc;
    private final ObjectMapper mapper;

    @Override
    public void save(Task task) {
        jdbc.update("""
                INSERT INTO a2a_task (task_id, context_id, state, task_json, updated_at)
                VALUES (?, ?, ?, ?::jsonb, now())
                ON CONFLICT (task_id) DO UPDATE
                SET state = EXCLUDED.state,
                    task_json = EXCLUDED.task_json,
                    updated_at = now()
                """,
                task.id(), task.contextId(), task.status().state().name(),
                mapper.writeValueAsString(task));
    }

    @Override
    public Optional<Task> get(String taskId) {
        return jdbc.query("SELECT task_json FROM a2a_task WHERE task_id = ?",
                (rs, i) -> deserialize(rs.getString(1)), taskId)
                .stream().findFirst();
    }
}
```

> 任务必须持久化，才能支持长任务与恢复。

## 5.3 边界与生产建议

- 坑：`task_json` 直接整存整取，任务很大时写放大明显，可考虑列式拆分热点字段。
- 建议：为 `updated_at` 建索引，便于清理过期任务。

# 6. 客户端调用

结论先行：客户端有两种主要模式——轮询与流式。轮询简单但延迟高，流式实时但需要长连接支持。

## 6.1 轮询模式

```java
@Service
public class A2AClientService {

    public String callTravelAgent(String request) {
        // ① 获取 Agent Card
        AgentCard card = client.getAgentCard(
                "https://travel-agent.example.com");

        // ② 创建客户端
        A2AClient a2aClient = A2AClient.builder()
                .agentCard(card)
                .build();

        // ③ 发送消息（创建任务）
        Task task = a2aClient.sendMessage(MessageSendParams.builder()
                .message(new Message(Role.USER,
                        List.of(TextPart.from(request))))
                .build());

        // ④ 轮询或流式获取结果
        while (task.status().state() == TaskState.WORKING
                || task.status().state() == TaskState.SUBMITTED) {
            sleep(1000);
            task = a2aClient.getTask(task.id());
        }

        // ⑤ 提取产出
        return extractArtifacts(task);
    }
}
```

## 6.2 流式模式

### 6.2.1 事件类型

流式回调中会收到多种事件，需按类型分发。

```java
public void callStreaming(String request, Consumer<String> onUpdate) {
    client.sendMessageStreaming(
            MessageSendParams.builder()
                    .message(new Message(Role.USER, List.of(TextPart.from(request))))
                    .build(),
            event -> {
                if (event instanceof TaskStatusUpdateEvent update) {
                    onUpdate.accept("状态：" + update.status().state());
                } else if (event instanceof TaskArtifactUpdateEvent artifact) {
                    onUpdate.accept("产出：" + extractText(artifact.artifact()));
                }
            });
}
```

### 6.2.2 推送模式

若服务端声明 `pushNotifications(true)`，客户端可注册 Webhook，由服务端在状态变更时主动回调，适合超长任务。

## 6.3 边界与生产建议

- 坑：轮询间隔过短会放大服务端压力，过长则增加延迟，建议指数退避。
- 建议：优先使用流式；仅在不支持长连接时回退轮询。

# 7. 人机协同

结论先行：A2A 原生支持"任务中途需要输入"的协同模式，通过 `INPUT_REQUIRED` 状态与 `MessageEvent` 实现。

## 7.1 INPUT_REQUIRED 机制

当执行器发现缺少必要信息时，投递 `INPUT_REQUIRED` 状态（非终态）并附带一条向用户提问的消息。任务挂起，等待客户端补充输入后继续。

## 7.2 示例

```java
@Override
public void execute(RequestContext context, EventQueue eventQueue) {
    if (missingRequiredInfo(context)) {
        // 请求用户补充信息
        eventQueue.enqueue(new TaskStatusUpdateEvent(
                context.getTaskId(),
                TaskState.INPUT_REQUIRED,
                false));

        eventQueue.enqueue(new MessageEvent(
                new Message(Role.AGENT,
                        List.of(TextPart.from("请提供出行日期与预算")))));
        return;
    }
    // ...
}
```

> A2A 原生支持"任务中途需要输入"的人机协同模式。

## 7.3 边界与生产建议

- 坑：`INPUT_REQUIRED` 不是终态，若客户端误判为结束会提前终止流程。
- 建议：为等待输入设置超时，超时后转 `FAILED` 避免任务永久悬挂。

# 8. 安全

结论先行：A2A 的发现端点公开、调用端点必须鉴权。生产环境还需来源白名单、双向认证与全流程审计。

## 8.1 认证与授权

```java
@Configuration
public class A2ASecurityConfig {

    @Bean
    public SecurityFilterChain a2aSecurity(HttpSecurity http) throws Exception {
        return http
                .securityMatcher("/a2a/**")
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/.well-known/agent-card.json").permitAll()
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
                .build();
    }
}
```

## 8.2 来源白名单与审计

| 措施 | 说明 |
|------|------|
| Agent Card 公开 | 能力发现无需认证 |
| A2A 端点鉴权 | JWT/OAuth2 |
| 双向认证 | 高安全场景用 mTLS |
| 来源白名单 | 只接受可信 Agent |
| 调用审计 | 记录任务全流程 |

## 8.3 生产建议

- 拒绝匿名调用；对可信 Agent 签发短期凭证。
- 审计日志至少记录：调用方标识、任务 ID、状态流转时间线。

# 9. 生产注意事项

结论先行：A2A 上线前需逐项核对持久化、超时、幂等、信任、审计与长任务六类风险。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 任务未持久化 | 必须存库 |
| 无超时 | 设置任务超时 |
| 无幂等 | 任务 ID 幂等 |
| 信任外部 Agent | 白名单 + 鉴权 |
| 无审计 | 记录全流程 |
| 长任务阻塞 | 异步 + 推送 |

## 9.2 可观测性

建议对任务状态流转打点，暴露"进行中任务数、平均耗时、失败率"等指标，便于容量规划与告警。

# 10. 小结

- A2A Java 实现：Agent Card + AgentExecutor + TaskStore + Client。
- 任务必须持久化，支持长任务与恢复。
- 支持轮询、流式、推送三种交互。
- 原生支持 INPUT_REQUIRED 人机协同。
- 端点鉴权与来源白名单是生产必备。
