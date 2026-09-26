# 1. 固定查询工具

## 1.1 工具定义

### 1.1.1 结论先行

固定查询工具是系统的第一道能力，也是安全基线。SQL 在代码中写死，只有参数可变，且参数必须做类型、范围与枚举校验。任何用户输入都不得进入 SQL 文本，只能作为绑定参数。

### 1.1.2 完整代码

```java
@Component
@RequiredArgsConstructor
public class QueryTools {

    private final JdbcTemplate jdbc;

    @Tool("按日期范围查询订单汇总（最多 100 条）")
    public List<Map<String, Object>> queryOrdersByDate(
            @ToolParam(description = "开始日期 yyyy-MM-dd") String startDate,
            @ToolParam(description = "结束日期 yyyy-MM-dd") String endDate) {

        return jdbc.queryForList("""
                SELECT DATE(created_at) AS day,
                       COUNT(*) AS order_count,
                       SUM(amount) AS total_amount
                FROM orders
                WHERE created_at BETWEEN ?::date AND ?::date
                GROUP BY DATE(created_at)
                ORDER BY day
                LIMIT 100
                """, startDate, endDate);
    }

    @Tool("按区域查询销售额")
    public List<Map<String, Object>> querySalesByRegion(
            @ToolParam(description = "统计周期，如 2026-08") String period) {

        return jdbc.queryForList("""
                SELECT region, SUM(amount) AS total_amount, COUNT(*) AS order_count
                FROM orders
                WHERE TO_CHAR(created_at, 'YYYY-MM') = ?
                GROUP BY region
                ORDER BY total_amount DESC
                LIMIT 100
                """, period);
    }
}
```

### 1.1.3 参数校验

| 参数 | 校验 |
|------|------|
| startDate | 匹配 yyyy-MM-dd，且 start <= end |
| endDate | 匹配 yyyy-MM-dd，且不超过今天 |
| period | 匹配 yyyy-MM |
| region | 属于预定义枚举 |

## 1.2 注册、调用与边界

### 1.2.1 注册与调用流程

`@Tool` 注解的方法会被框架扫描并注册为可调用工具，模型通过函数调用（function calling）触发。注册时需暴露清晰的工具名与描述，避免模型误选。

```
用户问题
   │
   ▼
模型选择工具 + 抽取参数
   │
   ▼
参数校验（失败则拒绝）
   │
   ▼
执行写死的 SQL（绑定参数）
   │
   ▼
返回结果
```

### 1.2.2 边界与生产建议

- 坑：工具描述模糊导致模型传错参数，需在 `@ToolParam` 中写清格式。
- 坑：参数为日期字符串，需在工具内做严格解析，不能直接拼接。
- 反例：为省事把整个 WHERE 子句作为参数传入，等于开放注入。
- 工具数量控制在可枚举范围，定期清理低频工具；统一加日志与埋点；结果统一走行数限制。

# 2. Text2SQL

## 2.1 生成与执行

### 2.1.1 结论先行

Text2SQL 是长尾覆盖手段，流程为"生成 → 提取 → 校验 → 权限注入 → 执行"。其中校验与权限注入是硬约束，任何一步不通过都直接拒绝。

### 2.1.2 完整代码

```java
@Service
@RequiredArgsConstructor
public class Text2SqlService {

    private final ChatClient chatClient;
    private final JdbcTemplate jdbc;
    private final SqlValidator validator;

    public QueryResult query(String question, UserContext user) {
        // ① 生成 SQL
        String sql = chatClient.prompt()
                .system(SQL_SYSTEM_PROMPT)
                .user(question)
                .call().content();

        sql = extractSql(sql);                         // 清理 Markdown 包裹

        // ② 校验
        SqlValidationResult validation = validator.validate(sql);
        if (!validation.safe()) {
            throw new UnsafeQueryException(validation.reason());
        }
        sql = validation.sql();

        // ③ 注入行级权限
        sql = permissionService.injectRowLevelFilter(sql, user);

        // ④ 执行（只读 + 超时）
        try (Connection conn = readOnlyDataSource.getConnection()) {
            conn.setReadOnly(true);
            conn.setNetworkTimeout(executor, 10_000);

            List<Map<String, Object>> rows = jdbc.queryForList(sql);
            return new QueryResult(sql, limit(rows, 100));
        } catch (Exception e) {
            return QueryResult.error(sql, e.getMessage());
        }
    }
}
```

