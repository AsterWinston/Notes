# 1. Spring AI 的 MCP 支持

结论先行：Spring AI 把 MCP 封装成 starter（起步依赖），Server 与 Client 都能"引依赖 + 写配置"完成接入，底层仍是官方 Java SDK，因此协议行为与 `3_java_sdk_server.md`、`4_java_sdk_client.md` 一致。

## 1.1 模块矩阵

| 模块 | 作用 |
|------|------|
| `spring-ai-starter-mcp-server-webmvc` | MCP Server（WebMVC） |
| `spring-ai-starter-mcp-server-webflux` | MCP Server（WebFlux） |
| `spring-ai-starter-mcp-client` | MCP Client |

> 详细配置见 `9_Frameworks/SpringAI/13_mcp_integration.md`，本文补充协议与实战视角。

## 1.2 与原生 SDK 对比

| 维度 | 原生 Java SDK | Spring AI Starter |
|------|---------------|-------------------|
| 依赖装配 | 手工构建 Server/Client | 自动配置 |
| 工具注册 | 构建器/注解 | `@Tool` + `ToolCallbackProvider` |
| 传输配置 | 代码 | `application.yml` |
| 与模型集成 | 手工桥接 | 自动进入工具列表 |

## 1.3 选型建议

- 纯 MCP 服务、无 Spring 环境：用原生 SDK。
- 已是 Spring Boot 应用、需要与 ChatClient 协作：用 Starter。
- 需要精细控制协议行为：Starter 打底，必要时注入自定义 `McpTransport`。

## 1.4 整体架构

```
用户 -> ChatClient -> 模型
                 |
                 | 工具调用
                 v
        ToolCallback(桥接) -> MCP Client -> MCP Server -> 真实资源
                 ^
                 |
        ToolCallbackProvider(本地@Tool 与远端工具聚合)
```

- 本地工具与 MCP 工具最终都以 `ToolCallback` 形式进入模型工具列表。
- Server 端能力经 Client 发现后，由 Starter 自动装配为 Bean。

# 2. 开发 MCP Server

结论先行：Server 端只需"声明工具、注册提供者、配置传输"三步，Spring AI 会自动完成协议层封装。

## 2.1 依赖与配置

### 2.1.1 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-server-webmvc</artifactId>
</dependency>
```

### 2.1.2 配置项

```yaml
spring:
  ai:
    mcp:
      server:
        name: knowledge-server
        version: 1.0.0
        protocol: STREAMABLE        # STREAMABLE / STATELESS / STDIO
        instructions: "企业知识库服务"
```

| 配置 | 含义 |
|------|------|
| `name` / `version` | 服务标识，握手时返回 |
| `protocol` | 传输模式 |
| `instructions` | 给 Client/模型的说明 |

## 2.2 暴露工具

### 2.2.1 @Tool 注解

```java
@Service
public class KnowledgeTools {

    @Tool(name = "search_knowledge", description = "搜索企业知识库")
    public List<KnowledgeItem> search(
            @ToolParam(description = "搜索关键词") String query) {
        return knowledgeService.search(query);
    }
}
```

### 2.2.2 注册 ToolCallbackProvider

```java
@Configuration
public class McpServerConfig {

    @Bean
    public ToolCallbackProvider knowledgeToolProvider(KnowledgeTools tools) {
        return MethodToolCallbackProvider.builder()
                .toolObjects(tools)
                .build();
    }
}
```

## 2.3 传输模式

```
STREAMABLE  单端点，支持流式，推荐远程
STATELESS   无会话状态，易水平扩展
STDIO       本地子进程，无网络
```

## 2.4 边界与坑

- `protocol` 选择要与部署方式匹配，stdio 模式无法直接监听端口。
- `@Tool` 的名称要与描述一起打磨，这是模型选型的依据。
- 返回复杂对象时注意序列化，必要时用 `ToolResponse` 包装。
- 多 Bean 提供工具时，注意重复注册同一工具名。

## 2.5 完整 Server 示例

```java
@SpringBootApplication
public class McpServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }
}

