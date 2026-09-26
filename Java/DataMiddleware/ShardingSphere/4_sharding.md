
# 1. 分片核心概念

```
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

| 概念 | 说明 |
|------|------|
| **逻辑表** | 应用程序中使用的表名（t_order） |
| **物理表** | 实际数据库中的表名（t_order_0, t_order_1） |
| **绑定表** | 按相同分片键分片的主子表，避免笛卡尔积 |
| **广播表** | 每个数据源中全量存储的小表 |
| **分片键** | 用于水平拆分的数据列 |

# 2. 分片算法

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

**优点**：简单直接，性能最高  
**限制**：仅支持简单数学运算（+ - * / %），不支持复杂逻辑

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

**效果**：2024-01 → t_order_202401，2024-02 → t_order_202402

## 2.3 HASH_MOD 取模

```yaml
shardingAlgorithms:
  hash_mod:
    type: HASH_MOD
    props:
      sharding-count: 4
```

## 2.4 AUTO_INTERVAL 自动区间

```yaml
shardingAlgorithms:
  auto_interval:
    type: AUTO_INTERVAL
    props:
      datetime-lower: 2024-01-01 00:00:00
      datetime-upper: 2024-12-31 23:59:59
      sharding-seconds: 2592000    # 30 天
```

## 2.5 CLASS_BASED 自定义

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

# 3. 分片策略

## 3.1 STANDARD — 精确分片

```yaml
tables:
  t_order:
    databaseStrategy:
      standard:
        shardingColumn: user_id
        shardingAlgorithmName: database_inline
```

**匹配**：`WHERE user_id = ?` 和 `WHERE user_id IN (?, ?, ?)`

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
        // 根据多个分片键计算
    }
}
```

## 3.3 HINT — 强制路由

```java
// 代码中强制指定分片
HintManager hintManager = HintManager.getInstance();
hintManager.setDatabaseShardingValue(0);  // 强制路由到 ds0
try {
    jdbcTemplate.query("SELECT * FROM t_order", ...);
} finally {
    hintManager.close();
}
```

## 3.4 NONE — 不分片

用于不分片的表（全局表或广播表）。

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

**效果**：

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

# 5. 广播表

```yaml
rules:
  - !SHARDING
    broadcastTables:
      - t_config          # 配置表
      - t_district        # 字典表
```

广播表在每个数据源存储**全量数据**，常用于字典表、配置表。

# 6. 分布式主键生成器

| 类型 | 说明 | 性能 |
|------|------|------|
| `SNOWFLAKE` | 雪花算法，趋势递增 | 极高 |
| `UUID` | UUID 随机字符串 | 极高 |
| `NANOID` | NanoID 短 ID | 极高 |
| `COSID_SNOWFLAKE` | CosId 增强雪花 | 极高 |
| `AUTO_INCREMENT` | 依赖数据库自增 | 低 |

```yaml
keyGenerators:
  snowflake:
    type: SNOWFLAKE
    props:
      worker-id: 1
      max-vibration-offset: 3
      max-tolerate-time-difference-milliseconds: 10
```

# 7. 分页优化

```sql
-- 普通分页（性能差）
SELECT * FROM t_order ORDER BY order_id LIMIT 100, 10;
-- 问题：所有分片各取 110 条 → 归并 → 丢弃前 100 条

-- 优化 1：流式归并 + 提前终止
-- 归并排序时，累计 skip 100 条后开始收集

-- 优化 2：非分片键排序的分页
-- 无法优化，需要全分片取数归并
```
