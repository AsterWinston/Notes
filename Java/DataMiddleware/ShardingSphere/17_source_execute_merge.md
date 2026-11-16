# 1. 执行引擎

执行引擎的职责是**把 RouteContext 里的执行单元调度到物理数据源上并发跑，并把结果封装为可归并的 QueryResult 集合**。它的骨架：

```java
public final class ShardingSphereExecutor {

    public List<QueryResult> execute(RouteContext routeContext) {
        // 1. 生成执行组
        Collection<ExecutionGroup> executionGroups =
            generateExecutionGroups(routeContext);

        // 2. 选择执行引擎（内存模式 / 连接模式）
        ExecutorEngine engine = selectEngine();

        // 3. 并发执行
        return engine.execute(executionGroups);
    }
}
```

5.x 的类名与 4.x 相比已有出入（`DriverExecutorEngine`/`ProxyExecutorEngine` 等按前端形态分化），但模型稳定：**执行分组策略先定，再进线程池**。DML 与 SELECT 走不同分组倾向：DML 强调同数据源串行（复用一条连接、保证库内顺序），SELECT 强调最大并行（多连接扇出取回最快分片）。

# 2. 执行组生成

```java
// 根据连接模式生成执行组
private Collection<ExecutionGroup> generateExecutionGroups(
        RouteContext routeContext) {

    if (connectionMode == ConnectionMode.MEMORY_STRICTLY) {
        // 内存限制模式：所有执行单元并发执行
        return routeContext.getRouteUnits().stream()
            .map(unit -> new ExecutionGroup(
                unit.getDataSourceName(),
                Collections.singletonList(unit)))
            .collect(toList());
    }

    if (connectionMode == ConnectionMode.CONNECTION_STRICTLY) {
        // 连接限制模式：同一数据源的执行单元合并
        return groupByDataSource(routeContext.getRouteUnits());
    }
}
```

## 2.1 两种连接模式的取舍

| 模式 | 物理连接占用 | 内存占用 | 适用 |
|------|-------------|---------|------|
| MEMORY_STRICTLY | 高（单元数 = 连接数） | 低（流式游标各自小缓冲） | 查询、分片少、连接池富裕 |
| CONNECTION_STRICTLY | 低（一库一连接串行跑完再回） | 高（多结果集驻留） | 库多表多、连接紧张 |

5.x 的默认策略不是让用户选，而是**按语句类型自动决定**：DML 强制 CONNECTION_STRICTLY（同一物理连接上顺序执行，避免自己和自己死锁、也避免 XA 下连接爆炸），SELECT 默认 MEMORY_STRICTLY；`max-connections-size-per-query`（默认 1）限制单查询每库最多连接数，超过即把同库单元折叠成串行——**把它调大提升读并发，调小保护连接池**，是执行层最重要的一个旋钮。CONNECTION_STRICTLY 下有个经典副作用：同库跨表批量插入时参数绑定复用同一 PreparedStatement，ORM 层若持有语句状态要留意。

# 3. ExecutorEngine

```java
public final class ExecutorEngine {

    private final int executorSize;  // 线程池大小

    public List<QueryResult> execute(
            Collection<ExecutionGroup> executionGroups) {

        // 提交所有执行组到线程池
        List<Future<QueryResult>> futures = new ArrayList<>();
        for (ExecutionGroup group : executionGroups) {
            futures.add(executorService.submit(() ->
                executeGroup(group)
            ));
        }

        // 收集结果
        List<QueryResult> results = new ArrayList<>();
        for (Future<QueryResult> future : futures) {
            results.add(future.get());
        }
        return results;
    }
}
```

线程池规模由 `kernel-executor-size` 控制（默认 CPU 核数），**池是全局共享的，不是每查询独占**——高扇出查询多时线程池排队会反压整个应用的 SQL 延迟，这是"连接够、库不忙、但整体 RT 飙升"类故障的常见根因（排查看 jstack 的 ShardingSphere-Executor 线程堆积，20 篇）。错误传播遵循"快速失败 + 全组回滚"语义：任一单元抛 SQLException，收集阶段 future.get 抛出即中断，已执行成功的其他分片**不会自动补偿**，DML 部分成功问题交由事务层兜底——这就是"无事务的批量写跨分片不是原子"的执行层解释。

