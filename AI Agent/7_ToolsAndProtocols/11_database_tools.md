# 1. 数据库工具的两种模式

## 1.1 固定查询工具

**结论先行**：把常用查询封装为带明确参数的固定工具，是生产环境访问数据库的首选方式。

### 1.1.1 定义与特点

固定查询工具指开发者预先编写 SQL（Structured Query Language，结构化查询语言）或调用 Repository（仓储）方法，只把有限的业务参数暴露给模型。SQL 本身由开发者掌控，模型无法改写。

| 特点 | 说明 |
|------|------|
| SQL 可控 | 语句固定，模型只能填参数 |
| 权限收敛 | 每个工具对应明确的数据范围 |
| 易审计 | 工具名即业务语义 |
| 灵活度低 | 新查询需新增工具 |

### 1.1.2 适用场景

- 高频、稳定的业务查询，如按订单号查询、按日期范围统计。
- 涉及敏感数据的查询，需要严格控制返回字段。
- 对响应时间有明确要求的场景。

## 1.2 自然语言转 SQL

**结论先行**：Text2SQL 让模型根据自然语言生成 SQL，灵活度极高但风险极大，仅适合探索性分析，且必须配合严格防护。

### 1.2.1 定义与特点

模型根据用户问题与表结构提示，生成 SQL 语句，再由应用执行。它把"写查询"的能力交给了模型，因此能覆盖开发者未预料的组合。

### 1.2.2 主要风险

- SQL 注入：模型可能被诱导生成恶意语句。
- 越权访问：可能查询未授权的表或字段。
- 全表扫描：缺少索引与 LIMIT，拖垮数据库。
- 数据泄露：通过报错或结果泄露敏感信息。

## 1.3 模式对比与选型

| 模式 | 安全 | 灵活 | 适用 |
|------|:----:|:----:|------|
| 固定查询 | 高 | 低 | 生产推荐 |
| Text2SQL | 低 | 高 | 探索性分析 |

> 生产优先用**固定查询工具**；Text2SQL 需严格防护。

## 1.4 边界与生产建议

**边界与坑**：

- 固定查询工具过多会造成工具选择困难，应按领域归类。
- Text2SQL 在表结构复杂时准确率骤降，不要迷信模型能力。

**生产建议**：

- 默认只提供固定查询，Text2SQL 作为受限的补充能力。
- 任何进入生产的 Text2SQL 都必须经过完整防护链。

# 2. 固定查询工具实现

## 2.1 设计原则

**结论先行**：固定查询工具必须做到参数化、限制条数、只返回必要字段，三者缺一不可。

### 2.1.1 参数化

参数必须通过占位符传入，禁止字符串拼接 SQL，从根源上杜绝注入。

```java
// 正确：使用占位符
String sql = "SELECT * FROM orders WHERE status = ? AND created_at >= ?";
jdbc.query(sql, rowMapper, status, startDate);
```

### 2.1.2 限制条数

每次查询都必须设置上限，避免一次拉回全表。

```java
List<Order> orders = orderRepository.findByDateRange(startDate, endDate, 50);
```

### 2.1.3 只返回必要字段

| 原则 | 反例 | 正例 |
|------|------|------|
| 字段最小化 | 返回整行 | 返回 id/status/amount |
| 隐藏敏感列 | 返回手机号 | 返回脱敏手机号 |
| 控制体积 | 返回明细 | 返回聚合统计 |

## 2.2 完整示例

```java
@Component
@RequiredArgsConstructor
public class OrderQueryTools {

    private final OrderRepository orderRepository;

    @Tool("按订单号查询订单")
    public ToolResult getOrderById(
            @ToolParam(description = "订单号，如 A123") String orderId) {

        return orderRepository.findById(orderId)
                .map(o -> ToolResult.ok(toSummary(o)))
                .orElse(ToolResult.fail("订单不存在：" + orderId));
    }

    @Tool("按日期范围查询订单（最多返回 50 条）")
    public ToolResult queryOrdersByDate(
            @ToolParam(description = "开始日期 yyyy-MM-dd") String startDate,
            @ToolParam(description = "结束日期 yyyy-MM-dd") String endDate) {

        List<Order> orders = orderRepository.findByDateRange(
                startDate, endDate, 50);
        return ToolResult.ok(orders.stream().map(this::toSummary).toList());
    }

    private Map<String, Object> toSummary(Order o) {
        return Map.of(
                "orderId", o.id(),
                "status", o.status(),
                "amount", o.amount(),
                "createdAt", o.createdAt());
    }
}
```

