# 1. MCP 是什么

**结论先行**：MCP（Model Context Protocol，模型上下文协议）是 Anthropic 提出的开放协议，用于标准化"AI 应用如何接入外部能力"，把过去"每个应用为每个工具写一套适配"的 N×M 问题，降维成 N+M。

## 1.1 定位与价值

传统集成方式中，若 AI 应用有 N 个、外部工具有 M 个，适配成本是 N×M；引入统一协议后，工具方只需实现一次 MCP Server，AI 应用只需实现一次 MCP Client，成本降为 N+M。

```
传统方式：每个 AI 应用为每个工具写一套适配
MCP 方式：工具方提供 MCP Server，AI 应用作为 MCP Client 统一接入
```

价值可以归纳为四点：

| 维度 | 传统集成 | MCP 集成 |
|------|----------|----------|
| 适配成本 | N×M，重复造轮子 | N+M，协议复用 |
| 工具复用 | 绑定具体框架 | 跨语言、跨框架 |
| 权限治理 | 散落在各应用 | 协议层统一建模 |
| 生态 | 各自为政 | 可插拔的 Server 市场 |

## 1.2 协议栈与通信模型

MCP 基于 JSON-RPC 2.0（一种轻量远程调用协议），底层传输可以是 stdio（标准输入输出）、SSE（Server-Sent Events，服务端推送）或 Streamable HTTP。

```
┌─────────────┐        MCP        ┌──────────────┐
│  AI 应用     │ ◀───────────────▶ │  MCP Server  │
│ (MCP Client)│   JSON-RPC 2.0    │  (工具/资源)  │
└─────────────┘                   └──────────────┘
                                        │
                              ┌─────────┼─────────┐
                              ▼         ▼         ▼
                           文件系统   数据库     Git/API
```

一次完整交互大致经历：初始化握手 → 能力协商 → 请求/响应 → 通知。Server 在握手时声明自己支持哪些能力（capabilities），Client 据此决定可用范围。

```
Client                          Server
  │  initialize (protocolVersion) │
  │──────────────────────────────▶│
  │  initialize result            │
  │◀──────────────────────────────│
  │  initialized (notification)   │
  │──────────────────────────────▶│
  │  tools/list                   │
  │──────────────────────────────▶│
  │  tools/list result            │
  │◀──────────────────────────────│
  │  tools/call                   │
  │──────────────────────────────▶│
  │  tools/call result            │
  │◀──────────────────────────────│
```

## 1.3 与函数调用、插件的关系

三者常被混淆，边界如下：

| 概念 | 层次 | 解决的问题 | 是否开放 |
|------|------|------------|----------|
| 函数调用 | 模型能力 | 模型如何表达"我要调工具" | 各厂商自有 |
| 插件 | 厂商生态 | 在某平台上扩展能力 | 平台私有 |
| MCP | 工具供给协议 | 工具如何被标准化接入 | 开放协议 |

MCP 与函数调用是互补关系：MCP 负责"把工具接进来并治理"，函数调用负责"让模型决定调哪个"。

> 协议细节见 `7_ToolsAndProtocols/MCP`。本文件聚焦 Spring AI 的集成方式。

# 2. MCP 的核心原语

**结论先行**：MCP 用三个原语覆盖"模型能做什么、能读什么、按什么模板说"三类需求，分别是 Tools、Resources、Prompts。

## 2.1 Tools（工具）

Tools 是可被模型主动调用的函数，具有输入 Schema（参数结构）和输出。它是最常被使用的原语，等价于函数调用中的 function。

## 2.2 Resources（资源）

Resources 是可被读取的数据源，类似"只读的文件句柄"。它通常由应用侧或用户显式选择，而非模型自动决定。

## 2.3 Prompts（提示）

Prompts 是服务端预定义的提示模板，客户端可参数化调用，用于沉淀领域最佳实践。

## 2.4 原语对比

| 原语 | 说明 | 类比 | 触发方 |
|------|------|------|--------|
| Tools | 可被模型调用的函数 | 函数调用 | 模型 |
| Resources | 可读取的数据源 | 文件/接口 | 应用/用户 |
| Prompts | 预定义提示模板 | 提示片段 | 应用/用户 |

# 3. 两种角色

**结论先行**：Spring AI 同时提供 Server 与 Client 两套 starter，一个应用可以只做其中一种，也可以同时扮演两种角色。

## 3.1 MCP Server

MCP Server 对外暴露工具、资源与提示。Spring AI 对应模块为 `spring-ai-starter-mcp-server`（含 webmvc / webflux 变体）。

## 3.2 MCP Client

MCP Client 消费一个或多个 MCP Server 的能力，把远端工具纳入本地的工具列表。对应模块为 `spring-ai-starter-mcp-client`。

