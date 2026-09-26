# 1. 什么是 Agent 生命周期

Agent 作为可演进的软件资产，也有"版本、发布、运行、下线"的过程：

```
定义 → 版本化 → 测试 → 发布 → 灰度 → 运行 → 下线
```

其中"定义"包括：提示、工具集、模型配置、权限策略。

## 1.1 生命周期阶段

| 阶段 | 目标 | 产出 |
|------|------|------|
| 定义 | 描述 Agent 行为 | AgentDefinition |
| 版本化 | 固化不可变快照 | version |
| 测试 | 验证正确性 | 单测 / 评测报告 |
| 发布 | 切换生效版本 | ACTIVE |
| 灰度 | 小流量验证 | 灰度指标 |
| 运行 | 承接真实流量 | Trace / 指标 |
| 下线 | 停止使用 | DEPRECATED |

## 1.2 为什么需要生命周期管理

没有生命周期管理，会出现典型问题：

- 改一行 Prompt 直接上生产，出问题无法定位是哪个版本。
- 无法回滚，只能紧急改代码再发一次。
- 多环境配置不一致，测试通过线上翻车。
- 审计缺失，谁在什么时候改了什么无从追溯。

## 1.3 与其他模块的关系

```
配置治理（Architecture 4） ─┐
可观测（Observability）     ─┼─▶ 生命周期管理
CI/CD（Deployment 2/3）     ─┘
```

生命周期管理是"配置治理 + 发布流程 + 可观测"在 Agent 维度的统一。

# 2. 需要版本化的对象

## 2.1 版本化清单

| 对象 | 说明 |
|------|------|
| Prompt | 提示模板 |
| 工具集 | 挂载了哪些工具 |
| 模型配置 | 模型、参数 |
| 记忆策略 | 窗口大小、压缩策略 |
| 权限策略 | 工具授权范围 |
| 依赖版本 | 框架、SDK |

## 2.2 定义数据结构

```java
public record AgentDefinition(
        String agentId,
        String version,
        String systemPrompt,
        List<String> toolNames,
        ModelConfig model,
        MemoryConfig memory,
        AutonomyPolicy autonomy,
        Instant publishedAt
) {}
```

子结构示例：

```java
public record ModelConfig(String model, double temperature, int maxTokens) {}

public record MemoryConfig(int windowSize, String compression) {}

public record AutonomyPolicy(
        Set<String> allowedTools, boolean requireApproval) {}
```

> 用不可变 `record` 承载定义，天然适合版本快照与比较。

## 2.3 什么不该版本化

并非所有东西都进版本库：

- **密钥与凭证**：走密钥管理，不进定义（只放引用）。
- **运行时状态**：会话、记忆内容属于数据，不属于定义。
- **环境差异**：区域、副本数走环境配置，不混入定义。

# 3. 版本管理

## 3.1 存储接口

```java
public interface AgentDefinitionStore {
    AgentDefinition get(String agentId, String version);
    AgentDefinition getLatest(String agentId);
    AgentDefinition getActive(String agentId);       // 当前生效版本
    void publish(AgentDefinition definition);
    void activate(String agentId, String version);   // 切换生效版本
}
```

补充常用方法：

```java
AgentDefinition getCandidate(String agentId);        // 灰度候选
List<AgentDefinition> history(String agentId);       // 历史版本
void updateStatus(String agentId, String version, Status status);
```

## 3.2 数据库设计

### 3.2.1 定义表

```sql
CREATE TABLE agent_definition (
    agent_id     VARCHAR(64) NOT NULL,
    version      VARCHAR(32) NOT NULL,
    definition   JSONB       NOT NULL,
    status       VARCHAR(16) NOT NULL,   -- DRAFT / ACTIVE / DEPRECATED
    published_at TIMESTAMP   NOT NULL,
    PRIMARY KEY (agent_id, version)
);
```

### 3.2.2 生效版本表

辅助表记录"当前生效版本"：

```sql
CREATE TABLE agent_active (
    agent_id     VARCHAR(64) PRIMARY KEY,
    version      VARCHAR(32) NOT NULL,
    activated_at TIMESTAMP   NOT NULL
);
```

## 3.3 版本号策略

| 策略 | 示例 | 优点 | 缺点 |
|------|------|------|------|
| 语义化 | 1.2.0 | 语义清晰 | 需人工判断 |
| 时间戳 | 20260924.1 | 有序、自动 | 无语义 |
| 内容哈希 | a1b2c3 | 可去重 | 不可读 |
| 自增序号 | 42 | 简单 | 跨环境冲突 |

推荐：`语义化 + 内容哈希短码`，如 `1.2.0-a1b2c3`，兼顾可读与去重。

# 4. 发布流程

