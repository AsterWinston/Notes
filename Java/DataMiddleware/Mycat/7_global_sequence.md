
# 1. 概述

Mycat2 的全局序列（Sequence）为分片表提供**全局唯一递增 ID**，与分布式主键紧密配合。区别在于序列更通用，不仅限于主键。

# 2. 序列配置

```json
{
  "schemas": {
    "order_db": {
      "sequences": {
        "order_no_seq": {
          "clazz": "io.mycat.router.sequence.SnowflakeSequenceHandler",
          "properties": {
            "workerId": 1,
            "dataCenterId": 1
          }
        }
      },
      "tables": {
        "orders": {
          "type": "sharding",
          "autoIncrement": true,
          "sequence": "order_no_seq"
        }
      }
    }
  }
}
```

# 3. 序列类型

## 3.1 Snowflake 序列

```json
{
  "clazz": "io.mycat.router.sequence.SnowflakeSequenceHandler",
  "properties": {
    "workerId": 1,
    "dataCenterId": 1,
    "startTimestamp": 1700000000000
  }
}
```

**特点**：
- 趋势递增（非严格递增）
- 高性能（每秒可生成百万级 ID）
- 生成的 ID 长度固定（19 位）

## 3.2 数据库递增序列

```json
{
  "clazz": "io.mycat.router.sequence.DatabaseSequenceHandler",
  "properties": {
    "url": "jdbc:mysql://192.168.1.100:3306/mycat_seq",
    "user": "mycat",
    "password": "mycat123",
    "tableName": "sequence",
    "step": 1000
  }
}
```

**工作原理**：

```
Mycat2 实例 1                    DB 序列表
     │                               │
     │ UPDATE current_value =        │
     │   current_value + 1000        │
     │ WHERE seq_name = 'orders'     │
     │ ───────────────────────────→  │
     │ ←── 返回 old_value = 1000 ──  │
     │                               │
     │ 实例 1 获得区间: 1001~2000     │
     │ 缓存在本地，逐步分配            │
```

**特点**：
- 严格递增
- 有数据库依赖（单点瓶颈）
- 适合 ID 数量不大、需要严格递增的场景

## 3.3 缓存序列

```json
{
  "clazz": "io.mycat.router.sequence.CacheSequenceHandler",
  "properties": {
    "cacheSize": 10000,
    "retryTimes": 3
  }
}
```

**特点**：
- 本地缓存一批 ID
- 定时从远端序列刷新
- 适合高并发场景，减少序列获取的 RPC 开销

# 4. 序列使用场景

```sql
-- 1. 获取序列值
SELECT NEXT VALUE FOR MYCATSEQ_ORDER_NO;
-- → 202401151200001

-- 2. 插入时引用序列
INSERT INTO orders(id, order_no, user_id)
VALUES (NEXT VALUE FOR MYCATSEQ_ORDER_ID,
        NEXT VALUE FOR MYCATSEQ_ORDER_NO,
        1);
```

# 5. 序列属性

```json
{
  "sequences": {
    "my_seq": {
      "clazz": "...",
      "properties": {
        "startWith": 10000,       // 起始值
        "incrementBy": 1,         // 步长
        "minValue": 1,            // 最小值
        "maxValue": 9999999999,   // 最大值
        "cycle": false            // 是否循环
      }
    }
  }
}
```

# 6. 多实例序列一致性

```
问题：两个 Mycat2 实例各自生成 ID，如何保证不冲突？

方案 1：Snowflake（推荐）
  - 每个实例分配不同的 workerId
  - workerId 保证 ID 不冲突

方案 2：数据库序列 + step
  - 实例 A 获取 1~1000，实例 B 获取 1001~2000
  - 通过 step 隔离

方案 3：共用序列服务
  - 独立部署序列服务（基于 Redis/ZK）
  - 所有实例调用同一服务获取 ID
```

# 7. 序列监控

```sql
-- 管理端口查看序列状态
/*!mycat:showSequences*/

-- 返回示例：
-- +------------------+--------+----------+--------+
-- | sequence_name    | type   | current  | cached |
-- +------------------+--------+----------+--------+
-- | order_no_seq     | SNOW   | 20240001 | 5000   |
-- | user_id_seq      | DB     | 100000   | 200    |
-- +------------------+--------+----------+--------+
```
