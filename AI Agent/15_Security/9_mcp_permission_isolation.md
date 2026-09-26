# 1. 为什么 MCP 需要专门隔离

## 1.1 MCP 引入的新风险

### 1.1.1 外部工具即外部依赖

**结论先行**：MCP（Model Context Protocol，模型上下文协议）让 Agent 能接入外部提供的工具，把"工具"从内部代码变成了外部依赖，权限边界必须重新划定。

### 1.1.2 五类风险

**结论先行**：MCP 的风险集中在"外部不可控"这一点上，可归纳为五类。

| 风险 | 说明 |
|------|------|
| 危险工具 | 第三方 Server 暴露危险工具 |
| 权限串用 | 多用户共用一个 Client，串权限 |
| 恶意内容 | Server 被注入，返回恶意内容 |
| 诱导调用 | 工具描述诱导模型调用 |
| 供应链 | 不可信的 Server 与依赖 |

## 1.2 与传统工具的差异

### 1.2.1 权限边界重新划定

内部工具的鉴权由我们自己的代码控制；MCP 工具来自外部 Server，其行为、凭证与网络访问都不完全受控，因此需要 Server、工具、会话三层隔离。

## 1.3 边界与生产建议

**边界**：MCP 隔离不能替代工具内鉴权——即使工具被正确裁剪，Server 侧仍可能越权，必须同时做参数范围注入与审计。

**生产建议**：先建立"可信 Server 清单"，再在其上做工具裁剪与会话隔离；对外部 Server 默认不信任。

# 2. MCP 的三层隔离

## 2.1 三层模型

### 2.1.1 Server 级隔离

不同用户/租户使用不同的 Server 实例与凭证，避免共享高权限连接。

### 2.1.2 工具级隔离

同一 Server 内按用户权限裁剪工具，做到"不挂载等于不可调用"。

### 2.1.3 会话级隔离

凭证与上下文不跨会话，防止 A 用户的登录态、临时文件被 B 用户复用。

```
① Server 级隔离：不同用户/租户用不同 Server 实例
② 工具级隔离：同一 Server 内按权限裁剪工具
③ 会话级隔离：凭证与上下文不跨会话
```

## 2.2 层间关系

| 层 | 隔离对象 | 主要手段 |
|----|----------|----------|
| Server 级 | 连接与凭证 | 每租户独立实例 |
| 工具级 | 可见工具集 | 权限裁剪 |
| 会话级 | 运行时上下文 | 会话作用域 |

## 2.3 边界与生产建议

**边界**：三层隔离都有成本，Server 级隔离的连接数会随租户增长，需要连接池与生命周期管理。

**生产建议**：工具级隔离是核心，任何场景都必须做；Server 级与会话级按多租户、多用户并发情况加强。

# 3. Server 级隔离

## 3.1 每租户独立实例

**结论先行**：绝不能所有租户共用一个带高权限凭证的 MCP Client，否则无法区分责任，也无法隔离。

## 3.2 完整 Java 示例

### 3.2.1 注册表

```java
public class McpServerRegistry {

    // 每个租户独立的 Server 连接
    private final Map<String, McpClient> tenantClients = new ConcurrentHashMap<>();

    public McpClient clientFor(String tenantId) {
        return tenantClients.computeIfAbsent(tenantId, t ->
                McpClient.builder()
                        .serverName("tools-" + t)
                        .credentials(credentialProvider.forTenant(t))   // 租户凭证
                        .build());
    }
}
```

### 3.2.2 生命周期

```java
// 租户注销或凭证轮换时，关闭并移除对应连接
public void evict(String tenantId) {
    McpClient client = tenantClients.remove(tenantId);
    if (client != null) {
        client.close();
    }
}
```

## 3.3 边界与坑

| 坑 | 说明 |
|----|------|
| 连接泄漏 | 租户增多未回收 |
| 凭证共享 | 多租户共用账号 |
| 连接串用 | 缓存键用错维度 |
| 未处理失效 | 凭证轮换后旧连接仍用 |

## 3.4 生产建议

以租户为键缓存连接并设置上限；凭证轮换时主动驱逐；对连接数、错误率做监控。

# 4. 工具级权限隔离（核心）

## 4.1 不挂载等于不可调用

**结论先行**：MCP 工具级隔离的核心是按用户权限裁剪工具，把无权工具从模型中"消失"。

## 4.2 完整 Java 示例

### 4.2.1 过滤逻辑