> 固定查询**参数化、限制条数、只返回必要字段**。

## 2.3 边界与生产建议

**边界与坑**：

- 日期参数要在服务端解析校验，非法格式直接拒绝。
- 分页游标比 OFFSET 更稳定，深分页时 OFFSET 性能急剧下降。

**生产建议**：

- Repository 层限制最大返回条数，工具层再次兜底。
- 为高频查询建立合适索引，并用监控观察慢查询。

# 3. 只读账号与最小权限

## 3.1 数据库层权限模型

**结论先行**：Agent 使用的数据库账号必须是只读的，且只能访问必要表，这是最后一道、也是最可靠的防线。

```sql
-- 为 Agent 创建只读账号
CREATE USER agent_readonly WITH PASSWORD 'xxx';
GRANT CONNECT ON DATABASE app TO agent_readonly;
GRANT USAGE ON SCHEMA public TO agent_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO agent_readonly;

-- 禁止写操作
REVOKE INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public FROM agent_readonly;
```

> **最小权限**：Agent 用的账号只读，且只能访问必要表。

## 3.2 PostgreSQL 配置示例

```sql
-- 只授权必要的表
GRANT SELECT ON orders, order_items TO agent_readonly;
REVOKE SELECT ON users, payment_methods FROM agent_readonly;

-- 行级安全：只能看到自己租户的数据
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::bigint);
```

## 3.3 多数据源隔离

| 数据源 | 用途 | 权限 |
|--------|------|------|
| 只读源 | Agent 查询 | SELECT |
| 主源 | 业务写入 | 全权限 |

## 3.4 边界与生产建议

**边界与坑**：

- 只读账号仍可执行 `SELECT pg_sleep(3600)` 等资源消耗语句，需配合超时。
- 行级安全依赖会话变量，连接池复用时要确保变量被正确设置与清理。

**生产建议**：

- 为只读账号设置独立连接池与并发上限。
- 定期审计账号权限，移除历史遗留授权。

# 4. Text2SQL 实现

## 4.1 执行流程

**结论先行**：Text2SQL 的执行必须经过"生成 → 校验 → 只读执行 → 限行 → 返回"五步，任何一步不可省略。

```
用户问题
   |
   v
[模型生成 SQL] --> [SQL 校验] --> [只读连接执行]
                                       |
                                       v
                              [限制行数] --> [结构化返回]
```

## 4.2 代码示例

### 4.2.1 工具定义

```java
@Tool("""
        查询订单数据库（只读）。
        表结构：
        - orders(id, user_id, amount, status, created_at)
        - order_items(order_id, sku, quantity, price)
        仅支持 SELECT 查询。
        """)
public ToolResult queryDatabase(
        @ToolParam(description = "SQL 查询语句（仅 SELECT）") String sql) {
    return executor.run(sql);
}
```

### 4.2.2 执行封装

```java
public ToolResult run(String sql) {
    // ① SQL 校验
    SqlValidationResult validation = sqlValidator.validate(sql);
    if (!validation.safe()) {
        return ToolResult.fail("SQL 校验未通过：" + validation.reason());
    }

    // ② 只读连接
    try (Connection conn = readOnlyDataSource.getConnection()) {
        conn.setReadOnly(true);
        // ③ 超时
        conn.setNetworkTimeout(executor, 5000);
        // ④ 限制行数
        List<Map<String, Object>> rows = jdbc.queryForList(sql);
        return ToolResult.ok(limit(rows, 100));
    } catch (Exception e) {
        return ToolResult.fail("查询失败：" + e.getMessage());
    }
}
```

## 4.3 边界与生产建议

**边界与坑**：

- `setReadOnly` 只是提示，某些驱动并不强制，真正的保障来自只读账号。
- 校验通过的 SQL 仍可能全表扫描，必须配合语句超时与资源组限制。

**生产建议**：

- 把执行封装为独立组件，便于统一埋点与限流。
- 对 Text2SQL 单独限流，避免挤占固定查询的资源。

# 5. SQL 安全校验

## 5.1 校验层次

**结论先行**：SQL 校验应包含"首关键字、危险关键字、多语句、注释、LIMIT、表白名单"六个层次。

