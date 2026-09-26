# 1. LlmAgent 概述与创建

LlmAgent（大模型智能体，Large Language Model Agent）是 Google ADK Java 中最基础、最常用的 Agent 实现。它把"指令（instruction）+ 模型（model）+ 工具（tools）+ 子 Agent（subAgents）"封装成一个可被 Runner 执行的单元。

## 1.1 结论先行

- LlmAgent 是"由大模型驱动"的 Agent：下一步是直接回答、调用工具还是委托子 Agent，由模型动态决定，而非硬编码。
- 一切从 `LlmAgent.builder()` 开始；构建完成后的 Agent 应视为不可变对象，运行时由 `Runner` 驱动。
- 它与 `SequentialAgent`、`ParallelAgent`、`LoopAgent` 等确定性工作流 Agent 互补：前者负责"决策"，后者负责"编排"。
- 最简可用配置只需 `name`、`model`、`instruction` 三项，但生产环境通常还要加 `description`、`tools`、`outputKey` 与生成参数。

> 记忆口诀：LlmAgent 负责"想"，Runner 负责"跑"，Session 负责"记"。

## 1.2 机制拆解

### 1.2.1 核心属性

| 属性 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | String | 是 | Agent 唯一名称，用于日志、委托与 Trace |
| `description` | String | 否 | 描述，供父 Agent 选择子 Agent 时参考 |
| `model` | String | 是 | 模型标识，如 `gemini-2.0-flash` |
| `instruction` | String | 否 | 系统指令（系统提示），决定行为边界 |
| `tools` | List | 否 | 工具列表，可为对象、注解方法或内置工具 |
| `subAgents` | List | 否 | 子 Agent 列表，实现委托 |
| `generateContentConfig` | Config | 否 | 温度、topP、最大输出 token 等生成参数 |
| `outputKey` | String | 否 | 将本 Agent 最终结果写入 State 的键 |

### 1.2.2 执行循环

```
用户输入
   |
   v
Runner 组装上下文(指令 + 历史 + 工具 schema)
   |
   v
调用模型 --返回函数调用?-- 是 --> 执行工具 --> 结果回填 --+
   | 否                                                   |
   v                                                      |
返回最终文本事件 <----------------------------------------+
```

- 模型每次返回要么是"最终文本"，要么是"函数调用（function call）"。
- 若是函数调用，ADK 执行对应工具，把结果作为新的内容追加进对话，再次调用模型，直到产出文本。
- 这个"调用—执行—回填"的循环是 Agent 与普通 Chatbot 的本质区别。

### 1.2.3 模型选择

| 模型 | 特点 | 适用 |
|------|------|------|
| `gemini-2.0-flash` | 快、便宜 | 高频对话、抽取 |
| `gemini-2.0-pro` | 强推理 | 复杂规划 |
| 自定义微调模型 | 领域适配 | 垂直场景 |

- 模型以字符串标识传入，便于通过配置切换。
- 生产环境建议封装"模型工厂"，把模型名与生成参数集中管理，避免散落。

## 1.3 代码示例

### 1.3.1 最小 Agent

```java
LlmAgent agent = LlmAgent.builder()
        .name("assistant")
        .model("gemini-2.0-flash")
        .instruction("你是一个专业的中文助手。")
        .build();
```

### 1.3.2 客服 Agent（完整版）

```java
LlmAgent agent = LlmAgent.builder()
        .name("customer-service")
        .description("企业客服助手，处理订单与知识库问题")
        .model("gemini-2.0-flash")
        .instruction("""
                你是企业客服助手。

                规则：
                1. 只依据知识库回答；
                2. 资料不足时明确说明；
                3. 无法处理时转人工。
                """)
        .tools(new OrderTools(), new KnowledgeTools())
        .generateContentConfig(
                GenerateContentConfig.builder()
                        .temperature(0.2f)
                        .maxOutputTokens(1024)
                        .build())
        .build();
```

## 1.4 边界与坑

