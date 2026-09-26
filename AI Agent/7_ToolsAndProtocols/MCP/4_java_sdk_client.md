# 1. 依赖与客户端概览

结论先行：MCP Java Client（客户端）与 Server 共用同一个 `mcp` 核心包，负责连接一个或多个 MCP Server，并把 Server 暴露的工具、资源、提示翻译成宿主应用（如 Spring AI、LangChain4j）可用的形式。

## 1.1 Maven 依赖

### 1.1.1 直接依赖

```xml
<dependency>
    <groupId>io.modelcontextprotocol.sdk</groupId>
    <artifactId>mcp</artifactId>
</dependency>
```

### 1.1.2 BOM 方式

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.modelcontextprotocol.sdk</groupId>
            <artifactId>mcp-bom</artifactId>
            <version>0.18.3</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 1.2 客户端能力矩阵

| 能力 | 方法 | 说明 |
|------|------|------|
| 发现工具 | `listTools` | 返回工具清单 |
| 调用工具 | `callTool` | 执行并返回结果 |
| 读取资源 | `readResource` | 按 URI 读取 |
| 获取提示 | `getPrompt` | 拉取提示模板 |
| 订阅变更 | notifications | 工具/资源列表变化 |

## 1.3 同步与异步

| 模式 | 入口 | 适用 |
|------|------|------|
| 同步 | `McpClient.sync(transport)` | 阻塞式调用、简单集成 |
| 异步 | `McpClient.async(transport)` | 响应式、高并发 |

## 1.4 客户端工作流程

结论先行：一次完整的 MCP 交互可拆成"握手、发现、执行"三阶段，理解它有助于定位问题出在哪一层。

```
[握手]  initialize  -> 能力协商 -> initialized 通知
[发现]  tools/list / resources/list / prompts/list
[执行]  tools/call / resources/read / prompts/get
[订阅]  notifications/*_list_changed  (可选)
```

| 阶段 | 失败典型 | 排查方向 |
|------|----------|----------|
| 握手 | 超时、版本不兼容 | 命令/URL、协议版本 |
| 发现 | 清单为空 | 权限、Server 注册 |
| 执行 | isError、参数错误 | Schema、业务逻辑 |

# 2. 连接 stdio Server

结论先行：stdio 客户端会以子进程方式启动 Server，并与其标准输入输出交换 JSON-RPC 消息，适合本地工具（文件系统、Git 等）。

## 2.1 机制说明

```
McpClient(宿主)          Server(子进程)
    |  spawn + 管道          |
    |----------------------->|
    |  initialize            |
    |----------------------->|
    |  initialize result     |
    |<-----------------------|
    |  tools/list            |
    |----------------------->|
    |  tools/call            |
    |----------------------->|
    |  result                |
    |<-----------------------|
```

## 2.2 完整代码

```java
// 启动子进程作为 Server
McpClient client = McpClient.sync(
        new StdioClientTransport(
                new ServerParameters("npx",
                        List.of("-y", "@modelcontextprotocol/server-filesystem", "D:/workspace"),
                        Map.of())))
        .requestTimeout(Duration.ofSeconds(30))
        .clientInfo(new Implementation("my-app", "1.0.0"))
        .build();

// 初始化
client.initialize();
```

## 2.3 边界与坑

- 子进程环境变量需要显式传入，默认不继承全部宿主变量。
- 进程泄漏：Client 不关闭时子进程可能常驻。
- 启动失败（命令不存在）表现为初始化超时，需要捕获并给出可读错误。
- Windows 下 `npx` 可能是 `npx.cmd`，必要时指定完整路径。

## 2.4 生产建议

- 固定 Server 命令与版本，避免"今天能连、明天连不上"。
- 为子进程设置工作目录与最小环境变量。
- 监控子进程存活，异常退出时触发重连。

# 3. 连接远程 Server

结论先行：远程 Server 优先使用 Streamable HTTP；鉴权、超时、重连都要在传输层与客户端层同时配置。

## 3.1 传输对比

| 传输 | 建连方式 | 备注 |
|------|----------|------|
| stdio | 子进程 | 本地、无鉴权 |
| SSE | 长连接 | 历史方案 |
| Streamable HTTP | 单端点 | 推荐 |

## 3.2 Streamable HTTP 实现

