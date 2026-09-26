# 1. 依赖引入与工程结构

结论先行：官方 Java SDK（`io.modelcontextprotocol.sdk`）用 BOM（Bill of Materials，物料清单）统一版本，工程通常只引入 `mcp` 一个核心包；Server 端三大能力（Tools 工具、Resources 资源、Prompts 提示）都通过 `McpServer` 构建器声明式注册。

## 1.1 Maven 依赖

### 1.1.1 BOM 方式（推荐）

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

<dependencies>
    <dependency>
        <groupId>io.modelcontextprotocol.sdk</groupId>
        <artifactId>mcp</artifactId>
    </dependency>
</dependencies>
```

### 1.1.2 直接指定版本

```xml
<dependency>
    <groupId>io.modelcontextprotocol.sdk</groupId>
    <artifactId>mcp</artifactId>
    <version>0.18.3</version>
</dependency>
```

> 版本请以官方最新为准；Spring AI 也提供 `spring-ai-starter-mcp-server-*`（见 `5_spring_ai_mcp.md`）。

## 1.2 工程结构与运行环境

### 1.2.1 JDK 与构建

| 项目 | 建议 |
|------|------|
| JDK | 17 或 21（LTS） |
| 构建 | Maven 3.9+ / Gradle 8+ |
| 传输 | stdio / Streamable HTTP |
| 日志 | SLF4J + Logback |
| 测试 | JUnit 5 + 内存传输 |

### 1.2.2 包结构建议

```
server/
  config/      Server 构建与传输配置
  tools/       @Tool 工具实现
  resources/   资源提供者
  prompts/     提示模板
  security/    路径与权限防护
```

## 1.3 版本与兼容性

结论先行：SDK 仍处于 0.x，API 可能在次版本间调整，因此务必锁定 BOM 版本并保留升级测试。

- 协议版本：Server 与 Client 会在初始化（initialize）阶段协商 `protocolVersion`。
- 能力协商：Server 声明的 `capabilities` 必须与实际注册的规格一致，否则 Client 调用会失败。
- 兼容策略：升级前先用内存传输跑一遍集成测试，再灰度到生产。

# 2. 最小 Server（stdio）

结论先行：stdio（Standard Input/Output，标准输入输出）是最简单的传输方式，Server 由宿主以子进程方式启动，通过标准流收发 JSON-RPC（JSON Remote Procedure Call，JSON 远程过程调用）消息。

## 2.1 整体机制

### 2.1.1 启动流程

```
Host(Client)              Server(子进程)
    |  spawn                |
    |---------------------->|  进程启动
    |  initialize (JSON-RPC)|
    |---------------------->|
    |  initialize result    |
    |<----------------------|  能力协商
    |  initialized 通知     |
    |---------------------->|
    |  tools/list           |
    |---------------------->|
    |  tools/call           |
    |---------------------->|
    |  result               |
    |<----------------------|
```

### 2.1.2 同步与异步

| 模式 | 入口 | 适用 |
|------|------|------|
| 同步 | `McpServer.sync(transport)` | 阻塞式工具、简单场景 |
| 异步 | `McpServer.async(transport)` | 高并发、非阻塞 I/O |

> 关键约束：stdio 模式下**禁止向 stdout 打印任何非协议日志**，日志必须走 stderr，否则会污染 JSON-RPC 流。

## 2.2 完整代码

```java
public class FileServer {