## 4.1 状态流转

```
DRAFT ──测试──▶ CANDIDATE ──灰度──▶ ACTIVE ──下线──▶ DEPRECATED
```

| 阶段 | 动作 |
|------|------|
| DRAFT | 开发中，不影响线上 |
| CANDIDATE | 通过单测与评测 |
| ACTIVE | 生效版本 |
| DEPRECATED | 保留但不再使用 |

## 4.2 发布实现

```java
public void publish(String agentId, String version) {
    AgentDefinition def = store.get(agentId, version);

    // 1. 校验
    validate(def);

    // 2. 跑评测回归
    EvalReport report = evaluator.evaluate(def);
    if (!report.passed()) {
        throw new IllegalStateException("评测未通过: " + report);
    }

    // 3. 激活
    store.activate(agentId, version);
    auditLog.record("PUBLISH", agentId, version);
}
```

## 4.3 发布门禁

发布前应通过的门禁：

| 门禁 | 检查项 | 阻断级别 |
|------|--------|----------|
| 结构校验 | 必填字段、工具存在性 | 阻断 |
| 单测 | 工具调用正确性 | 阻断 |
| 评测回归 | 成功率不低于基线 | 阻断 |
| 成本检查 | 单次成本不超预算 | 警告 |
| 安全扫描 | Prompt 注入防护 | 阻断 |

# 5. 灰度发布

## 5.1 灰度路由

```java
public AgentDefinition resolve(String agentId, UserContext user) {
    AgentDefinition active = store.getActive(agentId);

    // 灰度：按用户哈希决定是否使用新版本
    AgentDefinition candidate = store.getCandidate(agentId);
    if (candidate != null && inCanary(user, candidate.version(), 0.1)) {
        return candidate;                       // 10% 流量
    }
    return active;
}

private boolean inCanary(UserContext user, String version, double ratio) {
    int hash = Math.abs((user.userId() + version).hashCode());
    return (hash % 100) < ratio * 100;
}
```

> 同一用户 + 同一版本哈希稳定，保证用户不会在灰度中"忽新忽旧"。

## 5.2 灰度维度

| 维度 | 说明 | 适用 |
|------|------|------|
| 用户哈希 | 稳定分流 | 通用 |
| 租户 | 按租户整体切换 | B 端 |
| 地域 | 按区域灰度 | 多区域 |
| 白名单 | 内部用户优先 | 早期验证 |
| 流量比例 | 随机比例 | 压测 |

## 5.3 灰度观测与推进

推进节奏建议：

```
1% → 观察 30min → 5% → 观察 1h → 20% → 观察 4h → 100%
```

每步对比指标：成功率、P99 延迟、单次成本、工具错误率。任一显著劣化即暂停或回滚。

# 6. 热更新

## 6.1 原子替换

配置变更无需重启：

```java
@Component
public class HotReloadableAgent {

    private volatile Assistant assistant;
    private volatile String currentVersion;

    @EventListener
    public void onDefinitionChanged(DefinitionChangedEvent event) {
        rebuild(event.definition());
    }

    private synchronized void rebuild(AgentDefinition def) {
        Assistant next = AiServices.builder(Assistant.class)
                .chatModel(modelFactory.get(def.model()))
                .tools(toolRegistry.resolve(def.toolNames()))
                .chatMemoryProvider(memoryFactory.get(def.memory()))
                .build();

        this.assistant = next;                  // volatile 原子替换
        this.currentVersion = def.version();
    }
}
```

> 用 `volatile` + 原子替换实现无锁热更新；旧请求继续用旧实例直到完成。

## 6.2 无锁热更新的边界

### 6.2.1 常见边界

原子替换并非万能：

| 边界 | 说明 | 处理 |
|------|------|------|
| 长会话跨版本 | 会话中途切版本 | 会话内固定版本 |
| 状态不兼容 | 新版本读旧状态 | 状态带 schema 版本 |
| 构建失败 | 新实例构建异常 | 保留旧实例，告警 |
| 多实例不一致 | 部分实例未更新 | 版本广播 + 校验 |

### 6.2.2 构建失败的兜底

```java
private synchronized void rebuild(AgentDefinition def) {
    try {
        Assistant next = build(def);
        this.assistant = next;
        this.currentVersion = def.version();
    } catch (Exception e) {
        log.error("热更新失败，保留旧版本 {}: {}",
                currentVersion, e.getMessage());
        // 不替换，保持旧实例可用
    }
}
```

## 6.3 热更新与长会话

长会话建议"会话内固定版本"：会话创建时解析一次定义并绑定，整个会话复用，避免中途行为突变。

```java
SessionContext ctx = SessionContext.builder()
        .sessionId(sessionId)
        .definition(def)          // 会话内固定
        .build();
```