- `name` 在同一 Runner 的 Agent 树内必须唯一，重名会导致委托路由与 Trace 混乱。
- `description` 缺失时，父 Agent 只能靠名称猜测子 Agent 用途，委托准确率显著下降。
- `instruction` 不是"越长越好"：超长指令会挤占上下文预算并抬高成本，应精炼为规则条目。
- 构建后修改 Agent 字段通常无效（不可变语义），需要新建实例。

## 1.5 生产建议

- 为每个 Agent 建立命名规范，例如 `<域>-<职责>`（`order-query`、`refund-apply`）。
- `instruction` 与 `description` 纳入版本管理，变更需回归评测。
- 生成参数按场景固化：抽取类任务低温度，创意类任务可放宽。
- 用 `outputKey` 把结构化结果落进 State，便于后续节点消费。

# 2. 指令（Instruction）设计

指令是 LlmAgent 行为的"宪法"，它决定角色、任务、约束与工具使用方式。ADK 中指令以字符串形式传入 `instruction`。

## 2.1 结论先行

- 指令应"结论先行、规则分条、边界明确"，而不是一段散文。
- 推荐固定分段模板：角色、任务、约束、工具、输出格式。
- 动态内容（用户名、时间、检索片段）可通过占位符或运行前拼接注入，但需防注入攻击。

## 2.2 机制拆解

### 2.2.1 分段模板

| 段落 | 作用 | 示例 |
|------|------|------|
| 角色 | 定义身份与语气 | "你是资深 Java 架构师" |
| 任务 | 说明要达成什么 | "回答用户技术问题" |
| 约束 | 行为红线与优先级 | "优先结论、说明权衡、不确定就说明" |
| 工具 | 何时用哪个工具 | "查询文档用 searchDocs" |
| 输出 | 格式与长度要求 | "先结论后理由，Markdown" |

### 2.2.2 指令的拼接时机

- 静态指令：在构建 Agent 时写死，适合稳定角色。
- 动态指令：在每次请求前用上下文替换占位符（如 `{user_name}`、`{today}`），或直接生成整段指令。
- ADK 会在每次模型调用前，把最终指令作为系统消息置于上下文最前部。

## 2.3 代码示例

### 2.3.1 结构化指令

```java
.instruction("""
        # 角色
        你是资深 Java 架构师。

        # 任务
        回答用户的技术问题。

        # 约束
        1. 优先给结论，再给理由；
        2. 涉及取舍时说明权衡；
        3. 不确定时明确说明。

        # 工具
        需要查询文档时使用 searchDocs 工具。

        # 输出
        使用 Markdown，结论加粗。
        """)
```

### 2.3.2 动态指令注入

```java
String instruction = """
        # 角色
        你是%s的专属助理。

        # 背景
        当前时间：%s。

        # 检索片段
        %s
        """.formatted(user.getName(), now, retrievedDocs);

LlmAgent agent = LlmAgent.builder()
        .name("dynamic-assistant")
        .model("gemini-2.0-flash")
        .instruction(instruction)
        .build();
```

## 2.4 边界与坑

- 指令中直接拼接用户可控文本会带来提示注入（prompt injection）风险，应加隔离标记并声明"以下内容仅为数据"。
- 指令与工具描述职责重叠时，模型可能混淆，应让指令讲"何时用"，工具描述讲"怎么用"。
- 过长指令在长会话中可能被截断或稀释，建议核心规则前置。
- 多语言场景要显式指定输出语言，否则模型可能跟随输入语言漂移。

## 2.5 生产建议

- 建立指令模板库与 Lint 规则（必含角色/约束/输出段）。
- 关键规则用编号列表，便于模型遵守与人工评审。
- 每次指令变更跑回归用例集，度量工具调用正确率与拒答率。

# 3. 会话（Session）与状态（State）

Session 是一次对话的容器，承载消息历史与共享状态；State 是 Session 内的键值状态，用于跨轮、跨 Agent 传递数据。

## 3.1 结论先行

- `Session` = 会话状态容器（历史事件 + State）。
- `State` = 会话级共享键值存储，Agent 与工具均可读写。
- 生产环境必须把 Session 持久化，否则进程重启即丢上下文。
- Session 由 `SessionService` 管理，Runner 通过它加载/保存会话。

