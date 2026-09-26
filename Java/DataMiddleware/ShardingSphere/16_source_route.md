
# 1. 路由入口

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
    // ds1.t_order_0 JOIN ds1.t_order_item_0
    // ds1.t_order_1 JOIN ds1.t_order_item_1
}
```

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
    return result;
}
```

# 6. 路由结果结构

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
