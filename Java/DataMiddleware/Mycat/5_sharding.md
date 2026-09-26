
# 1. 概述

分库分表（Sharding）是 Mycat2 最核心的能力，将一张逻辑表水平拆分为多个物理子表，分布在不同的数据库实例或同一个实例的不同表中。

```
逻辑表 users (1000万行)
    │
    │ 按 id % 4 分片
    │
    ├──→ db0.users (id=0,4,8,...)     # 分片 0
    ├──→ db1.users (id=1,5,9,...)     # 分片 1
    ├──→ db2.users (id=2,6,10,...)    # 分片 2
    └──→ db3.users (id=3,7,11,...)    # 分片 3
```

# 2. Schema 分片配置

```json
{
  "schemas": {
    "order_db": {
      "tables": {
        "orders": {
          "type": "sharding",
          "dataNodes": "dn0.orders,dn1.orders,dn2.orders,dn3.orders",
          "sharding": {
            "function": {
              "clazz": "io.mycat.router.function.PartitionByMod",
              "properties": {
                "columnName": "id",
                "count": 4
              }
            }
          }
        }
      }
    }
  }
}
```

# 3. 分片算法

## 3.1 取模分片（PartitionByMod）

```json
{
  "clazz": "io.mycat.router.function.PartitionByMod",
  "properties": {
    "columnName": "id",
    "count": 4
  }
}
```

**计算**：`分片 = id % count`

**特点**：简单均匀，但扩容时需要大量数据迁移。

## 3.2 范围分片（PartitionByRange）

```json
{
  "clazz": "io.mycat.router.function.PartitionByRange",
  "properties": {
    "columnName": "id",
    "ranges": "0-1000000=0,1000001-2000000=1,2000001-3000000=2,3000001-4000000=3"
  }
}
```

**特点**：方便按范围扩容，但可能存在热点。

## 3.3 一致性 Hash（PartitionByMurmurHash）

```json
{
  "clazz": "io.mycat.router.function.PartitionByMurmurHash",
  "properties": {
    "columnName": "id",
    "count": 256,
    "seed": 0,
    "virtualBucketTimes": 160
  }
}
```

**特点**：扩容时数据迁移量小，适合对分布均匀性要求高的场景。

## 3.4 日期分片（PartitionByDate）

```json
{
  "clazz": "io.mycat.router.function.PartitionByDate",
  "properties": {
    "columnName": "create_time",
    "dateFormat": "yyyy-MM-dd",
    "sBeginDate": "2024-01-01",
    "sPartionDay": "30"
  }
}
```

**特点**：按时间自然分片，适合日志、订单等时间序列数据。

## 3.5 枚举分片（PartitionByIntFileMap）

```json
{
  "clazz": "io.mycat.router.function.PartitionByIntFileMap",
  "properties": {
    "columnName": "region_id",
    "mapFile": "partition-hash-int.txt",
    "type": "Integer"
  }
}
```

**partition-hash-int.txt**：
```
1=0    # 华东 → 分片 0
2=1    # 华南 → 分片 1
3=2    # 华北 → 分片 2
4=3    # 西南 → 分片 3
```

## 3.6 算法对比

| 算法 | 均匀性 | 扩容成本 | 范围查询 | 适用场景 |
|------|--------|---------|---------|---------|
| 取模 | ★★★★★ | 高（全量迁移） | 差 | 均匀分布主键 |
| 范围 | ★★★☆☆ | 低（追加新范围） | 好 | 区间查询频繁 |
| 一致性Hash | ★★★★☆ | 中（部分迁移） | 差 | 缓存/分布式存储 |
| 日期 | ★★★★☆ | 低（自然追加） | 好 | 日志/订单 |
| 枚举 | ★★★★☆ | 低（按枚举扩） | 好 | 地区/类型字段 |

# 4. 分片键选择原则

```
✓ 好的分片键：
  - 高区分度（唯一或接近唯一）
  - 查询中高频出现（WHERE 条件包含）
  - 不可变（一旦写入不修改）
  - 数据分布均匀

✗ 差的分片键：
  - 低区分度（只有少数几个值）→ 数据倾斜
  - 查询中不出现 → 全分片扫描
  - 会频繁修改 → 数据迁移
```

# 5. 分库分表组合

```json
{
  "tables": {
    "orders": {
      "type": "sharding",
      "dataNodes": "db0.orders_0,db0.orders_1,db1.orders_0,db1.orders_1",
      "sharding": {
        "function": {
          "clazz": "io.mycat.router.function.PartitionByMod",
          "properties": {
            "columnName": "id",
            "count": 4
          }
        }
      }
    }
  }
}
```

**分片布局**：

```
db0 (MySQL 实例 1)          db1 (MySQL 实例 2)
  ├── orders_0 (分片 0)       ├── orders_0 (分片 2)
  └── orders_1 (分片 1)       └── orders_1 (分片 3)
```

# 6. 扩容策略

## 6.1 停机扩容

```bash
# 1. 停止写入
# 2. 导出数据
mysqldump -h old_db -u root -p order_db orders > orders.sql

# 3. 修改分片配置（count: 4 → 8）

# 4. 重新导入数据到新分片

# 5. 重启 Mycat2 加载新配置
```

## 6.2 在线扩容（推荐配合 Canal）

```
1. Mycat2 配置新分片（但不启用）
2. 通过 Canal 同步增量数据到新分片
3. 全量迁移历史数据到新分片
4. 验证数据一致
5. Mycat2 热加载新分片配置
6. 新数据同时写入新老分片
7. 下线旧分片
```

# 7. 跨分片查询处理

| SQL 类型 | 处理方式 |
|---------|---------|
| `WHERE id = 1` | 精确分片 → 单节点执行 |
| `WHERE id IN (1,5,9)` | 多分片并行 → 结果合并 |
| `ORDER BY age LIMIT 10` | 全分片执行 → 归并排序 |
| `GROUP BY type` | 全分片聚合 → 二次聚合 |
| `COUNT(*)` | 全分片统计 → 求和 |
| `JOIN` | 需配合 ER 表或全局表 |

跨分片查询的性能随分片数量线性下降，应尽量避免全分片扫描。