> **设计考量**：5.x 在 Proxy 形态引入虚拟线程/协程化的讨论与实验分支，动机是执行模型"阻塞 IO + 线程扇出"对高并发点查不友好；JDBC 形态受宿主应用线程模型约束，跟随 JDK 版本演进即可，业务侧不需要额外动作。

# 4. 归并引擎

```java
public final class MergeEngine {

    public MergedResult merge(List<QueryResult> queryResults,
                               SQLStatement sqlStatement) {
        // 使用装饰者模式组合多种归并策略
        MergedResult result;

        // 1. 基础：遍历归并（迭代器模式）
        result = new IteratorStreamMergedResult(queryResults);

        // 2. 排序归并（ORDER BY）
        if (sqlStatement.hasOrderBy()) {
            result = new OrderByStreamMergedResult(
                result, sqlStatement.getOrderBy());
        }

        // 3. 分组归并（GROUP BY）
        if (sqlStatement.hasGroupBy()) {
            result = new GroupByStreamMergedResult(
                result, sqlStatement.getGroupBy());
        }

        // 4. 分页归并（LIMIT）
        if (sqlStatement.hasLimit()) {
            result = new LimitDecoratorMergedResult(
                result, sqlStatement.getLimit());
        }

        return result;
    }
}
```

归并模型是**装饰器链**：最内层 `IteratorStreamMergedResult` 定义"逐结果集取完再取下一个"的裸迭代；外层按需叠 `Sort`/`GroupBy`/`Distinct`/`Limit` 装饰器，每层只改 `next()`/`currentRow()` 的行为。**链的组装顺序由 SQLStatement 语义分析决定，不是运行时动态**——这是归并结果可预期、可测试的原因。ORM 拿到的 `ResultSet` 是这条链的游标视图，`ResultSet#next()` 每被调一次驱动一层装饰器推进。

# 5. 排序归并

```java
public class OrderByStreamMergedResult extends StreamMergedResult {

    private final PriorityQueue<OrderByValue> priorityQueue;

    public OrderByStreamMergedResult(List<QueryResult> queryResults,
                                      OrderBySegment orderBy) {
        // K 路归并：使用优先队列
        this.priorityQueue = new PriorityQueue<>(queryResults.size());
        for (QueryResult result : queryResults) {
            if (result.next()) {
                priorityQueue.offer(
                    new OrderByValue(result.currentRow(), orderBy));
            }
        }
    }

    @Override
    public boolean next() {
        if (priorityQueue.isEmpty()) return false;

        // 取出最小元素
        OrderByValue min = priorityQueue.poll();
        setCurrentRow(min.getRow());

        // 从该元素所在的结果集取下一行
        if (min.next()) {
            priorityQueue.offer(min);
        }
        return true;
    }
}
```

**K 路归并的前提是每个分片内部已按 ORDER BY 列有序**（改写后的物理 SQL 保留了 ORDER BY），因此归并层不需要物化任何数据，内存占用恒定为"每分片一行"。比较器基于 OrderByItem 的列值 + ASC/DESC，多列按声明序逐列比较。**排序列不在 SELECT 列表时的处理**：改写引擎会把 order by 列**临时追加**到物理 SQL 投影中，归并层按位置找到该列参与比较、返回给客户端前再丢弃——"明明没投影的列为什么物理 SQL 里有"是这一机制，不是 bug。

## 5.1 去重归并

`DistinctStreamMergedResult` 维护一个已见行的 Set（按去重列哈希），流式跳过重复行；基数大时 Set 常驻内存，**DISTINCT 列基数不可控时等价于内存风险**，与 15 篇 COUNT(DISTINCT) 的困境同源——治理思路同样是近似（ES/ClickHouse）或业务约束。

# 6. 分组归并

```java
public class GroupByStreamMergedResult extends StreamMergedResult {

    @Override
    public boolean next() {
        // 1. 从底层结果集取一行
        if (!super.next()) return false;

        // 2. 判断当前行是否与上一行属于同一组
        // 3. 如果同组 → 聚合计算（SUM/COUNT/AVG）
        // 4. 如果不同组 → 输出上一组结果，开始新组
        aggregate();
        return true;
    }
}
```

