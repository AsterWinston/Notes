# 1. 为什么流式

结论先行：流式（streaming）不减少总耗时，但把“等待 5 秒后一次性出字”变成“0.2 秒出首字、逐字呈现”，显著改善体感，是对话类应用的标配。

## 1.1 体感与总耗时

### 1.1.1 首字延迟

首字延迟（TTFT，Time To First Token）是用户感知速度的核心指标。

```text
非流式：等待 5 秒 -> 一次性返回
流式：  0.2 秒出首字 -> 逐字返回
```

### 1.1.2 感知速度

| 指标 | 非流式 | 流式 |
|------|--------|------|
| 首字延迟 | 等于总耗时 | 极短 |
| 感知速度 | 慢 | 快 |
| 总耗时 | 相同 | 相同 |

## 1.2 适用场景

### 1.2.1 对话类

聊天、助手、代码补全等交互场景应默认开启流式。

### 1.2.2 批处理

离线抽取、批量分类无需流式，反而增加解析复杂度。

## 1.3 代价

结论先行：流式以“连接资源与实现复杂度”换取“体感”。并非所有场景都值得。

- 服务端需维持长连接，占用连接资源。
- 错误处理更复杂：错误可能在流中途发生。
- 客户端需增量渲染，且重连会丢失已生成内容。

| 维度 | 非流式 | 流式 |
|------|--------|------|
| 连接占用 | 短 | 长 |
| 错误处理 | 一次判定 | 中途可能出错 |
| 客户端复杂度 | 低 | 中 |
| 体感 | 差 | 好 |
| 适合 | 批处理 | 交互 |

# 2. 协议格式

结论先行：LLM 流式普遍采用 SSE（Server-Sent Events，服务器推送事件）。它以 `data:` 行推送增量，以 `[DONE]` 结束。

## 2.1 SSE 基础

### 2.1.1 报文格式

```text
data: 第一段

data: 第二段

data: [DONE]
```

每个事件以空行分隔；`data:` 后的内容为该事件的载荷。

### 2.1.2 结束标记

`[DONE]` 是 OpenAI 约定的结束标记，收到后应关闭流。

## 2.2 OpenAI 兼容流

### 2.2.1 delta 结构

```text
data: {"choices":[{"delta":{"content":"你"},"index":0}]}

data: {"choices":[{"delta":{"content":"好"},"index":0}]}

data: {"choices":[{"delta":{},"finish_reason":"stop"}],"usage":{...}}

data: [DONE]
```

每个 chunk 只携带**增量**（delta），而非完整文本。

### 2.2.2 usage 与 finish_reason

| 特点 | 说明 |
|------|------|
| 逐块返回 delta | 只有增量内容 |
| `[DONE]` 结束标记 | 表示流结束 |
| usage 在最后 | 部分厂商只在最后返回 |
| finish_reason | 在最后一个有效 chunk |

## 2.3 解析要点

| 坑 | 处理 |
|----|------|
| chunk 跨行 | 按空行切分事件 |
| `delta.content` 缺失 | 工具调用时可能只有 `tool_calls` |
| 空 delta | 忽略而非报错 |
| `[DONE]` 非 JSON | 先判断标记再解析 |
| 多行 data | 拼接后再解析 |

一个健壮的 SSE 解析骨架：

```java
void parse(InputStream in) throws IOException {
    BufferedReader reader = new BufferedReader(new InputStreamReader(in, StandardCharsets.UTF_8));
    String line;
    while ((line = reader.readLine()) != null) {
        if (line.isBlank() || !line.startsWith("data:")) {
            continue;                       // 忽略注释行与空行
        }
        String payload = line.substring("data:".length()).trim();
        if ("[DONE]".equals(payload)) {
            break;                          // 结束标记
        }
        StreamChunk chunk = objectMapper.readValue(payload, StreamChunk.class);
        String delta = chunk.firstContentOrNull();
        if (delta != null && !delta.isEmpty()) {
            onDelta(delta);
        }
    }
}
```

# 3. Spring AI 流式

结论先行：Spring AI 用 Reactor 的 `Flux` 表达流，`stream().content()` 给出纯文本流，`stream().chatResponse()` 给出带元数据的对象流。

## 3.1 基础流

### 3.1.1 content 流

```java
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> stream(@RequestParam String message) {
    return chatClient.prompt()
            .user(message)
            .stream()
            .content();
}
```

### 3.1.2 chatResponse 流

```java
Flux<ChatResponse> flux = chatClient.prompt()
        .user(message)
        .stream()
        .chatResponse();

flux.subscribe(resp -> {
    String delta = resp.getResult().getOutput().getText();
    // ...
});
```

## 3.2 元数据与选项

```java
Flux<ChatResponse> flux = chatClient.prompt()
        .user(message)
        .options(ChatOptions.builder().temperature(0.7).build())
        .stream()
        .chatResponse();

// 取用量（通常在最后一块）
flux.doOnNext(r -> {
    if (r.getMetadata().getUsage() != null) {
        log.info("tokens={}", r.getMetadata().getUsage().getTotalTokens());
    }
});
```

