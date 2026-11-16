# 1. 路由入口

路由引擎回答的问题是：**这条 SQL 应该发到哪些"数据源 × 表"组合上执行**。它的输入是三样东西——SQLStatement（解析产物，可能为 null）、ShardingRule（逻辑表与实际节点的映射）、以及 SQL 参数，输出是 `RouteContext`（一组 RouteUnit）。入口骨架：

```java
public final class ShardingRouter {

    public RouteContext route(SQLStatement sqlStatement,
                               ShardingRule rule,
                               ConfigurationProperties props) {

        // 1. 获取路由策略
        ShardingStrategy strategy = rule.getDatabaseShardingStrategy();

        // 2. 提取分片条件
        ShardingConditions conditions = getShardingConditions(
            sqlStatement, strategy);

        // 3. 执行路由
        RouteContext routeContext = executeRoute(
            sqlStatement, conditions, rule);

        return routeContext;
    }
}
```

5.x 的真实分层比骨架更细：`RouteEngine` 按语句类型分发（分片路由 / 读写分离路由 / 加密影子路由 / 单表路由 / 联邦路由），分片路由内部再按**库策略与表策略各自独立执行、结果做笛卡尔积合并**。RouteContext 里同时携带 `sqlStatement` 引用，改写引擎直接复用，**路由决策与改写动作是解耦的两段**——这一点在排查"路由对了但 SQL 没改写"类问题时很关键。

## 1.1 路由的输入决定论

路由的全部智慧来自分片条件提取，提取不到就只有两条路：全路由（能容忍读、危险于写）或 hint 强制。因此**路由优化 = 分片条件优化**，源码里 ShardingCondition 的表达面决定了什么谓词能裁剪：等值（`=`、`IN`）、范围（`BETWEEN`、`>`、`<`，仅部分算法支持）、以及复合 AND 组合都参与裁剪；**OR 两侧分片键不同、函数包裹分片键（`WHERE MOD(order_id,2)=0`）、分片键参与 LIKE/正则**这三类谓词在提取阶段就返回空条件，等价于放弃裁剪。想验证某条 SQL 的路由行为，不要猜，用 `PREVIEW SQL`（Proxy）或开 `sql-show` 看 Actual SQL 数量。

# 2. 分片条件提取

```java
public ShardingConditions getShardingConditions(
        SQLStatement sqlStatement, ShardingStrategy strategy) {

    // 1. 从 WHERE 条件中提取分片键值
    // SELECT * FROM t_order WHERE order_id = 1
    // → shardingValue = 1

    // 2. WHERE order_id IN (1, 3, 5)
    // → shardingValues = [1, 3, 5]

    // 3. WHERE order_id BETWEEN 1 AND 100
    // → shardingRange = [1, 100]

    return new ShardingConditions(shardingValues);
}
```

实现上有两个细节。第一，**条件按"绑定到哪个表别名"组织**：JOIN 场景下 `o.order_id` 与 `i.order_id` 分别记账，绑定表路由依赖两侧都能提取；ON 条件里的分片键同样参与提取（不只是 WHERE）。第二，**批量插入的每行 VALUES 独立生成一组条件**，一条 INSERT...SELECT 或 VALUES 多行的路由集合是各行的并集，这就是批量写天然可能多播的原因。提取出的 `ShardingConditions` 还有"AND 语义/ OR 语义"之分：`ShardingConditionEngine` 把同一分片键的多条件合并（AND 取交集语义由算法实现决定，如精确算法对 IN 取多值路由）。

> **注意**：参数类型陷阱。路由以**值的类型**匹配算法的输入域——INLINE/GROUP_MODIFY 期望数字，收到字符串 `"1001"` 时部分算法会做转换、部分直接返回全路由；配置分片键列为 CHAR/VARCHAR 业务码时用 STRING_HASH 类算法，而不是让数字算法碰字符串。

# 3. 路由类型

```java
public enum RouteType {
    SINGLE,          // 单播：精确到一个数据源的一个表
    BROADCAST,       // 广播：对所有数据源的所有表
    ALL_DATASOURCES,  // 全库：对所有数据源
    ALL_TABLES,       // 全表：对某个数据源的所有表
    COMPLEX           // 复杂：多个数据源多个表
}
```

## 3.1 单播路由

```java
// WHERE order_id = 1 → 精确到一个物理表
public RouteContext singleRoute(ShardingConditions conditions) {
    RouteContext result = new RouteContext();
    for (ShardingValue value : conditions.getValues()) {
        String actualTable = shardingAlgorithm.doSharding(
            availableTables, value);
        result.addRouteUnit("ds0", actualTable);
    }
    return result;
}
```

## 3.2 广播路由

```java
// t_config 是广播表 → 每个数据源都要执行
public RouteContext broadcastRoute() {
    RouteContext result = new RouteContext();
    for (DataSource ds : allDataSources) {
        result.addRouteUnit(ds.getName(), "t_config");
    }
    return result;
}
```

## 3.3 全库表路由

