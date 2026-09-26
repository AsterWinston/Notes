# 1. 整体架构

## 1.1 架构总览

### 1.1.1 结论先行

数据分析 Agent 的架构是一条主链路：意图理解 → 查询生成与校验 → 执行 → 结果分析 → 图表生成 → 报告输出。主链路之上叠加两层横切能力：安全防护（贯穿生成、校验、执行）与可观测性（审计、监控、追踪）。任何一环失败都要明确提示并支持降级，不得静默失败。

### 1.1.2 端到端流程

```
用户问题（自然语言）
   │
   ▼
① 意图理解
   ├── 简单查询 → 固定查询工具
   └── 复杂查询 → Text2SQL
   │
   ▼
② 查询生成与校验
   │
   ▼
③ 执行（只读 + 权限过滤 + LIMIT + 超时）
   │
   ▼
④ 结果分析（LLM 解读数据）
   │
   ▼
⑤ 图表生成（可选）
   │
   ▼
⑥ 输出报告
```

## 1.2 组件与分层

### 1.2.1 组件清单

| 组件 | 职责 | 关键约束 |
|------|------|----------|
| 意图路由器 | 判断走工具还是 Text2SQL | 只分类，不查询 |
| 固定工具集 | 确定性查询 | 参数校验、SQL 写死 |
| Text2SQL 服务 | 生成并执行 SQL | 必须过校验器 |
| SQL 校验器 | 语法与白名单检查 | 拒绝优先 |
| 权限服务 | 注入行/列级过滤 | 代码强制，非模型 |
| 执行器 | 只读连接执行 | 超时 + LIMIT |
| 分析服务 | LLM 解读结果 | 基于数据，不编造 |
| 图表服务 | 生成图表配置 | 失败降级为表格 |
| 审计服务 | 全量记录查询 | 不可绕过 |

### 1.2.2 调用链与分层

```
Controller
  └─> Orchestrator
        ├─> IntentRouter
        ├─> (Tool | Text2SqlService)
        │      └─> SqlValidator -> PermissionService -> Executor
        ├─> AnalysisService
        ├─> ChartService
        └─> AuditService
```

### 1.2.3 同步异步边界与坑

| 环节 | 模式 | 理由 |
|------|------|------|
| 意图与生成 | 同步 | 用户等待首字节 |
| 执行 | 同步（带超时） | 结果决定后续 |
| 分析 | 可异步补 | 先返回数据 |
| 图表 | 可异步补 | 前端先渲染表格 |
| 审计 | 异步 | 不阻塞主链路 |

- 反例：把安全校验写在 Controller，能力层被内部调用时绕过校验。校验必须在执行器入口强制。
- 反例：图表生成失败抛异常导致整个请求失败。图表是增强项，必须可降级。
- 坑：多轮上下文无限增长，需按窗口裁剪并保留"已确认口径"。

# 2. 两种查询模式

## 2.1 模式对比

### 2.1.1 结论先行

系统提供两条查询路径：固定查询工具（安全优先）与 Text2SQL（灵活优先）。生产默认走固定工具，Text2SQL 仅用于固定工具覆盖不到的长尾探索，且必须经过完整防护。

### 2.1.2 对比表与决策树

| 模式 | 安全 | 灵活 | 适用 | 成本 |
|------|:----:|:----:|------|:----:|
| 固定查询工具 | 高 | 低 | 常用查询 | 低 |
| Text2SQL | 低 | 高 | 探索性 | 高 |

```
问题到达
   │
   ├── 命中已注册工具？── 是 ──▶ 固定查询工具
   │
   └── 否
        ├── 用户有 Text2SQL 权限？── 否 ──▶ 拒绝并提示
        └── 是 ──▶ Text2SQL（带防护）
```

## 2.2 固定查询工具

### 2.2.1 工具契约

固定工具是一组预定义、参数化的查询方法，SQL 写死、仅参数可变。参数需做类型与范围校验，杜绝拼接注入。

```
优先固定查询工具：
  queryOrdersByDate(start, end)
  getOrderStats(period)
  querySalesByRegion(region)

Text2SQL 仅用于固定工具覆盖不到的场景
```

### 2.2.2 参数化示例

```java
@Tool("按区域查询销售额")
public List<Map<String, Object>> querySalesByRegion(String region) {
    if (!VALID_REGIONS.contains(region)) {
        throw new IllegalArgumentException("非法区域: " + region);
    }
    return jdbc.queryForList("""
            SELECT region, SUM(amount) AS total_amount
            FROM orders
            WHERE region = ?
            GROUP BY region
            LIMIT 100
            """, region);
}
```

