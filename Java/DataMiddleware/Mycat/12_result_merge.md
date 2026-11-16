# 1. 概述

**结果归并（Result Merge）是分库分表中间件的核心竞争力**：一条 SQL 被路由到 N 个分片并行执行后，Mycat 必须把 N 个结果集拼成一个语义正确的整体流回给客户端。归并做不好，轻则性能雪崩，重则结果错误（乱序、重复、聚合值偏差）。归并策略取决于 SQL 类型——排序走归并排序、聚合走二次计算、分页走裁剪、DISTINCT 走内存去重，Mycat2 借助 Calcite 把这几种形态统一到了计划树上（见第 11 篇）。

当 SQL 需要跨多个分片执行时，Mycat2 需要将各分片的**部分结果合并为最终结果**：

```text
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

> **注意**：归并的前提是**后端结果本身已经是部分有序或可局部计算的**。分片侧 MySQL 不知道自己是"分片"，它只按本地数据返回；全局语义（排序、去重、聚合）完全由 Mycat 兜底。因此"每个分片返回的行是否有序、有序到什么程度"决定了归并算法的选择——这也是为什么 ORDER BY 列与分片算法无关时，Mycat 只能各分片先排再归并。

# 2. 聚合合并

聚合是最"便宜"的跨分片运算，因为大多数聚合函数存在**可分解的两段式语义**：分片先算局部值，中间件再合并全局值。

## 2.1 SUM / COUNT

```sql
SELECT COUNT(*), SUM(amount) FROM orders;
```

```text
db0: COUNT=1000, SUM=50000
db1: COUNT=800,  SUM=40000
db2: COUNT=1200, SUM=60000

         │ 求和
         ▼

最终: COUNT=3000, SUM=150000
```

COUNT/SUM/MAX/MIN 都满足**结合律**（局部结果再聚合等于全局聚合），归并只需一次线性扫描，内存占用为常数级——每个分片只保留一行局部聚合值。COUNT 在无 WHERE 时还可以进一步改写为各分片 `SELECT COUNT(*)`，避免任何行传输。

## 2.2 AVG

```sql
SELECT AVG(amount) FROM orders;
```

```text
db0: SUM=50000, COUNT=1000
db1: SUM=40000, COUNT=800
db2: SUM=60000, COUNT=1200

         │ 二次计算
         ▼

最终: AVG = (50000+40000+60000) / (1000+800+1200) = 150000/3000 = 50
```

**AVG 是不可直接合并的典型**：各分片的 AVG 求平均 ≠ 全局 AVG（分片行数不等时偏差明显）。Mycat 的处理是**改写**——把 `AVG(x)` 拆成下发 `SUM(x), COUNT(x)`，归并后再相除。这类"聚合函数改写"是 Calcite 计划阶段的产物，用 `EXPLAIN` 能看到下发 SQL 已与用户 SQL 不同。

## 2.3 MAX / MIN

```sql
SELECT MAX(age), MIN(age) FROM users;
```

```text
db0: MAX=65, MIN=18
db1: MAX=72, MIN=22
db2: MAX=58, MIN=15

         │ 取极值
         ▼