@Service
class DocumentTools {

    private final DocumentService service;

    DocumentTools(DocumentService service) {
        this.service = service;
    }

    @Tool(name = "list_documents", description = "列出知识库文档")
    public List<String> listDocuments() {
        return service.listNames();
    }

    @Tool(name = "get_document", description = "按 ID 获取文档正文")
    public String getDocument(
            @ToolParam(description = "文档 ID") String id) {
        return service.content(id);
    }
}

@Configuration
class McpServerConfig {

    @Bean
    ToolCallbackProvider documentTools(DocumentTools tools) {
        return MethodToolCallbackProvider.builder()
                .toolObjects(tools)
                .build();
    }
}
```

## 2.6 生产建议

- 工具实现只做编排，业务逻辑下沉到 Service，便于测试与复用。
- 为每个工具补充参数校验，非法输入直接拒绝。
- 通过配置切换协议，禁止在代码里写死传输方式。

# 3. 接入 MCP Client

结论先行：Client 只需在配置里列出要连接的 Server，开启 `toolcallback.enabled` 后，远端工具会自动并入模型工具列表。

## 3.1 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-mcp-client</artifactId>
</dependency>
```

## 3.2 配置多个 Server

### 3.2.1 stdio 与 SSE

```yaml
spring:
  ai:
    mcp:
      client:
        enabled: true
        name: my-ai-app
        version: 1.0.0
        type: SYNC
        toolcallback:
          enabled: true             # 自动注册为 ToolCallback
        stdio:
          connections:
            filesystem:
              command: npx
              args: ["-y", "@modelcontextprotocol/server-filesystem", "D:/workspace"]
        sse:
          connections:
            remote-tools:
              url: http://localhost:8080
```

### 3.2.2 Streamable HTTP

```yaml
        streamable-http:
          connections:
            modern-tools:
              url: http://localhost:9090
              endpoint: /mcp
```

## 3.3 使用

配置 `toolcallback.enabled=true` 后，MCP 工具自动进入模型工具列表：

```java
String answer = chatClient.prompt()
        .user("列出 D:/workspace 下的所有文件")
        .call()
        .content();
```

## 3.4 边界与坑

- 启动时若某个 Server 不可用，可能拖慢应用启动，需评估失败策略。
- `type: SYNC` 与 `ASYNC` 影响线程模型，响应式应用优先 `ASYNC`。
- 自动注册是"全量暴露"，生产必须叠加权限裁剪（见第 4 节）。

## 3.5 完整配置示例

```yaml
spring:
  ai:
    mcp:
      client:
        enabled: true
        name: my-ai-app
        version: 1.0.0
        type: SYNC
        request-timeout: 30s
        toolcallback:
          enabled: false           # 生产改为手动裁剪
        stdio:
          connections:
            filesystem:
              command: npx
              args: ["-y", "@modelcontextprotocol/server-filesystem", "D:/workspace"]
        streamable-http:
          connections:
            knowledge:
              url: https://mcp.example.com
              endpoint: /mcp
```

## 3.6 生产建议

- 关闭自动 `toolcallback`，改为按用户权限手动注入。
- 为每个连接单独设置超时，避免一个慢 Server 拖垮全局。
- 对远程连接强制 TLS 与鉴权头。

# 4. 手动获取与裁剪工具

结论先行：自动注册便于开发，但生产环境必须按用户权限裁剪工具，**不能依赖"模型不会调用"来保证安全**。

## 4.1 手动获取工具

```java
@Service
@RequiredArgsConstructor
public class McpToolService {

    private final ToolCallbackProvider mcpToolProvider;

    public List<ToolCallback> allMcpTools() {
        return Arrays.asList(mcpToolProvider.getToolCallbacks());
    }

    public ToolCallback findTool(String name) {
        return Arrays.stream(mcpToolProvider.getToolCallbacks())
                .filter(cb -> cb.getToolDefinition().name().equals(name))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("工具不存在: " + name));
    }
}
```