```java
// Streamable HTTP
HttpClientStreamableHttpTransport transport =
        HttpClientStreamableHttpTransport.builder("http://localhost:8080")
                .endpoint("/mcp")
                .build();

McpClient client = McpClient.sync(transport)
        .requestTimeout(Duration.ofSeconds(30))
        .build();

client.initialize();
```

## 3.3 鉴权与自定义请求头

### 3.3.1 自定义请求头

```java
// 伪代码：为每个请求注入凭证
HttpClientStreamableHttpTransport transport =
        HttpClientStreamableHttpTransport.builder("https://api.example.com")
                .endpoint("/mcp")
                .customizeRequest(req -> req.header(
                        "Authorization", "Bearer " + tokenProvider.get()))
                .build();
```

### 3.3.2 OAuth 说明

- 远程 Server 若启用 OAuth（开放授权），Client 需在初始化前完成令牌获取。
- 令牌过期要能自动刷新，避免长会话中途失效。
- 不要把令牌写入日志。

## 3.4 边界与坑

- 网关超时通常短于业务超时，需对齐两处配置。
- 多实例 Server 若使用会话状态，需要粘性路由或共享存储。
- 证书校验不可关闭，生产环境禁用自签名跳过。

# 4. 列出与调用工具

结论先行：工具调用是"先发现、后执行"两段式；调用结果通过 `isError` 区分业务失败与成功，Client 必须显式处理。

## 4.1 调用时序

```
listTools  -> 得到工具清单与 inputSchema
callTool   -> 传入 name + arguments
result     -> content + isError
```

## 4.2 完整代码

```java
// ① 列出工具
ListToolsResult tools = client.listTools();
tools.tools().forEach(t ->
        System.out.println(t.name() + ": " + t.description()));

// ② 调用工具
CallToolResult result = client.callTool(
        new CallToolRequest("read_file", Map.of("path", "/data/a.txt")));

if (result.isError()) {
    System.err.println("调用失败: " + extractText(result));
} else {
    System.out.println(extractText(result));
}
```

## 4.3 边界与坑

- 参数必须匹配 Server 的 `inputSchema`，类型错误会被 Server 拒绝。
- 返回内容可能是多种类型（文本、图片），不能假设一定是 `TextContent`。
- 空结果与错误结果语义不同，不能都当成"无数据"。

## 4.4 生产建议

- 缓存 `listTools` 结果并监听变更，减少重复发现开销。
- 对参数做本地预校验，减少无效网络往返。
- 记录每次调用的耗时与结果大小，便于容量规划。

# 5. 读取资源与提示

结论先行：资源与提示是工具之外的补充能力；资源用于读取只读数据，提示用于获取预置模板。

## 5.1 资源读取

```java
// 列出
ListResourcesResult resources = client.listResources();
resources.resources().forEach(r ->
        System.out.println(r.uri() + " - " + r.name()));

// 读取
ReadResourceResult result = client.readResource(
        new ReadResourceRequest("file:///config"));
result.contents().forEach(c -> {
    if (c instanceof TextResourceContents text) {
        System.out.println(text.text());
    }
});
```

## 5.2 提示获取

```java
ListPromptsResult prompts = client.listPrompts();
prompts.prompts().forEach(p -> System.out.println(p.name()));

GetPromptResult prompt = client.getPrompt(
        new GetPromptRequest("code-review", Map.of("language", "java")));
prompt.messages().forEach(m ->
        System.out.println(m.role() + ": " + extractText(m.content())));
```

## 5.3 边界与坑

- 资源 URI 需要转义，含空格或中文时尤其注意。
- 提示参数缺失应显式报错。
- 大资源要做流式或分块处理。

## 5.4 生产建议

- 资源读取结果按 URI 做短期缓存，注意失效策略。
- 提示模板的变更要能热更新。
- 对资源大小设置上限，防止拉取超大内容拖垮宿主。

# 6. 将 MCP 工具接入模型

结论先行：MCP 工具不能直接被大模型识别，必须桥接成宿主框架的工具格式（Spring AI 的 `ToolCallback` 或 LangChain4j 的 `ToolSpecification`）。

## 6.1 桥接原理

```
MCP Tool(schema)  --->  模型工具定义(schema)
模型产生 tool_call --->  callMcpTool  --->  MCP callTool
MCP result        --->  回填为工具消息  --->  模型继续生成
```

## 6.2 Spring AI 桥接

