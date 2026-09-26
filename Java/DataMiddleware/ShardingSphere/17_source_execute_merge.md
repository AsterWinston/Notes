
# 1. 执行引擎

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

# 8. 流式归并 vs 内存归并

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **流式归并** | 边读边归并，内存占用小 | 大数据量 |
| **内存归并** | 全部加载到内存后归并 | ORDER BY + GROUP BY 组合 |

ShardingSphere 优先使用流式归并，仅在 ORDER BY 和 GROUP BY 同时存在时使用内存归并。