```java
public class McpToolIsolation {

    private final McpClient mcpClient;
    private final ToolPermissionService permissionService;
    private final McpTrustService mcpTrustService;

    public List<ToolCallback> toolsFor(UserContext user) {
        // ① 获取 MCP Server 暴露的所有工具
        List<ToolCallback> allMcpTools = mcpClient.listTools();

        // ② 按用户权限过滤
        Set<String> allowed = permissionService.allowedTools(user);

        // ③ 按 Server 白名单过滤（只允许可信 Server）
        Set<String> trustedServers = mcpTrustService.trustedServers(user.tenantId());

        return allMcpTools.stream()
                .filter(t -> allowed.contains(t.name()))
                .filter(t -> trustedServers.contains(t.serverName()))
                .toList();
    }
}
```

### 4.2.2 挂载

```java
// 挂载时只暴露有权限的 MCP 工具
List<ToolCallback> tools = mcpToolIsolation.toolsFor(user);

chatClient.prompt()
        .user(question)
        .toolCallbacks(tools)
        .call()
        .content();
```

## 4.3 边界与坑

| 坑 | 说明 |
|----|------|
| 工具重名 | 不同 Server 同名工具混淆 |
| 未过滤 Server | 只按权限漏了可信校验 |
| 动态变更 | 权限变更未重新裁剪 |
| 描述注入 | 工具描述本身含指令 |

## 4.4 生产建议

工具名建议带 Server 前缀避免重名；权限与可信校验必须同时做；工具描述纳入审查。

# 5. 会话级隔离

## 5.1 会话状态风险

**结论先行**：会话级隔离防止 A 用户的 MCP 状态（登录态、临时文件）被 B 用户复用。

## 5.2 完整 Java 示例

```java
public class SessionScopedMcpClient {

    // 每会话独立的 MCP 会话，避免上下文串用
    public String chat(String sessionId, String question, UserContext user) {
        try (McpSession session = mcpClient.openSession(
                Map.of(
                        "sessionId", sessionId,
                        "userId", user.userId(),
                        "tenantId", user.tenantId()))) {

            List<ToolCallback> tools = isolateTools(session, user);

            return chatClient.prompt()
                    .user(question)
                    .toolCallbacks(tools)
                    .call()
                    .content();
        }   // 会话关闭，资源与凭证释放
    }
}
```

## 5.3 边界与坑

**边界**：会话作用域会增加创建开销，高频短会话场景需评估性能。

**反例**：把 MCP Session 缓存到全局 Map 并跨请求复用，等于取消会话隔离。

## 5.4 生产建议

会话结束必须释放资源；会话键包含用户与租户；禁止跨请求复用同一 Session。

# 6. MCP 工具的参数与结果安全

## 6.1 参数不可信

**结论先行**：MCP 工具的参数来自模型，必须视为不可信；服务端要强制注入租户与用户范围。

## 6.2 完整 Java 示例

### 6.2.1 参数校验与范围注入

```java
public class SecureMcpToolWrapper {

    public ToolResult call(ToolCall call, UserContext user) {
        // ① 参数校验
        validateArgs(call);

        // ② 数据范围注入（服务端强制）
        Map<String, Object> scopedArgs = new HashMap<>(call.arguments());
        scopedArgs.put("tenantId", user.tenantId());       // 强制注入，不信任模型
        scopedArgs.put("userId", user.userId());

        // ③ 调用
        ToolResult result = mcpClient.call(call.name(), scopedArgs);

        // ④ 结果清洗（防间接注入）
        String sanitized = injectionSanitizer.sanitize(result.content());

        // ⑤ 审计
        auditLog.record("MCP_CALL", call.name(), user.userId());

        return ToolResult.ok(sanitized);
    }
}
```

### 6.2.2 结果清洗

工具返回内容与检索内容一样不可信，需剥离其中的指令样式文本，并以"数据"身份注入上下文。

## 6.3 边界与坑

**边界**：范围注入要求 Server 侧配合；若 Server 忽略 tenantId，隔离就落空，需在契约中约定。

**反例**：把 tenantId 作为普通参数交给模型填写，攻击者可改写成他人租户。

## 6.4 生产建议

租户/用户范围由服务端注入且不可被模型覆盖；结果清洗后再入上下文；全量审计。

# 7. 第三方 MCP Server 的供应链治理

## 7.1 治理措施

**结论先行**：对第三方 Server 要从来源、版本、运行环境、能力、网络、审计六方面同时治理。

| 措施 | 说明 |
|------|------|
| 来源白名单 | 只接入审核过的 Server |
| 版本锁定 | 固定版本，避免自动升级 |
| 沙箱运行 | 第三方 Server 在沙箱内 |
| 能力审查 | 审查其暴露的工具与权限 |
| 出网管控 | 限制其可访问的网络 |
| 调用审计 | 记录所有调用 |

## 7.2 完整 Java 示例

```java
public class McpTrustService {

    private static final Set<String> TRUSTED_SERVERS = Set.of(
            "internal-file-server",
            "internal-db-server");

    public boolean isTrusted(String serverName) {
        return TRUSTED_SERVERS.contains(serverName);
    }
}
```

