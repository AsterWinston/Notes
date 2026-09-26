# 1. 协议基础

结论先行：MCP 建立在 JSON-RPC 2.0 之上，所有交互归结为 Request（请求）、Response（响应）、Notification（通知）三类消息。理解这三类消息与错误模型，是读懂后续一切交互的基础。

## 1.1 JSON-RPC 2.0 消息模型

```
MCP 基于 JSON-RPC 2.0：
- 请求：{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{...}}
- 响应：{"jsonrpc":"2.0","id":1,"result":{...}}
- 通知：{"jsonrpc":"2.0","method":"notifications/...","params":{...}}
```

| 字段 | 出现于 | 含义 |
|------|--------|------|
| `jsonrpc` | 所有 | 固定 `"2.0"` |
| `id` | 请求/响应 | 关联请求与响应 |
| `method` | 请求/通知 | 方法名 |
| `params` | 请求/通知 | 参数对象 |
| `result` | 响应 | 成功结果 |
| `error` | 响应 | 错误对象 |

## 1.2 请求、响应与通知

- Request：必须携带 `id`，对端必须返回 Response。
- Response：携带与请求相同的 `id`，要么 `result` 要么 `error`。
- Notification：无 `id`，不期待响应，用于列表变更等事件。

## 1.3 错误模型

| 场景 | 处理 |
|------|------|
| 协议级错误 | JSON-RPC `error` 字段（code/message/data） |
| 工具执行错误 | `tools/call` 返回 `isError: true` |

```json
{"jsonrpc":"2.0","id":1,
 "error":{"code":-32602,"message":"Invalid params"}}
```

## 1.4 边界与生产建议

- 坑：把工具业务错误当协议错误返回，会让客户端误判连接故障。
- 建议：区分"协议错误"与"工具执行错误"，前者用 `error`，后者用 `isError`。

# 2. 角色与连接

结论先行：MCP 采用 Host / Client / Server 三角色，Client 与 Server 之间是一对一有状态连接。

## 2.1 三角色

```
Host（应用）
  └── Client（每连接一个）
        ↕ 1:1 连接
      Server
```

| 角色 | 职责 |
|------|------|
| Host | 管理 Client、协调能力 |
| Client | 与单个 Server 通信 |
| Server | 提供能力 |

## 2.2 连接生命周期

连接从 `initialize` 开始，经 `initialized` 通知进入就绪态，正常交互后由 `shutdown` 或连接关闭结束。一个 Client 只服务一个 Server，连接状态与 Server 一一对应。

## 2.3 边界与生产建议

- 坑：多个 Server 共用一个 Client 会串扰能力与通知。
- 建议：连接池按 Server 维度隔离，并做健康探测。

# 3. 能力协商

结论先行：连接建立时必须交换能力（capabilities），双方只能使用对方声明的能力，未协商的能力不可调用。

## 3.1 initialize 交换

连接建立时交换能力：

```json
// Client → Server
{"method":"initialize","params":{
  "protocolVersion":"2026-07-28",
  "capabilities":{"roots":{"listChanged":true}},
  "clientInfo":{"name":"my-app","version":"1.0.0"}
}}

// Server → Client
{"result":{
  "protocolVersion":"2026-07-28",
  "capabilities":{
    "tools":{"listChanged":true},
    "resources":{"subscribe":true,"listChanged":true},
    "prompts":{"listChanged":true}
  },
  "serverInfo":{"name":"file-server","version":"1.0.0"}
}}
```

## 3.2 能力矩阵

| 能力 | 声明方 | 子能力 |
|------|--------|--------|
| tools | Server | listChanged |
| resources | Server | subscribe、listChanged |
| prompts | Server | listChanged |
| roots | Client | listChanged |
| sampling | Client | - |
| elicitation | Client | - |

## 3.3 边界与生产建议

- 坑：版本不匹配可能导致字段语义不同，必须校验 `protocolVersion`。
- 建议：能力协商失败时明确报错，而不是静默降级。

# 4. 三大原语详解

结论先行：三大原语各有独立的列表与调用方法，并支持列表变更通知。Tools 由模型调用，Resources 由应用读取，Prompts 由用户/应用获取。

## 4.1 Tools

```
tools/list   → 列出可用工具
tools/call   → 调用工具
notifications/tools/list_changed → 工具列表变更通知
```

```json
// tools/list 响应
{"result":{"tools":[
  {"name":"read_file","description":"读取文件","inputSchema":{
    "type":"object","properties":{"path":{"type":"string"}},"required":["path"]}}
]}}

// tools/call 请求
{"method":"tools/call","params":{
  "name":"read_file","arguments":{"path":"/data/a.txt"}}}

// tools/call 响应
{"result":{"content":[{"type":"text","text":"文件内容..."}],"isError":false}}
```

### 4.1.1 工具定义要点