# 4. LangChain4j 流式

结论先行：LangChain4j 用 `TokenStream` 暴露回调式流，适合命令式代码；Spring AI 用 `Flux`，适合响应式代码。

## 4.1 TokenStream

### 4.1.1 回调

```java
TokenStream stream = assistant.chat("讲个笑话");

stream.onPartialResponse(System.out::print)
      .onCompleteResponse(r -> System.out.println("\n完成"))
      .onError(Throwable::printStackTrace)
      .start();
```

### 4.1.2 生命周期

| 回调 | 触发时机 |
|------|----------|
| `onPartialResponse` | 每个增量 |
| `onCompleteResponse` | 正常结束 |
| `onError` | 异常结束 |
| `onRetrieved` | RAG 检索到内容 |

### 4.1.3 取消与超时

```java
TokenStream stream = assistant.chat("写一篇长文");
stream.onPartialResponse(System.out::print)
      .onCompleteResponse(r -> log.info("完成"))
      .start();

// 需要中断时
stream.cancel();
```

## 4.2 与 Spring AI 对比

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 模型 | Flux（响应式） | TokenStream（回调） |
| 取消 | 订阅取消 | `cancel()` |
| 组合 | 操作符丰富 | 命令式 |
| 适合 | WebFlux 应用 | 命令式应用 |

# 5. SSE 事件协议设计

结论先行：不要直接把模型 delta 透传给前端，而应定义清晰的事件协议（start/message/done/error），便于前端处理与排障。

## 5.1 事件类型

### 5.1.1 start/message/done/error

| 事件 | 用途 |
|------|------|
| `start` | 会话开始 |
| `message` | 内容增量 |
| `done` | 正常结束 |
| `error` | 异常结束 |

### 5.1.2 事件载荷

| 事件 | data 示例 |
|------|-----------|
| `start` | `{"conversationId":"c1"}` |
| `message` | `你` |
| `done` | `[DONE]` |
| `error` | `{"code":"RATE_LIMIT"}` |

## 5.2 实现

### 5.2.1 服务端

```java
@GetMapping(value = "/chat/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> stream(@RequestParam String message) {
    return Flux.concat(
            // 开始事件
            Flux.just(ServerSentEvent.<String>builder()
                    .event("start").data("{}").build()),
            // 内容事件
            chatClient.prompt().user(message).stream().content()
                    .map(t -> ServerSentEvent.<String>builder()
                            .event("message").data(t).build()),
            // 结束事件
            Flux.just(ServerSentEvent.<String>builder()
                    .event("done").data("[DONE]").build()))
            .onErrorResume(e -> Flux.just(ServerSentEvent.<String>builder()
                    .event("error").data(e.getMessage()).build()));
}
```

### 5.2.2 心跳

长连接可能被中间层空闲超时切断，需定期发送注释行心跳：

```java
Flux<ServerSentEvent<String>> heartbeat =
        Flux.interval(Duration.ofSeconds(15))
            .map(i -> ServerSentEvent.<String>builder().comment("ping").build());
```

# 6. 中断处理

结论先行：客户端断开后必须取消上游 LLM 调用，否则会继续计费并占用线程。

## 6.1 取消传播

### 6.1.1 Reactor 取消

```java
return chatClient.prompt()
        .user(message)
        .stream()
        .content()
        .doOnCancel(() -> log.info("客户端断开，取消生成"))
        .doOnTerminate(() -> log.info("流结束"));
```

> Reactor 订阅取消会传播取消信号，但**要确认底层 HTTP 客户端支持取消**。

### 6.1.2 底层客户端

| 客户端 | 取消支持 |
|--------|----------|
| Reactor Netty | 支持 |
| JDK HttpClient | 需自行关闭 |
| OkHttp | 支持 |

## 6.2 资源回收

| 资源 | 回收动作 |
|------|----------|
| 上游连接 | 取消订阅即关闭 |
| 会话状态 | 在 `doFinally` 清理 |
| 计量 | 记录已生成 Token |

```java
.doFinally(signal -> {
    log.info("流终止: {}", signal);
    metrics.recordStreamEnd(signal);
})
```

# 7. 流式 RAG

结论先行：RAG（Retrieval-Augmented Generation，检索增强生成）先做同步检索，再流式生成，并把“来源”作为单独事件先行下发。

## 7.1 先检索再生成

### 7.1.1 流程

```text
用户问题
   │
   ▼
 检索(同步) ──▶ sources 事件
   │
   ▼
 组装 Prompt
   │
   ▼
 流式生成 ──▶ message 事件 ... done 事件
```

### 7.1.2 代码