    public static void main(String[] args) {
        McpServer server = McpServer.sync(
                new StdioServerTransportProvider())
                .serverInfo("file-server", "1.0.0")
                .capabilities(ServerCapabilities.builder()
                        .tools(true)
                        .resources(true, true)
                        .build())
                .build();

        // 启动
        server.start();
    }
}
```

## 2.3 边界与坑

| 现象 | 原因 | 处理 |
|------|------|------|
| Client 卡住不返回 | 日志写到了 stdout | 日志改到 stderr |
| 启动即退出 | main 结束、无阻塞 | 保留 `server.start()` 或阻塞主线程 |
| 中文乱码 | 编码非 UTF-8 | 统一 UTF-8 |
| 能力声明不匹配 | capabilities 与实际规格不符 | 声明与注册保持一致 |

## 2.4 生产建议

- 用参数控制传输类型（stdio / HTTP），避免硬编码。
- 启动时做一次自检：构造 Server 成功后再进入服务循环。
- 为进程退出注册钩子，优雅关闭资源。

# 3. 注册工具（Tools）

结论先行：Tools（工具）是 Server 最重要的原语，模型通过 `tools/list` 发现工具、通过 `tools/call` 执行工具；每个工具由名称、描述和 JSON Schema（JSON 模式）参数定义构成。

## 3.1 工具原语机制

### 3.1.1 inputSchema 规范

```json
{
  "type": "object",
  "properties": {
    "path": {"type": "string", "description": "文件路径"}
  },
  "required": ["path"]
}
```

- `inputSchema` 必须是合法 JSON Schema，否则部分 Client 会拒绝加载。
- 参数名与实现里的键必须一致，SDK 不会自动做别名映射。

### 3.1.2 返回结构

| 字段 | 含义 |
|------|------|
| `content` | 内容列表，可为文本、图片等 |
| `isError` | 是否为业务错误 |

## 3.2 编程式注册

```java
McpServer server = McpServer.sync(transport)
        .serverInfo("file-server", "1.0.0")
        .capabilities(ServerCapabilities.builder().tools(true).build())
        .tools(
                new McpServerFeatures.SyncToolSpecification(
                        new Tool("read_file", "读取文本文件",
                                """
                                {
                                  "type": "object",
                                  "properties": {
                                    "path": {"type":"string","description":"文件路径"}
                                  },
                                  "required": ["path"]
                                }
                                """),
                        (exchange, args) -> {
                            String path = (String) args.get("path");
                            try {
                                String content = Files.readString(Path.of(path));
                                return new CallToolResult(
                                        List.of(new TextContent(content)), false);
                            } catch (Exception e) {
                                return new CallToolResult(
                                        List.of(new TextContent("读取失败：" + e.getMessage())),
                                        true);
                            }
                        })
        )
        .build();
```

## 3.3 注解式注册（推荐）

部分 SDK 版本与 Spring AI 支持注解方式：

```java
@Component
public class FileTools {

    @Tool(name = "read_file", description = "读取指定路径的文本文件")
    public String readFile(
            @ToolParam(description = "文件绝对路径") String path) throws IOException {
        Path p = Path.of(path).normalize();

        // 安全：限制在允许目录内
        if (!p.startsWith(ALLOWED_ROOT)) {
            throw new SecurityException("路径越界");
        }
        return Files.readString(p);
    }

    @Tool(name = "list_files", description = "列出目录下的文件")
    public List<String> listFiles(
            @ToolParam(description = "目录路径") String dir) throws IOException {
        try (Stream<Path> stream = Files.list(Path.of(dir))) {
            return stream.map(Path::getFileName)
                    .map(Path::toString)
                    .toList();
        }
    }
}
```

## 3.4 边界与坑

- 返回类型会影响序列化：`List<String>`、POJO 需要能被序列化为文本或结构化内容。
- 工具描述是模型选型的唯一依据，描述含糊会显著降低调用准确率。
- 参数缺失时不要直接 NPE，应返回 `isError=true` 的友好提示。
- 工具数量过多会挤占上下文预算，建议按场景裁剪（见第 7 节与 `4_java_sdk_client.md`）。

# 4. 注册资源（Resources）

结论先行：Resources 是"可读数据"的抽象，由 URI 标识，适合暴露配置文件、文档等只读内容，与工具的区别在于它没有副作用。

## 4.1 资源原语机制

| 概念 | 说明 |
|------|------|
| URI | 资源唯一标识，如 `file:///config` |
| MIME 类型 | 决定内容如何解释 |
| 静态/模板 | 固定 URI 或带占位符的模板 |

## 4.2 完整示例

```java
.resources(new McpServerFeatures.SyncResourceSpecification(
        new Resource("file:///config", "config", "应用配置",
                "text/plain", null),
        (exchange, request) -> {
            String content = loadConfig();
            return new ReadResourceResult(
                    List.of(new TextResourceContents(
                            request.uri(), "text/plain", content)));
        }))
```

## 4.3 边界与坑

- 资源是**只读**的：任何写入语义都应设计成工具。
- 大文件不要一次性读入内存，应做分页或大小限制。
- 模板资源需要显式声明参数，否则 Client 无法构造合法 URI。
- 敏感资源必须做权限校验，不能因为"只读"就默认安全。

# 5. 注册提示（Prompts）

结论先行：Prompts 是 Server 预置的提示模板，Client 通过 `prompts/list` 与 `prompts/get` 获取，通常用于把最佳实践固化为可复用模板。