## 2.3 Text2SQL 与路由

### 2.3.1 风险与防护

| 风险 | 防护 |
|------|------|
| 生成写操作 | 关键字校验 + 只读账号 |
| 全表扫描 | 强制 LIMIT + 超时 |
| 越权访问 | 表白名单 + 行级注入 |
| 幻觉表名 | 表结构提示 + 校验 |

### 2.3.2 混合路由与回退

高频问题命中工具可复用缓存、降低延迟与成本；长尾问题由 Text2SQL 兜底，保证覆盖率。意图识别失败时回退 Text2SQL 并加严校验；执行失败先重试一次（带错误信息），仍失败则回退固定工具或提示用户补充信息。

# 3. 安全设计（核心）

## 3.1 七层防护

### 3.1.1 七层清单

安全采用纵深防御：任何单层被绕过仍有其他层兜底。最内层是数据库只读账号，即使应用全被攻破也无法写库。

```
七层防护：
① 只读账号（数据库层）
② 表白名单（应用层）
③ SQL 语法校验（应用层）
④ 强制 LIMIT（应用层）
⑤ 查询超时（连接层）
⑥ 结果行数限制（应用层）
⑦ 审计日志
```

## 3.2 SQL 校验器

### 3.2.1 校验流程

先判首关键字是否为 SELECT/WITH，再查禁止关键字，再判多语句与注释，再补 LIMIT，最后抽取表名比对白名单。

### 3.2.2 完整代码

```java
public SqlValidationResult validate(String sql) {
    String upper = sql.toUpperCase().trim();

    if (!upper.startsWith("SELECT")) {
        return unsafe("只允许 SELECT");
    }
    for (String keyword : FORBIDDEN_KEYWORDS) {
        if (upper.contains(keyword)) {
            return unsafe("包含禁止关键字: " + keyword);
        }
    }
    if (sql.contains(";") && sql.indexOf(';') < sql.length() - 1) {
        return unsafe("不允许多语句");
    }
    if (sql.contains("--") || sql.contains("/*")) {
        return unsafe("不允许注释");
    }
    if (!upper.contains("LIMIT")) {
        sql = sql + " LIMIT 100";
    }
    Set<String> tables = extractTables(sql);
    if (!ALLOWED_TABLES.containsAll(tables)) {
        return unsafe("访问未授权表");
    }
    return safe(sql);
}
```

### 3.2.3 边界与坑

- 坑：仅用字符串包含判断关键字，可能被大小写或空白绕过，须先规范化。
- 坑：表名抽取用正则易漏（子查询、CTE），需覆盖 WITH 与嵌套。
- 反例：只做应用层校验、不配只读账号，一旦校验有漏洞即写库。

## 3.3 执行层防护

### 3.3.1 只读连接与超时

```java
try (Connection conn = readOnlyDataSource.getConnection()) {
    conn.setReadOnly(true);
    conn.setNetworkTimeout(executor, 10_000);
    List<Map<String, Object>> rows = jdbc.queryForList(sql);
    return limit(rows, 100);
}
```

生产建议：超时设语句级 + 连接级双重；结果行数超限时截断并提示"仅显示前 N 行"；大字段（如 JSON）单独限制长度。

# 4. 数据权限

## 4.1 权限层级

### 4.1.1 三层表

权限分表级、列级、行级三层，缺一不可。只做表级权限，行级数据仍会泄露；只做行级，敏感列仍会暴露。

| 权限层级 | 说明 |
|----------|------|
| 表级 | 只能访问授权的表 |
| 列级 | 隐藏敏感列（如身份证） |
| 行级 | 只能看自己的数据 |

## 4.2 行级权限注入

### 4.2.1 机制与代码

行级过滤必须在执行前由代码强制注入，不能依赖模型"记得加条件"。注入点在权限服务，位于校验之后、执行之前。

```java
// 行级权限：在生成的 SQL 中强制注入过滤
public String injectRowLevelFilter(String sql, UserContext user) {
    // 例：只能查自己区域的数据
    if (!user.hasRole("ADMIN")) {
        return sql.replace("WHERE", "WHERE region = '%s' AND".formatted(user.region()));
    }
    return sql;
}
```

### 4.2.2 边界与坑

- 坑：`replace("WHERE", ...)` 只处理第一个 WHERE，且会误伤子查询。
- 坑：无 WHERE 的查询需追加 WHERE，而非替换。
- 反例：把过滤条件交给模型拼接，模型漏加即越权。

更稳健的做法是包裹子查询，使条件作用于外层，避免误伤嵌套 WHERE：

