# 1. 为什么工具必须鉴权

## 1.1 工具是最后的出口

### 1.1.1 模型不可信假设

**结论先行**：必须假设模型随时可能被注入劫持，因此不能把"是否该调用工具"的判断权完全交给模型。

```
模型可能被注入劫持 → 要求调用危险工具
若工具无鉴权 → 危险操作直接成功
```

### 1.1.2 出口失守的后果

**结论先行**：工具是 Agent 与真实世界交互的出口，出口失守意味着后端系统被直接暴露。

| 失守点 | 后果 |
|--------|------|
| 无鉴权 | 越权操作直接成功 |
| 仅挂载层校验 | 被绕过即无保护 |
| 无归属校验 | 横向读取他人数据 |
| 无审计 | 事后无法追责 |

## 1.2 鉴权的两个层次

### 1.2.1 挂载层

挂载层按权限裁剪工具，做到"不挂载等于不可能被调用"。它拦截大部分错误调用，但依赖编排代码正确。

### 1.2.2 执行层

执行层在每个工具内部校验角色、数据归属与参数。即使挂载层被绕过（如工具被其他路径复用），仍能兜底。

## 1.3 边界与生产建议

**边界**：只做挂载层裁剪，工具一旦被别处直接调用就失去保护；只做执行层校验，则错误工具仍会进入模型视野、浪费步数。

**生产建议**：双层都要做；把工具内的鉴权当作"不可省略的最后防线"；鉴权失败返回结构化错误并审计。

# 2. 权限模型

## 2.1 三要素

### 2.1.1 主体 × 客体 × 条件

**结论先行**：一次授权判断可拆成三个问题——谁、对什么工具、在什么条件下。

```
权限 = 主体（谁） × 客体（什么工具） × 条件（什么情况）
```

### 2.1.2 各维度说明

| 维度 | 说明 |
|------|------|
| 主体 | 用户、角色、租户、Agent 实例 |
| 客体 | 具体工具、工具分组 |
| 条件 | 时间、频率、金额、数据范围 |

## 2.2 模型选择

### 2.2.1 RBAC 与 ABAC 对比

**结论先行**：RBAC（Role-Based Access Control，基于角色）简单直观；ABAC（Attribute-Based Access Control，基于属性）表达力强但更复杂，实践中常组合使用。

| 维度 | RBAC | ABAC |
|------|------|------|
| 判断依据 | 角色 | 属性（角色/数据/金额/时间） |
| 表达力 | 中 | 强 |
| 维护成本 | 低 | 高 |
| 适用 | 粗粒度工具授权 | 细粒度、上下文相关 |

## 2.3 边界与生产建议

**边界**：RBAC 难以表达"同一工具在不同金额下权限不同"，ABAC 则容易因策略膨胀而难以维护。

**生产建议**：以 RBAC 为骨架做工具可见性，用 ABAC 补充金额、数据范围等条件；策略集中存储并版本化。

# 3. RBAC：基于角色的工具授权

## 3.1 原理

### 3.1.1 角色到工具映射

**结论先行**：为每个角色维护一份可调用工具集合，用户通过其角色集合间接获得工具权限。

### 3.1.2 多角色合并

一个用户可能有多个角色，其可用工具是各角色工具集的并集；管理员角色通常用通配符表示全部工具。

## 3.2 完整 Java 示例

### 3.2.1 权限服务

```java
public enum Role { USER, SUPPORT, ADMIN }

public class ToolPermissionService {

    private static final Map<Role, Set<String>> ROLE_TOOLS = Map.of(
            Role.USER, Set.of("queryOrder", "queryProduct", "queryPolicy"),
            Role.SUPPORT, Set.of("queryOrder", "queryProduct", "queryPolicy",
                                 "refundOrder", "createTicket"),
            Role.ADMIN, Set.of("*")        // 全部
    );

    public Set<String> allowedTools(UserContext user) {
        Set<String> allowed = new HashSet<>();

        for (Role role : user.roles()) {
            Set<String> tools = ROLE_TOOLS.getOrDefault(role, Set.of());
            if (tools.contains("*")) return Set.of("*");
            allowed.addAll(tools);
        }
        return allowed;
    }
}
```

### 3.2.2 挂载裁剪

```java
// 挂载时按权限裁剪
List<ToolCallback> callbacks = allTools.stream()
        .filter(t -> permissionService.canUse(user, t.name()))
        .toList();

Assistant assistant = AiServices.builder(Assistant.class)
        .chatModel(model)
        .tools(toToolObjects(callbacks))
        .build();
```

## 3.3 边界与坑

| 坑 | 说明 |
|----|------|
| 通配符滥用 | `*` 让白名单失效 |
| 角色爆炸 | 角色过多难维护 |
| 未刷新缓存 | 角色变更未生效 |
| 只挂载不校验 | 绕过挂载即失守 |

## 3.4 生产建议

**结论先行**：不挂载等于不可能被调用，比"调用了再拒绝"更安全。`*` 仅限管理员显式声明；角色变更后刷新缓存并审计。