## 4.2 按权限裁剪

### 4.2.1 裁剪实现

```java
@Service
@RequiredArgsConstructor
public class McpToolAuthorization {

    private final ToolCallbackProvider mcpToolProvider;
    private final PermissionService permissionService;

    public List<ToolCallback> toolsFor(UserContext user) {
        Set<String> allowed = permissionService.allowedTools(user);

        return Arrays.stream(mcpToolProvider.getToolCallbacks())
                .filter(cb -> allowed.contains(cb.getToolDefinition().name()))
                .toList();
    }
}
```

### 4.2.2 接入对话

```java
// 使用时按用户裁剪
List<ToolCallback> tools = authorization.toolsFor(user);

String answer = chatClient.prompt()
        .user(question)
        .toolCallbacks(tools)          // 只暴露有权限的
        .call()
        .content();
```

## 4.3 边界与坑

- 裁剪只解决"能看见"，服务端仍要二次鉴权，防止越权调用。
- 权限变更后要刷新工具列表，避免会话内继续持有旧权限。
- 工具名冲突时裁剪逻辑要按 Server 维度区分。

## 4.4 生产建议

- 裁剪策略集中在权限服务，避免散落各处。
- 对每次对话记录实际暴露的工具集合，便于审计。
- 权限服务不可用时默认拒绝（fail-closed，失败即关闭）。

# 5. Server 端资源与提示

结论先行：Spring AI 同样支持通过 Bean 注册 Resources 与 Prompts，适合把配置、模板等资产暴露给 Client。

## 5.1 资源注册

```java
@Configuration
public class McpResourcesConfig {

    @Bean
    public List<McpServerFeatures.SyncResourceSpecification> resources() {
        return List.of(
                new McpServerFeatures.SyncResourceSpecification(
                        new Resource("config://app", "app-config",
                                "应用配置", "application/json", null),
                        (exchange, req) -> new ReadResourceResult(
                                List.of(new TextResourceContents(
                                        req.uri(), "application/json", loadConfig())))));
    }
```

## 5.2 提示注册

```java
    @Bean
    public List<McpServerFeatures.SyncPromptSpecification> prompts() {
        return List.of(
                new McpServerFeatures.SyncPromptSpecification(
                        new Prompt("review", "代码审查",
                                List.of(new PromptArgument("code", "代码", true))),
                        (exchange, req) -> new GetPromptResult("代码审查",
                                List.of(new PromptMessage(Role.USER,
                                        new TextContent("审查：" + req.arguments().get("code")))))));
    }
}
```

## 5.3 边界与坑

- 资源内容可能含敏感配置，需鉴权与脱敏。
- 提示模板中的用户输入要防注入。
- 资源 URI 设计要有命名空间，避免多 Server 冲突。

## 5.4 生产建议

- 资源与提示的访问同样纳入鉴权。
- 对资源读取做缓存与大小限制。
- 提示模板变更走版本管理，避免影响在线会话。

# 6. 多 Server 聚合

结论先行：一个 Client 可连接多个 Server，聚合时最大的坑是**工具名冲突**，需要通过前缀或分组消歧。

## 6.1 聚合实现

```java
@Service
public class McpToolAggregator {

    private final List<ToolCallbackProvider> providers;

    public List<ToolCallback> allTools() {
        return providers.stream()
                .flatMap(p -> Arrays.stream(p.getToolCallbacks()))
                .toList();
    }
}
```

## 6.2 命名冲突处理

| 策略 | 说明 | 代价 |
|------|------|------|
| 加前缀 | `filesystem__read_file` | 名称变长 |
| 分组 | 按场景分别注入 | 逻辑更复杂 |
| 优先级 | 同名保留其一 | 可能丢能力 |

## 6.3 边界与坑