## 5.1 提示原语机制

- 每个 Prompt 由名称、描述和参数列表（`PromptArgument`）构成。
- `get` 返回一组消息（`PromptMessage`），包含角色与内容。

## 5.2 完整示例

```java
.prompts(new McpServerFeatures.SyncPromptSpecification(
        new Prompt("code-review", "代码审查提示",
                List.of(new PromptArgument("language", "编程语言", true))),
        (exchange, request) -> {
            String language = (String) request.arguments().get("language");
            return new GetPromptResult(
                    "代码审查",
                    List.of(new PromptMessage(Role.USER,
                            new TextContent("请审查下面的 " + language + " 代码："))));
        }))
```

## 5.3 边界与坑

- 参数声明为必填（`required=true`）时，Client 未传参应返回错误而非静默。
- 提示内容属于服务端资产，注意避免注入用户可控的危险文本。
- 提示模板变更要考虑向后兼容，老 Client 可能缓存了模板。

# 6. 网络传输（远程 Server）

结论先行：远程 Server 推荐使用 Streamable HTTP（可流式 HTTP）传输，它用单一端点同时支持普通请求与流式响应；SSE（Server-Sent Events，服务器发送事件）为历史方案。

## 6.1 传输方式对比

| 传输 | 端点 | 场景 | 备注 |
|------|------|------|------|
| stdio | 无网络 | 本地工具 | 简单、无鉴权 |
| SSE | 两段式 | 旧版远程 | 逐步被替代 |
| Streamable HTTP | 单端点 | 现代远程 | 推荐 |

## 6.2 Streamable HTTP 实现

### 6.2.1 Servlet 方式

```java
// Streamable HTTP
HttpServletStreamableServerTransportProvider transport =
        HttpServletStreamableServerTransportProvider.builder()
                .mcpEndpoint("/mcp")
                .build();

McpServer server = McpServer.sync(transport)
        .serverInfo("remote-server", "1.0.0")
        .capabilities(ServerCapabilities.builder().tools(true).build())
        .tools(...)
        .build();

// 注册到 Servlet 容器（如 Spring Boot）
```

### 6.2.2 WebFlux 方式

```java
// 伪代码：响应式传输提供者
WebFluxStreamableServerTransportProvider transport =
        WebFluxStreamableServerTransportProvider.builder()
                .mcpEndpoint("/mcp")
                .build();
```

## 6.3 边界与坑

- 远程部署必须加鉴权（Token/OAuth），stdio 的"进程隔离即安全"假设不再成立。
- 需要处理会话（session）标识，多实例部署时注意粘性会话或共享状态。
- 长连接要考虑网关超时与空闲回收。
- 开启 CORS 时仅允许可信来源。

# 7. 工具的安全实现

结论先行：工具是"把模型的话变成真实副作用"的入口，必须假设所有参数都不可信，做路径、资源、权限三重防护。

## 7.1 威胁模型

| 威胁 | 示例 | 后果 |
|------|------|------|
| 路径穿越 | `../../etc/passwd` | 读取越权文件 |
| 符号链接 | 链接指向外部 | 绕过根目录限制 |
| 资源耗尽 | 读取超大文件 | OOM |
| 越权调用 | 低权限用户调用管理工具 | 权限提升 |

## 7.2 完整安全代码

### 7.2.1 路径穿越防护

```java
Path root = Path.of("/data/workspace").toAbsolutePath();
Path target = root.resolve(path).normalize().toAbsolutePath();

if (!target.startsWith(root)) {
    throw new SecurityException("路径越界");
}
```

### 7.2.2 资源与类型限制

```java
@Tool(name = "read_file", description = "读取文件")
public String readFile(@ToolParam(description = "路径") String path) {
    Path root = Path.of("/data/workspace").toAbsolutePath();
    Path target = root.resolve(path).normalize().toAbsolutePath();

    // ① 路径穿越防护
    if (!target.startsWith(root)) {
        throw new SecurityException("路径越界");
    }
    // ② 符号链接防护
    if (Files.isSymbolicLink(target)) {
        throw new SecurityException("不允许访问符号链接");
    }
    // ③ 大小限制
    if (Files.size(target) > MAX_FILE_SIZE) {
        throw new SecurityException("文件过大");
    }
    // ④ 类型限制
    if (!isAllowedExtension(target)) {
        throw new SecurityException("不支持的文件类型");
    }
    return Files.readString(target);
}
```