### 6.2.1 转 ToolCallback

```java
public class McpToolBridge {

    private final McpClient mcpClient;

    // ① 把 MCP 工具转成 Spring AI 的 ToolCallback
    public List<ToolCallback> toToolCallbacks() {
        return mcpClient.listTools().tools().stream()
                .map(tool -> FunctionToolCallback
                        .builder(tool.name(), (Map<String, Object> args) ->
                                callMcpTool(tool.name(), args))
                        .description(tool.description())
                        .inputType(Map.class)
                        .build())
                .toList();
    }

    private String callMcpTool(String name, Map<String, Object> args) {
        CallToolResult result = mcpClient.callTool(
                new CallToolRequest(name, args));

        if (result.isError()) {
            return "工具执行失败：" + extractText(result);
        }
        return extractText(result);
    }
}
```

### 6.2.2 接入 ChatClient

```java
// 接入 ChatClient
List<ToolCallback> mcpTools = bridge.toToolCallbacks();

String answer = chatClient.prompt()
        .user("列出 D:/workspace 下的文件")
        .toolCallbacks(mcpTools)
        .call()
        .content();
```

## 6.3 LangChain4j 桥接

```java
public List<ToolSpecification> toToolSpecifications() {
    return mcpClient.listTools().tools().stream()
            .map(tool -> ToolSpecification.builder()
                    .name(tool.name())
                    .description(tool.description())
                    .parameters(toJsonSchema(tool.inputSchema()))
                    .build())
            .toList();
}
```

```java
// 伪代码：把调用路由到 MCP
ToolExecutor executor = (request, memoryId) -> {
    CallToolResult r = mcpClient.callTool(
            new CallToolRequest(request.name(), request.arguments()));
    return r.isError() ? "失败：" + extractText(r) : extractText(r);
};
```

## 6.4 桥接对照表

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| 载体类型 | `ToolCallback` | `ToolSpecification` |
| 参数类型 | `inputType` | `parameters(JsonSchema)` |
| 执行入口 | `FunctionToolCallback` | `ToolExecutor` |
| 注册方式 | `.toolCallbacks(...)` | 注入工具规格与执行器 |

## 6.5 边界与坑

- 桥接会丢失 MCP 的结构化内容类型，通常退化为文本。
- 工具名在不同框架中可能有字符限制（如不允许点号）。
- 模型侧的超时与 MCP 侧超时要分别设置，避免双重等待。

## 6.6 生产建议

- 桥接层做统一日志，记录模型意图与实际调用参数的差异。
- 对高风险工具（写操作）增加人工确认或二次校验。
- 桥接失败要降级为"无工具"模式而非中断对话。

# 7. 连接生命周期管理

结论先行：Client 应当复用而非每次调用都新建，并集中管理关闭与变更通知，否则会泄漏连接或使用过期工具清单。

## 7.1 复用与关闭

```java
@Component
public class McpClientManager implements AutoCloseable {

    private final Map<String, McpClient> clients = new ConcurrentHashMap<>();

    public McpClient getOrConnect(String serverName, McpTransport transport) {
        return clients.computeIfAbsent(serverName, name -> {
            McpClient client = McpClient.sync(transport)
                    .requestTimeout(Duration.ofSeconds(30))
                    .build();
            client.initialize();
            return client;
        });
    }

    // 处理工具列表变更通知
    public void onToolsChanged(String serverName, Runnable callback) {
        // 注册 notifications/tools/list_changed 处理
    }

    @Override
    public void close() {
        clients.values().forEach(McpClient::close);
    }
}
```

## 7.2 变更通知处理

| 通知 | 含义 | 处理 |
|------|------|------|
| tools/list_changed | 工具列表变化 | 刷新工具缓存 |
| resources/list_changed | 资源列表变化 | 刷新资源缓存 |
| prompts/list_changed | 提示列表变化 | 刷新提示缓存 |

## 7.3 边界与坑

- `computeIfAbsent` 内的初始化若阻塞，可能拖住其他线程。
- 变更通知是异步的，刷新缓存要注意并发安全。
- 关闭顺序应先停用调用再关闭连接。

## 7.4 生产建议

- 用 `try-with-resources` 或容器生命周期统一关闭。
- 为每个 Server 维护独立的连接状态与重连计数。
- 变更通知触发缓存刷新时，保留旧快照直到新快照就绪。

