数据分片是 ShardingSphere 的立身之本，其余功能（读写分离、影子、加密）都是围绕它或叠加在它之上。本篇按"**概念 → 算法家族 → 策略 → 表关系 → 主键 → 分页与代价**"的顺序展开：先弄清楚分片键怎么选、行表达式和各类算法各自适合什么场景，再讨论绑定表/广播表如何消灭笛卡尔积，最后给出跨分片查询的代价分级——这一节决定了你写 SQL 的习惯。所有配置均为 5.5.x 的 `!SHARDING` 规则格式。

# 1. 分片核心概念

```text
逻辑表 t_order (1000 万行)
    │
    │ user_id % 2 (分库) + order_id % 2 (分表)
    │
    ├──→ ds0 ── t_order_0  (order_id % 2 = 0, user_id % 2 = 0)
    │          └ t_order_1  (order_id % 2 = 1, user_id % 2 = 0)
    │
    └──→ ds1 ── t_order_0  (order_id % 2 = 0, user_id % 2 = 1)
               └ t_order_1  (order_id % 2 = 1, user_id % 2 = 1)
```

分片的本质是**用一个列（或几个列）的值决定数据落在哪个物理节点上**，读写时再按同一算法反向定位。这个"决定落点"的列就是分片键，它同时是路由的依据，因此**选分片键就是选查询模式**：绝大多数 SQL 的 WHERE 里出现谁，就应该拿谁做分片键。

| 概念 | 说明 |
|------|------|
| **逻辑表** | 应用程序中使用的表名（t_order） |
| **物理表** | 实际数据库中的表名（t_order_0, t_order_1） |
| **绑定表** | 按相同分片键分片的主子表，避免笛卡尔积 |
| **广播表** | 每个数据源中全量存储的小表 |
| **分片键** | 用于水平拆分的数据列 |
| **actualDataNodes** | 行表达式声明的物理数据节点集合 |

> **行业实践**：互联网业务分库分表的主流做法是"**分库键与分表键同源**"——都用 user_id（或订单号中嵌 user_id 段），保证同一用户的数据聚在同一库，绝大多数"我的订单"类查询单库完成。分片数一般取 **2 的幂**（16/32/64），为将来成倍扩容（翻倍分裂）留出空间，避免取 15 这种质数导致再分片时数据倾斜。

# 2. 分片算法

5.x 把分片算法抽成独立的 `shardingAlgorithms` 配置段，按"是否需要 SQL 中出现分片键"分为两大类：**自动分片算法**（Auto Sharding，无需在 SQL 条件中带分片键，内核自动追加条件）与**标准/复合/Hint 分片算法**（按策略类型绑定使用）。常用家族一览：

| 算法 type | 类别 | 适用 | 关键 props |
|-----------|------|------|-----------|
| `INLINE` | 标准（行表达式） | 简单取模、数值分片 | `algorithm-expression` |
| `INTERVAL` | 标准 | 时间范围（按月/天分表） | `datetime-pattern`、`sharding-suffix-pattern`、`datetime-interval-unit` |
| `HASH_MOD` | 自动 | 分片键不出现在 SQL 中 | `sharding-count` |
| `MOD` | 自动 | 数值直接取模 | `sharding-count` |
| `AUTO_INTERVAL` | 自动 | 时间自动分段 | `datetime-lower/upper`、`sharding-seconds` |
| `VOLUME_RANGE` / `BOUNDARY_RANGE` | 自动 | 按容量/边界范围 | `sharding-volume`、`sharding-ranges` |
| `COMPLEX_INLINE` | 复合 | 多列行表达式 | `sharding-columns`、`algorithm-expression` |
| `CLASS_BASED` | 任意 | 自定义 Java 类 | `strategy`、`algorithmClassName` |

## 2.1 INLINE 行表达式

```yaml
shardingAlgorithms:
  database_inline:
    type: INLINE
    props:
      algorithm-expression: ds${user_id % 2}
  table_inline:
    type: INLINE
    props:
      algorithm-expression: t_order_${order_id % 2}
```

**优点**：简单直接，配置即算法，无需写 Java；**限制**：基于 Groovy 表达式，仅支持简单数学运算（+ - * / %），只支持 `=` 和 `IN` 精确分片，默认**不允许范围查询**（`BETWEEN` 会报"INLINE does not support range queries"，除非设 `allow-range-query-with-inline-sharding: true`，而开了之后范围查询实际是全路由，慎用）。

## 2.2 INTERVAL 时间范围