### 7.2.3 权限校验

```java
// 伪代码：从调用上下文取用户身份
if (!permissionService.can(userId, "file:read", target)) {
    return new CallToolResult(
            List.of(new TextContent("无权限")), true);
}
```

## 7.3 边界与坑

- `normalize()` 只能消除 `.` 与 `..`，不能解析符号链接，需配合 `toRealPath()`。
- `startsWith` 对字符串路径成立不代表目录边界正确，比较时应基于规范化后的 `Path`。
- TOCTOU（Time-of-Check to Time-of-Use，检查与使用时间差）风险：校验与打开之间文件可能被替换。

# 8. 错误处理

结论先行：MCP 工具应**返回结构化错误（`isError=true`）而非抛出异常**，抛异常可能被当作协议层错误，甚至中断连接。

## 8.1 错误模型

| 层级 | 表现 | 处理 |
|------|------|------|
| 业务错误 | `isError=true` | 返回提示文本 |
| 协议错误 | JSON-RPC error | 由 SDK 处理 |
| 传输错误 | 连接中断 | 重连/降级 |

## 8.2 完整示例

```java
// MCP 工具返回 isError=true 而非抛异常
(exchange, args) -> {
    try {
        String result = doWork(args);
        return new CallToolResult(List.of(new TextContent(result)), false);
    } catch (Exception e) {
        return new CallToolResult(
                List.of(new TextContent("执行失败：" + e.getMessage())),
                true);              // isError = true
    }
}
```

> 与函数调用一致：**返回结构化错误，而非中断连接**。

## 8.3 边界与坑

- 错误信息不要泄露内部路径、堆栈等敏感信息。
- 区分"可重试"与"不可重试"错误，便于上层决策。
- 参数校验失败应尽早返回，避免执行副作用。

# 9. 测试

结论先行：测试应覆盖正常路径与安全边界，优先使用内存传输在进程内完成，无需真实网络。

## 9.1 测试策略

| 类型 | 目标 |
|------|------|
| 单元 | 工具函数本身 |
| 集成 | Server/Client 交互 |
| 安全 | 路径穿越、越权 |

## 9.2 完整测试代码

```java
@Test
void should_read_file_within_allowed_root() throws Exception {
    // 用内存传输测试
    McpServer server = buildServer();
    McpClient client = connectTestClient(server);

    CallToolResult result = client.callTool(
            new CallToolRequest("read_file", Map.of("path", "test.txt")));

    assertFalse(result.isError());
    assertEquals("内容", ((TextContent) result.content().get(0)).text());
}

@Test
void should_reject_path_traversal() {
    CallToolResult result = client.callTool(
            new CallToolRequest("read_file",
                    Map.of("path", "../../etc/passwd")));

    assertTrue(result.isError());
}
```

## 9.3 边界与坑

- 测试要显式覆盖 Windows 与 Linux 的路径差异。
- 断言内容类型而非仅断言非空。
- 安全测试应作为回归用例长期保留。

# 10. 生产注意事项

结论先行：上线前必须回答"谁可以调用、能调用什么、调用记录在哪、失败如何降级"四个问题。

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 路径穿越 | normalize + startsWith |
| 无资源限制 | 文件大小/数量限制 |
| 无超时 | 工具内设超时 |
| 异常中断 | 返回 isError |
| 无鉴权 | 校验调用方身份 |
| 无日志 | 记录调用 |
| 无测试 | 含安全测试 |

## 10.2 可观测性

- 记录工具名、参数摘要、耗时、结果状态，敏感字段脱敏。
- 对失败率、超时率设置告警。

## 10.3 部署建议

- stdio 场景与宿主进程绑定生命周期。
- HTTP 场景置于网关之后，统一鉴权与限流，灰度发布。

# 11. 小结

## 11.1 要点回顾

- Java SDK 提供同步/异步 Server API。
- 三大原语：Tools、Resources、Prompts。
- 工具实现必须做路径/资源/权限防护。
- 错误用 `isError=true` 返回，而非抛异常。
- 安全测试（路径穿越等）必须覆盖。

## 11.2 落地检查清单

- [ ] 版本锁定 BOM，能力声明与规格一致。
- [ ] stdio 日志走 stderr。
- [ ] 工具描述清晰、Schema 合法。
- [ ] 路径/资源/权限三重防护。
- [ ] 错误结构化返回。
- [ ] 安全回归测试覆盖。