```java
// 无分片键 → 所有分片都要查
public RouteContext allDataSourceRoute() {
    RouteContext result = new RouteContext();
    for (DataSource ds : allDataSources) {
        for (String table : ds.getActualTables()) {
            result.addRouteUnit(ds.getName(), table);
        }
    }
    // 4 个分片 × 2 个表 = 8 个执行单元
    return result;
}
```

**精确/范围/裁剪三档能力**对应算法实现接口：`PreciseShardingAlgorithm` 只处理等值与 IN（值枚举代入公式）；`RangeShardingAlgorithm` 处理 BETWEEN/比较（如区间算法 `INTERVAL`、按日期分表的 `DATE_RANGE`）；未实现对应接口的算法遇到范围条件即放弃裁剪——INLINE 表达式本质是 Groovy 脚本代入，对 range 默认不裁剪，除非显式开 `allow-range-query-with-inline-sharding` 并接受"全节点枚举代入"的代价。**这是"明明写了 BETWEEN user_id IN 推导却扫全库"的根因**：INLINE 算法拿到范围值无法反解满足 `ds${user_id % 2}` 的节点集合，只能保守广播。

# 4. 绑定表路由优化

```java
// t_order 和 t_order_item 是绑定表
// SELECT o.*, i.* FROM t_order o JOIN t_order_item i
//   ON o.order_id = i.order_id
// WHERE o.order_id = 1

public RouteContext bindingTableRoute() {
    // 不使用笛卡尔积路由（4 × 4 = 16）
    // 而是生成 4 条路由：
    // ds0.t_order_0 JOIN ds0.t_order_item_0
    // ds0.t_order_1 JOIN ds0.t_order_item_1
    // ds1.t_order_0 JOIN ds1.t_order_1
    // ds1.t_order_1 JOIN ds1.t_order_item_1
}
```

## 4.1 绑定表生效的三个前提

绑定表（5.5 用 `!SHARDING_TABLE_REFERENCE_RULE`/DistSQL `CREATE SHARDING TABLE REFERENCE RULE` 表达）不是"两表分片键同名"就自动生效：① 分库分表算法与分片数量必须完全一致；② **JOIN 的 ON 条件必须包含两侧分片键的等值关系**（`ON o.order_id = i.order_id`），且该条件可提取；③ 两侧的分片条件都提取成功。任一不满足即回退笛卡尔积。**最隐蔽的失效原因是第②条的写法漂移**：ORM 生成的 SQL 把关联条件写进 WHERE 而非 ON、或用了 `o.id = i.oid` 不同列名，绑定关系就名存实亡——排查时用 PREVIEW 对比路由单元数即可确认（绑定生效 N 单元，失效 N×M 单元）。

## 4.2 同片关联 vs 跨片关联的语义边界

绑定表保证"同一 order_id 的订单与明细必然同库同后缀"，JOIN 下推到单物理库执行，语义与单库一致。但**只绑库不绑表的中间态**（分库键相同、分表键不同）会退化为库内笛卡尔积表 JOIN，路由单元从 M 涨到 M×T，性能悬崖介于两者之间。设计上尽量让主从表的分片键与分表算法一起对齐。

# 5. 笛卡尔积路由

```java
// 无绑定的 JOIN → 笛卡尔积
// SELECT * FROM t_order o, t_user u WHERE o.user_id = u.id

public RouteContext cartesianRoute() {
    RouteContext result = new RouteContext();
    // t_order: 4 个路由单元
    // t_user: 2 个路由单元
    // 笛卡尔积 = 4 × 2 = 8 个路由单元
    // 性能差！强烈建议使用绑定表
}
```

笛卡尔积路由的代价是**乘法级**的：三表无绑定 JOIN、每表 4×2=8 节点，路由单元理论上限 8³=512，每条都是一个完整物理 SQL，解析与改写阶段各单元还要生成独立 SQL——**大扇出本身就会成为 CPU 与连接池压力**（执行层按数据源分组串行/并行见 17 篇）。控制手段按优先级：绑定表 > 广播表（小维表直接每库一份消灭分布问题）> 联邦查询（可下推的跨片 JOIN，代价换能力）> 业务层冗余分片键（把关联键冗余进表使其可同片裁剪）。

> **设计争议**：无分片键的 DML（UPDATE 无 WHERE、DELETE 条件不含分片键）要不要直接拒绝？ShardingSphere 当前允许广播执行，仅靠 `sql-audit` 与规约插件兜底；不少团队倾向自定义 `SQLAuditor` 在解析后、执行前拦截全路由 DML——比事后靠 `max_connections` 和慢查询救火主动得多（审计器是 SPI，见 20 篇）。

# 6. Hint 路由

当 SQL 本身无法表达路由意图（分片值在应用上下文而不在 SQL 里、需要强制走某片、读写分离强制主库），Hint 是旁路通道，两种载体：

