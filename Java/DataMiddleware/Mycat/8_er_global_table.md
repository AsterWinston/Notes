
# 1. 概述

在分片场景下，多表关联查询面临挑战：

```
SELECT o.*, u.name
FROM orders o JOIN users u ON o.user_id = u.id
-- 如果 orders 和 users 不在同一分片 → 无法直接 JOIN
```

Mycat2 提供两种解决方案：**ER 表**和**全局表**。

# 2. ER 表（父子表关联分片）

## 2.1 原理

ER 表将**子表与父表**按同一分片键存储在同一数据节点，使关联查询本地化。

```
分片 0 (db0)                    分片 1 (db1)
├── orders (id=1)               ├── orders (id=2)
│     └── order_items (order_id=1) │     └── order_items (order_id=2)
│           ├── item 1              │           ├── item 3
│           └── item 2              │           └── item 4
```

## 2.2 配置

```json
{
  "schemas": {
    "order_db": {
      "tables": {
        "orders": {
          "type": "sharding",
          "dataNodes": "dn0.orders,dn1.orders",
          "sharding": {
            "function": {
              "clazz": "io.mycat.router.function.PartitionByMod",
              "properties": { "columnName": "id", "count": 2 }
            }
          }
        },
        "order_items": {
          "type": "sharding",
          "dataNodes": "dn0.order_items,dn1.order_items",
          "parentName": "orders",        // 父表
          "joinKey": "order_id",         // 关联键
          "parentKey": "id",              // 父表主键
          "sharding": {
            "function": {
              "clazz": "io.mycat.router.function.PartitionByMod",
              "properties": { "columnName": "order_id", "count": 2 }
            }
          }
        }
      }
    }
  }
}
```

## 2.3 查询效果

```sql
-- 关联查询会路由到同一分片
SELECT o.*, i.*
FROM orders o
JOIN order_items i ON o.id = i.order_id
WHERE o.id = 1;
-- → 路由到 dn0，两个表在同一数据库，正常 JOIN

-- 不带分片键的查询 → 全分片扫描
SELECT o.*, i.*
FROM orders o
JOIN order_items i ON o.id = i.order_id;
-- → 每个分片独立 JOIN，结果归并
```

# 3. 全局表（广播表）

## 3.1 原理

全局表在**每个数据节点**存放完整副本，所有分片都可以本地关联。

```
分片 0 (db0)              分片 1 (db1)
├── orders (id=1,4)       ├── orders (id=2,5)
├── dict_province (全量)    ├── dict_province (全量)
└── dict_city (全量)       └── dict_city (全量)
```

## 3.2 配置

```json
{
  "tables": {
    "dict_province": {
      "type": "global",
      "dataNodes": "dn0.dict_province,dn1.dict_province"
    },
    "dict_city": {
      "type": "global",
      "dataNodes": "dn0.dict_city,dn1.dict_city"
    }
  }
}
```

## 3.3 全局表行为

| 操作 | 行为 |
|------|------|
| INSERT | 写入所有数据节点（多节点事务） |
| UPDATE | 更新所有数据节点 |
| DELETE | 删除所有数据节点 |
| SELECT | 随机选一个节点查询 |

```java
// 伪代码：全局表 INSERT
public void insert(String table, Row row) {
    List<DataNode> nodes = getAllDataNodes(table);
    // 开启分布式事务
    beginXA();
    try {
        for (DataNode node : nodes) {
            node.executeInsert(table, row);
        }
        commitXA();
    } catch (Exception e) {
        rollbackXA();
    }
}
```

# 4. ER 表 vs 全局表

| 维度 | ER 表 | 全局表 |
|------|-------|--------|
| 数据量 | 大（随业务增长） | 小（一般 < 10 万行） |
| 存储 | 每个子表记录仅在一个分片 | 每行在所有分片存储 |
| 写入开销 | 单分片写入 | 多分片事务写入 |
| 查询开销 | 带分片键 → 单分片 | 任意分片都可查 |
| 适用场景 | 订单+订单明细 | 字典表、配置表、地区表 |

# 5. 三层关联

```json
{
  "tables": {
    "orders": {
      "type": "sharding",
      "dataNodes": "dn0.orders,dn1.orders",
      "sharding": { ... }
    },
    "order_items": {
      "type": "sharding",
      "parentName": "orders",
      "joinKey": "order_id",
      "parentKey": "id",
      "dataNodes": "dn0.order_items,dn1.order_items"
    },
    "order_item_details": {
      "type": "sharding",
      "parentName": "order_items",
      "joinKey": "item_id",
      "parentKey": "id",
      "dataNodes": "dn0.order_item_details,dn1.order_item_details"
    }
  }
}
```

```
orders → order_items → order_item_details
 三级 ER 关联，所有关联数据聚集在同一分片
```

# 6. 注意事项

1. **全局表不宜过大**：数据量大时写入性能急剧下降（需多节点事务）
2. **ER 表分片键应包含在 WHERE 中**：否则退化为全分片扫描
3. **全局表修改需谨慎**：任何 UPDATE/DELETE 都是分布式事务
4. **全局表与 ER 表可组合**：全局字典表 + ER 子表配合使用
5. **避免深层 ER 嵌套**：ER 层级越深，跨分片查询越复杂