### 2.1.3 流程拆解

| 步骤 | 输入 | 输出 | 失败处理 |
|------|------|------|----------|
| 生成 | 问题 + 提示 | 原始 SQL 文本 | 重试 |
| 提取 | 原始文本 | 纯 SQL | 拒绝 |
| 校验 | SQL | 安全 SQL | 拒绝 |
| 注入 | SQL + 用户 | 带权限 SQL | 拒绝 |
| 执行 | SQL | 结果集 | 降级 |

## 2.2 校验与权限注入

### 2.2.1 校验器实现

```java
public SqlValidationResult validate(String sql) {
    String normalized = normalize(sql);
    if (!normalized.startsWith("SELECT") && !normalized.startsWith("WITH")) {
        return unsafe("只允许 SELECT");
    }
    if (containsForbidden(normalized) || normalized.contains(";")
            || normalized.contains("--") || normalized.contains("/*")) {
        return unsafe("禁止写操作/多语句/注释");
    }
    if (!normalized.contains("LIMIT")) {
        normalized += " LIMIT 100";
    }
    if (!ALLOWED_TABLES.containsAll(extractTables(normalized))) {
        return unsafe("访问未授权表");
    }
    return safe(normalized);
}
```

### 2.2.2 权限注入

```java
public String injectRowLevelFilter(String sql, UserContext user) {
    if (user.hasRole("ADMIN")) {
        return sql;
    }
    String predicate = "region = '" + escape(user.region()) + "'";
    return "SELECT * FROM (" + sql + ") t WHERE " + predicate;
}
```

### 2.2.3 边界与坑

- 坑：关键字匹配必须用词边界，避免 `updated_at` 被误判为 `UPDATE`。
- 坑：包裹子查询后外层字段名可能冲突，需保留原列名。
- 反例：仅检查是否以 SELECT 开头，`WITH ... DELETE` 可绕过。

## 2.3 重试机制

### 2.3.1 策略与代码

执行失败时，将数据库错误信息回喂给模型让其修正，最多重试 1~2 次；重试仍失败则回退固定工具或提示用户。

```java
public QueryResult queryWithRetry(String question, UserContext user) {
    String feedback = "";
    for (int i = 0; i < 2; i++) {
        SqlValidationResult v = validator.validate(extractSql(generate(question, feedback)));
        if (!v.safe()) {
            feedback = "校验失败: " + v.reason();
            continue;
        }
        try {
            return execute(v.sql(), user);
        } catch (Exception e) {
            feedback = "执行失败: " + e.getMessage();
        }
    }
    throw new QueryFailedException("重试后仍失败");
}
```

重试须设次数上限与总耗时上限；校验失败与执行失败分开计数；生成阶段设较低温度以减少随机性。

# 3. SQL 提取与清理

## 3.1 提取逻辑

### 3.1.1 结论先行

模型输出常带 Markdown 代码块或解释性文字，必须清洗为纯 SQL 再校验。清洗是安全前置步骤，不能跳过。

### 3.1.2 代码

```java
private String extractSql(String raw) {
    String cleaned = raw.trim();
    // 移除 Markdown 代码块
    if (cleaned.startsWith("```")) {
        cleaned = cleaned.replaceFirst("^```(sql)?\\s*", "")
                .replaceFirst("\\s*```$", "");
    }
    return cleaned.trim();
}
```

### 3.1.3 边界

| 情况 | 处理 |
|------|------|
| 前后有解释文字 | 提取首个代码块或首条语句 |
| 有 ```sql 标记 | 去除标记 |
| 多个代码块 | 只取第一个 |
| 无代码块 | 直接 trim |

## 3.2 规范化与建议

### 3.2.1 处理清单

规范化需去除多余空白与换行、统一去除末尾分号、剥离注释、处理全角字符。清洗后需再次校验，防止清洗引入绕过；保留原始输出用于审计；对无法提取的输出直接拒绝。

# 4. 结果分析

## 4.1 分析服务

### 4.1.1 结论先行