## 5.2 关键词与语法校验

### 5.2.1 白名单与黑名单

白名单优于黑名单：只允许 SELECT，其余一律拒绝；黑名单作为补充，拦截明显危险关键字。

### 5.2.2 多语句与注释

| 检查项 | 目的 |
|--------|------|
| 分号 | 阻止多语句执行 |
| `--` 与 `/*` | 阻止注释绕过 |
| 表名 | 限制访问范围 |

## 5.3 完整校验器

```java
@Component
public class SqlValidator {

    private static final Set<String> FORBIDDEN_KEYWORDS = Set.of(
            "INSERT", "UPDATE", "DELETE", "DROP", "TRUNCATE",
            "ALTER", "CREATE", "GRANT", "REVOKE", "EXEC", "EXECUTE");

    public SqlValidationResult validate(String sql) {
        String upper = sql.toUpperCase().trim();

        // ① 必须以 SELECT 开头
        if (!upper.startsWith("SELECT")) {
            return SqlValidationResult.unsafe("只允许 SELECT 查询");
        }

        // ② 禁止危险关键字
        for (String keyword : FORBIDDEN_KEYWORDS) {
            if (upper.contains(keyword)) {
                return SqlValidationResult.unsafe("包含禁止的关键字: " + keyword);
            }
        }

        // ③ 禁止多语句
        if (sql.contains(";") && sql.indexOf(';') < sql.length() - 1) {
            return SqlValidationResult.unsafe("不允许多语句执行");
        }

        // ④ 禁止注释（防绕过）
        if (sql.contains("--") || sql.contains("/*")) {
            return SqlValidationResult.unsafe("不允许 SQL 注释");
        }

        // ⑤ 强制 LIMIT
        if (!upper.contains("LIMIT")) {
            sql = sql + " LIMIT 100";
        }

        // ⑥ 表白名单
        Set<String> tables = extractTables(sql);
        if (!ALLOWED_TABLES.containsAll(tables)) {
            return SqlValidationResult.unsafe("访问了未授权的表");
        }

        return SqlValidationResult.safe(sql);
    }
}
```

## 5.4 边界与生产建议

**边界与坑**：

- 字符串包含判断会被大小写、空白、注释等变形绕过，黑名单不可靠。
- 正则解析 SQL 难以覆盖子查询、CTE（Common Table Expression，公共表表达式）等复杂语法。

**生产建议**：

- 优先使用成熟 SQL 解析器（Parser）做 AST（Abstract Syntax Tree，抽象语法树）级校验。
- 校验器要有充分的对抗性测试用例。

# 6. 纵深防御体系

## 6.1 七层防护

**结论先行**：数据库工具的安全依赖七层防护叠加，任一层被绕过，其他层仍能兜住。

```
① 只读账号（数据库层）
② 表白名单（应用层）
③ SQL 语法校验（应用层）
④ 强制 LIMIT（应用层）
⑤ 查询超时（连接层）
⑥ 结果行数限制（应用层）
⑦ 审计日志（记录所有查询）
```

> **纵深防御**：任一层被绕过，其他层仍能兜住。

数据库层（只读账号、行级安全、表级授权）是最终屏障；应用层负责表白名单、语法校验、强制 LIMIT 与结果限行；连接层提供语句超时、连接池上限与只读事务。

## 6.2 各层失效场景

| 层 | 可能的失效 | 兜底 |
|----|------------|------|
| 只读账号 | 配置错误给了写权限 | 语法校验 |
| 表白名单 | 解析遗漏 | 数据库授权 |
| 语法校验 | 变形绕过 | 只读账号 |
| 超时 | 未设置 | 连接池隔离 |

## 6.3 边界与生产建议

**边界与坑**：

- 多层防护会增加复杂度，需确保配置集中管理，避免遗漏。
- 防护层之间要有独立监控，任一层频繁拦截都应告警。

**生产建议**：

- 把防护配置写成代码并纳入评审。
- 定期做红队测试，验证各层是否真正生效。

# 7. Text2SQL 提示工程

## 7.1 提示设计要点

**结论先行**：Text2SQL 的系统提示必须明确表结构、允许的操作、必须遵守的约束，以及不确定时的行为。

## 7.2 完整系统提示