```yaml
shardingAlgorithms:
  date_interval:
    type: INTERVAL
    props:
      datetime-pattern: yyyy-MM-dd HH:mm:ss
      datetime-lower: 2024-01-01 00:00:00
      datetime-upper: 2024-12-31 23:59:59
      sharding-suffix-pattern: yyyyMM
      datetime-interval-amount: 1
      datetime-interval-unit: MONTHS
```

**效果**：2024-01 → t_order_202401，2024-02 → t_order_202402。适合按月归档的流水表、账单表。注意 `datetime-lower/upper` 圈定了算法**认得的**时间范围，超界数据要么报"no route"要么落默认策略，按年扩容时必须更新配置或改用 AUTO_INTERVAL。

## 2.3 HASH_MOD 取模（自动分片）

```yaml
shardingAlgorithms:
  hash_mod:
    type: HASH_MOD
    props:
      sharding-count: 4
```

HASH_MOD 属于**自动分片算法**：路由时不依赖 SQL 里出现分片键，执行计划找不到该列条件时，ShardingSphere 会**自动在 WHERE 中追加 `sharding_column = 生成值`**，把数据"推"进某分片。它解决的正是"分片键不在 SQL 中"的插入/查询路由问题，代价是查询也必须带上同一条件才能收敛。

## 2.4 AUTO_INTERVAL 自动区间

```yaml
shardingAlgorithms:
  auto_interval:
    type: AUTO_INTERVAL
    props:
      datetime-lower: 2024-01-01 00:00:00
      datetime-upper: 2024-12-31 23:59:59
      sharding-seconds: 2592000    # 30 天一片
```

与 INTERVAL 相比不要求预先枚举所有分表后缀，按秒数自动切片；超出上界的数据按最后一片处理，适合持续追加写入的时间序列表。

## 2.5 CLASS_BASED 自定义

内置算法覆盖不了时（如按租户 ID 映射到固定库表路由表、一致性哈希环），用 CLASS_BASED 桥接业务类：

```java
public class MyShardingAlgorithm implements StandardShardingAlgorithm<Long> {

    @Override
    public String doSharding(Collection<String> availableTargetNames,
                              PreciseShardingValue<Long> shardingValue) {
        // 自定义分片逻辑
        Long value = shardingValue.getValue();
        long shard = (value / 1000) % availableTargetNames.size();
        return "t_order_" + shard;
    }

    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames,
                                          RangeShardingValue<Long> shardingValue) {
        // 范围分片：返回区间覆盖的所有目标表
        return availableTargetNames;
    }
}
```

```yaml
shardingAlgorithms:
  custom:
    type: CLASS_BASED
    props:
      strategy: standard
      algorithmClassName: com.example.MyShardingAlgorithm
```

**要点**：`strategy` 指定 standard/complex/hint 三种之一；`doSharding` 的返回值**必须落在 availableTargetNames 内**，返回不存在的表名会得到路由错误而不是静默兜底。想跳过 SPI 注册文件，CLASS_BASED 就是为这个便利设计的。

# 3. 分片策略

策略（Strategy）决定"SQL 的哪个条件喂给哪个算法"。5.x 支持四种，全部挂在 `databaseStrategy`/`tableStrategy` 下。

## 3.1 STANDARD — 精确分片

```yaml
tables:
  t_order:
    databaseStrategy:
      standard:
        shardingColumn: user_id
        shardingAlgorithmName: database_inline
```

**匹配**：`WHERE user_id = ?` 和 `WHERE user_id IN (?, ?, ?)` 走精确路由；`BETWEEN`、`>`、`<` 走范围路由（由算法的 RangeShardingValue 分支处理，INLINE 默认拒绝）。一个 standard 策略只能绑定**一个分片列**。

## 3.2 COMPLEX — 复合分片键

```yaml
databaseStrategy:
  complex:
    shardingColumns: user_id, order_id
    shardingAlgorithmName: complex_algorithm
```

```java
public class ComplexAlgorithm implements ComplexKeysShardingAlgorithm<Comparable<?>> {
    @Override
    public Collection<String> doSharding(Collection<String> targets,
            ComplexKeysShardingValue<Comparable<?>> shardingValue) {
        // 根据多个分片键计算：Map<column, Range> + Map<column, Set<value>>
        // 常见做法：主键中嵌了 user_id 段时，两列取交集避免全路由
    }
}
```

复合策略的返回值是**目标集合**（可能是多个），所以"只给其中一个列的条件"也能工作，但要想避免路由放大，通常把分片键设计成"另一个键的函数"（如 order_id 生成时混入 user_id）。