最终: MAX=72, MIN=15
```

## 2.4 DISTINCT 与 COUNT(DISTINCT)

DISTINCT 没有可分解的两段式语义：**`COUNT(DISTINCT x)` 无法由各分片的局部去重计数合并得到**，因为同一个值可能同时出现在多个分片（db0 和 db1 都算出"北京"，各自计数后相加会重复）。Mycat 只能把各分片去重后的 x 值全部拉到中间件再做一次全局去重计数——代价是**传输量等于去重基数**。更糟的是 `DISTINCT + ORDER BY + LIMIT` 组合：每个分片要拉"去重且有序"的完整序列再全局归并，大表上极易 OOM。

> **注意**：基数估算需求（如 UV 统计）不要指望中间件层 `COUNT(DISTINCT)`，应在应用层用 HyperLogLog 等近似算法按分片预聚合；精确去重则尽量把去重键设计成分片键，让同值必同片，各分片局部去重后相加才等价于全局去重。

# 3. 排序合并（归并排序）

跨分片排序的标准解是 **K 路归并 + 大小为 K 的小顶堆**（K = 分片数）。关键性质：堆里任何时刻只驻留 K 行（每个分片当前最小未消费行），**内存占用与结果集总行数无关**，这是流式排序合并能处理大结果集的根本原因。时间复杂度 O(M log K)，M 为总行数；相比"全部拉到内存再排序"的 O(M log M) 且内存爆炸的做法，堆归并是数量级优势。

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

**两个容易忽略的正确性细节**：其一，**比较器的 NULL 语义必须与 MySQL 一致**（MySQL 中 ORDER BY ASC 时 NULL 最前），Mycat 自己实现行比较而不是依赖后端，比较逻辑出错就是静默的数据错误；其二，**排序键相同时的行序不确定性**，需要稳定顺序时必须追加唯一列作次级排序键。

# 4. 分页合并

分页是排序合并的特化：只需要前 `offset + limit` 条，归并到数量够即提前终止（堆排序天然支持，取出 limit 个最大值后丢弃堆）。

```sql
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 100;
```

**处理流程**：

```text
1. 从每个分片取 OFFSET + LIMIT = 120 条（已排序）
2. 归并排序，得到全局有序的 120 × 分片数 条
3. 截取第 101-120 条返回

优化：
  - 如果分片键在 ORDER BY 中 → 可直接裁剪分片范围的 OFFSET
```

## 4.1 跨库分页的 total 问题

**分页接口通常要返回总数，而 `COUNT(*)` 与数据页是两次独立查询**，这带来两个中间件特有的问题。第一是**成本**：翻页查询本身很便宜（LIMIT 下推后每片只取 offset+limit 行），但 total 要全分片 COUNT，分片多、表大时 total 反而是分页接口里最贵的一次操作——常见做法是 total 异步估算或缓存（前端"约 N 页"），或由业务保证不需要精确 total。第二是**一致性**：先查 COUNT=10001，再查第二页时数据已被并发删除，total 与页内容对不上；跨分片没有全局快照可依赖（Mycat 的分页两次查询通常落在不同连接、不同隐式事务边界上），这不是 Mycat 的缺陷，而是分布式系统 CAP 在分页场景的具象表现，前端按"近似总数 + 空页容错"设计是比中间件更强的解决方案更现实的路径。

## 4.2 深度分页优化

**深度分页的成本公式是"每片取 offset+limit 行 × 分片数"**，OFFSET 10000、LIMIT 20、16 分片意味着中间件要经手约 16 万行只为丢弃 99.98%。

```sql
-- 深度分页性能差：OFFSET 10000 LIMIT 20
-- 优化方案：使用上次查询的最大 ID 作为起点
SELECT * FROM users
WHERE id > 9999     -- 上次结果的最大 ID
ORDER BY id LIMIT 20;
```

游标分页（WHERE id > last_max）把成本从 O(offset × 分片数) 降回 O(limit × 分片数)，且天然适合"无限滚动"交互。**限制**：只能连续翻页，不能随机跳页；排序键必须唯一（否则用 (create_time, id) 复合游标）。需要随机跳页的场景（后台表格），要么接受浅分页 + total 上限截断，要么把检索交给 ES 等宽表引擎（见第 19 篇最佳实践）。

# 5. GROUP BY 合并

GROUP BY 兼具聚合与去重的性质：**分组可以并行、组内聚合可以两段式合并，但分组键的全局对齐必须在中间件完成**。Mycat 的改写规则与 AVG 同理：可合并函数（SUM/COUNT/MAX/MIN）分片算局部值，AVG 拆成 SUM+COUNT，下发 SQL 里"隐形"追加 GROUP BY 列与中间聚合列。

```sql
SELECT city, COUNT(*), AVG(age) FROM users GROUP BY city;
```

```text
Phase 1 (各分片):
db0 → [{北京, 500, 32.5}, {上海, 300, 28.3}]
db1 → [{北京, 400, 31.2}, {上海, 350, 29.1}]
db2 → [{北京, 600, 33.0}, {上海, 250, 27.8}]

         │ 二次聚合

