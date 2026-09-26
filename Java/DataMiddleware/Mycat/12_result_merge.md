
# 1. 概述

当 SQL 需要跨多个分片执行时，Mycat2 需要将各分片的**部分结果合并为最终结果**。合并策略取决于 SQL 类型。

```
SQL: SELECT * FROM users ORDER BY age LIMIT 10

db0 → [{id:3, age:20}, {id:1, age:25}]
db1 → [{id:5, age:18}, {id:2, age:30}]
db2 → [{id:6, age:22}, {id:4, age:28}]

         │ 归并排序 (按 age)
         ▼

最终: [{id:5, age:18}, {id:3, age:20}, {id:6, age:22},
       {id:1, age:25}, {id:4, age:28}, {id:2, age:30}]
         └─── 取前 10 ────┘
```

# 2. 聚合合并

## 2.1 SUM / COUNT

```sql
SELECT COUNT(*), SUM(amount) FROM orders;
```

```
db0: COUNT=1000, SUM=50000
db1: COUNT=800,  SUM=40000
db2: COUNT=1200, SUM=60000

        │ 求和
        ▼

最终: COUNT=3000, SUM=150000
```

## 2.2 AVG

```sql
SELECT AVG(amount) FROM orders;
```

```
db0: SUM=50000, COUNT=1000
db1: SUM=40000, COUNT=800
db2: SUM=60000, COUNT=1200

        │ 二次计算
        ▼

最终: AVG = (50000+40000+60000) / (1000+800+1200) = 150000/3000 = 50
```

## 2.3 MAX / MIN

```sql
SELECT MAX(age), MIN(age) FROM users;
```

```
db0: MAX=65, MIN=18
db1: MAX=72, MIN=22
db2: MAX=58, MIN=15

        │ 取极值
        ▼

最终: MAX=72, MIN=15
```

# 3. 排序合并（归并排序）

```java
// 伪代码：跨分片排序合并
public List<Row> mergeWithOrderBy(List<Future<ResultSet>> futures,
                                    Comparator<Row> comparator) {
    // 1. 每个分片的结果作为有序流
    List<Iterator<Row>> iterators = new ArrayList<>();
    for (Future<ResultSet> future : futures) {
        ResultSet rs = future.get();
        iterators.add(new ResultSetIterator(rs));
    }

    // 2. 归并排序（K 路归并）
    PriorityQueue<Row> heap = new PriorityQueue<>(comparator);
    for (int i = 0; i < iterators.size(); i++) {
        if (iterators.get(i).hasNext()) {
            heap.offer(new Row(i, iterators.get(i).next()));
        }
    }

    // 3. 依次取出最小元素
    List<Row> result = new ArrayList<>();
    while (!heap.isEmpty()) {
        Row min = heap.poll();
        result.add(min);
        int source = min.getSourceIndex();
        if (iterators.get(source).hasNext()) {
            heap.offer(new Row(source, iterators.get(source).next()));
        }
    }
    return result;
}
```

# 4. 分页合并

```sql
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 100;
```

**处理流程**：

```
1. 从每个分片取 OFFSET + LIMIT = 120 条（已排序）
2. 归并排序，得到全局有序的 120 × 分片数 条
3. 截取第 101-120 条返回

优化：
  - 如果分片键在 ORDER BY 中 → 可直接裁剪分片范围的 OFFSET
```

**深度分页优化**：

```sql
-- 深度分页性能差：OFFSET 10000 LIMIT 20
-- 优化方案：使用上次查询的最大 ID 作为起点
SELECT * FROM users
WHERE id > 9999     -- 上次结果的最大 ID
ORDER BY id LIMIT 20;
```

# 5. GROUP BY 合并

```sql
SELECT city, COUNT(*), AVG(age) FROM users GROUP BY city;
```

```
Phase 1 (各分片):
db0 → [{北京, 500, 32.5}, {上海, 300, 28.3}]
db1 → [{北京, 400, 31.2}, {上海, 350, 29.1}]
db2 → [{北京, 600, 33.0}, {上海, 250, 27.8}]

         │ 二次聚合

Phase 2 (Mycat2 归并):
北京: COUNT=1500, AVG=(500*32.5+400*31.2+600*33.0)/1500
上海: COUNT=900,  AVG=(300*28.3+350*29.1+250*27.8)/900
```

# 6. JOIN 合并

```sql
-- ER 表 JOIN（同分片）
SELECT o.*, i.product_id
FROM orders o
JOIN order_items i ON o.id = i.order_id
WHERE o.id = 1;
-- → 精确路由到分片 0 → 后端 MySQL 本地 JOIN

-- 跨分片 JOIN（非 ER 表）
SELECT u.name, o.amount
FROM users u
JOIN orders o ON u.id = o.user_id;
-- → 全分片 JOIN → 结果归并（性能差）
```

# 7. 流式合并

```java
// 当结果集很大时，Mycat2 使用流式合并避免 OOM
public class StreamingMerger {

    private static final int BUFFER_SIZE = 1024;

    public void merge(List<ResultSet> resultSets, OutputStream out) {
        // 分批从各分片读取
        // 分批归并
        // 分批写入客户端
        // 避免将全部结果加载到内存
    }
}
```

# 8. 内存控制

```yaml
# mycat.yml
server:
  merge:
    maxHeapSize: 268435456       # 256MB 最大堆内内存
    maxDirectSize: 134217728     # 128MB 最大堆外内存
    batchSize: 1000              # 每批次处理行数
    timeout: 30000               # 合并超时(ms)
```

# 9. 合并性能优化策略

| 策略 | 说明 | 效果 |
|------|------|------|
| LIMIT 下推 | 每个分片只取需要的行数 | 减少 60-90% 数据传输 |
| 流式合并 | 分批处理，不全部加载到内存 | 避免 OOM |
| 精确路由 | 分片键在 WHERE 中 | 单节点执行，无需合并 |
| 并行获取 | 多线程同时向各分片请求 | 减少等待时间 |
| 提前截断 | 排序合并时，超出 LIMIT 的数据提前丢弃 | 减少内存占用 |