| 角色 | 说明 | Spring AI 模块 |
|------|------|----------------|
| MCP Server | 对外暴露工具/资源 | `spring-ai-starter-mcp-server` |
| MCP Client | 消费 MCP Server 的能力 | `spring-ai-starter-mcp-client` |

## 3.3 角色组合模式

常见的三种拓扑：

```
1) 单 Client 单 Server    2) 单 Client 多 Server    3) Client 兼 Server
   ┌────┐   ┌────┐          ┌────┐  ┌──┐ ┌──┐        ┌────┐  ┌────┐
   │App │──▶│Srv │          │App │─▶│S1│ │S2│        │App │◀▶│下游│
   └────┘   └────┘          └────┘  └──┘ └──┘        └─┬──┘  └────┘
                                                        │
                                                      ┌─▼──┐
                                                      │上游│
                                                      └────┘
```

模式 3（网关式）常用于企业内部统一能力出口：对上游暴露统一工具集，对下游聚合多个业务 Server。

# 4. 开发 MCP Server

**结论先行**：Server 侧的核心是"用 `@Tool` 注解声明能力，用 `ToolCallbackProvider` 注册，再通过配置选择传输协议"。

## 4.1 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

> 也可选 `mcp-server-webflux`（响应式）或 stdio 传输。

## 4.2 暴露工具

用 `@Tool` 标注方法即成为一个可被调用的工具；`@ToolParam` 描述参数，帮助模型正确填参。

```java
@Service
public class FileTools {

    private static final Path ALLOWED_ROOT = Path.of("D:/workspace").toAbsolutePath();

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

    @Tool(name = "list_files", description = "列出目录下的文件名")
    public List<String> listFiles(
            @ToolParam(description = "目录绝对路径") String dir) throws IOException {
        Path d = Path.of(dir).normalize();
        if (!d.startsWith(ALLOWED_ROOT)) {
            throw new SecurityException("路径越界");
        }
        try (Stream<Path> s = Files.list(d)) {
            return s.map(Path::getFileName).map(Path::toString).toList();
        }
    }
}
```

### 4.2.1 返回值的结构化

工具返回值会被序列化后回传给模型。返回对象或 `Map` 比返回裸字符串更利于模型理解：

```java
public record FileInfo(String name, long size, String modifiedAt) {}

@Tool(name = "stat_file", description = "获取文件元信息")
public FileInfo stat(@ToolParam(description = "文件路径") String path) throws IOException {
    Path p = Path.of(path).normalize();
    if (!p.startsWith(ALLOWED_ROOT)) throw new SecurityException("路径越界");
    BasicFileAttributes a = Files.readAttributes(p, BasicFileAttributes.class);
    return new FileInfo(p.getFileName().toString(), a.size(),
            a.lastModifiedTime().toString());
}
```

## 4.3 注册工具

仅标注 `@Tool` 还不够，必须通过 `ToolCallbackProvider` 把工具对象注册进容器。

```java
@Configuration
public class McpServerConfig {

    @Bean
    public ToolCallbackProvider fileToolProvider(FileTools fileTools) {
        return MethodToolCallbackProvider.builder()
                .toolObjects(fileTools)
                .build();
    }
}
```

### 4.3.1 多工具对象聚合

```java
@Bean
public ToolCallbackProvider allTools(FileTools files, DbTools db, GitTools git) {
    return MethodToolCallbackProvider.builder()
            .toolObjects(files, db, git)
            .build();
}
```

## 4.4 暴露资源与提示

除工具外，Server 还可暴露资源与提示模板。

```java
@Component
public class DocResources {

    @McpResource(uri = "file:///docs/{name}",
            name = "doc",
            description = "读取知识库文档")
    public String readDoc(@McpResourceParam String name) throws IOException {
        return Files.readString(Path.of("D:/docs", name + ".md"));
    }
}
```

> 资源与提示的注解命名在不同 Spring AI 版本中略有差异，落地前请对照当前版本 API。

## 4.5 配置

```yaml
spring:
  ai:
    mcp:
      server:
        name: file-mcp-server
        version: 1.0.0
        protocol: STREAMABLE       # 或 STATELESS / STDIO
```

### 4.5.1 关键配置项

| 配置项 | 作用 | 常用取值 |
|--------|------|----------|
| `name` | Server 标识 | 业务名 |
| `version` | 版本号 | 语义化版本 |
| `protocol` | 传输协议 | STREAMABLE / STATELESS / STDIO |
| `type` | 同步/异步 | SYNC / ASYNC |

## 4.6 本地自测

Server 开发完成后，应先用官方 Inspector 或简单 Client 验证工具可见与可调用，再接入真实应用：

