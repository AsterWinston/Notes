# 1. MCP 是什么

结论先行：**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 提出的开放协议，用于标准化"AI 应用如何接入外部能力"。它把工具、数据源与提示模板统一抽象为可运行时发现、可跨应用复用的服务。

## 1.1 一句话定义

MCP 是一套基于 JSON-RPC 2.0 的客户端/服务端协议：AI 应用扮演客户端，外部能力以服务端形式暴露，双方通过统一的请求/响应/通知语义通信。

## 1.2 从 N×M 到 N+M

```
传统方式：N 个 AI 应用 × M 个工具 = N×M 套适配
MCP 方式：工具方实现一次 MCP Server，所有 AI 应用都能接入 = N+M
```

### 1.2.1 适配数量对比

| 模式 | 适配数量 | 新增工具成本 | 新增应用成本 |
|------|----------|--------------|--------------|
| 传统点对点 | N×M | 每应用改一次 | 每工具接一次 |
| MCP | N+M | 一次 | 一次 |

### 1.2.2 成本模型

当 N=10、M=20 时，点对点为 200 套适配，MCP 仅 30 套。规模越大，收益越明显；反之若只有 1 个应用、1 个工具，MCP 反而引入协议开销。

## 1.3 类比与边界

### 1.3.1 USB 类比

> 类比：MCP 之于 AI 应用，如同 USB 之于外设。

工具方按标准接口实现，应用方按标准接口接入，双方无需点对点约定。

### 1.3.2 边界

边界：MCP 只标准化"接口与传输"，不负责工具的业务正确性、权限模型与数据治理，这些仍需使用方自行保障。

## 1.4 生产建议

- 先用 MCP 覆盖"多应用复用"的工具，再逐步迁移单应用工具。
- 对每个引入的 Server 做可用性与安全评估。

# 2. 解决的问题

结论先行：MCP 针对的是"适配重复、接口割裂、接入静态"三类痛点，用统一协议与运行时发现化解。

## 2.1 问题清单

| 问题 | 表现 |
|------|------|
| 每个应用重复适配工具 | 同一工具被多应用各写一遍 |
| 工具接口不统一 | 参数、返回、错误语义各不相同 |
| 生态割裂 | 工具无法组合与迁移 |
| 动态接入困难 | 新增能力需改代码、重新部署 |

## 2.2 方案映射

| 问题 | MCP 的方案 |
|------|------------|
| 每个应用重复适配工具 | 工具方实现一次，多方复用 |
| 工具接口不统一 | 统一协议（JSON-RPC 2.0） |
| 生态割裂 | 标准化生态，可组合 |
| 动态接入困难 | 运行时发现与调用 |

### 2.2.1 接口统一

工具以 `name` + `description` + `inputSchema`（JSON Schema）描述，模型可据此生成调用参数，消除各家私有格式。

### 2.2.2 运行时发现

客户端通过 `tools/list`、`resources/list`、`prompts/list` 在运行时获取能力，并通过 `list_changed` 通知感知变更，无需重启。

## 2.3 边界与反例

MCP 无法解决"工具本身设计得差"的问题。若工具粒度太粗、参数不可枚举，即使接入 MCP，模型也难以正确调用。因此工具设计（见第 4 节）比协议本身更关键。

# 3. 核心价值

结论先行：MCP 的价值按角色分为三层——工具方"一次实现、多方复用"，应用方"一次接入、获得生态"，用户方"自由组合、不被绑定"。

## 3.1 对工具提供方

```
对工具提供方：
  实现一次 MCP Server → 被所有 MCP Client 使用
```

## 3.2 对 AI 应用与用户

```
对 AI 应用：
  接入一次 MCP Client → 获得生态中所有工具

对用户：
  自由组合工具，不被单一厂商绑定
```

## 3.3 生产建议

- 工具方应把 MCP Server 当作"产品接口"维护，保证版本兼容与向后兼容。
- 应用方应建立 Server 白名单与健康检查，避免引入不可用能力。

# 4. 三大原语

结论先行：MCP 定义三大原语——Tools（工具，模型主动调用）、Resources（资源，应用读取数据）、Prompts（提示模板，预定义片段）。理解三者的触发方与用途差异，是正确建模的前提。