```java
// 方式一：Java API（ThreadLocal，用完必须 close）
try (HintManager hintManager = HintManager.getInstance()) {
    hintManager.addDatabaseShardingValue("t_order", 11L); // 指定库分片值
    hintManager.addTableShardingValue("t_order", 0L);     // 指定表分片值
    // 执行 SQL：路由算法对 hint 值代入公式，不再解析 WHERE
    jdbcTemplate.query("SELECT * FROM t_order WHERE status = 1", ...);
    // 读写分离场景：
    // hintManager.setWriteRouteOnly();
    // hintManager.setReadRouteOnly();
}

// 方式二：SQL 注释 hint（对 ORM 透明，可经 Proxy 传递）
// SELECT /* SHARDINGSPHERE_HINT: t_order.SHARDING_DATABASE_VALUE=0, t_order.SHARDING_TABLE_VALUE=1 */ * FROM t_order
// 其他常用键：WRITE_ROUTE_ONLY=true（强制主库）、SHADOW=true（影子库）、SKIP_SQL_REWRITE=true
```

**Hint 语义优先级高于 WHERE 提取**：hint 提供分片值时条件提取被跳过（而非取交集），所以业务代码里"WHERE 写片 A、hint 指片 B"的 bug 会以 hint 为准，静默查错片。`HintManager` 是 ThreadLocal，线程池复用场景忘 `close()` 会把 hint 泄漏给同线程下一个请求——官方反复强调 try-with-resources，代码评审把这一点当硬规。**注释 hint 的适用边界**：JDBC 模式两者皆可；Proxy 模式只有 SQL 注释 hint 能穿透（线程本地 API 不在 Proxy 进程内）。

# 7. 全表扫描触发条件与代价

触发全路由（full route）的完整清单：

```text
① SQLStatement 解析失败/为 null（见 15 篇降级路径）
② WHERE/ON 完全不含分片键（最常见）
③ 分片键被函数/表达式包裹：MOD(order_id,2)=?、UPPER(shard_col)=?
④ 算法不支持当前谓词：INLINE + BETWEEN（未开 allow-range）
⑤ OR 连接了不同表的非对齐条件，提取失败
⑥ 广播表 DML / 未配置规则的表落入单表/默认策略
⑦ DDL 与部分 TCL：默认按所有数据源执行
```

| 场景 | 代价形态 | 优化方向 |
|------|---------|---------|
| SELECT 全路由 | 扇出 N 单元并行扫 + 归并 | 冗余分片键到查询条件；走 ES |
| UPDATE/DELETE 全路由 | 同事务锁 N 库，放大死锁窗口 | 强制带分片键；审计拦截 |
| 无分片键 ORDER BY+LIMIT | 每片取 offset+limit 再归并，读放大 k× | 游标分页；限制 offset 上限 |
| 全路由 + 大结果集 | 内存归并 OOM 风险 | 流式结果集 fetchSize；限流 |

**代价的不对称性要记牢**：读全路由是"慢"，写全路由是"危险"——后者在一个分布式事务里同时打开 N 个库的事务，锁持有时间取最慢分片，XA 场景还把悬挂事务概率乘上去。容量规划阶段就应该按"业务 SQL 中全路由占比"做一次 PREVIEW 普查（19 篇上线 checklist 的固定项）。

# 8. 路由结果结构

```java
public final class RouteContext {

    // 路由单元列表
    private final Collection<RouteUnit> routeUnits = new LinkedHashSet<>();

    // 每个路由单元包含：数据源名 + SQL改写结果
    public static class RouteUnit {
        private final RouteMapper dataSourceMapper;   // 数据源
        private final Collection<RouteMapper> tableMappers; // 表映射
    }
}
```

```java
// 路由结果示例
RouteContext {
    routeUnits: [
        RouteUnit("ds0", ["t_order_0"]),
        RouteUnit("ds0", ["t_order_1"]),
        RouteUnit("ds1", ["t_order_0"]),
        RouteUnit("ds1", ["t_order_1"])
    ]
}
```

RouteUnit 的 `dataSourceMapper` 记录"逻辑库名→实际库名"映射（读写分离下还含"逻辑组→主/从"两层映射），`tableMappers` 记录逻辑表→物理表映射，改写引擎随后用同一份映射生成物理 SQL——**路由产出即改写的输入字典**，所以"Actual SQL 里表名对但库名不对"这类诡异问题几乎总是元数据映射不一致（逻辑库名与连接 schema 混写），而非算法问题。RouteContext 还有一个常被忽略的字段：`isNeedRewrite`，广播表/单表等无映射场景置 false 走直通，减少一次无谓改写。

# 9. 小结

1. **路由质量=分片条件提取质量**：函数包裹、类型不匹配、OR 跨键都会导致放弃裁剪
2. **INLINE 不裁范围、复合键缺一不裁、广播 DML 要审计拦截**是三条高频红线
3. **绑定表生效三前提**：算法一致、ON 含双侧分片键、条件可提取，PREVIEW 单元数即体检表
4. **Hint 优先级高于 WHERE 且线程本地必须 close**，Proxy 场景只有注释 hint 能透传
5. **读全路由是慢，写全路由是危险**——上线前用路由占比指标量化