```bash
# 以 stdio 方式启动并检查工具列表
npx @modelcontextprotocol/inspector java -jar target/mcp-server.jar
```

自测清单：

- 工具是否全部出现在 `tools/list` 结果中；
- 参数 Schema 是否与预期一致；
- 异常路径（越界、不存在）是否返回可读错误；
- 大数据量返回值是否被截断。

# 5. 接入 MCP Client

**结论先行**：Client 侧只需声明"连接哪些 Server"，开启 `toolcallback.enabled` 后，远端工具会自动进入模型的工具列表，业务代码几乎零改动。

## 5.1 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
```

## 5.2 配置 Server 连接

```yaml
spring:
  ai:
    mcp:
      client:
        enabled: true
        name: my-ai-app
        version: 1.0.0
        type: SYNC                  # 或 ASYNC
        toolcallback:
          enabled: true             # 自动把 MCP 工具注册为 ToolCallback
        stdio:
          connections:
            filesystem:
              command: npx
              args:
                - "-y"
                - "@modelcontextprotocol/server-filesystem"
                - "D:/workspace"
        sse:
          connections:
            remote-tools:
              url: http://localhost:8080
```

### 5.2.1 连接参数说明

| 字段 | 含义 | 备注 |
|------|------|------|
| `stdio.command` | 启动本地 Server 的命令 | 进程随应用生命周期管理 |
| `stdio.args` | 命令参数 | 注意路径分隔符与空格 |
| `sse.url` | 远端 Server 地址 | 需保证网络可达 |
| `type` | 调用模式 | SYNC 简单，ASYNC 高吞吐 |

## 5.3 自动注册工具

配置 `toolcallback.enabled=true` 后，MCP 工具会自动出现在模型可调用工具列表中：

```java
String answer = chatClient.prompt()
        .user("列出 D:/workspace 下的所有文件")
        .call()
        .content();
```

## 5.4 手动使用 MCP 工具

也可绕过模型，手动按名称调用：

```java
@Service
public class McpToolService {

    private final ToolCallbackProvider toolCallbackProvider;

    public McpToolService(ToolCallbackProvider provider) {
        this.toolCallbackProvider = provider;
    }

    public String callTool(String name, String args) {
        return Arrays.stream(toolCallbackProvider.getToolCallbacks())
                .filter(cb -> cb.getToolDefinition().name().equals(name))
                .findFirst()
                .map(cb -> cb.call(args))
                .orElseThrow();
    }
}
```

### 5.4.1 工具名冲突

多个 Server 可能暴露同名工具。接入时应对工具名做命名空间前缀（如 `filesystem__read_file`），否则会出现覆盖或歧义。此逻辑通常在自定义 `ToolCallbackProvider` 中统一处理。

## 5.5 动态增删连接

生产环境中，Server 的可用性会变化。应支持运行时感知连接状态，并在 Server 不可用时从工具列表中剔除其工具，避免模型调用到"死工具"。

# 6. 传输方式对比

**结论先行**：本地进程优先 stdio，生产远程优先 Streamable HTTP；SSE 属于过渡方案，Stateless 适合 Serverless。

## 6.1 各传输详解

| 传输 | 说明 | 适用 |
|------|------|------|
| stdio | 通过标准输入输出通信 | 本地进程、CLI 工具 |
| SSE | Server-Sent Events | 远程、长连接 |
| Streamable HTTP | 更现代的双向 HTTP | 生产远程（推荐） |
| Stateless | 无状态 HTTP | 无状态部署、Serverless |

## 6.2 选型建议

```
本地同机工具        → stdio（零网络开销、进程隔离）
远程长连接服务      → Streamable HTTP（推荐）
已有 SSE 存量系统   → SSE（逐步迁移）
无状态/Serverless  → Stateless
```

### 6.2.1 边界与坑

- stdio 的 Server 进程随 Client 启停，若 Server 崩溃不会自动重启，需外部守护。
- SSE 在部分网关下会被缓冲，导致事件延迟，需关闭代理缓冲。
- Streamable HTTP 对会话状态有要求，多实例部署要保证粘性会话或外部会话存储。

# 7. MCP 安全与权限隔离（重点）

**结论先行**：MCP 把外部能力接进系统，也引入了新的攻击面；安全必须由服务端强制，而不是寄希望于"模型不会调用"。

MCP 把外部能力接进系统，也引入了新的攻击面：

| 风险 | 说明 | 防御 |
|------|------|------|
| 工具越权 | 模型调用了不该调的 MCP 工具 | 工具白名单、按用户授权 |
| 路径穿越 | `read_file("../../etc/passwd")` | 路径归一化 + 根目录限制 |
| 供应链风险 | 引入不可信的第三方 MCP Server | 审核来源、锁定版本、沙箱运行 |
| 数据外泄 | 工具把敏感数据发往外部 | 出网管控、数据脱敏 |
| 提示注入 | 工具返回内容含恶意指令 | 工具结果视为不可信数据 |

## 7.1 工具级权限隔离

不同角色只挂载其被授权的工具集：

```java
// 不同角色挂载不同工具集
Set<String> allowed = permissionService.allowedTools(currentUser());