Phase 2 (Mycat2 归并):
北京: COUNT=1500, AVG=(500*32.5+400*31.2+600*33.0)/1500
上海: COUNT=900,  AVG=(300*28.3+350*29.1+250*27.8)/900
```

**内存模型取决于分组基数**：Phase 2 需要一张"分组键 → 累加器"的 HashMap，内存 O(全局分组数)。分组基数小（城市、状态）时没问题；基数大（按 user_id 分组再 SUM 千万订单）时这张表就是 OOM 主源。若 GROUP BY 附带 ORDER BY 与 LIMIT（"取金额 Top 100 的用户"），归并必须收齐所有分片的全部分组才能排序截断——**中间件无法对 GROUP BY 做 LIMIT 下推**（各分片 Top 100 ≠ 全局 Top 100 的候选裁剪，只有 ORDER BY+LIMIT 且无分组时才能下推），这是聚合改写能力边界的第二条。

# 6. JOIN 合并

JOIN 的归并成本差异极大，核心判定是**两侧数据的分布是否对齐**（机制详见第 11 篇 4.4）：

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

跨分片 JOIN 在中间件层的典型执行是**按驱动侧结果逐行到被驱动侧点查**（嵌套循环）：被驱动侧的查询条件来自驱动侧每一行的 join key，若 join key 不是被驱动表的分片键则每次点查又变成全分片扫描——**两层全扫描相乘，灾难级成本**。Mycat2 对全局表 JOIN 分片表能走广播下推，是唯一值得依赖的"真 JOIN"形态；其余跨片 JOIN 生产上应该用宽表/冗余列/应用层组装替代。

# 7. 流式合并

**内存合并与流式合并的分水岭是"结果集能否装进内存"**。内存合并把各分片结果全部 fetch 到堆内再归并，实现简单、可支持任意多次遍历，但 OOM 风险随数据量线性上升；流式合并把每个分片结果保持为**一条只向前读的游标**，堆上只驻留归并状态（排序堆 K 行、分组表 O(分组数)），随读随写回客户端。

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

## 7.1 cursor fetchsize 与 JDBC 陷阱

流式的前提是**后端驱动本身以游标模式返回**。MySQL JDBC 默认一次性把整个结果集读进客户端内存——所谓"流式"必须显式开启：`stmt.setFetchSize(Integer.MIN_VALUE)`（MySQL Connector/J 特有的逐行流式标志）或使用游标声明。这带来一个容易被忽略的副作用：**游标未读完时，该后端连接被这一行结果"占用"，不能执行任何其他 SQL**。于是产生一个资源死锁形态：前端 5000 个并发大结果导出，每个占用 16 个后端连接共 8 万连接，连接池（maxCon）瞬间打满，正常业务查询全部排队。

> **行业实践**：大结果集导出**不要走在线 Mycat 集群**，要么从后端 MySQL 备库直连导出，要么走专门的数据同步链路（DataX、Canal 等）；确需在线导出的，限制导出并发（信号量）、单用户 maxCon，并配 statement timeout。同时把 `fetchsize` 明确设进 JDBC URL/语句，不设置时"MySQL 驱动默认全量加载"，此时 Mycat 的流式合并名存实亡，堆内存被 fetch 缓冲吃掉的是"MySQL 内存 × 分片数"。

## 7.2 归并算法的内存-能力对照

| SQL 形态 | 归并算法 | 峰值内存 | 能否流式 |
|---------|---------|---------|---------|
| 无 ORDER BY 多分片查询 | 顺序拼接（读完一片读下一片） | O(1) | 能 |
| ORDER BY (+LIMIT) | K 路堆归并 | O(分片数) | 能 |
| 两段式聚合（SUM/COUNT/MAX） | 累加合并 | O(分组数) | 分组能、去重不能 |
| DISTINCT | HashSet 去重 | O(去重基数) | 否（需全量对齐） |
| GROUP BY + ORDER BY + LIMIT | 分组表 + 全量排序 | O(分组数 × 行数) | 否 |
| 跨分片 JOIN | 逐行点查/拉全量 | O(小表侧) | 否 |

# 8. 内存控制

**Mycat 的堆内存被归并占用是常态而非异常**——中间件本身不做数据存储，堆里装的基本就是"正在归并的结果"。生产上必须为归并设置显式上限并让它"失败得清楚"：超限时宁可报错 `memory exceeded, merge aborted`，也不要放任 GC 抖死整台机器。

```yaml
# mycat.yml
server:
  merge:
    maxHeapSize: 268435456       # 256MB 最大堆内内存
    maxDirectSize: 134217728     # 128MB 最大堆外内存
    batchSize: 1000              # 每批次处理行数
    timeout: 30000               # 合并超时(ms)