```java
@SystemMessage("""
        你是 SQL 生成器。根据用户问题生成只读 SQL 查询。

        数据库表结构：
        orders(id, user_id, amount, status, created_at)
          - status: PAID/CANCELLED/REFUNDED
        order_items(order_id, sku, quantity, price)

        规则：
        1. 只生成 SELECT 语句；
        2. 必须包含 LIMIT（最多 100）；
        3. 不确定时不要猜测，说明需要什么信息；
        4. 不要生成任何写操作。

        只输出 SQL，不要其他内容。
        """)
```

## 7.3 边界与生产建议

**边界与坑**：

- 表结构提示过长会占用大量 Token，应只注入相关表。
- 模型可能忽略"只输出 SQL"，需要输出解析与兜底。

**生产建议**：

- 维护动态 Schema 注入，按问题检索相关表。
- 对生成 SQL 记录样本，持续评估准确率。

# 8. 结果处理与呈现

## 8.1 结果结构设计

**结论先行**：查询结果应结构化为 `count + columns + rows`，并限制行数、标注列名，方便模型理解。

```java
private ToolResult formatQueryResult(List<Map<String, Object>> rows) {
    if (rows.isEmpty()) {
        return ToolResult.ok(Map.of("count", 0, "rows", List.of()));
    }
    return ToolResult.ok(Map.of(
            "count", rows.size(),
            "columns", rows.get(0).keySet(),
            "rows", rows.stream().limit(100).toList()));
}
```

> 结果要**限制行数、结构化、标注列名**。

## 8.2 列裁剪与脱敏

| 处理 | 说明 |
|------|------|
| 列裁剪 | 去掉无关列 |
| 值脱敏 | 手机号、身份证打码 |
| 截断 | 超长文本截断并标记 |

## 8.3 边界与生产建议

**边界与坑**：

- 空结果与查询失败要区分，空结果不应报错。
- 大字段（如 JSON、TEXT）会迅速撑爆上下文，需截断。

**生产建议**：

- 为结果体积设定上限，超限时提示模型缩小范围。
- 提供分页参数，支持"继续获取"。

# 9. 审计与合规

## 9.1 审计事件模型

**结论先行**：所有数据库查询都必须记录审计事件，包含谁、查了什么、返回多少、耗时多久。

```java
public ToolResult queryDatabase(String sql, UserContext user) {
    long start = System.currentTimeMillis();

    ToolResult result = doQuery(sql);

    auditLog.record(new AuditEvent(
            "DB_QUERY",
            user.userId(),
            Map.of(
                    "sql", maskSensitive(sql),
                    "rowCount", result.rowCount(),
                    "durationMs", System.currentTimeMillis() - start),
            Instant.now()));

    return result;
}
```

## 9.2 敏感信息脱敏

| 内容 | 处理 |
|------|------|
| 手机号 | 中间四位打码 |
| 身份证 | 保留首尾 |
| 密码类字段 | 直接剔除 |
| SQL 中的字面量 | 记录时脱敏 |

## 9.3 边界与生产建议

**边界与坑**：

- 审计日志本身可能成为泄露源，必须加密存储与访问控制。
- 审计不能阻塞主流程，应异步写入。

**生产建议**：

- 审计日志独立存储，保留期符合合规要求。
- 支持按用户、表、时间检索审计记录。

# 10. 生产注意事项与小结

## 10.1 风险清单

| 风险 | 建议 |
|------|------|
| 写操作 | 只读账号 |
| SQL 注入 | 参数化 + 校验 |
| 全表扫描 | 强制 LIMIT + 超时 |
| 访问敏感表 | 表白名单 |
| 无审计 | 记录所有查询 |
| Text2SQL 不可控 | 固定查询优先 |
| 结果过大 | 限制行数 |

## 10.2 上线检查表

- Agent 账号是否为只读且仅授权必要表。
- 所有查询是否参数化，是否存在字符串拼接。
- 是否强制 LIMIT 并设置语句超时。
- Text2SQL 是否经过校验器与表白名单。
- 查询结果是否裁剪、限行、脱敏。
- 是否记录审计日志并支持检索。
- 是否有慢查询与拦截告警。

## 10.3 小结

- 生产优先用固定查询工具，Text2SQL 需严格防护。
- 只读账号 + 表白名单 + SQL 校验 + 强制 LIMIT。
- 纵深防御，任一层被绕过仍有兜底。
- 查询要超时、限行、审计。
- Text2SQL 的提示要明确表结构与限制。
