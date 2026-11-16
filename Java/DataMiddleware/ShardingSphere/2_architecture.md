ShardingSphere 的架构可以拆成三层：**接入层**（JDBC/Proxy/Sidecar 形态的 Facade）、**内核层**（解析→路由→改写→执行→归并的 SQL 引擎流水线）、**基础设施层**（Mode 运行模式、事务、治理、可观测等横向能力）。分片、读写分离、加密、影子这些功能都不直接碰 JDBC 或协议，而是注册成内核流水线上的规则与插件——这种"内核 + 规则插件"分层，是它能用一套代码同时支撑 JDBC 和 Proxy 的根本原因。

# 1. 整体架构

```text
┌──────────────────────────────────────────────────────────┐
│                   ShardingSphere                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ Sharding │  │  Read-   │  │  Data    │  ← 功能层     │
│  │  Sphere  │  │  Write   │  │ Encrypt  │               │
│  │  JDBC    │  │ Splitting│  │          │               │
│  │  Proxy   │  └────┬─────┘  └────┬─────┘               │
│  └────┬─────┘       │             │                      │
│  ┌────┴─────────────┴─────────────┴─────┐               │
│  │              Kernel                   │  ← 内核层     │
│  │  ┌──────┐ ┌───────┐ ┌──────┐ ┌─────┐ │               │
│  │  │Parse │→│Rewrite│→│Route │→│Execute│→│Merge │      │
│  │  └──────┘ └───────┘ └──────┘ └─────┘ │               │
│  └──────────────────────────────────────┘               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │   Mode   │  │  Trans-  │  │  Govern- │  ← 基础设施层 │
│  │  (模式)  │  │  action  │  │  ance    │               │
│  └──────────┘  └──────────┘  └──────────┘               │
└──────────────────────────────────────────────────────────┘
```

这张图里三个横切面各有职责。**功能层**是一组"规则（Rule）"：每个功能把配置解析成 `ShardingSphereRule` 注入内核，内核在流水线各阶段回调规则提供的 SPI（分片算法、负载均衡算法、加密算法等），加新功能不必改内核代码。**Kernel** 是与功能无关的 SQL 处理流水线，所有逻辑 SQL 到这里都走同一条路。**基础设施层**提供运行模式、事务、权限、可观测等支撑能力。源码组织对应 `infra` → `kernel` → `mode` → 接入形态；Facade 层只做协议适配与上下文组装，从 `Connection.prepareStatement()` 或 Proxy 的 `CommandExecuteEngine` 往下钻几步就会汇入同一条流水线。

# 2. 五大 SQL 处理引擎

```text
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

五个引擎的输入输出串成一条链：**Parse 输入 SQL 字符串输出 AST；Route 基于 AST + 规则算出"在哪些数据源的哪些物理表执行"；Rewrite 拿着路由结果把逻辑 SQL 翻译成物理 SQL；Execute 并发执行得到多个结果集游标；Merge 把多游标归并成一个对上层透明的结果集**。两个常被误解的细节：其一，逻辑上**路由先于改写**（改写依赖路由产物），流水线有时写作 Parse→Route→Rewrite→Execute→Merge，两种表述指同一批阶段；其二，**归并不仅是结果拼接**，GROUP BY/ORDER BY/聚合的语义正确性全靠 Merge 的装饰者组合兜底，这也是分片后 SQL 语义与单库不完全一致（如 AVG 需改写补偿）的根源。

# 3. SQL 解析引擎（Parse）

解析引擎的目标是把 SQL 文本变成**带元数据语义的抽象语法树**。ShardingSphere 用 ANTLR4 为每种方言维护 `.g4` 文法，生成的 Parser/Visitor 把 SQL 转成 `SQLStatement` 对象树。

```java
String sql = "SELECT o.id, o.user_id FROM t_order o WHERE o.id = ?";

// 1. 语法解析 → AST
SQLStatement sqlStatement = parser.parse(sql);