- 同名工具参数可能不同，简单覆盖会造成误调用。
- 聚合后工具数量膨胀，注意上下文预算。
- 单个 Server 故障不应导致整体不可用，需要隔离。

## 6.4 生产建议

- 为每个 Server 建立独立的超时与熔断配置。
- 聚合工具时保留来源信息，便于日志与排障。
- 定期校验工具清单，发现冲突及时告警。

# 7. 健康检查与可观测性

结论先行：MCP Server 是外部依赖，必须纳入健康检查与指标监控，才能在故障时快速定位。

## 7.1 健康检查

```java
@Component
@RequiredArgsConstructor
public class McpHealthIndicator implements HealthIndicator {

    private final ToolCallbackProvider mcpProvider;

    @Override
    public Health health() {
        try {
            int toolCount = mcpProvider.getToolCallbacks().length;
            return Health.up()
                    .withDetail("mcpTools", toolCount)
                    .build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

## 7.2 指标与日志

| 观测项 | 建议 |
|--------|------|
| 工具数量 | 作为健康指标 |
| 调用次数 | 按工具/用户打点 |
| 错误率 | 区分业务与连接错误 |
| 耗时 | 记录 P95/P99 |
| 审计 | 记录调用与参数摘要 |

## 7.3 边界与坑

- 健康检查不宜过重，避免频繁触发远端调用。
- 日志要脱敏，工具参数可能含隐私数据。
- 指标基数要控制，工具名 + 用户可能爆炸。

## 7.4 生产建议

- 健康检查区分"存活"与"就绪"，就绪需确认 MCP 可用。
- 对连接失败设置告警阈值。
- 审计日志单独存储，与业务日志分离。

# 8. 生产注意事项

结论先行：Spring AI 让接入变得简单，但"简单"也意味着默认行为偏开放，生产必须主动收紧。

## 8.1 风险清单

| 风险 | 建议 |
|------|------|
| 全量暴露工具 | 按权限裁剪 |
| 工具名冲突 | 加前缀/分组 |
| Server 不可用 | 超时 + 熔断 + 降级 |
| 无健康检查 | 加 HealthIndicator |
| 配置错误 | 启动时校验连接 |
| 无审计 | 记录 MCP 调用 |

## 8.2 安全与审计

- 所有写操作工具需二次确认或权限校验。
- 记录"谁在何时调用了哪个工具、参数摘要、结果状态"。
- 对敏感工具设置更严格的超时与限流。

## 8.3 部署建议

- Server 与 Client 分开部署，独立扩缩容。
- 远程连接统一走网关鉴权。
- 启动时校验 MCP 连接，失败可配置为快速失败或降级启动。

## 8.4 常见问题速查

| 症状 | 可能原因 | 处理 |
|------|----------|------|
| 启动变慢/失败 | 某 Server 不可达 | 配置失败策略 |
| 模型不调用工具 | 工具未注册或描述不清 | 检查 Provider 与描述 |
| 调用报无权限 | 裁剪过度 | 核对权限映射 |
| 工具名重复 | 多 Server 同名 | 加前缀/分组 |
| 会话中途失效 | 凭证过期 | 自动刷新 |

# 9. 小结

## 9.1 要点回顾

- Spring AI 提供 MCP Server/Client 全套 starter。
- Server 用 `@Tool` + `ToolCallbackProvider` 暴露能力。
- Client 配置连接即可把 MCP 工具纳入模型工具列表。
- **按权限裁剪 MCP 工具**是生产必备。
- 多 Server 聚合时注意工具名冲突与健康检查。

## 9.2 落地检查清单

- [ ] Server 协议模式与部署方式匹配。
- [ ] Client 关闭全量暴露，叠加权限裁剪。
- [ ] 工具命名空间清晰，无冲突。
- [ ] 超时、熔断、降级策略就绪。
- [ ] 健康检查与审计日志接入。
- [ ] 启动校验连接，明确失败策略。
