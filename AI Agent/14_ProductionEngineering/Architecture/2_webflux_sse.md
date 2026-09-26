# 1. 流式的价值与本质

## 1.1 结论先行

流式（Streaming）**不减少总耗时，但大幅改善用户体感**。它把"等待一个完整结果"的过程，变成"结果正在逐步生成"的过程，用户从"界面卡住"变为"正在输入"。

## 1.2 机制拆解

### 1.2.1 时间线对比

```
非流式：
t=0s ───────────────────────────── t=5s
[         等待，界面无任何输出          ] → 一次性返回全部

流式：
t=0s     t=0.2s                                t=5s
[ 等待 ] → 首字 → 逐字 → 逐字 → 逐字 → ... → 结束
```

### 1.2.2 关键指标

| 指标 | 非流式 | 流式 | 说明 |
|------|:------:|:----:|------|
| 首字节时间（TTFB） | 5s | 0.2s | 用户开始看到内容 |
| 总时长 | 5s | 5s | 完成时间基本不变 |
| 用户体感 | 差 | 好 | 从"卡住"变为"正在输入" |

> 感知延迟主要由首字节时间决定，而不是总时长。

## 1.3 边界与反例

- 若结果极短（如一次分类，300ms 内返回），流式收益有限，反而增加实现复杂度。
- 若消费方是机器（批处理、离线任务），流式无意义。
- 流式的错误处理更复杂：可能已经输出一半才失败，需要补偿协议。

## 1.4 生产建议

- 对话类、长文本生成类接口默认开启流式。
- 明确区分"人眼消费"与"机器消费"两类接口，只有前者需要流式。

# 2. 技术选型

## 2.1 结论先行

**SSE（Server-Sent Events，服务器发送事件）是 LLM 流式的事实标准**：基于 HTTP、实现简单、天然支持断线重连。WebSocket 适用于双向交互，gRPC Stream 适用于内部服务间高性能传输。

## 2.2 机制拆解

### 2.2.1 四种传输方式对比

| 技术 | 说明 | 方向 | 适用 |
|------|------|:----:|------|
| SSE | Server-Sent Events，基于 HTTP | 单向 | 对话流式（推荐） |
| WebSocket | 全双工长连接 | 双向 | 双向交互 |
| HTTP 分块 | Transfer-Encoding: chunked | 单向 | 简单场景 |
| gRPC Stream | HTTP/2 多路复用 | 单向/双向 | 内部服务 |

### 2.2.2 选型决策

```
需要客户端实时上行？
  ├─ 是 → WebSocket（如协作编辑、实时语音）
  └─ 否 → 只需要服务端推流？
            ├─ 是 → SSE（对话生成首选）
            └─ 否 → 简单分块即可
内部服务间高吞吐？ → gRPC Stream
```

## 2.3 边界与坑

- SSE 是纯文本协议，二进制数据需 Base64 编码。
- 浏览器对同域名 SSE 连接数有限制（HTTP/1.1 下约 6 个），HTTP/2 可缓解。
- WebSocket 需要额外的鉴权与心跳设计，SSE 可直接复用 HTTP 认证头。

## 2.4 生产建议

- 对外对话接口统一使用 SSE，降低前端接入成本。
- 内部服务间调用优先 gRPC Stream，减少序列化开销。

# 3. Spring WebFlux 与 SSE 基础

## 3.1 结论先行

在 Spring WebFlux 中，控制器返回 `Flux<ServerSentEvent<T>>` 即可自动映射为 SSE 流；`produces = MediaType.TEXT_EVENT_STREAM_VALUE` 是必需的声明。

## 3.2 机制拆解

### 3.2.1 返回类型映射

| 返回类型 | 行为 |
|----------|------|
| `Flux<String>` | 每个元素作为默认 `data:` 行 |
| `Flux<ServerSentEvent<T>>` | 可自定义 `event`/`id`/`retry`/`data` |
| `Mono<T>` | 退化为普通 JSON 响应 |

### 3.2.2 背压与取消

Reactor 的 `Flux` 天然携带背压（Backpressure）与取消（Cancel）信号。当 HTTP 连接关闭，WebFlux 会向上游发送取消信号，从而终止模型调用。

## 3.3 完整示例