# 4. ABAC：基于属性的授权

## 4.1 原理

### 4.1.1 属性维度

**结论先行**：ABAC 把角色、数据范围、金额、时间等都当作属性，逐条比对策略。

### 4.1.2 与 RBAC 的关系

ABAC 不是替代 RBAC，而是在其之上补充上下文条件；可以先按角色收敛工具，再用属性做二次判断。

## 4.2 完整 Java 示例

```java
public class AttributeBasedToolAuth {

    public boolean canUse(UserContext user, ToolCall call) {
        ToolPolicy policy = policyStore.get(call.name());

        // ① 角色
        if (!policy.roles().stream().anyMatch(user.roles()::contains)) {
            return false;
        }

        // ② 数据范围
        if (policy.dataScoped() && !user.canAccess(call.dataScope())) {
            return false;
        }

        // ③ 金额阈值
        if (policy.amountLimited()) {
            BigDecimal amount = call.amount();
            if (amount.compareTo(user.approvalLimit()) > 0) {
                return false;           // 需审批
            }
        }

        // ④ 时间窗口
        if (!policy.allowedHours().contains(LocalTime.now().getHour())) {
            return false;
        }
        return true;
    }
}
```

## 4.3 边界与坑

**边界**：属性可能来自不可信输入（如模型生成的金额），必须做类型与范围校验后再参与判断。

**反例**：把金额直接取自模型输出，攻击者可传负数或超大值绕过阈值判断。

## 4.4 生产建议

属性来源必须可信（服务端注入）；策略集中管理并版本化；对拒绝结果记录原因，便于排查。

# 5. 工具内鉴权（最后防线）

## 5.1 双层防护

**结论先行**：即使挂载层被绕过，工具内部也要校验角色、数据归属与限额，这是不可省略的最后防线。

## 5.2 完整 Java 示例

### 5.2.1 查询类工具

```java
@Component
@RequiredArgsConstructor
public class SecureOrderTools {

    private final OrderService orderService;
    private final SecurityContext securityContext;

    @Tool("查询订单详情")
    public ToolResult queryOrder(@P("订单号") String orderId) {
        UserContext user = securityContext.current();

        // ① 角色校验
        if (!user.hasRole("USER")) {
            return ToolResult.fail("无权限");
        }

        // ② 数据归属校验（防水平越权）
        if (!orderService.belongsTo(user.userId(), orderId)) {
            auditLog.record("TOOL_ACCESS_DENIED", user.userId(), "queryOrder", orderId);
            return ToolResult.fail("无权查询该订单");
        }

        return ToolResult.ok(orderService.get(orderId));
    }
}
```

### 5.2.2 写操作工具

```java
    @Tool("退款")
    public ToolResult refundOrder(
            @P("订单号") String orderId,
            @P("金额") BigDecimal amount) {

        UserContext user = securityContext.current();

        // ① 角色
        if (!user.hasRole("SUPPORT")) {
            return ToolResult.fail("无退款权限");
        }

        // ② 金额上限
        if (amount.compareTo(user.refundLimit()) > 0) {
            return ToolResult.fail("超出退款限额，需主管审批");
        }

        // ③ 幂等
        String key = "%s:%s:%s".formatted(user.userId(), orderId, amount);
        return ToolResult.ok(orderService.refundIdempotent(key, orderId, amount));
    }
}
```

## 5.3 边界与坑

| 坑 | 说明 |
|----|------|
| 校验放调用方 | 复用工具时无保护 |
| 只校验角色 | 漏掉数据归属 |
| 无幂等 | 重试造成重复副作用 |
| 抛异常 | 模型无法理解失败 |

## 5.4 生产建议

校验必须写在工具内部；写操作必须幂等；失败返回结构化错误并审计拒绝事件。

# 6. 数据级权限（防水平越权）

## 6.1 水平越权与垂直越权

**结论先行**：垂直越权是"低权限执行高权限操作"，水平越权是"访问同角色他人数据"，后者更隐蔽。

| 越权类型 | 说明 | 防护 |
|----------|------|------|
| 水平越权 | 访问同角色他人数据 | 归属校验 |
| 垂直越权 | 低权限执行高权限操作 | 角色校验 |

## 6.2 完整 Java 示例

```java
@Tool("查询客户信息")
ToolResult queryCustomer(@P("客户ID") String customerId) {
    UserContext user = securityContext.current();

    // 只能查自己负责的客户
    if (!customerService.isAssignedTo(customerId, user.userId())
            && !user.hasRole("ADMIN")) {
        return ToolResult.fail("无权访问该客户");
    }
    return ToolResult.ok(customerService.get(customerId));
}
```

## 6.3 边界与生产建议

**边界**：归属校验依赖业务字段正确；若数据模型缺少负责人字段，需在服务层补齐，否则无法防水平越权。

**生产建议**：所有"按 ID 取数"的工具都要做归属校验；用统一的归属判断抽象，避免各处实现不一致。