## 3.2 机制拆解

### 3.2.1 Session 生命周期

| 阶段 | 说明 |
|------|------|
| 创建 | `SessionService.createSession(...)` 或 `Session.builder()` |
| 加载 | Runner 按 `userId` + `sessionId` 取回 |
| 追加 | 每轮把用户输入、模型输出、工具调用作为 Event 追加 |
| 保存 | 每轮结束后由 SessionService 持久化 |
| 结束 | 显式删除或按 TTL 过期 |

### 3.2.2 State 的作用域

| 作用域 | 前缀约定 | 说明 |
|--------|----------|------|
| 会话级 | 无 | 仅当前 Session 可见 |
| 用户级 | `user:` | 跨该用户所有会话共享 |
| 应用级 | `app:` | 全局共享（慎用） |
| 临时级 | `temp:` | 单次运行内有效，不持久化 |

### 3.2.3 持久化表结构（示意）

```sql
CREATE TABLE sessions (
    app_name   VARCHAR(64)  NOT NULL,
    user_id    VARCHAR(64)  NOT NULL,
    session_id VARCHAR(64)  NOT NULL,
    state      JSON         NULL,
    updated_at TIMESTAMP    NOT NULL,
    PRIMARY KEY (app_name, user_id, session_id)
);
```

## 3.3 代码示例

```java
// 创建会话
Session session = Session.builder()
        .userId("user-1")
        .sessionId("session-1")
        .build();

// 运行
RunResult result = runner.run(session, "你好");

// 读写状态
session.state().put("userId", "u-1");
session.state().put("user:level", "gold");
String userId = (String) session.state().get("userId");
```

| 概念 | 说明 |
|------|------|
| `Session` | 会话状态容器 |
| `State` | 会话中的共享状态 |
| `Events` | 会话事件（消息、工具调用） |

## 3.4 边界与坑

- 只创建 Session 不持久化，等于每次重启都"失忆"。
- 并发写同一 Session 需要加锁或串行化，否则历史顺序错乱。
- `app:` 级状态在多租户下极易越权泄漏，非必要不用。
- State 值应是可序列化对象；存入不可序列化对象在持久化时会失败。

## 3.5 生产建议

- 用 `DatabaseSessionService` 或托管会话服务落库。
- 为 Session 设置 TTL 与容量上限，避免无限增长。
- 用户级偏好放 `user:`，请求级临时数据放 `temp:`，职责清晰。

# 4. 事件（Event）与执行器（Runner）

Runner 是 Agent 的执行器，负责组装上下文、驱动模型与工具循环、产出事件流。

## 4.1 结论先行

- `Runner.builder().agent(agent).build()` 创建执行器，可再挂 `sessionService`、`appName` 等。
- 提供同步 `run(...)` 与流式 `runStreaming(...)` 两种执行方式。
- 所有中间产物都以 `Event` 形式暴露，是 Trace、调试与前端流式渲染的统一接口。

## 4.2 机制拆解

### 4.2.1 Event 类型

| 事件类型 | 含义 |
|----------|------|
| 用户消息 | 用户输入 |
| 模型文本 | 模型增量/完整文本 |
| 函数调用 | 模型请求调用工具 |
| 函数响应 | 工具返回结果 |
| 委托 | 转交子 Agent |
| 错误 | 执行异常 |

### 4.2.2 同步 vs 流式

| 维度 | `run` | `runStreaming` |
|------|-------|----------------|
| 返回 | 完整 `RunResult` | `Flowable<Event>` |
| 延迟 | 全量结束后返回 | 逐 token 推送 |
| 适用 | 批处理、后台任务 | 对话 UI、低延迟 |
| 取消 | 较难 | 支持退订取消 |

### 4.2.3 错误处理

| 错误 | 处理 |
|------|------|
| 模型限流 | 指数退避重试 |
| 模型不可用 | 切换备用模型 |
| 工具异常 | 返回结构化错误给模型 |
| 超时 | 中断并降级 |

- 工具异常应转成模型可理解的文本，让 Agent 有机会重试或换路径，而不是直接崩溃。