```java
@RestController
@RequestMapping("/api/chat")
public class StreamingChatController {

    private final ChatClient chatClient;

    public StreamingChatController(ChatClient chatClient) {
        this.chatClient = chatClient;
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> stream(
            @RequestParam String message,
            @RequestHeader("X-Session-Id") String sessionId) {

        ServerSentEvent<String> start = ServerSentEvent.<String>builder()
                .event("start")
                .data("{\"sessionId\":\"" + sessionId + "\"}")
                .build();

        Flux<ServerSentEvent<String>> body = chatClient.prompt()
                .user(message)
                .stream()
                .content()
                .map(chunk -> ServerSentEvent.<String>builder()
                        .event("message")
                        .data(chunk)
                        .build());

        ServerSentEvent<String> done = ServerSentEvent.<String>builder()
                .event("done")
                .data("[DONE]")
                .build();

        return Flux.just(start)
                .concatWith(body)
                .concatWith(Mono.just(done))
                .onErrorResume(e -> Mono.just(ServerSentEvent.<String>builder()
                        .event("error")
                        .data("生成失败：" + e.getMessage())
                        .build()));
    }
}
```

## 3.4 边界与坑

- `onErrorResume` 若在流中间触发，已发送的内容无法撤回，只能追加 `error` 事件。
- 不要在流中执行阻塞 IO（如 JDBC），会阻塞事件循环线程，应切换到 `Schedulers.boundedElastic()`。

## 3.5 生产建议

- 首帧先发 `start` 事件，便于前端建立会话状态。
- 统一用 `ServerSentEvent` 而非裸字符串，方便扩展协议字段。

# 4. 事件协议设计

## 4.1 结论先行

一个清晰的 SSE 事件协议应至少包含 `start`、`message`、`done`、`error` 四类事件；缺失 `done` 或 `error` 会导致前端无法判断流是否正常结束。

## 4.2 事件类型定义

### 4.2.1 核心事件

| 事件 | 用途 |
|------|------|
| `start` | 会话开始（含会话 ID 等元信息） |
| `message` | 内容增量 |
| `done` | 正常结束（含 token、成本统计） |
| `error` | 异常结束（含错误码与信息） |

### 4.2.2 扩展事件

| 事件 | 用途 |
|------|------|
| `thinking` | 模型思考/推理中间状态 |
| `tool_call` | 工具调用开始 |
| `tool_result` | 工具调用结果 |
| `sources` | RAG 引用来源 |

## 4.3 协议示例

```
event: start
data: {"sessionId":"s1"}

event: message
data: 你

event: message
data: 好

event: done
data: {"tokens":15,"cost":0.001}

event: error
data: {"code":"MODEL_TIMEOUT","message":"模型超时"}
```

## 4.4 边界与坑

- SSE 的 `data:` 不能包含裸换行，多行内容需拆成多个 `data:` 行，前端会自动用换行拼接。
- 若 `data` 是 JSON，前端需自行 `JSON.parse`，注意转义。
- 事件名大小写敏感，前后端必须约定一致。

## 4.5 生产建议

- 为每个事件定义稳定的 JSON Schema，避免前后端字段漂移。
- `done` 事件附带统计信息，便于前端展示耗时与成本。

# 5. 取消与中断处理

## 5.1 结论先行

客户端断开时，必须**取消上游模型调用**，否则会持续消耗 token 与连接资源。

## 5.2 机制拆解

Reactor 在订阅取消时会自动向下游传播取消信号。WebFlux 检测到连接关闭后，会取消订阅，从而触发上游（如 HTTP 客户端到模型网关）的取消。但**前提是底层 HTTP 客户端支持取消**，部分客户端会继续把响应读完。

```
浏览器关闭连接
      │
      ▼
WebFlux 检测断开 → 取消订阅
      │
      ▼
Reactor 传播 cancel 信号
      │
      ▼
模型 HTTP 客户端中断请求（若支持）
```

