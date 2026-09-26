# 1. 审计日志与普通日志的区别

结论先行：审计日志（Audit Log）用于追责与合规，普通应用日志（Application Log）用于排障，两者目标不同，不能混为一谈。

## 1.1 定位差异

### 1.1.1 普通日志的定位

- 普通日志用于排障（Troubleshooting）：关注“系统发生了什么”。
- 可采样、可轮转、可短期保留。

### 1.1.2 审计日志的定位

- 审计日志用于追责与合规（Accountability & Compliance）：关注“谁在何时做了什么”。
- 不可采样、不可篡改、长期保留。

## 1.2 对比表

### 1.2.1 六维对比

| 维度 | 普通日志 | 审计日志 |
|------|----------|----------|
| 目的 | 排障 | 追责、合规 |
| 内容 | 技术细节 | 谁、何时、做了什么 |
| 保留期 | 短（天级） | 长（月~年） |
| 完整性 | 可丢失 | 不可篡改 |
| 访问 | 开发可看 | 受控访问 |
| 写入 | 可采样 | 不可采样 |

### 1.2.2 不可采样的原因

> 审计日志不能采样：被丢弃的那条，可能正是追责所需。

# 2. 审计什么

结论先行：审计范围应覆盖认证、授权、数据、模型、工具、检索、变更、敏感操作八类事件，且不得记录会成为泄露源的内容。

## 2.1 审计事件范围

### 2.1.1 八类必审事件

```
AI 应用中必须审计的事件：
① 用户认证与授权
② 数据访问（读/写/删）
③ 模型调用（模型、Token、成本）
④ 工具调用（工具、参数、结果）
⑤ 检索访问（查询、命中的文档）
⑥ 权限变更
⑦ 配置变更
⑧ 敏感操作（导出、删除、外发）
```

### 2.1.2 事件分类表

| 类别 | 事件示例 |
|------|----------|
| 认证 | 登录成功/失败、令牌校验失败 |
| 授权 | 访问被拒绝（DENIED） |
| 数据 | 读取、修改、删除、导出 |
| 模型 | 调用、Token、成本、模型名 |
| 工具 | 工具名、参数摘要、结果状态 |
| 检索 | 查询、命中文档 ID |
| 变更 | 权限、配置、策略变更 |

## 2.2 不该审计的内容

审计日志本身不能成为泄露源，以下内容不应原样记录：

- 明文密码、密钥、令牌。
- 完整 PII（身份证号、银行卡号、手机号）。
- 完整 Prompt 与完整模型输出（除非脱敏且必要）。

凡不能确定安全的字段，一律脱敏后再落库；宁可牺牲可读性，不可牺牲安全性。

## 2.3 审计事件的采集位置

### 2.3.1 四层采集点

| 位置 | 采集内容 | 优点 | 缺点 |
|------|----------|------|------|
| 网关 | 认证、限流、IP | 统一入口 | 看不到业务语义 |
| 应用层 | 业务动作、决策 | 语义完整 | 需埋点 |
| 数据层 | 读写、SQL | 无遗漏 | 语义弱、量大 |
| 模型/工具层 | 调用、Token、参数 | 贴近 AI 行为 | 需框架支持 |

> 推荐“应用层为主、网关与数据层为辅”，避免只依赖单层。

# 3. 审计记录模型

结论先行：审计事件应是一张“不可变的事实表”，字段固定、语义清晰，便于长期查询与聚合。

## 3.1 字段设计

### 3.1.1 必填字段

| 字段 | 含义 | 是否必填 |
|------|------|----------|
| eventId | 事件唯一 ID | 是 |
| traceId | 链路 ID | 是 |
| timestamp | 时间戳（UTC） | 是 |
| tenantId | 租户 | 是 |
| userId | 用户 | 是 |
| action | 动作类型 | 是 |
| resource | 资源类型 | 是 |
| result | SUCCESS/DENIED/ERROR | 是 |

### 3.1.2 可选字段

| 字段 | 含义 | 是否必填 |
|------|------|----------|
| resourceId | 资源 ID | 否 |
| detail | 脱敏后的细节 | 否 |
| clientIp | 客户端 IP | 否 |
| userAgent | 客户端标识 | 否 |

## 3.2 事件模型实现

### 3.2.1 record 定义

```java
public record AuditEvent(
        String eventId,
        String traceId,
        Instant timestamp,
        String tenantId,
        String userId,
        String action,                 // MODEL_CALL / TOOL_CALL / DATA_ACCESS ...
        String resource,               // 目标资源
        String resourceId,
        String result,                 // SUCCESS / DENIED / ERROR
        Map<String, Object> detail,    // 脱敏后的细节
        String clientIp,
        String userAgent
) {}
```

使用 `record` 保证事件不可变；`detail` 存放结构化细节，便于后续查询与聚合。