## 3.3 HINT — 强制路由

```java
// 代码中强制指定分片：SQL 里完全不带分片键也能精准路由
HintManager hintManager = HintManager.getInstance();
hintManager.setDatabaseShardingValue(0);  // 强制路由到 ds0
try {
    jdbcTemplate.query("SELECT * FROM t_order", ...);
} finally {
    hintManager.close();   // ThreadLocal 实现，必须 close，否则线程复用串路由
}
```

也可配 `HINT` 策略让算法从 `HintShardingValue` 里取值（配合 HINT_INLINE 表达式 `${value % 2}`）。典型用途：**历史数据修复任务**按分片位逐个扫描、基因法失效时的兜底、Proxy 场景通过 `/* SHARDINGSPHERE_HINT: ... */` 注释打标。

## 3.4 NONE — 不分片

用于不分片的表。注意与广播表的区别：NONE 只落在一个数据节点；广播表每个库都存全量且 JOIN 时自动同库对齐。

> **注意**：`HintManager` 是 **ThreadLocal 语义**。用线程池时必须"用完即关"，异常路径也要放进 finally；异步执行器（CompletableFuture 换线程）里 Hint 会丢，跨线程路由要显式传递或使用 SQL Hint 注释。

# 4. 绑定表（Binding Table）

```yaml
rules:
  - !SHARDING
    bindingTables:
      - t_order,t_order_item    # 主子表按相同分片键分片
    tables:
      t_order:
        actualDataNodes: ds${0..1}.t_order_${0..1}
        databaseStrategy: {...}
        tableStrategy: {...}
      t_order_item:
        actualDataNodes: ds${0..1}.t_order_item_${0..1}
        databaseStrategy: {...}
        tableStrategy: {...}
```

绑定表的判定标准很严格：**分库键相同、分表键相同、分片算法相同**，三者齐备才能声明绑定。声明后主子表 JOIN 按分片单元一一对应执行：

```sql
-- 无绑定表：关联查询产生笛卡尔积路由
-- ds0.t_order_0 JOIN ds0.t_order_item_0
-- ds0.t_order_0 JOIN ds0.t_order_item_1  ← 无意义
-- ds0.t_order_1 JOIN ds0.t_order_item_0  ← 无意义
-- ds0.t_order_1 JOIN ds0.t_order_item_1

-- 有绑定表：仅产生 4 条而不是 16 条
-- ds0.t_order_0 JOIN ds0.t_order_item_0
-- ds0.t_order_1 JOIN ds0.t_order_item_1
-- ds1.t_order_0 JOIN ds1.t_order_item_0
-- ds1.t_order_1 JOIN ds1.t_order_item_1
```

**设计考量**：绑定表解决的是**分片内 JOIN** 的执行计划问题，但两张表还必须"同主键值落同分片"才有正确性——订单与订单项都按 order_id 分片时 JOIN 才不会跨库。若主子表分别按 user_id、order_id 分片，绑定也救不了跨分片 JOIN，这是分片键设计阶段就要想清楚的事。

# 5. 广播表

```yaml
rules:
  - !SHARDING
    broadcastTables:
      - t_config          # 配置表
      - t_district        # 字典表
```

广播表在每个数据源存储**全量数据**，常用于字典表、配置表。读任意分片本地可得，与分片表 JOIN 自动同库执行；写操作会**同步发到所有分片**，所以只适合低频变更的小表。写入放大 = 分片数，频繁写的表当广播表用是灾难。5.x 中广播表是独立规则 `!BROADCAST`，与 SHARDING 规则可共存。

# 6. 分布式主键生成器

单库自增在分片下失效（多表各自增会撞号），必须全局唯一：

| 类型 | 说明 | 特点 |
|------|------|------|
| `SNOWFLAKE` | 雪花算法，趋势递增 | 高吞吐；依赖时钟，回拨有风险 |
| `UUID` | 随机字符串 | 无序，InnoDB 页分裂重 |
| `NANOID` | 短随机 ID | 更短，同无序 |
| `COSID` / `COSID_SNOWFLAKE` | CosId 号段/增强雪花 | 可配时钟回拨容忍 |

```yaml
keyGenerators:
  snowflake:
    type: SNOWFLAKE
    props:
      worker-id: 1
      max-vibration-offset: 3
      max-tolerate-time-difference-milliseconds: 10
```