```

> **注意**：Mycat 1.6.x 时代没有精细的归并内存配额，实践中的兜底配置是 schema.xml 的 `sqlMaxLimit`——给所有不带 LIMIT 的 SELECT 自动追加默认 LIMIT（如 100），防止应用裸写 `SELECT *` 拖垮中间件。**这个默认值会静默改变结果语义**（应用拿到的是被截断的 100 条而不是全量或报错），2.x 延续了类似保护但应显式配置而非依赖默认，团队内部要统一口径："所有查询必带 LIMIT"作为 SQL 规范，防火墙兜底（见第 16 篇）。

# 9. 合并性能优化策略

| 策略 | 说明 | 效果 |
|------|------|------|
| LIMIT 下推 | 每个分片只取需要的行数 | 减少 60-90% 数据传输 |
| 流式合并 | 分批处理，不全部加载到内存 | 避免 OOM |
| 精确路由 | 分片键在 WHERE 中 | 单节点执行，无需合并 |
| 并行获取 | 多线程同时向各分片请求 | 减少等待时间 |
| 提前截断 | 排序合并时，超出 LIMIT 的数据提前丢弃 | 减少内存占用 |

**并行获取**值得一提：Mycat 对各分片的下发是并行的，整体 RT ≈ max(各分片 RT) + 归并时间，而非 sum(各分片 RT)——慢分片决定整体延迟（木桶效应），单点慢 SQL、分片间数据倾斜都会把"全分片查询"的 P99 拉得很难看，这也是监控要按分片维度看 RT 分布的原因。

# 10. 小结

| 要点 | 结论 |
|------|------|
| 归并策略选择 | 无排序→顺序拼接；排序→K 路堆归并 O(K) 内存；聚合→两段式改写（AVG 拆 SUM/COUNT）；DISTINCT/GROUP BY 大基数→内存表兜底，慎用 |
| 内存合并 vs 流式 | 流式靠游标 fetchsize + 只向前归并，内存与结果总量解耦；MySQL 驱动不设 fetchsize 则流式名存实亡 |
| 分页 | LIMIT 下推救得了浅分页，救不了 OFFSET 深分页与 total；游标分页是唯一 O(limit) 方案，跳页需求外移 ES |
| 跨库 total | 成本上 total 常比数据页更贵，一致性上无法保证快照级精确——业务按近似值设计 |
| 危险形态 | `COUNT(DISTINCT)`、GROUP BY 高基数、`DISTINCT+ORDER BY`、跨分片键 JOIN——归并成本不可约，应在 SQL 规范或防火墙层面禁止 |

一句话收束：**归并层的优化空间是"少传行、少驻留"，而不是"传了也能扛住"**——扛住靠的是流式与配额，传得少靠的是路由与改写。下一篇（13 心跳与探活）从结果通道转向上游连接的健康管理：分片活着才有结果可归并。