```java
public String injectRowLevelFilter(String sql, UserContext user) {
    if (user.hasRole("ADMIN")) {
        return sql;
    }
    String predicate = "region = '" + escape(user.region()) + "'";
    return "SELECT * FROM (" + sql + ") t WHERE " + predicate;
}
```

## 4.3 列级权限与脱敏

### 4.3.1 敏感列与脱敏方式

在元数据中标注敏感列（id_card、phone、salary），查询改写时移除或脱敏。权限规则集中管理、配置化，权限变更需审计，并定期用越权用例回归。

| 列类型 | 脱敏方式 |
|--------|----------|
| 手机号 | 138****5678 |
| 身份证 | 保留前 3 后 4 |
| 姓名 | 张* |

# 5. 表结构提示

## 5.1 提示设计

### 5.1.1 结论先行

表结构提示是提升 Text2SQL 准确率最有效的手段。提示需包含表名、字段、枚举取值与生成规则，减少模型幻觉。

### 5.1.2 系统提示

```java
@SystemMessage("""
        你是 SQL 生成器。根据问题生成只读 SQL。

        可用表：
        orders(id, user_id, region, amount, status, created_at)
          - status: PAID/CANCELLED/REFUNDED
        order_items(order_id, sku, quantity, price)
        products(sku, name, category, price)

        规则：
        1. 只生成 SELECT；
        2. 必须包含 LIMIT（最多 100）；
        3. 不确定时不要猜，说明需要什么信息；
        4. 不要生成任何写操作。
        """)
```

### 5.1.3 提示要素与反例

| 要素 | 作用 |
|------|------|
| 表与字段 | 避免表名/字段名幻觉 |
| 枚举取值 | 避免状态值写错 |
| 生成规则 | 约束输出格式 |
| 口径说明 | 统一指标计算方式 |

- 反例：提示列出 100 张表，模型反而抓不住重点。应按问题检索相关表再注入。
- 反例：不说明枚举含义，模型把 status 写成数字。

## 5.2 元数据注入

### 5.2.1 动态提示

从元数据表动态生成表结构描述，避免提示与库结构脱节。核心逻辑：遍历表元数据，拼接"表名(列名列表)"，并追加枚举提示；实际使用前按问题检索相关表，控制注入规模。

生产建议：元数据变更后自动刷新提示；保留 few-shot 示例（问题 → SQL 对）；明确时间字段与口径（如"上月"如何计算）。

# 6. 分析与图表

## 6.1 结果分析

### 6.1.1 结论先行

执行完查询只是拿到数据，用户要的是"结论"。分析服务用 LLM 基于结果给出关键发现、趋势与建议，且必须严格基于数据、不编造。

### 6.1.2 代码

```java
// 结果分析
String analysis = chatClient.prompt()
        .system("分析查询结果，给出关键洞察与趋势。")
        .user("问题：%s\n数据：%s".formatted(question, toTable(rows)))
        .call().content();
```

### 6.1.3 边界与坑

- 坑：结果为空时不应调用模型，直接返回"无数据"。
- 坑：结果行数过多导致提示超长，需截断或预聚合。
- 反例：让模型"估算"未查询到的数据，导致编造。

## 6.2 图表与报告

### 6.2.1 图表类型选择

| 数据形态 | 图表 |
|----------|------|
| 单维度 + 单指标 | 柱状图 / 折线图 |
| 时间序列 | 折线图 |
| 占比 | 饼图 |
| 多指标 | 分组柱状图 |

```java
// 图表配置
ChartConfig chart = chartGenerator.generate(question, rows);
// → {type: "bar", xAxis: [...], series: [...]}
```

报告结构为：结论 → 图表 → 明细表 → 所用 SQL 与口径 → 数据时间范围。报告需注明数据更新时间与查询耗时，展示所用 SQL 以增强可解释性，导出时脱敏。

# 7. 接口设计

## 7.1 请求与响应

### 7.1.1 接口定义

对外暴露统一查询接口，入参为自然语言问题，出参包含 SQL、数据、分析、图表四部分，任一部分缺失都要显式标记。

```java
POST /api/analysis/query
{
  "question": "上月各区域销售额对比"
}
→ {
  "sql": "SELECT region, SUM(amount) ...",
  "data": [...],
  "analysis": "华东区域销售额最高...",
  "chart": {...}
}
```

### 7.1.2 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| question | string | 自然语言问题 |
| sql | string | 实际执行的 SQL |
| data | array | 查询结果 |
| analysis | string | 分析结论 |
| chart | object | 图表配置，可为空 |
| traceId | string | 追踪 ID |