## 4.3 代码示例

```java
Runner runner = Runner.builder()
        .appName("customer-service")
        .agent(agent)
        .sessionService(sessionService)
        .build();

// 单次运行
RunResult result = runner.run(session, "查询订单 A123");
result.events().forEach(e -> log.info("事件: {}", e));

// 流式
runner.runStreaming(session, "你好")
        .subscribe(event -> System.out.print(event.content()));
```

## 4.4 边界与坑

- 流式订阅若未处理背压（backpressure），前端消费慢时可能积压事件。
- `run` 在长任务上会阻塞线程，Web 场景应使用异步或流式。
- Event 顺序即执行顺序，持久化时须保序，否则回放会错乱。

## 4.5 生产建议

- 为 Runner 统一注入 `sessionService`、`artifactService` 与鉴权上下文。
- 对流式加超时与取消信号，用户关闭页面时及时释放模型调用。
- 把 Event 落到可检索存储，支撑回放、审计与评测。

# 5. 状态传递与上下文管理

Agent 之间、Agent 与工具之间需要共享数据，ADK 通过 State 与事件传递，同时受上下文窗口约束。

## 5.1 结论先行

- State 是跨轮、跨 Agent 传值的主通道；`outputKey` 是常用写入方式。
- 上下文窗口有限，长会话必须做压缩、摘要或检索裁剪。
- 传递结构化数据优先于传递自然语言，减少模型解析负担。

## 5.2 机制拆解

### 5.2.1 outputKey 写回

```java
LlmAgent extractor = LlmAgent.builder()
        .name("extractor")
        .model("gemini-2.0-flash")
        .instruction("从用户描述中抽取订单号，仅输出订单号。")
        .outputKey("order_id")
        .build();

LlmAgent handler = LlmAgent.builder()
        .name("handler")
        .model("gemini-2.0-flash")
        .instruction("根据 state 中的 order_id 处理订单。")
        .build();
```

### 5.2.2 上下文压缩策略

| 策略 | 做法 | 代价 |
|------|------|------|
| 滑动窗口 | 只保留最近 N 轮 | 丢失早期信息 |
| 摘要 | 用模型压缩历史 | 额外调用成本 |
| 检索 | 按需召回相关片段 | 需向量库 |
| 结构化 State | 关键事实入 State | 需设计 schema |

## 5.3 代码示例

```java
// 运行前注入上下文
session.state().put("locale", "zh-CN");
session.state().put("user:plan", "enterprise");

// 工具内读取 State
@Tool(description = "按当前用户偏好回复")
public String reply(@ToolParam(description = "文本") String text) {
    String plan = (String) context.state().get("user:plan");
    return service.render(text, plan);
}
```

## 5.4 边界与坑

- State 中的大对象会随每轮请求进入上下文，造成 token 爆炸。
- 覆盖写同名 key 会丢失历史，需要保留历史时应追加到列表。
- 跨 Agent 传值若依赖模型复述，信息会失真，应走 State。

## 5.5 生产建议

- 为 State 建立 key 命名规范与 schema 校验。
- 大文档存 Artifact（制品）而非 State，仅在需要时按引用取用。
- 定期审计 State 大小，设上限并告警。

# 6. 回调与生命周期钩子

回调（Callback）允许在 Agent 执行的关键节点插入自定义逻辑，用于鉴权、日志、改写、拦截与埋点。

## 6.1 结论先行

- 回调是"横切关注点"的落点：鉴权、审计、限流、内容安全。
- 回调可改写输入/输出，也可直接短路返回。
- 回调抛异常会中断执行，需谨慎处理。

## 6.2 机制拆解

### 6.2.1 常见钩子

| 钩子 | 触发时机 | 典型用途 |
|------|----------|----------|
| BeforeAgent | Agent 执行前 | 鉴权、注入上下文 |
| AfterAgent | Agent 执行后 | 落库、脱敏 |
| BeforeModel | 调模型前 | 改写提示、限流 |
| AfterModel | 调模型后 | 过滤内容、埋点 |
| BeforeTool | 工具调用前 | 参数校验、授权 |
| AfterTool | 工具调用后 | 结果裁剪、审计 |