# 8. 错误处理

结论先行：MCP 调用错误分三层：协议错误（`McpError`）、连接错误（IO/超时）、业务错误（`isError`）；三者处理策略不同。

## 8.1 错误分类

| 类别 | 表现 | 处理 |
|------|------|------|
| 业务错误 | isError=true | 回填文本给模型 |
| 协议错误 | McpError + code | 记录并提示 |
| 连接错误 | IOException/超时 | 重试/熔断 |

## 8.2 完整代码

```java
public CallToolResult safeCall(String name, Map<String, Object> args) {
    try {
        return mcpClient.callTool(new CallToolRequest(name, args));
    } catch (McpError e) {
        log.error("MCP 调用失败: {} code={}", name, e.getJsonRpcError().code());
        return new CallToolResult(
                List.of(new TextContent("工具调用失败：" + e.getMessage())), true);
    } catch (Exception e) {
        log.error("MCP 连接异常: {}", name, e);
        return new CallToolResult(
                List.of(new TextContent("工具服务不可用")), true);
    }
}
```

## 8.3 边界与坑

- 对连接错误做指数退避重试，对协议错误通常不应盲目重试。
- 重试要有上限与熔断，避免雪崩。
- 失败信息回填给模型时，避免诱导模型反复重试同一错误。

## 8.4 重试与熔断示例

```java
// 伪代码：带退避与熔断的调用
public CallToolResult callWithRetry(String name, Map<String, Object> args) {
    int attempt = 0;
    while (true) {
        try {
            return mcpClient.callTool(new CallToolRequest(name, args));
        } catch (McpError e) {
            // 协议错误通常不重试
            throw e;
        } catch (Exception e) {
            if (++attempt >= MAX_ATTEMPTS || circuit.isOpen(name)) {
                throw new IllegalStateException("工具服务不可用", e);
            }
            sleepBackoff(attempt);   // 指数退避
        }
    }
}
```

## 8.5 生产建议

- 为每个 Server 维护独立熔断器，避免相互影响。
- 重试仅用于幂等工具，写操作需谨慎。
- 把失败分类暴露为指标，便于区分业务失败与基础设施故障。

# 9. 生产注意事项

结论先行：客户端侧的核心风险是"连接不可靠 + 权限无边界 + 不可观测"，需要分别用重连、裁剪、监控来应对。

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 连接泄漏 | 显式关闭 |
| 无超时 | 设置 requestTimeout |
| Server 崩溃 | 重连 + 熔断 |
| 工具列表变更 | 处理 list_changed |
| 无鉴权 | 携带凭证 |
| 工具越权 | 按用户裁剪（见 `6_security.md`） |
| 无监控 | 记录调用与耗时 |

## 9.2 可观测性

- 记录每个 Server 的调用量、错误率、P95 耗时。
- 对连接状态做健康探测。
- 对工具清单变化打点，便于排查"模型突然找不到工具"。

## 9.3 部署建议

- 客户端连接池按 Server 维度隔离。
- 远程 Server 走网关，统一鉴权与限流。
- 灰度切换 Server 版本，先验证工具清单与 Schema。

## 9.4 常见问题速查

| 症状 | 可能原因 | 处理 |
|------|----------|------|
| initialize 超时 | 命令/URL 错误 | 检查传输配置 |
| 工具清单为空 | 权限裁剪过度 | 核对授权策略 |
| 调用参数被拒 | Schema 不匹配 | 对齐 inputSchema |
| 长会话中途失败 | 令牌过期 | 自动刷新凭证 |
| 内存持续上涨 | 连接/资源未释放 | 检查关闭逻辑 |

# 10. 小结

## 10.1 要点回顾

- Java SDK 客户端支持 stdio 与 HTTP 传输。
- 能力：listTools/callTool、listResources/readResource、listPrompts/getPrompt。
- MCP 工具需桥接为模型的工具格式。
- 连接要管理生命周期（复用 + 关闭）。
- 工具列表变更要处理，调用要有错误处理。

## 10.2 落地检查清单

- [ ] 依赖版本锁定。
- [ ] 传输超时与网关超时对齐。
- [ ] 凭证注入且不落日志。
- [ ] 工具桥接后验证模型可用。
- [ ] 连接复用与优雅关闭。
- [ ] 变更通知与错误处理覆盖。