## 4.1 Tools

| 维度 | 说明 |
|------|------|
| 触发方 | 模型（通过工具调用） |
| 语义 | 执行动作、产生副作用 |
| 典型 | 读文件、发请求、写数据库 |
| 类比 | 函数调用 |

### 4.1.1 定义与触发

工具是唯一可由模型主动调用的原语，适合"查询后行动"的场景，如查天气、发邮件。

### 4.1.2 Java 示例

```java
// Spring AI 风格：方法即工具，由框架注册
@Component
public class FileTools {

    @Tool(description = "读取指定路径的文件内容")
    public String readFile(String path) throws IOException {
        return Files.readString(Path.of(path));
    }
}

// 注册为 MCP 工具回调
ToolCallbackProvider tools = MethodToolCallbackProvider.builder()
        .toolObjects(new FileTools())
        .build();

// 调用：模型产生工具调用后由框架分发
tools.call("readFile", Map.of("path", "/data/a.txt"));
```

## 4.2 Resources

| 维度 | 说明 |
|------|------|
| 触发方 | 应用（把数据注入上下文） |
| 语义 | 读取数据、无副作用 |
| 典型 | 文件、数据库行、接口返回 |
| 类比 | 文件/接口 |

### 4.2.1 定义与触发

资源以 URI 标识，应用读取后注入模型上下文。它不产生副作用，适合把"只读数据"喂给模型。

### 4.2.2 Java 示例

```java
// 读取资源（伪代码，API 以所用 SDK 为准）
ResourceContent content = mcpClient.readResource("file:///data/a.txt");
String text = content.text();

// 订阅资源变更
mcpClient.subscribe("file:///data/a.txt", uri -> {
    // 收到 notifications/resources/updated 后重新读取
    refreshContext(uri);
});
```

## 4.3 Prompts

| 维度 | 说明 |
|------|------|
| 触发方 | 用户/应用 |
| 语义 | 提供预定义提示模板 |
| 典型 | 代码审查模板、翻译模板 |
| 类比 | 提示片段 |

### 4.3.1 定义与触发

提示模板由用户或应用显式获取，可带参数渲染，适合把团队最佳实践固化成可复用片段。

### 4.3.2 Java 示例

```java
// 获取并渲染提示模板（伪代码）
PromptResult prompt = mcpClient.getPrompt(
        "code-review", Map.of("language", "java"));
String rendered = prompt.messages().get(0).content();
```

## 4.4 三者的选择准则

| 原语 | 说明 | 类比 |
|------|------|------|
| **Tools** | 可被模型调用的函数 | 函数调用 |
| **Resources** | 可读取的数据源 | 文件/接口 |
| **Prompts** | 预定义提示模板 | 提示片段 |

选择准则：需要模型"决定并执行"用 Tools；需要"读数据入上下文"用 Resources；需要"可复用的提示"用 Prompts。

## 4.5 边界与坑

- 把只读数据做成 Tools 会让模型误触发副作用，应优先 Resources。
- Resources 不宜过大，否则上下文膨胀，需分页或摘要。
- Prompts 不宜过度参数化，参数越多模型越易误用。

# 5. 与函数调用的关系

结论先行：函数调用是进程内的本地机制，MCP 是跨进程/跨应用的工具调用协议；二者不是替代关系，MCP 在传输层可复用函数调用能力。

## 5.1 位置与发现差异

```
函数调用：模型 → 框架 → 本地 Java 方法
MCP：    模型 → MCP Client → MCP Server（可能远程）→ 工具
```

## 5.2 对比表

| 维度 | 函数调用 | MCP |
|------|----------|-----|
| 位置 | 进程内 | 进程内/远程 |
| 发现 | 代码注册 | 运行时发现 |
| 复用 | 应用内 | 跨应用 |
| 协议 | 厂商私有 | 开放标准 |

> MCP 本质是**跨进程/跨应用的工具调用协议**。

## 5.3 边界与生产建议

- 进程内单应用工具用 `@Tool` 即可，引入 MCP 只增加开销。
- 需要多应用共享或远程能力时，才升级到 MCP。
- 混合使用时，可在同一应用内同时注册本地工具与 MCP 工具。