| 要点 | 说明 |
|------|------|
| name | 唯一、语义清晰 |
| description | 决定模型是否调用 |
| inputSchema | JSON Schema，需完整 |

### 4.1.2 调用语义

`tools/call` 成功时返回 `content` 数组；执行失败时返回 `isError: true` 但仍属协议成功响应。

## 4.2 Resources

```
resources/list     → 列出资源
resources/read     → 读取资源
resources/subscribe→ 订阅变更
```

```json
{"result":{"resources":[
  {"uri":"file:///data/a.txt","name":"a.txt","mimeType":"text/plain"}
]}}
```

### 4.2.1 资源标识

资源以 `uri` 唯一标识，`mimeType` 提示内容类型。读取返回 `contents` 数组。

### 4.2.2 订阅变更

声明 `subscribe` 能力后，客户端可订阅资源，服务端在变更时发送 `notifications/resources/updated`。

## 4.3 Prompts

```
prompts/list  → 列出提示模板
prompts/get   → 获取提示（可带参数）
```

```json
{"result":{"prompts":[
  {"name":"code-review","description":"代码审查","arguments":[
    {"name":"language","required":true}]}
]}}
```

## 4.4 边界与生产建议

- 坑：`inputSchema` 不完整会导致模型生成非法参数。
- 建议：对工具参数做服务端二次校验，不信任客户端。

# 5. 服务端能力 vs 客户端能力

结论先行：能力是双向的——Server 可反向使用 Client 的 sampling、roots、elicitation 能力，这是 MCP 区别于普通 RPC 的关键。

## 5.1 双向能力表

| 方向 | 能力 | 说明 |
|------|------|------|
| Server → Client | sampling | 请求 Client 调用 LLM |
| Server → Client | roots | 获取文件系统根目录 |
| Server → Client | elicitation | 请求用户输入 |
| Client → Server | tools | 调用工具 |
| Client → Server | resources | 读取资源 |
| Client → Server | prompts | 获取提示 |

> 注意：**sampling 是反向的**——Server 可以请求 Client 的 LLM 能力。

## 5.2 sampling 示例

```java
// 伪代码：Server 请求 Client 调用 LLM
SamplingResult result = client.sample(
        SamplingRequest.builder()
                .messages(List.of(userMessage("总结这段文本")))
                .maxTokens(512)
                .build());
```

## 5.3 边界与生产建议

- 坑：sampling 会把 Server 的提示词送进 Client 的 LLM，需评估数据边界与成本。
- 建议：对 sampling 请求做速率限制与审计。

# 6. 生命周期

结论先行：MCP 连接遵循"初始化 → 就绪 → 交互 → 关闭"四阶段，任何交互都必须在就绪之后。

## 6.1 四阶段

```
① initialize        能力协商
② initialized       通知 Server 就绪
③ 正常交互          tools/list、tools/call ...
④ shutdown          优雅关闭
```

## 6.2 示例

```java
// 伪代码
client.initialize();
client.sendInitialized();
List<Tool> tools = client.listTools();
ToolResult result = client.callTool("read_file", args);
client.close();
```

### 6.2.1 Spring AI 风格

```java
// Spring AI 风格：交由框架管理生命周期（伪代码）
@Bean
public McpSyncClient mcpClient() {
    McpSyncClient client = McpClient.sync(
            new StdioClientTransport(
                    new ServerParameters("npx", List.of("server-filesystem"))))
            .requestTimeout(Duration.ofSeconds(30))
            .build();
    client.initialize();          // 内部完成 initialize/initialized
    return client;
}
```

### 6.2.2 LangChain4j 风格

```java
// LangChain4j 风格（伪代码）
McpClient client = new DefaultMcpClient.Builder()
        .transport(new StdioMcpTransport.Builder()
                .command(List.of("npx", "-y", "@modelcontextprotocol/server-filesystem"))
                .build())
        .build();
List<ToolSpecification> tools = client.listTools();
```

## 6.3 边界与生产建议

- 坑：在 `initialized` 之前调用业务方法，多数实现会直接拒绝。
- 建议：封装连接状态机，未就绪时排队或快速失败。
- 建议：客户端为每个 Server 单独持有连接，避免共享导致串扰。

# 7. 传输层对比

结论先行：四种传输各有取舍，生产远程优先 Streamable HTTP。

## 7.1 四种传输

| 传输 | 连接 | 适用 | 特点 |
|------|------|------|------|
| stdio | 父子进程 | 本地 | 简单、无网络 |
| SSE | HTTP 长连接 | 远程 | 单向流 |
| Streamable HTTP | HTTP | 生产 | 双向、可恢复 |
| Stateless | HTTP | Serverless | 无会话 |

## 7.2 选型建议

| 场景 | 推荐 |
|------|------|
| 本地工具 | stdio |
| 生产远程 | Streamable HTTP |
| Serverless | Stateless |
| 存量过渡 | SSE |

## 7.3 边界与生产建议