## 5.3 代码示例

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestParam String message) {
    return chatClient.prompt()
            .user(message)
            .stream()
            .content()
            .doOnCancel(() -> log.info("客户端断开，取消生成"))
            .doOnError(e -> log.error("流式异常", e));
}
```

## 5.4 边界与坑

- `doOnCancel` 只做日志，真正取消依赖客户端实现；需确认所用 HTTP 客户端（如 Reactor Netty、OkHttp）支持中断。
- 服务端若把生成放入独立线程池，需手动向该线程传递取消信号。

## 5.5 生产建议

- 监控取消率，异常升高往往意味着前端或网关存在问题。
- 对长耗时生成设置最大时长兜底，避免泄漏。

# 6. 流式 RAG

## 6.1 结论先行

流式 RAG 的核心是**先同步检索，再流式生成**；先发检索状态与来源，再逐字输出答案，体验最佳。

## 6.2 阶段编排

```
① 收到问题
② 检索文档（同步，几百毫秒）
③ 发送 event: sources（引用来源）
④ 拼接 Prompt，流式生成
⑤ 逐字发送 event: message
⑥ 发送 event: done
```

## 6.3 代码示例

```java
@GetMapping(value = "/rag/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> ragStream(@RequestParam String q) {
    return Flux.defer(() -> {
        // ① 先检索
        List<Document> docs = retriever.retrieve(q);
        String context = buildContext(docs);

        // ② 发送来源
        Flux<ServerSentEvent<String>> sources = Flux.just(
                ServerSentEvent.<String>builder()
                        .event("sources")
                        .data(toJson(docs))
                        .build());

        // ③ 流式生成
        Flux<ServerSentEvent<String>> answer = chatClient.prompt()
                .user(buildPrompt(context, q))
                .stream()
                .content()
                .map(t -> ServerSentEvent.<String>builder()
                        .event("message").data(t).build());

        return sources.concatWith(answer);
    });
}
```

## 6.4 边界与坑

- `Flux.defer` 保证每次订阅才执行检索，避免结果被缓存复用。
- 检索是阻塞操作，应放在 `boundedElastic` 调度器上，避免阻塞事件循环。
- 若检索耗时较长，可先发 `thinking` 事件提示"正在检索"。

## 6.5 生产建议

- 来源与答案分开发送，前端可先展示引用卡片。
- 对检索失败做降级：无来源时直接生成，并标注"未使用知识库"。

# 7. Agent 流式

## 7.1 结论先行

Agent 的多步执行（规划、工具调用、观察）必须反馈进度，否则用户面对长时间静默会以为系统卡死。

## 7.2 机制拆解

用 `Sinks.Many`（多值发射器）在后台线程中发射事件，控制器返回 `sink.asFlux()`。每完成一步就 `tryEmitNext`，最后 `tryEmitComplete` 或 `tryEmitError`。

```
Agent 线程                     响应式流
   │                              │
   ├─ thinking  ───────────────▶ 事件
   ├─ tool_call ───────────────▶ 事件
   ├─ tool_result ─────────────▶ 事件
   ├─ ... 循环 N 步 ...
   └─ done / error ────────────▶ 完成/错误
```

## 7.3 代码示例

```java
public Flux<AgentEvent> runStreaming(String task) {
    Sinks.Many<AgentEvent> sink = Sinks.many().unicast().onBackpressureBuffer();

    executor.submit(() -> {
        try {
            sink.tryEmitNext(AgentEvent.thinking("正在规划..."));

            for (int step = 0; step < MAX_STEPS; step++) {
                ToolCall call = decide(state);
                sink.tryEmitNext(AgentEvent.toolCall(call.name()));

                ToolResult result = execute(call);
                sink.tryEmitNext(AgentEvent.toolResult(call.name(), result.summary()));
            }
            sink.tryEmitNext(AgentEvent.done(finalAnswer));
            sink.tryEmitComplete();
        } catch (Exception e) {
            sink.tryEmitError(e);
        }
    });

    return sink.asFlux();
}
```

## 7.4 边界与坑

- `tryEmitNext` 可能失败（返回 `EmitResult`），高并发下需检查并处理 `FAIL_OVERFLOW`。
- 后台线程必须自行捕获异常并 `tryEmitError`，否则流永远不结束。
- Agent 步骤过多会稀释流式收益，应设置最大步数。

## 7.5 生产建议

- 为每个工具调用附带耗时，便于前端展示进度。
- 长任务考虑持久化中间状态，支持断线后恢复。

# 8. 前端对接

## 8.1 结论先行

`EventSource` 是接入 SSE 最简单的方式，但它只支持 GET 且无法自定义请求头；需要 POST 或自定义鉴权时，应改用 `fetch` + `ReadableStream`。

## 8.2 两种接入方式

### 8.2.1 EventSource

```javascript
const es = new EventSource('/api/chat/stream?message=' + encodeURIComponent(msg));