# 6. MCP 与相邻协议对比

结论先行：MCP 与 OpenAPI、gRPC 定位不同——前者面向"模型驱动的能力发现与调用"，后两者面向"服务间确定性调用"。

## 6.1 与 OpenAPI 对比

| 维度 | OpenAPI | MCP |
|------|---------|-----|
| 主要消费者 | 开发者/代码生成 | 模型/Agent |
| 发现方式 | 静态文档 | 运行时 list |
| 调用语义 | HTTP 端点 | 工具/资源/提示 |
| 状态 | 无状态为主 | 有状态会话 |

## 6.2 与 gRPC 对比

| 维度 | gRPC | MCP |
|------|------|-----|
| 传输 | HTTP/2 + Protobuf | JSON-RPC 2.0 |
| 契约 | .proto 强类型 | JSON Schema |
| 适用 | 高吞吐内部服务 | 工具接入 |
| 动态性 | 需重新生成 | 运行时发现 |

## 6.3 选型建议

- 面向模型的工具接入选 MCP。
- 面向服务的确定性调用选 gRPC/OpenAPI。
- 二者可并存：MCP Server 内部调用 gRPC 服务。

# 7. 生态现状

结论先行：MCP 生态已形成"官方 Server + 社区 Server + 企业自建 + 多客户端"的格局。

## 7.1 Server 生态

| 类型 | 例子 |
|------|------|
| 官方 Server | 文件系统、Git、Fetch、Memory |
| 社区 Server | 数据库、云服务、SaaS 集成 |
| 企业自建 | 内部系统适配 |

## 7.2 Client 生态

| 客户端 | 说明 |
|--------|------|
| Claude Desktop | 桌面 AI 应用 |
| IDE | 编辑器插件 |
| Spring AI | Java 框架集成 |
| LangChain4j | Java 框架集成 |

## 7.3 生产建议

- 优先选用维护活跃、有版本发布的 Server。
- 企业场景建议自建私有 Server，避免把内部数据交给第三方。

# 8. 典型架构

结论先行：MCP 采用 Host / Client / Server 三层结构，Client 与 Server 之间是一对一连接。

## 8.1 分层图

```
┌──────────────────────────────────────────────┐
│  Host（AI 应用 / IDE / 桌面客户端）            │
│  ┌────────────────┐  ┌────────────────┐      │
│  │ MCP Client A   │  │ MCP Client B   │      │
│  └───────┬────────┘  └───────┬────────┘      │
└──────────┼───────────────────┼───────────────┘
           │ MCP (JSON-RPC)    │
           ▼                   ▼
    ┌────────────┐      ┌────────────┐
    │ MCP Server │      │ MCP Server │
    │ 文件系统    │      │ 数据库      │
    └────────────┘      └────────────┘
```

## 8.2 角色职责

| 角色 | 说明 |
|------|------|
| Host | 承载 AI 的应用 |
| Client | Host 内连接 Server 的组件 |
| Server | 提供工具/资源的服务 |

## 8.3 边界与生产建议

- 一个 Server 对应一个 Client 连接，隔离性好但连接数会随 Server 增长。
- Host 需统一管理连接生命周期与鉴权，避免泄漏。

# 9. 传输方式

结论先行：MCP 支持四种传输——stdio（本地进程）、SSE（远程长连接）、Streamable HTTP（现代双向 HTTP）、Stateless HTTP（无状态）。

## 9.1 四种传输

| 传输 | 说明 | 适用 |
|------|------|------|
| stdio | 标准输入输出 | 本地进程 |
| SSE | Server-Sent Events | 远程长连接 |
| Streamable HTTP | 现代双向 HTTP | 生产远程（推荐） |
| Stateless HTTP | 无状态 | Serverless |

## 9.2 选型建议

生产远程优先 Streamable HTTP；Serverless 场景用 Stateless；本地工具用 stdio；SSE 属过渡方案，新项目不建议采用。

### 9.2.1 stdio 配置示例