分析服务把结果集转为 Markdown 表格后交给 LLM，要求给出关键发现、趋势与建议，并强调"基于数据、不编造"。空结果不调用模型。

### 4.1.2 完整代码

```java
@Service
@RequiredArgsConstructor
public class AnalysisService {

    private final ChatClient chatClient;

    public String analyze(String question, List<Map<String, Object>> rows) {
        if (rows.isEmpty()) {
            return "查询结果为空。";
        }
        return chatClient.prompt()
                .system("""
                        你是数据分析师。基于查询结果给出分析：
                        1. 关键发现（2~3 条）；
                        2. 趋势或异常；
                        3. 一条建议。
                        简洁、基于数据，不编造。
                        """)
                .user("问题：%s\n\n数据：\n%s".formatted(question, toTable(rows)))
                .call().content();
    }

    private String toTable(List<Map<String, Object>> rows) {
        // 转 Markdown 表格
        if (rows.isEmpty()) return "";
        Set<String> columns = rows.get(0).keySet();
        StringBuilder sb = new StringBuilder("| " + String.join(" | ", columns) + " |\n");
        sb.append("|").append("---|".repeat(columns.size())).append("\n");
        for (Map<String, Object> row : rows) {
            sb.append("| ").append(columns.stream()
                    .map(c -> String.valueOf(row.get(c)))
                    .collect(Collectors.joining(" | "))).append(" |\n");
        }
        return sb.toString();
    }
}
```

### 4.1.3 表格转换说明

| 要点 | 说明 |
|------|------|
| 列顺序 | 按结果集列序，避免乱序 |
| 空值 | 统一渲染为 "-" |
| 数值 | 保留原始精度 |
| 行数 | 超限先截断 |

## 4.2 提示、边界与降级

### 4.2.1 提示要点与反例

提示需明确角色（数据分析师）、输出结构（发现/趋势/建议）与约束（简洁、基于数据、不编造）。反例：不限制结构则输出长篇大论；不强调"基于数据"则给出与数据无关的泛泛之谈。

### 4.2.2 边界与降级

结果为空直接返回固定文案，不调用模型，避免幻觉；行数或列数过多时先做预聚合或截断，再交给模型。分析结果可缓存（同问题 + 同数据），设分析超时，超时先返回数据，并对分析文本做敏感信息过滤。

# 5. 图表生成

## 5.1 生成规则

### 5.1.1 结论先行

图表生成优先用规则：第一列作维度，后续数值列作指标，按数据形态选择图表类型。规则失败或结果不适合可视化时返回空图表。

### 5.1.2 完整代码

```java
@Service
public class ChartGenerator {

    public ChartConfig generate(String question, List<Map<String, Object>> rows) {
        if (rows.isEmpty() || rows.get(0).size() < 2) {
            return ChartConfig.none();
        }
        // 简单规则：第一列作维度，数值列作指标
        Set<String> columns = rows.get(0).keySet();
        Iterator<String> it = columns.iterator();
        String dimension = it.next();
        String metric = it.next();

        return new ChartConfig(
                "bar",
                rows.stream().map(r -> String.valueOf(r.get(dimension))).toList(),
                rows.stream().map(r -> toNumber(r.get(metric))).toList(),
                dimension, metric);
    }
}
```

### 5.1.3 类型选择规则

| 条件 | 图表 |
|------|------|
| 维度为时间 | line |
| 维度类别 <= 8 且单一指标 | bar |
| 单一维度占比 | pie |
| 多指标 | grouped-bar |

## 5.2 数据结构与边界

### 5.2.1 ChartConfig 定义

```java
public record ChartConfig(
        String type,
        List<String> xAxis,
        List<Number> series,
        String dimension,
        String metric) {

    public static ChartConfig none() {
        return new ChartConfig("none", List.of(), List.of(), null, null);
    }
}
```

前端按 `type` 分派到 ECharts 等图表库，`xAxis` 与 `series` 直接映射。

### 5.2.2 边界与坑

- 反例：把 ID 列当维度，图表无意义。
- 反例：数值列含 null，图表渲染异常，需过滤或补零。
- 图表是增强项，失败必须降级为表格；配置生成做缓存；大数据量时先聚合再绘图。

# 6. 审计

## 6.1 审计实现

### 6.1.1 结论先行