es.addEventListener('message', e => {
    document.getElementById('output').textContent += e.data;
});

es.addEventListener('done', e => {
    es.close();
});

es.addEventListener('error', e => {
    console.error('流式错误', e);
    es.close();
});
```

### 8.2.2 fetch + ReadableStream

```javascript
const resp = await fetch('/api/chat/stream', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'Authorization': token },
  body: JSON.stringify({ message })
});

const reader = resp.body.getReader();
const decoder = new TextDecoder();
while (true) {
  const { value, done } = await reader.read();
  if (done) break;
  append(decoder.decode(value, { stream: true }));
}
```

## 8.3 对比

| 维度 | EventSource | fetch + ReadableStream |
|------|-------------|------------------------|
| 请求方法 | 仅 GET | 任意方法 |
| 自定义头 | 不支持 | 支持 |
| 自动重连 | 内置 | 需手动实现 |
| 浏览器兼容 | 好 | 现代浏览器 |

## 8.4 边界与坑

- SSE 自带断线重连，但**重连会丢失已生成内容**，需前端自行累积与去重。
- `EventSource` 无法携带 `Authorization` 头，鉴权通常靠 Cookie 或 URL 参数（后者有泄漏风险）。

## 8.5 生产建议

- 生产环境优先 `fetch` + 流式读取，便于统一鉴权与错误处理。
- 前端对每个 chunk 做增量渲染，避免整段重排。

# 9. 网关与代理配置

## 9.1 结论先行

**流式被缓冲是最常见的坑**：网关若开启响应缓冲，会等整个响应完成才下发，流式效果完全消失。

## 9.2 机制拆解

反向代理默认会缓冲上游响应以便计算 `Content-Length`、压缩或缓存。对流式响应，缓冲会把"逐块下发"退化为"一次性下发"。

```
无缓冲：上游 chunk ─▶ 代理 ─▶ 浏览器（立即）
有缓冲：上游 chunk ─▶ 代理缓存 ─▶ ... ─▶ 全部完成后一次性下发
```

## 9.3 配置示例

### 9.3.1 Nginx

```nginx
location /api/chat/stream {
    proxy_pass http://backend;
    proxy_buffering off;
    proxy_cache off;
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    chunked_transfer_encoding off;
    proxy_read_timeout 300s;
}
```

### 9.3.2 通用配置项

| 组件 | 配置 |
|------|------|
| Nginx | `proxy_buffering off` |
| CDN | 关闭缓冲、关闭压缩聚合 |
| 负载均衡 | 支持长连接、读超时足够长 |
| 应用 | 禁用响应压缩（或排除流式路径） |

## 9.4 边界与坑

- 某些云负载均衡默认 60s 空闲超时，会切断长连接。
- 开启 gzip 压缩可能导致缓冲，需排除 SSE 路径。
- HTTP/2 下 `chunked_transfer_encoding` 无意义，注意区分协议版本。

## 9.5 生产建议

- 上线前用 `curl -N` 验证流式是否真正逐块下发。
- 为流式路径单独设置路由与超时，避免影响普通接口。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 代理缓冲 | 关闭 buffering |
| 超时过短 | 读超时 ≥ 5 分钟 |
| 客户端断开未取消 | 确认取消传播 |
| 无错误事件 | 定义 error 事件 |
| 无结束标记 | 定义 done 事件 |
| 长连接占资源 | 连接数上限 + 心跳 |
| 阻塞 IO | 切换 boundedElastic |

## 10.2 监控指标

| 指标 | 含义 |
|------|------|
| 首字节时间 | 反映模型与网关延迟 |
| 活跃 SSE 连接数 | 容量与泄漏观测 |
| 取消率 | 前端/网关健康度 |
| 错误事件率 | 生成失败比例 |

## 10.3 容量与心跳

- 设置单实例最大 SSE 连接数，超限快速失败。
- 空闲时发送注释行心跳（`: ping`）保持连接，避免被中间设备断开。

# 11. 小结

- 流式不减少总耗时，但大幅改善体感。
- SSE 是 LLM 流式的标准方案，WebSocket 用于双向交互。
- 定义清晰的事件协议（start/message/done/error）。
- 客户端断开要取消上游调用，注意确认取消传播。
- 网关必须关闭缓冲，否则看不到流式效果。
- Agent 与 RAG 都要反馈中间进度。
