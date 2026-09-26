
# 1. 整体架构

```
┌──────────────────────────────────────────────────────────┐
│                   ShardingSphere                          │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Sharding │  │  Read-   │  │  Data    │               │
│  │  Sphere  │  │  Write   │  │ Encrypt  │  ← 功能层     │
│  │  JDBC    │  │ Splitting│  │          │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
│       │             │             │                      │
│  ┌────┴─────────────┴─────────────┴─────┐               │
│  │              Kernel                    │               │
│  │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │               │
│  │  │Parse │→│Rewrite│→│Route │→│Execute│→│Merge    │   │
│  │  └──────┘ └──────┘ └──────┘ └──────┘ │               │
│  └──────────────────────────────────────┘               │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │   Mode   │  │  Trans-  │  │  Govern- │               │
│  │  (模式)  │  │  action  │  │  ance    │               │
│  └──────────┘  └──────────┘  └──────────┘               │
└──────────────────────────────────────────────────────────┘
```

# 2. 五大 SQL 处理引擎

```
应用程序 SQL
     │
     ▼
┌──────────────┐
│ 1. Parse     │  ANTLR 将 SQL 解析为 AST（抽象语法树）
│   (解析)     │  输出：SQLStatement 对象
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ 2. Rewrite   │  逻辑 SQL → 物理 SQL
│   (改写)     │  - 逻辑表名 → 物理表名（order → order_0）
└──────┬───────┘  - 自增主键改写
       │          - 分页改写（LIMIT OFFSET）
       ▼
┌──────────────┐
│ 3. Route     │  确定目标数据源和表
│   (路由)     │  - 单播路由：精确分片键 → 一个数据源
└──────┬───────┘  - 广播路由：全分片
       │          - 全库表路由：无分片键 → 所有数据源
       ▼
┌──────────────┐
│ 4. Execute   │  执行 SQL 到目标数据库
│   (执行)     │  - 内存模式（JDBC）：多线程并发调用
└──────┬───────┘  - 连接模式（Proxy）：异步发送请求
       │
       ▼
┌──────────────┐
│ 5. Merge     │  合并多个结果集
│   (归并)     │  - 遍历归并 / 排序归并 / 分组归并 / 分页归并
└──────────────┘  - 装饰者模式组合多种归并策略
```

# 3. SQL 解析引擎（Parse）

```java
// ANTLR4 生成解析器
// 支持的 SQL 方言：MySQL, PostgreSQL, openGauss, SQLServer, Oracle

// 解析示例
String sql = "SELECT o.id, o.user_id FROM t_order o WHERE o.id = ?";

// 1. 语法解析 → AST
SQLStatement sqlStatement = parser.parse(sql);

// 2. AST 转换为 ShardingSphere SQLStatement
// SelectStatement
//   ├── from: SimpleTableSegment(t_order)
//   ├── projection: [ColumnProjection(id), ColumnProjection(user_id)]
//   └── where: BinaryExpression(Column(id), =, ParameterMarker(?))
```

## 3.1 方言适配

```
MySQL      → MySQLSelectParser / MySQLStatementVisitor
PostgreSQL → PostgreSQLSelectParser
openGauss  → openGaussSelectParser
Oracle     → OracleSelectParser
SQLServer  → SQLServerSelectParser
```

# 4. 改写引擎（Rewrite）

```java
// 改写示例：t_order → t_order_0, t_order_1
// 原始 SQL
SELECT o.id FROM t_order o WHERE o.id = 1;

// 改写后
SELECT o.id FROM t_order_0 o WHERE o.id = 1;

// 自增主键改写
// 原始 SQL: INSERT INTO t_order (user_id) VALUES (?);
// 改写后: INSERT INTO t_order_0 (id, user_id) VALUES (?, ?);
//         id = 分布式主键生成器.nextId()
```

**改写类型**：
| 改写 | 说明 |
|------|------|
| 表名改写 | 逻辑表 → 物理表 |
| 自增主键 | 补充主键值和列名 |
| 分页改写 | OFFSET LIMIT 优化 |
| 聚合改写 | AVG → SUM/COUNT |

# 5. 路由引擎（Route）

```java
// 路由流程
public RouteContext route(SQLStatement sqlStatement, ShardingSphereRule rule) {
    // 1. 提取分片条件
    ShardingConditions conditions = extractShardingConditions(sqlStatement);

    // 2. 生成路由结果
    RouteContext routeContext = new RouteContext();

    // 情况 1：精确分片（WHERE id = 1）
    if (conditions.isSingleShard()) {
        // → 路由到 ds0.t_order_0
        routeContext.addRouteUnit("ds0", "t_order_0");
    }
    // 情况 2：广播路由（无分片键）
    else if (conditions.isEmpty()) {
        // → 路由到所有数据源
        routeContext.addRouteUnit("ds0", "t_order_0");
        routeContext.addRouteUnit("ds0", "t_order_1");
        routeContext.addRouteUnit("ds1", "t_order_0");
        routeContext.addRouteUnit("ds1", "t_order_1");
    }
    return routeContext;
}
```

# 6. 执行引擎（Execute）

```java
// JDBC 模式：多线程并发执行
public List<QueryResult> execute(Collection<ExecutionUnit> units) {
    ExecutorEngine engine = new ExecutorEngine(executorSize);
    return engine.execute(units, unit -> {
        Connection conn = dataSource.getConnection();
        PreparedStatement ps = conn.prepareStatement(unit.getSql());
        // 执行 SQL
        return ps.executeQuery();
    });
}
```

**连接模式**：
- `MEMORY_STRICTLY`：内存限制，一次执行所有分片
- `CONNECTION_STRICTLY`：连接限制，分批执行

# 7. 归并引擎（Merge）

```java
// 归并策略：装饰者模式组合
MergedResult result;

// 遍历归并（基础）
result = new IteratorStreamMergedResult(queryResults);

// 排序归并（ORDER BY）
result = new OrderByStreamMergedResult(result, orderByItems);

// 分组归并（GROUP BY）
result = new GroupByStreamMergedResult(result, groupByItems);

// 分页归并（LIMIT）
result = new LimitDecoratorMergedResult(result, paginationContext);
```

# 8. JDBC vs Proxy 核心架构差异

| 维度 | JDBC | Proxy |
|------|------|------|
| SQL 入口 | `ShardingSphereDataSource` 拦截 | MySQL 协议解析 |
| 执行引擎 | 内存模式（多线程 JDBC） | 连接模式（数据库协议） |
| 归并引擎 | 内存/流式归并 | 内存/流式归并 |
| 治理中心 | 可选 | 内置 |
| 配置方式 | YAML / Java API / DistSQL | DistSQL / YAML |