- 坑：SSE 单向，服务端无法主动请求客户端，反向能力受限。
- 建议：生产环境为长连接配置心跳与重连退避。

# 8. 消息类型

结论先行：消息分 Request、Response、Notification 三类，通知用于解耦的异步事件。

## 8.1 类型表

| 类型 | 说明 |
|------|------|
| Request | 需响应 |
| Response | 对 Request 的响应 |
| Notification | 不需响应（如变更通知） |

## 8.2 常见通知

| 通知 | 触发 |
|------|------|
| notifications/tools/list_changed | 工具列表变更 |
| notifications/resources/updated | 资源内容变更 |
| notifications/prompts/list_changed | 提示列表变更 |

## 8.3 边界与生产建议

- 坑：客户端未处理通知会错过能力变更，导致缓存过期。
- 建议：收到列表变更通知后刷新本地能力缓存。

# 9. 内容类型

结论先行：工具与资源可返回文本、图片、内嵌资源等多模态内容。

## 9.1 多模态内容

```json
// 工具返回内容可以是多种类型
{"content":[
  {"type":"text","text":"..."},
  {"type":"image","data":"base64...","mimeType":"image/png"},
  {"type":"resource","resource":{"uri":"file://...","text":"..."}}
]}
```

## 9.2 类型表

| type | 字段 | 说明 |
|------|------|------|
| text | text | 纯文本 |
| image | data、mimeType | Base64 图片 |
| resource | resource | 内嵌资源 |

## 9.3 边界与生产建议

- 坑：大图片走 Base64 会显著膨胀消息体，应改用资源引用。
- 建议：对内容大小设上限并做截断。

## 9.4 Java 处理示例

```java
// Spring AI 风格：遍历多模态内容（伪代码）
public String render(ToolResult result) {
    StringBuilder sb = new StringBuilder();
    for (Content c : result.content()) {
        switch (c.type()) {
            case "text" -> sb.append(c.text());
            case "image" -> sb.append("[image:").append(c.mimeType()).append("]");
            case "resource" -> sb.append(readResource(c.resource()));
            default -> sb.append("[unsupported]");
        }
    }
    return sb.toString();
}
```

# 10. 生产注意事项

结论先行：上线前需核对版本、连接、超时、内容、能力与错误六类风险。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 协议版本不匹配 | 校验并协商 |
| 连接泄漏 | 显式关闭 |
| 无超时 | 设置调用超时 |
| 大内容 | 分页/截断 |
| 能力未协商就用 | 先检查 capabilities |
| 无错误处理 | 处理 JSON-RPC error |

## 10.2 可观测性

建议记录每次请求的方法、耗时与结果码，并对连接数与错误率告警。

# 11. 错误处理与重试

结论先行：MCP 的错误分两层——协议级错误走 JSON-RPC `error`，工具执行错误走 `isError`。重试只对幂等的读操作安全。

## 11.1 错误分层

### 11.1.1 协议错误

参数非法、方法不存在、版本不匹配等，返回 `error` 对象，调用方应视为请求失败。

### 11.1.2 工具执行错误

工具内部抛错时，`tools/call` 仍返回成功响应，但 `isError` 为真，内容通常是错误说明。

```json
{"jsonrpc":"2.0","id":2,
 "result":{"content":[{"type":"text","text":"文件不存在"}],"isError":true}}
```

## 11.2 重试策略

| 场景 | 是否重试 | 说明 |
|------|----------|------|
| 读工具超时 | 可重试 | 幂等安全 |
| 写工具失败 | 谨慎 | 需幂等键 |
| 协议错误 | 不重试 | 参数问题重试无益 |
| 连接断开 | 重连后重试 | 需会话恢复 |

## 11.3 边界与生产建议

- 坑：对非幂等工具盲目重试会产生重复副作用。
- 建议：为重试引入指数退避与最大次数，并记录重试原因。

# 12. 安全与鉴权

结论先行：MCP 连接需要传输安全、调用方授权与最小权限三层防护。

## 12.1 传输安全

远程传输必须走 TLS；stdio 场景限制子进程文件与网络权限。

## 12.2 授权与最小权限

| 措施 | 说明 |
|------|------|
| 调用方认证 | Token/OAuth2 |
| 工具级授权 | 按调用方授予工具子集 |
| 参数校验 | 服务端二次校验 |
| 审计 | 记录调用链 |

## 12.3 边界与生产建议

- 坑：把内部 Server 直接暴露公网且无鉴权，等于开放内部能力。
- 建议：Server 前置网关，统一鉴权、限流与审计。

# 13. 小结

- MCP 基于 JSON-RPC 2.0，包含 Request/Response/Notification。
- 三大原语：Tools（模型调用）、Resources（读数据）、Prompts（模板）。
- 连接时进行能力协商。
- 传输方式：stdio、SSE、Streamable HTTP、Stateless。
- sampling 是 Server 反向使用 Client 的 LLM 能力。