# 4. 关键审计场景

结论先行：AI 应用的关键审计场景有五类——模型调用、工具调用、数据访问、权限与配置变更、检索访问，每类关注点不同。

## 4.1 模型调用审计

### 4.1.1 Advisor 实现

```java
public class ModelAuditAdvisor implements CallAdvisor {

    private final AuditLog auditLog;
    private final String modelName;

    @Override
    public ChatClientResponse adviseCall(ChatClientRequest request,
                                         CallAdvisorChain chain) {
        ChatClientResponse response = chain.nextCall(request);
        Usage usage = response.chatResponse().getMetadata().getUsage();

        auditLog.record(new AuditEvent(
                UUID.randomUUID().toString(),
                MDC.get("traceId"),
                Instant.now(),
                TenantContext.current(),
                UserContextHolder.current().userId(),
                "MODEL_CALL",
                "chat",
                modelName,
                "SUCCESS",
                Map.of(
                        "inputTokens", usage.getPromptTokens(),
                        "outputTokens", usage.getCompletionTokens(),
                        "model", modelName),
                null, null));

        return response;
    }

    @Override
    public String getName() { return "ModelAuditAdvisor"; }

    @Override
    public int getOrder() { return 300; }
}
```

### 4.1.2 Token 与成本

模型审计要记录 Token 与成本，既用于追责，也用于计费与配额核算。

## 4.2 工具调用审计

### 4.2.1 包装器实现

```java
public class ToolAuditWrapper {

    public ToolResult execute(ToolCall call, UserContext user) {
        long start = System.currentTimeMillis();
        ToolResult result;

        try {
            result = doExecute(call);
        } catch (Exception e) {
            result = ToolResult.error(e.getMessage());
        }

        auditLog.record(new AuditEvent(
                UUID.randomUUID().toString(),
                MDC.get("traceId"),
                Instant.now(),
                user.tenantId(),
                user.userId(),
                "TOOL_CALL",
                "tool",
                call.name(),
                result.success() ? "SUCCESS" : "FAILED",
                Map.of(
                        "args", maskArgs(call.arguments()),      // 参数脱敏
                        "durationMs", System.currentTimeMillis() - start),
                null, null));

        return result;
    }
}
```

工具参数可能含凭据，落库前必须走 `maskArgs` 脱敏，只保留可追溯摘要。

## 4.3 数据访问审计

### 4.3.1 读写记录

```java
public class DataAccessAuditor {

    public void recordRead(UserContext user, String resourceType,
                           String resourceId, int count) {
        auditLog.record(new AuditEvent(
                UUID.randomUUID().toString(),
                MDC.get("traceId"),
                Instant.now(),
                user.tenantId(),
                user.userId(),
                "DATA_READ",
                resourceType,
                resourceId,
                "SUCCESS",
                Map.of("count", count),
                null, null));
    }
}
```

批量读取与导出属于高风险动作，需额外记录数量、范围与目的。

## 4.4 权限与配置变更审计

### 4.4.1 变更模型

权限、配置、策略的变更属于高风险事件，必须记录“变更前/变更后”。

```java
public record ConfigChange(
        String operatorId,
        String target,
        String before,
        String after,
        Instant changedAt
) {}
```

| 变更类型 | 记录重点 |
|----------|----------|
| 权限授予 | 授予对象、权限项 |
| 权限回收 | 回收对象、权限项 |
| 策略变更 | 前后策略 diff |
| 密钥轮换 | 密钥 ID、生效时间 |

## 4.5 检索访问审计

### 4.5.1 记录内容

检索访问必须审计“查询内容、命中文档、过滤条件”，以证明未越权访问。

```java
public void recordRetrieval(UserContext user, String query,
                            List<String> hitDocIds) {
    auditLog.record(new AuditEvent(
            UUID.randomUUID().toString(),
            MDC.get("traceId"),
            Instant.now(),
            user.tenantId(),
            user.userId(),
            "RETRIEVAL",
            "knowledge_base",
            null,
            "SUCCESS",
            Map.of(
                    "queryHash", sha256(query),   // 查询内容哈希，避免存原文
                    "hitCount", hitDocIds.size(),
                    "docIds", hitDocIds),
            null, null));
}
```

> 检索审计能回答“这个用户看到了哪些文档”，是数据泄露追责的关键证据。

# 5. 审计日志脱敏

结论先行：审计日志本身不能成为泄露源，落库前必须同时做“字段级遮蔽”与“内容级脱敏”。

## 5.1 脱敏策略

### 5.1.1 字段级遮蔽

敏感字段整体遮蔽：如 `password`、`token`、`secret`。

### 5.1.2 内容级脱敏

字符串内容脱敏：识别其中的 PII 并替换。

## 5.2 实现

### 5.2.1 Masker 实现