## 7.2 错误码与接口安全

### 7.2.1 错误码表

| 码 | 含义 | 处理 |
|----|------|------|
| 4001 | SQL 校验不通过 | 拒绝并说明 |
| 4002 | 越权访问 | 拒绝并审计 |
| 4003 | 查询超时 | 提示重试 |
| 4004 | 表/字段不存在 | 提示补充信息 |
| 5001 | 模型调用失败 | 降级或重试 |

### 7.2.2 鉴权与限流

接口需携带用户身份与角色用于权限注入，禁止匿名调用；按用户与租户限流，防止批量取数或攻击。请求体记录审计但需脱敏，SSE 流式返回可更快给出首字节。

# 8. 数据模型

## 8.1 核心实体

### 8.1.1 实体清单

| 实体 | 说明 |
|------|------|
| QueryRequest | 查询请求 |
| QueryResult | 查询结果（SQL + 数据） |
| UserContext | 用户上下文（角色、区域） |
| ChartConfig | 图表配置 |
| QueryAudit | 审计记录 |

## 8.2 审计与缓存

### 8.2.1 审计表结构

```sql
CREATE TABLE query_audit (
    id           BIGSERIAL PRIMARY KEY,
    user_id      VARCHAR(64)  NOT NULL,
    question     TEXT         NOT NULL,
    sql_masked   TEXT         NOT NULL,
    row_count    INT          NOT NULL,
    created_at   TIMESTAMPTZ  NOT NULL,
    trace_id     VARCHAR(64)
);
```

### 8.2.2 缓存与生产建议

缓存键为"问题规范化 + 用户权限范围"，值为结果集；缓存必须按权限维度隔离，否则会串数据。高频问题优先缓存，权限或口径变更时失效缓存，缓存结果同样受行数限制。审计不可丢，异步写 + 告警。

# 9. 关键技术点

## 9.1 问题与方案

### 9.1.1 对照表

| 问题 | 方案 |
|------|------|
| SQL 注入 | 校验 + 只读账号 |
| 全表扫描 | 强制 LIMIT + 超时 |
| 越权 | 表/列/行级权限 |
| 查询错误 | 表结构提示 + 重试 |
| 结果解读 | LLM 分析 |
| 图表 | 结构化图表配置 |

## 9.2 反例与边界

### 9.2.1 反例

- 反例：仅靠提示词禁止写库，模型仍可能生成 UPDATE。
- 反例：把行级过滤交给模型，模型漏加条件即越权。

### 9.2.2 边界与生产建议

| 边界 | 处理 |
|------|------|
| 空结果 | 明确说明，不编造 |
| 超时 | 降级返回 |
| 图表失败 | 回退表格 |
| 校验失败 | 直接拒绝 |

安全校验不可绕过，放在执行器入口；审计不可丢，异步写 + 告警；成本可控，高频问题缓存 + 固定工具。

# 10. 部署与演进

## 10.1 部署形态

### 10.1.1 拓扑与配置

```
客户端 ──> API 网关 ──> Agent 服务 ──> 只读数据源
                              │
                              ├──> 元数据服务
                              ├──> 审计库
                              └──> 模型服务
```

| 配置 | 建议 |
|------|------|
| 只读账号 | 独立账号、最小授权 |
| 连接池 | 限制大小与超时 |
| 超时 | 语句级 + 连接级 |
| 限流 | 按用户/租户 |

## 10.2 演进路线

### 10.2.1 阶段划分

```
阶段1：只读报表（固定查询工具）
阶段2：Text2SQL（仅管理员）
阶段3：扩大范围（带权限过滤）
```

### 10.2.2 演进原则

| 阶段 | 能力 | 风险控制 |
|------|------|----------|
| 1 | 固定工具 | 无需 Text2SQL |
| 2 | Text2SQL 试点 | 限管理员、全审计 |
| 3 | 全面开放 | 权限过滤 + 监控 |

从严到宽逐步放开，每次放开都要有对应的监控与回滚手段。

# 11. 小结

## 11.1 核心结论

- 架构：意图 → 查询 → 执行 → 分析 → 图表 → 报告。
- 优先固定查询工具，Text2SQL 严格防护。
- 七层安全防护 + 表/列/行级权限。
- 表结构提示提升 SQL 准确率。
- 结果分析与图表提升可用性。
- 权限过滤由代码强制注入，绝不依赖模型。

## 11.2 决策口诀

> 能用固定工具就不放 Text2SQL；能只读就绝不写；能用权限过滤就不靠提示词；安全与准确优先于灵活与炫技。