// 2. AST 转换为 ShardingSphere SQLStatement
// SelectStatement
//   ├── from: SimpleTableSegment(t_order)
//   ├── projection: [ColumnProjection(id), ColumnProjection(user_id)]
//   └── where: BinaryExpression(Column(id), =, ParameterMarker(?))
```

解析分两段：**语法解析**（SQL → 通用 SQLStatement）与**方言重构**（补上方言特有语义，如 MySQL 的 `LIMIT` 分页段、Oracle 的 `ROWNUM`）。产出经 `SQLAudit` 审计后，结合从真实库采样来的表结构元数据填充成"带语义的 AST"，后续路由/改写只依赖这棵树。**5.x 相对 4.x 的重要升级就是这套解析引擎**：4.x 用 druid parser，5.x 换成分方言 ANTLR 文法后，才支撑起 DistSQL、SQL 翻译、联邦查询这些"必须真正懂 SQL 结构"的功能。

## 3.1 方言适配

```text
MySQL      → MySQLSelectParser / MySQLStatementVisitor
PostgreSQL → PostgreSQLSelectParser
openGauss  → openGaussSelectParser
Oracle     → OracleSelectParser
SQLServer  → SQLServerSelectParser
```

方言差异体现在三处：**文法**（每种数据库一套 g4）、**分页语句**（LIMIT / OFFSET-FETCH / ROWNUM 三种抽象）、**函数与类型系统**。

> **注意**：解析能力 ≠ 执行能力。文法支持的 SQL 远多于分片场景可用的 SQL——`PARSE SELECT ...`（DistSQL RUL）可验证解析结果，但路由是否合理、归并能否保持语义要单独评估，不要用"能解析"推断"能正确分片执行"。

# 4. 改写引擎（Rewrite）

改写引擎沿 AST 遍历，把**需要替换的 token 收集成分片单元**再批量替换。加密列、影子表、主键补充、方言翻译都注册在同一个 `SQLRewriteEntry` 上，按规则叠加。
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
| 广播表改写 | 补齐数据源维度路由 |
| 加解密改写 | 明文列 → 密文列 + 参数加密（见 6_data_encryption） |

分页改写是高频考点：`LIMIT 10000, 10` 跨分片时物理 SQL 会被改写为 `LIMIT 0, 10010`（各分片取 offset+count 条），归并时丢弃前 10000 条——分片越多、offset 越大放大越严重，深分页要么改游标（`WHERE id > last_max LIMIT 10`），要么业务限制最大页码。AVG 改写同理：SELECT 补出 `SUM(x), COUNT(x)`，归并后再算平均。

# 5. 路由引擎（Route）

路由引擎吃 AST + 分片条件，产出 `RouteContext`（数据源 × 表 的执行单元集合）。分片条件的来源优先级是：**Hint 强制路由 > SQL 中的分片键表达式 > 无（全路由）**。

```java
public RouteContext route(SQLStatement sqlStatement, ShardingSphereRule rule) {
    // 提取分片条件
    ShardingConditions conditions = extractShardingConditions(sqlStatement);
    RouteContext routeContext = new RouteContext();
    // 情况 1：精确分片（WHERE id = 1）→ 路由到 ds0.t_order_0
    if (conditions.isSingleShard()) {
        routeContext.addRouteUnit("ds0", "t_order_0");
    }
    // 情况 2：无分片键 → 全库全表路由（ds0/ds1 × t_order_0/1）
    else if (conditions.isEmpty()) {
        routeContext.addAllRouteUnits();
    }
    return routeContext;
}
```

分片条件被抽象成 `ShardingCondition`（列 + 值列表 + 表达式类型），表达式类型决定路由宽度：`EQUAL`/`IN` 走精确路由，`BETWEEN` 走范围路由（INLINE 默认禁止范围查询，需开 `allow-range-query-with-inline-sharding`），`IS NULL`、函数包裹列、`OR` 连接非分片键时降级为**全路由**。范围路由和全路由的性能差距是数量级的——"分片键是否出现在条件里、以什么形式出现"直接决定代价分级（详见 4_sharding）。

# 6. 执行引擎（Execute）

执行引擎拿着路由/改写产物（`ExecutionUnit` = 数据源 + SQL + 参数）并发执行，对 JDBC 形态是"同进程多线程 + 各库连接池"，对 Proxy 形态是"后端连接组 + 异步收发"。

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

**连接模式**：`MEMORY_STRICTLY`（内存严格，一次执行所有分片）与 `CONNECTION_STRICTLY`（连接严格，分批执行）。两种模式是互逆的取舍。`CONNECTION_STRICTLY`（默认）为每个执行单元取独立连接**并发执行**，吞吐高但一次占 N 个连接；`MEMORY_STRICTLY` 用 `max-connections-size-per-query` 限制连接数、按组串行，适合全路由大结果集导出（配合 fetchSize 流式读取），代价是并发下降，且与"多分片同时持游标"的 ORDER BY 流式归并冲突——`PreparedStatement` 设 fetchSize 的校验报错多源于此。慢分片会拖垮整个查询的 P99（木桶效应），热点查询应尽量命中单分片。

# 7. 归并引擎（Merge）

归并把 N 个 `QueryResult`（各分片 ResultSet 游标的包装）合成 1 个。实现是教科书级的**装饰者模式**：每种归并规则是一个 `MergedResult` 装饰器，按需叠加。

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

流式归并（各游标堆顶维护小顶堆、逐条弹出）适合**内存放不下**的大结果集排序；内存归并先聚合再吐出，适合 GROUP BY 基数小的场景。需要警惕**归并语义不完备的 SQL**：`DISTINCT` 与聚合混用、`HAVING` 引用未投影列、跨分片 `UNION` 等，5.x 部分场景直接报错，部分"结果对但代价高"，上线前用 `PREVIEW SQL` 看物理 SQL 是最低成本的校验。

> **设计考量**：**归并引擎是分片中间件的语义天花板**——所有"数据库原生能做但跨分片做不了"的能力（子查询下推、窗口函数、跨库外键）要么报错要么靠联邦查询引擎兜底。评估 ShardingSphere 能否满足业务，本质是评估 SQL 能否被"改写 + 归并"等价还原。

# 8. JDBC vs Proxy 核心架构差异

| 维度 | JDBC | Proxy |
|------|------|------|
| SQL 入口 | `ShardingSphereDataSource` 拦截 | MySQL 协议解析 |
| 执行引擎 | 内存模式（多线程 JDBC） | 连接模式（数据库协议） |
| 归并引擎 | 内存/流式归并 | 内存/流式归并 |
| 治理中心 | 可选 | 内置 |
| 配置方式 | YAML / Java API / DistSQL | DistSQL / YAML |

同内核、不同外壳，差异集中在四处：**入口协议**（JDBC 接口调用 vs 自研 Frontend 解析 MySQL 报文）、**后端连接管理**（JDBC 复用应用内连接池；Proxy 自维护数据源连接池，前端/后端连接是两组资源）、**安全边界**（Proxy 有 authority 用户体系，JDBC 的信任边界等于应用本身）、**配置分发**（JDBC 配置在 classpath，Proxy 可落元数据仓库共享）。Proxy 故障域更集中——挂了影响所有接入方，反过来升级不动业务应用。

# 9. 插件化 SPI 体系

内核与功能之间用 SPI 解耦，`ShardingSpherePluginServiceLoader`（底层 Java SPI + 自定义 `TypedSPI`）按 type 加载实例。常用扩展点：

| SPI 接口 | 功能 | 内置实现示例 |
|----------|------|-------------|
| `ShardingAlgorithm` | 分片路由计算 | INLINE、INTERVAL、HASH_MOD、AUTO_INTERVAL |
| `KeyGenerator` | 分布式主键 | SNOWFLAKE、UUID、COSID |
| `LoadBalanceAlgorithm` | 读负载均衡 | ROUND_ROBIN、RANDOM、WEIGHT |
| `Encryptor` | 加解密 | AES、RC4、SM3、SM4、MD5 |
| `ShadowAlgorithm` | 影子流量判定 | VALUE_MATCH、REGEX_MATCH、SQL_HINT |
| `ModeRepository` | 元数据仓库 | ZooKeeper、JDBC |
| `TransactionManager` | 事务管理 | LOCAL、XA（Atomikos/Narayana）、BASE（Seata） |

扩展流程固定三步：**实现接口 → `getType()` 返回类型名 → 在 `META-INF/services/` 注册**，配置里 `type: 自定义名` 引用即可；CLASS_BASED 系列不需要写 SPI 注册，直接指定业务类全限定名，是最低门槛的扩展入口。

> **设计争议**：SPI 全插件化的代价是**类型安全后移**——配置写错 `type`、props 名拼错，都要到启动或首次路由时才暴露；生产变更建议先在测试环境 `PREVIEW SQL` 验证算法行为，而不是只看启动日志是否报错。

# 10. 小结

| 层 | 关键组件 | 一句话 |
|----|---------|--------|
| 接入 | JDBC / Proxy / Sidecar（规划） | 同一内核多形态复用 |
| 流水线 | Parse → Rewrite → Route → Execute → Merge | 一切功能都是"AST 上的操作" |
| 路由 | ShardingCondition + RouteContext | 分片键出现形式决定代价分级 |
| 执行 | CONNECTION_STRICTLY / MEMORY_STRICTLY | 并发与流式不可兼得 |
| 归并 | MergedResult 装饰者链 | 跨分片语义正确性的兜底者 |
| 扩展 | TypedSPI 插件体系 | 算法、方言、仓库、事务皆可替换 |

理解了五阶段流水线，后面各功能篇都能统一按"配置生成什么规则、规则在哪一阶段介入、边界在哪"三问来读。下一篇（3_quick_start）把 Spring Boot 3 + 5.5.x 的最小可运行配置走通。