```java
// 以本地子进程方式连接文件系统 Server（伪代码）
McpClient client = McpClient.builder()
        .transport(StdioTransport.builder()
                .command("npx")
                .args(List.of("-y", "@modelcontextprotocol/server-filesystem", "/data"))
                .build())
        .build();
```

### 9.2.2 Streamable HTTP 配置示例

```java
// 以远程 HTTP 方式连接（伪代码）
McpClient client = McpClient.builder()
        .transport(StreamableHttpTransport.builder()
                .baseUrl("https://mcp.example.com/mcp")
                .build())
        .build();
```

## 9.3 边界与坑

- stdio 无网络隔离，需限制子进程权限。
- 长连接需处理断线重连与心跳。

# 10. 适用场景

结论先行：MCP 适合"工具生态复用、内部系统适配、多应用共享、动态扩展"四类场景。

## 10.1 场景清单

| 场景 | 说明 |
|------|------|
| 工具生态复用 | 接入现成 MCP Server |
| 内部系统适配 | 自建 MCP Server 暴露内部能力 |
| 多应用共享工具 | 一次实现，多应用使用 |
| 动态扩展能力 | 运行时接入新工具 |

## 10.2 生产建议

- 内部系统适配时，Server 应做最小权限暴露，只开放必要能力。

# 11. 不适用场景

结论先行：进程内单应用工具、极低延迟要求、无复用需求时，不应引入 MCP。

## 11.1 反例清单

| 场景 | 更合适方案 |
|------|------------|
| 单一应用内部工具 | 直接用 `@Tool` |
| 极低延迟要求 | 进程内函数调用 |
| 不需要复用 | 无需引入协议开销 |

## 11.2 边界说明

MCP 的协议与序列化开销在微秒到毫秒级，对高频、低延迟路径是显著成本，应谨慎评估。

# 12. 安全与权限概览

结论先行：MCP 把外部能力引入 AI 应用，安全边界必须显式设计，至少覆盖认证、授权、最小权限与审计。

## 12.1 威胁模型

| 威胁 | 说明 |
|------|------|
| 未授权调用 | Server 端点被匿名访问 |
| 越权工具 | 工具暴露过大权限 |
| 提示注入 | 资源内容携带恶意指令 |
| 数据外泄 | 内部数据流入第三方 Server |

## 12.2 防护措施

| 措施 | 说明 |
|------|------|
| 端点鉴权 | Token/OAuth2 校验调用方 |
| 最小权限 | 只暴露必要工具与数据 |
| 内容校验 | 资源内容与参数做校验 |
| 审计日志 | 记录调用方与调用链 |

### 12.2.1 认证与授权

Server 端在接入层校验调用方身份，再按调用方授予不同工具集。

```java
// Spring AI / Spring Security 风格（伪代码）
@Bean
public SecurityFilterChain mcpSecurity(HttpSecurity http) throws Exception {
    return http
            .securityMatcher("/mcp/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
            .build();
}
```

### 12.2.2 最小权限与审计

- 工具按"读/写"分级，默认只开放读工具。
- 每次调用记录：调用方、工具名、参数摘要、结果状态、耗时。
- 对高风险工具（写文件、执行命令）增加二次确认。

# 13. 本模块结构

结论先行：本模块从协议架构、Java 服务端/客户端开发、Spring AI 集成、安全与实战依次展开。

## 13.1 文件导航

| 文件 | 主题 |
|------|------|
| `2_mcp_architecture.md` | 协议架构与消息 |
| `3_java_sdk_server.md` | Java 开发 Server |
| `4_java_sdk_client.md` | Java 客户端接入 |
| `5_spring_ai_mcp.md` | Spring AI 集成 |
| `6_security.md` | 安全与权限隔离 |
| `7_examples.md` | 实战示例 |

## 13.2 阅读顺序

建议按文件编号顺序阅读，先建立协议认知，再进入编码实践。

# 14. 小结

- MCP 是标准化 AI 应用接入外部能力的开放协议。
- 解决 N×M 适配问题，实现 N+M 复用。
- 三大原语：Tools、Resources、Prompts。
- 本质是跨进程/跨应用的工具调用协议。
- 适合工具复用与生态接入，进程内工具无需 MCP。