# 7. 回滚

## 7.1 回滚实现

```java
public void rollback(String agentId) {
    List<AgentDefinition> history = store.history(agentId);
    AgentDefinition previous = history.get(1);   // 上一个 ACTIVE

    store.activate(agentId, previous.version());
    log.warn("回滚 Agent {} 到版本 {}", agentId, previous.version());
}
```

回滚必须快：**配置切换即可，无需重新部署**。

## 7.2 回滚的边界条件

| 情况 | 是否可快速回滚 | 原因 |
|------|----------------|------|
| 仅 Prompt/参数变更 | 可 | 纯配置 |
| 工具集增删 | 可（需兼容） | 旧工具仍存在 |
| 状态 schema 变更 | 需谨慎 | 数据已按新格式写入 |
| 依赖升级 | 否 | 需重新部署 |

## 7.3 回滚演练

- 定期在生产/预发演练回滚，测量实际恢复时间（MTTR）。
- 回滚脚本化，避免依赖人工操作。
- 回滚后自动通知与记录审计。

# 8. 下线与废弃

## 8.1 废弃流程

```
DEPRECATED → 停止新流量 → 观察期 → 删除
```

```java
public void deprecate(String agentId, String version) {
    store.updateStatus(agentId, version, Status.DEPRECATED);
    // 观察期结束后清理
    scheduleCleanup(agentId, version, Duration.ofDays(30));
}
```

## 8.2 观察期与清理

### 8.2.1 清理流程

| 阶段 | 时长 | 动作 |
|------|------|------|
| 停止新流量 | 立即 | 新会话不再路由 |
| 在途会话放行 | 按最长会话 | 等待自然结束 |
| 观察期 | 7–30 天 | 监控是否仍被引用 |
| 清理 | 观察期后 | 删除定义与缓存 |

## 8.3 废弃的反例

- 直接删除仍在被引用的版本 → 线上报错。
- 观察期过短 → 长尾会话被截断。
- 无审计记录 → 事后无法复盘。

# 9. 生命周期与可观测

## 9.1 版本标签

```
AgentDefinition (版本)
        │
        ▼
Trace 中记录 agentId + version
        │
        ▼
按版本对比指标：成功率、延迟、成本、工具调用
```

```java
// Trace 属性
span.setAttribute("agent.id", def.agentId());
span.setAttribute("agent.version", def.version());
```

> 有了版本标签，才能判断"新版本是否更好"。

## 9.2 按版本对比指标

| 指标 | 含义 | 对比意义 |
|------|------|----------|
| success_rate | 任务成功率 | 新版本是否更稳 |
| p99_latency | 尾延迟 | 是否变慢 |
| cost_per_task | 单次成本 | 是否更贵 |
| tool_error_rate | 工具错误率 | 工具是否兼容 |
| token_usage | Token 消耗 | Prompt 是否膨胀 |

## 9.3 审计日志

```java
public record AuditEntry(
        Instant time,
        String actor,
        String action,       // PUBLISH / ROLLBACK / DEPRECATE
        String agentId,
        String version,
        String reason
) {}
```

审计日志需满足：不可篡改、可检索、保留期符合合规要求。

# 10. 生产注意事项

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 定义散落代码 | 集中版本化存储 |
| 无评测就发布 | 发布前跑评测 |
| 无法回滚 | 保留历史版本 + 快速切换 |
| 灰度无监控 | 按版本对比指标 |
| 热更新有状态泄漏 | 原子替换，旧实例自然退役 |
| 无审计 | 记录发布/回滚/下线 |

## 10.2 权限与审计

| 操作 | 所需权限 | 是否需审批 |
|------|----------|------------|
| 创建 DRAFT | 开发者 | 否 |
| 发布 CANDIDATE | 开发者 | 否 |
| 激活 ACTIVE | 发布者 | 是 |
| 回滚 | 值班/发布者 | 否（紧急） |
| 删除版本 | 管理员 | 是 |

## 10.3 配置漂移防护

- 定义只读，运行时不可修改。
- 定期扫描线上实际生效版本与期望版本是否一致。
- 多实例启动时校验版本一致性，不一致则告警。

# 11. 小结

## 11.1 核心要点

- Agent 定义（Prompt/工具/模型/权限）需要版本化管理。
- 发布流程：DRAFT → CANDIDATE → ACTIVE → DEPRECATED。
- 发布前必须通过评测，支持灰度与快速回滚。
- Trace 带版本标签，才能按版本对比效果。

## 11.2 落地建议

- 热更新用原子替换，避免重启。
- 回滚速度决定 MTTR，必须脚本化并定期演练。
- 长会话应在会话内固定版本，避免行为突变。