所有查询必须审计：谁、问了什么、执行了什么 SQL、返回多少行、何时、traceId。审计是不可绕过的一环。

### 6.1.2 完整代码

```java
public void auditQuery(String question, String sql,
                       int rowCount, UserContext user) {
    auditLog.record(new QueryAudit(
            user.userId(),
            question,
            maskSensitive(sql),
            rowCount,
            Instant.now(),
            MDC.get("traceId")));
}

private String maskSensitive(String sql) {
    return sql.replaceAll("'(\\d{11})'", "'****'");
}
```

## 6.2 审计存储

### 6.2.1 表结构

```sql
CREATE TABLE query_audit (
    id          BIGSERIAL PRIMARY KEY,
    user_id     VARCHAR(64) NOT NULL,
    question    TEXT        NOT NULL,
    sql_masked  TEXT        NOT NULL,
    row_count   INT         NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    trace_id    VARCHAR(64)
);
```

### 6.2.2 生产建议

审计写入走异步队列，避免阻塞主链路；队列积压需告警，防止审计丢失。审计日志保留期按合规要求设定，写入失败要告警而非静默，查询支持按用户与时间检索。

# 7. 配置与部署

## 7.1 数据源配置

### 7.1.1 只读连接池

```java
@Bean("readOnlyDataSource")
public DataSource readOnlyDataSource() {
    HikariConfig cfg = new HikariConfig();
    cfg.setJdbcUrl("jdbc:postgresql://db:5432/dw");
    cfg.setUsername("readonly_user");
    cfg.setReadOnly(true);
    cfg.setMaximumPoolSize(10);
    cfg.setConnectionTimeout(3_000);
    return new HikariDataSource(cfg);
}
```

### 7.1.2 配置项

| 配置 | 值 | 说明 |
|------|----|------|
| setReadOnly | true | 连接只读 |
| maximumPoolSize | 10 | 限制并发 |
| connectionTimeout | 3000ms | 连接超时 |
| networkTimeout | 10000ms | 查询超时 |

## 7.2 应用配置

### 7.2.1 超时与限流

| 项 | 建议 |
|----|------|
| 生成超时 | 5s |
| 执行超时 | 10s |
| 分析超时 | 5s |
| 单用户 QPS | 按需限流 |

### 7.2.2 配置示例

```yaml
analysis:
  timeout:
    generate: 5s
    execute: 10s
    analyze: 5s
  limit:
    max-rows: 100
    max-concurrent-per-user: 2
```

超时分环节配置并以总超时兜底；连接池大小结合数据库容量设定；灰度期间降低并发上限。

# 8. 难点与解决

## 8.1 难点对照

### 8.1.1 对照表

| 难点 | 解决 |
|------|------|
| SQL 注入 | 七层防护 + 只读账号 |
| 全表扫描 | 强制 LIMIT + 超时 |
| 越权 | 行级权限注入 |
| SQL 错误 | 表结构提示 + 重试 |
| 结果解读 | LLM 分析 |
| 复杂查询 | 优先固定工具 |

## 8.2 反例与边界

### 8.2.1 常见反例

- 反例：把权限过滤放在模型提示中，模型可能遗漏。
- 反例：不限制结果行数，大表查询拖垮数据库。
- 反例：审计同步写入，慢查询导致请求超时。

### 8.2.2 边界处理

| 边界 | 处理 |
|------|------|
| 空结果 | 返回固定文案 |
| 校验失败 | 抛异常拒绝 |
| 执行超时 | 降级返回 |
| 图表失败 | 回退表格 |

关键路径全部有测试覆盖；每个异常都要有明确错误码；上线前跑完整安全用例。

# 9. 小结

## 9.1 核心结论

- 固定查询工具优先，Text2SQL 严格防护。
- 七层安全：只读账号、白名单、校验、LIMIT、超时、行数、审计。
- 行级权限在 SQL 中强制注入。
- 结果用 LLM 分析与图表生成。
- 所有查询必须审计。
- 清洗、校验、注入、执行四步不可跳过任何一步。

## 9.2 决策口诀

> 能固定工具就不放 Text2SQL；能只读就绝不写；权限过滤靠代码不靠提示词；先拒绝再降级，绝不静默失败。