```java
@GetMapping(value = "/rag/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> ragStream(@RequestParam String q) {
    return Flux.defer(() -> {
        // ① 先检索
        List<Document> docs = retriever.retrieve(q);

        // ② 发送来源
        Flux<ServerSentEvent<String>> sources = Flux.just(
                ServerSentEvent.<String>builder()
                        .event("sources").data(toJson(docs)).build());

        // ③ 流式生成
        Flux<ServerSentEvent<String>> answer = chatClient.prompt()
                .user(buildPrompt(docs, q))
                .stream().content()
                .map(t -> ServerSentEvent.<String>builder()
                        .event("message").data(t).build());

        return sources.concatWith(answer);
    });
}
```

## 7.2 来源事件与顺序

用 `Flux.defer` 保证检索在订阅时才执行，避免提前触发副作用。

| 顺序 | 事件 | 说明 |
|------|------|------|
| 1 | `sources` | 来源文档 |
| 2 | `message` | 生成增量 |
| 3 | `done` | 结束 |

# 8. 前端对接

结论先行：浏览器原生 `EventSource` 最简单，但只支持 GET 且会自动重连；需要 POST 或精细控制时用 `fetch` + 流读取。

## 8.1 EventSource

### 8.1.1 基本用法

```javascript
const es = new EventSource('/chat/stream?message=' + encodeURIComponent(msg));

es.addEventListener('message', e => {
    output.textContent += e.data;
});

es.addEventListener('done', () => es.close());

es.addEventListener('error', () => {
    console.error('流式错误');
    es.close();
});
```

### 8.1.2 局限

| 局限 | 说明 |
|------|------|
| 仅 GET | 无法传大 body |
| 自动重连 | 重连会重复请求 |
| 无法自定义头 | 鉴权不便 |

> `EventSource` 默认会自动重连，但**重连会丢失已生成内容**，需前端处理。

### 8.1.3 重连处理

自动重连会从头请求，需前端记录 `Last-Event-ID` 或由服务端支持断点续传；否则应主动 `es.close()` 后由用户触发重试。

## 8.2 fetch + ReadableStream

```javascript
const resp = await fetch('/chat/stream', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({message: msg})
});

const reader = resp.body.getReader();
const decoder = new TextDecoder();
while (true) {
    const {done, value} = await reader.read();
    if (done) break;
    output.textContent += decoder.decode(value, {stream: true});
}
```

> `fetch` 方式可自定义请求头与 POST body，但需自行解析 SSE 帧与处理断流重试。

# 9. 网关配置

结论先行：流式最常见的“线上不流式”，根因是反向代理缓冲。必须关闭缓冲并放宽读超时。

## 9.1 缓冲问题

### 9.1.1 Nginx

```nginx
location /chat/stream {
    proxy_pass http://backend;
    proxy_buffering off;              # 关闭缓冲
    proxy_cache off;
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    chunked_transfer_encoding off;
    proxy_read_timeout 300s;          # 长连接超时
}
```

### 9.1.2 组件表

| 组件 | 配置 |
|------|------|
| Nginx | `proxy_buffering off` |
| CDN | 关闭缓冲 |
| 负载均衡 | 支持长连接 |
| 超时 | 读超时 ≥ 5 分钟 |

## 9.2 超时与长连接

### 9.2.1 分层超时

| 层 | 建议 |
|----|------|
| 网关读超时 | ≥ 300s |
| 连接空闲 | 心跳保活 |
| 连接数 | 设上限并监控 |

### 9.2.2 连接池与并发

```yaml
# Reactor Netty 连接池示例
reactor:
  netty:
    connection:
      pool:
        max-connections: 500
        acquire-timeout: 10s
```

# 10. 生产注意事项

结论先行：流式的问题多发生在“链路中间层”而非业务代码，需把代理、超时、取消、监控逐项检查。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 代理缓冲 | 关闭 buffering |
| 超时过短 | 读超时足够长 |
| 断开未取消 | 确认取消传播 |
| 无错误事件 | 定义 error 事件 |
| 无结束标记 | 定义 done 事件 |
| 长连接占资源 | 连接数上限 + 心跳 |
| 无监控 | 监控首字延迟与时长 |

## 10.2 上线检查表

- [ ] 网关/CDN 已关闭缓冲。
- [ ] 读超时 ≥ 5 分钟且有心跳。
- [ ] 客户端断开后上游调用被取消。
- [ ] 事件协议含 start/message/done/error。
- [ ] 前端处理重连与内容去重。
- [ ] 监控 TTFT 与流时长。

# 11. 小结

## 11.1 要点回顾

- 流式不减少总耗时，但显著改善体感。
- SSE 是 LLM 流式的标准方案。
- 定义清晰的事件协议（start/message/done/error）。
- 客户端断开要取消上游调用。
- 网关必须关闭缓冲，否则看不到流式效果。

## 11.2 落地建议

- 先打通“网关关闭缓冲 + 事件协议”两条链路，再优化渲染。
- 把 TTFT 与流时长纳入监控，作为体验的量化指标。
- 为长连接设置心跳与连接数上限，防止资源耗尽。