List<ToolCallback> callbacks = Arrays.stream(allTools.getToolCallbacks())
        .filter(cb -> allowed.contains(cb.getToolDefinition().name()))
        .toList();

chatClient.prompt()
        .user(q)
        .toolCallbacks(callbacks)     // 只暴露有权限的工具
        .call()
        .content();
```

> **不要依赖"模型不会调用"来保证安全**，必须在服务端做工具级授权。

### 7.1.1 授权模型设计

建议采用"主体-工具-动作"三元组授权：

| 主体 | 工具 | 动作 | 说明 |
|------|------|------|------|
| 普通用户 | `read_file` | allow | 只读 |
| 普通用户 | `delete_file` | deny | 禁止删除 |
| 管理员 | `delete_file` | allow | 需二次确认 |

### 7.1.2 参数级校验

即使工具被授权，参数仍可能越界，需在工具内部再校验一次：

```java
private void assertWithin(Path p) {
    if (!p.toAbsolutePath().normalize().startsWith(ALLOWED_ROOT)) {
        throw new SecurityException("路径越界");
    }
}
```

## 7.2 会话级隔离

多用户共用一个 MCP Client 时，必须保证 A 用户的操作不会命中 B 用户的上下文/凭证。做法：

- 每会话独立 MCP Client 或独立凭证；
- 工具执行时校验"当前主体"；
- 审计每次工具调用。

### 7.2.1 反例：共享可变状态

```java
// 反例：工具持有全局可变上下文，多用户并发时串号
@Service
public class BadTool {
    private String currentUser;                 // 危险：实例级可变状态

    public void setUser(String u) { this.currentUser = u; }
}
```

正确做法是使用 `ThreadLocal`、请求作用域 Bean，或把用户身份作为参数显式传入。

> 详见 `7_ToolsAndProtocols/MCP/6_security.md` 与 `15_Security/9_mcp_permission_isolation.md`。

## 7.3 输入校验与路径限制

外部输入一律视为不可信：校验类型、长度、范围、字符集；路径类参数必须归一化并限制根目录；命令类参数禁止拼接 shell。

## 7.4 审计与可观测

每次工具调用记录：主体、工具名、参数摘要、结果状态、耗时、TraceId。审计日志与业务日志分离存储，保留期满足合规要求。

# 8. 生产注意事项

**结论先行**：生产化的核心是"稳定性、性能、兼容性、可治理"四条线。

## 8.1 稳定性

| 风险 | 建议 |
|------|------|
| Server 不可用 | 超时、重试、降级、熔断 |
| 工具调用慢 | 异步化、并行调用、超时限制 |
| 无审计 | 记录工具名、参数、主体、结果 |

## 8.2 性能

- 对可并行且无依赖的工具调用，启用并行以减少总延迟。
- 为工具设置独立超时，避免单个慢工具拖垮整条链路。
- 对高频只读工具增加缓存。

## 8.3 版本与兼容

| 风险 | 建议 |
|------|------|
| 版本不兼容 | 锁定 MCP SDK 版本，回归测试 |
| 工具过多 | 按场景筛选后再挂载 |

### 8.3.1 灰度与回滚

Server 升级采用灰度：先切部分流量，观察工具错误率与延迟，再全量；保留快速回滚能力。

## 8.4 治理清单

上线前逐项确认：

| 检查项 | 要求 | 状态 |
|--------|------|------|
| 工具授权 | 按主体白名单挂载 | 必选 |
| 路径限制 | 归一化 + 根目录约束 | 必选 |
| 超时设置 | 每个工具独立超时 | 必选 |
| 审计日志 | 记录主体/工具/参数摘要 | 必选 |
| 版本锁定 | SDK 与 Server 版本固定 | 建议 |
| 出网管控 | 限制 Server 可访问的目标 | 建议 |
| 降级策略 | Server 不可用时剔除工具 | 建议 |

# 9. 小结

- MCP 让"接入外部能力"标准化，Spring AI 提供 Server/Client 全套 starter。
- Server 侧用 `@Tool` + `ToolCallbackProvider` 暴露能力。
- Client 侧配置连接即可把 MCP 工具纳入模型工具列表。
- 传输方式按场景选：本地 stdio、远程 Streamable HTTP、Serverless Stateless。
- **安全是 MCP 的重中之重**：工具级授权、路径限制、来源审核、调用审计缺一不可。