### 6.2.2 回调执行顺序

```
BeforeAgent -> (BeforeModel -> 模型 -> AfterModel -> BeforeTool -> 工具 -> AfterTool)* -> AfterAgent
```

### 6.2.3 回调返回值语义

| 返回 | 含义 |
|------|------|
| `Optional.empty()` | 继续正常流程 |
| `Optional.of(x)` | 短路，用 x 替代默认结果 |
| 抛异常 | 中断执行并上抛 |

- 短路常用于命中缓存或鉴权失败时直接返回，避免不必要的模型调用。

## 6.3 代码示例

```java
LlmAgent agent = LlmAgent.builder()
        .name("guarded-agent")
        .model("gemini-2.0-flash")
        .instruction("你是受管助手。")
        .beforeModelCallback(ctx -> {
            if (rateLimiter.exceeded(ctx.userId())) {
                return Optional.of(shortCircuit("请求过于频繁"));
            }
            return Optional.empty();
        })
        .afterModelCallback((ctx, response) -> {
            audit.log(ctx.sessionId(), response.text());
            return Optional.of(response);
        })
        .build();
```

## 6.4 边界与坑

- 回调内做重 IO 会拖慢整条链路，应异步或轻量化。
- 短路返回需符合框架约定类型，否则可能被忽略。
- 回调中吞掉异常会掩盖故障，应记录并决定是否上抛。

## 6.5 生产建议

- 把鉴权、限流、内容安全统一收敛到回调，避免散落各处。
- 回调中不打印敏感原文，遵循脱敏规范。
- 为回调耗时建立指标，纳入延迟预算。

# 7. 生产注意事项

把 Agent 从 Demo 推向生产，关键在可观测、可靠性与安全。

## 7.1 结论先行

- 没有 Trace 的 Agent 等于"黑盒"，故障无法定位。
- 会话持久化、工具鉴权、超时与降级是上线底线。
- 成本主要来自 token 与工具调用次数，需设预算与限流。

## 7.2 风险与对策

| 风险 | 影响 | 建议 |
|------|------|------|
| 指令过长 | 成本高、遵守率低 | 精简、模板化 |
| 无会话持久化 | 重启丢上下文 | 持久化 Session |
| 无 Trace | 无法排障 | 记录事件链 |
| 工具无鉴权 | 越权操作 | 工具内校验 |
| 无限循环 | 成本失控 | 限制轮次/工具调用 |
| 无降级 | 模型故障全挂 | 备用模型 |

## 7.3 可观测与成本

```java
runner.runStreaming(session, input)
        .doOnNext(e -> metrics.record(e))
        .doOnError(err -> log.error("run failed", err))
        .subscribe();
```

| 指标 | 说明 |
|------|------|
| 首 token 延迟 | 用户体感 |
| 总耗时 | 端到端 |
| token 用量 | 输入/输出 |
| 工具调用数 | 成本与异常信号 |
| 委托次数 | 多 Agent 开销 |

## 7.4 安全建议

- 工具内二次鉴权，不信任模型传参。
- 对输入做提示注入检测，对输出做敏感信息过滤。
- 记录完整审计日志，满足合规与回溯。

# 8. 小结

## 8.1 核心要点回顾

- 用 `LlmAgent.builder()` 定义 Agent，核心属性为 name、description、model、instruction、tools、subAgents。
- 指令采用"角色/任务/约束/工具/输出"分段模板，动态注入需防注入。
- Session 管理会话状态与事件，State 支持会话/用户/应用/临时多级作用域。
- Runner 执行 Agent，支持同步与流式，Event 是统一观测接口。
- 回调承载鉴权、审计、限流等横切逻辑。

## 8.2 生产检查清单

- [ ] 每个 Agent 有唯一 name 与清晰 description。
- [ ] 指令经过评审且纳入版本管理。
- [ ] Session 已持久化并设置 TTL。
- [ ] 工具调用有鉴权、超时、审计。
- [ ] 已建立 Trace、指标与成本看板。
- [ ] 具备降级与限流策略。