**注意 worker-id**：Standalone 模式下 SNOWFLAKE 不配 `worker-id` 时默认取 **0**——多实例各自独立跑就会撞号重复发号；Cluster 模式下由注册中心**自动生成且同 namespace 内不重复**，所以多实例场景要么上 Cluster、要么手工保证每实例 `worker-id` 唯一。另有 `max-vibration-offset`（同毫秒内抖动上限，把 ID 低位打散做分片键时建议配 (2^n)-1）与 `max-tolerate-time-difference-milliseconds`（时钟回拨容忍毫秒数，超阈值直接抛异常停写）。

> **设计争议**：分片主键用**数据库自增 + 步长**（每库设 offset/interval）还是**分布式生成器**？自增方案保住 InnoDB 顺序写性能，但扩容改步长极痛苦、异构库难统一；生成器方案通用，但 UUID 打散 B+ 树、雪花依赖时钟。主流折中是"雪花 + 业务前缀"或直接引入 Leaf/CosId 这类中心发号服务。

# 7. 分页优化

```sql
-- 普通分页（性能差）
SELECT * FROM t_order ORDER BY order_id LIMIT 100, 10;
-- 问题：所有分片各取 110 条 → 归并 → 丢弃前 100 条

-- 优化 1：流式归并 + 提前终止
-- 归并排序时，累计 skip 100 条后开始收集，物理 SQL 仍取 offset+count

-- 优化 2：游标分页（推荐）
SELECT * FROM t_order WHERE order_id > #{lastMaxId} ORDER BY order_id LIMIT 10;
-- 每分片仍各取 10 条，但代价恒定，与页码深度无关

-- 优化 3：非分片键排序的分页
-- 无法优化，必须全分片取数归并；业务上限制最大页码
```

深分页的本质矛盾：排序键与分片键无关时，第 N 页数据"可能在任何分片"，物理层无法下推 offset。能带分片键就带（单分片后数据库原生 LIMIT 即可），不能带就游标化，这是唯一两条稳的路。

# 8. 跨分片查询代价分级

把日常 SQL 按路由宽度分四档，心里有了这张表就很少写出灾难 SQL：

```text
代价
 ▲
 │  ┌────────────┐
 │  │ Ⅳ 全库全表  │  SELECT * FROM t_order（无分片键）
 │  ├────────────┤
 │  │ Ⅲ 范围/部分 │  BETWEEN、IN 多值、allow-range 的 INLINE
 │  ├────────────┤
 │  │ Ⅱ 单库多表  │  只带分库键：ds1 的 2 张表
 │  ├────────────┤
 │  │ Ⅰ 单库单表  │  带全分片键：WHERE user_id=? AND order_id=?
 │  └────────────┘
 └──────────────────────────────▶ 频率（应当反过来）
```

| 级别 | 路由结果 | 物理 SQL 数 | 适用 |
|------|---------|------------|------|
| Ⅰ 精确 | 1 库 1 表 | 1 | 在线交易主路径 |
| Ⅱ 单库多表 | 1 库 N 表 | N | 带分库键的列表页 |
| Ⅲ 范围 | 部分分片 | 中等 | 时间维度报表 |
| Ⅳ 全路由 | 全部分片 | 分片总数 × 归并 | 后台导出、低频统计 |

> **行业实践**：交易类系统要求 **95% 以上 SQL 落在Ⅰ/Ⅱ级**；Ⅳ级查询一律赶到大促低峰或走异构（ES/ClickHouse）链路。上线前用 `PREVIEW SELECT ...`（DistSQL RUL）或压测看 Actual SQL 数，是最便宜的容量审计。

# 9. 小结

| 主题 | 一句话结论 |
|------|-----------|
| 分片键 | 选分片键 = 选查询模式，主流用法是同源（user_id）+ 2 的幂分片数 |
| 算法 | INLINE 简单快、INTERVAL 管时间、自动分片解"键不在 SQL"、CLASS_BASED 兜底 |
| 策略 | STANDARD 一列、COMPLEX 多列、HINT 强制、NONE 不分；Hint 是 ThreadLocal 必须 close |
| 绑定表 | 分片列与算法完全一致才可绑定，消灭 JOIN 笛卡尔积 |
| 广播表 | 读本地化、写放大 N 倍，只配字典/配置 |
| 主键 | SNOWFLAKE worker-id 要显式管；UUID 慎入 InnoDB |
| 分页 | 深分页无银弹：带分片键或游标化 |
| 代价 | 以路由宽度分级衡量 SQL，PREVIEW 验证后再上线 |

下一篇（5_read_write_splitting）看读流量如何在主从之间分配，以及与分片叠加后的行为。