```java
public class AuditMasker {

    private final PiiMasker piiMasker;

    public Map<String, Object> mask(Map<String, Object> detail) {
        Map<String, Object> masked = new HashMap<>();

        detail.forEach((k, v) -> {
            if (isSensitiveKey(k)) {
                masked.put(k, "***");                       // 敏感字段整体遮蔽
            } else if (v instanceof String s) {
                masked.put(k, piiMasker.mask(s));           // 字符串脱敏
            } else {
                masked.put(k, v);
            }
        });
        return masked;
    }

    private boolean isSensitiveKey(String key) {
        return Set.of("password", "apiKey", "token", "secret")
                .contains(key.toLowerCase());
    }
}
```

嵌套结构需递归处理，避免只脱敏顶层字段而漏掉内层 PII。

## 5.3 反例

### 5.3.1 四类反例

| 反例 | 风险 |
|------|------|
| 记录完整 Prompt | Prompt 可能含 PII 或机密 |
| 记录工具原始参数 | 参数可能含凭据 |
| 记录明文令牌 | 审计库成为密钥库 |
| 只遮蔽字段名不遮蔽内容 | 内容中的 PII 泄露 |

任何一条反例都可能导致“审计系统本身成为最大泄露源”。

# 6. 审计日志的存储

结论先行：审计存储要在“可查询、不可篡改、可长期保留”三者间平衡，通常采用热库查询 + 冷库归档的分层方案。

## 6.1 存储选型

| 存储 | 特点 | 适用 |
|------|------|------|
| 关系库 | 可查询、事务 | 中小规模 |
| Elasticsearch | 全文检索、聚合 | 大规模查询 |
| 对象存储 | 便宜、不可变 | 长期归档 |
| 专用审计系统 | 合规、不可篡改 | 强合规 |

中小规模用关系库起步；查询量大时引入 Elasticsearch；长期归档用对象存储并启用合规锁。

## 6.2 表结构与索引

### 6.2.1 DDL

```sql
CREATE TABLE audit_log (
    event_id     VARCHAR(64) PRIMARY KEY,
    trace_id     VARCHAR(64),
    ts           TIMESTAMP   NOT NULL,
    tenant_id    VARCHAR(64) NOT NULL,
    user_id      VARCHAR(64),
    action       VARCHAR(32) NOT NULL,
    resource     VARCHAR(64),
    resource_id  VARCHAR(128),
    result       VARCHAR(16) NOT NULL,
    detail       JSONB,
    client_ip    VARCHAR(64)
);
```

### 6.2.2 索引设计

```sql
-- 按月分区，便于归档
CREATE INDEX idx_audit_tenant_ts ON audit_log (tenant_id, ts DESC);
CREATE INDEX idx_audit_user_ts ON audit_log (user_id, ts DESC);
CREATE INDEX idx_audit_action ON audit_log (action, ts DESC);
```

## 6.3 分区与归档

- 按时间分区（如按月），便于归档与删除。
- 热数据留查询库，冷数据转对象存储。
- 归档数据保持不可变，附校验和。

## 6.4 写入链路与异步

### 6.4.1 异步管道

审计写入不应阻塞主流程，也不应因失败而丢失。

```
业务线程 ──► 内存队列 ──► 批量消费者 ──► 审计存储
                │
                └─► 本地磁盘兜底（队列满/宕机）
```

| 风险 | 应对 |
|------|------|
| 队列满 | 阻塞或落盘兜底 |
| 消费者宕机 | 重试 + 死信队列 |
| 进程崩溃 | 本地 WAL 先落盘 |
| 重复写入 | eventId 幂等去重 |

> 审计可以“稍后写入”，但不能“丢失写入”。

# 7. 不可篡改

结论先行：不可篡改需要“发现机制”与“阻止机制”双管齐下——链式哈希负责发现，WORM 存储负责阻止。

## 7.1 链式哈希

### 7.1.1 哈希链实现

```java
// 链式哈希，防止篡改
public class TamperProofAuditLog {

    private String previousHash = "GENESIS";

    public void append(AuditEvent event) {
        String content = serialize(event) + previousHash;
        String hash = sha256(content);

        store.save(new AuditRecord(event, previousHash, hash));
        previousHash = hash;              // 链接下一条
    }
}
```

### 7.1.2 校验

```java
    // 校验完整性
    public boolean verify() {
        List<AuditRecord> records = store.findAll();
        String prev = "GENESIS";

        for (AuditRecord r : records) {
            if (!r.previousHash().equals(prev)) return false;
            if (!sha256(serialize(r.event()) + prev).equals(r.hash())) return false;
            prev = r.hash();
        }
        return true;
    }
}
```

链式哈希（Hash Chain）让任何一条被篡改都会导致后续哈希不匹配，从而被发现。

## 7.2 只追加与 WORM

### 7.2.1 机制清单