分组归并能流式的前提是改写引擎做了两件事：**物理 SQL 补投影 GROUP BY 列与 ORDER BY 列**（保证分组键有序进入归并），**聚合列改写为归并可消费的中间形态**（AVG→SUM+COUNT，见 15 篇）。聚合状态由 `AggregationUnit` 族维护：`SumAggregationUnit`/`CountAggregationUnit` 累加、`AverageAggregationUnit` 保 sum/count 两个累加器最后相除、`MaxMinAggregationUnit` 取极值。**内存占用 = O(一个组)** 而非 O(全部数据)——前提是同组行连续到达，这正是"补 ORDER BY 分组键"的深层理由。

# 7. 分页归并（装饰者）

```java
public class LimitDecoratorMergedResult extends DecoratorMergedResult {

    private final int limit;
    private final int offset;
    private int currentRow;

    @Override
    public boolean next() {
        // 跳过 offset 行
        while (currentRow < offset) {
            if (!super.next()) return false;
            currentRow++;
        }
        // 返回 limit 行
        if (currentRow - offset >= limit) return false;
        currentRow++;
        return super.next();
    }
}
```

分页装饰器永远处于链的最外层（先排序分组、最后截取），与 15 篇改写侧的"每片取 offset+limit"配对：物理多读的行在归并层被 skip 掉。**ORDER BY + LIMIT 但排序列无索引、或分片内 limit 语义不同（带 JOIN 的 limit 下推规则）时，改写与归并的 offset 计算必须一致**，历史上多个版本修复过"JOIN + LIMIT"的边界 case，跨版本升级后回归测试要覆盖分页专项。

# 8. 流式归并 vs 内存归并

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **流式归并** | 边读边归并，内存占用小 | 大数据量 |
| **内存归并** | 全部加载到内存后归并 | ORDER BY + GROUP BY 组合 |

ShardingSphere 优先使用流式归并，仅在 ORDER BY 和 GROUP BY 同时存在时使用内存归并。

## 8.1 内存归并的真实触发条件

更精确的说法：**需要跨全量数据才能确定的语义必须物化**——无 ORDER BY 的 GROUP BY（组与组之间无到达顺序保证，无法边流边出）、DISTINCT 全量去重、以及部分 UNION 场景，走 `MemoryQueryResult`（先读完所有物理结果集入内存再模拟归并）。流式/内存的选择在归并引擎入口自动判定，判定依据来自 SQLStatement 的语义标记，因此**"改 SQL 写法"能实质改变内存曲线**：给 GROUP BY 查询显式加 `ORDER BY 分组键`，多数场景即回到流式。Proxy 形态下这条曲线还叠加 Netty 写缓冲与前后端异步流控（`proxy-frontend-flush-threshold`），大结果集导出应优先走客户端 fetchSize + 流式拉取，而不是 SELECT 到内存再分页翻页。

> **注意**：流式的前提是物理游标不被提前物化——MySQL 驱动默认把整个结果集拉到客户端，必须 `useCursorFetch=true` + `fetchSize>0`（或 `stmt.setFetchSize(Integer.MIN_VALUE)` 的老式流）才真正游标化；PG 驱动需 `autoCommit=false` + fetchSize。"明明归并是流式，内存还是被打满"九成是物理层已经全量缓冲，归并层无力回天。

# 9. 小结

1. **执行分组决定连接与内存的此消彼长**：DML 连接严格、SELECT 内存严格，max-connections-size-per-query 是主旋钮
2. **执行线程池全局共享**，高扇出查询反压整机 RT，排查看 executor 线程堆积
3. **归并是装饰器链**：流式遍历→排序（K 路归并）→分组（聚合单元累加）→分页（skip/take）
4. **流式的前提是到达有序 + 物理游标真流式**：补 ORDER BY、配 fetchSize，否则内存归并/全量缓冲
5. **无事务的多分片写不原子**，原子性由事务层而非执行层保证（见 18 篇）