# 7. 工具分组与场景授权

## 7.1 场景收敛权限

**结论先行**：同一用户在不同场景下可用工具不同，场景是权限收敛的又一维度。

## 7.2 完整 Java 示例

```java
public class ScenarioToolMapping {

    private static final Map<String, Set<String>> SCENARIO_TOOLS = Map.of(
            "customer-service", Set.of("queryOrder", "queryPolicy", "createTicket"),
            "finance",          Set.of("queryOrder", "refundOrder", "queryInvoice"),
            "analytics",        Set.of("queryMetrics", "queryReport")
    );

    public Set<String> toolsFor(String scenario) {
        return SCENARIO_TOOLS.getOrDefault(scenario, Set.of());
    }
}
```

## 7.3 边界与生产建议

**边界**：场景必须由服务端确定，不能由用户或模型自由指定，否则可"切场景"提权。

**生产建议**：场景与角色、租户一起取交集；未配置场景默认空集。

# 8. 动态授权

## 8.1 运行中权限变化

**结论先行**：权限不应在会话开始时一次性算定，而要随运行状态动态调整。

## 8.2 完整 Java 示例

```java
public class DynamicToolAuth {

    public List<ToolCallback> resolveTools(UserContext user, AgentContext ctx) {
        Set<String> allowed = permissionService.allowedTools(user);

        // 运行中动态调整：如已退款成功，移除退款工具
        if (ctx.hasCompleted("refundOrder")) {
            allowed.remove("refundOrder");
        }

        // 上下文相关：仅在订单已确认时可调用发货
        if (!ctx.isOrderConfirmed()) {
            allowed.remove("shipOrder");
        }

        return allTools.stream()
                .filter(t -> allowed.contains(t.name()))
                .toList();
    }
}
```

## 8.3 边界与生产建议

**边界**：动态裁剪会增加状态管理复杂度；移除工具后若模型仍尝试调用，应返回明确错误。

**生产建议**：把"已完成的危险操作"从可用集中移除；状态变化实时生效，避免使用陈旧快照。

# 9. 权限拒绝的处理

## 9.1 结构化错误

**结论先行**：权限拒绝应返回结构化错误，而非抛异常，让模型知道"此路不通"，避免反复尝试。

## 9.2 完整 Java 示例

```java
@Tool("删除用户")
ToolResult deleteUser(@P("用户ID") String userId) {
    if (!securityContext.current().hasRole("ADMIN")) {
        // 返回结构化错误，而非抛异常
        return ToolResult.fail("无权限执行该操作");
    }
    return ToolResult.ok(userService.delete(userId));
}
```

## 9.3 边界与生产建议

**边界**：错误信息不应泄露内部细节（如"该用户存在但无权限"会暴露数据存在性）。

**生产建议**：统一错误结构（状态、原因、可否重试）；对拒绝事件计数并告警。

# 10. 权限审计

## 10.1 审计内容

**结论先行**：审计要能回答"谁在何时调用了什么工具、参数是什么、结果如何"。

## 10.2 完整 Java 示例

```java
public class ToolAuditLogger {

    public void log(ToolCall call, UserContext user, ToolResult result) {
        auditLog.record(new ToolAudit(
                user.tenantId(),
                user.userId(),
                call.name(),
                maskArgs(call.arguments()),        // 参数脱敏
                result.status(),
                Instant.now(),
                MDC.get("traceId")));
    }
}
```

## 10.3 边界与生产建议

**边界**：审计日志本身可能包含敏感信息，必须脱敏；同时要防止日志被篡改。

**生产建议**：参数脱敏后再落盘；审计写入与业务同事务或可靠异步；按 traceId 串联全链路。

# 11. 生产注意事项

## 11.1 风险与建议总表

| 风险 | 建议 |
|------|------|
| 工具无鉴权 | 每个工具内部校验 |
| 只挂载不校验 | 双层防护 |
| 无归属校验 | 防水平越权 |
| 权限写死 | 配置化或可动态调整 |
| 拒绝抛异常 | 返回结构化错误 |
| 无审计 | 记录所有工具调用 |
| 参数未脱敏 | 审计日志脱敏 |

## 11.2 常见反模式

**结论先行**：权限问题多源于"以为挂载层就够了"。

- 只在挂载层做校验。
- 归属校验散落各处、实现不一致。
- 权限拒绝抛异常导致模型重试。
- 审计记录明文参数。

# 12. 小结

## 12.1 核心结论

- 工具是 Agent 与真实世界交互的出口，必须鉴权。
- 权限模型：主体 × 客体 × 条件。
- 双层防护：挂载时按权限裁剪 + 工具内鉴权。
- 数据级权限防水平越权，角色校验防垂直越权。
- 权限拒绝返回结构化错误，并全程审计。

## 12.2 一句话记忆

**"挂载裁剪缩可见面，工具内校验守最后关；拒绝要说清，调用必留痕。"**