## 7.3 边界与坑

**边界**：白名单只能约束"接入哪个 Server"，不能约束 Server 内部行为，仍需沙箱与出网管控。

## 7.4 生产建议

白名单与版本清单纳入配置管理；第三方 Server 一律沙箱运行；定期复审其工具与权限。

# 8. MCP 凭证管理

## 8.1 禁止共用高权限凭证

**结论先行**：禁止所有用户共用一个 MCP 高权限凭证，否则无法区分责任，也无法隔离。

## 8.2 完整 Java 示例

```java
public interface McpCredentialProvider {

    // 每个用户/租户独立凭证，避免共用一个高权限账号
    McpCredentials forUser(UserContext user);
}

@Component
public class VaultMcpCredentialProvider implements McpCredentialProvider {

    private final SecretManager secretManager;

    @Override
    public McpCredentials forUser(UserContext user) {
        return secretManager.get(
                "mcp/%s/%s".formatted(user.tenantId(), user.userId()),
                McpCredentials.class);
    }
}
```

## 8.3 边界与坑

| 坑 | 说明 |
|----|------|
| 凭证落盘 | 明文写入配置文件 |
| 长期有效 | 无轮换机制 |
| 权限过大 | 凭证权限超出所需 |
| 缓存明文 | 内存中长期驻留 |

## 8.4 生产建议

凭证从密钥管理服务动态获取；按最小权限签发并定期轮换；内存中避免明文长期驻留。

# 9. MCP 安全配置示例

## 9.1 配置要点

**结论先行**：配置层面要做到"只启用可信连接 + 工具白名单"。

## 9.2 YAML 示例

```yaml
spring:
  ai:
    mcp:
      client:
        enabled: true
        # 只启用可信连接
        stdio:
          connections:
            internal-files:
              command: java
              args: ["-jar", "/opt/mcp/internal-file-server.jar"]
              # 不配置任何外部 Server
        # 工具白名单
        tool-filter:
          include: ["read_file", "list_files", "query_db"]
          exclude: ["delete_file", "execute_command"]
```

## 9.3 边界与坑

**边界**：白名单配置只覆盖显式列出的工具；新版本 Server 新增工具时，默认行为取决于框架（可能被放行）。

## 9.4 生产建议

采用"允许清单"而非"拒绝清单"；配置纳入版本管理与评审；升级 Server 后复核工具清单。

# 10. 越权测试

## 10.1 测试策略

**结论先行**：隔离是否有效必须用越权测试验证，而非依赖配置正确。

## 10.2 完整 Java 示例

```java
@Test
void user_A_cannot_use_user_B_mcp_tools() {
    // 用户 A 只能读自己租户的文件
    List<ToolCallback> toolsA = mcpToolIsolation.toolsFor(userA);

    ToolResult result = callTool(toolsA, "read_file",
            Map.of("path", "/tenant-B/secret.txt"));

    assertFalse(result.success());
    assertTrue(result.error().contains("无权"));
}

@Test
void untrusted_mcp_server_is_not_exposed() {
    List<ToolCallback> tools = mcpToolIsolation.toolsFor(user);

    assertTrue(tools.stream()
            .noneMatch(t -> "untrusted-server".equals(t.serverName())));
}
```

## 10.3 边界与生产建议

**边界**：测试用例要覆盖水平越权（跨用户）与垂直越权（跨权限），并覆盖不可信 Server。

**生产建议**：把越权测试纳入 CI；每次调整权限策略后回归；对越权尝试告警。

# 11. 生产注意事项

## 11.1 风险与建议总表

| 风险 | 建议 |
|------|------|
| 共享高权限凭证 | 按用户/租户独立凭证 |
| 全量暴露工具 | 按权限裁剪 |
| 信任第三方 Server | 来源白名单 + 沙箱 |
| 会话串用 | 会话级隔离 |
| 参数来自模型 | 服务端强制注入范围 |
| 结果未清洗 | 防间接注入 |
| 无审计 | 记录所有 MCP 调用 |

## 11.2 常见反模式

**结论先行**：MCP 事故多来自"把外部 Server 当内部工具用"。

- 全租户共用一个 MCP Client。
- 只做工具裁剪，不做参数范围注入。
- 直接信任工具返回内容。
- 凭证长期不轮换。

# 12. 小结

## 12.1 核心结论

- MCP 引入外部工具，权限边界必须重新划定。
- 三层隔离：Server 级、工具级、会话级。
- 核心是**工具级权限裁剪**：不挂载等于不可调用。
- 凭证按用户/租户独立，禁止共用高权限账号。
- 第三方 Server 需来源白名单 + 沙箱 + 审计。

## 12.2 一句话记忆

**"外部工具当外敌：按 Server 隔离、按工具裁剪、按会话清场，参数范围服务端说了算。"**