| 机制 | 说明 |
|------|------|
| 只追加（Append-Only） | 不允许更新与删除 |
| WORM | 一次写入多次读取，存储层保证 |
| 异地备份 | 防止单点删除 |
| 数字签名 | 每条事件签名，防伪造 |

强合规场景优先 WORM + 数字签名；一般场景至少做到只追加。

## 7.3 存储层不可篡改

### 7.3.1 存储手段

| 手段 | 说明 |
|------|------|
| 对象存储合规锁 | 桶级 WORM 策略 |
| 数据库只读账号 | 应用仅有插入权限 |
| 定时锚定 | 周期性把哈希写入外部可信存储 |
| 异地备份 | 防止单点删除与勒索 |

> 链式哈希能“发现”篡改，存储层 WORM 能“阻止”篡改，二者应同时使用。

# 8. 审计查询

结论先行：审计查询往往发生在事故期间，必须在平时就建好索引与查询接口，并压测验证性能。

## 8.1 查询接口

### 8.1.1 接口定义

```java
public interface AuditQuery {
    List<AuditEvent> byUser(String userId, Instant from, Instant to);
    List<AuditEvent> byAction(String action, Instant from, Instant to);
    List<AuditEvent> byResource(String resourceId);
    List<AuditEvent> byTrace(String traceId);      // 完整链路
}
```

大结果集应分页或异步导出，避免拖垮在线库。

## 8.2 典型场景

| 场景 | 查询 |
|------|------|
| 追查某次操作 | 按 traceId |
| 用户行为分析 | 按 userId |
| 安全事件 | 按 action=DENIED |
| 合规审查 | 按时间段导出 |

每个场景都应能映射到已建索引，避免全表扫描。

## 8.3 查询性能与分区

### 8.3.1 优化手段

| 优化 | 说明 |
|------|------|
| 时间分区 | 按天/月分区，剪枝扫描 |
| 组合索引 | tenant + ts、user + ts |
| 冷热分离 | 热库查询、冷库归档 |
| 结果限制 | 大结果集分页/异步导出 |

> 审计查询往往发生在事故期间，性能必须在平时就压测验证。

# 9. 与 Trace 结合

结论先行：用同一个 traceId 把技术链路（Trace）与业务审计（Audit）关联，实现“既能排障又能追责”。

## 9.1 关联模型

### 9.1.1 traceId 桥接

```
Trace（技术链路）     ← 用于排障
  └── traceId
Audit（业务行为）     ← 用于追责
  └── traceId（关联）
```

审计事件与业务日志都携带同一 `traceId`，查询时可用它还原完整时间线。

## 9.2 实现

### 9.2.1 注入 traceId

```java
// 在入口生成 traceId 并写入 MDC 与 Reactor Context
String traceId = UUID.randomUUID().toString();
MDC.put("traceId", traceId);
```

| 维度 | Trace | Audit |
|------|-------|-------|
| 目的 | 排障 | 追责 |
| 粒度 | 技术调用 | 业务行为 |
| 保留 | 短 | 长 |
| 关联键 | traceId | traceId |

# 10. 生产注意事项

结论先行：审计系统的生产落地围绕脱敏、防篡改、保留、可查、关联、授权、异步、不采样八个要点展开。

## 10.1 风险清单

### 10.1.1 八项风险

| 风险 | 建议 |
|------|------|
| 审计日志含 PII | 必须脱敏 |
| 可被修改 | 链式哈希 / 只追加 |
| 无保留策略 | 按合规设保留期 |
| 无查询能力 | 建索引 + 查询接口 |
| 与 Trace 脱节 | 用 traceId 关联 |
| 访问无控制 | 审计日志受控访问 |
| 写入阻塞主流程 | 异步写入 |
| 采样丢弃 | 审计不采样 |

逐项建立检查项，纳入上线前的安全评审。

## 10.2 落地建议

- 审计与业务日志分开存储、分开授权。
- 审计写入做幂等与兜底，宁可延迟不可丢失。
- 定期校验哈希链并演练按 traceId 的追溯查询。

# 11. 小结

## 11.1 核心结论

### 11.1.1 五条结论

- 审计日志用于追责与合规，与普通日志定位不同。
- 审计范围：认证、数据访问、模型/工具调用、权限与配置变更。
- 审计日志本身必须脱敏与防篡改。
- 用 traceId 关联技术链路与业务审计。
- 异步写入，避免阻塞主流程。

审计日志是“不可采样的、不可篡改的、可追溯的事实记录”。

## 11.2 常见误区

### 11.2.1 四类误区

- 把普通日志当审计日志，保留期与完整性都不达标。
- 审计日志里记录完整 Prompt 与凭据。
- 审计可采样，导致关键事件缺失。
- 审计与业务系统同一权限，可被内部人员篡改。

把审计从“日志体系”中独立出来，单独存储、单独授权、单独保留。
